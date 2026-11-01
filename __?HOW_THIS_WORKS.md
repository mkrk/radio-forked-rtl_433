# How rtl_433 Works — Q&A

A walkthrough of the rtl_433 codebase: entry point, program lifecycle, how it
talks to the SDR hardware, and its networking. Written as a running Q&A.

---

## Q1: Where's the entry point — which part of the code starts when the program is executed?

**Entry point: `int main(int argc, char **argv)` at `src/rtl_433.c:1439`.**

This is the `main` function for the `rtl_433` executable, which is the sole
program target — defined in `src/CMakeLists.txt:486`
(`add_executable(rtl_433 rtl_433.c)`).

### What happens when the program starts (first part of `main`):

1. **`print_version()`** — prints version info (line 1444)
2. **`r_init_cfg(cfg)`** — initializes the global config struct `g_cfg` (line 1447)
3. **Config loading** — looks for a default conf file if `-c` isn't passed, then
   parses command-line args via `parse_conf_args()` (lines 1455–1459)
4. **Frequency / sample-rate setup** — applies defaults like `DEFAULT_FREQUENCY` (lines 1461–1470)
5. **Input files** — remaining positional args are added as input files via `add_infile()` (lines 1473–1475)
6. **Pulse detection, output handlers, logging** set up (lines 1477–1520)
7. **`register_all_protocols()`** — registers all the default device decoders unless disabled (line 1524)
8. ...then it sets up the SDR device / file reading loop and runs the demodulation main loop.

### A note on the other `main`s
Several other files contain `main` functions (`abuf.c`, `bit_util.c`,
`bitbuffer.c`, `fileformat.c`, `optparse.c`, `r_util.c`), but those are
**unit-test / self-test harnesses** guarded by compile-time flags and are not
part of the normal `rtl_433` binary. The real program entry point is the one in
`src/rtl_433.c`.

---

## Q2: Walk through the rest of `main` and the decode pipeline.

After setup, `main` branches into one of several mutually-exclusive modes, then
(in live mode) enters an event loop.

### 1. Mode dispatch (which "input" is active)

`main` checks these in order, each ending in `exit()`:

| Lines | Mode | Trigger |
|-------|------|---------|
| 1587–1674 | **Streaming test data** | `-y -` / `-y @file` (reads lines of hex/rfraw from stdin or a file) |
| 1676–1697 | **String test data** | `-y "<string>"` (one decode of a literal string) |
| 1699–1878 | **File input mode** | positional sample files / `-r file` |
| 1880+ | **Live SDR** (the normal case) | no test data, no input files |

The test-data and file paths feed samples/pulses directly into the decoders and
then `exit()` — they never reach the live event loop.

### 2. File input mode (`src/rtl_433.c:1699`)
For each input file it detects the format from the filename
(`file_info_parse_filename`), then either:
- **Pulse/OOK files** (`PULSE_OOK`): loads pulse data and runs
  `run_fsk_demods` / `run_ook_demods` directly (lines 1767–1806).
- **IQ/AM/FM sample files**: reads `DEFAULT_BUF_LENGTH` blocks in a `do/while`
  loop, converts format if needed (CF32→CS16, CS8→CU8), and calls
  **`process_sdr_frame()`** per block (line 1853) — the same function the live
  path uses.

### 3. Live SDR mode (`src/rtl_433.c:1886` → end)
1. **Signal handlers** installed (`SIGINT`, `SIGTERM`, etc. → `sighandler`, which
   sets `exit_async`) — lines 1886–1900.
2. **`start_sdr(cfg)`** (line 1906) opens the radio and starts a background
   thread that pumps samples.
3. A dummy mongoose socket + timer is registered for periodic events (frequency
   hopping, stats) — lines 1919–1923.
4. **The main loop** (lines 1925–1927):
   ```c
   while (!cfg->exit_async) {
       mg_mgr_poll(cfg->mgr, 500);
   }
   ```
   This is the real "running" state of the program — a mongoose network event
   loop. It polls for: SDR sample batches broadcast from the capture thread, the
   hop/stats timer, and any network outputs (MQTT, HTTP, syslog).
5. On exit: `sdr_stop`, flush stats, `close_dumpers`, `r_free_cfg`, and `return`
   the exit code (lines 1935–1954).

### 4. How samples become decoded messages

Shared by file and live modes:

