# Proposal

## Why

Milestones M0 to M2 give two live audio streams, a watchdog, and finalized speech segments. Nothing writes them to disk yet. The user needs one Markdown file per meeting that survives a crash and that Claude Code can summarize. This change is milestone M3. It turns `notetaker record` into a full command.

## What Changes

- Add the transcript file. It covers the name, slug, collisions, front matter, a `## Notes` section, a `## Transcript` section, ordering, merge, and low-confidence marks.
- Add new front matter fields `speakers` and `audio_route`.
- Add notes typed on stdin during a recording. Each non-empty line becomes a note with the time the user typed it.
- Stop on Ctrl-C (SIGINT) or on the line `/stop`. This replaces "stops on Enter" in `docs/spec.md`.
- Rewrite `<name>.md.partial` after each finalized segment with a temp file and a rename. A clean stop writes the final file and removes the partial file.
- Add the record lifecycle. It covers the terminal output, the mic permission request, the model install through change 003, `--keep-audio`, mid-recording stream failure, and exit codes 0, 8, and 9.
- Plan an update to `docs/spec.md`, `README.md`, and `.claude/commands/summarize-meeting.md` for the new format and the new stop rule.

Out of scope:

- A GUI or a menu bar app.
- Summaries or any LLM feature. Summaries stay in `/summarize-meeting`.
- Calendar integration.
- Duplicate removal when the mic hears the speakers.
- Speaker identification beyond Me and Them.

## Capabilities

### New Capabilities

- `transcript-file`: The Markdown file that one recording writes. It covers name, front matter, notes, transcript lines, the partial file, and the final file. Code OUT.
- `record-command`: The `notetaker record` lifecycle. It covers start, terminal output, stdin notes, stop, SIGINT, permissions, model install, kept audio, warnings, and exit codes. Code REC.

### Modified Capabilities

None.

## Impact

- New code in `NotetakerKit`: a pure file writer and a record session that joins capture (change 002) and transcription (change 003).
- The `notetaker` executable wires `record` to the session. Argument parsing and the exit code table come from change 001.
- New unit tests in `NotetakerKitTests`. New gate script `scripts/gates/004-record.sh` for the dev Mac with audio.
- Docs: `docs/spec.md`, `README.md`, and `.claude/commands/summarize-meeting.md` get the new format and stop rule.
- No new dependency. No network call.
