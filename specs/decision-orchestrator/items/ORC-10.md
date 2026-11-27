---
id: ORC-10
title: Decision pipeline: Observe and Recommend
priority: P0
score: 9
effort: L
depends_on: [ORC-04, ORC-05, ORC-06, ORC-07, ORC-08, ORC-09]
milestone: M1
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-10] Decision pipeline: Observe and Recommend

## Problem
DESIGN.md §4 defines the lifecycle (observe, recognize, evaluate scope, gather and decide) for "one
pending question in one session", with "one pending automatic write per session, with a bounded pool
of independent review jobs across sessions. An unanswered question in session A cannot hold session
B's input lock." The M1 exit condition (§8) is: "Two concurrent sessions produce correctly attributed
recommendations. The worker has no write capability in recommend mode." §7 adds that a slow job
"still answers only its originating session" and that prompt changes or process exits cancel the old
decision. This item joins the parts from ORC-05 to ORC-09 into that per-session pipeline.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Missing | `src/orchestrator/` (new, ORC-05.1) and every pipeline stage | - |
| Missing | Passive observer stream (planned in ORC-04 as `Hub::attach_observer`) | `src/hub.rs` `Hub::join_session` is the viewer path today |
| Exists | Viewer input reaches the sharer as `WriteToPtyRequested`, `CommandExecutionRequested`, `ViewerTerminalSizeReported`, `AgentPromptRequested`, `ControlActionRequested` | `src/hub.rs` `Hub::handle_viewer_message` |
| Exists | In-process session creation and sharer messages for tests | `src/hub.rs` `Hub::create_session`, `Hub::handle_sharer_message`; `src/hub_tests.rs` |
| Exists | End-to-end test pattern over the real router | `tests/websocket_e2e.rs` `start_server` |
| Exists | `tokio::sync` (Semaphore, mpsc) is already enabled | `Cargo.toml` tokio features |
| Exists | Notify-on-change pattern for the tray | `src/tray_support.rs` `LogTail::create` (`on_write`) |

## Proposed change
1. **Session state machine** in `src/orchestrator/decision.rs` (new), tests in
   `src/orchestrator/decision_tests.rs` (new). `SessionPipeline` holds the enrollment (ORC-07.2),
   the current prompt identity `(prompt_id, generation)` from ORC-06, and a status:
   `Idle`, `Observed`, `Deciding`, `Recommended { option_id, rationale }`,
   `Escalated { reason }`, `Manual` or `Cancelled { reason }`.
   - Inputs: recognizer events (opened, updated with a new generation, closed), run-state and gap
     signals from ORC-05, decider results tagged with the `(prompt_id, generation)` they were asked
     for, and take-over / resume.
   - Observe: record the prompt and journal it; never build evidence or call the decider.
   - Recommend: ORC-08 policy gives the eligible options (an empty set escalates), ORC-09.1 builds
     the packet, the decider runs as a job, and the result is applied only if its tag still matches
     the current prompt. A stale result is discarded and journaled as stale.
   - Closed prompt, new generation, session end or take-over cancel the job (drop its handle) and
     journal the reason. Take-over sets `Manual` until `resume` (M1 meaning; ORC-14 extends it to
     remote and host take-over).
   - Replay or an unresolved gap (ORC-05) opens no decision.
   - **Structural no-write:** `SessionPipeline::observe_or_recommend(..)` takes the observer
     receiver, the decider and the journal, and nothing that can write. `HostExtLink` (new, ORC-12)
     does not exist in M1; ORC-14 keeps the only sender in `src/orchestrator/delegate.rs` (new, ORC-14). The receiver from ORC-04 has
     no upstream path. A unit test asserts `has_write_path() == false` for both modes.
2. **Supervisor** in `src/orchestrator/mod.rs` (new, ORC-05.1) extends `OrchestratorHandle` (ORC-07.3):
   - `enroll` attaches an observer (ORC-04) and spawns one task per session that runs the ORC-05
     observer and screen, the ORC-06 recognizer and the `SessionPipeline`.
   - `unenroll`, `SessionEnded` and `take_over` detach or cancel.
   - Decider jobs share a `tokio::sync::Semaphore` sized by `max_concurrent_decisions` (ORC-07.1),
     so a slow job in A takes one permit and never blocks B's task.
   - `snapshot() -> Vec<SessionView>` merges `Hub::list_sessions` (host, pwd) with pipeline state;
     `drain_notices()` returns recommendation and escalation events. An `on_change` callback, like
     `LogTail::create`, lets the tray call `notice.notice()` (ORC-11).
   - Journal rows go through ORC-08: session id, prompt id, generation, mode, eligible ids, evidence
     references and digests, provider and model, outcome, reason.
