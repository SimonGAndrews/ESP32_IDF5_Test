# ESP32-S3 dual-core versus single-core ping comparison

On 26 November 2026, the dual-core control reproduced the ping-result
corruption and interpreter crash tracked in [#2797](https://github.com/espruino/Espruino/issues/2797).
The single-core probe reached 200 pings without observed callback corruption
before a native ping task allocation failure.

## Dual-core control

- Board definition `ESP32S3`, tested manually in the Web IDE over USB serial.
- Espruino `2v29.539`, source `ed5daf244d0e396bb4689f3f57e2736dc093e1f7`,
  branch `experiment/esp32s3-unicore`. The generated `sdkconfig` contains
  `# CONFIG_FREERTOS_UNICORE is not set`.
- The standard provisioning script set up ESP-IDF `v5.5.3`
  (`2c211b236707889e8400c4dc5644dd5c4ee071e0`). Simon reported successful
  `make BOARD=ESP32S3 RELEASE=1 BINDIR=bin_s3_dual clean` and
  `make BOARD=ESP32S3 RELEASE=1 BINDIR=bin_s3_dual -j2 flash` runs.
- The panic reported ELF hash prefix `391549ca0154914a`, matching the
  SHA-256 of `bin_s3_dual/build/espruino.elf`,
  `391549ca0154914a171e65c1d84f5c5991d97de3615fc6228271158433243d6d`.

Simon ran the following in the Web IDE REPL, with a four-second timer.

```javascript
echo(false);
var count = 0;
require("Wifi").ping("127.0.0.1", function (result) {
  print("CALLBACK " + (++count) + " " + JSON.stringify(result));
});
setTimeout(function () {
  print("COUNT " + count);
}, 4000);
```

Callbacks 1 and 2 returned valid objects. Callback 3 returned the number
`9223372036848143360`. Callback 4 returned a valid object. The interpreter
then printed `OUT OF MEMORY` at `JSON.stringify()` and rebooted after a
core-0 `LoadStoreError`. The transcript contains no `COUNT` or time-lock
abort. The reported PC `0x4201b654` decoded to
`jsvStringIteratorGetChar()` at `src/jsvariterator.h:92` and the faulting
address was `0x40000000`. The matching ELF places this read in
`JSON.stringify()` under the interpreter's event callback execution.

An independent direct-USB run on the same dual-core image stopped after
11 trials. Five completed successfully, three returned malformed results,
and two had incomplete output. Trial 11 rebooted with ELF prefix
`391549ca0154914a`. Its decoded stack runs from `jsiIdle()` through
`jshGetSystemTime()` and `get_adjusted_boot_time()` into newlib's
`lock_acquire_generic()`, matching the [WIFI-011 time-lock abort](../../../docs/investigations/wifi/wifi-011-timer-lock.md).

## Single-core comparison

- Simon built and flashed the same S3 using the same baseline, ESP-IDF,
  and release flags with `BINDIR=bin_s3_unicore`. The generated `sdkconfig`
  contains `CONFIG_FREERTOS_UNICORE=y`, and the generated C header sets
  `CONFIG_FREERTOS_NUMBER_OF_CORES` to `1`.
- SHA-256 of `bin_s3_unicore/build/espruino.elf` was
  `d0b025bb7923c344853db7e40e309f5834391e666d9c1d92555ad00b5932fb21`.
- Simon reported 200 pings without callback corruption using the same
  four-second Web IDE probe. The next reported result was
  `ping_sock: esp_ping_new_session(228): create ping task failed`, then
  `Uncaught Error: Failed to initialize ping session`. The timer reached
  `COUNT 0`, and the prompt returned. No malformed callback, time-lock abort,
  or reboot appears in the supplied failure excerpt.

ESP-IDF returns `ESP_ERR_NO_MEM` on this `xTaskCreate()` failure path, before
the new session can start. This matches the failure class previously seen in
the C3 soak. That run measured native heap depletion and identified delayed
reclamation of self-deleted ping tasks as a plausible mechanism on a busy
single-core target. The S3 excerpt has no heap or task-reclamation measurements.
[ESP-IDF session creation](https://github.com/espressif/esp-idf/blob/v5.5.3/components/lwip/apps/ping/ping_sock.c#L212-L228) · [C3 soak evidence](2026-11-20-esp32-current-master-wifi-async-baseline.md#sequential-c3-sessions-exhaust-native-heap) · [Prior cleanup analysis](2026-11-03-esp32-idf5-ping-hardening-candidate.md#repeated-session-and-heap-qualification)
