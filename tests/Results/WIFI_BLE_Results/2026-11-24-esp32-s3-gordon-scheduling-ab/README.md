# ESP32-S3 Gordon combined scheduling A/B

Date: 2026-11-24

## Conclusion

Gordon Williams's complete least-invasive scheduling proposal does not resolve
issue #2797 on the ESP32-S3. The candidate placed the native ping task,
`sys_evt` and `espruinoTask` at priority 5 and explicitly yielded after every
`jsiLoop()` iteration. Only one of ten isolated single-ping probes passed.

Three runs cleanly reproduced the issue-#2797 event-queue failure class: one
printed five invalid-callback diagnostics and missed `DONE`; two delivered
zero callbacks, printed four invalid-callback diagnostics and ended
`DONE=FAIL`. These three runs alone reject the candidate as a safe production
correction.

Three other runs hit the separately tracked S3 `gettimeofday()` time-lock
abort and are excluded from the #2797 result. The remaining three showed a
callback type mismatch, lost/malformed input, or an interpreter `JsVar` lock
panic during a later upload. They are retained but are not needed to reject
the scheduling proposal.

The candidate remains flashed on the S3 after the automated run for manual
testing. It is an uncommitted investigation diff, not a production correction.

## Candidate definition

| Item | Value |
|---|---|
| Firmware repository | linked worktree of `/home/simon/MaBecker/Espruino_master` |
| Investigation branch | `investigate/esp32-ping-priority-2797-2026-11-24` |
| Base revision | `0cd5e987e9f2cace0ad7a532c6afebab907175fe` |
| Board definition | `ESP32S3` |
| Espruino version | `2v29.536` |
| ESP-IDF | 5.5.3 |
| Build mode | `RELEASE=1`, with `DEBUG` absent from the environment |
| `espruinoTask` | priority 5, pinned to core 0 |
| ping task | explicitly changed from 6 to 5 |
| `sys_evt` | located after default-loop creation and changed from `ESP_TASK_PRIO_MAX - 5` to 5 |
| additional handoff | `taskYIELD()` after every `jsiLoop()` iteration |
| FreeRTOS time slicing | enabled |

The complete source difference is retained as
[`gordon-scheduling-candidate.diff`](gordon-scheduling-candidate.diff),
SHA-256
`f76dcebe899ce2cb183a495f9109dbb3df3ff04718f2a799126767c207a76dfa`.
It changes only:

- `libs/network/esp32/jswrap_esp32_network.c`; and
- `targets/esp32/main.c`.

The `sys_evt` change uses `xTaskGetHandle("sys_evt")` immediately after
`esp_event_loop_create_default()` and lowers the returned task with
`vTaskPrioritySet()`. This is suitable for a controlled experiment; it is not
presented as a production interface to ESP-IDF's default event loop.

| Artifact | SHA-256 |
|---|---|
| `bin/build/espruino.elf` | `f98f8dfd1d30e4bbfd2a8c35e3e65d72fbffadb5b79145efd10c66b8fd5139fb` |
| `bin/build/espruino.bin` | `4e59f6fc0e37bdc857bc8f22be07d612f1d1411d05f82cfb247be256ca59fccb` |
| `bin/espruino_2v29.536_esp32s3.bin` | `4e59f6fc0e37bdc857bc8f22be07d612f1d1411d05f82cfb247be256ca59fccb` |

The raw `esptool` flash completed three hash-verified writes and an RTS hard
reset. The retained flash log has SHA-256
`af03909f2a6a9130720afe2be0e56dc57f6286edafce5e120f7041583c1b2472`.

## Automated results

Every run reset the interpreter, uploaded the same saved
`esp32_wifi_ping_single_handoff.js` probe and issued one loopback ping request.
No run issued a second request.

| Run | Runner | Observation | Classification |
|---|---:|---|---|
| 01 | 2 | `gettimeofday()` lock abort and reboot | separate time-lock issue |
| 02 | 2 | `gettimeofday()` lock abort and reboot | separate time-lock issue |
| 03 | 0 | five valid callbacks, `DONE=PASS` | pass |
| 04 | 2 | five invalid event-queue callback diagnostics, no `DONE` | issue #2797 failure |
| 05 | 1 | callback argument rejected as object; subsequent interpreter input damage | retained secondary failure |
| 06 | 1 | zero callbacks and malformed interpreter input, `DONE=FAIL` | retained secondary failure |
| 07 | 1 | zero callbacks, four invalid event-queue callback diagnostics, `DONE=FAIL` | issue #2797 failure |
| 08 | 1 | zero callbacks, four invalid event-queue callback diagnostics, `DONE=FAIL` | issue #2797 failure |
| 09 | 2 | `LoadProhibited` panic during later probe upload | retained interpreter-corruption failure |
| 10 | 2 | `gettimeofday()` lock abort and reboot | separate time-lock issue |

Run 09 resolves against the exact candidate ELF as:

```text
espruinoTask -> jsiLoop -> jsiIdle -> jsiHandleIOEventForConsole
             -> jsiHandleConsoleChar -> jsiHandleNewLine
             -> jspEvaluateVar -> jslInit -> jsvLockAgain
```

It is not the newlib time-lock abort. Because it occurred while the runner was
uploading a later probe rather than directly inside a ping callback, the three
clean invalid-event-queue runs remain the primary result.

## Retained logs

| Log | SHA-256 |
|---|---|
| `s3-gordon-scheduling-run-01.log` | `bada4ebb12b978c9dcfcdf954763db0a9514b03ff237a722f5b1b87fd73b3379` |
| `s3-gordon-scheduling-run-02.log` | `3cf8643ebd974e658e350a4ac44e95b5783580c2431e7b4560dd5af9051c68d3` |
| `s3-gordon-scheduling-run-03.log` | `fe8a61fbab41d2435c8349f91fee4a2d111cf43c8f5f6bc8cb453456b07069af` |
| `s3-gordon-scheduling-run-04.log` | `23b661af7dfe12899b20b4c7507102afd32f3e33c685bfc8a1994df9b60f7771` |
| `s3-gordon-scheduling-run-05.log` | `f9f395ba04d08146b49e6569c211da59635db9851915ec333a774a1c73e099fe` |
| `s3-gordon-scheduling-run-06.log` | `0a4e76d8d493ac16039876e09f8086554d6369d74692a83e7347ea6fa89fa749` |
| `s3-gordon-scheduling-run-07.log` | `64a8e15858874e016a178829ab651d305282a6e0e3acbba943ae81154937486a` |
| `s3-gordon-scheduling-run-08.log` | `492e5d76c6702a67c82153eef0c43aa7d7f561cb3085cda4a8ca567f668d0126` |
| `s3-gordon-scheduling-run-09.log` | `e049ec32b81c6b588e1a31b3b496c3fed45f417f0aea82587e5a4ee95e6393c3` |
| `s3-gordon-scheduling-run-10.log` | `90c177e18b76a02cf10bd4a6c057ef918a292b7385d23a4952ae8de4aea6345d` |

The test uses IPv4 loopback and the logs contain no Wi-Fi credentials.

## Disposition

The complete scheduling proposal has now been tested, not only equal ping
priority. More frequent yielding and equal `sys_evt` priority alter timing but
do not establish exclusive ownership of Espruino variable memory or its event
queue. The native-record/interpreter-task handoff remains necessary.

The candidate should not be qualified on classic ESP32 or C3 as a correction,
because it already fails its smallest S3 reproduction. It remains installed
on the S3 solely for requested manual investigation.

