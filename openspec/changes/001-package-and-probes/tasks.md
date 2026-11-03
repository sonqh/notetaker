# Tasks

## 1. Package skeleton

- [ ] 1.1 Create `Package.swift` with tools version 6.2, platform `.macOS(.v26)`, no dependency, and the targets `NotetakerKit`, `notetaker`, `tap-probe`, `speech-probe`, and `NotetakerKitTests`, each with a stub source file [REQ-PKG-001] [REQ-PRB-006] Verify: `swift build`
- [ ] 1.2 Add `Sources/notetaker/Info.plist` with both usage descriptions. Embed it with the linker `-sectcreate __TEXT __info_plist` flags on `notetaker` and `tap-probe`. Write `scripts/gates/001-plist.sh` to decode the `otool` dump and look for both keys [REQ-PKG-002] Verify: `scripts/gates/001-plist.sh`

## 2. Command line

- [ ] 2.1 Add the exit code enum to `NotetakerKit` with the values from `docs/spec.md`, and add `ExitCodeTests` [REQ-PKG-004] Verify: `swift test --filter ExitCodeTests`
- [ ] 2.2 Add the hand-written parser for `doctor`, `record`, and `debug-audio` with defaults, range rules, and usage errors. Add `CommandLineTests` for each scenario [REQ-PKG-003] Verify: `swift test --filter CommandLineTests`
- [ ] 2.3 Wire `main` in `notetaker`. Print usage on stderr for a usage error, and map typed errors to their code and other errors to 70. Write `scripts/gates/001-m0.sh` for the full M0 gate [REQ-PKG-003] [REQ-PKG-004] Verify: `scripts/gates/001-m0.sh`

## 3. No-network guard and install

- [ ] 3.1 Add the network API scanner as a pure function in `NotetakerKit`. Add `NoNetworkTests`, which scans every file under `Sources/` and also scans planted samples [REQ-PKG-005] [REQ-PRB-006] Verify: `swift test --filter NoNetworkTests`
- [ ] 3.2 Write `scripts/install.sh` for a user-level install to `$HOME/.local/bin`, and write `scripts/gates/001-install.sh` to run it with a temp `HOME` and verify the result [REQ-PKG-006] [REQ-PRB-006] Verify: `scripts/gates/001-install.sh`

## 4. tap-probe

- [ ] 4.1 Add the `tap-probe` option parser to `NotetakerKit` for `--seconds`, `--write`, and `--no-tink`, and add `TapProbeOptionsTests` [REQ-PRB-003] Verify: `swift test --filter TapProbeOptionsTests`
- [ ] 4.2 Add peak and RMS as pure functions in `NotetakerKit`, and add `SignalLevelTests` with zeros, a full-scale sine, and a known short buffer [REQ-PRB-001] Verify: `swift test --filter SignalLevelTests`
- [ ] 4.3 Build the tap in a private aggregate device and the non-blocking IO callback with preallocated counters. Add the one-second level lines and teardown on exit and SIGINT [REQ-PRB-001] Verify: `swift build --product tap-probe`
- [ ] 4.4 Add the Tink self-test with `afplay` in a child process and the pass rule on peak above 0.01. Add exit 6 with the permission hint, and the parent app line. Write `scripts/gates/001-tap-probe.sh`. Needs the dev Mac with audio [REQ-PRB-002] Verify: `scripts/gates/001-tap-probe.sh`
- [ ] 4.5 Add `--write` with a consumer task that writes a 32-bit float CAF, and add a `--write` run with an `afinfo` step to `scripts/gates/001-tap-probe.sh`. Needs the dev Mac with audio [REQ-PRB-003] Verify: `scripts/gates/001-tap-probe.sh`

## 5. speech-probe

- [ ] 5.1 Add the text normalizer, word recall, and word-level WER as pure functions in `NotetakerKit`, and add `TranscriptScoreTests` with the known pairs [REQ-PRB-005] Verify: `swift test --filter TranscriptScoreTests`
- [ ] 5.2 Add the `speech-probe` option parser to `NotetakerKit` for the file, `--reference`, and `--engine`, and add `SpeechProbeOptionsTests` [REQ-PRB-004] Verify: `swift test --filter SpeechProbeOptionsTests`
- [ ] 5.3 Build `speech-probe` with `SpeechAnalyzer` on a file, the `SpeechTranscriber` preset `timeIndexedProgressiveTranscription` as default, `DictationTranscriber` for `--engine dictation`, exit 7 when the model is missing, and the score lines [REQ-PRB-004] [REQ-PRB-005] Verify: `swift build --product speech-probe`
- [ ] 5.4 Add a reference paragraph of at least 150 words to `Tests/Fixtures/`, and write `scripts/gates/001-speech-probe.sh` to make a `say -v Samantha -o` file, run both engines, and fail below 0.90 recall. Needs the dev Mac with the `en_US` model [REQ-PRB-005] [REQ-PRB-004] Verify: `scripts/gates/001-speech-probe.sh`

## 6. Findings

- [ ] 6.1 Run the tap gate from Terminal and from the VS Code terminal. Record in `docs/findings.md` the peaks, the app that the prompt named, and the result from each host. Needs the dev Mac with audio [REQ-PRB-002] Verify: `scripts/gates/001-tap-probe.sh`
- [ ] 6.2 Run the speech gate. Record in `docs/findings.md` the recall and WER of both engines and the fixture name. Note that `say` audio gives an upper bound [REQ-PRB-005] Verify: `scripts/gates/001-speech-probe.sh`
