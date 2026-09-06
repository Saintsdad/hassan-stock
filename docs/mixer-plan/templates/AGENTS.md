# AGENTS.md — OPENMIX (working codename, placeholder)

Shared, tool-agnostic instructions for any coding agent working in this repository. Tool-specific tweaks live in `CLAUDE.md` and `GEMINI.md`; this file is the common ground both build on.

## What this project is

An open, cross-platform re-implementation of the "Mixboard Max" feature set: an 8-strip x 8-bus routing matrix (34 input channels, 64 output channels), full DSP chain, VBAN-compatible NetAudio plus an optional sync transport, and a compatible Remote-API control surface. Full plan: `docs/mixer-plan/`. Source of truth for scope: `docs/mixer-plan/feature-test-matrix.csv`.

## The contract

Every unit of work is a row in `feature-test-matrix.csv`:

1. Pick a row. Its `requirement` and `acceptance_test` fields define done.
2. Write the test named by `acceptance_test`, tagged with the row's `test_id`. It must fail first.
3. Implement the minimum to make it pass.
4. All CI gates for the row's area pass.
5. The other model reviews before merge.

Do not implement a row without its failing test existing first. Do not close a row's test coverage with a weaker paraphrase of its `acceptance_test`.

## Build and test commands

```
cargo build --workspace                       # build everything that doesn't need a platform SDK
cargo test --workspace --features backend-null  # run the full null-backend test suite
cargo test -p engine-core                     # a single crate
cargo bench                                   # criterion perf benchmarks (see docs/mixer-plan/03-test-kit.md)
cargo fuzz run <target>                       # fuzz a specific target, bounded time for local runs
```

Platform-specific backends (`backend-windows`, `backend-macos`, `driver-windows`, `driver-macos`) require their target OS and are behind feature flags — do not expect them to build on a Linux CI runner, and do not add them to the default feature set.

## Code style and architecture constraints

- Zero allocation, zero locking on the audio thread. This is enforced (`assert_no_alloc` in debug/test builds, an RTSan-class sanitizer pass in CI), not a style preference.
- All engine code goes through the `AudioBackend` trait (`backend-trait/`). Never call an OS audio API directly from `engine-*` crates.
- The UI process talks to the engine only through the control protocol (`control-protocol/`). It has no direct engine dependency.
- The ASIO SDK and the Windows WDK are never vendored into this repository. They are supplied locally by whoever builds those specific workstreams.
- Follow the parameter grammar already established in `control-protocol/` (`Strip[i].Gain` style) for any new parameter — do not invent a parallel naming scheme.

## Where to look

- `docs/mixer-plan/01-scope-architecture.md` — architecture and the key reference numbers.
- `docs/mixer-plan/02-workstreams.md` — per-area goals, licences, risks.
- `docs/mixer-plan/03-test-kit.md` — test layers, harnesses, CI gates, definition of done.
- `docs/mixer-plan/04-agent-operating-model.md` — who does what, PR rules, review checklist.
- `docs/mixer-plan/05-milestones-risks.md` — milestones, open questions for the human.

## What no agent should do unilaterally

- Change licensing terms or add a dependency with a licence not already named in `docs/mixer-plan/02-workstreams.md` without flagging it for human review first.
- Merge a PR touching the Windows kernel driver, ASIO SDK integration, or the `security` area without human sign-off.
- Rename the project or change the OPENMIX placeholder codename — that is an explicitly open human decision (`docs/mixer-plan/05-milestones-risks.md`).
