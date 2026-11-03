# Spec Delta

## Purpose

Show the live capture numbers of both streams for a short run, so a developer can see what macOS really delivers.

## ADDED Requirements

### Requirement: REQ-DBG-001 Timed capture run
`notetaker debug-audio [--seconds S]` SHALL run Me and Them capture for S seconds and then stop both. S is a whole number of 1 or more, with a default of 10. An invalid S SHALL be a usage error with exit code 1.

Refs: REQ-CAP-001, REQ-CAP-003, ADR-004

#### Scenario: SCN-DBG-001-1 Default length
- **WHEN** the user runs `notetaker debug-audio`
- **THEN** the run lasts 10 seconds and ends with both captures stopped

#### Scenario: SCN-DBG-001-2 Bad value
- **WHEN** the user runs `notetaker debug-audio --seconds 0`
- **THEN** the tool prints a usage error and exits 1

### Requirement: REQ-DBG-002 Per-second stream report
Once per second, debug-audio SHALL print one line for each stream. The line SHALL show the stream name, peak, RMS, callback count, non-zero sample count, overflow count, and watchdog state for that second.

Refs: REQ-WD-001, REQ-WD-002, REQ-CAP-006

#### Scenario: SCN-DBG-002-1 Line content
- **WHEN** one second of capture ends
- **THEN** the tool prints a Me line and a Them line, each with all seven fields

#### Scenario: SCN-DBG-002-2 Speech on Them
- **WHEN** another shell runs `say "testing one two three"` during the run
- **THEN** a Them line shows a peak above 0.01 and the state live

#### Scenario: SCN-DBG-002-3 Quiet Mac
- **WHEN** no app plays sound during the run
- **THEN** the Them lines show the state idle after the first N seconds

### Requirement: REQ-DBG-003 Device and route events
Debug-audio SHALL print a line for each device or route event that capture reports during the run. The line SHALL show the time relative to t0, the event kind, and the device name.

Refs: REQ-CAP-008

#### Scenario: SCN-DBG-003-1 Headphones connect
- **WHEN** the default output device changes during the run
- **THEN** the tool prints an event line with the new output device name

### Requirement: REQ-DBG-004 No file output
Debug-audio SHALL write no file. It SHALL keep no audio and no transcript on disk.

Refs: ADR-006

#### Scenario: SCN-DBG-004-1 Folder unchanged
- **WHEN** a run ends
- **THEN** the working folder and the default meetings folder hold the same files as before the run

### Requirement: REQ-DBG-005 Exit code and latency summary
Debug-audio SHALL exit 0 when both captures started, even when the signal is zero. When a capture cannot start, it SHALL exit with that failure's code. At the end it SHALL print chunk latency p50 and p95 for each stream.

Refs: REQ-CAP-004, ADR-003

#### Scenario: SCN-DBG-005-1 Silent but started
- **WHEN** both captures start and every sample is zero
- **THEN** the tool exits 0

#### Scenario: SCN-DBG-005-2 Tap cannot start
- **WHEN** the aggregate device cannot be created
- **THEN** the tool exits 4

#### Scenario: SCN-DBG-005-3 Latency summary
- **WHEN** the run ends
- **THEN** the tool prints p50 and p95 of the time from a chunk's last sample to its arrival at the consumer, per stream
