---
id: ORC-14
title: Delegate mode with take-over and recovery
priority: P0
score: TBD
effort: L
depends_on: [ORC-02, ORC-08, ORC-09, ORC-10, ORC-12, ORC-13]
milestone: M2
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-14] Delegate mode with take-over and recovery

## Problem
DESIGN §1 "Delegate" applies eligible choices automatically and escalates the rest. §4 steps 6-7 and §7 require one
pending automatic write per session, persisted pending records, take-over that cancels outstanding decisions, and
recovery that never repeats a keystroke. §8 M2's exit condition is "pass all terminal race, duplicate-delivery,
isolation, and recovery cases" on the supported CLI version. This item is the orchestrator side of M2 and owns the
§8 test matrix against the simulated host; ORC-15 repeats the positive control on a real host.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Missing | Any orchestrator code (`src/orchestrator/` (new) does not exist) | `src/lib.rs` (modules: auth, hub, launch, server, tray_support, web) |
| Missing | Decision pipeline for Observe/Recommend | `src/orchestrator/decision.rs` (proposed, ORC-10) |
| Missing | Journal and atomic pending store | `src/orchestrator/journal.rs` (proposed, ORC-08) |
| Missing | Host link | `HostExtLink` (proposed, ORC-12) in `src/hub_host_ext.rs` (new) |
| Conflict | Resume fails today: warp sends `session_secret: Default::default()`, `Hub::resume_session` returns `WrongPassword`, the terminal re-shares under a new id and enrollment is lost | `src/hub.rs` `Hub::resume_session`; warp `app/src/terminal/shared_session/sharer/network.rs`; fix is ORC-02 |
| Exists | Viewer input that reaches the sharer (remote take-over signal) | `src/hub.rs` `Hub::handle_viewer_message` |
| Exists | Scripted sharer for tests | `examples/fake_sharer.rs`; `--ext` (proposed, ORC-12), `--gate` (proposed, ORC-13) |
| Exists | E2E pattern over real TCP | `tests/websocket_e2e.rs` `start_server`, `connect` |

## Proposed change
- `src/orchestrator/delegate.rs` (new): the only module that holds a `HostExtLink` (proposed, ORC-12) sender. The
  Observe/Recommend task from ORC-10 never receives one (old AC-22 "no write path by construction" stays true).
  Per session: `Idle → Armed (decision.opened sent) → Dispatched (intent journaled, answer sent) → Resolved |
  Uncertain | Cancelled`. One `Dispatched` per session; independent sessions run in their own tasks so a slow
  decider in A never holds B.
- Dispatch preconditions: Delegate mode in the enrollment; a `hello` for the current `execution_id` and a fresh
  `state` on the link; prompt generation and `host_generation` unchanged since arming; the policy eligible set (ORC-08)
  still contains the option; decision-relevant evidence digests unchanged (DESIGN §6 "Recheck").
- No link, no `hello`, a link-down or a `take_over` means fail closed: escalate, journal, send nothing.
- Recovery: before sending, journal the attempt in the pending store (proposed, ORC-08). After a link drop with a
  dispatched attempt, resend the identical `answer` (same `attempt_id`) once on the new link to read its recorded
  status (ORC-13/AC-8); if the host has a new `execution_id` or cannot tell, mark `uncertain`. On orchestrator
  restart, dispatched records without a receipt become `uncertain` and are never re-sent.
- Take-over sources: tray action through the orchestrator handle (proposed, ORC-11), any remote-input event
  (ORC-12/AC-10), a host `take_over` or `decision.closed{human_input}`. Resume needs a fresh `hello` and observation.
- Confirmation: `answer.result` `submitted` without `confirmed` by the deadline shows "delivery uncertain" and re-observes.
- `tests/orchestrator_delegate_e2e.rs` (new): real router + `fake_sharer --ext --gate` (spawned like
  `fake_gate_e2e.rs` (new) in ORC-13.2) + scripted decider stub (proposed, ORC-09). Pattern: `tests/websocket_e2e.rs`.

## Scope / non-goals
- In: Delegate state machine, fail-closed rules, take-over, recovery, audit rows, the DESIGN §8 matrix (simulated host).
- Out: real-host positive control and `delegate_versions` enforcement (ORC-15), the host gate (ORC-13), a reviewer
  step, native requests (M3), a remote Take over button in `web/app.js` (remote input already counts as take-over).

