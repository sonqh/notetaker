# Proposal

## Why

The repo has docs and no code. Milestone M0 needs a Swift package that builds and an embedded Info.plist. It also needs argument parsing, the exit code table, and a test that keeps network APIs out of the tool.

Two risks can stop the project before M1. The system audio tap can deliver silence when the permission goes to the wrong app. Speech accuracy can be too low for useful notes. Each risk has a cheap experiment. This change builds both experiments as small probes, so we learn early and spend little.

## What Changes

- Add `Package.swift` with the `NotetakerKit` library, the `notetaker` executable, and the `NotetakerKitTests` test target. Tools version 6.2 or later, platform `.macOS(.v26)`, no dependency.
- Embed an Info.plist in the `notetaker` binary with a linker `__TEXT,__info_plist` section. It holds `NSMicrophoneUsageDescription` and `NSAudioCaptureUsageDescription`.
- Parse `doctor`, `record`, and `debug-audio` and their options from `docs/spec.md` with plain Swift. An unknown option exits 1.
- Define the exit code table from `docs/spec.md` in one place.
- Add a unit test that fails when a file under `Sources/` names a networking API.
- Add a user-level install script that copies the binary to `~/.local/bin` with no `sudo`.
- Add the `tap-probe` dev executable. It taps all system audio, plays Tink in a child process, and prints peak, RMS, and callbacks per second. It can write a capture file.
- Add the `speech-probe` dev executable. It transcribes an audio file with `SpeechAnalyzer` and scores it against a reference text with word recall and WER.
- Add gate scripts under `scripts/gates/` for the checks that need the dev Mac with audio.
- Record the probe results in `docs/findings.md`.

Out of scope:

- Capture streams for real use, ring buffers, and `AsyncStream` chunks (M1).
- The signal watchdog (M1).
- The transcription pipeline in `record` (M2).
- The Markdown writer and the partial file (M3).
- The menu bar app and any GUI (M4).
- App bundle signing. The install script copies a binary only.

## Capabilities

### New Capabilities

- `package`: the Swift package layout, the embedded Info.plist, argument parsing, the exit code table, the no-network test, and the user-level install.
- `probes`: the `tap-probe` and `speech-probe` dev executables and their measured output.

### Modified Capabilities

None.

## Impact

- New files: `Package.swift`, `Sources/NotetakerKit/`, `Sources/notetaker/`, `Sources/tap-probe/`, `Sources/speech-probe/`, `Tests/NotetakerKitTests/`, `scripts/install.sh`, and `scripts/gates/001-*.sh`.
- Frameworks: CoreAudio, AVFoundation, and Speech from the SDK. No third-party package.
- The probes trigger the system audio recording prompt. The owner records which app the prompt names.
- Later milestones build on the argument parser, the exit codes, and the tap setup that the probe proves.
