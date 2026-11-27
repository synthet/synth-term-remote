---
id: ORC-09
title: Evidence packet and Jev decider
priority: P1
score: 7
effort: M
depends_on: [ORC-05, ORC-07]
milestone: M1
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-09] Evidence packet and Jev decider

## Problem
DESIGN.md §6 requires a bounded evidence packet (goal, question, eligible options, recent terminal
context, authorized excerpts with path, range and digest) and a decider whose answer the orchestrator
validates. Read access and permission to send an excerpt to a provider are separate settings, and
credentials stay out of the packet. §3 makes Jev the decider, picking an option or `ask_user`, with
optional confidence. §7 ("Reviewer or Jev unavailable") and §8 ("Inconclusive judgment") turn every
failure into a visible human decision with zero keystrokes. The owner deferred the reviewer step:
Jev decides directly behind a `Decider` trait, and tests use a contract stub.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Missing | Evidence builder, `Decider` trait, Jev client, contract stub | - |
| Partial | `tokio` lacks the `process` and `io-util` features a stdio child needs | `Cargo.toml` `[dependencies]` |
| Partial | `winapi` features omit `winbase`, where `CREATE_NO_WINDOW` (0x08000000) lives in winapi 0.3.9 um/winbase.rs | `Cargo.toml` `[target.'cfg(windows)'.dependencies]` |
| Exists | `tokio::process::Command` has `creation_flags` (Windows), `env_remove` and `kill_on_drop` at the locked 1.50.0 (once the `process` feature is on) | tokio 1.50.0 src/process/mod.rs |
| Exists | Remote-control token is read from `SYNTH_RC_TOKEN` / `SYNTH_RC_TOKEN_FILE`, so the server's own environment can hold it | `src/launch.rs` `Args` |
| Exists | Example-binary pattern for a scripted peer, located by path under target/debug/examples | `examples/fake_sharer.rs`; `web-tests/tests/harness.js` `sharerBin` |
| Exists (Jev) | `jev-mcp mcp` runs a stdio MCP server; `jev_choice(state, instructions, criteria, model?, question_id="answer")` | jev-mcp src/jev_mcp/mcp_server.py `jev_choice` (v0.3.0) |
| Exists (Jev) | Envelope `{ok, result: {model, usage, answers}}` or `{ok:false, error: {code, message, retryable, ...}}`; choice answer `{type, choice, confidence: number or null, probabilities}` | jev-mcp docs/api.md; src/jev_mcp/models.py `ChoiceAnswer`, `ToolEnvelope` |
| Conflict | `ask_user` is not built into Jev; it is a criteria key the caller must add (§6 example) | jev-mcp src (no `ask_user`) |
| Conflict | The child loads its own checkout `.env` and `JEV_ENV_FILE` on start (existing env wins), so "key from the environment only" binds the orchestrator, not the child | jev-mcp src/jev_mcp/dotenv_util.py `load_credential_env` |

## Proposed change
1. **Evidence** in `src/orchestrator/evidence.rs` (new), tests in `src/orchestrator/evidence_tests.rs` (new).
   - Input `DecisionInput { goal, question, options: Vec<OptionRef { id, label }>, terminal_rows }`,
     filled by ORC-10 from the recognizer (ORC-06) and the eligible set (ORC-08), so this module has
     no recognizer dependency.
   - `EvidencePacket { goal, question, options, terminal_context, excerpts, truncated }` with
     `Excerpt { rel_path, lines, sha256, text: Option<String> }`.
   - Path check: join each profile attachment to `workspace_root`, `std::fs::canonicalize` both, and
     require the target to start with the canonical root. That resolves symlinks and junctions
     before the check. Missing files and escapes are refused with a reason.
   - `text` is filled only when the profile sets `send_excerpts = true` (ORC-07.1); otherwise the
     packet holds metadata only. Byte budgets come from the profile; overflow truncates and sets
     `truncated`.
   - `redact(values)` replaces the remote-control token (passed in by ORC-10 from `AppState::token`)
     in terminal rows and excerpt text with `[redacted]`.
   - `recheck(&packet) -> Vec<StaleExcerpt>` recomputes digests, for ORC-14's pre-apply check.
