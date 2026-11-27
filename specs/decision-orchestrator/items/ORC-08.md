---
id: ORC-08
title: Deterministic policy, audit journal and pending store
priority: P0
score: 8
effort: M
depends_on: [ORC-05, ORC-06, ORC-07]
milestone: M1
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-08] Deterministic policy, audit journal and pending store

## Problem
DESIGN.md §4 step 4 (Evaluate scope) asks for project choices and tool permissions to be kept apart. The actual operation, target and duration are checked against the user's grants, and ineligible options are removed before review and judgment. An "Allow once" label without verifiable operation details does not justify an automatic grant. §7 says eligibility for a one-time grant cannot make an "always allow" option eligible. §4 step 7 and §7 require an audit record per decision with bounded metadata. §4 also says pending records are persisted before dispatch and reconciled after a worker restart. §5 says that if a crash falls between write intent and receipt, the uncertainty is kept and the write is never replayed automatically. Every later stage (ORC-10 Recommend, ORC-14 Delegate) needs one deterministic gate and one durable record of what happened.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Missing | No policy, journal or pending store; no orchestrator module | `src/lib.rs` |
| Exists (pattern) | `LogTail`: a `Mutex`-guarded `File`, `create_dir_all` on the parent, `Write` impl, tests with `temp_path` under `std::env::temp_dir()` plus a uuid. It opens with `File::create` (truncates), which suits a per-run log but not an audit journal | `src/tray_support.rs` `LogTail::create`; `src/tray_support_tests.rs` `temp_path` |
| Partial | The tray data dir is `%LOCALAPPDATA%\SynthRemoteControl` from a private `data_dir()`; the console server has no data dir. ORC-07 adds `--data-dir` | `src/tray/app.rs` `data_dir`; `src/launch.rs` `Args`; ORC-07.1 |
| Missing | No `tempfile`, `chrono` or `time` crate in Cargo.lock; `serde_json` and `uuid` are direct deps; `unix_secs` exists as a private helper | `Cargo.lock`; `Cargo.toml`; `src/hub.rs` `unix_secs` |
| Missing (inputs) | Prompt kind, option scope and `operation` (ORC-06.1); session eligibility (ORC-05.1); profile `mode`, `workspace_root`, `send_excerpts`, `decision_classes` (names owned by this item) and `policy_revision: u64` (ORC-07.1). ORC-07 defines no tool/target grant fields | ORC-05.1, ORC-06.1, ORC-07.1 |

## Proposed change
1. **Policy (ORC-08.1).** `src/orchestrator/policy.rs` (new). It is a pure function, `evaluate(prompt: &RecognizedPrompt, eligibility: &Eligibility, profile: &<ORC-07.1 profile type>, target_in_scope: Option<bool>, policy_revision: u64) -> PolicyVerdict`. `target_in_scope` is computed by the caller (ORC-10) with ORC-09.1's canonical-root resolver, so the policy does no I/O. The verdict has `outcome` (`Proceed` or `Escalate(reason)`), an eligible option list, the removed options with reasons, and the revision. The rules run in order and the first match wins:
   - session ineligible (replay, gap, unknown run state, decode failure): `Escalate(observation_not_trusted)`;
   - decision class not granted by the profile: `Escalate(class_not_delegated)`;
   - `Permission` without `operation`: `Escalate(missing_operation_details)`;
   - no grant covers the operation's tool and target: `Escalate(no_matching_grant)`;
   - remove `FreeText` options; remove every `Persistent` option unless a persistent grant explicitly covers the operation. An allow-once grant never counts (no persistent grants in this release, owner to confirm);
   - empty eligible set: `Escalate(no_eligible_option)`.
   Decision class names (ORC-07 leaves them to this item): `choice` delegates `Choice` prompts; `permission_once:<tool_name>` is an allow-once grant for that tool. It covers the operation only when `tool_input` names a path target and `target_in_scope` is `Some(true)` for the profile's `workspace_root`. Any other target is `no_matching_grant`. There is no persistent grant class (owner to confirm). No I/O, clock or randomness, so the same inputs give the same verdict. Observe mode never reaches the decider. The policy still runs so the journal can record eligibility.
