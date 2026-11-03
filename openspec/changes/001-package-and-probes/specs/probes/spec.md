# Spec Delta

## Purpose

The probes capability defines two small dev executables. They test the two largest risks early: a silent system audio tap and poor speech accuracy.

## ADDED Requirements

### Requirement: REQ-PRB-001 tap-probe system audio capture

The `tap-probe` executable MUST create a global stereo process tap that excludes its own process. It MUST put the tap in a private aggregate device and start an IO callback. Once per second it MUST print the peak, the RMS, and the callback count. It MUST destroy the aggregate device and the tap on a normal exit and on SIGINT.

Refs: ADR-002, ADR-003

#### Scenario: SCN-PRB-001-1 The tap sits in a private aggregate device
- **WHEN** a developer runs `tap-probe` on the dev Mac
- **THEN** the probe prints the tap ID and the aggregate device ID before the first level line
- **AND** the aggregate device does not appear in the user's device list in Audio MIDI Setup

#### Scenario: SCN-PRB-001-2 One level line per second
- **WHEN** the probe runs for 5 seconds
- **THEN** it prints 5 level lines
- **AND** each line holds the peak, the RMS, and the callbacks in that second

#### Scenario: SCN-PRB-001-3 Teardown on Ctrl-C
- **WHEN** the user presses Ctrl-C while the probe runs
- **THEN** the probe stops the IO callback, destroys the aggregate device and the tap, and prints a teardown line
- **AND** a new run of the probe creates a new tap with no error

### Requirement: REQ-PRB-002 tap-probe Tink self-test

After the tap starts, `tap-probe` MUST run `/usr/bin/afplay /System/Library/Sounds/Tink.aiff` in a child process. It MUST judge the test by measured signal only. The test passes when the peak goes above 0.01. Else it fails with exit 6 and a hint about the system audio permission. It MUST print the host app name and a reminder to note which app the prompt named.

Refs: ADR-002, ADR-003, REQ-PKG-004

#### Scenario: SCN-PRB-002-1 Tink is heard
- **WHEN** the system audio permission is on for the app that started the probe and the probe plays Tink
- **THEN** the probe prints the highest peak and the word PASS
- **AND** the process exits 0

#### Scenario: SCN-PRB-002-2 Silence fails the test
- **WHEN** the tap delivers only zeros while Tink plays
- **THEN** the probe prints FAIL and names the system audio recording permission in System Settings
- **AND** the process exits 6, even if every Core Audio call returned success

#### Scenario: SCN-PRB-002-3 The prompt app is a manual note
- **WHEN** the probe starts
- **THEN** it prints the name of the parent process, such as Terminal or Code
- **AND** it prints one line that asks the developer to record in `docs/findings.md` which app the permission prompt named

### Requirement: REQ-PRB-003 tap-probe options

`tap-probe` MUST accept `--seconds S` to set the run length, with a default of 5. It MUST accept `--write <file.caf>` to save the tapped audio as 32-bit float PCM at the native rate. A consumer task MUST write the file, never the IO callback. `--no-tink` MUST skip the self-test so the probe can capture a real call.

Refs: ADR-003, ADR-006

#### Scenario: SCN-PRB-003-1 The run length follows --seconds
- **WHEN** a developer runs `tap-probe --seconds 3`
- **THEN** the probe prints 3 level lines and exits

#### Scenario: SCN-PRB-003-2 --write saves a readable file
- **WHEN** a developer runs `tap-probe --seconds 10 --write /tmp/tap.caf` while audio plays
- **THEN** `afinfo /tmp/tap.caf` reports 32-bit float PCM and a length of about 10 seconds

#### Scenario: SCN-PRB-003-3 A bad option exits 1
- **WHEN** a developer runs `tap-probe --seconds -1` or `tap-probe --bogus`
- **THEN** the probe prints usage on stderr and exits 1

### Requirement: REQ-PRB-004 speech-probe file transcription

`speech-probe <audio-file> [--reference <txt>] [--engine transcriber|dictation]` MUST run `SpeechAnalyzer` on the file with `en_US` and print the finalized transcript. The default engine MUST be `SpeechTranscriber` with the preset `timeIndexedProgressiveTranscription`. It MUST NOT start a model download, and MUST exit 7 when the model is missing.

Refs: ADR-008, ADR-007, REQ-PKG-004

#### Scenario: SCN-PRB-004-1 The default engine prints a transcript
- **WHEN** a developer runs `speech-probe /tmp/say.aiff` on a file that `say -o` wrote
- **THEN** the probe prints the finalized text with the start time of each result
- **AND** the process exits 0

#### Scenario: SCN-PRB-004-2 The dictation engine runs on the same file
- **WHEN** a developer runs `speech-probe /tmp/say.aiff --engine dictation`
- **THEN** the probe uses `DictationTranscriber` and prints its transcript and the engine name

#### Scenario: SCN-PRB-004-3 A missing model exits 7
- **WHEN** the `en_US` model is not installed
- **THEN** the probe prints that the model is missing and that `notetaker record` asks the OS to install it
- **AND** the process exits 7 with no download request

### Requirement: REQ-PRB-005 speech-probe accuracy score

When the user passes `--reference`, `speech-probe` MUST print word recall and word error rate against the reference text. Both texts MUST be normalized first: lower case, punctuation stripped, whitespace collapsed. The scoring code MUST live in `NotetakerKit` as pure functions, so it can serve as the offline quality tool in later milestones.

Refs: ADR-008, ADR-003

#### Scenario: SCN-PRB-005-1 Normalization
- **WHEN** the normalizer reads `Hello, World!  It's  fine.`
- **THEN** it returns the words `hello`, `world`, `its`, and `fine`

#### Scenario: SCN-PRB-005-2 Recall and WER on known pairs
- **WHEN** a unit test scores the hypothesis `the cat sat` against the reference `the cat sat down`
- **THEN** recall is 0.75 and WER is 0.25
- **AND** an exact match gives recall 1.0 and WER 0.0

#### Scenario: SCN-PRB-005-3 A say file reaches 90% recall
- **WHEN** the gate script writes a known paragraph of at least 150 words with `say -v Samantha -o` and runs `speech-probe` on it with that paragraph as the reference
- **THEN** the printed recall is 0.90 or higher
- **AND** the script prints the real recall and WER

### Requirement: REQ-PRB-006 Probes stay dev-only and offline

The probes MUST be separate executable products under `Sources/`, so the no-network test covers them. They MUST make no network call. `scripts/install.sh` MUST NOT install them. Each hardware check MUST run from a script under `scripts/gates/` that prints the real probe output.

Refs: REQ-PKG-005, REQ-PKG-006, ADR-007

#### Scenario: SCN-PRB-006-1 The no-network test covers the probes
- **WHEN** a developer runs `swift test --filter NoNetworkTests`
- **THEN** the test reads the files under `Sources/tap-probe/` and `Sources/speech-probe/`

#### Scenario: SCN-PRB-006-2 The install leaves the probes out
- **WHEN** a developer runs `HOME=<temp folder> scripts/install.sh`
- **THEN** `<temp folder>/.local/bin/` holds `notetaker` only
