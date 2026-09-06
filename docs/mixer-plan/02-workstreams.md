# Workstreams

One section per `area` value in `feature-test-matrix.csv`, ordered by row count. Each section gives the goal, the interfaces it exposes or depends on, the hardest part of the work, the licences and dependencies research selected for it (exact licence names, see `07-research-notes.md` for sources), the main risks, the dominant owner, the dominant milestone, and the row/test-id range so a matrix row can be traced straight back here.

Owner values in the matrix are `opus`, `flash`, or `opus+flash` (Opus designs/writes the core, Flash writes the bulk tests or peripheral bindings under Opus review). "Dominant owner" below is the most common value in that area, not the only one; always check the row.

## dsp-strip (37 rows, T-dsp-strip-001..037)

**Goal**: per-strip processing — gain, limiter (-40 to +12 dB, pre-gain), 6-cell parametric EQ (-36 to +18 dB, Q 1-100), pan, mute/solo, insert point.

**Interfaces**: sits between the backend's input capture and the routing matrix; reads parameters off the lock-free queue; writes to the insert/monitor tap.

**Hardest part**: the EQ cells and limiter must be bit-reproducible across runs (golden tests) while still being sample-accurate under automation (smooth parameter interpolation, no zipper noise), and correct at every documented sample rate from 32 kHz to 192 kHz without re-deriving coefficients per call.

**Licences/dependencies**: RBJ cookbook biquad formulas (public formulas, MIT when published standalone) and Vicanek matched-Z biquads (published algorithm, no redistribution licence issue) implemented in-house; no third-party DSP crate required for the EQ/limiter core.

**Risks**: coefficient drift at extreme sample rates; denormal stalls without FTZ/DAZ handling (mitigate as described in `03-test-kit.md`).

**Dominant owner**: opus+flash. **Dominant milestone**: M2.

## dsp-bus (35 rows, T-dsp-bus-001..035)

**Goal**: per-bus parametric EQ, per-channel trim (-24 to +24 dB) and delay (0-500 ms), bus output delay (0-500 ms, applied before physical/virtual/Virtual-ASIO/NetAudio outputs), routing summation.

**Interfaces**: consumes strip outputs per the routing matrix; feeds backend output and NetAudio send.

**Hardest part**: delay lines that are sample-accurate and reconfigurable in real time without clicks, and that apply consistently across every output path (physical, virtual, Virtual ASIO, NetAudio) per the reference's explicit statement that bus delay is not limited to physical buses.

**Licences/dependencies**: in-house delay-line and biquad implementation, same basis as dsp-strip; no external dependency.

**Risks**: delay-line memory growth at 192 kHz/500 ms per channel across 8 buses x 8 channels; must be bounded and tested for worst-case allocation at startup, not per-change.

**Dominant owner**: opus. **Dominant milestone**: M2.

## defect-fix (34 rows, T-defect-fix-001..034)

**Goal**: implement the Part 2 defect fixes named in the decision spine: undo/redo, auto device reconnect, per-app volume, resizable/DPI-aware UI, clock-sync option, encrypted option, no licence nag, CLAP hosting (later milestone).

**Interfaces**: cuts across engine-core (undo/redo, presets), backend-trait (device reconnect), control-protocol (per-app volume needs OS-level per-process audio session hooks), and UI (DPI/theming).

**Hardest part**: these are fixes to problems the reference has, not new isolated features, so each one has to be tested against the *symptom* (e.g. "USB hot-plug does not require Restart Audio Engine") not just a unit in isolation — most of this area's tests are integration or manual for that reason.

**Licences/dependencies**: per-app volume on Windows uses the OS-provided per-session volume APIs (no third-party dependency); CLAP hosting later uses `clap-sys` (no fixed licence stated in research; verify before use) or NIH-plug-style hosting patterns.

**Risks**: "prove a negative" tests (no crash, no nag, no restart required) are inherently harder to make deterministic; scope creep risk is highest here because "fix the disadvantages" invites solving problems beyond what the matrix specifies — stick to the row's acceptance test.

**Dominant owner**: opus. **Spread across milestones M0-M8**, heaviest at M5 (12 rows), M8 (7), M6 (6).

