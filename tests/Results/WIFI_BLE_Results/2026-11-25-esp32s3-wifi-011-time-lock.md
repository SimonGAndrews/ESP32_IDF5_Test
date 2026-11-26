# ESP32-S3 WIFI-011 timekeeping patch

This run compares the ESP32 timekeeping patch with the firmware that produced
the Web IDE `gettimeofday()` abort on 25 November 2026. The patch keeps the
blocking clock read in task context. GPIO interrupts and other callers that
cannot yield use `esp_timer_get_time()` and a synchronized wall-clock offset.
The change is independent of the ping callback handoff in issue #2797.
The single-commit export applies to the baseline. [Standalone patch](../../../docs/investigations/wifi/0001-ESP32-avoid-blocking-time-reads-in-interrupt-context.patch)

## Bench setup and firmware

- ESP32-S3 QFN56 revision 0.1, MAC `68:b6:b3:27:2c:40`, USB-powered with
  no harness jumpers. Native USB Serial/JTAG console at 115200 baud on
  `/dev/serial/by-id/usb-Espressif_USB_JTAG_serial_debug_unit_68:B6:B3:27:2C:40-if00`.
- Baseline `ed5daf244d0e396bb4689f3f57e2736dc093e1f7`, Espruino `2v29.539`.
  Its ELF hash begins `7e8c3263ee2f3836`, matching the Web IDE abort trace.
  The decoded stack goes from `espruinoTask` through `jsiIdle`,
  `jshGetSystemTime()`, and `gettimeofday()` to the newlib mutex abort.
- Candidate `896255bffbe9fe8d3b70bb7118c34130e92dec71`, Espruino `2v29.540`,
  a single commit on top of the baseline. Used ESP-IDF 5.5.3
  (`2c211b236707889e8400c4dc5644dd5c4ee071e0`) for both builds. Build flags
  were `BOARD=ESP32S3 RELEASE=1` with `DEBUG` unset. Flashed only the candidate
  application at `0x10000` with `esptool` at 460800 baud. The write passed hash
  verification and the REPL confirmed the candidate version and revision.

| Candidate artifact | SHA-256 |
| --- | --- |
| `bin/build/espruino.elf` | `64be2171f7fd96ea5a58588fe42e7f14ec0e79b9a567cbf1212a29b42565d418` |
| `bin/build/espruino.bin` | `1598453a15d3a653f40d8d81c0f291a6339ef9b61fca71c338ed0d5233a1315e` |

## Procedure

1. Verify the board, Espruino version, and source commit over the REPL. Reboot
   the chip and disable echo. For each of ten loopback trials, reset `count`
   to zero, call `Wifi.ping("127.0.0.1", callback)`, print each numbered callback
   with `JSON.stringify(result)`, and print `COUNT` from a 7000 ms timeout.
   Allow up to ten seconds on the host, reboot after a callback or timer
   failure, and halt on a panic so its stack can be decoded. Keep a successful
   boot running between trials. A pass requires five complete ordered results
   with consistent totals and a final `COUNT 5`.
2. On the candidate, arm a 250 ms timeout, advance the wall clock by one hour
   via `setTime()`, and check `getTime()` and `Date`. Register a rising-edge
   watch on D2 and switch the pin to output with input sensing enabled by the
   ESP32 GPIO backend. Toggle D2 from the pending timeout and compare the
   interrupt event timestamp against the wall clock. Remove the watch and
   restore the prior clock at the end.
3. On the candidate, use the credentialed bench runner to join the local AP.
   Set the time to 1 January 2020 and call `Wifi.setSNTP("pool.ntp.org", "UTC0")`.
   Wait for the wall clock to advance beyond the sentinel epoch and repeat the
   D2 watch check. For a second attempt, bind a UDP listener at port 123 on
   the disconnected board, then call `Wifi.setSNTP("127.0.0.1", "UTC0")`.
   The listener is set up to reply only to loopback requests with a known
   timestamp. Reboot to stop SNTP and clear the runtime Wi-Fi state.

## Results

- The baseline produced six completed ping trials and four trials with a
  malformed callback or incomplete output. None of these ten automated
  trials hit the time-lock abort. The earlier Web IDE trace on the same
  firmware remains the positive observation of the abort.
- The candidate produced two completed trials and eight trials with callback
  or timer failures, including a duplicate result field and a serialization
  exception. The serial runner stalled during recovery after trial six. A
  reset through the USB loader recovered the board, and the final four trials
  ran afterward. No time-lock abort or panic backtrace was captured across
  the ten candidate trials. With zero observed time-lock aborts in the
  automated baseline, this is a limited hardware check of the fix.
- The candidate passed `setTime()`, `Date`, and GPIO timestamp checks. The
  pending 250 ms timeout ran about 269 ms after the clock was set. The rising
  edge was timestamped about 0.61 ms after the pre-toggle clock reading and
  was processed about 3.44 ms after the edge timestamp.
- SNTP remains unverified. The bench-network attempt timed out before
  confirming synchronization. The loopback listener received no request
  during its 16-second observation window.
- The board was left at a clean boot on candidate `896255bff`, with Wi-Fi mode
  `NULL` verified over serial.

## Manual Web IDE follow-up

On 25 November 2026, Simon reported no `gettimeofday()` lock abort after
50 pings using the loopback REPL probe on the patched S3. This is a manual
observation alongside the automated comparison. A separately posted Web IDE
trace from the same build matched ELF prefix `64be2171f7fd96ea` and decoded to
a `LoadStoreError` in `jsvStringIteratorGetChar()` during `JSON.stringify()`
of the ping result. That trace matches the earlier [#2797 callback corruption
signature](2026-11-24-esp32-s3-ping-isolated-webide-reproduction.md#simplified-single-request-panic).

In a further Web IDE run after reset, Simon reported 26 ping executions on
the same patched build. The failing call printed two valid callbacks with
sequence numbers 26 and 27, then reported `OUT OF MEMORY` and rebooted on a
core-0 `LoadStoreError`. The matching ELF resolves PC `0x4201b710` to
`jsvStringIteratorGetChar()` under `JSON.stringify()`. The faulting address
was `0x40000000`, matching the previous patched-build crash. The Web IDE
reported a prompt timeout during reboot, and the transcript ends at a live
`2v29.540` REPL prompt.

Another Web IDE transcript showed four valid callback records with sequence
numbers 11–14. The seven-second timer printed `COUNT 4`, and the REPL also
reported an `InternalError` at `JSON.stringify()`. The Web IDE reported an
upload/prompt recovery warning and eventually displayed a live REPL prompt.
The supplied trace shows incomplete callback delivery and a serialization
error, with no panic, reboot, or time-lock abort visible.

Simon later reported that repeated REPL executions on the patched S3 still
had not produced the `gettimeofday()` lock abort. The number of executions
in this follow-up was unspecified.