```
SDR thread → sdr_handler()           src/rtl_433.c:1200   (live: via mg_broadcast, rtl_433.c:1279)
   → process_sdr_frame()             src/rtl_433.c:1086   (side effects: dumpers, analyzers)
      → push_sdr_flow()              src/r_flow.c:104     (the DSP core)
         → AM/FM demodulation + pulse_detect_package()   src/r_flow.c:243
            → run_ook_demods()       src/r_flow.c:259     } calls each
            → run_fsk_demods()       src/r_flow.c:302     } r_device->decode_fn
               → protocol decoders   src/devices/*.c
```

`push_sdr_flow()` converts the raw IQ buffer to AM/FM envelopes, slices it into
pulses via `pulse_detect_package()`, and when a complete packet of pulses is
found hands it to `run_ook_demods` / `run_fsk_demods`, which loop over every
registered `r_device` and invoke its `decode_fn`. Those decoders live in
`src/devices/` — one file per protocol — registered at
`register_all_protocols()` (line 1524).

### TL;DR of the lifecycle
```
main (rtl_433.c:1439)
  ├─ init config, parse args, load conf
  ├─ register_all_protocols()            ← all src/devices/*.c decoders
  ├─ start_outputs()                     ← kv/json/mqtt/etc.
  ├─ [test-data mode]  → run_*_demods → exit
  ├─ [file mode]       → process_sdr_frame loop → exit
  └─ [live SDR mode]
        start_sdr()                      ← capture thread
        while (!exit_async)
            mg_mgr_poll()                ← event loop (samples, timers, net)
                 └─ sdr_handler → process_sdr_frame → push_sdr_flow
                        → pulse detect → run_ook/fsk_demods → decoders
        sdr_stop / cleanup / return exit code
```

---

## Q3: What do "mode dispatch", "file input mode" and "live SDR mode" actually mean?

These are just labels for the three different ways rtl_433 can get its input.
rtl_433 decodes radio signals, but the radio data can come from three different
places, and `main()` picks one based on the command-line options.

### Mode dispatch — not a mode, just the decision
It's the **branching logic** where `main` looks at your options and decides which
path to run:
```
Did you pass test data (-y)?        → test-data mode
Did you pass a file to read (-r)?   → file input mode
Neither?                            → live SDR mode
```

### File input mode — "read radio samples from a file on disk"
You already recorded the signal (or downloaded a recording), e.g.:
```
rtl_433 -r mysignal.cu8
```
rtl_433 reads samples from the file and decodes them. **No radio hardware
involved.** Used for testing/debugging decoders and analyzing captured signals.
(The `-y` "test data" variant is the same idea with a short string on the
command line.)

### Live SDR mode — "read radio samples from an actual radio in real time"
The normal, everyday use. SDR = **Software-Defined Radio** — a USB dongle (like
an RTL-SDR stick) that listens to radio frequencies. You run:
```
rtl_433
```
with no input file. It opens the USB radio, tunes it (default 433.92 MHz), and
continuously decodes live signals until you press Ctrl-C.

### Key difference between the two modes
Both do the **exact same decoding** and share the same core function
(`process_sdr_frame` → `push_sdr_flow`). The only difference is **where the raw
samples come from**:

| | Source of samples | Runs forever? | Needs hardware? |
|---|---|---|---|
| **File input mode** | a file on disk | no — stops at end of file | no |
| **Live SDR mode** | a physical radio dongle | yes — until Ctrl-C | yes |

---

## Q4: In live SDR mode, how does the program know how to query the SDR dongle, since it can be connected to various USB ports?

**rtl_433 doesn't touch USB ports directly — it uses device indices.** USB
dongles aren't addressed by port; they're addressed by an **index number** (and
optionally a serial) that the underlying library assigns.

### The call chain
```
main (rtl_433.c:1906)
  → start_sdr (rtl_433.c:1291)
      → sdr_open (sdr.c)
          → sdr_open_rtl (sdr.c:361)   ← for RTL-SDR dongles
```

### What `sdr_open_rtl` does (`src/sdr.c:361`)

**Step 1 — enumerate all dongles** (line 363):
```c
uint32_t device_count = rtlsdr_get_device_count();
```
`librtlsdr` uses **libusb** to scan the whole USB bus and count every device
matching a known RTL-SDR chip — regardless of port. Prints "No supported devices
found" and bails if the count is 0.

**Step 2 — decide which one to open.** The `dev_query` string comes from the
`-d` option (`rtl_433.c:523`):

| `-d` value | Code path | Meaning |
|---|---|---|
| *(omitted)* | loops over all devices | "use the first one that opens" |
| `-d 1` | `atoi(dev_query)` → index 1 (line 386) | "use device index #1" |
| `-d :00000123` | `rtlsdr_get_index_by_serial()` (line 375) | "find the dongle with serial 00000123" |

