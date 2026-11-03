# Proposal

## Why

Capture is the risky part of notetaker. macOS audio APIs can return success and deliver silence. A denied permission, the wrong device, or the echo canceller can each cause this. Milestone M1 must prove that the tool hears both sides of a meeting before any transcription work starts.

This change builds the two audio streams, the signal watchdog, the capture checks of `notetaker doctor`, and `notetaker debug-audio`. The tool then measures signal and does not trust a return code alone.

## What Changes

- Capture "Them" (system audio) with a global stereo process tap inside a private aggregate device. The tap excludes this process. The tool destroys the tap and the aggregate device on every exit path that it can catch.
- Capture "Me" (microphone) from the default input device. Voice processing stays off.
- Deliver each stream as an `AsyncStream` of chunks. Each chunk holds samples, sample rate, channel count, and the host time of its first sample. Both streams share one host-time reference, t0.
- Keep the audio callback non-blocking. It copies samples into a preallocated bounded ring buffer and counts overflow. A consumer task fills each drop with zeros of the same length, so the timeline stays continuous.
- Observe default device changes and engine configuration changes. Report them, restart Me capture, and keep the timeline.
- Add a level meter (peak, RMS, non-zero count) and a per-stream watchdog with the states live, idle, suspect, and dead.
- Add `notetaker doctor` with the capture checks: macOS version, microphone permission, tap creation, the Them self-test with Tink, and the default devices.
- Add `notetaker debug-audio [--seconds S]`. It prints levels, counters, and watchdog state once per second for each stream, and prints device events.
- Put hardware access behind small protocols. Unit tests then feed synthetic chunks and need no audio hardware.
- Add gate scripts under `scripts/gates/` for the M1 hardware checks, and fill the M1 rows in `docs/findings.md`.

## Out of scope

- Transcription and the English speech model check in `doctor`. Change 003 owns them.
- The Markdown writer, file output, and `--keep-audio`.
- The `record` command and its exit behavior on watchdog states. This change gives `record` the states. It does not act on them.
- Any GUI or menu bar app.
- A tap on one app only.
- Duplicate removal for laptop speakers.

## Capabilities

### New Capabilities

- `audio-capture`: the Me and Them streams, the chunk type, the shared host time, the non-blocking ring buffer, timeline continuity, teardown, and device change handling.
- `signal-watchdog`: the level meter and the per-stream watchdog states, as pure logic that runs on synthetic chunks.
- `doctor`: the capture checks of `notetaker doctor` and its exit code rule.
- `debug-audio`: the `notetaker debug-audio` command that prints live capture numbers and writes no file.

### Modified Capabilities

None.

## Impact

- New code in `Sources/NotetakerKit/` for capture, the meter, and the watchdog. New command handlers in `Sources/notetaker/` for `doctor` and `debug-audio`.
- Builds on change 001: the package, argument parsing, the exit code table, the embedded Info.plist, and the tap probe.
- New tests in `Tests/NotetakerKitTests/`. New scripts `scripts/gates/002-*.sh`.
- New rows in `docs/findings.md`: permission attribution, false suspect rate, and chunk latency.
- The tool triggers the standard microphone and system audio recording prompts. It changes no System Settings and no TCC state.
- No new dependency. No network call.
