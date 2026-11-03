# Spec Delta

## Purpose

Turn the Me and Them chunk streams into a live English transcript on the Mac. Times sit on the t0 timeline, and the tool makes no network call.

## ADDED Requirements

### Requirement: REQ-TRN-001 One analyzer per stream
The tool SHALL run one on-device speech analyzer for Me and one for Them. Each analyzer SHALL use a speech transcriber for `en_US` with volatile results, fast results, audio time ranges, and word confidence. Each analyzer SHALL read only the chunks of its own stream.

Refs: REQ-CAP-004, ADR-001, ADR-007, ADR-008

#### Scenario: SCN-TRN-001-1 Two analyzers run at once
- **WHEN** `record` starts with both captures live
- **THEN** two analyzers run at the same time and each yields results for its own stream only

#### Scenario: SCN-TRN-001-2 Transcriber options
- **WHEN** the tool builds a transcriber
- **THEN** it uses locale `en_US`, the `timeIndexedProgressiveTranscription` preset, and the `transcriptionConfidence` attribute

#### Scenario: SCN-TRN-001-3 Words from each stream
- **WHEN** a known sentence plays through the speakers and a person speaks a different sentence into the microphone
- **THEN** the Them results hold the played words and the Me results hold the spoken words

### Requirement: REQ-TRN-002 Format conversion off the callback
A consumer task SHALL convert each chunk to the format that the analyzer reports as its best available format. The audio callback SHALL do no transcription work. The tool SHALL use no Speech API that needs macOS 27 without an availability check.

Refs: REQ-CAP-006, ADR-001, ADR-003

#### Scenario: SCN-TRN-002-1 Converted format
- **WHEN** a 48 kHz stereo float chunk arrives and the analyzer asks for 16 kHz mono 16-bit
- **THEN** the analyzer input holds 16 kHz mono 16-bit samples of the same duration within one output sample

#### Scenario: SCN-TRN-002-2 Converter state across chunks
- **WHEN** ten consecutive chunks of a 1 kHz sine pass through the converter
- **THEN** the output has no gap or repeat at chunk edges, and its total sample count matches the input duration within one output sample

#### Scenario: SCN-TRN-002-3 No macOS 27 symbol
- **WHEN** the package builds with deployment target macOS 26
- **THEN** the build has no availability error and no source file uses `AnalyzerInputConverter` or `ignoresResourceLimits` outside an `#available` check

### Requirement: REQ-TRN-003 Result times on the t0 timeline
Each analyzer input SHALL carry a buffer start time equal to the chunk's first-sample host time minus t0. A segment's start and end SHALL then equal its result time range, in seconds from t0. Samples before t0 SHALL not reach the analyzer.

Refs: REQ-CAP-005, REQ-CAP-007

#### Scenario: SCN-TRN-003-1 Chunk start time
- **WHEN** a chunk's first sample has host time t0 plus 2.5 s
- **THEN** its analyzer input has buffer start time 2.5 s

#### Scenario: SCN-TRN-003-2 Samples before t0
- **WHEN** a chunk starts 0.1 s before t0 and lasts 0.3 s
- **THEN** the tool drops its first 0.1 s and the input starts at 0 s with 0.2 s of samples

#### Scenario: SCN-TRN-003-3 Time of a played sentence
- **WHEN** a sentence starts to play 10 s after t0
- **THEN** the first Them segment of that sentence starts between 9.5 s and 10.5 s

### Requirement: REQ-TRN-004 Segment value
A finalized result SHALL become one segment. A segment holds the speaker (Me or Them), and the start and end in seconds from t0. It also holds the trimmed text, and each word with its confidence when the result reports one. A result with empty text SHALL make no segment.

Refs: REQ-TRN-003, ADR-006

#### Scenario: SCN-TRN-004-1 Fields of a segment
- **WHEN** the Them analyzer finalizes " We meet on Tuesday." over 2.58 s to 4.33 s with four word confidences
- **THEN** the segment is Them, start 2.58, end 4.33, text "We meet on Tuesday.", and four words with their confidences

#### Scenario: SCN-TRN-004-2 Empty result
- **WHEN** a finalized result holds only white space
- **THEN** the tool makes no segment

#### Scenario: SCN-TRN-004-3 No confidence reported
- **WHEN** a finalized result has no confidence attribute on a word
- **THEN** the segment keeps the word and marks its confidence as unknown

