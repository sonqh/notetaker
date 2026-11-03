# ADR-004 The CLI core comes first, the menu bar app follows

Status: accepted. Date: 2026-11-03.

## Decision

- `NotetakerKit` holds all logic: capture, watchdog, transcription, and the file writer.
- The `notetaker` CLI comes first. It drives every milestone gate.
- A menu bar app (`MenuBarExtra`) follows as a thin front end over the same kit and the same file format.
- This removes the old non-goal "A GUI or menu bar app".

## Why

- The risky parts are capture and speech. A terminal shows their numbers most plainly.
- The long-term goal is a personal app in the style of Granola. A menu bar app is the friendly way to start and stop.
- An `.app` bundle owns its permissions. A CLI binary may get them through the terminal app.

## Consequences

- Two front ends must stay in step. Both call the same kit, so the logic exists once.
- The menu bar app needs a bundle script and a signing identity (ADR-005).

Refs: REQ-APP-012
