# AGENTS.md

Rules for coding agents in this repo. `CLAUDE.md` imports this file.

## What lives where

| Path | Holds |
| --- | --- |
| `docs/product.md` | The product, the roadmap, and the cut line |
| `openspec/changes/` | One change per milestone: proposal, design, delta specs, tasks |
| `openspec/specs/` | The specs of archived changes, the current truth |
| `decisions/` | Decision records `ADR-NNN` |
| `docs/findings.md` | Measurements and the manual test table |
| `.ctx/` | spec-ctx state: evidence, handoff, questions |

If a spec and the code disagree, follow the spec and report the difference. If a spec is wrong or unclear, stop and ask. Do not guess.

## Workflow

The repo uses [OpenSpec](https://github.com/Fission-AI/OpenSpec) with [spec-ctx](https://github.com/sonqh/spec-ctx).

1. Start each session with `specctx status`.
2. Run the `specctx-run` skill. It runs `specctx next --json` and acts on the answer.
3. Before you edit, run `specctx task start <task>` or `specctx context <ID|path>`. Do not open `openspec/specs/` by hand.
4. Close a task only with `specctx task done <task>`. It runs the `Verify:` command. Never tick a checkbox by hand. Never run `/opsx:apply`.
5. End the session with `specctx handoff write`.

The loop stops for a human at each plan and before each archive. Do not answer those questions yourself.

## Environment

- macOS 27 on Apple Silicon, Xcode 27, Swift 6.4. The deployment target is macOS 26.
- The SDK interface files are the source of truth for API names. Tutorials are often older than this SDK.
- npm installs `openspec` and `specctx` per user. Set `DO_NOT_TRACK=1` in the shell. OpenSpec telemetry is off in its global config.

## Scope

- Write Swift. Use no third-party dependency. Ask before you add one (ADR-001).
- Work on one change at a time, in the order of `docs/product.md`.
- Do not add an LLM or summary feature. Summaries belong to Claude Code (ADR-007).

## Code rules

- Make no network call from the app (ADR-007).
- Measure signal. Do not trust an audio API return code alone (ADR-003).
- Never block the audio callback. Do not lock, allocate, write files, log, or `await` in it.
- Keep logic that does not touch hardware testable with synthetic chunks.
- Tag code with the requirement it implements: `// REQ-CAP-001`. Tag tests with `// Covers: REQ-CAP-001`.
- Put a check that needs real audio hardware in `scripts/gates/`.

## Safety

- Work only inside this repo.
- Use no `sudo`. Install nothing that needs admin rights (ADR-005).
- Do not change System Settings or TCC state by script. You may trigger the standard permission prompts.
- Do not create signing certificates by script.
- Do not commit audio or transcripts.
- Commit only when the user asks.
- `CLAUDE.local.md` and `.local/` may hold local, untracked notes. Never commit them, and never copy their content into a tracked file, a commit message, or a PR.

## Licenses

- Do not copy code from `owengretzinger/meetingnotes`. It is LGPL-3.0.
- If you reuse MIT code, keep its copyright notice and tell the user which file.

## Checks

- Run `swift build` and `swift test`.
- Run `specctx check --strict` after each task group, and `specctx gate --strict` before archive.
- Paste real command output in your report. Do not describe results you did not observe.
- Record measurements in `docs/findings.md`. Leave a row as "not tested" unless you tested it.

## Docs

- Write Markdown prose in English, in short active sentences. The docs follow the [ste-humanize](https://github.com/sonqh/ste-humanize-skill) skill.
- `specctx gate` lints the docs with `specctx lint tells` and `specctx lint ste`. Run a lint with `< /dev/null`, or it can wait for stdin.
- Keep code, commands, and paths unchanged.
