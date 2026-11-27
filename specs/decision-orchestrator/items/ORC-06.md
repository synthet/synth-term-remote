---
id: ORC-06
title: Claude Code menu recognizer and prompt identity
priority: P0
score: 9
effort: M
depends_on: [ORC-03, ORC-05]
milestone: M1
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-06] Claude Code menu recognizer and prompt identity

## Problem
DESIGN.md §4 step 3 (Recognize) requires a terminal adapter to establish an *active supported menu* from its layout, screen mode, foreground program and input behaviour, to keep one prompt identity across cosmetic redraws, and to open a new generation when the question, an option or the input target materially changes. §3 makes the adapter own recognition and declare its supported versions. §8 "False prompt and redraw" requires that a log printing "1/2/3" or an old transcript containing "yes/no" never produces a decision, and that redraws produce at most one pending decision. The owner chose Claude Code CLI menus as the first adapter.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Missing | No recognizer, adapter trait or prompt tracker in synth-term-remote | `src/` (no orchestrator module) |
| Missing | No Claude Code menu captures; ORC-03 produces them under `tests/fixtures/claude_code/` (new, ORC-03) | ORC-03 |
| Missing | Screen and OSC input types come from ORC-05 (`ScreenSnapshot` with `replay`/`stable`, `OscSignal`, `Eligibility`) and run state from ORC-01 (`hub::RunState`) | ORC-05.1, ORC-05.4, ORC-01.1 |
| Exists | The OSC 777 `warp://cli-agent` body carries `event` (`permission_request`, `question_asked`, `permission_replied`, ...), `tool_name` and `tool_input` | warp `app/src/terminal/cli_agent_sessions/event/v1.rs` `parse`; warp `crates/warp_core/src/cli_agent_protocol.rs` `CLIAgentNotification` |
| Exists (client) | The client derives `CLIAgentSessionStatus::Blocked { source: BlockedSource::{PermissionRequest, QuestionAsked, NeedsInput} }` from those events | warp `app/src/terminal/cli_agent_sessions/mod.rs` `BlockedSource` |
| Partial | On Windows the Claude Code hooks are not auto-installed (`can_auto_install` returns `!cfg!(windows)`), so the OSC signal, and with it `tool_input`, may be absent | warp `app/src/terminal/cli_agent_sessions/plugin_manager/claude.rs` |
| Missing | No stable digest crate (`sha2` absent from Cargo.lock); `uuid` v4 is available | `Cargo.lock`; `Cargo.toml` `uuid` |

## Proposed change
1. **Adapter contract and tracker (ORC-06.1).** `src/orchestrator/recognize/mod.rs` (new):
   - `trait Recognizer { fn adapter_id(&self) -> &str; fn covered_versions(&self) -> &[String]; fn recognize(&self, snapshot: &ScreenSnapshot, osc: &[OscSignal]) -> Option<RecognizedPrompt>; }`.
   - `RecognizedPrompt` holds the kind (`Permission` or `Choice`), question text, ordered `PromptOption { ordinal, label, scope }` with `scope` one of `Once`, `Persistent`, `Reject`, `Plain` or `FreeText`, the highlighted ordinal if the capture shows one, and `operation: Option<OperationDetails { tool_name, tool_input }>`.
   - `PromptTracker` (one per session) turns successive recognitions into `Opened { prompt_id, generation }`, `Updated` (generation + 1) and `Closed`. `prompt_id` is a uuid v4 minted by the orchestrator, so two sessions with identical menus never share one.
   - The material fingerprint is the normalized question text, option labels and order, kind, and `operation`, so a changed OSC `tool_input` opens a new generation. The highlighted ordinal, cursor, colours and whitespace are cosmetic. Compare normalized strings in memory: no new crate.
   - This `prompt_id` lives on the orchestrator side. The host-bound prompt id and generation belong to ORC-12 and ORC-13.
2. **Claude Code recognizer (ORC-06.2).** `src/orchestrator/recognize/claude_code.rs` (new). Every layout rule comes from the ORC-03 captures and is written down in the PR: frame, option numbering, highlight marker, position relative to the cursor and screen bottom, and screen mode. Nothing comes from memory.
   - It recognizes only on a snapshot that is `stable` and not `replay`, while `hub::RunState` is not `Idle`. Eligibility does not suppress recognition: it travels with the prompt to the policy, which escalates (ORC-08/AC-4), so Observe and Recommend still show the question and a transient ineligibility opens no new `prompt_id`.
   - A `permission_request` or `question_asked` OSC signal seen since the last `Closed` corroborates the prompt kind and supplies `operation` from `tool_name`/`tool_input`. Screen text alone never fills `operation`.
   - Option `scope` classification ("don't ask again" family is `Persistent`) comes from the capture sidecar for each covered version. ORC-03's sidecar (ORC-03/AC-5) records the version, mode, menu kind and chosen option today. This item needs it to also list the option labels in screen order with their scope, a requested addition.
   - Fixture tests replay captures through ORC-05's observer and screen. They also replay negative captures: command output that prints "1/2/3", and an old transcript that contains yes/no. ORC-03/AC-4 lists only the four positive fixtures, so these two are a requested addition to ORC-03.2. They are recorded with `examples/record_session.rs` (new, ORC-03.1) from a real share.