2. **Audit journal (ORC-08.2).** `src/orchestrator/journal.rs` (new) `Journal::open(dir: &Path)` writes `<dir>/orchestrator/journal.jsonl`. It follows `LogTail`'s shape but opens in append mode, not `File::create`. One `serde_json` line per record, with a flush after each. If the file does not end in `\n` (torn write), the next record starts on a new line. Record fields follow DESIGN §7:
   - `ts` (unix seconds), `session_id`, execution identity when known, `prompt_id`, generation, option ids, verdict and reasons, mode, `policy_revision`;
   - evidence references and digests (filled by ORC-09), decider/provider version, selected option, attempt id and status, takeover source, short rationale.
   The record type has no field for the server token or provider keys, and none for excerpt or transcript text. ORC-07 has no retention setting, so retention stays a later, separate option (DESIGN §7). Rotation is out of scope.
3. **Pending store (ORC-08.3).** `PendingStore` in `src/orchestrator/journal.rs` (new) keeps `<dir>/orchestrator/pending.json`. Record states are `Open`, `Recommended`, `Dispatching` (write intent), `Submitted`, `Confirmed`, `Rejected`, `Cancelled`, `Escalated` and `Uncertain`. Terminal states are journaled, then dropped.
   - Each save writes `pending.json.tmp` in the same directory, calls `sync_all`, then `std::fs::rename` over `pending.json`. The replace-existing behaviour on Windows needs checking against the std docs for the pinned toolchain, and so does failure while another process holds the file.
   - `record_intent` returns a dispatch permit only after the rename succeeds.
   - On load, `Dispatching` and `Submitted` become `Uncertain` and are journaled. `Open` and `Recommended` stay suspended until ORC-10/ORC-14 re-observe.
   - No API re-dispatches an `Uncertain` record. An unparseable file is moved aside to `pending.json.corrupt-<ts>` and every session starts uncertain (owner to confirm).

## Scope / non-goals
- No decider, evidence packet or validation of the decider's answer (ORC-09/ORC-10).
- No reconciliation with the live host after restart (ORC-14 uses this store).
- No config parsing or `policy_revision` computation (ORC-07).
- No journal viewer UI (ORC-11), no rotation, no new crates.