**Step 3 — open it** (loop at lines 406–436):
```c
rtlsdr_get_device_usb_strings(i, vendor, product, serial);  // read USB descriptor
r = rtlsdr_open(&dev->rtlsdr_dev, i);                        // open device #i
```
With no `-d`, it grabs the first device that opens successfully. Once open, the
library returns an opaque handle (`dev->rtlsdr_dev`) used for all further calls.

### Summary
1. Dongles are **not** addressed by port — libusb/librtlsdr enumerate the bus and
   assign a 0-based **index** (and expose a serial).
2. With **no `-d`**, rtl_433 takes the **first device that opens**.
3. With **`-d <n>`** pick by index; with **`-d :<serial>`** pick by serial — the
   serial is the robust way to always get the same physical dongle.

Other backends in `sdr.c`: `sdr_open_soapy` (line 877, SoapySDR devices like
HackRF/LimeSDR) and `rtltcp_open` (line 134, network source). `sdr_open`
dispatches based on the `-d` string.

---

## Q5: How does the program communicate with the dongle once it's opened (`rtlsdr_open(&dev->rtlsdr_dev, i)`)? Is it named pipes or something totally different?

**Totally different — no pipes, sockets, files, or OS IPC.** The dongle is a USB
device, so communication happens over the **USB bus** via **libusb**, which
issues **USB transfers** through kernel `ioctl` calls.

### The layers
```
rtl_433 (src/sdr.c)
  rtlsdr_open(), rtlsdr_read_async(), rtlsdr_set_center_freq(), ...
        │  normal C function calls (linked library)
librtlsdr  (external .so / .dll)
  translates each call into USB control/bulk transfers
        │  libusb_control_transfer() / libusb_submit_transfer()
libusb  (external .so / .dll)
  packages USB Request Blocks (URBs)
        │  ioctl() on /dev/bus/usb/BBB/DDD  (Linux usbfs)
OS kernel USB stack  →  USB host controller (xHCI)
        │  actual electrical USB packets on the wire
The RTL-SDR dongle (RTL2832U chip + tuner)
```

rtl_433 only ever calls `rtlsdr_*` functions. Those are **ordinary C function
calls into a linked library in the same process** — not IPC.

### Two kinds of USB traffic

**1. Control traffic — small commands (frequency, gain, sample rate).**
`rtlsdr_set_center_freq()` (`sdr.c:1252`), `rtlsdr_set_sample_rate()`
(`sdr.c:1503`), etc. Each becomes a **USB control transfer**
(`libusb_control_transfer()`) that writes a register on the chip/tuner. Quick
request/response.

**2. Bulk streaming — the firehose of IQ samples.**
The dongle streams millions of bytes/sec over a **USB bulk endpoint**. rtl_433
receives it via **`rtlsdr_read_async()`** (`sdr.c:553`):
```c
r = rtlsdr_read_async(dev->rtlsdr_dev, rtlsdr_read_cb, dev, buf_num, buf_len);
```
Under the hood in librtlsdr:
1. Allocates `buf_num` buffers and submits them to libusb as **async USB
   transfers** (`libusb_submit_transfer`).
2. The kernel USB driver fills those buffers directly (often via DMA).
3. When a buffer fills, libusb invokes librtlsdr's completion callback, which
   calls **your** callback `rtlsdr_read_cb` (`sdr.c:~495`).
4. `rtlsdr_read_cb` calls `dev->rtlsdr_cb(...)` (`sdr.c:527`) — rtl_433's handler
   that pushes samples into the decode pipeline (`push_sdr_flow`).

`rtlsdr_read_async()` blocks and loops forever, which is why rtl_433 runs it on a
**separate thread** while the main thread polls the mongoose loop.
`rtlsdr_cancel_async()` (`sdr.c:1762`) stops it; `rtlsdr_reset_buffer()`
(`sdr.c:1706`) flushes stale samples before streaming starts.

### Why async callbacks instead of a pipe-like read?
There's a synchronous variant (`rtlsdr_read_sync`), but rtl_433 uses async
because the sample rate is too high to allow gaps: with multiple buffers the
kernel fills buffer #2 via DMA while your callback processes buffer #1 — no
dropped samples, minimal copying. A pipe would add a kernel copy and a sync
point.

