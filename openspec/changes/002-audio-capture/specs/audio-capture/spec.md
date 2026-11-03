# Spec Delta

## Purpose

Capture the two meeting audio streams, Me from the microphone and Them from system audio, as aligned chunks that never block the audio thread.

## ADDED Requirements

### Requirement: REQ-CAP-001 Them capture with a process tap
The tool SHALL capture system audio as the Them stream with a global stereo process tap that excludes the tool's own process. The tool SHALL place the tap in a private aggregate device, because a tap alone delivers no buffers. Them holds all system audio, not only the meeting app.

Refs: ADR-002, ADR-005

#### Scenario: SCN-CAP-001-1 Them hears another process
- **WHEN** Them capture runs and another process plays sound
- **THEN** the Them stream delivers chunks with non-zero samples

#### Scenario: SCN-CAP-001-2 Tap leaves out own output
- **WHEN** the tool builds the tap description
- **THEN** the tap excludes the tool's own process and is private to the tool

#### Scenario: SCN-CAP-001-3 Tap creation fails
- **WHEN** the tap or the aggregate device cannot be created
- **THEN** capture start reports a tap error that maps to exit code 4

### Requirement: REQ-CAP-002 Teardown on every catchable exit
The tool SHALL destroy the aggregate device and the process tap on every exit path that the process can catch. This covers a normal end, an error, SIGINT, and SIGTERM.

Refs: REQ-CAP-001, ADR-002

#### Scenario: SCN-CAP-002-1 Normal end
- **WHEN** a command that started Them capture ends
- **THEN** the tool destroys the aggregate device and the tap before the process exits

#### Scenario: SCN-CAP-002-2 Signal during capture
- **WHEN** the process gets SIGINT or SIGTERM while Them capture runs
- **THEN** the tool destroys the aggregate device and the tap and then exits

#### Scenario: SCN-CAP-002-3 Error after creation
- **WHEN** a later start step fails after the tap exists
- **THEN** the tool destroys what it created before it reports the error

### Requirement: REQ-CAP-003 Me capture without voice processing
The tool SHALL capture the microphone as the Me stream from the input node of an audio engine on the default input device. The tool SHALL never enable voice processing on that engine, because voice processing can stop the process tap.

Refs: ADR-002, ADR-003

#### Scenario: SCN-CAP-003-1 Me hears the room
- **WHEN** Me capture runs and a person speaks into the microphone
- **THEN** the Me stream delivers chunks with non-zero samples

#### Scenario: SCN-CAP-003-2 Voice processing stays off
- **WHEN** Me capture starts or restarts
- **THEN** voice processing on the input node is off

### Requirement: REQ-CAP-004 Chunk stream per source
Each capture SHALL expose its stream as an asynchronous sequence of chunks. A chunk holds the samples, the sample rate, the channel count, and the host time of its first sample. Me and Them each have their own sequence.

Refs: REQ-CAP-005

#### Scenario: SCN-CAP-004-1 Chunk fields
- **WHEN** a consumer reads a chunk from either stream
- **THEN** the chunk gives its samples, sample rate, channel count, and first-sample host time

#### Scenario: SCN-CAP-004-2 Stream ends with capture
- **WHEN** a capture stops
- **THEN** its chunk sequence finishes and the consumer loop ends

### Requirement: REQ-CAP-005 Shared host-time reference
Both streams SHALL stamp chunks in host time. The recording start t0 SHALL be the host time when both captures run. Every relative time SHALL count from t0.

Refs: REQ-CAP-004

#### Scenario: SCN-CAP-005-1 Relative time of a chunk
- **WHEN** a chunk has a first-sample host time h
- **THEN** its relative time is h minus t0, in seconds

#### Scenario: SCN-CAP-005-2 Streams line up
- **WHEN** Me and Them both hear one short sound
- **THEN** the relative times of the two chunks that hold it differ by less than 100 ms

### Requirement: REQ-CAP-006 Non-blocking audio callback
The audio callback SHALL copy samples into a preallocated bounded ring buffer and return. It SHALL NOT lock, allocate, log, write files, or await. When the buffer is full, it SHALL drop the new samples and add their count to an overflow counter.

Refs: ADR-003

#### Scenario: SCN-CAP-006-1 Normal write
- **WHEN** the callback writes a block and the ring buffer has room
- **THEN** the consumer reads the same samples in the same order

#### Scenario: SCN-CAP-006-2 Full buffer
- **WHEN** the callback writes a block and the ring buffer has too little room
- **THEN** the callback drops the block, adds its sample count to the overflow counter, and returns

#### Scenario: SCN-CAP-006-3 Callback path is clean
- **WHEN** a reviewer or a test checks the callback source
- **THEN** the callback path has no lock, allocation, log call, file write, or await

### Requirement: REQ-CAP-007 Continuous timeline after drops
The consumer of each stream SHALL keep the timeline continuous. When samples drop on overflow, it SHALL insert zeros of the same length at the place of the drop and count the drop. Chunk host times then stay correct.

Refs: REQ-CAP-006, REQ-CAP-005

#### Scenario: SCN-CAP-007-1 Gap filled with zeros
- **WHEN** 480 samples drop between two written blocks
- **THEN** the consumer yields 480 zero samples between them and the dropped count goes up by 480

#### Scenario: SCN-CAP-007-2 Host time after a drop
- **WHEN** a chunk follows a filled gap
- **THEN** its host time equals the previous chunk's host time plus the duration of all samples in between

### Requirement: REQ-CAP-008 Device change handling
The tool SHALL observe changes of the default input device, the default output device, and the Me engine configuration. On a change it SHALL report the event, restart Me capture on the new default input, and keep the Me timeline continuous.

Refs: REQ-CAP-003, REQ-CAP-007

#### Scenario: SCN-CAP-008-1 New default input
- **WHEN** the default input device changes during capture
- **THEN** the tool reports the event with the new device name and Me chunks continue

#### Scenario: SCN-CAP-008-2 Timeline across a restart
- **WHEN** Me capture restarts after a change
- **THEN** the consumer fills the restart gap with zeros so relative times stay correct

#### Scenario: SCN-CAP-008-3 New default output
- **WHEN** the default output device changes during capture
- **THEN** the tool reports the event and Them capture keeps its timeline
