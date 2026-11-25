# ESP32-S3 ping priority-5 A/B

Date: 2026-11-24

## Conclusion

Matching the ESP-IDF ping task to the priority-5 Espruino interpreter does not
resolve issue #2797. The first non-overlapping single-session run passed, but
the second delivered no JavaScript callbacks and reported four invalid event-
queue callback diagnostics before the completion timer produced `DONE=FAIL`.

This tests the least-invasive scheduling change requested by Gordon Williams
after issue #2797 was opened. Equal priority changes the timing relationship;
it does not establish exclusive ownership of Espruino `JsVar` memory or the
interpreter event queue. It is therefore not a reliable production correction.

No queue or native-to-interpreter handoff was implemented. The separate S3
`gettimeofday()` time-lock abort did not occur in these runs.

## Maintainer-requested comparison

Gordon asked whether the observable failure was a side effect of Espruino
commit `1b820a413`, which raised the ESP-IDF ping task from its default priority
2 to priority 6, above the priority-5 interpreter. He proposed first trying
ping at the same priority as the interpreter before adding a handoff queue.

The production baseline and candidate were therefore:

| Variant | Source | Ping priority | Result |
|---|---|---:|---|
| current master | exact `0cd5e987e` | 6 | first retained non-debug run reproduced interpreter memory/string-conversion failure and missed `DONE` |
| scheduling candidate | `0cd5e987e` plus the recorded two-line diff | 5 | run 1 passed; run 2 delivered zero callbacks and `DONE=FAIL` |

Priority 2 was subsequently retested on the same S3 and failed in four of six
single-session runs; one further run hit the separately tracked time-lock
abort. That later result is recorded in
[`../2026-11-24-esp32-s3-ping-default-priority-ab/README.md`](../2026-11-24-esp32-s3-ping-default-priority-ab/README.md).
Earlier current-master evidence behind Espruino issue #2792 also established
that priority 2 can starve ping execution on the single-core ESP32-C3.

## Source and build identity

| Item | Value |
|---|---|
| Firmware repository | linked worktree of `/home/simon/MaBecker/Espruino_master` |
| Investigation branch | `investigate/esp32-ping-priority-2797-2026-11-24` |
| Base revision | `0cd5e987e9f2cace0ad7a532c6afebab907175fe` |
| Board definition | `ESP32S3` |
| Espruino version | `2v29.536` |
| ESP-IDF | 5.5.3 |
| Build mode | `RELEASE=1`, with `DEBUG` absent from the environment |

The complete source difference was:

```diff
-  // Run above Espruino's priority-5 interpreter on single-core targets.
-  ping_config.task_prio = 6;
+  // Match Espruino's priority-5 interpreter for the issue #2797 A/B test.
+  ping_config.task_prio = 5;
```

The candidate remained an uncommitted investigation diff so it could not be
mistaken for an accepted correction. Its exact source difference and binary
hashes are part of this evidence.

| Artifact | SHA-256 |
|---|---|
| `bin/build/espruino.elf` | `2e55582e2e73387c1eda40f4bdefbf25e968d6a6b4cd5d261182cc7342d07b86` |
| `bin/build/espruino.bin` | `64e9a9e0420e34c42ee68d5dfb3ae014a0981e1eb8af19c0a4711c93fee770dc` |
| `bin/espruino_2v29.536_esp32s3.bin` | `64e9a9e0420e34c42ee68d5dfb3ae014a0981e1eb8af19c0a4711c93fee770dc` |

The S3 target, stable serial path, raw-`esptool` command, runner and test file
were unchanged from the exact non-debug current-master reproduction. All flash
writes passed hash verification and completed with an RTS hard reset. See
[`flash-priority5.log`](flash-priority5.log).

## Results

### Run 1

The first invocation delivered five complete result objects and ended with:

```text
METRIC callback_count=5
METRIC invalid_result_count=0
PASS single_ping_result_handoff
DONE=PASS
```

This clean run is consistent with the known timing sensitivity and does not
establish safety.

### Run 2

The second invocation accepted one loopback request but delivered no JavaScript
callback. Its raw stream contained:

```text
INFO call=accepted
ERROR: Unknown type of callback in Event Queue
ERROR: Unknown type of callback in Event Queue
ERROR: Unknown type of callback in Event Queue
ERROR: Unknown type of callback in Event Queue
METRIC callback_count=0
METRIC invalid_result_count=0
FAIL single_ping_result_handoff callbacks=0 invalid=0
DONE=FAIL
```

The runner returned 1 because the probe completed with one failing check. No
second ping request overlapped this session. This result is consistent with
the unsafe native-task event-queue access identified by issue #2797, but it
does not claim which individual event object or callback reference was
corrupted.

## Retained logs

| Log | SHA-256 |
|---|---|
| `flash-priority5.log` | `8f41473114d48211a2186c6e9793e045da46164ddde1c879c3d4e4aa31243493` |
| `s3-priority5-run-01.log` | `fe8a61fbab41d2435c8349f91fee4a2d111cf43c8f5f6bc8cb453456b07069af` |
| `s3-priority5-run-02.log` | `64a8e15858874e016a178829ab651d305282a6e0e3acbba943ae81154937486a` |

The logs contain no SSID or password because the test uses the local loopback
address.

## ESP-IDF 5.5.6 check

ESP-IDF 5.5.6 is available as a later bugfix release, but the source comparison
from 5.5.3 to 5.5.6 contains no change under `components/lwip/apps/ping/`.
There are event-loop deletion/concurrency changes under `components/esp_event/`
and broader FreeRTOS/newlib changes, but none moves Espruino `JsVar` work out of
the native ping or `sys_evt` callbacks. An IDF update may be qualified as a
separate firmware-line change; it is not a source-level resolution of #2797.

## Disposition

The priority-only path is rejected by physical evidence. Gordon's conditional
fallback is now the relevant design direction: use the shared IO-event buffer,
add a network-specific pending event such as `EV_NETWORK_PENDING`, and keep a
common `jsnet_exec_pending` dispatcher outside the ESP32-specific backend as
far as practical.

Before production implementation, confirm the first-PR scope: establish the
common transport with ping as its first producer, or move all audited ESP32
Wi-Fi/IP event paths together. If the shared IO buffer is selected, its ESP32
multi-producer writes must also be made safe with an appropriate FreeRTOS
primitive.