## Acceptance criteria
- **ORC-14/AC-1**: While a Delegate session has no host link with a `hello` for its current `execution_id`, the orchestrator shall send no `answer` and keep the question as a pending human decision.
- **ORC-14/AC-2**: The orchestrator shall have at most one dispatched, unresolved answer per session.
- **ORC-14/AC-3**: When the orchestrator dispatches an answer, it shall journal the attempt in the pending store before sending the `answer` frame.
- **ORC-14/AC-4**: While an enrollment is in Observe or Recommend mode, the orchestrator shall hold no `HostExtLink` sender for that session.
- **ORC-14/AC-5**: When sessions A and B show identical menus with different briefs and A's decider is delayed, the orchestrator shall deliver B's answer without waiting for A.
- **ORC-14/AC-6**: The orchestrator shall send each answer only on the host link of the session whose evidence produced it.
- **ORC-14/AC-7**: When a delegated session prints "1/2/3" in a log or shows yes/no in old transcript text, the orchestrator shall send no `decision.opened` and no `answer`.
- **ORC-14/AC-8**: When a recognized menu is redrawn cosmetically 20 times, the orchestrator shall send at most one `answer` for it.
- **ORC-14/AC-9**: When a phone viewer answers while the decider is delayed and the verdict is then released, the simulated host shall receive zero additional keys.
- **ORC-14/AC-10**: When the simulated host reports local desktop input while the decider is delayed and the verdict is then released, the simulated host shall receive zero additional keys.
- **ORC-14/AC-11**: When Take over is invoked through the orchestrator handle while the decider is delayed and the verdict is then released, the simulated host shall receive zero additional keys.
- **ORC-14/AC-12**: If option order or meaning changes, the process is replaced, or the shell returns to idle between review and submission, the orchestrator shall not apply the stale answer.
- **ORC-14/AC-13**: If the prompt changes between a staged navigation step and its final Enter, the simulated host shall write no Enter to the new prompt.
- **ORC-14/AC-14**: If the host link drops before an answer is sent, the orchestrator shall not send that answer until a fresh `hello` and a re-armed decision.
- **ORC-14/AC-15**: If the host link drops after an answer is sent and before its `answer.result`, the orchestrator shall recover the status by resending the identical `attempt_id` once or mark the attempt `uncertain`.
- **ORC-14/AC-16**: If the orchestrator restarts after an attempt's intent is journaled and before its receipt is journaled, it shall mark the attempt `uncertain` and never re-send it.
- **ORC-14/AC-17**: Across every drop and restart case in ORC-14/AC-14 to ORC-14/AC-16, the simulated host shall receive each attempt's key at most once.
- **ORC-14/AC-18**: If a file grant is removed, a decisive excerpt's digest changes, or delegation is revoked during review, the orchestrator shall discard or re-evaluate the verdict instead of answering.
- **ORC-14/AC-19**: The orchestrator shall never pass an excerpt from an ungranted path to the decider in any delegate test.
- **ORC-14/AC-20**: When a menu offers allow once, always allow and deny under a one-time grant, the orchestrator shall never answer the persistent option even if the decider returns it.
- **ORC-14/AC-21**: If a permission menu lacks verifiable operation details, the orchestrator shall escalate it and send no answer.
- **ORC-14/AC-22**: When the decider returns `ask_user`, an unoffered option or a timeout, or the files contradict each other, the orchestrator shall keep a visible pending human decision and send zero answers.
- **ORC-14/AC-23**: When `take_over` comes from the tray, a remote viewer's input or the host, the orchestrator shall stop automatic answers for that session and cancel and journal its outstanding decisions.
- **ORC-14/AC-24**: After a take-over, the orchestrator shall resume delegation for that session only after a fresh `hello` and observation.
- **ORC-14/AC-25**: If an answer is `submitted` but not `confirmed` by its deadline, the orchestrator shall show "delivery uncertain", re-observe and never re-send the key.
- **ORC-14/AC-26**: For every dispatched answer, the orchestrator shall write a journal row with session, execution, prompt, option and attempt ids, mode and final status.
- **ORC-14/AC-27**: When a PR adds or changes the delegate path, the PR shall record a review with `.claude/skills/threat-modeling-agentic-tools`.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run orchestrator`; `cargo nextest run -E 'binary(orchestrator_delegate_e2e)'` |
| macOS | Same in CI (no tray dependency in these tests) |
| Linux | Same in CI |

DESIGN §8 mapping (simulated host): Simultaneous prompts AC-5, AC-6; False prompt and redraw AC-7, AC-8; Human wins race
AC-9..AC-11; Changed target AC-12, AC-13; Reconnect and crash AC-14..AC-17; Context and policy AC-18, AC-19; Permission
scope AC-20, AC-21; Inconclusive judgment AC-22. Positive control and the release criterion are ORC-15. Delays use
the stub decider's scripted latency, never wall-clock sleeps longer than the test needs.

## Security / license review
Touches the remote-control surface (the first automatic input path): needs the
`.claude/skills/threat-modeling-agentic-tools` review (ORC-14/AC-27). Focus: fail-closed defaults (AC-1, AC-14),
duplicate input (AC-15..AC-17), cross-session leakage (AC-6), authority creep from provider output (AC-20, AC-22),
evidence exfiltration (AC-19). No new crates expected beyond those ORC-05/ORC-09 add.

## Overlaps and existing work
- Replaces warpr-g7d60 (T-13). Old AC-25/AC-26/AC-30 are split here; their missing sub-cases are added (desktop and
  take-over repeats AC-10/AC-11, staged Enter AC-13, the third drop point AC-16, ungranted file AC-19, correct process AC-6).
- Needs ORC-02 (resume keeps the session id), ORC-08 (policy, journal, pending store), ORC-09 (decider stub),
  ORC-10 (pipeline), ORC-12 (link), ORC-13 (gate contract and `fake_sharer --gate`).
- Roadmap: docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/AGT-07.md (AGT-07.1 local task queue, a different
  layer); items/SEC-01.md (SEC-01.3 trust store would later feed enrollment grants). Cite, do not duplicate.

## Open questions (owner to confirm)
- Confirmation deadline for AC-25: recommend a per-adapter value from ORC-03 captures; alternative one global default.
- Remote take-over: recommend any relayed viewer input (ORC-12/AC-10) counts; alternative only `WriteToPty`/`ExecuteCommand`.
- Re-arming after take-over: recommend the user re-enables Delegate per session from the tray; alternative automatic
  after a fresh `hello` and a quiet period.

## References
- DESIGN.md §1, §4, §5, §7, §8.
- `src/hub.rs`, `examples/fake_sharer.rs`, `tests/websocket_e2e.rs`; items ORC-08, ORC-10, ORC-12, ORC-13.
