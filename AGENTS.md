# AGENTS.md

Rules for coding agents in this repo.

## Read first

1. `docs/spec.md`
2. `docs/build-plan.md`

## Rules

- Write Swift. Use no third-party dependency. Ask before you add one.
- Make no network call from the tool.
- Measure signal. Do not trust an audio API return code.
- Never block the audio callback.
- Work only inside this repo.
- Do not change System Settings or TCC state by script.
- Do not commit audio or transcripts.
- Do not copy code from `owengretzinger/meetingnotes`. It is LGPL-3.0.
- If you reuse MIT code, keep its copyright notice and tell the user which file.
- Do not add a summary or LLM feature. Summaries belong to Claude Code.

## Verify

- Run `swift build` and `swift test`.
- Run the gate for the milestone you finish.
- Paste real command output in your report. Do not describe results you did not observe.

## Docs

Write Markdown prose in short, active sentences. These docs follow the [ste-humanize](https://github.com/sonqh/ste-humanize-skill) skill. Keep code, commands, and paths unchanged.
