# Repository instructions for Codex

- Build reusable hardware test rigs and repeatable tests for Espruino on ESP32-family boards. Follow the project scope and hardware test workflow in `README.md`.
- Use the sibling checkout at `../Espruino` for firmware source investigations. Use its `master` branch as the baseline for fixes and pull requests to `espruino/Espruino`.
- When documenting a hardware test, record the board, firmware revision, test setup, steps, and observed result so the run can be reproduced. If a test could not be run on hardware, record that status.

## Repository layout

- `docs/` holds harness designs, target wiring and bring-up guides, and handover notes for continuing work.
- `docs/investigations/` records specific firmware issues, including observed symptoms, bench evidence, and conclusions.
- `tests/` contains JavaScript tests for running Espruino on real hardware and saved results for comparing firmware builds.
- `tools/` contains Python runners and utilities that connect to boards, run wiring and functional checks, and capture serial output.
