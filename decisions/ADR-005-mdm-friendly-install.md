# ADR-005 Install and signing work on a managed Mac without admin rights

Status: accepted. Date: 2026-11-03.

## Decision

- Nothing needs `sudo` or admin rights. There is no driver, no system folder, and no LaunchDaemon.
- The CLI installs to `~/.local/bin`. The app installs to `~/Applications`.
- The bundle script signs the app with a stable self-signed identity in the login keychain. The user creates it once in Keychain Access.
- Without that identity, the bundle script signs ad hoc and prints a warning.
- Scripts never create certificates and never change TCC or System Settings.
- Developer tools install per user: Node through nvm, `openspec` and `spec-ctx` through npm, skills in `~/.claude/skills`.

## Why

- A work Mac is often managed by MDM, and its user may have no admin rights.
- TCC ties a permission grant to the code signature. An ad hoc signature changes on every build, so grants can reset.
- A self-signed identity needs no Apple account. A managed Mac may block the sign-in that Apple Development signing needs.

## Consequences

- An MDM policy can still deny the microphone or audio capture. The app detects this by signal (ADR-003) and does not try to work around it.
- Sharing the app with other people needs Developer ID signing and notarization. That is out of scope for now.

Refs: REQ-PKG-006, REQ-APP-001, REQ-APP-002, REQ-APP-007