## netaudio (30 rows, T-netaudio-001..030)

**Goal**: the VBAN-compatible plain transport — header parsing, 5 sub-protocols (AUDIO/PCM, SERIAL/MIDI, TEXT, FRAME, PING), 8 incoming/8 outgoing stream slots, sample-rate table, packet-size limits, Network Quality buffering behaviour.

**Interfaces**: `engine-net` sits beside the routing matrix (streams act like extra strips/buses) and below `network-ext`'s sync-transport layer.

**Hardest part**: wire compatibility with a protocol family whose canonical spec (VBAN Protocol v13, Sep 2025) was not fully accessible to research (PDF fetch blocked) — the implementation must be validated against real independent implementations, not just the spec text, hence the heavy interop-test weighting in this area.

**Licences/dependencies**: implement the wire format in-house from the specification and from interop testing (cleanroom), citing the spec by version. Do not vendor or link GPL implementations such as `quiniouben/vban` (GPLv3) or the existing Rust crates `vban-rs`/`rusty-vban` (licences not confirmed in research) into the shipped binary; they are acceptable as independent test-only fixtures in the interop rig (see `03-test-kit.md`), run out-of-process, never linked. PipeWire's own `module-vban-send`/`recv` (LGPL-2.1+, part of PipeWire itself) is a legitimate interop target on Linux, again out-of-process.

**Risks**: the VBAN spec PDF was inaccessible to research; before implementation, Opus must obtain and read the actual v13 spec (or the version current at build time) rather than relying solely on this plan's secondhand description. This is flagged again in `07-research-notes.md`.

**Dominant owner**: opus. **Dominant milestone**: M4 (all 30 rows).

## control-api (26 rows, T-control-api-001..026)

**Goal**: the canonical JSON/WebSocket + OSC control protocol, the `Strip[i].Gain`-style parameter grammar, up to 8 concurrent clients, and the compat shim's C-ABI surface.

**Interfaces**: sits above engine-core's lock-free parameter queue; the compat shim translates the incumbent C ABI into calls against this same protocol.

**Hardest part**: matching the incumbent's parameter grammar closely enough that ported scripts and existing wrappers work, while keeping the underlying protocol clean enough to extend — the grammar is a compatibility contract, not just an internal naming convention.

**Licences/dependencies**: no third-party protocol library required beyond a permissively licensed WebSocket/JSON crate already used elsewhere in the Rust ecosystem (MIT/Apache-2.0 class); OSC via a permissively licensed OSC crate.

