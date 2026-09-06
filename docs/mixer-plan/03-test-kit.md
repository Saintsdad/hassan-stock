# Test kit

This is the full test strategy. Every matrix row's `test_types` field is one or more of the layers below; every area maps to at least one harness; CI gates decide what blocks a merge; fixture policy decides what "deterministic" means in practice.

## Layers

| Layer | What it proves | Runs where |
|---|---|---|
| **unit** | A single function/module behaves per spec, in isolation. | Every CI run, every platform. |
| **property** | An invariant holds across randomly generated inputs (no NaN/Inf, energy conservation, monotonic decay, range validation). | Every CI run, `backend-null`. |
| **golden** | Output matches a stored reference file within a documented tolerance. | Every CI run, `backend-null`. |
| **integration** | Multiple modules work together correctly (e.g. routing + DSP + recorder). | Every CI run, `backend-null`, plus platform-specific runs where the row names a real backend. |
| **interop** | Wire/behavioural compatibility with an independent third-party implementation. | Dedicated interop CI job, not every PR. |
| **fuzz** | No panic/crash/memory-safety violation under adversarial or malformed input. | Continuous fuzzing job, not gating every PR (see CI gates below), plus a bounded-time run per PR touching a fuzzed target. |
| **e2e-ui** | The UI, driven like a user would, produces the right screen and the right accessibility tree. | Dedicated UI CI job (Playwright + axe-core). |
| **perf** | A latency/CPU/memory budget is met, and does not regress. | Dedicated perf CI job, nightly plus a lighter check on PRs touching audio-thread code. |
| **manual** | A human (or a scripted human-in-the-loop VM lane) confirms something CI genuinely cannot, such as real OS-update compatibility or physical Bluetooth/2.4 GHz behaviour. | Milestone gate, not per-PR. |

A row can carry more than one type (`unit,golden`, `integration,perf`, etc.); all listed types must pass.

## Harnesses

### Offline render harness

Renders a full session (routing + DSP + recorder + NetAudio send/receive) with no real-time callback and no audio-thread preemption: input is a buffer or file, output is a buffer or file, computed as fast as the CPU allows. This is what makes golden-file and property tests bit-reproducible: same input, same seed, same output, every run, on every machine. LFO/pitch-wobble/random-trigger elements use fixed seeds specifically so a "random" test is still deterministic across reruns.

### Null/loopback backend

`backend-null` (see `01-scope-architecture.md`) is not a mock in the usual sense — it is a real, if simple, implementation of the `AudioBackend` trait: an in-process ring buffer with a virtual clock the test harness drives. Every `engine`, `dsp-strip`, `dsp-bus`, `fx`, `netaudio`, `control-api`, `recorder`, `macro`, `midi`, `routing`, `scripting` and `security` row runs against it. `devices`, `virtual-io`, `asio`, `platform` rows that specifically test a real OS backend's behaviour cannot run against it and are routed to their platform-specific lane instead.

### VBAN interop rig

A dedicated CI job (not run on every PR — see CI gates) that starts OPENMIX's `engine-net` plain-transport implementation alongside one or more independent, out-of-process VBAN-family implementations as black-box fixtures:

- **PipeWire's `module-vban-send`/`recv`** (LGPL-2.1+), run as a separate PipeWire instance in the job's container.
- **`quiniouben/vban`** (GPLv3) command-line emitter/receptor tools, run as a separate process — used only as an external test fixture, never linked into the engine binary, which keeps its GPLv3 licence from applying to OPENMIX itself.
- **`vban-cmd` (Python)** for TEXT/SERVICE subprotocol interop checks specifically.

The rig streams a known signal in both directions and asserts correct audio transfer, correct header/sub-protocol handling, and correct behaviour under injected packet loss/reordering. This is the primary tool for de-risking the netaudio area's biggest open question (the VBAN v13 spec PDF was not fully accessible to research — see `07-research-notes.md`): if OPENMIX and multiple independent implementations agree, the wire format is right regardless of any gap in the plan's secondhand description of the spec.

### Remote-API shim conformance run

Runs the compat shim DLL (control-api area) against the actual existing third-party wrapper libraries it must remain compatible with — python, Go and Node wrappers written against the incumbent's Remote API — rather than only against hand-written test doubles. A wrapper library calling the shim exactly as it would call the real incumbent DLL, and getting correct results back, is the conformance bar. This run is what catches an ABI mismatch a purely internal unit test would miss.

### GUI a11y + screenshot tests

Playwright drives the UI process (talking to a running engine, or a scripted `backend-null` engine instance) and asserts two things per scenario: `toMatchAriaSnapshot()` (a YAML accessibility-tree comparison, stable across CSS and rendering changes) and `toHaveScreenshot()` (pixel-diff against a baseline, with a configured threshold for anti-aliasing noise). `axe-core` runs alongside for WCAG rule scanning. A UI change that passes the screenshot diff but fails the aria snapshot (or vice versa) is a failing test, not a warning — this is the mechanism that keeps the reference's fixed-pixel/no-accessibility defect from creeping back in.

