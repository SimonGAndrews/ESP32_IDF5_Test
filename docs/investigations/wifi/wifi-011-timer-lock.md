# WIFI-011 — ESP32-S3 abort while reading the system clock

## What happens

An ESP32-S3 running Espruino `2v29.539` (`ed5daf244`, ESP-IDF 5.5.3) can
reboot during repeated `Wifi.ping("127.0.0.1")` calls. In the captured run,
two callbacks printed valid results before the board aborted. The backtrace
identifies a clock read in the interpreter's idle loop. The ESP-IDF library
needed a mutex at a point where the interpreter was unable to wait for it.
[S3 bench record](../../../tests/Results/WIFI_BLE_Results/2026-11-25-esp32s3-wifi-011-time-lock.md)

## Why the clock read can abort

`jshGetSystemTime()` reads the current time and returns it in microseconds.
The interpreter's idle loop, `jsiIdle()`, uses that value to decide whether
any JavaScript timers are due. A GPIO interrupt handler uses the same clock
function to timestamp a pin change. In the original ESP32 code,
`jshGetSystemTime()` always enters a critical section before calling
`gettimeofday()`.
[Espruino clock helper](https://github.com/espruino/Espruino/blob/ed5daf244d0e396bb4689f3f57e2736dc093e1f7/targets/esp32/jshardware.c#L700-L710)

The original crash trace showed the interpreter's side of the failure. Two
controlled tests on the S3 used that same Espruino baseline with an
instrumented ESP-IDF 5.5.3 to show what happens when the native `ping` task
owns the mutex.

1. The native `ping` task calls C `gettimeofday()`. Newlib, the C library
   used by ESP-IDF, takes `s_time_lock` to read the adjusted boot time. In
   stock IDF, the ping task makes one clock read before waiting for the reply
   (`ping_sock.c:172`) and another after the receive returns (line 174). Each
   controlled test inserted a 250 ms delay while one of those calls held the
   mutex.
   [Ping timing source](https://github.com/espressif/esp-idf/blob/v5.5.3/components/lwip/apps/ping/ping_sock.c#L169-L175) · [Newlib clock read](https://github.com/espressif/esp-idf/blob/v5.5.3/components/newlib/src/time.c#L89-L96)
2. While `ping` holds `s_time_lock`, `espruinoTask` calls
   `jshGetSystemTime()` from `jsiIdle()`. The helper takes Espruino's
   `JSmicrosMux` spinlock and masks interrupts on its core. These are two
   separate locks, so the interpreter can enter its critical section while
   ping holds the time mutex. [ESP-IDF critical-section rules](https://docs.espressif.com/projects/esp-idf/en/v5.5.3/esp32s3/api-reference/system/freertos_idf.html#restrictions-and-considerations)
3. Still inside the critical section, the interpreter calls
   `gettimeofday()` and newlib tries to take `s_time_lock`. Newlib checks
   `xPortCanYield()` and sees that it cannot wait. The immediate attempt
   to take the mutex fails, so newlib calls `abort()`.
   [Newlib mutex handling](https://github.com/espressif/esp-idf/blob/v5.5.3/components/newlib/src/locks.c#L141-L170)

At the failed acquisition, the diagnostic captured the owner before
FreeRTOS released the lock protecting its mutex bookkeeping. Both runs
reported `waiter=espruinoTask` and `owner=ping`. The instrumented source line
was 182 for the read before receive and 185 for the read after receive. Both
panics decoded to the same interpreter stack as the original crash.

```text
espruinoTask → jsiLoop → jsiIdle → jshGetSystemTime
             → gettimeofday → _gettimeofday_r → get_adjusted_boot_time
             → _lock_acquire → lock_acquire_generic → abort
```

The controlled tests confirm that either native ping clock read can cause
the abort if it still owns the mutex when the interpreter calls
`gettimeofday()`. Ten ordinary loopback sessions with the diagnostic delay
disabled finished without a lock abort. The owner in the original Web IDE
crash remains unconfirmed. [Diagnostic bench record](../../../tests/Results/WIFI_BLE_Results/2026-11-25-esp32s3-wifi-011-lock-owner-diagnostic.md) · [ESP-IDF diagnostic patch](wifi-011-lock-owner-diagnostic.patch)

## Reproduce and identify it

Use an ESP32-S3 with baseline Espruino `ed5daf244`, built with ESP-IDF 5.5.3
using `BOARD=ESP32S3 RELEASE=1` and `DEBUG` unset. Connect the Web IDE over
native USB Serial/JTAG. Run this loopback probe, waiting for `COUNT` or a
reboot before the next attempt. A healthy call prints five valid callbacks
and `COUNT 5`. The loopback address needs no access point.

```javascript
echo(false);
var count = 0;
require("Wifi").ping("127.0.0.1", function (result) {
  print("CALLBACK " + (++count) + " " + JSON.stringify(result));
});
setTimeout(function () {
  print("COUNT " + count);
}, 7000);
```

Use the reported ELF hash to identify the matching build, then decode any
reboot backtrace. The baseline time-lock trace reported ELF hash prefix
`7e8c3263ee2f3836`. Malformed callback data, `JSON.stringify()` errors, and
the string-iterator `LoadStoreError` seen during ping are tracked under
[#2797](https://github.com/espruino/Espruino/issues/2797). A reboot alone does
not identify WIFI-011.

## What the standalone patch changes

The candidate keeps `gettimeofday()` in a task context that can wait for
the mutex. Interrupt and other non-yielding callers use
`esp_timer_get_time()` plus a cached offset so their timestamps stay on
Espruino's wall-clock time scale. The offset is refreshed at startup, during
ordinary clock reads, after `setTime()`, and on SNTP synchronization. The
critical section protects access to the offset and contains no wall-clock
call. [ESP Timer documentation](https://docs.espressif.com/projects/esp-idf/en/v5.5.3/esp32s3/api-reference/system/esp_timer.html#obtaining-current-time) · [Candidate patch](0001-ESP32-avoid-blocking-time-reads-in-interrupt-context.patch)

The patched S3 passed `setTime()`, `Date`, a pending timeout, and a GPIO
watch timestamp check. The time-lock abort was absent in the ten automated
trials on each build, and in 50 manually reported pings on the patched build.
The automated comparison therefore lacks a baseline abort for a direct
A/B check. SNTP remains unverified. [Bench results and Web IDE follow-up](../../../tests/Results/WIFI_BLE_Results/2026-11-25-esp32s3-wifi-011-time-lock.md)

## ESP32-C3 follow-up

On 26 November 2026, Simon ran the loopback probe repeatedly on an ESP32-C3
in the REPL. The probe printed each `Wifi.ping("127.0.0.1")` result via
`JSON.stringify()` and printed `COUNT` after four seconds. After 130 ping
events, he reported no time-lock abort or callback corruption. The firmware
identity checked afterward over USB on the connected C3 was Espruino
`2v29.525`, commit `1b820a413`, with ESP-IDF `v5.5.3`. That commit has the
same source tree as the upstream ping-priority merge `e58f6ca81`. Official
`master` was at `ed5daf244` when checked on 26 November 2026.
