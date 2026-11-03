# ADR-002 A Core Audio process tap captures system audio

Status: accepted. Date: 2026-11-03.

## Decision

- "Them" comes from a global stereo process tap that excludes this process. The tap sits in a private aggregate device.
- "Me" comes from the `AVAudioEngine` input node. Voice processing stays off.
- The app destroys the tap and the aggregate device on every exit path it can catch.

## Why

| Option | Permission | User setup | Fits the rules |
| --- | --- | --- | --- |
| Process tap | System audio recording | None | Yes |
| ScreenCaptureKit | Screen Recording, a heavier grant | None | Yes |
| Virtual audio driver | None, but the install needs admin rights | Reroute the output | No |

- The tap needs the smallest permission and no setup.
- A driver needs admin rights, which a managed Mac may not give.
- No bot joins the call. The call app sees nothing different.

## Consequences

- Them holds all system audio, including notification sounds.
- A tap alone delivers no buffers. The app must put it in an aggregate device.
- macOS 26 adds `bundleIDs` and `processRestoreEnabled`. A later change can tap one app only.

Refs: REQ-CAP-001, REQ-CAP-002, REQ-CAP-003
