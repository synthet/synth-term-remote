---
id: ORC-15
title: Real-host positive control and release gate
priority: P1
score: TBD
effort: M
depends_on: [ORC-03, ORC-07, ORC-11, ORC-13, ORC-14]
milestone: M2
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-15] Real-host positive control and release gate

## Problem
DESIGN §8 "Positive control": for a real known menu, choose the intended option, capture its specific acknowledgment
or expected next state, link it to the attempt, and also test share-before-launch and attach-mid-command. The
release criterion enables delegation "per adapter and version only after its real-host positive control and
stale-input tests pass", and tracks precision, missed questions, abstentions, latency, manual overrides and
uncertain deliveries by adapter version. Simulated tests (ORC-14) cannot prove that the key Synth Term sends lands
in the real Claude Code menu.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Partial | Last real-client report: live output observed, mid-command join reported idle, browser-to-terminal input not verified | `docs/reports/remote-control-verification-2026-11-05.md` |
| Missing | Real Claude Code menu captures with version | `tests/fixtures/claude_code/` (proposed, ORC-03) |
| Missing | Version allowlist for delegation | `delegate_versions` (proposed, ORC-07) in `config/orchestrator.example.toml` (proposed, ORC-07) |
| Missing | Journal to compute release metrics from | `src/orchestrator/journal.rs` (proposed, ORC-08) |
| Missing | Host gate | warp-m0pj2 (ORC-13) |
| Conflict | No reliable Claude Code version on the wire: `CLIAgentNotification` carries `plugin_version` and protocol `v`, not the CLI version; hooks are not auto-installed on Windows | warp `crates/warp_core/src/cli_agent_protocol.rs` `CLIAgentNotification`; warp `app/src/terminal/cli_agent_sessions/plugin_manager/claude.rs` `can_auto_install` |
| Exists | Tray app hosting the orchestrator UI | `src/tray/app.rs`; UI from ORC-11 (proposed) |

## Proposed change
1. **Release gate (code).** In the delegate path (`src/orchestrator/delegate.rs` (new) from ORC-14) refuse Delegate unless the
   session's (adapter id, recognizer version, Claude Code version) is listed in `delegate_versions` (proposed, ORC-07);
   fall back to Recommend and journal the reason. The shipped example config lists nothing until a report covers it.
   Version source: see open questions.
2. **Positive control (manual, Windows).** With Synth Term built with warp-m0pj2 and `SYNTH_WARP_REMOTE_CONTROL_DECISIONS`
   (new) set, the tray running the orchestrator, and a disposable workspace:
   - permission menu and AskUserQuestion menu, each from share-before-launch and from attach-mid-command;
   - for each, the intended option is chosen and linked to its confirmation (`answer.result confirmed` plus the
     expected next screen state);
   - stale-input repeats on the real host: phone answer first, desktop answer first, Take over, prompt changed
     before submission.
3. **Report** `docs/reports/decision-orchestrator-verification-<date>.md` (new): versions and commits, per-case result, the
   journal-derived metrics table, and the version line added to `delegate_versions` in the example config.

## Scope / non-goals
- In: version gate, Windows real-host run, report, example-config update.
- Out: macOS run (open question), additional adapters, native harnesses (M3), automated real-provider evaluation
  (DESIGN §8 keeps it separate from transport tests), a metrics dashboard.

## Acceptance criteria
- **ORC-15/AC-1**: If a session's adapter id, recognizer version or Claude Code version is not in `delegate_versions`, the orchestrator shall refuse Delegate for that session and journal the reason.
- **ORC-15/AC-2**: While Delegate is refused by the version gate, the orchestrator shall continue to produce Recommend output for that session.
- **ORC-15/AC-3**: The shipped `config/orchestrator.example.toml` (proposed, ORC-07) shall list only versions covered by a passing verification report.
- **ORC-15/AC-4**: When a real Claude Code permission menu in Synth Term on Windows is answered by delegation, the orchestrator shall journal the attempt linked to its `confirmed` result and the observed next state.
- **ORC-15/AC-5**: When a real Claude Code AskUserQuestion menu in Synth Term on Windows is answered by delegation, the orchestrator shall journal the attempt linked to its `confirmed` result and the observed next state.
- **ORC-15/AC-6**: When the terminal is shared before Claude Code launches, the orchestrator shall select the intended option in both menu types.
- **ORC-15/AC-7**: When the orchestrator attaches while a Claude Code menu is already shown, the orchestrator shall stay ineligible until it has a stable fresh frame and then select the intended option.
- **ORC-15/AC-8**: When a human answers from the phone or the desktop, or uses Take over, before a delayed verdict is released on the real host, Synth Term shall write zero additional bytes to the PTY.
- **ORC-15/AC-9**: If the prompt changes on the real host between arming and submission, the gate shall reject the answer with zero bytes written.
- **ORC-15/AC-10**: The verification report shall tabulate detection precision, missed questions, abstentions, latency, manual overrides and uncertain deliveries by adapter version from the journal.
- **ORC-15/AC-11**: The verification report shall name the Claude Code version, the Synth Term commit, the server commit and the adapter version it adds to `delegate_versions`.

## Test plan
| Platform | How |
|---|---|
| Windows | AC-1..AC-3: `cargo nextest run orchestrator`. AC-4..AC-11: manual run per the report checklist, tray + Synth Term + phone on the same tailnet or LAN |
| macOS | Console server only; real-host run deferred (open question) |
| Linux | AC-1..AC-3 in CI only |

Evidence per case: journal excerpt (ids, statuses), a screenshot or capture of the next state, and the attempt id.

## Security / license review
The gate code (AC-1..AC-3) narrows the remote-control surface; review it in the ORC-14 threat-model pass. The manual
run uses a disposable workspace and a token kept in the environment, never in the report (AGENTS.md "Safety").
Journal excerpts in the report contain ids and short rationale only, no transcript text or excerpts. No new crates.

## Overlaps and existing work
- Replaces warpr-0jznw (T-14).
- Needs warp-m0pj2 (ORC-13.1) on the Synth Term build, ORC-03 captures (same Claude Code version), ORC-11 tray UI.
- Roadmap: docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/INT-02.md (INT-02.1 re-verifies mid-command input
  against synth-term-remote; share the run and cross-link the reports, do not run it twice).
- Supersedes the "not verified" input entry in `docs/reports/remote-control-verification-2026-11-05.md` only if ORC-03
  has not already done so.

## Open questions (owner to confirm)
- Claude Code version source: recommend the enrollment profile declares it and the orchestrator compares it with the
  version recorded in the ORC-03 captures; alternative Synth Term reports the foreground agent's version in `hello`
  (no such field exists today; verify feasibility in warp-i9ils).
- macOS: DESIGN §8 M0/M2 name Windows and macOS; spec.md defers the macOS tray. Recommend Windows-only for this
  release with macOS as a follow-up item; alternative run the console server on macOS for the same checklist.
- Metrics: recommend counting from the journal by hand or a short script in the report; alternative a
  `examples/journal_report.rs` (new) summarizer if the run produces too many rows to count.

## References
- DESIGN.md §8 (Positive control, Release criterion).
- `docs/reports/remote-control-verification-2026-11-05.md`; items ORC-03, ORC-07, ORC-11, ORC-13, ORC-14.
