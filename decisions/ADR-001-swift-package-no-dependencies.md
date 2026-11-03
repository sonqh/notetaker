# ADR-001 One Swift package with no third-party dependency

Status: accepted. Date: 2026-11-03.

## Decision

- The app is one Swift package with the targets `NotetakerKit`, `notetaker`, and the test target. The menu bar app joins it later as another thin target.
- The package uses no third-party dependency. Argument parsing, tests, and helpers use the Swift standard library and Apple frameworks only.
- The deployment target is macOS 26 (`.macOS(.v26)`). Code that needs a macOS 27 API must sit behind `#available`.

## Why

- Apple frameworks cover every need: Core Audio, AVFAudio, Speech, SwiftUI.
- A dependency adds a supply chain and a license to review. On a managed work Mac it can also need approval.
- macOS 26 is the first release with `SpeechAnalyzer`.

## Consequences

- Agents ask the owner before they add any dependency.
- We write some small helpers by hand, such as an argument parser.

Refs: REQ-PKG-001, REQ-PKG-005
