# ESP32-S3 Wi-Fi smoke test on `ed5daf244`

On 25 November 2026, station connection, loopback ping, and standalone scan
passed on the Wi-Fi hardening branch baseline. Two disconnect paths showed
different failures. The scan-after-disconnect sequence stalled before scanning,
and an isolated deliberate disconnect completed with a duplicate callback.

## Setup

- ESP32-S3 QFN56 revision 0.1, firmware `ESP32S3` / `2v29.539` / `ed5daf244`,
  branch `fix/esp32-idf5-wifi-hardening`, ESP-IDF 5.5.3 release build.
  The REPL reported the same identity before both sets of probes. See the
  [build and flash record](2026-11-25-esp32s3-wifi-hardening-build-flash.md).
- Native USB Serial/JTAG at 115200 baud. The stable port
  `/dev/serial/by-id/usb-Espressif_USB_JTAG_serial_debug_unit_68:B6:B3:27:2C:40-if00`
  resolved to `/dev/ttyACM0`. The board was USB-powered, with no peer board
  or harness jumpers. The credentialed fixtures used the local 2.4 GHz bench AP
  and the ignored `tests/WIFI_BLE/local_wifi_credentials.json` file.

## Observations in run order

| Fixture | Result | Board observation |
| --- | --- | --- |
| `wifi_ping_legacy_smoke.js` | Pass | Loopback returned five callbacks with `bytes=64` and `error=0`. |
| `wifi_connect_legacy_smoke.js` | Pass | The connect callback reported success once and the station acquired an IP address. |
| `esp32_wifi_scan_after_disconnect.js` | Fail | Initial connection completed. The fixture timed out in the disconnect phase waiting for the public `disconnected` event, so the scan was never called. A subsequent cleanup disconnect still left `getStatus().station` reporting `connected` and `getIP().ip` nonzero. |
| `esp32_wifi_scan_empty.js` | Pass | After a hardware reboot, the preparation scan and repeat scan each returned five AP records. |
| `esp32_wifi_disconnect_connected.js` | Fail | After a fresh hardware reboot and one successful connection callback, deliberate disconnect delivered two completion callbacks and one public `disconnected` event. At the end of the observation window the station reported `ASSOC_LEAVE` and had no IP. The historical runner exited 0 because the fixture records counts without asserting exactly-once behavior. |

After the final `ESP32.reboot()`, the REPL reported station `UNINITIALIZED`,
mode `NULL`, IP `0.0.0.0`, and `typeof global.WIFI_TEST_CONFIG` as `undefined`.
The stalled phase was assessed from the board API and its events. There was
no peer-side observation of association state in this run.

## Reproduce

Run these commands from the hardware test repository. Set `s3_port` to the
stable port above and `repros` to `docs/investigations/wifi/repros` in an
ESP32_SGATest checkout at `93a0b7f0b0875e61915fa264ea20b040079e3301`.

```sh
python3 tools/repl/run_test.py tests/WIFI_BLE/wifi/wifi_ping_legacy_smoke.js \
  --port "$s3_port" --timeout 18
python3 tools/repl/run_wifi_local_test.py tests/WIFI_BLE/wifi/wifi_connect_legacy_smoke.js \
  --port "$s3_port" --timeout 38
python3 tools/repl/run_wifi_scan_stateful.py --script "$repros/esp32_wifi_scan_after_disconnect.js" \
  --port "$s3_port" --startup-timeout 0 --timeout 40
```

Reboot the ESP32 with `ESP32.reboot()` before each isolated probe. For the
standalone scan, call `run_direct()` from `tools/repl/run_test.py` with
`$repros/esp32_wifi_scan_empty.js`, the serial port, 115200 baud, and a
35-second timeout. Check both scan count markers and the completion marker.
For the disconnect observation, use `run_wifi_local_test.py` with
`$repros/esp32_wifi_disconnect_connected.js` and a 38-second timeout. Check
the callback and event counts in `METRIC deliberate_disconnect` even if the
runner reports success. Filter scan records and disconnect event details from
any shared logs to avoid recording local AP identifiers. Reboot again and
verify the station and IP state above.

The archived fixtures are available at the pinned [ESP32_SGATest revision](https://github.com/SimonGAndrews/ESP32_SGATest/tree/93a0b7f0b0875e61915fa264ea20b040079e3301/docs/investigations/wifi/repros).
