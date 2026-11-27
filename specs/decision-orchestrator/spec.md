# Headless decision orchestrator: spec

Status: proposed. Baseline: synthet/synth-term-remote \`main\` 58e796c (2026-11-27) and synthet/warp \`master\` 0e6fd3ac. Design source: [DESIGN.md](DESIGN.md) (proposal v0.1, owner's document). Index: [README.md](README.md). Tasks: [tasks.md](tasks.md).

## Direction
A developer runs several coding agents in Synth Term terminals shared to this server. The orchestrator watches each enrolled session, recognizes one pending question at a time, gathers bounded evidence, asks Jev for a choice, and then either recommends it (M1) or applies it through a conditional answer that the Synth Term host validates (M2). It is **headless**: it runs in-process in the Windows tray app and the console server, attaches to \`Hub\` as its own passive participant, and needs no browser. Enrollment comes from \`orchestrator.toml\` plus tray controls. Results go to tray balloons, tray window rows and an audit journal.

Owner decisions (2026-11-27): M0-M2 in scope; in-process placement; config plus tray UX; Claude Code menus first; Jev only for enrollments that opt in.

## Baseline (at 58e796c unless noted)
| Area | Exists | Gap |
|---|---|---|
| Session discovery and replay | \`Hub::join_session\` sends \`JoinedSuccessfully\` then \`Session::replay_events\`; \`GET /api/sessions\` via \`Hub::list_sessions\` | No passive attach: every joiner is a viewer with \`config.viewer_role\` (default \`Role::Full\`), shows in \`participant_list()\`, and its \`ReportTerminalSize\` reaches the sharer |
| Run state | \`command_running\` flips on \`CommandExecutionStarted\`/\`Finished\` in \`Hub::handle_sharer_message\` | Starts \`false\` for a mid-command share, no \`unknown\`; \`web/app.js\` \`onJoined\` hard-codes \`running = false\` |
| Resume | \`Hub::resume_session\` checks secret and reconnect token | synthet/warp \`sharer/network.rs\` reconnects with \`session_secret: Default::default()\`, so resume fails with \`WrongPassword\` and the terminal re-shares under a new id |
| Input | Viewer \`WriteToPty\`/\`ExecuteCommand\` relayed when \`role.can_execute()\`; warp \`terminal_view_adaptor.rs\` \`WriteToPtyRequested\` arm writes via \`write_viewer_bytes_to_pty\` | No conditional answer, generation or attempt id; real browser-to-terminal input unverified (2026-11-05 report) |
| Protocol | \`session-sharing-protocol\` pinned to a warpdotdev rev shared with synthet/warp | Cannot carry decision messages; needs a separate Synth extension channel |
| Prompt signals | warp \`CLIAgentSessionStatus::Blocked{PermissionRequest, QuestionAsked, NeedsInput}\` fed by OSC 777 \`warp://cli-agent\` (\`tool_name\`, \`tool_input\`) | No options in the signal; Claude Code hooks not auto-installed on Windows (\`can_auto_install\` is \`!cfg!(windows)\`) |
| PTY decoding | \`examples/fake_sharer.rs\` frames with \`lz4_flex::block::compress_prepend_size\` | \`lz4_flex\` is dev-only; no VT emulator in the crate |
| Tray | \`src/tray/app.rs\` window, tray menu, \`LogTail\`; \`launch.rs\` shared startup | No orchestrator, no session list, no notifications beyond the log |
| Decider | Jev stdio MCP server (\`jev-mcp mcp\`, \`jev_choice\`) | No Rust client; no stub |
| Tests | \`hub_tests.rs\`, \`tests/websocket_e2e.rs\`, \`examples/fake_sharer.rs\`, Playwright \`web-tests/\` | Playwright version is a placeholder (\`PLAYWRIGHT_VERSION\`) |

## Backlog
| ID | Item | Milestone | Priority | Effort | Depends on | Baseline | Covered by / replaces |
|---|---|---|---|---|---|---|---|
| [ORC-01](items/ORC-01.md) | Authoritative run state for shared sessions (running/idle/unknown) | M0 | P0 | M | - | Partial | warpr-me1nx (T-2) |
| [ORC-02](items/ORC-02.md) | Resume accepts the client's empty secret when the reconnect token matches | M0 | P0 | S | ORC-04 | Conflict | warpr-jmobk (T-16) |
| [ORC-03](items/ORC-03.md) | Real-transport verification and Claude Code menu captures | M0 | P0 | M | ORC-01 | Missing | warpr-pdz0k (T-15); overlaps warp INT-02.1 |
| [ORC-04](items/ORC-04.md) | Passive observer attach in Hub | M0 | P0 | M | ORC-01 | Missing | warpr-sczw3 (T-3) |
| [ORC-05](items/ORC-05.md) | Orchestrator scaffold, stream observer, VT screen and OSC 777 scanner | M1 | P0 | L | ORC-01, ORC-03, ORC-04 | Missing | warpr-vh8nh (T-1), warpr-66a37 (T-7) |
| [ORC-06](items/ORC-06.md) | Claude Code menu recognizer and prompt identity | M1 | P0 | M | ORC-03, ORC-05 | Missing | warpr-xli7c (T-8) |
| [ORC-07](items/ORC-07.md) | orchestrator.toml, enrollment and startup wiring | M1 | P1 | M | ORC-05 | Missing | warpr-slzc1 (T-5) |
| [ORC-08](items/ORC-08.md) | Deterministic policy, audit journal and pending store | M1 | P0 | M | ORC-05, ORC-06, ORC-07 | Missing | warpr-6mylk (T-9); related warp SEC-01.4 |
| [ORC-09](items/ORC-09.md) | Evidence packet and Jev decider | M1 | P1 | M | ORC-05, ORC-07 | Missing | warpr-daijn (T-10) |
| [ORC-10](items/ORC-10.md) | Decision pipeline: Observe and Recommend | M1 | P0 | L | ORC-04, ORC-05, ORC-06, ORC-07, ORC-08, ORC-09 | Missing | warpr-tcjap (T-11) |
| [ORC-11](items/ORC-11.md) | Headless tray UX: sessions, Enroll, Take over, balloons | M1 | P1 | M | ORC-07, ORC-10 | Missing | warpr-7tsb0 (T-12) |
| [ORC-12](items/ORC-12.md) | Host extension channel (schema v1, server route, host client) | M2 | P0 | L | ORC-04 | Missing | warpr-gcfh8 (T-4), warpr-bzxpe (T-6); host part covered by warp-i9ils |
| [ORC-13](items/ORC-13.md) | Host conditional answer gate | M2 | P0 | L | ORC-12 | Missing | covered by warp-m0pj2 |
| [ORC-14](items/ORC-14.md) | Delegate mode with take-over and recovery | M2 | P0 | L | ORC-02, ORC-08, ORC-09, ORC-10, ORC-12, ORC-13 | Missing | warpr-g7d60 (T-13) |
| [ORC-15](items/ORC-15.md) | Real-host positive control and release gate | M2 | P1 | M | ORC-03, ORC-07, ORC-11, ORC-13, ORC-14 | Missing | warpr-0jznw (T-14) |

## Corrections the code forced
These change the design proposal or the first plan. Each is backed by the item that cites the evidence.
1. **Resume is broken today.** synthet/warp `sharer/network.rs` reconnects with `session_secret: Default::default()` and `Hub::resume_session` rejects it, so every dropped socket re-shares under a new id and would drop an enrollment (ORC-02).
2. **`did_execute` alone does not mean running.** It is also true for static blocks, so the rule needs a non-null `start_ts` and a null `completed_ts`. The share's UDI `long_running_command_agent_interaction` is a second candidate signal, to be confirmed by real captures (ORC-01, ORC-03).
3. **Real idle shares are unverified.** `fake_sharer` always sends one block shape. If a real idle share derives as `unknown`, the web viewer keeps today's idle routing for `unknown` (ORC-01).
4. **Web tests cannot run.** `web-tests/package.json` pins `@playwright/test` to the placeholder `PLAYWRIGHT_VERSION`, so `npm install` fails; ORC-01.2 pins it. `onJoined` should derive state from `body.scrollback`, because fetching `/api/sessions` races the replay (owner to confirm).
5. **`ReconnectToken` has no byte accessor** at protocol rev f86afe1 (`Display`/`Debug` print `***`). The extension route compares its serde string with `auth::constant_time_eq`, and the host must not build the header with `to_string()` (ORC-12).
6. **More paths reach the sharer than `WriteToPty`.** `ExecuteCommand`, `SendAgentPrompt` and `SendControlAction` do too, so the remote-input notice that invalidates a pending decision covers all four. Sessions end on three paths (`EndSession`, the size limit, `reap_abandoned`), and each closes the extension link (ORC-12, ORC-14).
7. **The write counter belongs to the host client** (warp-i9ils), not the gate: `host_generation` needs it first. Focus reports go through `write_to_pty` from `maybe_report_focus_in`/`out` and are excluded (ORC-12, ORC-13).
8. **On Windows, permission prompts will escalate.** Claude Code hooks are not auto-installed on Windows (`can_auto_install` is `!cfg!(windows)`), so OSC `tool_input` is absent and ORC-08 escalates every permission prompt without operation details. Delegate then effectively covers question menus only (ORC-08).
9. **The Jev child would inherit the remote-control token.** The server reads `SYNTH_RC_TOKEN`/`SYNTH_RC_TOKEN_FILE` from its environment, so the decider child removes both (`env_remove`). `jev-mcp` also loads its own `.env` and `JEV_ENV_FILE`, so "key from the environment only" binds this crate, not Jev. `ask_user` is a criteria key the caller adds; Jev confidence may be null (ORC-09).
10. **Startup needs a handle.** `launch::serve` builds the `Hub` privately and `Args` fields are private, so ORC-07.3 changes `serve` to return an orchestrator handle. `toml` is only a build-dependency transitively, so it becomes a direct dependency. `CREATE_NO_WINDOW` needs winapi's `winbase` feature (or tokio's `creation_flags`). nwg `ListView` needs the `list-view` feature; balloons already work through `TrayNotification::show` (ORC-07, ORC-09, ORC-11).
11. **Untrusted sizes.** `lz4_flex` `decompress` allocates from the 4-byte size prefix, so the observer caps it before decoding. OSC 777 is `ESC ] … BEL` and its body can contain `;`; PTY bytes keep the `?2026` synchronized-output framing (ORC-05).
12. **Test selection.** `cargo nextest run websocket_e2e` selects no tests from an integration binary; tasks use `-E 'binary(…)'`. The first plan's commands had the same mistake (ORC-12, ORC-14).
13. **No Claude Code version on the wire.** The OSC notification carries `plugin_version` only, so where the release gate reads the adapter version is open (ORC-15).

## Architecture
- **Passive attach (ORC-04).** \`Hub::attach_observer\` / \`detach_observer\` / \`resync_observer\`. Observers live in a separate \`Session.observers\` map, receive the same downstream stream as viewers, have no upstream path, and never appear in \`participant_list()\` or \`viewer_count\`, which keeps Synth Term's single-uid viewer-driven sizing working. The dashboard gets \`observed: bool\`.
- **Core (ORC-05, ORC-06, ORC-08, ORC-10).** \`src/orchestrator/\` (new) consumes \`viewer::DownstreamMessage\`: LZ4 decode, event numbering with replay suppression and gap resync, a VT screen, an OSC 777 scanner, the Claude Code recognizer with prompt identity across redraws, deterministic policy, and a per-session decision state machine. One pending automatic write per session; a bounded pool of decider jobs across sessions.
- **Evidence and decider (ORC-09).** Bounded packet (goal, question, eligible options, recent screen, authorized excerpts with digests). \`Decider\` trait with a Jev stdio MCP client and a contract stub. \`ask_user\`, invalid option, timeout or error escalate.
- **Host extension channel (ORC-12, ORC-13).** WebSocket \`/synth/v1/sessions/{id}/host\`, JSON \`{"v":1,"type":…}\`: \`hello\`, \`state\`, \`observe.refresh\`, \`decision.opened\`, \`answer\`, \`answer.result\`, \`decision.closed\`, \`take_over\`. Auth is \`check_ws_auth\` plus \`x-synth-reconnect-token\` compared with \`auth::constant_time_eq\`. \`answer\` carries option id, index and label but no bytes; the host owns key encoding, \`host_generation\` and the final gate. A host without the extension never connects, so delegation fails closed.
- **Config, startup and tray (ORC-07, ORC-11).** \`launch.rs\` gains the orchestrator config and data dir; the tray passes \`data_dir()\`. UI logic lives in a testable \`orchestrator/ui.rs\`; \`src/tray/app.rs\` renders it.
- **Persistence (ORC-08).** \`<data_dir>/orchestrator/journal-YYYYMMDD.jsonl\` and an atomic \`pending.json\`. Write intent before dispatch; after restart, dispatched-but-unconfirmed attempts become \`uncertain\` and are never replayed.

## Milestones and exit gates
| Milestone | Items | Exit gate |
|---|---|---|
| M0 Real transport | ORC-01, ORC-02, ORC-03, ORC-04 | Live state is authoritative or \`unknown\`; resume keeps the session id; a passive observer receives the stream without being visible or able to write; real menus captured on Windows and real input verified |
| M1 Recommend | ORC-05 to ORC-11 | Two concurrent sessions get correctly attributed recommendations; the orchestrator has no write path in Observe or Recommend |
| M2 Delegate | ORC-12 to ORC-15 | All doc §8 race, duplicate, isolation and recovery cases pass against the simulated host, then the real-host positive control passes per adapter version |

## Spec-level acceptance criteria
- **SPEC/AC-1**: While an enrollment is in Observe or Recommend mode, the orchestrator shall hold no capability that can send terminal input.
- **SPEC/AC-2**: If run state, event continuity or host identity is unknown, then the orchestrator shall not send an automatic answer for that session.
- **SPEC/AC-3**: When a human acts on a session from the desktop, the phone or the tray, the orchestrator shall cancel its outstanding automatic decision for that session before any automatic key is sent.
- **SPEC/AC-4**: The orchestrator shall never send the remote-control token, credentials or an unrelated session's content to a decider.
- **SPEC/AC-5**: When \`orchestrator.toml\` is absent, the server and tray shall behave exactly as at the baseline commit.

## Verification matrix
| Check | Command | Covers |
|---|---|---|
| Format, lint, unit and integration tests | \`cargo fmt --check\`; \`cargo clippy --all-targets -- -D warnings\`; \`cargo nextest run --no-fail-fast\` | every task (DoD) |
| Browser regressions | \`cargo build --bins --examples\` then \`npm test\` in \`web-tests/\` (after the Playwright pin) | ORC-01 |
| Orchestrator end to end | \`cargo nextest run -E 'test(orchestrator_e2e)'\` | ORC-10 |
| Delegation races | \`cargo nextest run -E 'test(orchestrator_delegate_e2e)'\` | ORC-14 |
| Host gate (synthet/warp) | \`cargo nextest run -p warp -E 'test(synth_host_ext)'\` | ORC-13 |
| Real host, Windows | manual runs recorded in \`docs/reports/\` | ORC-03, ORC-15 |
| Spec consistency | \`python -I spec_checks.py coverage|graph|paths specs/decision-orchestrator\` | this spec |

## Open decisions (owner to confirm)
1. **`onJoined` state source**: derive from `body.scrollback` (recommended) rather than `/api/sessions` (ORC-01).
2. **Resume rule**: accept `SessionSecret::default()` only with a matching reconnect token, server-side (recommended), versus changing the client to store the secret, which costs a full Synth Term rebuild (ORC-02).
3. **Emulator crate**: `vt100` unless the ORC-05.2 spike shows a gap; `alacritty_terminal` is the alternative. Verify API and license at the pinned version.
4. **Tray UI model placement**: `src/tray_support.rs` per AGENTS.md (recommended) or `src/orchestrator/ui.rs` (new) as first planned (ORC-11).
5. **Adapter version source for the release gate**: OSC `plugin_version` plus a recognizer layout version (recommended) versus parsing `claude --version` on the host (ORC-15).
6. **macOS**: DESIGN.md M0 names Windows and macOS; this spec defers macOS positive control because the tray is Windows-only. The console server still runs there.
7. **Windows permission prompts**: accept that Delegate covers question menus only until Claude Code hooks run on Windows (recommended), or add a hook installer as a separate item.
8. **Scores**: item `score` fields are placeholders (the source has no scoring rubric).

## Risks
- Claude Code menu layout and key behaviour (does a digit select and submit?) change between versions; only real captures settle them (ORC-03, ORC-06, ORC-15).
- A mid-command attach can show a partial frame until the next full redraw; sessions stay ineligible until the frame is stable for a quiet period (ORC-05).
- Residual races in a third-party TUI: the host gate narrows them but a PTY has no exactly-once write across a crash; uncertain attempts are never replayed (ORC-13, ORC-14).
- Cross-repo sequencing: warp-htojy also edits `self_hosted.rs`; the warp-i9ils and warp-m0pj2 bodies predate this re-key and need updating (see README).
- The Jev provider is a paid external service; excerpts leave the machine only for enrollments that opt in.
