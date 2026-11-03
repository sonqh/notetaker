# ADR-003 Measured signal decides if capture works

Status: accepted. Date: 2026-11-03.

## Decision

- The app never trusts an audio API return code alone. It measures peak, RMS, callback count, and non-zero samples.
- A watchdog gives each stream a state: live, idle, suspect, or dead.
- Them with exact zeros is idle when no other process plays audio. It is suspect when another process runs audio output.
- `doctor` and the first-run window play a system sound in a separate process. The check passes only when the tap hears it.

## Why

- macOS can return success and deliver silence. Known causes are a denied permission, the wrong device, and the echo canceller.
- No public API reports the system audio permission state.
- A quiet meeting also gives exact zeros on Them, so zeros alone cannot mean failure.

## Consequences

- Some apps keep audio output running while they play silence. The suspect state can then be a false alarm. M1 measures how often.
- An MDM policy that denies audio capture shows up as a failed self-test. The app tells the user to ask IT.

Refs: REQ-WD-005, REQ-WD-006, REQ-DOC-005, REQ-APP-006
