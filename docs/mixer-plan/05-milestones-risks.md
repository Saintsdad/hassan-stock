# Milestones and risks

## Milestone table

Row counts are from `feature-test-matrix.csv` (406 rows total, after `06-part3-network-use-cases.md`'s additions). A milestone's "entry" is what must already be true to start work; "exit" is what must be true, matrix rows and all, to call it done.

### M0 — Foundations spike (5 rows: engine 1, settings 1, platform 2, defect-fix 1)

**Entry**: repository scaffolding exists; `AGENTS.md`/`CLAUDE.md`/`GEMINI.md` are in place; Rust workspace builds empty crates per the layout in `01-scope-architecture.md`.

**Exit**: the `AudioBackend` trait signature is agreed and reviewed by the human; `backend-null` exists in skeletal form (compiles, no behaviour yet); UI toolkit choice (`egui` per `02-workstreams.md`, or the alternative if the spike finds a blocker) is locked; cross-platform build target list confirmed. This is a spike milestone, not a feature milestone — its rows are placeholders proving the scaffolding, not user-visible behaviour.

### M1 — Engine core + control protocol (43 rows: engine 19, control-api 21, settings 3)

**Entry**: M0 exit criteria met.

**Exit**: the 8x8 routing matrix exists and passes all `engine` rows against `backend-null`; the lock-free parameter queue is proven zero-alloc/zero-lock under the audio-thread sanitizer gates; the JSON/WebSocket + OSC control protocol implements the full parameter grammar the `control-api` rows require, with up to 8 concurrent clients; basic settings persistence exists. This is the most consequential milestone in the plan — everything else is built on top of the matrix and the control protocol, so no downstream area starts in earnest before this exits.

### M2 — Strip and bus DSP core (70 rows: dsp-strip 32, dsp-bus 27, routing 5, engine 1, settings 1, defect-fix 3, virtual-io 1)

**Entry**: M1 exit criteria met.

**Exit**: every strip (mute/solo/mono/gain/limiter/6-cell EQ/pan) and bus (mute/EQ/trim/delay/bus-output-delay) row passes its golden/property tests; the core routing model (strip-to-bus connections, clickless switching) is locked, since `routing`'s remaining rows in M3/M5/M6 build on this being stable.

### M3 — FX, recorder, routing completion (53 rows: fx 23, recorder 19, routing 6, dsp-strip 1, dsp-bus 3, defect-fix 1)

**Entry**: M2 exit criteria met.

**Exit**: the FX chain (denoiser, Karaoke modes, reverb, dynamics) passes its golden-tolerance tests, with the denoiser licensing open question (see below) resolved one way or the other before this milestone closes; the recorder writes all four formats (WAV/AIFF/BWF/MP3) with correct boundary behaviour; routing's remaining rows (recorder arming, NetAudio-adjacent routing) are complete.

### M4 — NetAudio, MIDI, security groundwork (55 rows: netaudio 30, midi 12, control-api 4, security 4, network-ext 1, defect-fix 1, routing 1, settings 1, macro 1)

**Entry**: M1 exit criteria met (this milestone can start in parallel with M2/M3 once the control protocol is stable, since NetAudio and MIDI depend on the parameter queue and control protocol, not on strip/bus DSP being finished).

**Exit**: the VBAN interop rig passes both directions against at least the PipeWire and `quiniouben/vban` fixtures; all 8 incoming/8 outgoing stream slots plus NetAudio-MIDI/TEXT/FRAME work; MIDI mapping and learn are functional; the security area's fuzz targets for netaudio and control-api exist and are running continuously, even though most of `security`'s rows land at M7/M8.

### M5 — Devices, virtual I/O, ASIO, platform (69 rows: devices 15, virtual-io 13, asio 9, platform 9, defect-fix 12, dsp-strip 3, dsp-bus 4, fx 1, routing 2, midi 1)

**Entry**: M2 and M4 exit criteria met.

**Exit**: device enumeration/hot-plug/auto-reconnect works without a full engine restart (the reference's single worst defect, closed here); the Linux (PipeWire) and macOS (HAL plug-in) virtual-device paths pass their VM-lane tests; ASIO works against a locally-supplied SDK on a maintainer runner; the Windows kernel driver's own exit criteria are tracked separately (see the risk treatment below) and are NOT a hard gate for this milestone closing, per decision spine #3 — the rest of M5 must not wait on driver signing.

### M6 — UI, macro, scripting, settings polish (71 rows: ui 19, macro 16, scripting 12, settings 6, defect-fix 6, dsp-strip 1, dsp-bus 1, devices 1, asio 3, midi 3, control-api 1, routing 1, virtual-io 1)

**Entry**: M1 (control protocol) and M4 (MIDI, for macro triggers) exit criteria met; can otherwise run in parallel with M3/M5.

**Exit**: the full UI passes its a11y+screenshot suite (this is where the reference's fixed-pixel/no-theming/no-a11y defects are closed for good); Macro Buttons and scripting are functional end to end, including the 4 NetAudio-TEXT output streams; settings persistence/backup/migration is solid.

### M7 — Network extensions (28 rows: network-ext 21, scripting 1, security 3, defect-fix 3)

**Entry**: M4 exit criteria met (this is layered directly on top of the netaudio plain transport).

**Exit**: the sync transport (timestamps, optional FEC, optional encryption) works and meets its documented alignment tolerance; multi-room static and dynamic grouping work; RTSP/HLS/Icecast ingest and camera backchannel work against local test fixtures; VPN/routed-tunnel operation is proven in the interop rig; Bluetooth/2.4 GHz/roaming/clock-drift rows are covered to the extent CI-simulatable, with the genuinely hardware-dependent ones (physical Bluetooth radio drop, physical AP roaming) marked for the manual/VM lane rather than blocking this milestone's automated-gate closure.

### M8 — Security hardening, final platform work, release readiness (12 rows: security 4, defect-fix 7, platform 1)

**Entry**: all prior milestones' automated gates green; driver workstream at whatever state the human decided in the open questions below.

**Exit**: remaining security rows (manual-review-heavy: credential/secret leakage checks, encrypted-transport final review) closed; remaining defect-fix rows (the "prove a negative" ones — no crash, no licence nag, robust across OS updates) verified at the milestone-review level, not just per-PR; platform packaging finalized for a release candidate.

## Row counts per milestone (summary)

| Milestone | Rows |
|---|---|
| M0 | 5 |
| M1 | 43 |
| M2 | 70 |
| M3 | 53 |
| M4 | 55 |
| M5 | 69 |
| M6 | 71 |
| M7 | 28 |
| M8 | 12 |
| **Total** | **406** |

## Critical path

M0 -> M1 -> M2 -> M3 is the strict sequential spine (each depends on the routing/control-protocol contract the previous one locked). M4 (NetAudio/MIDI) can start as soon as M1 exits and run in parallel with M2/M3. M5 (devices/virtual-io/ASIO/platform) needs both M2 (DSP is stable enough that virtual devices carry real signal) and M4 (device-change handling touches the same backend-trait surface NetAudio uses for its own device-facing bits). M6 (UI/macro/scripting) needs M1 and M4 but not M2/M3/M5, so it can run substantially in parallel with them. M7 (network-ext) is strictly after M4. M8 is last, gated on everything.

The single longest pole is the **Windows kernel driver**, which is not on the critical path for any milestone's automated-gate exit (by design, decision spine #3) but is on the critical path for a *complete, driver-included* release — its own schedule (WHCP certification lead time, HLK testing) should be tracked separately from the M0-M8 sequence and reconciled at release-planning time, not folded into a milestone that would otherwise be blocked by it.

## Top 10 risks and mitigations

1. **Windows kernel driver signing/certification timeline.** April 2026 changes require WHCP or attestation signing; HLK submission lead time is unknown and outside this plan's control. *Mitigation*: isolate as its own workstream (already done); track separately from M0-M8; have the human decide the v1 bridge question (see open questions) so the rest of the plan doesn't stall on it.
2. **VBAN v13 spec was not fully accessible to research.** The PDF fetch was blocked; the plan's description of the wire format is secondhand. *Mitigation*: Opus must obtain and read the actual spec before implementing `engine-net`; the interop rig (real independent implementations) is the actual correctness check, not this plan's text.
3. **Denoiser/reverb algorithm licensing is unconfirmed.** RNNoise/DeepFilterNet licences were not confirmed in research; Freeverb's exact licence varies by source. *Mitigation*: human confirms licence before M3 exit; in-house fallback (simpler noise gate/spectral subtraction, patent-free reverb from a published paper) is budgeted as the safe path if licensing can't be confirmed in time.
4. **MP3 encoder licensing (LAME, LGPL) adds distribution complexity.** *Mitigation*: decide (see open questions) whether to ship LAME dynamically linked, delegate to a platform encoder, or drop MP3 encode from v1 and keep decode-only plus WAV/AIFF/BWF encode.
5. **AAC/M4A patent licensing fee.** A one-time cost is cited in research but not confirmed current for 2026; most legacy patents have expired, not all. *Mitigation*: do not add AAC encode/decode to v1 scope at all; revisit only with an explicit human decision and current legal confirmation.
6. **ASIO SDK relicensing (Oct 2025, GPLv3/proprietary dual) changes the compliance picture project-wide if mishandled.** *Mitigation*: strict "never vendored, locally supplied only" rule (decision spine #3), enforced by a CI check that fails the build if SDK files appear in the repository.
7. **Audio-thread real-time-safety regressions are easy to introduce invisibly.** A single added allocation or lock can pass functional tests and still glitch in production. *Mitigation*: `assert_no_alloc` and RTSan-class checks as hard CI gates on every relevant PR, not just at milestone review; the "Flash must never touch audio-thread code" rule exists specifically to bound this risk's surface area.
8. **Cross-model instruction drift / hallucination compounding on a long-running Flash task.** *Mitigation*: PR-sized task scoping, explicit escalation rules, adversarial review checklist — all in `04-agent-operating-model.md`.
9. **OS vendor changes break virtual-device/driver compatibility with no notice** (the reference's own 24H2 incident is precedent, not hypothetical). *Mitigation*: budget standing reactive capacity in `platform`, not just planned milestone work; keep drivers/plug-ins on a "known to work with latest OS build" tracking cadence.
10. **Scope creep into the reference's disadvantages beyond what the matrix specifies.** The temptation to "also fix" something Part 2 mentions but the matrix doesn't cover is real, especially given how thorough the defects catalogue is. *Mitigation*: the matrix is the contract; a new row is how a new fix enters scope, not an unplanned PR.

## Open questions the human must decide

- **Licensing model for the project itself.** Not addressed by any research file or the decision spine; needed before a public release regardless of individual dependency licences chosen in `02-workstreams.md`.
- **Naming.** OPENMIX is explicitly a placeholder throughout this plan. A real name needs a trademark check the agents cannot perform.
- **Whether to ship a Windows kernel driver v1 at all, or bridge through an existing virtual-cable product (e.g. a widely used existing virtual audio cable) for the initial release and build the native driver as a fast-follow.** This directly trades the highest-risk item in the plan (item 1 above) against a real, if less clean, alternative — a call only the human should make given the trademark, support-burden and certification-timeline trade-offs involved.
- **MP3 encoder approach**: LAME (LGPL, dynamically linked and swappable) vs. platform-delegated encoding vs. dropping MP3 encode from v1.
- **Denoiser algorithm licensing**, pending confirmation of RNNoise/DeepFilterNet licence terms, or a decision to ship the simpler in-house fallback instead.
- **Whether AAC support is wanted at all**, given the patent-licensing complexity and the fact that the reference itself does not obviously require it for Max-level parity beyond general "codec support" expectations.
- **CLAP plugin hosting scope and timing** — decision spine marks it "later milestone" but does not fix which one; the human should decide whether it lands inside this M0-M8 sequence or genuinely after.
