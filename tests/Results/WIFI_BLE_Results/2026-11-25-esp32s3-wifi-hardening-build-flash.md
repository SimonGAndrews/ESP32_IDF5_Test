# ESP32-S3 Wi-Fi hardening branch build and flash

On 25 November 2026, the connected ESP32-S3 booted Espruino `2v29.539` and
reported commit `ed5daf244` over USB serial. This run checked provisioning,
compilation, flash verification and REPL startup. Wi-Fi functionality was not
tested in this run.

## Setup

- Firmware checkout `../Espruino`, branch `fix/esp32-idf5-wifi-hardening`, clean
  source at `ed5daf244d0e396bb4689f3f57e2736dc093e1f7`.
- Board definition `ESP32S3`. The target was an ESP32-S3 QFN56 revision 0.1
  with 8 MB embedded PSRAM and MAC `68:b6:b3:27:2c:40`, powered and connected
  over native USB Serial/JTAG. The stable device path resolved to `/dev/ttyACM0`.
- ESP-IDF `v5.5.3` at `2c211b236707889e8400c4dc5644dd5c4ee071e0`, using the
  installed toolchain from the earlier bench runs. The build resolved
  `espressif/mdns` version `1.15.1`.

## Procedure and observations

1. From the Espruino checkout, unset `DEBUG`, source the installed IDF
   `export.sh`, export `ESP_IDF_PATH="$IDF_PATH"`, and run
   `make BOARD=ESP32S3 RELEASE=1 -j2`. The build completed and produced
   `bin/espruino_2v29.539_esp32s3.tgz`. The application image was 1,554,784
   bytes with 24% of the app partition free. IDF reported Kconfig and
   deprecation warnings during the build.
2. From `bin/build`, flash over the board's native USB port using the
   generated `flash_args` and the provisioned ESP-IDF Python environment.

   ```sh
   s3_port=/dev/serial/by-id/usb-Espressif_USB_JTAG_serial_debug_unit_68:B6:B3:27:2C:40-if00
   python -m esptool --chip esp32s3 --port "$s3_port" -b 460800 \
     --before default_reset --after hard_reset write_flash @flash_args
   ```

   `esptool` 4.12.0 reported successful hash verification for the bootloader at
   `0x0`, app at `0x10000`, and partition table at `0x8000`, then hard-reset
   the board.
3. Open the same port at 115200 baud and query `process.env.BOARD`,
   `process.version`, and `process.env.GIT_COMMIT` using `collect_metadata()`
   from `tools/repl/run_test.py`. The REPL returned
   `{"board":"ESP32S3","version":"2v29.539","git_commit":"ed5daf244"}`.

| Build artifact in `../Espruino` | SHA-256 |
| --- | --- |
| `bin/build/espruino.bin` | `6de1d0aec7593b39c44ca553d7718518f6a032c1de158399afe4982ce418b537` |
| `bin/build/espruino.elf` | `7e8c3263ee2f383620ab0f5c6045a34fa1f87fb547fc13b8f93b8800e5620598` |
