# Build plan

Read `docs/spec.md` first. This file says how to build the tool in three milestones.

Each milestone has a gate. If a gate fails, stop and report. Do not start the next milestone.

## M1. Capture proof

Record both streams as separate PCM streams. Give them one shared host-time reference so they stay aligned.

- Expose each stream as an `AsyncStream` of chunks. A chunk holds samples, sample rate, channel count, and the host time of its first sample.
- Add a watchdog. A stream is silent when it has no callbacks, or when every sample is zero for N seconds. The default N is 5.
- When a stream is silent, print its name and the likely cause. Exit with a distinct code.

To pass the gate:

1. Run `notetaker record` on the dev machine while the system plays `say "testing one two three"`.
2. The Them peak is above a threshold.
3. The watchdog stays quiet.
4. Unit tests cover the level meter and the watchdog with synthetic buffers: normal signal, all zeros, no callbacks, and recovery after silence.

## M2. Live transcription

Run one transcription session per stream. Feed each session from the M1 chunks. Convert audio to the format the speech framework needs.

Gate the microphone with speech detection. Room noise must not become invented sentences.

To pass the gate:

1. Play a known paragraph with `say -v Samantha "<long text>"`.
2. At least 90% of its words appear in the Them lines. Report the real number.
3. Run the human-voice microphone test. Report what you observed.

## M3. Markdown writer and lifecycle

Implement the file format from the spec. Write the `.md.partial` file as you go. Stop cleanly on Enter and on SIGINT. If a stream fails mid-recording, finish the file and record the failure in the terminal and in `warnings:`.

To pass the gate:

1. A 5-minute recording produces a correct file.
2. `kill -9` during recording leaves a readable `.md.partial`.

## Known traps

- A tap alone delivers no buffers. Put it inside a private aggregate device. Exclude this process from the tap. Clean up the aggregate device on exit.
- A missing system-audio permission can produce silence with no error. Detect it by signal.
- `NSAudioCaptureUsageDescription` must be in the Info.plist of the binary that requests the permission. A SwiftPM executable has no Info.plist by default. Find a working way to embed one, for example a linker `__info_plist` section, and check that the prompt appears. If the permission attaches to the terminal app instead, document that.
- Do not enable voice processing (echo cancellation) on the microphone engine. It can stop the process tap.
- With laptop speakers, the microphone hears the far side. Me lines then duplicate Them lines. Measure how often this happens. Build duplicate removal only if the measurement shows a real problem. Otherwise document "use headphones".
- One report says Core Audio taps may not capture Bluetooth A2DP output. Test with AirPods and record the result. Do not assume.
- The audio callback must never block. Do not lock, allocate, write files, or `await` in it. Hand samples to a bounded queue and count overflow.
- The default input or output device can change mid-meeting, for example when AirPods connect. Handle it and document what you observed.

## References

Read these to understand the problem. Do not copy code from them, with one exception below.

- Apple sample code: "Capturing system audio with Core Audio taps".
- `insidegui/AudioCap` on GitHub: a small process-tap example.
- `yuvrajadhikari/oats`, especially `docs/m0-findings.md` and `HANDOFF.md`: measured traps for the same stack. It is MIT licensed. If you reuse any of its code, keep the copyright notice and tell the user which file.
- `owengretzinger/meetingnotes` is LGPL-3.0. Do not copy code from it.

## Acceptance criteria

1. `swift build` and `swift test` pass.
2. All three gates pass. The final report pastes the real command output.
3. A test or script fails if the package references a networking API, for example `URLSession`, `NWConnection`, or `CFNetwork`. This proves the "no network" claim.
4. `README.md` covers install, permissions, usage, and the file format.
5. `docs/findings.md` lists exit codes, the Info.plist approach, and the gotchas you hit. Mark a manual test row "not tested" unless you tested it.

## Authorization boundaries

- Work only inside this repo.
- You may trigger the standard microphone and system-audio permission prompts. Do not change System Settings or TCC state by script.
- Use no `sudo`. Install no packages. The tool makes no network access.
- Do not commit audio or transcripts.
- Ask before you add a dependency or change the scope.

## Report format

End each milestone with a short report:

1. Result
2. Key decisions
3. Verification: the commands you ran and their real output
4. Remaining risks
5. Next action

State what you could not check, and why.