2. **Decider** in `src/orchestrator/jev.rs` (new), tests in `src/orchestrator/jev_tests.rs` (new).
   - `trait Decider: Send + Sync` with a boxed-future `decide(&EvidencePacket) -> DeciderOutcome`.
     `DeciderOutcome::Choose { option_id, confidence: Option<f64>, model: Option<String> }` or
     `Escalate { reason: EscalateReason }`, where the reasons are `AskUser`, `InvalidOption`,
     `ProviderError { code }`, `Timeout` and `Protocol`. ORC-09 owns this trait; a reviewer can
     implement it later.
   - `JevDecider` builds one `jev_choice` call: `state` = goal, question, terminal context and
     evidence; `instructions` = the §6 text; `criteria` = each eligible option id with its label,
     plus `ask_user`. A recognizer option id that equals `ask_user` makes the call Escalate(Protocol).
     The answer is read at `result.answers[question_id]` and accepted only if `choice` is an offered id.
     `confidence` is recorded, never used to widen authority.
   - Transport: a minimal newline-delimited JSON-RPC client over the child's stdio (initialize, then
     `tools/call`). The MCP handshake fields, the protocol version and where the envelope sits in the
     tool result: verify at the pinned jev-mcp (`mcp>=2,<3`). Alternative: an MCP client crate
     (owner to confirm, license to verify).
   - Child: `command` and `args` from `[decider.jev]` only; inherited environment with
     `env_remove("SYNTH_RC_TOKEN")` and `env_remove("SYNTH_RC_TOKEN_FILE")`; `kill_on_drop(true)`;
     on Windows `creation_flags(CREATE_NO_WINDOW)`; stderr discarded (owner to confirm). One child is
     reused and respawned on the next call after a crash or timeout. A timeout never switches
     provider (§6).
   - Add tokio `process` and `io-util` here (ORC-05.1 adds only `lz4_flex` and `sha2`); add the
     winapi `winbase` feature or a local constant (owner to confirm).
3. **Contract stub** `examples/jev_stub.rs` (new): the same stdio subset with scripted replies.
   Proposed switches (owner to confirm): `--choice <id>`, `--ask-user`, `--invalid`, `--error <code>`,
   `--null-confidence`, `--delay-ms <n>`. For the §6 fixture it returns `read_old_save_new`. Fixtures
   live in `tests/fixtures/jev/` (new). How tests locate the built example is an open question.

## Scope / non-goals
- In: packet, path checks, redaction, digests, the `Decider` trait, Jev client, stub, child hygiene.
- Not: the reviewer step, `jev_system_one` batching, eligibility (ORC-08), journaling (ORC-10 writes
  the rows through ORC-08), live paid Jev runs in CI, workspace revision via git (deferred).