3. **End-to-end test** `tests/orchestrator_e2e.rs` (new): an in-process `Hub`, two sessions created
   with `Hub::create_session`, the real-capture menu streams from ORC-03 fed through
   `handle_sharer_message`, the `jev_stub` example (ORC-09.2) or an in-process slow `Decider`, and both
   sharer channels checked for input messages.

## Scope / non-goals
- In: per-session state machine, supervisor, bounded job pool, journal rows, snapshot and notices for
  the UI, M1 take-over (cancel and suspend).
- Not: Delegate mode, conditional answers, attempt ids, recovery (ORC-14); the reviewer step; tray
  rendering (ORC-11); a WebSocket transport for the orchestrator.

## Acceptance criteria
- **ORC-10/AC-1**: When sessions A and B show the same menu and A's decider is slow, the pipeline shall deliver B's recommendation before A's.
- **ORC-10/AC-2**: When a recommendation is produced, the pipeline shall attribute it only to the session id and prompt id it was requested for.
- **ORC-10/AC-3**: While a session is in Observe mode, the pipeline shall not call the decider.
- **ORC-10/AC-4**: While a session is in Observe or Recommend mode, its pipeline shall hold no write-capable handle, as shown by its constructor signature and has_write_path.
- **ORC-10/AC-5**: When the e2e test runs Observe and Recommend sessions, each sharer shall receive no input, command, resize, agent-prompt or control-action message from the orchestrator.
- **ORC-10/AC-6**: When a recommendation or escalation is produced, the pipeline shall write one journal row with session, prompt, generation, mode, evidence references, provider and outcome.
- **ORC-10/AC-7**: If a decider result arrives for a prompt id or generation that is no longer current, then the pipeline shall discard it and journal it as stale.
- **ORC-10/AC-8**: When the prompt closes, its generation changes or the session ends, the pipeline shall cancel the outstanding decision for that session.
- **ORC-10/AC-9**: The pipeline shall keep at most one outstanding decision per session and at most max_concurrent_decisions decider calls in total.
- **ORC-10/AC-10**: When take_over is called for a session, the pipeline shall cancel its outstanding decision and suspend new decisions for it until resume.
- **ORC-10/AC-11**: When the decider escalates or the eligible set is empty, the pipeline shall show a pending human decision with the reason.
- **ORC-10/AC-12**: While the observer reports replay or an unresolved event gap, the pipeline shall open no decision for that session.
- **ORC-10/AC-13**: When the ORC-10.1 and ORC-10.2 pull requests open, each shall record a threat-modeling-agentic-tools review.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run -E 'test(orchestrator::decision)'` and `cargo nextest run --test orchestrator_e2e` |
| macOS | Same; the pipeline has no platform code |
| Linux | Same |

Unit tests drive `SessionPipeline` with scripted recognizer events and a scripted `Decider` (delay,
ask_user, invalid, stale). AC-1 uses a gated decider for A and asserts B finishes first. The e2e test
uses ORC-03 captures only, never hand-written menus.

## Security / license review
This item decides whether a write path exists in M1, so it touches the remote-control surface: run
`.claude/skills/threat-modeling-agentic-tools` on ORC-10.1 and ORC-10.2 (AC-4, AC-5 are the evidence).
Auth and hub roles are unchanged; ORC-04 owns the observer's position in `Hub`. Terminal text is
untrusted input to the recognizer and the decider; authority comes only from the enrollment and
ORC-08. No new crates.

## Overlaps and existing work
- Replaces warpr-tcjap (T-11).
- warp AGT-07.1 (local task queue and orchestration spike, docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/AGT-07.md): Synth Term's own
  subagents. Different layer; no shared code.
- ORC-14 adds Delegate on top of this state machine; ORC-11 renders `snapshot` and notices.

## Open questions (owner to confirm)
- Default `max_concurrent_decisions`. Recommendation: 2, matching the "two sessions" first slice (§8).
- M1 take-over: suspend until an explicit Resume (recommended) or only until the current prompt closes.
- Should Observe journal every redraw-stable prompt or only opened and closed events? Recommendation: opened and closed.

## References
- specs/decision-orchestrator/DESIGN.md §4, §7, §8 (M1 row, "Simultaneous prompts", "False prompt and redraw").
- specs/decision-orchestrator/spec.md (old AC-21, AC-22), tasks.md (T-11).
