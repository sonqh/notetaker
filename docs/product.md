# Product

notetaker is a personal, local-first meeting notes app for macOS, in the style of Granola. It runs next to any call app. No bot joins the call. It captures the audio on the Mac, transcribes it on the Mac, and writes one English Markdown file. Claude Code writes the summary.

## Principles

- No bot in the call. The app reads the local audio only (ADR-002).
- Local only. The app makes no network call and has no LLM (ADR-007).
- Measured signal. The app proves capture works with numbers (ADR-003).
- One plain file per meeting. It survives a crash (ADR-006).
- No admin rights needed. It works on a managed Mac (ADR-005).
- Swift and Apple frameworks only (ADR-001).

## Roadmap

Each stage is one OpenSpec change. Build them in order. Each change has a gate in its `tasks.md`. If a gate fails, stop and report.

| Order | Change | Milestone | What it adds |
| --- | --- | --- | --- |
| 1 | `001-package-and-probes` | M0 | Package, CLI parsing, exit codes, embedded Info.plist, no-network test, `tap-probe`, `speech-probe` |
| 2 | `002-audio-capture` | M1 | Me and Them streams, watchdog, `doctor`, `debug-audio` |
| 3 | `003-live-transcription` | M2 | One analyzer per stream, live and finalized results, speech model check |
| 4 | `004-record-to-markdown` | M3 | `record`, the file format, notes, crash-safe writes |
| 5 | `005-menu-bar-app` | M4 | Menu bar app, first-run checks, mic-use prompt, saved notification |
| later | notepad | M5 | A notes window. Notes and transcript in one file. |
| later | library | | Past meetings, search, rename, open in Claude Code |
| later | calendar | | Titles and attendees from EventKit |

The first change runs two experiments before the real build. They retire the two biggest risks:

1. A tap probe plays a system sound and checks that the tap hears it. It runs from Terminal, from the VS Code terminal, and later from the `.app`.
2. A speech probe transcribes an audio file and reports word recall against a reference text.

## Cut line for the first usable version

| Must | Should | Later |
| --- | --- | --- |
| `record` with Me and Them, crash-safe file | `debug-audio` | Notepad window |
| `doctor` with the system sound self-test | The Them "suspect" state, after measurement | Library and search |
| Watchdog and `warnings` | `--keep-audio` | Calendar |
| Stop with `/stop` or Ctrl-C | Consent reminder setting | Start at login |
| Notes typed during the meeting | | Duplicate removal on laptop speakers |
| Menu bar start and stop, meters, saved notification | | A tap on one app only |
| Mic-use prompt that never records on its own | | Vocabulary hints |

## Non-goals

- An LLM or summary feature inside the app.
- Network calls, sync, sharing, accounts.
- Speaker identification beyond "Me" and "Them".
- Any language other than English.
- A database.
- Auto-update and analytics.

## Consent

- A setting shows one quiet line while recording: "Tell participants you are taking a transcript."
- Consent law differs by place. Some places need the consent of every party. Employer rules and NDAs can be stricter. This is not legal advice.

## Quality measurement

- Scripted dialogues live in `Tests/Fixtures/`.
- `say -o` writes speech to a file. `speech-probe` then transcribes the file with the same pipeline.
- The measures are word recall, word error rate, speaker label accuracy, Me segments per silent minute, and recall of a term list.
- Text-to-speech audio is cleaner than real speech. These numbers are upper bounds.
- Results go in `docs/findings.md`.

## References

Read these to understand the problem. Do not copy code from them, with one exception.

- Apple sample code: "Capturing system audio with Core Audio taps".
- `insidegui/AudioCap` on GitHub: a small process tap example.
- `yuvrajadhikari/oats`, mainly `docs/m0-findings.md` and `HANDOFF.md`: measured traps for the same stack. It is MIT licensed. If you reuse its code, keep the copyright notice and tell the owner which file.
- `owengretzinger/meetingnotes` is LGPL-3.0. Do not copy code from it.