## Acceptance criteria
- **ORC-09/AC-1**: If an attachment resolves outside its workspace root after symlink or junction resolution, then the evidence builder shall refuse it with a reason.
- **ORC-09/AC-2**: While a profile does not set send_excerpts, the evidence builder shall emit excerpt path, line range and digest without text.
- **ORC-09/AC-3**: When terminal rows or excerpt text contain the remote-control token, the evidence builder shall replace each occurrence with a redaction marker.
- **ORC-09/AC-4**: When evidence exceeds the profile byte budget, the evidence builder shall truncate it and mark the packet truncated.
- **ORC-09/AC-5**: When an attached file changes after the packet was built, recheck shall report that excerpt as stale.
- **ORC-09/AC-6**: The Jev decider shall add an ask_user criterion to every jev_choice request.
- **ORC-09/AC-7**: When Jev returns ask_user, the Jev decider shall return Escalate with reason AskUser.
- **ORC-09/AC-8**: If the returned choice is not an offered option id, then the Jev decider shall return Escalate with reason InvalidOption.
- **ORC-09/AC-9**: If the envelope has ok false, the child exits, or the call exceeds timeout_secs, then the Jev decider shall return Escalate with the matching reason.
- **ORC-09/AC-10**: When the DESIGN.md §6 fixture is sent to the contract stub, the Jev decider shall return Choose with read_old_save_new.
- **ORC-09/AC-11**: If an answer has a null confidence, then the Jev decider shall accept a valid choice and record confidence as absent.
- **ORC-09/AC-12**: When the Jev child starts, the Jev decider shall take its command and args only from the decider.jev table.
- **ORC-09/AC-13**: When the Jev child starts, the Jev decider shall remove SYNTH_RC_TOKEN and SYNTH_RC_TOKEN_FILE from the child environment.
- **ORC-09/AC-14**: When a sentinel provider key is set in the environment, the orchestrator shall keep it out of orchestrator.toml, logs, journal rows and evidence packets.
- **ORC-09/AC-15**: When the Jev child starts on Windows, the Jev decider shall set the CREATE_NO_WINDOW creation flag.
- **ORC-09/AC-16**: When the ORC-09 pull requests open, each shall record a threat-modeling-agentic-tools review.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run -E 'test(orchestrator::evidence) or test(orchestrator::jev)'`; junction escape made with `cmd /C mklink /J` in a temp dir (skip with a message if it fails); manual check: tray with a real `jev-mcp mcp` shows no console window |
| macOS | Same nextest filter; escape made with `std::os::unix::fs::symlink`; the creation flag test is `cfg(windows)` |
| Linux | Same as macOS |

Stub-driven tests cover AC-6 to AC-11 and AC-13 (the stub echoes its request and selected env names
to a temp file). AC-14 uses a fake `JEV_TOKEN` sentinel. No test calls TypeSafe.

## Security / license review
This touches the remote-control surface (packet content leaves the machine; the child shares the
server's environment), so ORC-09.1 and ORC-09.2 need the `.claude/skills/threat-modeling-agentic-tools`
review. Auth and hub roles are unchanged. Risks: path escape (AC-1), the remote-control token leaking
through the environment or the screen (AC-3, AC-13), prompt-injected terminal or file text steering the
choice (only offered ids are accepted, AC-8; authority stays with enrollment and ORC-08), and the child
reading its own `.env` (documented; owner accepts or points `[decider.jev]` at a dedicated checkout).
New crates: none if the JSON-RPC client is hand-written; tokio features only. An MCP crate would need
a license check (verify at pinned version).

## Overlaps and existing work
- Replaces warpr-daijn (T-10).
- ORC-08 provides the eligible option set; ORC-10 calls the decider and journals provider and
  evidence references; ORC-14 uses `recheck`.
- warp SEC-01 (docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/SEC-01.md, SEC-01/AC-4 child env holds only named credentials): same rule in
  Synth Term; this item applies it to the Jev child.

## Open questions (owner to confirm)
- Hand-written JSON-RPC client (recommended, no new crate) or an MCP client crate.
- Child env: inherit minus `SYNTH_RC_*` (recommended) or an explicit allowlist (`PATH`, `SYSTEMROOT`, `JEV_*`, `TYPESAFE_API_KEY`).
- Should a low confidence escalate? Recommendation: no threshold in M1; record only.
- Stderr of the child: discard (recommended) or log a bounded first line.
- Locating the stub: target/debug/examples as `web-tests/tests/harness.js` does for `fake_sharer` (verify
  that `cargo nextest run` builds examples first), or an in-process `Decider` for unit tests plus one
  stdio test that builds the example explicitly (recommended).

## References
- specs/decision-orchestrator/DESIGN.md §3 (Jev row), §6, §7, §8 ("Inconclusive judgment").
- specs/decision-orchestrator/spec.md (old AC-19, AC-20, AC-31); jev-mcp README.md, docs/api.md.