### Direct answer
No IPC between rtl_433 and a separate driver process. librtlsdr and libusb are
**libraries linked into the rtl_433 process** — calling them is calling C
functions in the same address space. They move data over the USB bus using USB
**control transfers** (commands) and async **bulk transfers** (samples), reaching
the kernel via `ioctl()` on a USB device node. The handle `dev->rtlsdr_dev` is
just an opaque pointer to a librtlsdr struct, not a file descriptor you read.

---

## Q6: Is the communication chain the same on Windows? Does Windows have libusb too?

**The top of the chain is identical; the bottom is different.** rtl_433 and
librtlsdr are written against libusb, which is cross-platform — so rtl_433's own
code doesn't change. How libusb reaches the hardware, and which driver sits on
the dongle, differ on Windows.

Evidence from this repo: `src/sdr.c` `#include <libusb.h>` on every platform
(line 34). The `_WIN32` guards in `sdr.c` (lines 51, 150, 1819, 1827) are about
**networking/sockets and signals**, **not USB**. There is zero Windows-specific
USB code in rtl_433 itself.

### The chain, Linux vs Windows
```
                         LINUX                              WINDOWS
rtl_433 (src/sdr.c)   rtlsdr_* calls          rtl_433 (src/sdr.c)   rtlsdr_* calls   ← IDENTICAL
librtlsdr                                     librtlsdr.dll                          ← IDENTICAL (recompiled)
libusb (.so)                                  libusb-1.0.dll                         ← SAME API, different backend
usbfs → ioctl()                               WinUSB API (or libusbK/libusb0)        ← DIFFERENT
kernel USB stack (usbcore/xHCI)               WinUSB.sys + Windows USB stack          ← DIFFERENT
RTL-SDR dongle                                RTL-SDR dongle                          ← same hardware
```

### What's different
1. **libusb's backend.** Linux: usbfs via `ioctl()` on `/dev/bus/usb/BBB/DDD`.
   Windows: no usbfs — the backend drives **WinUSB** (`winusb.dll`), or
   **libusbK** / legacy **libusb0.sys**, using `DeviceIoControl` + overlapped I/O.
   The same `libusb_control_transfer()` / `libusb_submit_transfer()` calls work on
   both.
2. **The driver on the dongle (the big practical difference).** Linux auto-loads a
   driver and libusb can claim the device (aside from blacklisting the
   `dvb_usb_rtl28xxu` DVB-T driver). Windows has **no usable driver out of the
   box** — you must install **WinUSB** via the **Zadig** utility, or
   `rtlsdr_open()` fails even though the code is identical.
3. **Async I/O model.** Linux: kernel + libusb event loop on `ioctl`/`epoll`.
   Windows: overlapped I/O via WinUSB. libusb hides this — rtl_433 still just gets
   its `rtlsdr_read_cb` callback (`sdr.c:527`).

### Direct answers
- **Same chain?** Yes down through libusb (same design, recompiled source; same
  USB control + async bulk transfer semantics). No below it (WinUSB/DeviceIoControl
  vs usbfs/ioctl; different kernel driver).
- **Does Windows have libusb?** Yes — `libusb-1.0` is cross-platform; on Windows
  it ships as `libusb-1.0.dll` over WinUSB/libusbK/libusb0. The one extra Windows
  requirement is binding the WinUSB driver to the dongle (via Zadig).

The fact that `src/sdr.c`'s only `_WIN32` guards are for sockets and signals —
never USB — is direct evidence the USB path is fully delegated to libusb and
portable without rtl_433-side changes.

---

## Q7: What networking/sockets does this program use? Does it expose an endpoint, or connect to one?

**Both** — rtl_433 is a network *client* (connecting out) for most things, and
can also be a *server* (listening) for a couple of features. There are two
separate networking subsystems with different code:

| Subsystem | Library used | Where |
|---|---|---|
| **SDR input over network** (`rtl_tcp`) | **raw BSD sockets** (`socket`/`connect`/`recv`/`listen`/`accept`) | `src/sdr.c`, `src/output_rtltcp.c` |
| **Data outputs** (MQTT, InfluxDB, HTTP, UDP, syslog) | **mongoose** (bundled embedded net library, `src/mongoose.c`) | `src/output_*.c`, `src/http_server.c` |

The `_WIN32`/Winsock guards are in the **raw-socket** parts, because raw BSD
sockets differ between POSIX and Windows (`<sys/socket.h>` vs `<winsock2.h>`,
`close()` vs `closesocket()`, needing `WSAStartup()`). Mongoose hides those
internally.