**Risks**: the compat shim's exact exported-function signatures are inferred from public wrapper behaviour, since the incumbent's Remote API PDF terms were not independently re-derivable by research; validate the shim against the existing python/Go/Node wrappers directly (see `03-test-kit.md`'s Remote-API shim conformance run) rather than the PDF alone.

**Dominant owner**: opus+flash. **Dominant milestone**: M1 (21 of 26 rows) — this area is largely on the critical path since almost everything downstream is driven through it.

## fx (24 rows, T-fx-001..024)

**Goal**: the effects chain: denoiser (LEARN ~10 s, BYPASS), Karaoke modes (K-m, K-1, K-2, Kv), reverb, multiband dynamics, and other per-strip/per-bus FX beyond the base EQ.

**Interfaces**: inserted into the same signal path as dsp-strip/dsp-bus, selectable pre/post per the insert model.

**Hardest part**: the denoiser and reverb are the two DSP blocks where quality is subjective (MOS-style judgement) as well as objective (THD+N, SNR), so their acceptance tests combine golden-file comparison with documented tolerance bands rather than exact bit-matching.

**Licences/dependencies**: reverb uses patent-free designs — Freeverb-style Schroeder reverberator (LGPL with GRAME's Faust exception, or BSD in older Faust examples; verify the exact source before copying any code, else reimplement from the published algorithm) or Dattorro plate reverb / FDN (patent-free published algorithms, implement from the paper, no redistribution licence issue). Denoiser: RNNoise-style or DeepFilterNet-style approach is the reference class of algorithm; **licence for both is unconfirmed in research** — do not vendor either without the human confirming licence terms; budget for an in-house lightweight noise-gate/spectral-subtraction denoiser as the safe fallback if licensing cannot be confirmed by M3.

**Risks**: denoiser licensing is an open question (see `05-milestones-risks.md`); reverb CPU cost at 192 kHz across many strips needs an early perf budget, not a late one.

**Dominant owner**: opus. **Dominant milestone**: M3 (23 of 24 rows).

## network-ext (22 rows, T-network-ext-001..022)

**Goal**: everything from Part 3 that is not plain VBAN interop — sync transport (timestamps, optional FEC, optional encryption), multi-room grouping (static and dynamic), VPN operation, RTSP/HLS/Icecast ingest and backchannel, Bluetooth/2.4 GHz endpoint handling, Wi-Fi roaming, and long-session clock-drift reconciliation. See `06-part3-network-use-cases.md` for the full derivation.

**Interfaces**: layers on top of `engine-net`'s plain transport; ingest rows (RTSP/HLS/Icecast) are new input sources feeding a virtual strip, symmetrical with any other input.

**Hardest part**: this area mixes real-time media plumbing (RTSP/RTP decode) with distributed-systems concerns (clock drift, roaming handoff, dynamic group membership) that are inherently harder to make deterministic in CI — most of its perf/property tests need simulated network fixtures, not real Wi-Fi hardware.

**Licences/dependencies**: RTSP/RTP/HLS/Icecast ingest via GStreamer (LGPL-2.1+ for core/base/good plugins; GPL-licensed plugins such as `gst-plugins-ugly` are avoidable since none of the target codecs require them) or an `ffmpeg`-based sidecar built without `--enable-gpl`/`--enable-nonfree` to stay LGPL; alternatively a `go2rtc`/`MediaMTX` sidecar process (check their licences before adoption — not fully covered in research) kept out-of-process so its licence does not affect the engine binary. VPN test fixture: `boringtun` (BSD-3-Clause) as a userspace WireGuard endpoint for the interop rig only. Sync-transport encryption: a Noise Protocol Framework implementation (the framework itself is public domain; pick a permissively licensed Rust crate implementing it) rather than rolling custom cryptography.

**Risks**: this is the newest, least-precedented area in the matrix (12 of its 22 rows were derived from Part 3 headings for this plan rather than pre-existing in the original matrix); expect the acceptance tests here to need the most refinement once implementation starts. Ingest sidecars run as separate processes specifically so a codec-licence problem in a sidecar cannot force a relicense of the engine.

**Dominant owner**: opus, with opus+flash and flash on the ingest/UI-facing rows. **Dominant milestone**: M7 (21 of 22 rows; one row is M4).

## engine (21 rows, T-engine-001..021)

**Goal**: the core routing matrix (8x8, 34 in/64 out), the lock-free parameter queue, the backend trait itself, and the zero-allocation/zero-lock audio-thread guarantee.

**Interfaces**: everything else in the engine sits on top of this; the backend trait is the seam every OS backend and the null backend implement.

**Hardest part**: proving zero allocation and zero locking on the audio thread is a continuous property, not a one-time check — this is where `assert_no_alloc` and RTSan-style tooling matter most (see `03-test-kit.md`).

**Licences/dependencies**: `assert_no_alloc` (custom allocator for detecting forbidden allocations, permissive licence per crates.io) in debug/test builds; a Rust RTSan wrapper (LLVM/Clang 20+ based; check current licence at adoption time, not fully confirmed in research) as an additional CI-only sanitizer pass on Linux/macOS.

**Risks**: this area is the foundation for every other area — a design mistake here (e.g. a hidden lock in the parameter queue) is expensive to unwind later; it is scheduled first (M1, plus the M0 backend-trait bring-up) for that reason.

**Dominant owner**: opus. **Dominant milestone**: M1 (19 of 21 rows).

## recorder (19 rows, T-recorder-001..019)

**Goal**: WAV/AIFF/BWF/MP3 recording, 8/16/24/32-bit (float), 1-8 channels, target sample rate independent of engine rate, 2 GB limit on WAV/AIFF/MP3 (unlimited BWF), auto-stop at limit.

**Interfaces**: taps bus or strip output; writes files; independent SRC from engine rate to file rate.

**Hardest part**: BWF's `bext`/`iXML` metadata chunks and the 2 GB boundary behaviour (auto-stop, not corruption) need dedicated golden-file and boundary tests, not just "does it write a valid WAV."

**Licences/dependencies**: WAV/AIFF/BWF via an in-house or permissively licensed writer (container formats are unencumbered); MP3 decode via `symphonia-bundle-mp3` (Rust, MPL-2.0) for any read-back/verification tooling; MP3 *encode* is the licence-sensitive part — LAME is LGPL (acceptable if dynamically linked and swappable, but adds complexity) or delegate to a platform encoder where available. Do not add AAC/M4A encode in v1: the AAC patent pool carries a real one-time licensing fee, and most legacy patents have expired but not all, so this is a call for the human (see `05-milestones-risks.md`), not a default-yes.

**Risks**: MP3 encoder licensing choice is unresolved; BWF long-file behaviour (unlimited size) needs a dedicated soak test since it is the one recorder path without the 2 GB ceiling.

**Dominant owner**: opus+flash. **Dominant milestone**: M3 (all 19 rows).

## ui (19 rows, T-ui-001..019)

**Goal**: the DPI-aware, themeable, resizable, accessible desktop UI, talking only over the control protocol.

**Interfaces**: control-protocol client; nothing else — the UI process has no engine dependency at all.

**Hardest part**: accessibility is a first-class acceptance criterion here (fixing the reference's fixed-pixel-layout defect), so every UI row needs an a11y-tree assertion in addition to a screenshot, and the two must agree (a screenshot that "looks right" but has no accessible name is a failing test).

**Licences/dependencies**: `egui` (MIT/Apache-2.0, immediate-mode Rust GUI, DPI-aware via a logical-points scale factor) is the recommended toolkit over JUCE (dual AGPLv3/commercial), Slint (GPL-3.0/commercial) and Qt (LGPLv3/commercial) specifically because it carries no commercial-tier licensing question and no GPL propagation risk for a UI that talks to (but does not link) the engine. `iced` (also permissive, wgpu-based) is an acceptable alternative if `egui`'s theming proves insufficient; both are Rust-native and avoid introducing a second language into the UI layer.

**Risks**: choosing a GUI toolkit is itself a decision with downstream cost if reversed late; lock this in during M0/M1 spike work, not mid-M6.

**Dominant owner**: flash (all 19 rows), under Opus review of the a11y/protocol-boundary tests specifically, since a UI that quietly does something over the control protocol it shouldn't is a security-relevant bug even though it "looks like" pure UI work.

**Dominant milestone**: M6 (all 19 rows).

## macro (17 rows, T-macro-001..017)

**Goal**: Macro Buttons — Push/2-Positions button types, keyboard/MIDI/gamepad/audio-level triggers (with In/Out thresholds and Hold, measured pre-fader/post-fader/post-mute), 4 NetAudio-TEXT output streams.

**Interfaces**: control-protocol for triggering actions; engine-net for the NetAudio-TEXT output streams; midi/devices areas for MIDI and gamepad trigger sources.

**Hardest part**: audio-level triggers need the same pre-fader/post-fader/post-mute tap points as metering, and must not introduce audio-thread allocation or locking just because a UI-configured trigger changed.

**Licences/dependencies**: gamepad input via a permissively licensed Rust input crate (e.g. an XInput-class binding, MIT/Apache-2.0 class); no other special dependency.

**Risks**: the "4 NetAudio-TEXT out streams" limit is a hard contract number from the reference; do not silently allow more without an explicit spec change.

**Dominant owner**: flash. **Dominant milestone**: M6 (16 of 17 rows).

## devices (16 rows, T-devices-001..016)

**Goal**: device enumeration, selection, hot-plug handling, auto-reconnect (defect fix), per-device status (SR, buffer, channel count, bit resolution, share mode).

**Interfaces**: backend-trait's `enumerate_devices`/`on_device_change`; feeds the UI's device picker and the defect-fix auto-reconnect logic.

**Hardest part**: hot-plug and device-ID-change handling (the reference's "Restart Audio Engine" defect) must work without a full engine restart, which means the backend trait's device-change callback has to be able to hand off a live stream to a re-enumerated device ID.

**Licences/dependencies**: none beyond the backend trait itself; on Linux this exercises PipeWire's own device-change notifications (LGPL-2.1+, used as a runtime dependency, not linked into engine-core).

**Risks**: this area is where the reference's single worst usability complaint (Restart Audio Engine) lives; treat its acceptance tests as the definition of "fixed," not just "improved."

**Dominant owner**: opus. **Dominant milestone**: M5 (15 of 16 rows).

## midi (16 rows, T-midi-001..016)

**Goal**: MIDI mapping for parameters and macro triggers, MIDI learn, Mackie-Control/HUI-style surface support (implied by the reference, not confirmed in detail — flagged in `07-research-notes.md`).

**Interfaces**: a device-facing input similar to `devices`, feeding parameter changes into the control-protocol layer, not directly into the audio thread.

**Hardest part**: MIDI learn's "listen for the next message on any mapped device" state machine has to be robust to multiple simultaneous learn requests and to devices disconnecting mid-learn.

**Licences/dependencies**: a permissively licensed Rust MIDI crate (MIT/Apache-2.0 class) for transport; no protocol-specific licence issue (MIDI itself is an open standard).

**Risks**: Mackie Control/HUI support was not independently verified against the reference in research; treat any row implying full DAW-control-surface compatibility as needing spec confirmation before the acceptance test is finalized.

**Dominant owner**: opus+flash. **Dominant milestone**: M4 (12 of 16 rows).

## routing (15 rows, T-routing-001..015)

**Goal**: the strip-to-bus routing buttons themselves, route persistence, and route-change behaviour (clickless switching).

**Interfaces**: the matrix's connection topology, read by dsp-strip/dsp-bus, written by control-protocol and UI.

**Hardest part**: route changes must be clickless (cross-fade or sample-accurate mute-window) even under rapid toggling, and must be atomic with respect to the parameter queue so a partially-applied route is never audible.

**Licences/dependencies**: none beyond engine-core.

**Risks**: spread across four milestones (M2 through M6) because routing touches UI, NetAudio-routed buses and Virtual-ASIO patch points; keep the core routing model locked by M2 so later areas build on a stable contract.

**Dominant owner**: opus+flash. **Dominant milestone**: split across M2/M3, no single dominant milestone.

## virtual-io (15 rows, T-virtual-io-001..015)

**Goal**: virtual devices other applications select as input/output — the Windows kernel driver's user-visible behaviour, macOS HAL plug-in, PipeWire virtual nodes, and the Insert Virtual ASIO device (34 channels, PRE-FX/POST-FX switch).

**Interfaces**: backend-trait's `VirtDev` side; consumed by other applications on the host OS, not by OPENMIX's own UI.

**Hardest part**: this is the one area whose correctness cannot be fully verified against `backend-null` — a virtual device's contract is "does another application see it correctly," which needs the actual OS's audio stack in the loop (see the driver test VM lane in `03-test-kit.md`).

**Licences/dependencies**: Linux uses PipeWire virtual nodes directly (LGPL-2.1+, runtime dependency); macOS uses the AudioServerPlugIn pattern (Apple framework, no third-party licence; BlackHole is a public open-source example of the pattern under GPL-3.0 but its code is not to be copied — build from Apple's own sample code and documentation); Windows virtual device is the kernel-driver workstream, treated separately below.

**Risks**: platform-specific by construction; cannot be de-risked by more null-backend testing, only by the VM/HAL/PipeWire lanes actually exercising real OS audio stacks.

**Dominant owner**: opus. **Dominant milestone**: M5 (13 of 15 rows).

## scripting (13 rows, T-scripting-001..013)

**Goal**: the Macro Buttons scripting language (NetAudio-TEXT-driven and/or a small expression/branching language per the reference's "if/else on a variable holding a mixer level query result" style tests).

**Interfaces**: control-protocol (script actions call the same parameter grammar); macro area for triggers.

**Hardest part**: parsing must be strict and safe against malformed scripts (this is explicitly a fuzz-tested row), since scripts can come from imported presets or pasted community snippets, not just the built-in UI editor.

**Licences/dependencies**: an in-house recursive-descent parser or a permissively licensed parser-combinator crate (MIT/Apache-2.0 class); no external scripting-language dependency needed given the documented feature's small surface.

**Risks**: scope creep toward a general-purpose scripting language; the matrix's acceptance tests define the actual surface — do not add language features the tests do not require.

**Dominant owner**: flash. **Dominant milestone**: M6 (12 of 13 rows).

## asio (12 rows, T-asio-001..012)

**Goal**: ASIO host I/O on Windows, subject to decision spine #3 (SDK supplied locally, never vendored), and the multi-client limitation the reference documents (A1 only via ASIO; other buses reach ASIO only through the A1 device's channel patch).

**Interfaces**: `backend-windows`'s ASIO path; feeds the same backend-trait `open_stream` contract as WASAPI/KS.

**Hardest part**: ASIO natively allows only one open device at a time; the reference's own workaround (route everything through A1's channel patch) has to be replicated faithfully, or a wrapper pattern (see FlexASIO below) adopted, rather than inventing a different multi-client model that would break user expectations.

**Licences/dependencies**: the Steinberg ASIO SDK is dual-licensed GPLv3/proprietary as of October 2025 — per decision spine #3, CI and contributor machines must supply it locally (e.g. via an environment variable pointing at a locally-downloaded SDK); it is never committed to the repository or fetched automatically by any build script. `asio-sys` (Apache-2.0) is the Rust binding layer against that locally-supplied SDK. `FlexASIO` (existing open-source ASIO-over-PortAudio wrapper) is a reference pattern for multi-client support, not a dependency to vendor.

**Risks**: any CI job that needs ASIO either skips on runners without the SDK present (feature-flagged out, not a hard failure) or runs only on a small number of maintainer-controlled runners with the SDK installed; this must never block the main CI gate for contributors without SDK access.

**Dominant owner**: opus. **Dominant milestone**: M5 (9 of 12 rows).

## settings (12 rows, T-settings-001..012)

**Goal**: persisted application settings (device selections, preferred sample rate, UI preferences) and their migration/backup behaviour.

**Interfaces**: engine-core's preset/persistence layer; UI reads/writes through control-protocol, not by touching files directly.

**Hardest part**: settings persistence has to survive version upgrades without corrupting or silently dropping user configuration — this is a defect the reference itself does not fully solve (decision spine #7's "settings persistence" fix).

**Licences/dependencies**: a permissively licensed serialization format (JSON or a binary format with a JSON export) for settings files; no external dependency of note.

**Risks**: none area-specific beyond general migration-testing discipline; low technical risk, moderate correctness risk if skipped.

**Dominant owner**: opus+flash. **Dominant milestone**: M6 (6 of 12 rows), with early groundwork at M1.

## platform (12 rows, T-platform-001..012)

**Goal**: cross-platform packaging, install/uninstall behaviour, OS-version compatibility checks (the reference's Windows-11-25H2/24H2 lessons apply here by analogy).

**Interfaces**: touches backend-windows/macos/linux at the install/uninstall boundary; not audio-thread code.

**Hardest part**: OS feature-update compatibility cannot be fully tested in CI (new OS builds arrive on Microsoft's/Apple's schedule, not ours); this area leans on the manual test-type more than most, and on the driver-specific VM lane described in `03-test-kit.md` for the Windows case.

**Licences/dependencies**: standard OS packaging tools (MSI/installer tooling on Windows, `.pkg`/notarization on macOS, distro packages on Linux); no runtime-library licence implications.

**Risks**: this is the area most likely to be affected by an OS vendor's unannounced change (the reference documents exactly this happening to the incumbent around Windows 11 24H2/25H2); budget standing capacity for reactive fixes here, not just planned milestone work.

**Dominant owner**: opus. **Dominant milestone**: M5 (9 of 12 rows).

## security (11 rows, T-security-001..011)

**Goal**: the encrypted sync-transport option, input validation/fuzzing across all network-facing parsers (VBAN, control protocol, scripting), and general hardening.

**Interfaces**: cuts across engine-net, control-protocol and scripting; every fuzz target in the codebase reports here for tracking purposes even if its code lives in another area's crate.

**Hardest part**: proving the *absence* of a class of bug (memory safety on malformed network input) rather than the presence of a feature — this is why this area is fuzz- and manual-review-heavy rather than unit-test-heavy.

**Licences/dependencies**: a Noise Protocol Framework implementation (framework is public domain; pick a permissively licensed Rust crate) for the sync transport's authenticated encryption, following the WebRTC DTLS-SRTP precedent's design goals (confidentiality, integrity, replay protection) without adopting DTLS itself, since the sync transport is not WebRTC-based; `boringtun` (BSD-3-Clause) informs the "userspace crypto tunnel in Rust" pattern for the VPN interop test rig.

**Risks**: security work is 100% Opus-owned in this matrix (11 of 11 rows) — never delegate a security row to Flash even for "just the test," per the agent operating model's "what Flash must never touch" list.

**Dominant owner**: opus (all 11 rows). **Dominant milestone**: spread across M4, M7, M8.

---

## Windows kernel driver: explicit risk treatment

The Windows virtual audio device (part of `virtual-io`, and the reason `driver-windows` is its own crate) is the highest-risk single workstream in this plan, per decision spine #3. It is called out here separately from the `virtual-io` section above because it needs its own entry and exit criteria, not just a shared risk note.

**Why it is highest risk**:

1. **Signing.** As of the April 2026 breaking change, legacy cross-signed drivers are no longer trusted by Windows; the only paths are WHCP (Windows Hardware Compatibility Program) certification or attestation signing with an EV/Azure Trusted Signing certificate. Neither is a one-time cost: WHCP requires HLK (Hardware Lab Kit) test submission, and attestation signing has its own ongoing compliance requirements.
2. **HVCI.** Drivers must meet HyperVisor-protected Code Integrity requirements — memory safety and input validation constraints that go beyond normal kernel-driver hygiene and are non-negotiable for any driver expected to load on a default-configuration Windows 11 machine.
3. **The 24H2 lesson.** The reference documents a real incident: leftover outdated virtual-audio drivers caused a Windows 11 24H2 compatibility hold and system crashes, fixed only by reinstalling (updating) the driver. This is a concrete example of what happens when a virtual-audio kernel driver is not kept current with Windows Update's compatibility database — it is not a hypothetical risk.
4. **SDK not vendored.** Building a modern Windows audio driver means ACX (Audio Class eXtension, KMDF-based, Microsoft's recommended replacement for PortCls) against the WDK. The WDK itself is a Microsoft-distributed toolchain with its own licence terms; treat it the same as the ASIO SDK — supplied locally by whoever builds this workstream, never vendored into the repository.

**What must ship and test without it**: everything else. Decision spine #3 is explicit: the driver is isolated so that its schedule risk cannot block the rest of the plan. `backend-null` covers 100% of engine-level tests with no dependency on any virtual device; `PipeWire`-based Linux virtual nodes and the macOS `AudioServerPlugIn` HAL plug-in are lower-risk, user-mode-adjacent alternatives that can ship on their own schedule.

**Recommended approach**: reference Microsoft's own SysVAD sample (`microsoft/Windows-driver-samples`) for architecture only — its exact licence terms were not confirmed by research and must be checked before copying any code, not just its structure — and budget for WHCP certification lead time as a schedule item, not an afterthought. See `05-milestones-risks.md` for the open question on whether to ship a v1 kernel driver at all versus bridging through an existing virtual-cable product initially.

## ASIO: explicit risk treatment

Summarized from the `asio` section above because decision spine #3 calls it out specifically alongside the driver: ASIO support is only ever built against a locally-supplied Steinberg SDK (dual GPLv3/proprietary since October 2025), is single-device-open-at-a-time by the SDK's own design, and any CI coverage of it must be optional/skippable rather than a hard gate for contributors who have not installed the SDK. Treat "ASIO builds and passes its tests" as a maintainer-runner concern, not a default expectation for every contribution.