### Requirement: REQ-TRN-005 Volatile and finalized results
Each stream SHALL yield volatile updates and finalized segments on one ordered sequence. A volatile update SHALL replace the previous volatile text of its stream. A finalized segment SHALL clear the volatile text that its range covers. Finalized segments of one stream SHALL come in start order.

Refs: REQ-TRN-004

#### Scenario: SCN-TRN-005-1 Volatile replaced
- **WHEN** a stream yields the volatile texts "we meet" and then "we meet on Tuesday"
- **THEN** the current volatile text of the stream is "we meet on Tuesday"

#### Scenario: SCN-TRN-005-2 Final clears volatile
- **WHEN** a finalized segment covers the range of the current volatile text
- **THEN** the stream yields the segment and its volatile text becomes empty

#### Scenario: SCN-TRN-005-3 Start order
- **WHEN** the engine finalizes a result that starts before the last finalized result of the same stream
- **THEN** the tool yields both in start order and reports the reorder in the debug count

### Requirement: REQ-TRN-006 Live terminal transcript
`record` SHALL print each finalized segment on stdout as one line `[HH:MM:SS] Speaker: text`, and flush it at once. On a terminal it SHALL show the current volatile text on one updating line. It SHALL print warnings on stderr. It SHALL write no transcript file in this change.

Refs: REQ-TRN-005, ADR-004

#### Scenario: SCN-TRN-006-1 Finalized line
- **WHEN** a Me segment starts at 75.2 s with text "Can you hear me?"
- **THEN** stdout gets the line `[00:01:15] Me: Can you hear me?`

#### Scenario: SCN-TRN-006-2 Not a terminal
- **WHEN** stdout is a pipe
- **THEN** the tool prints finalized lines only and no volatile line

#### Scenario: SCN-TRN-006-3 Warning channel
- **WHEN** a stream warning occurs
- **THEN** the warning goes to stderr and stdout holds only transcript lines

### Requirement: REQ-TRN-007 Speech gate on Me
The Me analyzer SHALL include a speech detector, so room noise does not become text. The Them analyzer SHALL include one only if the silence measurement shows invented Them text. `docs/findings.md` SHALL record the measured choice.

Refs: REQ-TRN-001, ADR-003, ADR-008

#### Scenario: SCN-TRN-007-1 Me modules
- **WHEN** the tool builds the Me analyzer
- **THEN** its modules are the transcriber and a speech detector

#### Scenario: SCN-TRN-007-2 Them choice recorded
- **WHEN** the silence measurement for Them ends
- **THEN** `docs/findings.md` states whether Them uses a detector and gives the measured count behind the choice

#### Scenario: SCN-TRN-007-3 No speech, no Me segment
- **WHEN** the detector reports no speech over a range
- **THEN** the tool yields no Me segment that lies fully inside that range

### Requirement: REQ-TRN-008 Speech model check in doctor
`doctor` SHALL report whether the OS has the `en_US` transcriber model. It SHALL never request, reserve, or download a model. If the model is missing, the check SHALL fail with exit code 7 and say that `notetaker record` asks the OS to install it.

Refs: REQ-DOC-001, REQ-PKG-004, ADR-005, ADR-007

#### Scenario: SCN-TRN-008-1 Model installed
- **WHEN** the installed locales of the transcriber hold `en_US`
- **THEN** doctor prints a passing row for the English speech model

#### Scenario: SCN-TRN-008-2 Model missing
- **WHEN** the installed locales do not hold `en_US`
- **THEN** doctor prints a failing row that names `notetaker record`, and exits 7 if no earlier check failed

#### Scenario: SCN-TRN-008-3 No install call
- **WHEN** doctor runs the model check against a fake model inventory
- **THEN** the fake records zero install, reserve, and download calls

### Requirement: REQ-TRN-009 Model install in record
If the `en_US` model is missing, `record` SHALL ask the OS to install it before it starts the analyzers, and print progress on stderr. If the install fails, `record` SHALL exit 7. This is the only path that can cause a download.

Refs: REQ-TRN-008, ADR-005, ADR-007

#### Scenario: SCN-TRN-009-1 Install with progress
- **WHEN** `record` starts and the fake inventory reports the model missing
- **THEN** the tool asks for an install, prints progress on stderr, and starts the analyzers after the install ends

