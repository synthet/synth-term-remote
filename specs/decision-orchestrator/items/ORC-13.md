---
id: ORC-13
title: Host conditional answer gate
priority: P0
score: TBD
effort: L
depends_on: [ORC-12]
milestone: M2
status: proposed
baseline: main 58e796c (2026-11-27); warp master 0e6fd3ac
---

# [ORC-13] Host conditional answer gate

## Problem
DESIGN §4 step 6 and §5 "Host response semantics" put the final check at the host: validate the live request,
policy, option, execution identity and generation, serialize with local and remote human input, give manual input
priority, and reject stale or invalid answers without writing. The server and orchestrator cannot see the live
grid or local keystrokes, so only Synth Term can make this check race-free. The work lives in synthet/warp; this
item states the contract so ORC-14 can test against a simulated gate that follows the same rules.

## Current state (warp master 0e6fd3ac; server main 58e796c)
| Status | What | Where |
|---|---|---|
| Exists | Remote write path: checks `FeatureFlag::SharedSessionWriteToLongRunningCommands`, `is_active_and_long_running`, the participant's role, then `write_viewer_bytes_to_pty` | warp `app/src/terminal/local_tty/terminal_view_adaptor.rs` `NetworkEvent::WriteToPtyRequested` arm |
| Exists | Only two PTY emitters in the view: `write_to_pty` (`Event::WriteBytesToPty`) and `write_agent_bytes_to_pty` (`Event::WriteAgentInputToPty`) | warp `app/src/terminal/view.rs` |
| Exists | Focus reports go through `write_to_pty` from `maybe_report_focus_in`/`maybe_report_focus_out` (`EscCodes::FOCUS_IN`/`FOCUS_OUT`) | warp `app/src/terminal/view.rs` |
| Exists | `write_user_bytes_to_pty` returns `false` and writes nothing when `is_agent_in_control` | warp `app/src/terminal/view.rs` |
| Exists | `CLIAgentSessionStatus::Blocked { source: BlockedSource }` | warp `app/src/terminal/cli_agent_sessions/mod.rs` |
| Partial | Claude Code hooks are not auto-installed on Windows (`can_auto_install`), so `Blocked` may be absent | warp `app/src/terminal/cli_agent_sessions/plugin_manager/claude.rs` |
| Exists | Shared-session banners (`insert_shared_session_started_banner`, `InlineBannerType::SharedSessionStart`) | warp `app/src/terminal/view/shared_session/view_impl.rs` |
| Missing | Gate, attempt store, Claude Code key mapping | warp `app/src/terminal/shared_session/synth_host_ext/` (new) `gate.rs`, `attempts.rs`, `claude_menu.rs` |
| Missing | Simulated gate for server-repo tests | `examples/fake_sharer.rs` (no `--gate`) |

## Proposed change
Host (warp-m0pj2):
- `synth_host_ext/gate.rs` (new): on `answer` from the ORC-12 link, inside one `terminal_view.update` on the UI thread,
  check: decisions enabled and no take-over; `execution_id`/`stream_epoch` match; `host_generation` unchanged since
  `decision.opened` armed it (no non-focus PTY write since); not expired; `"{index}. {label}"` present on the live grid;
  active block long-running and not under agent control; `Blocked` when that status is available. Then record intent,
  send the key, reply `answer.result`.
- `synth_host_ext/attempts.rs` (new): attempt_id → (content digest, status). Identical reuse returns the recorded status;
  conflicting reuse is an error. Intent is recorded before dispatch; an intent with no receipt becomes `uncertain`.
- `synth_host_ext/claude_menu.rs` (new): option index → key bytes for the Claude Code menu family captured in ORC-03.
  Whether a digit selects and submits is verified from captures; multi-step navigation re-reads the selection
  before the final Enter (DESIGN §5 "Key mapping").
- Manual priority: in the `WriteToPtyRequested` arm and the local input path, while a decision is armed send
  `decision.closed{human_input}` before the bytes reach the PTY.
- `view_impl.rs`: banner while decisions are armed, with Take over (sends `take_over`, suspends until a fresh `hello`).
- Write counter: provided by warp-i9ils (ORC-12/AC-18); the gate reads it and must exclude focus reports.
Server repo (ORC-13.2): `fake_sharer --gate` (new) applies the same rules to a scripted screen and prints the key it
would write, so ORC-14 tests the orchestrator against the contract, not a mock that always accepts.

## Scope / non-goals
- In: host validation, attempt idempotency, manual priority, take-over, banner, simulated gate.
- Out: recognition and policy (ORC-06, ORC-08), the orchestrator state machine (ORC-14), native requests (M3),
  any adapter other than Claude Code menus.

