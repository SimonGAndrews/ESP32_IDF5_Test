# ESP32-S3 WIFI-011 mutex owner diagnostic

The diagnostic build reproduced the original time-lock abort on the bench
ESP32-S3. It recorded `espruinoTask` as the waiter and the native `ping` task
as the owner of newlib's `s_time_lock` in both controlled runs. These runs
deliberately extended the lock hold to 250 ms. They establish the mechanism
when ping holds the mutex. The owner in the earlier Web IDE crash remains
unknown. [WIFI-011 explanation](../../../docs/investigations/wifi/wifi-011-timer-lock.md)

## Build and hardware

- 25 November 2026. ESP32-S3 QFN56 revision 0.1 with 8 MB embedded PSRAM,
  MAC `68:b6:b3:27:2c:40`. Native USB Serial/JTAG at 115200 baud. No access
  point or harness jumpers were needed for the loopback tests.
- Espruino `2v29.539` at `ed5daf244d0e396bb4689f3f57e2736dc093e1f7`.
  Used `BOARD=ESP32S3 RELEASE=1` with `DEBUG` unset. The build reports this
  baseline revision because the instrumentation is in ESP-IDF.
- ESP-IDF `v5.5.3` at `2c211b236707889e8400c4dc5644dd5c4ee071e0`, with the
  diagnostic patch applied to a private copy. The SDK reports `v5.5.3-dirty`.
  [ESP-IDF diagnostic patch](../../../docs/investigations/wifi/wifi-011-lock-owner-diagnostic.patch)
- Flashed the application at `0x10000` using `esptool` 4.12.0 at 460800 baud.
  The write passed hash verification. Both panics reported ELF prefix
  `2822e36f1818ba24` and the firmware returned to the USB REPL after reboot.

| Artifact | SHA-256 |
| --- | --- |
| `espruino.elf` | `2822e36f1818ba241b8157ec1611e20d16aadeb425534a7bfab48707678b22fa` |
| `espruino.bin` | `744f264b2a53024ef3489efb3760fda06320c2fbecca49610e5411bb9c28a48d` |

## How the owner is captured

The diagnostic wraps the two native ping calls to `gettimeofday()` with
a task-local tag containing the ping task handle and call-site line. When
FreeRTOS grants `s_time_lock`, it records the tag alongside the owner. If
the interpreter subsequently tries and fails to take the mutex in its
critical section, the diagnostic snapshots the owner from FreeRTOS's mutex
bookkeeping while that bookkeeping is still protected. Newlib prints the
snapshot after the FreeRTOS bookkeeping lock has been released, immediately
before its usual abort. The failing stack independently identifies the
interpreter's clock read.

## Repeat the check

For a rebuild, apply the diagnostic patch to a copy of ESP-IDF 5.5.3 and source
that copy's `export.sh`. Build the baseline Espruino checkout with
`export ESP_IDF_PATH="$IDF_PATH"` and `make BOARD=ESP32S3 RELEASE=1 -j2`.
Flash the resulting S3 image and connect to the REPL over USB.

Run one test at a time. The two diagnostic addresses select different native
clock reads. Each deliberately delays the ping task while it owns the mutex,
leaving the interpreter free to run and attempt a competing clock read.

```javascript
var tick = setInterval(function () { getTime(); }, 10);
setTimeout(function () { clearInterval(tick); }, 7000);
require("Wifi").ping("127.0.0.2");
```

| Loopback address | Selected clock read in `esp_ping_thread()` | Observed trace |
| --- | --- | --- |
| `127.0.0.2` | Before receive, diagnostic `ping_sock.c:182` (stock line 172) | `waiter=espruinoTask owner=ping ping_sock_line=182 forced_hold_ms=250` |
| `127.0.0.3` | After receive, diagnostic `ping_sock.c:185` (stock line 174) | `waiter=espruinoTask owner=ping ping_sock_line=185 forced_hold_ms=250` |

The full diagnostic line also prints the observed owner handle. Both traces
then decoded to the same waiter path.

```text
espruinoTask → jsiLoop → jsiIdle → jshGetSystemTime
             → gettimeofday → _gettimeofday_r → get_adjusted_boot_time
             → _lock_acquire → lock_acquire_generic → abort
```

Use `127.0.0.1` to observe ordinary loopback timing with the same owner
capture. If the time-lock abort occurs in that mode, `forced_hold_ms=0` in the
diagnostic line confirms that no deliberate delay was inserted for that read.

After the two controlled reboots, ten ordinary `127.0.0.1` sessions were
started 3.5 seconds apart with a 10 ms clock-reading timer. The optional
ping callback was omitted to avoid the separate JavaScript result-handoff
path. The REPL reached `NORMAL_DONE` and the serial capture contained no
time-lock abort. The diagnostic remains available on the board for more
ordinary-loopback observations.
