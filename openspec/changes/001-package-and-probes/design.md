# Design

## Context

The repo holds docs only. `docs/spec.md` fixes the stack: one Swift package, no third-party dependency, deployment target macOS 26, a Core Audio process tap for system audio, and `SpeechAnalyzer` for speech. The dev Mac runs macOS 27 with Xcode 27 and Swift 6.4. The Mac is MDM-managed, so the developer has no admin rights.

Two risks come before M1. A tap can deliver exact zeros with no error when the system audio permission is off or goes to the wrong app. Speech accuracy on call audio is unknown. `docs/product.md` asks for a cheap experiment for each before more code exists.

## Goals / Non-Goals

**Goals:**

- A package that builds and tests on any Mac with the SDK, with the M0 gate checks passing.
- One tested parser and one exit code type that M1 to M3 reuse.
- A tap probe that answers one question by signal. Does the tap hear Tink from Terminal and from the VS Code terminal?
- A speech probe that turns an audio file and a reference text into a recall and WER number. It stays as the offline quality tool.

**Non-Goals:**

- Microphone capture, ring buffers, chunks, and `AsyncStream` (M1).
- The watchdog and `doctor` checks (M1).
- Live transcription and the Markdown writer (M2 and M3).
- The menu bar app, app bundles, and code signing (M4).

## Decisions

- Use one Swift package with the targets `NotetakerKit`, `notetaker`, `tap-probe`, `speech-probe`, and `NotetakerKitTests`. An Xcode project would need a GUI step and adds files that SwiftPM can replace. Refs: REQ-PKG-001, REQ-PRB-006, ADR-001
- Embed the Info.plist with `linkerSettings: [.unsafeFlags(["-Xlinker", "-sectcreate", "-Xlinker", "__TEXT", "-Xlinker", "__info_plist", "-Xlinker", "<path>"])]` on the `notetaker` and `tap-probe` targets. The plist source lives in `Sources/notetaker/Info.plist`, and the target excludes it from resources. `unsafeFlags` is fine for a root package. A bundle would also work but is M4 work. Refs: REQ-PKG-002, ADR-001, ADR-005
- Write the argument parser by hand in `NotetakerKit` as a pure function from `[String]` to a command enum or a usage error. The option set is small. swift-argument-parser is a third-party dependency and the rules forbid it. Refs: REQ-PKG-003, ADR-001
- Define exit codes as an `Int32` raw-value enum in `NotetakerKit`. `main` catches every error, maps a typed error to its code, and maps any other error to 70. Refs: REQ-PKG-004
- The no-network test walks `Sources/` from `#filePath`, reads each `.swift` file, and runs a plain substring and regex scan. The scanner is a pure function, so a test can feed it a planted string. A source scan is cheap and catches the common APIs. It cannot catch a dynamic symbol lookup, and code review covers that. Refs: REQ-PKG-005, ADR-007
- `scripts/install.sh` runs `swift build -c release --product notetaker` and copies the binary to `$HOME/.local/bin`. It honors `HOME` so a test can point it at a temp folder. No signing in M0. A stable self-signed identity comes with the app in M4. Refs: REQ-PKG-006, ADR-005
- `tap-probe` builds a `CATapDescription(stereoGlobalTapButExcludeProcesses:)` with its own process object, sets it private and unmuted, then creates a private aggregate device with the tap in its tap list. The IO callback only adds to preallocated counters and copies samples into a preallocated buffer. A one-second timer on another thread reads and resets the counters. Refs: REQ-PRB-001, ADR-002, ADR-003
- The Tink self-test runs `afplay` with `Process` after the first callback arrives, and judges pass or fail from the peak alone. Core Audio return codes only decide whether to continue setup. Refs: REQ-PRB-002, ADR-003
- `tap-probe --write` hands buffers from the callback to a consumer task through the same preallocated buffer. The consumer writes the CAF with `AVAudioFile`. Refs: REQ-PRB-003, ADR-003, ADR-006
- `speech-probe` uses `SpeechAnalyzer(inputAudioFile:modules:)` with `SpeechTranscriber` and the preset `timeIndexedProgressiveTranscription` by default, or `DictationTranscriber` with `--engine dictation`. It checks `en_US` with `SpeechTranscriber.installedLocales` and never calls `AssetInventory`. It uses no API that needs macOS 27. Refs: REQ-PRB-004, ADR-008, ADR-007
- Word recall and WER live in `NotetakerKit` as pure functions. WER uses word-level Levenshtein distance divided by the reference length. Recall counts reference words matched in the alignment. Tests use small known pairs. Refs: REQ-PRB-005, ADR-008
- Hardware checks run from `scripts/gates/001-tap-probe.sh` and `scripts/gates/001-speech-probe.sh`. Each script prints the real probe output and exits with the probe's code. The developer runs the tap gate twice, once from Terminal and once from the VS Code terminal. Refs: REQ-PRB-006, REQ-PRB-002, REQ-PRB-005

## Risks / Trade-offs

- A tap alone delivers no buffers. The probe puts the tap in a private aggregate device and prints both IDs, so a missing step shows up at once.
- A missing system audio permission gives silence with no error. The probe judges by peak only and names the permission when it fails.
- macOS can attach the permission to the app that starts the tool and not to the binary. The result can differ between Terminal and VS Code. The gate runs from both and `docs/findings.md` records which app the prompt named. This is a manual note.
- A SwiftPM binary has no Info.plist unless the linker embeds one. The `otool` check proves the section exists. Only a real prompt proves that macOS reads it.
- The audio callback must never block. The probe callback does no lock, allocation, file write, log, or `await`. A code review of the callback is part of the tap task.
- Voice processing can stop the process tap. The probes never open the microphone, so this trap does not apply yet. M1 must keep it in mind.
- The tap may miss Bluetooth A2DP output. The probe can show this with AirPods, and `docs/findings.md` records the result. M0 does not depend on it.
- Text-to-speech audio is cleaner than real speech. A 90% recall on a `say` file is an upper bound. A real call sample from `tap-probe --write` gives a truer number.
- The `en_US` model may be missing on the dev Mac. `speech-probe` exits 7 and does not download. The developer installs it through `notetaker record` later or through the OS.
- The source scan for network APIs can miss an indirect call. It is a cheap guard, and code review stays the main check.
- `unsafeFlags` blocks use of the package as a dependency. This package is a root package only, so the cost is zero.
