# Spec Delta

## Purpose

Measure the signal of each audio stream and name its state, so the tool can tell normal silence from a broken capture.

## ADDED Requirements

### Requirement: REQ-WD-001 Level meter per chunk
The level meter SHALL compute three values for each chunk over all channels. They are the peak absolute sample value, the RMS value, and the count of samples that are not exactly zero. It SHALL run off the audio callback.

Refs: ADR-003

#### Scenario: SCN-WD-001-1 Known signal
- **WHEN** the meter reads a chunk of a full-scale sine wave
- **THEN** the peak is 1.0 and the RMS is 0.707, each within 0.001, and the non-zero count is greater than 0

#### Scenario: SCN-WD-001-2 Digital silence
- **WHEN** the meter reads a chunk of exact zeros
- **THEN** the peak is 0, the RMS is 0, and the non-zero count is 0

#### Scenario: SCN-WD-001-3 Stereo chunk
- **WHEN** the meter reads a stereo chunk with signal on one channel only
- **THEN** the peak and the non-zero count include that channel

### Requirement: REQ-WD-002 Watchdog window
The watchdog SHALL judge each stream over a window of N seconds. N is a whole number from 1 to 60, with a default of 5. "At start" SHALL mean the first N seconds after t0. Before a stream has signal or a full window, its state SHALL be pending.

Refs: REQ-CAP-005, ADR-003

#### Scenario: SCN-WD-002-1 Default window
- **WHEN** the caller gives no N
- **THEN** the watchdog uses N = 5

#### Scenario: SCN-WD-002-2 Out of range
- **WHEN** the caller gives N = 0 or N = 61
- **THEN** the watchdog setup fails with a usage error that maps to exit code 1

#### Scenario: SCN-WD-002-3 Verdict at start
- **WHEN** a stream reaches a dead state in the first window after t0
- **THEN** the watchdog marks the event as at start

### Requirement: REQ-WD-003 Dead on no callbacks
The watchdog SHALL set a stream to dead when the stream delivers no callbacks for N seconds. This rule applies to Me and to Them.

Refs: REQ-WD-002

#### Scenario: SCN-WD-003-1 No callbacks at start
- **WHEN** a stream delivers no chunk in the first N seconds after t0
- **THEN** its state is dead, marked at start

#### Scenario: SCN-WD-003-2 Callbacks stop later
- **WHEN** a live stream delivers no chunk for N seconds
- **THEN** its state is dead, not marked at start

### Requirement: REQ-WD-004 Me dead on all zeros
The watchdog SHALL set Me to dead when every Me sample in the last N seconds is exactly zero. A room microphone always has some noise, so exact zeros on Me mean a broken capture.

Refs: REQ-WD-002, REQ-CAP-003

#### Scenario: SCN-WD-004-1 Me zeros at start
- **WHEN** Me delivers only zero samples for the first N seconds
- **THEN** the Me state is dead, marked at start

#### Scenario: SCN-WD-004-2 Me with room noise
- **WHEN** Me delivers low but non-zero samples
- **THEN** the Me state is live

### Requirement: REQ-WD-005 Them idle on zeros with no other output
The watchdog SHALL set Them to idle when every Them sample in the last N seconds is zero and no other process runs audio output. Idle is normal and SHALL cause no warning.

Refs: REQ-WD-006, ADR-002

#### Scenario: SCN-WD-005-1 Quiet machine
- **WHEN** Them delivers zeros for N seconds and no process other than this one has its output running flag set
- **THEN** the Them state is idle and the watchdog emits no warning

### Requirement: REQ-WD-006 Them suspect on zeros with other output
The watchdog SHALL set Them to suspect when every Them sample in the last N seconds is zero while another process runs audio output. A suspect state SHALL emit a warning that names the likely missing system audio recording permission. It SHALL NOT end the command.

Refs: REQ-WD-005, ADR-003

#### Scenario: SCN-WD-006-1 Other app plays
- **WHEN** Them delivers zeros for N seconds and another process has its output running flag set
- **THEN** the Them state is suspect and the warning names the system audio recording permission

#### Scenario: SCN-WD-006-2 Own output ignored
- **WHEN** only this process has its output running flag set
- **THEN** the watchdog treats it as no other output, and Them zeros give idle

### Requirement: REQ-WD-007 Live state and recovery
The watchdog SHALL set a stream to live when it delivers a non-zero sample. When a stream returns to live after a dead or suspect warning, the watchdog SHALL emit one recovery note for that stream.

Refs: REQ-WD-003, REQ-WD-006

#### Scenario: SCN-WD-007-1 Normal signal
- **WHEN** both streams deliver non-zero samples
- **THEN** both states are live and the watchdog emits no warning

#### Scenario: SCN-WD-007-2 Recovery after a warning
- **WHEN** Me was dead and then delivers non-zero samples
- **THEN** the Me state is live and the watchdog emits one recovery note

#### Scenario: SCN-WD-007-3 No note after idle
- **WHEN** Them goes from idle to live
- **THEN** the watchdog emits no recovery note
