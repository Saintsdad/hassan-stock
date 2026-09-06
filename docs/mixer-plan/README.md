# OPENMIX build plan

OPENMIX is a working codename, a placeholder. It has no relation to any existing product name and should be replaced before release. This directory is the full build plan for an open, cross-platform re-implementation of the "Mixboard Max" feature set described in `mixboard.txt`: the same routing matrix, DSP, NetAudio-compatible networking and remote control, built clean, with an automated test kit for every function, component and feature.

The plan is written to be executed by two AI coding agents plus a human in the loop:

- **Claude Opus 5** does the hard and critical work: architecture, audio-thread code, DSP algorithms, sample-rate conversion, drivers, protocol and state-machine code, security, and adversarial review of every Gemini Flash pull request. Opus 5 holds merge authority.
- **Gemini 3.8 Flash** does volume work: test generation from the feature/test matrix, golden-file fixtures, wrapper and SDK bindings, docs, UI widgets, CI plumbing, triage and mechanical refactors. Flash never touches audio-thread code without Opus review.
- A human decides the open questions in `05-milestones-risks.md`, approves licensing choices, and is the final merge gate for the Windows kernel driver workstream.

## The contract every row follows

Every one of the 406 rows in `feature-test-matrix.csv` is a unit of work with one required sequence:

1. **Matrix row.** Pick a row: its `feature`, `requirement` and `acceptance_test` fields define what "done" means.
2. **Failing test first.** Write the test named by `acceptance_test`, tagged with the row's `test_id`. It must fail (or fail to compile) before any implementation exists.
3. **Implement.** Write the minimum code to make that test pass, following the architecture in `01-scope-architecture.md` and the workstream detail in `02-workstreams.md`.
4. **CI green.** All gates in `03-test-kit.md` for the row's area pass, not just the new test.
5. **Cross-model review.** The other model reviews the diff using the checklist in `04-agent-operating-model.md` before merge. Opus reviews every Flash PR. Flash does a mechanical pass (style, test coverage, doc consistency) on Opus PRs but cannot block them.

This is the same loop for a one-line defect fix and for a multi-week driver workstream; only the size of the row and the review weight change.

## How to read this plan

| File | What it covers |
|---|---|
| `README.md` | This file. |
| `01-scope-architecture.md` | Scope, component diagram, crate/module layout, OS-backend trait, null backend, control protocol, transports, UI process boundary, the key numbers from the reference. |
| `02-workstreams.md` | One section per matrix `area`: goal, interfaces, hardest parts, licences chosen, risks, owner, milestone, row/test-id counts. |
| `03-test-kit.md` | Test layers, harnesses, CI gates, fixture policy, performance budgets, definition of done, example tests per area. |
| `04-agent-operating-model.md` | Opus vs Flash routing table, cost controls, task-spec template, PR rules, cross-model review checklist, escalation rules, agent config templates. |
| `05-milestones-risks.md` | M0 to M8 entry/exit criteria, row counts per milestone, critical path, top 10 risks, open questions for the human. |
| `06-part3-network-use-cases.md` | Part 3 (RTSP/IP-camera, multi-room, VPN, Bluetooth/2.4 GHz/roaming/clock) turned into requirements, and the matrix rows it adds. |
| `07-research-notes.md` | Condensed research with sources, unverified items flagged. |
| `templates/AGENTS.md`, `templates/CLAUDE.md`, `templates/GEMINI.md` | Ready-to-drop repository agent configuration files. |
| `plan.html` | This plan as one browsable page, with a filterable view of the full matrix. |
| `feature-test-matrix.csv` | The 406-row feature-to-test matrix. Source of truth for scope; this plan explains and elaborates it, never contradicts it. |

## What is out of scope for this plan

This plan does not write engine code, does not stand up CI infrastructure, and does not pick a final product name. It is the plan the two agents execute against. Naming, licensing model and the Windows-driver-v1 decision are called out as open questions in `05-milestones-risks.md` because they need a human decision, not an agent one.
