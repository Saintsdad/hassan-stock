# Agent operating model

This is how Claude Opus 5 and Gemini 3.8 Flash actually work the matrix, in cost, process and review terms. It elaborates decision spine's "agent operating model" section; it does not change it.

## Model pricing and shape (from research, September 2026 — verify before relying on for budgeting)

| | Claude Opus 5 | Gemini 3.8 Flash |
|---|---|---|
| Context | 1M in / 128K out | 1M in / 64K out |
| Price (input/output per Mtok) | $5 / $25 | $0.75 / $3.75 (introductory, through Dec 31 2026; rises to $1.50/$7.50 in Jan 2027) |
| Strength | Multi-file features, long-horizon coherence (20+ step workflows), 79.2% SWE-bench Pro | Fast iteration, high-volume mechanical work, batch/cached pricing for repeated similar tasks |
| Known weakness | Verbose, scope creep, cost at volume | Instruction drift over long conversations, weak on algorithmic edge cases and complex multi-file refactors, higher hallucination rate on factual claims |

The general routing principle research surfaces (roughly 80% cheap-model / 20% premium-model traffic, with the premium model handling the harder 20%, outperforms single-model use on cost, latency and throughput) is why the split below is not 50/50: most matrix rows are mechanical once the hard design decisions are made, and that mechanical volume is Flash's job.

## Routing table: task type -> model -> effort

