# Spec Delta

## Purpose

Give the user one command that checks whether this Mac can capture both meeting audio streams, with measured signal as proof.

## ADDED Requirements

### Requirement: REQ-DOC-001 Doctor report and exit code
`notetaker doctor` SHALL run its checks in a fixed order and print one line per check with its name, pass or fail, and a detail. It SHALL exit 0 when all checks pass. Otherwise it SHALL exit with the code of the first failed check.

Refs: ADR-003, ADR-004

#### Scenario: SCN-DOC-001-1 All checks pass
- **WHEN** every check passes
- **THEN** doctor prints one pass line per check and exits 0

#### Scenario: SCN-DOC-001-2 Two checks fail
- **WHEN** the microphone check fails with code 3 and a later check fails with code 6
- **THEN** doctor prints a line for every check and exits 3

### Requirement: REQ-DOC-002 macOS version check
Doctor SHALL read the running macOS version and fail the check when the major version is lower than 26. A failed version check SHALL map to exit code 2.

Refs: ADR-001

#### Scenario: SCN-DOC-002-1 Supported version
- **WHEN** the Mac runs macOS 26 or later
- **THEN** the line shows the version and passes

#### Scenario: SCN-DOC-002-2 Old version
- **WHEN** the version source reports macOS 15
- **THEN** the check fails with code 2

### Requirement: REQ-DOC-003 Microphone permission check
Doctor SHALL report the microphone permission state without a prompt. A denied or restricted state SHALL fail the check with exit code 3. A not determined state SHALL pass with a note that `notetaker record` asks for it.

Refs: ADR-005

#### Scenario: SCN-DOC-003-1 Granted
- **WHEN** the permission reader returns the state `authorized`
- **THEN** the line passes and shows the state

#### Scenario: SCN-DOC-003-2 Denied
- **WHEN** the permission reader returns the state `denied`
- **THEN** the line fails with code 3 and names the System Settings pane

#### Scenario: SCN-DOC-003-3 No prompt
- **WHEN** the permission state is not determined
- **THEN** doctor shows no prompt and the line passes with a note

### Requirement: REQ-DOC-004 Process tap check
Doctor SHALL create a process tap and a private aggregate device, then destroy both. When either creation fails, the check SHALL fail with exit code 4.

Refs: REQ-CAP-001, REQ-CAP-002, ADR-002

#### Scenario: SCN-DOC-004-1 Tap works
- **WHEN** tap and aggregate device creation succeed
- **THEN** the line passes and no tap or aggregate device remains after doctor exits

#### Scenario: SCN-DOC-004-2 Tap fails
- **WHEN** tap creation returns an error
- **THEN** the line fails with code 4 and shows the error status

### Requirement: REQ-DOC-005 Them self-test with a system sound
Doctor SHALL start Them capture, play `/usr/bin/afplay /System/Library/Sounds/Tink.aiff` in a separate process, and pass when the Them peak goes above 0.01. Otherwise the check SHALL fail with exit code 6 and name the system audio recording permission.

Refs: REQ-CAP-001, REQ-WD-001, ADR-003

#### Scenario: SCN-DOC-005-1 Tap hears Tink
- **WHEN** the Them peak during playback goes above 0.01
- **THEN** the line passes and shows the measured peak

#### Scenario: SCN-DOC-005-2 Silence from the tap
- **WHEN** the Them peak stays at or below 0.01 until playback ends plus one second
- **THEN** the line fails with code 6 and tells the user to allow system audio recording for the app that started the tool

#### Scenario: SCN-DOC-005-3 Player missing
- **WHEN** afplay cannot start
- **THEN** the line fails with code 6 and says that the self-test did not run

#### Scenario: SCN-DOC-005-4 Tap check failed first
- **WHEN** the process tap check failed
- **THEN** the self-test line shows "skipped" and the exit code stays 4

### Requirement: REQ-DOC-006 Default devices
Doctor SHALL print the names of the default input device and the default output device. This line is information only. It SHALL NOT change the exit code.

Refs: REQ-CAP-008

#### Scenario: SCN-DOC-006-1 Devices shown
- **WHEN** doctor runs on a Mac with a built-in microphone and speakers
- **THEN** it prints both device names

#### Scenario: SCN-DOC-006-2 No input device
- **WHEN** the Mac has no default input device
- **THEN** the line shows "none" for the input device and the exit code does not change
