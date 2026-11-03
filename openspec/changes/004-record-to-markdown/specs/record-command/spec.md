# Spec Delta

## Purpose

Defines the `notetaker record` lifecycle: checks before start, start, terminal output, notes on stdin, stop, signals, kept audio, warnings, and exit codes.

## ADDED Requirements

### Requirement: REQ-REC-001 Checks before start
Before t0 `record` SHALL check that the output folder is writable, or exit 8. It SHALL request the microphone permission when its state is not determined. It SHALL exit 3 when the user denies the permission. When the `en_US` model is missing, it SHALL ask the OS to install it through change 003 and print progress on stderr. It SHALL exit 7 when the install fails. It SHALL never change System Settings or TCC state.

Refs: ADR-005, ADR-007, ADR-008

#### Scenario: SCN-REC-001-1 Permission not determined
- **WHEN** the microphone permission state is not determined
- **THEN** `record` shows the standard system prompt and waits for the answer
- **AND** `record` continues when the user allows it

#### Scenario: SCN-REC-001-2 Permission denied
- **WHEN** the user denied the microphone permission earlier or denies the prompt now
- **THEN** `record` prints a message that names the permission on stderr and exits 3
- **AND** the output folder holds no new file

#### Scenario: SCN-REC-001-3 Model missing
- **WHEN** the `en_US` speech model is not installed
- **THEN** `record` asks the OS to install it and prints the progress on stderr
- **AND** `record` starts the recording when the install completes

#### Scenario: SCN-REC-001-4 Model install fails
- **WHEN** the OS reports that the model install failed
- **THEN** `record` prints the reason on stderr and exits 7

#### Scenario: SCN-REC-001-5 Folder not writable
- **WHEN** the output folder cannot be created or written
- **THEN** `record` prints the path on stderr and exits 8 before it starts any capture

### Requirement: REQ-REC-002 Start the recording
`record` SHALL start the Me and Them captures and one analyzer per stream, and set t0. It SHALL then choose the file name, write the first partial file, and print the file path on stderr. When the watchdog of change 002 reports a failure at start, `record` SHALL exit with the code of that stream. It SHALL leave no `.md` and no `.md.partial` file.

Refs: REQ-OUT-001, REQ-OUT-007, ADR-002, ADR-003, ADR-004

#### Scenario: SCN-REC-002-1 Normal start
- **WHEN** both captures deliver signal in the first N seconds
- **THEN** `<name>.md.partial` exists with the front matter and the `## Transcript` heading
- **AND** stderr shows the path of the final `.md` file

#### Scenario: SCN-REC-002-2 Me silent at start
- **WHEN** the Me stream has only zero samples for the first N seconds
- **THEN** `record` exits 5
- **AND** the output folder holds no new `.md` or `.md.partial` file

### Requirement: REQ-REC-003 Terminal output
`record` SHALL print each finalized segment on stdout as `[HH:MM:SS] <Speaker>: <text>` when it finalizes. When stdout is a terminal, it SHALL show the newest volatile result on one line. It rewrites that line in place and clears it before a finalized line. It SHALL print warnings and recovery notes on stderr. Volatile text SHALL never reach the file.

Refs: REQ-OUT-007, ADR-006

#### Scenario: SCN-REC-003-1 Finalized line
- **WHEN** a Me segment that starts at 4 s finalizes with text `Morning`
- **THEN** stdout gets the line `[00:00:04] Me: Morning`

#### Scenario: SCN-REC-003-2 Volatile line
- **WHEN** stdout is a terminal and three volatile results arrive before the final one
- **THEN** the terminal shows one line that changes three times
- **AND** `record` clears the line before the finalized line prints

#### Scenario: SCN-REC-003-3 Not a terminal
- **WHEN** stdout is a pipe or a file
- **THEN** stdout holds only finalized lines and no volatile text and no control characters

### Requirement: REQ-REC-004 Notes and stop from stdin
During a recording `record` SHALL read stdin by line. The line `/stop` SHALL start a clean stop. Any other line that is not empty after trimming whitespace SHALL become a note, stamped with the time the line arrived relative to t0. Empty lines SHALL be ignored. The end of stdin SHALL end note input and SHALL not stop the recording.

Refs: REQ-OUT-004, REQ-REC-006