### Latency/glitch measurement rig

For `dsp-strip`, `dsp-bus`, `fx`, `network-ext` and `virtual-io` perf rows: a loopback-style measurement (real hardware loopback where the row needs real backend timing, `backend-null`'s virtual clock where it does not) that computes end-to-end latency and looks for glitches (buffer underrun/overrun events, discontinuities) under a documented load profile. Regressions beyond a configured tolerance (see performance budgets below) fail the perf gate.

### Driver test VM lane

For `virtual-io` (Windows kernel driver, macOS HAL plug-in), `asio` and `platform` rows that need a real OS: dedicated VM-based CI lanes (Hyper-V/VirtualBox-class for Windows with test-signing enabled during development; a macOS runner for the HAL plug-in and notarization checks). These lanes are not part of the default PR gate — they run on a schedule and before milestone exit, because they are slow, sometimes flaky by nature (real OS audio stacks), and in the ASIO case, gated on SDK availability on the runner (see `02-workstreams.md`'s ASIO risk treatment).

## CI gates and required coverage per area

| Gate | Trigger | Blocks merge? |
|---|---|---|
| Unit + property + golden + integration (backend-null) | Every PR | Yes |
| e2e-ui (Playwright + axe-core) | Every PR touching `ui/` or `control-protocol/` | Yes |
| Fuzz (bounded, e.g. 60s per changed fuzz target) | Every PR touching a fuzzed module (`netaudio`, `control-api`, `scripting`, `security`) | Yes (a bounded run passing; the continuous long-running fuzz job below runs separately) |
| Continuous fuzzing (unbounded, corpus-accumulating) | Nightly / continuous | No (files an issue on new crash, which does block that specific area's next merge until triaged) |
| Interop rig | Every PR touching `engine-net`, `network-ext` | Yes |
| Perf (lightweight) | Every PR touching an audio-thread crate | Yes, against the committed baseline with the tolerance in the table below |
| Perf (full/nightly) | Nightly | No (regression report, human-reviewed) |
| Driver/VM lanes | Scheduled + milestone exit | No per-PR (milestone gate only) |
| Manual | Milestone exit | Milestone gate only |

Required coverage per area: every row's `test_id` must have a corresponding test in the codebase before the row is considered implemented (this is the row-to-test contract from `README.md`). Coverage is tracked by `test_id`, not by line-coverage percentage — a row with one well-chosen property test can be "fully covered" while a row with ten shallow unit tests may not be.

## Fixture policy

- **Deterministic seeds.** Every property test and every fixture that involves randomness (LFO phase, jitter simulation, packet-loss injection) takes an explicit seed, logged on failure, so a CI failure is reproducible locally with the exact same seed.
- **Golden files with tolerances.** No golden comparison is bit-exact by default. Each golden test declares its tolerance explicitly in the test itself (for example, ±0.0001 amplitude for an EQ null test, ±0.5 dB for a frequency-response test, a documented sample-count window for latency). A golden file is regenerated only through an explicit, reviewed "update golden" step, never silently by a failing test's own re-run.
- **Interop fixtures are pinned.** The VBAN interop rig and the Remote-API wrapper conformance run pin exact versions of the external tools/libraries they use (PipeWire version, `quiniouben/vban` commit, wrapper library versions); bumping a pinned version is its own reviewed change, not an incidental side effect of an unrelated PR.
- **Real-hardware-dependent fixtures never gate the default PR pipeline.** Bluetooth, 2.4 GHz dongles, physical ASIO devices, and physical ES device hot-plug all require manual or VM-lane testing; a contributor without that hardware must still be able to get a green default CI run.

## Performance budgets

| Budget | Target | Where enforced |
|---|---|---|
| Audio-thread allocation | Zero, always | `assert_no_alloc` in every debug/test build of `engine-core`/`engine-dsp`; a CI failure, not a warning. |
| Audio-thread locking | Zero, always | RTSan-class sanitizer pass on Linux/macOS CI; `#[nonblocking]`-annotated audio-thread functions. |
| Parameter-change to audible-effect latency | Documented ~5-10 ms class (per `03-dsp.md` research), regression-gated at whatever exact figure the implementation lands on | Latency/glitch measurement rig, perf gate. |
| Denormal handling | FTZ/DAZ enabled on the audio thread; no measurable CPU spike on near-silent input | Property test injecting subnormal-range signals; perf gate watches for the spike pattern specifically. |
| Criterion-class regression tolerance | ±10% of baseline computation time per benchmarked DSP block | Nightly full perf run; a PR-time lightweight check uses a looser bound to avoid false failures from CI noise. |
| NetAudio round-trip latency (LAN) | Matches or improves on the reference's documented 5-20 ms small-buffer / 100+ ms large-buffer trade-off | Interop rig + latency rig. |
| Multi-room sync-transport alignment tolerance | A documented tight tolerance, materially better than the plain-protocol baseline (per T-network-ext-002's acceptance test) | network-ext perf gate. |

## Definition of done (per matrix row)

A row is done when all of the following hold, not just the first:

1. A test exists carrying the row's exact `test_id`, and it implements the row's `acceptance_test` field, not a paraphrase of it.
2. Every `test_types` value listed for the row has a corresponding passing test/job.
3. The implementation passes the relevant CI gates above, including any perf/fuzz gate applicable to the crate it touches.
4. The other model has reviewed the PR per `04-agent-operating-model.md`'s checklist and the row's owner-appropriate rules were followed (a `flash`-owned row touching audio-thread code was NOT implemented by Flash without Opus review, etc.).
5. The row's milestone gate (see `05-milestones-risks.md`) has not been reached with the row still open — a row that slips its milestone is a flagged risk, not a silent carry-over.

## Example tests per area (2-3 concrete, citing test_ids)

- **engine**: `T-engine-001` (8x8 routing matrix topology) as a property test generating random valid strip/bus pairs and asserting every pair is independently routable; `T-engine-002`/`T-engine-003` (physical/virtual strip channel widths) as unit tests asserting exactly 10 and 24 channels respectively are exposed.
- **dsp-strip**: `T-dsp-strip-001` (Strip Mute) as a unit test asserting zero output regardless of upstream signal; `T-dsp-strip-003` (Strip Mono for hardware strips) as a golden test comparing summed-mono output against a reference file within amplitude tolerance.
- **dsp-bus**: `T-dsp-bus-003` (Bus fader gain) as a combined property+golden test — property for gain-curve monotonicity across the full range, golden for exact dB-to-linear conversion at documented reference points.
- **fx**: `T-fx-001` (Reverb enable/AB) as a golden test comparing wet/dry output against a reference impulse response within a documented RT60 tolerance; `T-fx-003` (9 reverb room-type presets) as a parameterized golden test, one fixture per preset.
- **netaudio**: `T-netaudio-001` (wire protocol identifier compatibility) as an interop test against the VBAN interop rig; `T-netaudio-002` (28-byte little-endian packed header) as a fuzz target feeding malformed headers and asserting no panic.
- **network-ext**: `T-network-ext-002` (sync-transport timestamps) as an integration+perf test measuring cross-receiver alignment under simulated differing network paths; `T-network-ext-011` (RTSP ingest) as an interop test against a local RTSP fixture server.
- **control-api**: `T-control-api-001` (compat ABI shim) as an integration+interop test running an existing third-party wrapper library against the shim; `T-control-api-002` (login/logout lifecycle) as a unit test on the protocol state machine.
- **recorder**: `T-recorder-003` (recording formats) as a parameterized integration test, one case per format (WAV/AIFF/BWF/MP3), asserting correct headers and decodable output.
- **ui**: `T-ui-001` (resizable, DPI-aware main window) as an e2e-ui test resizing the window and asserting both the screenshot and the aria snapshot reflow correctly; `T-ui-003` (accessible UI) as an axe-core scan asserting zero WCAG violations on the main screen.
- **macro**: `T-macro-001` (button count/index range) as a property test; `T-macro-002` (Push vs 2 Positions) as an e2e-ui test clicking each button type and asserting the correct state-machine behaviour.
- **devices**: `T-devices-003` (physical input interface set) as an integration test against a simulated device list; hot-plug rows use the VM lane where real OS device-change events are needed.
- **midi**: `T-midi-002` (MIDI Learn workflow) as an e2e-ui test driving the learn UI and a simulated MIDI input.
- **routing**: `T-routing-001` (strip-to-bus routing bit matrix persistence) as an integration test asserting routes survive a save/reload cycle.
- **virtual-io**: `T-virtual-io-001` (virtual playback endpoints) as a unit+integration test against `backend-null`'s virtual-device emulation, promoted to the VM lane for the real Windows/macOS case.
- **scripting**: `T-scripting-003` (Wait(ms) sequencing) as a unit test on the script VM's timing model.
- **asio**: `T-asio-002` (ASIO SDK never vendored) as a build-system integration+manual check — CI asserts no SDK files are present in the repository; a human confirms local-SDK build instructions remain accurate at milestone review.
- **settings**: `T-settings-002` (settings versioning and daily backups) as an integration test running a simulated multi-day session and asserting backup files exist and are restorable.
- **platform**: `T-platform-003` (Linux PipeWire/ALSA backend) as an integration test against a real (containerized) PipeWire instance, distinct from `backend-null`.
- **security**: `T-security-002` (NetAudio-TEXT script injection hardening) as a fuzz test feeding adversarial script strings and asserting no unintended command execution; `T-security-003` (NetAudio wire-format fuzzing) as a continuous fuzz target.
- **defect-fix**: `T-defect-fix-002` (robust behaviour across OS feature/driver updates) as an integration+manual test — automated where a VM snapshot of an OS update can be simulated, manual where it genuinely cannot.