#### Scenario: SCN-TRN-009-2 Install fails
- **WHEN** the install request throws
- **THEN** `record` prints the error on stderr and exits 7

#### Scenario: SCN-TRN-009-3 Model present
- **WHEN** `record` starts and the OS has the model
- **THEN** the tool makes no install request

### Requirement: REQ-TRN-010 Analyzer errors become warnings
When an analyzer of one stream throws, the tool SHALL print a warning that names the stream, the time from t0, and the error. That stream's transcription SHALL stop, its chunks SHALL still drain, and the other stream SHALL go on. Resource errors SHALL get a plain message.

Refs: REQ-TRN-001, ADR-003

#### Scenario: SCN-TRN-010-1 One stream fails
- **WHEN** the fake Them engine throws after 30 s
- **THEN** stderr gets a Them warning with time 00:00:30, and Me segments keep coming

#### Scenario: SCN-TRN-010-2 Resource limit
- **WHEN** an engine throws `insufficientResources` or `tooManyAssetLocalesAllocated`
- **THEN** the warning names the cause in plain words and suggests closing other apps that use speech recognition

#### Scenario: SCN-TRN-010-3 Warning list
- **WHEN** a stream warning occurs
- **THEN** the tool adds the warning to the session warning list that the file writer reads later

### Requirement: REQ-TRN-011 Testable engine seam
The transcription logic SHALL reach the speech framework only through a small engine protocol. Unit tests SHALL drive conversion setup, time mapping, segment making, ordering, and error handling with a fake engine. The tests SHALL need no audio hardware and no speech model.

Refs: REQ-TRN-003, REQ-TRN-005, ADR-004

#### Scenario: SCN-TRN-011-1 Fake results
- **WHEN** a test feeds scripted volatile and final results into the fake engine
- **THEN** the test reads the expected segments and volatile updates from the stream

#### Scenario: SCN-TRN-011-2 Fake time mapping
- **WHEN** a test feeds a chunk at t0 plus 4 s to the fake engine
- **THEN** the fake engine receives buffer start time 4 s

#### Scenario: SCN-TRN-011-3 Runs anywhere
- **WHEN** `swift test` runs on a Mac with no microphone permission
- **THEN** every transcription unit test passes

### Requirement: REQ-TRN-012 Measured quality gate
M2 SHALL pass only on measured numbers. Word recall of a known English paragraph of at least 150 words SHALL be at least 90%. The tool SHALL report the Me segments in 60 s of room silence, with target 0. It SHALL report the p50 and p95 latency from speech end to finalized segment. `docs/findings.md` SHALL hold every number.

Refs: REQ-PRB-005, ADR-003, ADR-008

#### Scenario: SCN-TRN-012-1 Played paragraph
- **WHEN** `say -v Samantha` plays the fixture paragraph during `record`
- **THEN** at least 90% of the paragraph words appear in the Them segments after lower case and punctuation removal. The script prints the real share

#### Scenario: SCN-TRN-012-2 Room silence
- **WHEN** a person stays silent in a normal room for 60 s with the microphone on
- **THEN** the script prints the count of Me segments and findings records it

#### Scenario: SCN-TRN-012-3 Latency
- **WHEN** the paragraph run ends
- **THEN** the script prints p50 and p95 of arrival time minus segment end time for all finalized segments

### Requirement: REQ-TRN-013 Engine comparison on Me
The project SHALL compare the speech transcriber with the dictation transcriber, with punctuation and the far-field hint, on Me audio. The comparison SHALL report word recall, WER, punctuation, and latency for the same files. ADR-008 SHALL record the choice and its numbers.

Refs: REQ-PRB-004, REQ-PRB-005, ADR-008

#### Scenario: SCN-TRN-013-1 Same input
- **WHEN** the comparison runs
- **THEN** both engines read the same recorded Me files and the same reference text

#### Scenario: SCN-TRN-013-2 Report
- **WHEN** the comparison ends
- **THEN** `docs/findings.md` holds a table with recall, WER, punctuation, and p50 latency for each engine

#### Scenario: SCN-TRN-013-3 Decision recorded
- **WHEN** the table exists
- **THEN** ADR-008 names the engine for each stream and cites the table
