# ADR-007 The app has no LLM and makes no network call

Status: accepted. Date: 2026-11-03.

## Decision

- The app makes no network call. The OS may download the Apple speech model, and only `record` asks for it.
- The app has no LLM and no summary feature. The Claude Code command `/summarize-meeting` writes summaries.
- A unit test fails if any file under `Sources/` names a networking API.

## Why

- Meeting audio is sensitive. Local capture and local transcription keep it on the Mac.
- The summary step runs wherever the user runs Claude Code, under the user's own rules.

## Consequences

- Sharing, sync, and calendar features that need a server stay out of scope.
- EventKit is local, so a later calendar change is still allowed.

Refs: REQ-PKG-005, REQ-TRN-008, REQ-TRN-009
