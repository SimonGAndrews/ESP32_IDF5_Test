# ESP32-S3 ping default-priority A/B

Date: 2026-11-24

## Conclusion

Restoring the exact pre-`1b820a413` ESP-IDF ping-task priority does not resolve
Espruino issue #2797 on current IDF5 source. With the explicit priority-6
override removed, ESP-IDF used its default priority 2. One of six
non-overlapping single-session runs passed; four delivered invalid event-queue
callback diagnostics, and one hit the separately tracked S3 `gettimeofday()`
time-lock abort.

The four issue-#2797 failures accepted one loopback request but did not deliver
a valid JavaScript result. Three completed with zero callbacks and four
`Unknown type of callback in Event Queue` diagnostics. The fourth printed five
such diagnostics and did not reach the probe's `DONE` marker. No second ping
request overlapped any run.

This challenges the premise that commit `1b820a413` created the ownership
failure. Raising the ping task from priority 2 to 6 changes scheduling and was
needed to avoid single-core C3 starvation, but the S3 still corrupts queued
callback state at the original priority. Scheduling is therefore not a safe
substitute for a native-to-interpreter ownership boundary.

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

The complete source difference from current master removed the two lines added
by `1b820a413`:

```diff
-  // Run above Espruino's priority-5 interpreter on single-core targets.
-  ping_config.task_prio = 6;
```

`ESP_PING_DEFAULT_CONFIG()` therefore supplied priority 2, exactly matching
the scheduling used before that commit. The diff remained an uncommitted
investigation change and is not a proposed correction.

| Artifact | SHA-256 |
|---|---|
| `bin/build/espruino.elf` | `1e76cb36568db39a91b3fa0f0b46653f31386e46b26b84f223a1903844585cbf` |
| `bin/build/espruino.bin` | `fecb77502b255fe4948aa5d28a7ce9dc58a593e536531972fbfd21e359ce1449` |
| `bin/espruino_2v29.536_esp32s3.bin` | `fecb77502b255fe4948aa5d28a7ce9dc58a593e536531972fbfd21e359ce1449` |

The target, stable native-USB serial path, raw-`esptool` procedure, runner and
single-handoff reproducer were unchanged from the exact non-debug
current-master run. The flash completed with verified hashes and an RTS hard
reset; the transcript is retained as
[`flash-default-priority.log`](flash-default-priority.log).

## Results

| Run | Runner result | Observation | Classification |
|---|---:|---|---|
| 01 | 2 | first callback valid, then abort and reboot in `gettimeofday()` lock acquisition | separate S3 time-lock issue; excluded from #2797 result count |
| 02 | 0 | five valid callbacks, `DONE=PASS` | timing-sensitive pass |
| 03 | 1 | zero callbacks, four unknown callback diagnostics, `DONE=FAIL` | issue #2797 failure |
| 04 | 1 | zero callbacks, four unknown callback diagnostics, `DONE=FAIL` | issue #2797 failure |
| 05 | 2 | five unknown callback diagnostics, no `DONE` marker | issue #2797 failure |
| 06 | 1 | zero callbacks, four unknown callback diagnostics, `DONE=FAIL` | issue #2797 failure |

Runs 03 and 06 are byte-identical captures of the same observable failure.
The clean run does not invalidate the failures because the defect is timing
sensitive and the native task still accesses the interpreter's variable and
event state without ownership.

The run-01 backtrace resolves through:

```text
espruinoTask -> jsiLoop -> jsiIdle -> jshGetSystemTime -> gettimeofday
             -> _lock_acquire -> lock_acquire_generic -> abort
```

It is retained because it occurred during the controlled series, but it is not
used as evidence for #2797.

## Retained logs

| Log | SHA-256 |
|---|---|
| `flash-default-priority.log` | `0db104946d5f01347368431b35a856c170bcc02a92a9405d38aaaab361deb426` |
| `s3-default-priority-run-01.log` | `8f947054d6c3fb2434859f5bbaa9373ed83595225c6b5fd6333f352d53b86135` |
| `s3-default-priority-run-02.log` | `fe8a61fbab41d2435c8349f91fee4a2d111cf43c8f5f6bc8cb453456b07069af` |
| `s3-default-priority-run-03.log` | `64a8e15858874e016a178829ab651d305282a6e0e3acbba943ae81154937486a` |
| `s3-default-priority-run-04.log` | `98c976440df3e4938e3c135dfcb3d016d965e37cfaf59f24adae2f0f6df24dc1` |
| `s3-default-priority-run-05.log` | `23b661af7dfe12899b20b4c7507102afd32f3e33c685bfc8a1994df9b60f7771` |
| `s3-default-priority-run-06.log` | `64a8e15858874e016a178829ab651d305282a6e0e3acbba943ae81154937486a` |

The test uses IPv4 loopback and the logs contain no Wi-Fi credentials.

## IDF4 and IDF5 interpretation

The ownership violation predates the IDF5 migration at source level. ESP32
ping's `pingResults()` callback has created `JsVar` result objects and called
`jsiQueueEvents()` from the native result path since commit `e53b7a4f5` in
December 2018. IDF4.4.8 documents that callback as executing from its internal
ping task and also defaults that task to priority 2. The September 2026 IDF5
ping adaptation replaced the native API but preserved the same direct
`JsVar`/event-queue work.

Existing S3 IDF4 physical evidence at `70555065f` also demonstrated invalid
event-queue callbacks, assertions and reboots when an active native ping
survived JavaScript `reset()`. That proves the broader ownership/lifetime flaw
was physically present before IDF5. It does not by itself prove that the same
ordinary single-session corruption rate should occur on IDF4; backend data
shape, timing and scheduling can change observability.

## Disposition

Neither priority 5 nor the original priority 2 is an acceptable correction.
The maintainer-proposed shared handoff remains the relevant production
direction: native tasks capture ordinary C data, signal a network-pending IO
event, and common interpreter-owned code creates `JsVar` values and dispatches
callbacks. The shared ESP32 IO-event buffer must first be made safe for its
multiple FreeRTOS-task producers.