### As a CLIENT — rtl_433 connects *out*
- **`rtl_tcp` input** (`src/sdr.c:134`, `rtltcp_open`) — connects to a remote
  `rtl_tcp` server (`-d rtl_tcp://host:port`) to receive raw IQ samples.
  `getaddrinfo` → `socket()` → `connect()` → `recv()` (`sdr.c:176`, `:278`).
- **MQTT output** (`src/output_mqtt.c`, `-F mqtt://...`) — `mg_connect_opt` to a broker.
- **InfluxDB output** (`src/output_influx.c`, `-F influx://...`) — `mg_connect_http_opt` (line 183).
- **UDP output** (`src/output_udp.c`, `-F syslog:...`) — UDP datagrams to a collector.
- **HTTP/trigger outputs** — POST decoded events to a URL.

### As a SERVER — rtl_433 listens for inbound connections
- **HTTP server** (`src/http_server.c:1253`, `mg_bind_opt`) — the `-F http`
  output. Serves decoded events (REST/JSON + streaming). Clients connect to it.
- **`rtl_tcp` server output** (`src/output_rtltcp.c:264`) — re-serves the raw IQ
  stream to other `rtl_tcp` clients via `listen()` (line 264) and `accept()`
  (line 277).

### mongoose and the event loop
The main loop `while (!cfg->exit_async) { mg_mgr_poll(cfg->mgr, 500); }`
(`rtl_433.c:1925`) is **mongoose's** event loop. It services SDR timers *and* all
mongoose-based network outputs (MQTT keepalives, HTTP client/server). That's how
one single-threaded poll loop handles decoding plus several network endpoints.
The raw-socket `rtl_tcp` paths are the exception — they run on the SDR thread
with blocking `recv()`/`send()`, which is why they carry their own `_WIN32`
Winsock handling.

### Direct answer
**Both, depending on what you enable:**
- **Connects out (client)** — default for outputs: MQTT, InfluxDB, HTTP POST,
  UDP/syslog, and the `rtl_tcp://` *input* source.
- **Listens (server)** — only with the `-F http` output or the `rtl_tcp` *output*.

Out of the box with no network options, rtl_433 opens **no listening ports and
makes no outbound connections** — it just prints to the console. Networking
activates only when you add a network `-F` output or use a `rtl_tcp` source.

The `_WIN32` guards in `sdr.c` exist purely because the `rtl_tcp` feature uses
raw BSD sockets needing Winsock shims — nothing to do with the USB/SDR path,
which is handled by cross-platform libusb.

---

## Q8: So this program connects to the SDR dongle via `rtl_tcp://`?

**Not normally.** There are two separate ways rtl_433 gets samples, and
`rtl_tcp://` is the less common one.

### Default: direct USB (no `rtl_tcp`)
Running `rtl_433` with a local dongle uses the USB chain:
```
rtl_433 → librtlsdr → libusb → USB bus → dongle
```
via `sdr_open_rtl()` (`src/sdr.c:361`) and `rtlsdr_open()`. No networking.

### Optional: `rtl_tcp://` (dongle on another machine)
Used only when the dongle is plugged into a **different computer** running the
separate `rtl_tcp` server utility:
- **Machine A** has the dongle on USB and runs `rtl_tcp` → does the
  librtlsdr/libusb/USB work locally.
- **Machine B** runs `rtl_433 -d rtl_tcp://machineA:1234` → has no dongle, and
  connects over TCP to pull the sample stream. This goes through `rtltcp_open()`
  (`src/sdr.c:134`) with raw BSD sockets (the code with the `_WIN32`/Winsock
  guards).

### Which path is chosen (`-d` option)

| Command | How samples arrive | Code path |
|---|---|---|
| `rtl_433` *(no -d, or `-d 0`)* | **Direct USB** to local dongle | `sdr_open_rtl` → `rtlsdr_open` → libusb |
| `rtl_433 -d rtl_tcp://host:port` | **TCP network** to a remote `rtl_tcp` server | `rtltcp_open` → BSD sockets |

`sdr_open` dispatches by sniffing the `-d` string (`src/sdr.c:1138`): if it starts
with `"rtl_tcp"`, use the network path; otherwise direct USB.

### Direct answer
By default rtl_433 talks to a **local dongle over USB** (libusb) — no `rtl_tcp`,
no networking. `rtl_tcp://` is only for reading from a dongle attached to a
**remote machine** over TCP. The dongle is always a USB device; `rtl_tcp` just
relays its samples across a TCP link when the dongle isn't on the same computer
as rtl_433.