#### Scenario: SCN-REC-004-1 Note typed
- **WHEN** the user types `ask about budget` and Enter at 00:03:12
- **THEN** the next partial file holds `[00:03:12] ask about budget` under `## Notes`

#### Scenario: SCN-REC-004-2 Empty line
- **WHEN** the user presses Enter on an empty line or a line of spaces
- **THEN** the file gets no new note and the recording continues

#### Scenario: SCN-REC-004-3 Stop command
- **WHEN** the user types `/stop` and Enter
- **THEN** `record` stops cleanly as for the first SIGINT
- **AND** `/stop` is not a note

#### Scenario: SCN-REC-004-4 End of stdin
- **WHEN** stdin is `/dev/null` or closes during a recording
- **THEN** the recording continues until SIGINT

### Requirement: REQ-REC-005 SIGINT handling
The first SIGINT SHALL start a clean stop. A second SIGINT during that stop SHALL end the process at once with the default SIGINT status and keep the newest `<name>.md.partial`. The signal handler SHALL only set a flag or resume a waiter. It SHALL do no file or audio work.

Refs: REQ-REC-006, REQ-OUT-007

#### Scenario: SCN-REC-005-1 Ctrl-C
- **WHEN** the user presses Ctrl-C once
- **THEN** `record` stops cleanly, `<name>.md` exists, and `<name>.md.partial` does not exist

#### Scenario: SCN-REC-005-2 Second Ctrl-C
- **WHEN** the user presses Ctrl-C a second time before the clean stop ends
- **THEN** the process ends at once
- **AND** `<name>.md.partial` exists and is complete

### Requirement: REQ-REC-006 Clean stop and exit code
A clean stop SHALL stop both captures and let both analyzers finalize pending audio. Then it SHALL write the final file, delete the partial file, and print the file path on stdout. It SHALL exit 0 with no warnings and 9 with at least one warning. When the final write fails, it SHALL keep the partial file and exit 8.

Refs: REQ-OUT-008, ADR-004

#### Scenario: SCN-REC-006-1 Exit 0
- **WHEN** a recording with no warning stops cleanly
- **THEN** the last stdout line is the path of `<name>.md` and the exit code is 0

#### Scenario: SCN-REC-006-2 Exit 9
- **WHEN** a recording with one or more warnings stops cleanly
- **THEN** `<name>.md` holds the warnings and the exit code is 9

#### Scenario: SCN-REC-006-3 Final write fails
- **WHEN** the write of `<name>.md` fails
- **THEN** stderr names the error, `<name>.md.partial` stays, and the exit code is 8

### Requirement: REQ-REC-007 Stream failure during a recording
When a stream fails after the start window, `record` SHALL keep the other stream recording. It SHALL add a `warnings` entry that starts with the time and the stream, print it on stderr, and rewrite the partial file. When the stream recovers, it SHALL print a recovery note on stderr.

Refs: REQ-OUT-003, ADR-003

#### Scenario: SCN-REC-007-1 Them fails
- **WHEN** Them delivers no callbacks for N seconds at 00:12:35
- **THEN** Me lines keep arriving in the file
- **AND** the warnings hold `"00:12:40 Them: no callbacks for 5 s"` when N is 5

#### Scenario: SCN-REC-007-2 Them recovers
- **WHEN** Them signal returns after a warning
- **THEN** stderr gets a recovery note and Them lines arrive again

### Requirement: REQ-REC-008 Kept audio
With `--keep-audio`, `record` SHALL write `<name>.me.caf` and `<name>.them.caf` next to the Markdown file, as 32-bit float PCM at each stream's native sample rate. A consumer task SHALL write them, never the audio callback. Without the flag, `record` SHALL write no audio file.

Refs: REQ-OUT-002, ADR-002

#### Scenario: SCN-REC-008-1 Keep audio
- **WHEN** a 60-second recording runs with `--keep-audio` and stops cleanly
- **THEN** both `.caf` files exist with the same base name as the `.md` file
- **AND** each file is 32-bit float PCM and lasts about 60 seconds

#### Scenario: SCN-REC-008-2 No flag
- **WHEN** a recording runs without `--keep-audio`
- **THEN** the output folder holds no new `.caf` file
