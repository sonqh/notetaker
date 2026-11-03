# Design

## Context

Change 001 gives the package, argument parsing, and the exit code table. Change 002 gives the Me and Them chunk streams, t0, and watchdog events. Change 003 gives finalized segments with speaker, start, end, text, and word confidence, plus volatile results. This change joins them into `notetaker record` and writes the Markdown file.

The file rules come from `docs/spec.md` and ADR-006. The owner added four things after `docs/spec.md`: the `speakers` and `audio_route` fields, a `## Notes` section, `word[?]` marks, and stop by `/stop` on stdin in place of Enter.

The tool must stay crash safe. A `kill -9` at any moment must leave a readable file. The audio callback must never block, and the tool makes no network call.

## Goals / Non-Goals

**Goals:**

- A pure writer that turns a recording state into the file text. Unit tests cover it with no audio hardware.
- A partial file that is complete after every rewrite.
- A record session that handles notes, stop, signals, warnings, kept audio, and exit codes.
- A gate script that proves the file on the dev Mac with real audio.

**Non-Goals:**

- A GUI, a menu bar app, or a notepad window.
- Summaries or any LLM feature.
- Calendar data in the file.
- Duplicate removal when the mic hears the speakers.
- Edits to a file after the recording ends.

## Decisions

- The writer is a pure function from a `TranscriptDocument` value to a `String`. The value holds the metadata, the notes, the segments, and the warnings. File access sits in a small store type behind a protocol. Tests then check exact text without a disk or a microphone. Refs: REQ-OUT-003, REQ-OUT-004, REQ-OUT-005, ADR-004
- Each rewrite renders the whole file from all segments sorted again. An append-only file cannot place a late segment. A 90-minute meeting has about 2,000 segments and under 300 KB of text, so a full render stays cheap. Refs: REQ-OUT-005, REQ-OUT-007, ADR-006
- The sort key is start time, then speaker with Me before Them, then end time. The merge joins texts with one space. The merge runs after the sort on every render, so a late segment can split or join lines correctly. Refs: REQ-OUT-005
- Timestamps use whole seconds rounded down from the time relative to t0. The hour field has two digits or more. Refs: REQ-OUT-004, REQ-OUT-005
- An atomic rewrite writes `<name>.md.partial.tmp` in the output folder, calls `fsync`, and then calls `rename(2)` over the target. The temp file stays in the same folder so that the rename stays on one volume. `Data.write(options: .atomic)` can put its temp file elsewhere, so the store does not use it. Refs: REQ-OUT-007, REQ-OUT-008
- One writer actor owns the document state. Segments, notes, and warnings reach it on one `AsyncStream`. When events arrive during a rewrite, the actor applies them all and then writes once. No audio thread ever touches the writer. Refs: REQ-OUT-007, REQ-REC-003
- The session fixes the name at t0. A name is in use when `.md` or `.md.partial` exists, and also `.me.caf` or `.them.caf` with `--keep-audio`. The first partial file opens with `O_CREAT | O_EXCL`, so two runs in the same minute cannot share a name. Refs: REQ-OUT-001, REQ-OUT-002
- YAML output stays small and hand-written. The title is a plain scalar when it holds only letters, digits, spaces, and `-_.,()`. Otherwise it is a double-quoted string with `\` and `"` escaped. Warnings are always double-quoted. `speakers` is a fixed two-key map. Refs: REQ-OUT-003
- `audio_route` comes from the default output device at t0. The built-in speaker data source, HDMI, DisplayPort, and AirPlay give `speakers`. The built-in headphone data source, Bluetooth, and USB give `headphones`. An unknown device gives `speakers`, because that value warns the reader that Me can hold Them audio. The pure mapping has unit tests. Refs: REQ-OUT-003, ADR-002
- The confidence threshold is one constant in code. Its value and its source line come from the M2 entry in `docs/findings.md`. When change 003 gives no word confidence, the writer adds no mark. Refs: REQ-OUT-006, ADR-008
- A dedicated thread reads stdin by line with a blocking read. It sends each line with its arrival host time on an `AsyncStream`. The session trims the line, ignores it when empty, stops on `/stop`, and else adds a note. End of stdin ends the thread and leaves the recording running. Refs: REQ-REC-004
- SIGINT uses `signal(SIGINT, SIG_IGN)` and a `DispatchSourceSignal`. The first event starts the clean stop. The second event sets `SIG_DFL` and calls `raise(SIGINT)`. The partial file is complete after each rename, so the process can end at any point. Refs: REQ-REC-005, REQ-OUT-007
- The terminal shows the volatile line only when `isatty(1)` is true. It uses `\r` and the ANSI code that clears a line. Warnings and progress go to stderr. Finalized lines go to stdout in the line format of the file, without the merge. Refs: REQ-REC-003
- The clean stop takes the stop time when the stop starts, and `duration` uses that time. Then it stops both captures. It tells both analyzers to finalize to the end of input and waits for the writer to drain. Last, it writes the final file and deletes the partial file. Refs: REQ-REC-006, REQ-OUT-008
- With `--keep-audio`, a fan-out copies each chunk to a second consumer task. That task writes the chunk with `AVAudioFile` as float32 CAF at the native rate. The audio callback and the ring buffer stay as change 002 built them. Refs: REQ-REC-008, ADR-002
- The session maps each watchdog event from change 002 to a warning string `HH:MM:SS <Stream>: <reason>` and to one stderr line. A recovery event prints a stderr note and adds no warning. Refs: REQ-REC-007, ADR-003
- The checks before t0 run in a fixed order: output folder, microphone permission, speech model. A cheap check runs before a slow model install. The model install uses the change 003 entry point and adds no new network code. Refs: REQ-REC-001, ADR-005, ADR-007
- `scripts/gates/004-record.sh` runs the hardware gate on the dev Mac. It plays text with `say` and feeds notes and `/stop` through a FIFO on stdin. It sends `kill -9` and `kill -INT`. It prints the file head, the line count, and the file list. Refs: REQ-REC-002, REQ-REC-004, REQ-REC-005, REQ-OUT-007, ADR-003

## Risks / Trade-offs

- Typed note text and the volatile line share one terminal line. The volatile rewrite can hide what the user types. Mitigation: the session clears the volatile line after each note it reads, and the note is in the partial file at once. A later GUI ends the issue.
- `~/Documents` can sync to iCloud Drive. Each rename then makes a new upload. Mitigation: `--out` sets a local folder, and `README.md` names the trade-off.
- A second SIGINT loses segments that finalized after the last rename. Mitigation: the writer renames after each event, so the loss is at most the events of one rewrite.
- Bluetooth speakers read as `headphones`. Mitigation: the value is a hint for the summary, and the mapping can change after M3 measurements.
- The threshold for `[?]` is not known until M2 writes it. Mitigation: no marks until `docs/findings.md` holds the value.
- A full rewrite on each event costs more as the meeting grows. Mitigation: the actor coalesces events, and the gate measures the rewrite time on a 5-minute run.
