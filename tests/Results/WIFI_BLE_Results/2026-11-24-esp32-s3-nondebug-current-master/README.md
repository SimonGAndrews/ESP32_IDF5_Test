# ESP32-S3 non-debug current-master ping handoff reproduction

Date: 2026-11-24

## Conclusion

The first run of the retained single-session `Wifi.ping()` handoff probe
reproduced the issue #2797 failure class on an exact non-debug build of current
Espruino `master`.

The ESP32-S3 accepted one loopback ping request. Callback records one and two
were valid. While callback three attempted to serialize its result, the
interpreter reported `OUT OF MEMORY`, `MEMORY` and
`InternalError: Variable type cannot be converted to string`. Callback records
four and five were subsequently valid, but the seven-second completion timer
did not run, `DONE=` was absent and the runner returned 2.

This closes the debug-timing evidence gap recorded in issue #2797. The unsafe
handoff is physically observable without `-DDEBUG`; it is not dependent on the
diagnostic `espruino_ping_event` output present in the earlier retained image.
It does not select the correction transport or first-PR scope.

No firmware source was changed for this reproduction. The separate S3
`gettimeofday()` time-lock abort did not occur and is not part of this result.

## Source and build identity

| Item | Value |
|---|---|
| Firmware repository | linked worktree of `/home/simon/MaBecker/Espruino_master` |
| Source revision | `0cd5e987e9f2cace0ad7a532c6afebab907175fe` |
| Source state | detached exact `upstream/master`, clean |
| Board definition | `ESP32S3` |
| Espruino version | `2v29.536` |
| ESP-IDF | 5.5.3 |
| Build mode | `RELEASE=1`, with `DEBUG` absent from the environment |
| Generated defines | included `-DNO_ASSERT -DRELEASE`; did not include `-DDEBUG` |

At the start of the work, upstream `master` was one commit beyond the earlier
`4addf143c` evidence baseline. The intervening commit changes
`src/jsinteractive.c` packet-transfer terminal display handling; it does not
change the ESP32 network wrapper.

The build used a separate linked worktree so that cleaning it did not destroy
the retained debug-enabled artifacts in the primary Espruino checkout:

```bash
cd /home/simon/MaBecker/Espruino_master_2797_nondebug
unset DEBUG
source scripts/provision.sh ESP32S3
make BOARD=ESP32S3 clean
make BOARD=ESP32S3 RELEASE=1 -j2
```

| Artifact | SHA-256 |
|---|---|
| `bin/build/espruino.elf` | `f7841487a72459bbaadbff272a66143945a999e39753116dd44fea1e268bc7b3` |
| `bin/build/espruino.bin` | `8e7c837cd17d561cee0235549c8137c626c07b5b05e880976814cef16fb83a81` |
| `bin/espruino_2v29.536_esp32s3.bin` | `8e7c837cd17d561cee0235549c8137c626c07b5b05e880976814cef16fb83a81` |

## Target and flash

| Item | Value |
|---|---|
| Target | ESP32-S3 QFN56 revision 0.1, embedded 8 MB PSRAM |
| Target MAC | `68:b6:b3:27:2c:40` |
| Console/flash transport | native USB Serial/JTAG |
| Stable serial path | `/dev/serial/by-id/usb-Espressif_USB_JTAG_serial_debug_unit_68:B6:B3:27:2C:40-if00` |

The exact raw-`esptool` procedure from the command supplement was used at
460800 baud with the generated `@flash_args`. The bootloader, application and
partition-table writes all passed hash verification, and `esptool` completed
with a hard reset through RTS. The complete transcript is retained as
[`flash.log`](flash.log).

## Reproducer and runner

| File | SHA-256 |
|---|---|
| `docs/investigations/wifi/repros/esp32_wifi_ping_single_handoff.js` | `d488965b34a26d8d2414c4f031f75e8bfeab7ced882aba3ea8dec9ffde2ca94c` |
| `tools/repl/run_test.py` | `20c2326c7e129116c3c77d2a576e8f2b7638b3fd760a7999ce563c980e81b4c2` |

The run used the documented direct-pyserial procedure:

```bash
python3 tools/repl/run_test.py \
  docs/investigations/wifi/repros/esp32_wifi_ping_single_handoff.js \
  --port /dev/serial/by-id/usb-Espressif_USB_JTAG_serial_debug_unit_68:B6:B3:27:2C:40-if00 \
  --timeout 15 \
  --show-raw
```

The runner queried and reported the flashed identity before uploading the
probe:

```text
RUNNER board=ESP32S3
RUNNER version=2v29.536
RUNNER git_commit=0cd5e987e
```

It then recorded two valid callbacks, followed by:

```text
OUT OF MEMORYExecution Interrupted
New interpreter error: MEMORY
Uncaught InternalError: Variable type cannot be converted to string
```

The callback increments its index before validating and serializing its
record. The later visible indices four and five therefore show that callback
three entered JavaScript but failed while its record was being serialized.
The result is a single accepted, non-overlapping session; no second ping call
was made.

The complete runner stream is retained as
[`s3-single-handoff-run-01.log`](s3-single-handoff-run-01.log). Its SHA-256 is
`0c730405641bcf966a671570c6eaebd4553b5b3c88b11c6563b02a02eed425ba`.
The flash-log SHA-256 is
`779ab86f7b5160e99223a0ab88f37b52ccaf1e0e047d7de7ed4b8d654a3951bd`.

## Disposition

The requested non-debug confirmation is complete. No additional repetition is
required to establish that the failure exists without debug instrumentation;
clean repeats would not invalidate this timing-sensitive capture.

Firmware implementation remains gated on maintainer direction in issue #2797
about interpreter ownership, transport and first-PR scope. Once a focused
correction exists, qualify its smallest reproduction and recovery behaviour on
classic ESP32, ESP32-C3 and ESP32-S3, while keeping the time-lock abort as a
separate investigation.
