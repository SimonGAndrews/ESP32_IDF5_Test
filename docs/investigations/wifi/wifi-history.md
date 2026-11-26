# ESP32 Wi-Fi investigation history

Snapshot through 25 November 2026.

The ESP32_SGATest bench work established how to distinguish Wi-Fi API callbacks from actual network state. The XFSM greenhouse Wi-Fi manager then exposed the need for explicit ownership and reliable completion of asynchronous firmware operations. The current work resumes at Espruino issue #2797, where a single ping on the S3 demonstrated that native task callbacks can corrupt JavaScript state.

This chronology preserves the findings, upstream decisions and entry points for reproducing them. The [ESP32_SGATest archive](https://github.com/SimonGAndrews/ESP32_SGATest) remains the source of detailed evidence and bench tooling. Issue IDs `WIFI-001` through `WIFI-011` follow the [original issue register][register].

## 20 July 2026 — two-board bench and IP configuration

Classic ESP32 (legacy IDF3) and ESP32-C3 (IDF 4.4.8) acted as target and controlled peer. The probes checked JavaScript callbacks, local AP/station state, peer-observed association and DHCP, and application traffic independently. Wrong-password and unavailable-network trials bounded host waiting and checked that the target gained neither an IP address nor a completed peer join. [Controlled peer baseline][peer] · [Negative cases][negative]

- `WIFI-001` and `WIFI-002`. `Wifi.setAPIP()` applied the requested address but reported failure. On the tested legacy and IDF4 builds, `Wifi.setIP()` reported success while DHCP left the station address unchanged. The investigation distinguished the inverted callback result from the older backend's DHCP-client handling. [IP configuration findings][ip]
- `WIFI-003`. C3-station-to-classic-AP ping received no reply, while the reverse test direction and bidirectional UDP worked. Sender versus echo-responder attribution remained open. [Directional ping][directional]
- `WIFI-004`. Two legacy classic runs reset in follow-up sequences after `setIP()`. The trigger remained unisolated. [Post-setIP resets][postsetip]

## August–September 2026 — IDF5 port and focused fixes

- **31 August–5 September.** A debug-enabled IDF5 build failed because the Wi-Fi event diagnostic used the older callback's event field. [PR #2733](https://github.com/espruino/Espruino/pull/2733) corrected it and entered master in the official IDF5 merge `5d79af218`. Same-source classic testing at `e5341719a` found further radio regressions and kept IDF3 as a comparator. [Debug-build record][debug] · [Post-merge validation][idf5]
- **8–10 September.** The classic IDF5 ping/70 KB candidate passed controlled success, timeout and restart tests. Coexistence trials showed improved native heap margin for BLE with Wi-Fi and HTTPS. Separately, a current-master C3 station produced no ping callback while UDP and inbound ICMP worked. The later C3 scheduling investigation isolated that failure class. [Classic ping and heap qualification][heap] · [C3 peer qualification][c3peer]
- **14–18 September.** `WIFI-005` reproduced on classic IDF5. Starting the station for `Wifi.scan()` also started a connection, aborting the scan. Direct association and peer traffic passed. [Issue #2750](https://github.com/espruino/Espruino/issues/2750) led to [PR #2752](https://github.com/espruino/Espruino/pull/2752), merged as `fdb40abab`. A follow-up on the July IP findings showed that classic IDF5 changed the station address but still reported failure. [Issue #2753](https://github.com/espruino/Espruino/issues/2753) and merged [PR #2754](https://github.com/espruino/Espruino/pull/2754), `7327d8f75`, record the callback and legacy DHCP fixes. [Scan reproduction and fix validation][scan] · [IP configuration qualification][ip]
- **17 September.** The historical classic IDF5 WPA2 AP failure was reproduced at `e5341719a` and isolated to the old board allocation configuration. Three authentication attempts failed with resizable JsVars, and three passed with fixed JsVar allocation at the same 40 KB native reserve. Current master and the integration build with PRs #2749 and #2752 passed authentication, DHCP and UDP. A same-source IDF4 control also passed. [Allocation A/B][wpa-a-b] · [WPA2 follow-up][wpa-followup]
- **25 September.** Espruino `0b4ff45fa` added ping for IDF5. Later analysis traced the direct JavaScript result construction in the native callback back to the older ping implementation in `e53b7a4f5` from December 2018. [IDF5 ping commit](https://github.com/espruino/Espruino/commit/0b4ff45faaaa54b9258a4441521489f19110341c)

## October 2026 — master closeout and ping lifecycle

The classic ESP32 and C3 peer closeout confirmed scan, station transitions, saved configuration, AP recovery, static IP and UDP behaviour on the relevant images. The controlled ping test exposed a valid IPv4 string rejected during parsing, before ICMP could start. Correcting parsing exposed overlap and lifecycle defects. A local combined parser/lifecycle candidate passed the 35-check ping test and 28-session soak. This result qualifies the candidate described in the report. [October closeout][october]

## 2–7 November 2026 — XFSM-driven asynchronous hardening

The XFSM greenhouse soil-heater example needed reliable station lifecycle events and snapshots, alongside completion callbacks for individual requests. Firmware owns validation, native operation lifetime, completion and public event publication. XFSM owns provisioning, credentials, timeout, retry and application recovery. It still reconciles delayed lifecycle events against observed Wi-Fi state. The example is parked in `Espruino-XFSM` at `a892a37`, under `examples/wifi-greenhouse-soil-heater/`. [Lifecycle design][lifecycle]

- **2 November.** C3 probes showed a second connect could replace accepted callback state before validation or acceptance (`WIFI-006`), disconnect could complete twice while connecting (`WIFI-007`), and pending ping could target a discarded JavaScript runtime after reset (`WIFI-008`). These findings established a shared contract: validate before claiming ownership, reject overlap without disturbing active work, complete accepted operations once, and detach callbacks at reset. [Asynchronous probes][async-probes]
- **3 November.** The IDF5 ping hardening candidate `70555065f` passed the classic, C3 and S3 probes for overlap, reset and repeated sessions. The candidate fixed a native-task cleanup problem that had exhausted heap at 24 sessions. IDF4 comparisons separated IPv4 address handling, single-core C3 scheduling and the legacy extra terminal callback. The older S3 IDF4 backend also produced invalid callbacks and resets when a ping outlived its JavaScript runtime. [IDF5 candidate][ping-candidate] · [IDF4 S3 comparison][idf4-s3] · [IDF4 candidate][idf4-candidate]
- **5–7 November.** Local stacked candidates `610007d3f` (connect) and `14cd9932f` (disconnect) exercised request ownership, exactly-once completion, public event delivery, reset/recovery and repeat-use stability across classic, C3 and S3 IDF5. The latter also retained a separate classic IDF3 API regression record. [Connect candidate][connect] · [Disconnect candidate][disconnect] · [Legacy API comparison][legacy-api]

## November submission and consolidated-master restart

- The larger submission was documented in [issue #2779](https://github.com/espruino/Espruino/issues/2779) and [PR #2781](https://github.com/espruino/Espruino/pull/2781), both closed. Maintainer feedback favoured focused current-master reproductions, existing event mechanisms, simple overlap rejection and less static state. The local qualification remains evidence for the design and test cases. [Submission history][ping-restart]
- **18–20 November.** Espruino `9098da2d2` retired IDF3/IDF4 build definitions. The focused restart on consolidated IDF5 identified two independent blockers. For the IPv4 parser, Gordon applied the correction directly in `fe558e4b3`, closing [issue #2790](https://github.com/espruino/Espruino/issues/2790) and [PR #2791](https://github.com/espruino/Espruino/pull/2791). For C3 ping starvation, [issue #2792](https://github.com/espruino/Espruino/issues/2792) led to [PR #2793](https://github.com/espruino/Espruino/pull/2793), merged as `e58f6ca81`, moving ping above the interpreter's priority so it could run on C3. [Focused restart and qualification][ping-restart]
- **20 November.** A three-board baseline used the tree shared by upstream `e58f6ca81` and flashed revision `1b820a413`. It reconfirmed ping overlap and reset, connect overlap and blocked recovery after reset, and duplicate or stale disconnect completion. A single S3 ping corrupted result data (`WIFI-009`), and a sequential C3 test exhausted native heap after 24 sessions (`WIFI-010`). [Consolidated-master runtime baseline][async-baseline] · [Integrated hardening plan][plan]

## 23–25 November 2026 — native results, #2797 and S3 evidence

- **23–24 November.** [Issue #2797](https://github.com/espruino/Espruino/issues/2797) separated the single, non-overlapping S3 ping failure from session overlap. Source review and temporary instrumentation put result construction and queueing in the ESP-IDF `ping` task. The wider station/scan/AP/IP audit found direct JavaScript state access via `sys_evt`. The ping failure has physical corruption evidence. The wider audit establishes task context and the shared ownership risk. IDF4 source and reset evidence show that the unsafe ownership predates IDF5. [Task-context and ownership audit][task-context] · [Isolated S3 trace][s3-original]
- **24 November.** The S3 failure reproduced on non-debug Espruino `0cd5e987e` with ESP-IDF 5.5.3. Matching ping to interpreter priority produced one clean failure in two runs. Restoring default priority 2 produced four callback failures in six runs, with one separate time-lock abort. Matching ping, `sys_evt` and the interpreter at priority 5 and explicitly yielding produced three clear invalid-callback failures in ten runs. These comparisons support moving JavaScript work onto the interpreter task. The implementation of the combined experiment and the complete outcomes are in the archived reports. [Reported results on #2797](https://github.com/espruino/Espruino/issues/2797#issuecomment-7139513147)
- **24 November.** `WIFI-011` is the separate S3 `gettimeofday()`/newlib lock abort. Decode the reboot against the matching ELF and count it separately from malformed results or invalid queued callbacks. One run can exhibit both symptoms. ESP-IDF 5.5.6 was reviewed at source level as a separate qualification option. [Isolated S3 trace][s3-original]
- **25 November.** ESP32_SGATest commit `56f12fb33483704b650859411c51c265d75c48e2` preserves the four S3 test sets, flash/run logs and the combined scheduling diff. The ping-only interpreter-task handoff comparison remains pending.

## Resume from here

Use the sibling `../Espruino` checkout from the `ESP32_IDF5_Test` repository
root for firmware builds and changes. Its `origin` is
`SimonGAndrews/Espruino` and its `upstream` is official `espruino/Espruino`.
Fetch `upstream` and use a fresh branch or worktree from `upstream/master`
for the next experiment. The earlier S3 scheduling experiment used a worktree
linked to `Espruino_master`, and its patch is preserved in the archive.
Use `ESP32_IDF5_Test` for investigation notes, probes and new bench evidence.
`ESP32_SGATest` holds the historical evidence, and `Espruino-XFSM` is the
parked downstream application.

1. Read the latest [#2797 discussion](https://github.com/espruino/Espruino/issues/2797). Gordon proposed the shared I/O buffer as the fallback, with `EV_NETWORK_PENDING`, a common `jsnet_exec_pending()` dispatcher and safe concurrent ESP32 writes. Confirm the first production PR scope in that discussion.
2. Run the controlled S3 comparison on the same upstream revision, IDF, build options and task priorities. Compare the unmodified build with an experiment that copies plain ping results from the native task and constructs JavaScript results on `espruinoTask`. The earlier candidate's handoff commit `99cc0a5d4` is a reference for the experiment. Start each trial from a known clean boot. Use the same saved single-loopback probe and verify five correctly formed callbacks, consistent counts and completion of the seven-second timer. Record queue failures and the time-lock abort independently.
3. After the focused correction, qualify classic ESP32, C3 and S3. Continue in the integrated order of ping overlap and native cleanup, station connect ownership and disconnect completion, reset lifetime, then API and documentation gaps. Carry forward the open legacy anomalies, the debug diagnostic that logs the station password, and questions about `dhcp_timeout`, disconnect-reason mapping, native retries and optional callbacks. Revisit XFSM integration once firmware can provide reliable completions and public events. [Integrated hardening plan][plan]

## Recover the original evidence and probes

The linked historical reports are pinned to ESP32_SGATest revision `93a0b7f0b0875e61915fa264ea20b040079e3301`. The newer handover and S3 command supplement are at `632998c39efa6d8d6f4a6006e78ca03f1a9ee8cb` in `docs/handoff/`. The four S3 reports below are from local archive commit `56f12fb`. Use a checkout containing that commit to retrieve them.

| Evidence | Path in ESP32_SGATest |
| --- | --- |
| Original handover and command supplement | `docs/handoff/2026-11-24-esp32-wifi-hardening-issue-2797.md` and `docs/handoff/2026-11-24-esp32-s3-reproduction-command-supplement.md` |
| Non-debug baseline | `tests/Results/WIFI_BLE_Results/2026-11-24-esp32-s3-nondebug-current-master/README.md` |
| Priority comparisons | `tests/Results/WIFI_BLE_Results/2026-11-24-esp32-s3-ping-priority5-ab/README.md` and `tests/Results/WIFI_BLE_Results/2026-11-24-esp32-s3-ping-default-priority-ab/README.md` |
| Combined scheduling candidate and exact diff | `tests/Results/WIFI_BLE_Results/2026-11-24-esp32-s3-gordon-scheduling-ab/` |
| Active probe and host runner | `docs/investigations/wifi/repros/esp32_wifi_ping_single_handoff.js` and `tools/repl/run_test.py` |

For example, from the original archive checkout:

```bash
git show 56f12fb:tests/Results/WIFI_BLE_Results/2026-11-24-esp32-s3-gordon-scheduling-ab/README.md
```

For the S3 comparison, use the command supplement's native USB flashing procedure with the locally detected board port. Use `BOARD=ESP32S3 RELEASE=1` after `unset DEBUG` and provisioning the recorded ESP-IDF version. Run the saved loopback probe through the Python runner with `pyserial` installed. The runner resets JavaScript before upload. Arrange a board reset separately when the comparison calls for a fresh boot.

For any historical rerun, use the relevant report to recover the target and peer roles, board definitions, firmware revisions and source changes, IDF and build flags, harness/serial setup, and test script or runner configuration. Record the new observation with its provenance so a later thread can distinguish the rerun from the original.

[register]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/wifi/README.md
[peer]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-07-20-wifi-supervisor-peer-initial.md
[negative]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-07-20-wifi-supervisor-peer-negative-cases.md
[ip]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/wifi/ip-configuration-api-2026-07-20.md
[directional]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/wifi/directional-ping-asymmetry-2026-07-20.md
[postsetip]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/wifi/classic-post-setip-reset-2026-07-20.md
[debug]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/IDF5_port_testing/2026-08-31-pr1-wifi-debug-build.md
[idf5]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/handoff/2026-09-04-idf5-classic-validation-status.md
[heap]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-09-08-esp32-idf5-ping-70k-validation.md
[c3peer]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-09-10-esp32c3-current-master-peer-qualification.md
[scan]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/wifi/esp32-idf5-scan-empty-2026-09-14.md
[wpa-a-b]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-09-17-esp32-idf5-historical-wpa2-allocation-cause.md
[wpa-followup]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-09-17-esp32-idf5-wpa2-ap-follow-up.md
[october]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/2026-10-13-esp32-idf5-current-master-closeout.md
[lifecycle]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/wifi/async-operation-lifecycle-2026-11-02.md
[async-probes]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-11-02-esp32-wifi-async-operation-probes.md
[ping-candidate]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-11-03-esp32-idf5-ping-hardening-candidate.md
[idf4-s3]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-11-03-esp32s3-idf4-ping-characterisation.md
[idf4-candidate]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-11-03-esp32-idf4-basic-ping-correction.md
[connect]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-11-05-esp32-idf5-wifi-connect-hardening-candidate.md
[disconnect]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-11-07-esp32-idf5-wifi-disconnect-hardening-candidate.md
[legacy-api]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-11-07-esp32-classic-idf3-wifi-api-regression.md
[ping-restart]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/wifi/ping-current-master/README.md
[async-baseline]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-11-20-esp32-current-master-wifi-async-baseline.md
[plan]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/wifi/current-master-hardening-plan-2026-11-20.md
[task-context]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/docs/investigations/wifi/esp32-network-native-event-handoff-2026-11-23.md
[s3-original]: https://github.com/SimonGAndrews/ESP32_SGATest/blob/93a0b7f/tests/Results/WIFI_BLE_Results/2026-11-24-esp32-s3-ping-isolated-webide-reproduction.md