| Task type | Model | Effort/notes |
|---|---|---|
| Architecture decisions, crate boundaries, trait design | Opus | High effort; these are the decisions everything else depends on — get them reviewed by the human before wide implementation starts. |
| Audio-thread code (anything running in the real-time callback path) | Opus | High effort, always. Never Flash, not even a "simple" one-line change — a one-line change to audio-thread code can introduce a lock or allocation invisibly. |
| DSP algorithms (EQ, dynamics, reverb, denoiser, SRC) | Opus | High effort. Flash may write the golden-fixture generation scripts around an Opus-authored algorithm, not the algorithm. |
| Protocol/state-machine code (control protocol, NetAudio wire format, scripting VM) | Opus | High effort for the state machine itself; Flash may add additional test cases against an Opus-defined state machine. |
| Driver code (Windows kernel driver, macOS HAL plug-in) | Opus | Highest effort, treated as its own workstream per `02-workstreams.md`; human sign-off required before merge regardless of model. |
| Security-sensitive code (encryption, input validation on any network-facing parser) | Opus | High effort, always. See "what Flash must never touch" below — this is non-negotiable. |
| Test generation from a matrix row, once the interface is defined | Flash | Medium effort. This is Flash's primary job: turn `acceptance_test` text into a real, passing (once implemented) test. |
| Golden-file fixture generation (given an Opus-approved reference algorithm) | Flash | Medium effort. |
| Wrapper/SDK bindings (Python/Go/Node wrappers, CLAP-hosting glue once the CLAP surface is defined) | Flash | Medium effort. |
| Documentation (user-facing docs, this plan's own upkeep, code comments) | Flash | Low-medium effort. |
| UI widgets (once the control-protocol contract and a11y requirements are fixed) | Flash | Medium effort; Opus reviews any widget that could send unexpected control-protocol calls. |
| CI plumbing (pipeline config, fixture pinning, dependency bumps) | Flash | Low-medium effort; Opus reviews any change to a gating CI rule itself (not routine fixture updates). |
| Triage (categorizing a failing test, a fuzz crash report, a flaky test) | Flash | Low effort for the categorization; escalate the underlying fix per the table above based on what area it touches. |
| Mechanical refactors (rename, extract-function, dependency version bump with no behaviour change) | Flash | Low effort, but still requires the row's/PR's normal CI gate and a review — "mechanical" does not mean "unreviewed." |
| Cross-model adversarial review of Flash's own work | Opus | Every PR Flash opens. See checklist below. |
| Cross-model mechanical review of Opus's own work | Flash | Every PR Opus opens, style/coverage/doc-consistency pass; Flash cannot block an Opus PR (see PR rules). |

## Cost-control rules

1. **Route by task type first, not by "who's cheaper."** A hard task done twice (once wrong by Flash, once right by Opus after review rejection) costs more than doing it once with Opus. Use the table above as the default; deviate only with a stated reason in the PR description.
2. **Batch Flash's high-volume, low-urgency work.** Test-generation sweeps across many rows in one area are a good fit for batch-style processing (up to 50% pricing discount per research, 24-hour-class turnaround) rather than one-row-at-a-time interactive calls.
3. **Cache repeated context.** Flash's context caching (per research, up to ~90% input-token savings on high-volume repeated context) should be used when running many similar test-generation tasks against the same matrix/spec context, rather than re-sending the full matrix and spec on every call.
4. **Bound Opus's verbosity.** Research flags Opus 5 as prone to verbose output and "verifying its own work without being asked" — both cost money at Opus's per-token rate. Task specs (template below) should explicitly state the expected output shape and explicitly say when self-verification is/isn't wanted, rather than leaving it open-ended.
5. **Don't let Flash's instruction drift compound.** Research flags instruction drift as an architectural weakness that worsens as a Flash conversation extends. Keep Flash's task sessions scoped to one matrix-row group (see PR rules) rather than one long-running session across many rows; a fresh, tightly-scoped prompt per PR-sized unit of work costs less overall than debugging drift in a long one.
6. **Escalate early, not late.** A Flash task that is clearly hitting one of Flash's documented failure modes (algorithmic edge cases, complex multi-file refactor, factual hallucination on a spec detail) should escalate to Opus before more Flash tokens are spent iterating, not after several failed attempts.

## Task-spec template

Every task handed to either model (as a PR description, an issue, or a direct prompt) should include:

```markdown
## Matrix row(s)
<one or more F-XXXX ids, and their test_id(s)>

## Area / owner per matrix
<area value>, owner: <opus|flash|opus+flash>

## Goal
<what "done" means, in the acceptance_test's own words, not a summary>

## Interfaces touched
<which crates/modules; whether this crosses into audio-thread code>

## Constraints
<zero-alloc/zero-lock if audio-thread; licence constraints from 02-workstreams.md if relevant;
 "never touches X without Opus review" if it applies>

## Expected output shape
<a failing test first, then an implementation PR, or a single PR with both — state which>

## Self-verification expectation
<explicitly say whether the model should run/verify its own tests before opening the PR,
 to bound Opus's tendency to over-verify and Flash's tendency to under-verify>
```

## PR rules

1. **Size**: one matrix-row group per PR. A "row group" is either a single row, or a small cluster of rows in the same area that are meaningless to split (e.g. the 9 reverb room-type presets under `T-fx-003`-adjacent rows can be one PR). A PR spanning multiple areas, or spanning more than roughly 400 lines of non-test diff, should be split — this mirrors the ~200-400 line guidance research surfaces for fast-model-first branches, applied here to both models for consistency.
2. **Tests first.** The PR's first commit is the failing test(s) carrying the row's `test_id`(s). The implementation commit(s) follow. A PR that adds implementation and tests in one commit with no visible "tests failed, then passed" history is a process violation, not just a style nit — reviewers should ask for it to be split before merging, not merge it and note it for later.
3. **One matrix row group per PR** (repeated because it is the most commonly violated rule in practice): resist the temptation to "also fix" an adjacent row while in the area. File it as a separate row/PR.
4. **CI must be green before requesting review**, including the row's specific `test_types` gates.
5. **No self-merge.** Opus holds merge authority per the decision spine; Flash cannot merge its own PR even if Opus's automated review passed, without an explicit Opus (or human) merge action.

## Cross-model adversarial review checklist

Opus runs this against every Flash PR before merge. Treat "adversarial" literally: assume the test could be gamed or the implementation could be subtly wrong, and look for it.

- [ ] Does the test actually implement the row's `acceptance_test` text, or a weaker paraphrase of it?
- [ ] Could this test pass against a broken or incomplete implementation (a false positive)? Try to think of one.
- [ ] Does the implementation touch audio-thread code anywhere, directly or transitively? If yes, this PR should not have been Flash-owned in the first place — reject and re-route.
- [ ] Are all of the row's declared `test_types` actually present and passing, not just the easiest one?
- [ ] Does a golden-file fixture actually encode the documented tolerance, not an arbitrarily loose one chosen to make the test pass?
- [ ] Does a wrapper/binding change actually get exercised by the Remote-API shim conformance run, or only by a hand-written stub?
- [ ] Is there any hallucinated fact (a made-up spec detail, a fabricated licence claim, an invented API) that should instead cite `07-research-notes.md` or be flagged as an open question?
- [ ] Does the PR stay within its one row-group scope?

Flash runs a lighter, non-blocking checklist against every Opus PR (style, coverage-by-test_id, doc consistency with `02-workstreams.md`/`03-test-kit.md`) and reports findings as PR comments; per the decision spine, Flash's review does not gate an Opus merge, but a finding should still get a human's attention if Opus dismisses it without comment.

## Escalation rules

- A Flash task that fails its own CI gate twice on the same row escalates to Opus rather than a third Flash attempt.
- Any task touching the Windows kernel driver, ASIO SDK integration, or the security area escalates to human sign-off before merge, regardless of which model wrote it, per `02-workstreams.md`'s risk treatment.
- Any disagreement between Opus's adversarial review and Flash's self-assessment that isn't resolved in one review round escalates to the human rather than a second automated round — this is a discovered ambiguity in the matrix row itself, most likely.
- A fuzz target that finds a new crash on a security-area or netaudio-area module escalates immediately (not at the next scheduled triage) to Opus, because these are exactly the modules decision spine treats as highest-consequence.

## What Flash must never touch

Restated plainly, gathering the constraints above in one place:

- Audio-thread code in `engine-core`, `engine-dsp`, or any crate's real-time callback path, under any circumstance, including "trivial" changes.
- The `security` area's implementation code (test-writing under Opus's design is fine; the implementation is not).
- The Windows kernel driver and macOS HAL plug-in implementation code (`driver-windows`, `driver-macos`).
- The ASIO SDK integration code in `backend-windows` (SDK-facing glue, not the ASIO-adjacent UI or config-file work, which is fine for Flash).
- Merge action on its own PRs.
- Changes to the CI gates themselves (what blocks a merge), as opposed to routine fixture/dependency maintenance within those gates.
- The control-protocol's parameter-grammar or wire-format definition (Flash may add client-side bindings against an Opus-defined grammar).

## Agent configuration templates

Repository-level agent configuration files live under `docs/mixer-plan/templates/`: `AGENTS.md` (shared, tool-agnostic conventions per the Linux Foundation Agentic AI Foundation's minimal-markdown convention), `CLAUDE.md` (Claude Code-specific: hierarchical scopes, this operating model), and `GEMINI.md` (Gemini CLI-specific). Per research's best-practice recommendation, shared instructions live in `AGENTS.md` and tool-specific tweaks live in the other two files rather than duplicating everything three times. Copy these into the actual project repository root (and into subdirectories where a path-scoped rule is needed) when implementation begins; they are templates here, not yet wired into a live repository.
