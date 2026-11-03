# Spec Delta

## Purpose

The package capability defines how the tool builds and which permissions it declares. It also defines the command line, the exit codes, and an install with no admin rights.

## ADDED Requirements

### Requirement: REQ-PKG-001 Package layout

The repo MUST hold one Swift package with tools version 6.2 or later and the platform `.macOS(.v26)`. It MUST define the `NotetakerKit` library, the `notetaker` executable, and the `NotetakerKitTests` test target. The package MUST declare no package dependency.

Refs: ADR-001, ADR-004

#### Scenario: SCN-PKG-001-1 The package builds and tests pass
- **WHEN** a developer runs `swift build` and `swift test` in the repo root on the dev Mac
- **THEN** both commands exit 0
- **AND** `swift build --product notetaker` produces `.build/debug/notetaker`

#### Scenario: SCN-PKG-001-2 The package has no dependency
- **WHEN** a developer runs `swift package dump-package`
- **THEN** the `dependencies` array is empty
- **AND** the platforms list holds macOS 26.0

### Requirement: REQ-PKG-002 Embedded Info.plist

The `notetaker` binary MUST carry an Info.plist in a `__TEXT,__info_plist` section that the linker adds. The plist MUST hold `NSMicrophoneUsageDescription` and `NSAudioCaptureUsageDescription`, each with a short reason that a user can read in the permission prompt.

Refs: ADR-001, ADR-005

#### Scenario: SCN-PKG-002-1 otool shows both keys
- **WHEN** a developer runs `otool -s __TEXT __info_plist .build/debug/notetaker` and decodes the hex dump
- **THEN** the text holds `NSMicrophoneUsageDescription`
- **AND** the text holds `NSAudioCaptureUsageDescription`

#### Scenario: SCN-PKG-002-2 The source plist is valid
- **WHEN** a developer runs `plutil -lint` on the plist source file that the linker embeds
- **THEN** `plutil` reports the file as OK
- **AND** each usage description is a non-empty string

### Requirement: REQ-PKG-003 Command line parsing

The `notetaker` executable MUST parse the `doctor`, `record`, and `debug-audio` commands and their options from `docs/spec.md` with plain Swift code. The parser MUST live in `NotetakerKit` as a pure function that tests can call. A missing command, an unknown command, an unknown option, or a bad option value MUST print usage on stderr and exit 1.

Refs: REQ-PKG-004, ADR-001, ADR-004

#### Scenario: SCN-PKG-003-1 record options take their defaults
- **WHEN** the parser reads `record` with no option
- **THEN** the title is `Meeting`, the output folder is `~/Documents/Meetings`, keep-audio is off, and the silence timeout is 5 seconds

#### Scenario: SCN-PKG-003-2 record options take given values
- **WHEN** the parser reads `record --title "Standup" --out /tmp/m --keep-audio --silence-timeout 10`
- **THEN** the result holds the title `Standup`, the folder `/tmp/m`, keep-audio on, and a timeout of 10 seconds

#### Scenario: SCN-PKG-003-3 A silence timeout out of range is a usage error
- **WHEN** the parser reads `record --silence-timeout 0`, `record --silence-timeout 61`, or `record --silence-timeout abc`
- **THEN** the parser returns a usage error
- **AND** the executable exits 1

#### Scenario: SCN-PKG-003-4 debug-audio and doctor parse
- **WHEN** the parser reads `debug-audio`, `debug-audio --seconds 30`, and `doctor`
- **THEN** `debug-audio` has 10 seconds by default and 30 seconds with the option
- **AND** `doctor` parses with no option

#### Scenario: SCN-PKG-003-5 An unknown option exits 1
- **WHEN** a user runs `notetaker --bogus` or `notetaker record --bogus`
- **THEN** the tool prints usage on stderr
- **AND** the process exits 1

### Requirement: REQ-PKG-004 Exit code table

`NotetakerKit` MUST define every exit code from `docs/spec.md` in one type. The codes are 0 clean stop, 1 usage error, 2 macOS older than 26, 3 microphone denied, and 4 tap or aggregate device failed. Then 5 Me silent at start, 6 Them silent or self-test failed, and 7 speech model missing. Then 8 write failed, 9 stop with warnings, and 70 internal error.

Refs: ADR-003

#### Scenario: SCN-PKG-004-1 Each code has its value
- **WHEN** a unit test reads each case of the exit code type
- **THEN** the raw values are 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, and 70
- **AND** no two cases share a value

#### Scenario: SCN-PKG-004-2 An unhandled error maps to 70
- **WHEN** a command throws an error that has no exit code
- **THEN** the executable prints the error on stderr
- **AND** the process exits 70

### Requirement: REQ-PKG-005 No-network source test

A unit test MUST read every file under `Sources/` and fail when one names a networking API: `URLSession`, `URLRequest`, `NWConnection`, `NWListener`, `CFNetwork`, `CFSocket`, `import Network`, or a BSD `socket(` call. The failure MUST name the file and the match.

Refs: ADR-007

#### Scenario: SCN-PKG-005-1 The clean tree passes
- **WHEN** a developer runs `swift test --filter NoNetworkTests`
- **THEN** the test passes for the files under `Sources/`

#### Scenario: SCN-PKG-005-2 A planted mention fails
- **WHEN** the scanner reads a sample text that holds `URLSession.shared`
- **THEN** the scanner reports one match with the file name and the API name
- **AND** a sample text with no networking API gives no match

### Requirement: REQ-PKG-006 User-level install

`scripts/install.sh` MUST build the release binary and copy `notetaker` to `~/.local/bin`. It MUST create that folder when it is missing. It MUST use no `sudo`, write nothing outside the home folder, and tell the user when `~/.local/bin` is not on `PATH`.

Refs: ADR-005

#### Scenario: SCN-PKG-006-1 Install to a given home folder
- **WHEN** a developer runs `HOME=<temp folder> scripts/install.sh`
- **THEN** `<temp folder>/.local/bin/notetaker` exists and is executable
- **AND** `<temp folder>/.local/bin/notetaker --bogus` exits 1

#### Scenario: SCN-PKG-006-2 The script needs no admin rights
- **WHEN** a developer reads `scripts/install.sh`
- **THEN** the script has no `sudo`
- **AND** each `mkdir`, `cp`, or `install` target starts with `$HOME`
