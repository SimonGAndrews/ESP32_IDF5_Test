# ESP32_IDF5_Test

## Purpose

- Design and build reusable hardware test rigs for Espruino on ESP32-family boards.
- Run repeatable tests on real hardware to compare firmware versions, diagnose faults, and document findings for upstream fixes.

## Espruino source checkout

Use the sibling checkout at `../Espruino` for source code investigations. Its `master` branch is the baseline for developing fixes and preparing pull requests for `espruino/Espruino`.

## Hardware test workflow

1. **Develop and flash.** Build baseline firmware and firmware containing candidate fixes in the Espruino source checkout. Identify the connected ESP32 board via USB serial, flash it with the board-specific tools, and verify the version and commit of the firmware running on the board.
2. **Test and debug.** Write reusable JavaScript tests and use Python runners to execute them through the Espruino REPL on physical boards. Compare firmware builds under the same harness conditions. Save serial logs, build details, and jumper settings to help distinguish firmware regressions from wiring or runner problems.
