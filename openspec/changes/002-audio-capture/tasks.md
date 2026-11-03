# Tasks

## 1. Chunk type, ring buffer, and timeline

- [ ] 1.1 Add the chunk type with samples, sample rate, channel count, and first-sample host time, plus the t0 helper for relative time. [REQ-CAP-004] [REQ-CAP-005] Verify: `swift test --filter ChunkTests`
- [ ] 1.2 Add the preallocated single-producer, single-consumer ring buffer with atomic indices and an overflow counter. [REQ-CAP-006] Verify: `swift test --filter RingBufferTests`
- [ ] 1.3 Add a source scan test that fails when the callback files contain a lock, an allocation call, a log call, a file write, or `await`. [REQ-CAP-006] Verify: `swift test --filter CallbackPathTests`
- [ ] 1.4 Add the consumer that drains the ring buffer, fills gaps with zeros of equal length, counts drops, and yields chunks on an `AsyncStream`. [REQ-CAP-007] [REQ-CAP-004] Verify: `swift test --filter TimelineTests`

## 2. Level meter and watchdog

- [ ] 2.1 Add the level meter for peak, RMS, and non-zero count over all channels. [REQ-WD-001] Verify: `swift test --filter LevelMeterTests`
- [ ] 2.2 Add the watchdog window with N from 1 to 60, default 5, the pending state, and the at start mark. [REQ-WD-002] Verify: `swift test --filter WatchdogWindowTests`
- [ ] 2.3 Add the dead rules for no callbacks and for Me all zeros, with tests at start and later. [REQ-WD-003] [REQ-WD-004] Verify: `swift test --filter WatchdogDeadTests`
- [ ] 2.4 Add the Them idle and suspect rules from a fake output activity source, with and without another output process. [REQ-WD-005] [REQ-WD-006] Verify: `swift test --filter WatchdogThemTests`
- [ ] 2.5 Add the live state and the one-time recovery note after a warning, with a normal signal test. [REQ-WD-007] Verify: `swift test --filter WatchdogRecoveryTests`

## 3. Hardware protocols and capture

- [ ] 3.1 Define the protocols for tap source, microphone source, device monitor, output activity, permission reader, version reader, and sound player, with test fakes. [REQ-CAP-001] [REQ-CAP-003] [REQ-DOC-001] Verify: `swift test --filter FakeHardwareTests`
- [ ] 3.2 Build Them capture: the excluding stereo tap, the private aggregate device, and the IO callback into the ring buffer. [REQ-CAP-001] [REQ-CAP-006] Verify: `swift test --filter TapDescriptionTests`
- [ ] 3.3 Build the idempotent teardown and call it from `defer`, error paths, and the SIGINT and SIGTERM handler. Needs the dev Mac with audio. [REQ-CAP-002] Verify: `scripts/gates/002-teardown.sh`
- [ ] 3.4 Build Me capture on the input node with voice processing off, and set t0 when both captures run. [REQ-CAP-003] [REQ-CAP-005] Verify: `swift test --filter MicCaptureTests`
- [ ] 3.5 Add the output activity source that reads the running output flag of every process except this one. [REQ-WD-006] Verify: `swift test --filter OutputActivityTests`
- [ ] 3.6 Add device listeners and the engine configuration observer, post events, restart Me, and fill the restart gap. [REQ-CAP-008] Verify: `swift test --filter DeviceChangeTests`

## 4. Doctor capture tests

- [ ] 4.1 Add the doctor runner. It runs the doctor steps in order, prints one line per step and "skipped" lines, and exits with the first failed code. [REQ-DOC-001] Verify: `swift test --filter DoctorRunnerTests`
- [ ] 4.2 Add the doctor steps for the macOS version and for the microphone permission, with no prompt. [REQ-DOC-002] [REQ-DOC-003] Verify: `swift test --filter DoctorChecksTests`
- [ ] 4.3 Add the doctor step for the process tap and the default devices line. [REQ-DOC-004] [REQ-DOC-006] Verify: `swift test --filter DoctorTapTests`
- [ ] 4.4 Add the Them self-test with afplay Tink and the peak threshold of 0.01, tested with a fake player and fake tap. [REQ-DOC-005] Verify: `swift test --filter SelfTestTests`
- [ ] 4.5 Add the doctor gate script that runs `notetaker doctor` and asserts exit 0 and a self-test peak above 0.01. Needs the dev Mac with audio. [REQ-DOC-001] [REQ-DOC-005] Verify: `scripts/gates/002-doctor.sh`

## 5. Debug-audio command

- [ ] 5.1 Add `debug-audio` with `--seconds S`, default 10, and the usage error for a bad value. [REQ-DBG-001] Verify: `swift test --filter DebugAudioArgsTests`
- [ ] 5.2 Add the per-second report line per stream and the device event lines, tested with fake sources. [REQ-DBG-002] [REQ-DBG-003] Verify: `swift test --filter DebugAudioReportTests`
- [ ] 5.3 Add the exit code rule and the chunk latency p50 and p95 summary, and test that the command writes no file. [REQ-DBG-004] [REQ-DBG-005] Verify: `swift test --filter DebugAudioExitTests`
- [ ] 5.4 Add the speech gate script that runs `debug-audio` while `say "testing one two three"` plays, then asserts a Them peak above 0.01, Me non-zero samples, and overflow 0. Needs the dev Mac with audio. [REQ-DBG-002] [REQ-CAP-001] [REQ-CAP-003] Verify: `scripts/gates/002-debug-say.sh`
- [ ] 5.5 Add the silent gate script that runs `debug-audio` with no sound and asserts that Them shows idle, not dead or suspect. Needs the dev Mac with audio. [REQ-DBG-002] [REQ-WD-005] Verify: `scripts/gates/002-debug-silent.sh`

## 6. Findings

- [ ] 6.1 Run the tool from Terminal and from VS Code, and record which app each permission prompt names in `docs/findings.md`. Needs the dev Mac with audio. [REQ-DOC-005] [REQ-CAP-001] Verify: `scripts/gates/002-findings.sh attribution`
- [ ] 6.2 Measure the false suspect rate in a quiet session with common apps open, and the chunk latency p50 and p95, and record both in `docs/findings.md`. Needs the dev Mac with audio. [REQ-WD-006] [REQ-DBG-005] Verify: `scripts/gates/002-findings.sh measurements`