## Scope / non-goals
- No eligibility decisions (ORC-08) and no key encoding: the host-side adapter owns keys (DESIGN §5, ORC-13).
- No generic yes/no candidate detection for human inspection in this release.
- No other CLI families or native requests (M3).
- Hand-written menu fixtures are not allowed; synthetic `RecognizedPrompt` values are allowed only in tracker tests (ORC-06.1).

## Acceptance criteria
- **ORC-06/AC-1**: When an ORC-03 Claude Code permission-menu capture is replayed, the recognizer shall report a `Permission` prompt whose option labels and order match the capture sidecar.
- **ORC-06/AC-2**: When an ORC-03 AskUserQuestion capture is replayed, the recognizer shall report a `Choice` prompt whose option labels and order match the capture sidecar.
- **ORC-06/AC-3**: When an ORC-03 negative capture whose command output prints "1/2/3" is replayed, the recognizer shall report no prompt.
- **ORC-06/AC-4**: When an ORC-03 negative capture whose earlier transcript contains yes/no text is replayed, the recognizer shall report no prompt.
- **ORC-06/AC-5**: While a snapshot is marked replay or not stable, or the run state is `Idle`, the recognizer shall report no prompt.
- **ORC-06/AC-6**: When a recognized menu is redrawn 20 times with only cosmetic changes, the tracker shall keep one `prompt_id` and one generation.
- **ORC-06/AC-7**: When an option label, the option order, the question text, the prompt kind or the `operation` changes, the tracker shall increment the generation and keep the `prompt_id`.
- **ORC-06/AC-8**: When a recognized menu is no longer recognized on a stable live snapshot, the tracker shall emit `Closed` for its `prompt_id`.
- **ORC-06/AC-9**: When a menu is recognized after its predecessor closed, the tracker shall mint a new `prompt_id`.
- **ORC-06/AC-10**: When two sessions show identical menus, the trackers shall assign them different `prompt_id` values.
- **ORC-06/AC-11**: When a `permission_request` OSC signal with `tool_name` and `tool_input` arrived since the last `Closed`, the recognizer shall attach them to the permission prompt as `operation`.
- **ORC-06/AC-12**: If no corroborating OSC signal arrived for a permission prompt, then the recognizer shall report the prompt with `operation` empty.
- **ORC-06/AC-13**: When a capture's sidecar marks an option as persistent, the recognizer shall classify that option's scope as `Persistent`.
- **ORC-06/AC-14**: The recognizer shall report its `adapter_id` and the Claude Code versions covered by its fixtures with every recognized prompt.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run --no-fail-fast recognize`; tracker tests in `src/orchestrator/recognize/mod_tests.rs` (new); capture tests in `src/orchestrator/recognize/claude_code_tests.rs` (new) over `tests/fixtures/claude_code/` (new, ORC-03), positive and negative, share-before-launch and attach-mid-command; DoD `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` |
| macOS | Same nextest run on the same captures (captures are recorded on Windows; a macOS capture is deferred with macOS positive control) |
| Linux | Same nextest run |

## Security / license review
- Does not touch auth, hub roles or the remote-control surface. Terminal text is agent-controlled, so an agent can draw a fake menu. Mitigations: stable live snapshots only, capture-derived layout rules, and `operation` taken only from the OSC signal, never from screen text. Because the recognizer's output feeds automatic input in ORC-14, a `.claude/skills/threat-modeling-agentic-tools` pass together with ORC-08 is recommended (owner to confirm).
- No new crates.

## Overlaps and existing work
- Replaces warpr-xli7c (T-8).
- Inputs come from ORC-05 (screen, OSC, eligibility; ORC-05.4) and ORC-03.2 (captures; ORC-03 owns their format, path and sidecar). This item needs two additions there: sidecar `options` with scope, and the two negative captures.
- Host-bound prompt identity and key mapping: ORC-12, ORC-13 (covered by warp-m0pj2).
- The client's own blocked-status detection (warp `cli_agent_sessions`) is a reference, not a dependency.

## Open questions (owner to confirm)
- Operation details source: the recommendation is OSC `tool_input` only, so on Windows without hooks every permission prompt escalates (ORC-08/AC-2). The alternative is screen-parsed operation text matched against the OSC body when present, which weakens the fake-menu defence.
- Live Claude Code version: the recommendation is a field in the enrollment profile (ORC-07), checked against `covered_versions` by ORC-15's delegation gate. The alternative is screen-banner detection, if the captures show a stable one.
- Persistent-scope labels per version: taken from the capture sidecar. A new Claude Code version needs a new capture.

## References
- DESIGN.md §3 (Adapter), §4 (Recognize), §7 (Prompt changes), §8 (False prompt and redraw).
- Previous spec ACs AC-15, AC-16 in `specs/decision-orchestrator/spec.md`.