## Acceptance criteria
- **ORC-13/AC-1**: If an answer's `execution_id` or `stream_epoch` does not match the live share, Synth Term shall reject it with zero bytes written.
- **ORC-13/AC-2**: If any PTY write other than a focus report or the gate's own bytes for this attempt has happened since the decision was armed, Synth Term shall reject the answer with zero bytes written.
- **ORC-13/AC-3**: If an answer arrives after its `expires_at`, Synth Term shall reject it with zero bytes written.
- **ORC-13/AC-4**: If the option's `"{index}. {label}"` text is not on the live grid, Synth Term shall reject the answer with zero bytes written.
- **ORC-13/AC-5**: If decisions are disabled by flag or take-over, Synth Term shall reject every answer with zero bytes written.
- **ORC-13/AC-6**: If the active block is not active and long-running, or is under agent control, Synth Term shall reject the answer with zero bytes written.
- **ORC-13/AC-7**: Where the session's `CLIAgentSessionStatus` is available, if it is not `Blocked`, Synth Term shall reject the answer with zero bytes written.
- **ORC-13/AC-8**: When an `attempt_id` is reused with identical content, Synth Term shall return the recorded status without writing.
- **ORC-13/AC-9**: If an `attempt_id` is reused with different content, Synth Term shall return an error without writing.
- **ORC-13/AC-10**: When Synth Term accepts an answer, it shall record write intent for that attempt before dispatching the key.
- **ORC-13/AC-11**: If Synth Term cannot establish whether a recorded intent reached the PTY, it shall report the attempt as `uncertain` and never re-send its key.
- **ORC-13/AC-12**: When a human types locally or through a viewer while a decision is armed, Synth Term shall send `decision.closed{human_input}` before those bytes reach the PTY.
- **ORC-13/AC-13**: When Take over is used, Synth Term shall keep automatic answers suspended until a fresh `hello`.
- **ORC-13/AC-14**: The gate shall perform all checks and the key write inside one `terminal_view.update` on the UI thread.
- **ORC-13/AC-15**: Where a menu needs navigation before Enter, the gate shall re-read the selected option and revalidate before sending the final Enter.
- **ORC-13/AC-16**: While a decision is armed, Synth Term shall show a pane banner with a Take over action.
- **ORC-13/AC-17**: The gate shall read human-input state only from the warp-i9ils write counter (ORC-12/AC-18), which excludes focus reports from `maybe_report_focus_in` and `maybe_report_focus_out`.
- **ORC-13/AC-18**: When `fake_sharer` runs with `--gate` (new), it shall accept or reject answers by the rules of ORC-13/AC-1 to ORC-13/AC-15 against its scripted screen and print each key it writes.
- **ORC-13/AC-19**: When the gate lands, its PR shall record a review with `.claude/skills/threat-modeling-agentic-tools`.

## Test plan
| Platform | How |
|---|---|
| Windows | warp-m0pj2 unit tests for `gate.rs`/`attempts.rs`/`claude_menu.rs` (new) (`cargo nextest run -p warp`, filter verify at implementation); server `cargo nextest run -E 'binary(fake_gate_e2e)'` with `fake_sharer --gate` (new) |
| macOS | Same warp unit tests in CI; real-host run deferred to ORC-15 |
| Linux | Same unit tests in CI |

Each reject AC asserts that no `Event::WriteBytesToPty`/`WriteAgentInputToPty` was emitted (pattern: `view_tests.rs`
matching `Event::WriteBytesToPty`). AC-12 asserts event order (`decision.closed` before the write event).

## Security / license review
Touches the remote-control surface (a new path that results in PTY input): needs the
`.claude/skills/threat-modeling-agentic-tools` review (ORC-13/AC-19). Key threats: TOCTOU between check and write
(AC-14), a stale prompt receiving Enter (AC-15), replay (AC-8, AC-9, AC-11), answers while a human is typing (AC-12).
`answer` carries no bytes, so the server cannot inject keystrokes through this channel (ORC-12/AC-3). No new crates expected.

## Overlaps and existing work
- Covered by warp-m0pj2 (ready, `security` label, blocked by warp-i9ils). Its ACs map to ORC-13/AC-1..AC-5, AC-8, AC-9,
  AC-12, AC-13. New here, lead to amend the ticket: AC-6, AC-7, AC-10, AC-11, AC-14..AC-17 (its body mentions AC-7,
  AC-14, AC-16 and the counter but has no AC for them). Its body still cites spec.md "H-2"; re-point to this item.
- warp-m0pj2 is p2 while ORC-13 is P0; lead to align.
- Write counter: warp-m0pj2's body puts it in `write_to_pty`/`write_agent_bytes_to_pty`, but warp-i9ils AC-3 needs it
  first; assign it to warp-i9ils (ORC-12/AC-18) and have the gate only read it (ORC-13/AC-17).
- Roadmap: docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/SEC-01.md (SEC-01.4 fail-closed local approval
  evaluator, a different layer: ACP tool permissions, not PTY answers). Cite, do not merge.

## Open questions (owner to confirm)
- Attempt store durability: recommend in-memory per share, with every unreceipted intent reported `uncertain` after a
  host restart (a new `execution_id` makes old answers stale anyway); alternative a persisted store in the Synth Term data dir.
- Grid check: recommend matching `"{index}. {label}"` within the recognized menu rows only; alternative anywhere on screen.
- Gate's own writes: recommend the gate tags its bytes per attempt (navigation keys and the final Enter) so the counter
  ignores them for that attempt only; alternative restrict v1 to menus that submit on one key and skip navigation.

- Test home for ORC-13/AC-18: recommend a new `tests/fake_gate_e2e.rs` (new) that spawns the built example, like
  `web-tests/tests/harness.js` does; alternative move the gate logic to a shared test-support module used by both
  the example and the Rust tests.

## References
- DESIGN.md §4 step 6, §5, §7 "Human answers", "Missed acknowledgment".
- warp `app/src/terminal/local_tty/terminal_view_adaptor.rs`, `app/src/terminal/view.rs`,
  `app/src/terminal/view/shared_session/view_impl.rs`, `app/src/terminal/cli_agent_sessions/mod.rs`.