## Acceptance criteria
- **ORC-08/AC-1**: While the matching grant for a permission prompt is allow-once, the policy shall exclude every `Persistent` option from the eligible set.
- **ORC-08/AC-2**: If a `Permission` prompt has no `operation`, then the policy shall return `Escalate(missing_operation_details)`.
- **ORC-08/AC-3**: If no grant in the enrollment profile covers the operation's tool and target, then the policy shall return `Escalate(no_matching_grant)`.
- **ORC-08/AC-4**: While the session's eligibility is ineligible, the policy shall return `Escalate(observation_not_trusted)`.
- **ORC-08/AC-5**: If the profile does not delegate the prompt's decision class, then the policy shall return `Escalate(class_not_delegated)`.
- **ORC-08/AC-6**: When filtering leaves no eligible option, the policy shall return `Escalate(no_eligible_option)`.
- **ORC-08/AC-7**: The policy shall exclude every `FreeText` option from the eligible set.
- **ORC-08/AC-8**: When the same prompt, eligibility, profile and revision are evaluated twice, the policy shall return identical verdicts that carry that `policy_revision`.
- **ORC-08/AC-9**: When a verdict, recommendation, escalation, takeover or attempt status is recorded, the journal shall append exactly one JSON line to `orchestrator/journal.jsonl` under the data dir.
- **ORC-08/AC-10**: When the journal is reopened after a restart, the journal shall keep every earlier line.
- **ORC-08/AC-11**: If the journal file does not end with a newline when opened, then the journal shall start the next record on a new line.
- **ORC-08/AC-12**: The journal shall record evidence references and digests without excerpt or transcript text.
- **ORC-08/AC-13**: When a test runs with a sentinel server token and sentinel provider key, the journal file shall contain neither sentinel.
- **ORC-08/AC-14**: When a test runs with a sentinel server token and sentinel provider key, `pending.json` shall contain neither sentinel.
- **ORC-08/AC-15**: When the pending store saves, the store shall write `pending.json.tmp` in the same directory, sync it, and rename it over `pending.json`.
- **ORC-08/AC-16**: If saving the pending store fails, then `record_intent` shall return an error and no dispatch permit.
- **ORC-08/AC-17**: When the store loads a record in `Dispatching` or `Submitted` at startup, the store shall mark it `Uncertain` and journal the change.
- **ORC-08/AC-18**: The pending store shall offer no operation that dispatches or re-dispatches an `Uncertain` record.
- **ORC-08/AC-19**: If `pending.json` cannot be parsed at startup, then the store shall move it aside and report every session as uncertain.
- **ORC-08/AC-20**: If a `permission_once:<tool_name>` grant matches the tool but `target_in_scope` is not `Some(true)`, then the policy shall return `Escalate(no_matching_grant)`.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run --no-fail-fast orchestrator::policy orchestrator::journal`; table tests in `src/orchestrator/policy_tests.rs` (new) with synthetic `RecognizedPrompt` values (the policy input is a typed value, not a menu). `src/orchestrator/journal_tests.rs` (new) uses `temp_path` as in `src/tray_support_tests.rs`; AC-17 seeds a `pending.json` holding a `Dispatching` record; AC-16 points the store at a directory whose `orchestrator` path is a file; DoD `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` |
| macOS | Same nextest run (POSIX rename semantics) |
| Linux | Same nextest run |

## Security / license review
- Does not touch `src/auth.rs`, hub roles or the HTTP/WebSocket surface. The policy is the deterministic gate in front of automatic terminal input (DESIGN §4, §7), and the journal is the audit trail. A `.claude/skills/threat-modeling-agentic-tools` review is recommended for ORC-08.1 (owner to confirm). Review points: fail-closed defaults, no path from agent-controlled text to a grant, and no secrets in either file.
- `journal.jsonl` and `pending.json` sit beside the tray's `token` file in the data dir. They inherit the data dir's ACL (the per-user %LOCALAPPDATA% folder for the tray); no extra protection is added.
- No new crates.

## Overlaps and existing work
- Replaces warpr-6mylk (T-9).
- warp SEC-01.4 (docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/SEC-01.md, fail-closed local approval evaluator beside `acp/policy.rs`) is a different layer: native ACP permission requests inside the client. It shares the principles: unknown means escalate, and a grant is never promoted to always-allow. No code is shared, and this item does not plan SEC-01.4 work. SEC-01.3 (workspace trust store) is likewise client-side.
- Consumers: ORC-10 (Recommend), ORC-11 (tray rows read the journal), ORC-14 (recovery uses `PendingStore` and `Uncertain`). Evidence digests come from ORC-09.

## Open questions (owner to confirm)
- Persistent grants: recommendation is none in this release, so `Persistent` options are never auto-eligible. The alternative is an explicit persistent grant kind in ORC-07's schema.
- Corrupt `pending.json`: recommendation is to move it aside and start every session uncertain. The alternative is to refuse to start the orchestrator until an operator acts.
- Journal durability: recommendation is a flush per line. The alternative is `sync_data` per line, at a latency cost.

## References
- DESIGN.md §4 (Evaluate scope; Confirm and journal), §5 (Host response semantics), §7 (Permission scope; audit record), §8 (Permission scope; Reconnect and crash).
- Previous spec ACs AC-17, AC-18 in `specs/decision-orchestrator/spec.md`; pattern `src/tray_support.rs` `LogTail`.
