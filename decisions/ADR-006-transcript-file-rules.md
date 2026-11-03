# ADR-006 The transcript file is plain Markdown that survives a crash

Status: accepted. Date: 2026-11-03.

## Decision

- One UTF-8 Markdown file per meeting, with front matter, an optional `## Notes` section, and a `## Transcript` section.
- The file holds only finalized results. Volatile results show only in the terminal or the popover.
- After each finalized segment, the app rewrites `<name>.md.partial` in full: it writes a temporary file and renames it.
- A clean stop writes the `.md` file and removes the partial file.
- The user's own notes carry a timestamp and sit above the transcript.

## Why

- Claude Code reads plain Markdown well. The app needs no sidecar and no database.
- A full rewrite with a rename always leaves a complete file, also after `kill -9`.
- The full rewrite sorts every segment, so a late segment from one stream lands in the right place.
- Granola's main value is the user's own notes next to the transcript.

## Consequences

- The rewrite cost grows with the meeting length. A long meeting is still small text, but M3 measures it.

Refs: REQ-OUT-004, REQ-OUT-007, REQ-OUT-008
