# Design

## Context

Change 001 gives a Swift package with the targets `NotetakerKit`, `notetaker`, and `NotetakerKitTests`. It also gives argument parsing, the exit code table, an embedded Info.plist with both usage keys, and a small tap probe. This change turns the probe into real capture for milestone M1.

The constraints come from `docs/spec.md`, `docs/product.md`, and the decision records. The package uses Swift 6 and targets macOS 26. It has no third-party dependency and makes no network call. It must run on an MDM-managed Mac without admin rights. Audio APIs can return success and deliver silence, so every check measures signal.

No public API reports the system audio recording permission. The tap delivers exact zeros when the permission is off, and also when the Mac is quiet. The watchdog and the doctor self-test exist to tell these two cases apart.

## Goals / Non-Goals

**Goals:**

- Two aligned chunk streams, Me and Them, on one host-time timeline.
- An audio callback that never blocks, with overflow counted and gaps filled.
- A level meter and a watchdog as pure logic, tested with synthetic chunks.
- `notetaker doctor` capture checks with a measured Them self-test.
- `notetaker debug-audio` that prints the real numbers per second.
- Gate scripts and findings rows for the M1 gate.

**Non-Goals:**

- Transcription, the speech model check, and file output.
- The `record` command and its reaction to watchdog states.
- A GUI, a per-app tap, and duplicate removal for laptop speakers.

## Decisions

- Them uses a global stereo process tap that excludes this process, inside a private aggregate device. ScreenCaptureKit needs the heavier Screen Recording permission. A virtual driver needs admin rights. Refs: REQ-CAP-001, ADR-002, ADR-005
- Me uses the input node of an audio engine with voice processing off. Voice processing can stop the tap, and echo removal is not needed in M1. Refs: REQ-CAP-003, ADR-002
- Hardware sits behind seven small protocols. They cover the tap, the microphone, device events, output activity, permission state, the system version, and the sound player. Real types wrap Core Audio and AVFoundation. Test fakes feed synthetic chunks. Refs: REQ-WD-001, REQ-DOC-001, ADR-001
- The ring buffer is a single-producer, single-consumer buffer of Float samples with atomic indices. The capture allocates it once at start, with room for 2 seconds of audio per stream. The callback only copies and updates counters. A Mutex or a dispatch queue would block the audio thread. Refs: REQ-CAP-006, ADR-003
- The callback writes the host time of each block next to its samples in a small fixed metadata ring. The consumer compares that host time with the expected one and inserts zeros for any gap. Gaps from overflow and from a Me restart then use one code path. Refs: REQ-CAP-007, REQ-CAP-008
- A consumer task per stream drains the ring buffer about every 20 ms and yields chunks on an `AsyncStream` with a bounded buffering policy. The meter and the watchdog run in this task, never in the callback. Refs: REQ-CAP-004, REQ-WD-001
- t0 is the host time after both captures report their first callback, or after both start calls return when one stream is silent. All relative times use t0. Refs: REQ-CAP-005
- Teardown is one idempotent function. A `defer` block, the error paths, and a `DispatchSource` handler for SIGINT and SIGTERM all call it. SIGKILL cannot be caught, so a leaked private aggregate device is possible after `kill -9`. The OS removes a private aggregate device when its owner process ends. Refs: REQ-CAP-002, ADR-002
- The watchdog is a value type with no clock of its own. It takes chunk statistics, a tick with the current host time, and the output activity of other processes. It returns state changes and notes. Tests drive it with synthetic time. Refs: REQ-WD-002, REQ-WD-003, ADR-003
- A stream starts in a pending state. It leaves pending on its first non-zero sample or when the first window ends. This keeps the first seconds of `debug-audio` from showing a false dead or idle state. Refs: REQ-WD-002
- The output activity source lists the Core Audio process objects. It reads `kAudioProcessPropertyIsRunningOutput` for each, and skips the object of this process. The watchdog reads it once per tick and never from the callback. Refs: REQ-WD-005, REQ-WD-006
- Doctor runs every check even after a failure, so the user sees the full picture. A check that depends on a failed check prints "skipped". The exit code is the code of the first failed check in table order. Refs: REQ-DOC-001, REQ-DOC-005
- The self-test runs afplay through `Process` with a fixed path and no shell. It waits for the player to end plus one second, then reads the highest Them peak in that time. Refs: REQ-DOC-005, ADR-003
- The microphone check reads the authorization status only. It never calls the request API, so doctor shows no prompt. Refs: REQ-DOC-003, ADR-005
- Debug-audio measures chunk latency as the host time at consumer arrival minus the host time of the chunk's last sample. It prints p50 and p95 at the end. The M1 findings row takes these numbers. Refs: REQ-DBG-005
- Device events come from Core Audio property listeners on the default input and output device, plus the engine configuration change notification. The listener only posts an event to the consumer side. The consumer restarts the Me engine. Refs: REQ-CAP-008, REQ-DBG-003

## Risks / Trade-offs

- A tap alone delivers no buffers. Mitigation: the tap always sits in a private aggregate device, and doctor tests the pair.
- A missing system audio permission gives silence with no error. Mitigation: the doctor self-test with Tink, and the suspect state in the watchdog.
- The permission can attach to the app that started the tool, such as Terminal or VS Code, and not to the binary. Mitigation: measure it in M1, record it in `docs/findings.md`, and name the parent app in the self-test failure text.
- Voice processing on the input node can stop the tap. Mitigation: the code never enables it, and a unit test checks the flag on the fake engine.
- Bluetooth A2DP output may not reach the tap. A call can also move AirPods into HFP mode with a new input device and a low sample rate. Mitigation: test with AirPods, record the result, and handle the new format through the Me restart path.
- The default devices can change mid-meeting. Mitigation: device listeners, a Me restart, and zero fill so the timeline stays right. The Them tap follows the system output. M1 must test this on a device change.
- Some apps keep output running while they play silence. That gives a false suspect state on a quiet Mac. Mitigation: suspect only warns and never ends a command. M1 measures the false suspect rate.
- A ring buffer of 2 seconds can overflow when the consumer stalls. Mitigation: the overflow counter, zero fill, and the gate rule that overflow stays 0 in a normal run.
- Atomic indices in Swift 6 need the `Synchronization` module. It ships with the toolchain and adds no dependency.
