---
id: ORC-03
title: Real-transport verification and Claude Code menu captures
priority: P0
score: tbd
effort: M
depends_on: [ORC-01]
milestone: M0
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-03] Real-transport verification and Claude Code menu captures

## Problem
DESIGN.md §8 M0 exit: "demonstrate live screen reconstruction, correct input to a running menu, and
proper handling of a mid-command join" on a real client, and §8 asks for "captured menus with known
expected choices for offline evaluation". The 2026-11-05 report left browser-to-terminal input
unverified and no real Claude Code stream has ever been recorded, so ORC-05 (screen) and ORC-06
(recognizer) would otherwise be built on hand-written menus. This item is human-run on Windows.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Partial | Live output verified on Windows; "Input from the browser is not verified"; mid-command share showed idle | `docs/reports/remote-control-verification-2026-11-05.md` (AC table row 2, "Bugs found") |
| Exists | Scripted sharer with real protocol types and LZ4 framing; pattern for a protocol client example | `examples/fake_sharer.rs` (`init_payload`, `report`, `main`) |
| Exists | Viewer join route and handshake: first message must be `viewer::UpstreamMessage::Initialize`; display name from `?client=` (`web` or else "Warp viewer") | `src/server.rs` `viewer_join`, `run_viewer` |
| Exists | A joining viewer gets `JoinedSuccessfully` (scrollback, `universal_developer_input_context`, window size) and then every stored event | `src/hub.rs` `Hub::join_session`, `Session::replay_events` |
| Exists | `viewer::DownstreamMessage::to_json` / `from_json` for lossless JSONL | session-sharing-protocol rev f86afe1 src/viewer.rs |
| Exists | Viewers all carry the owner uid, so one more present viewer keeps `single_distinct_present_viewer_uid` single | `src/hub.rs` `HubConfig::owner_uid`; synthet/warp app/src/terminal/view/shared_session/view_impl.rs (`is_viewer_driven_sizing_eligible`) |
| Exists | Claude Code hooks are not auto-installed on Windows, so the OSC 777 `warp://cli-agent` status may be missing | synthet/warp app/src/terminal/cli_agent_sessions/plugin_manager/claude.rs (`can_auto_install`), crates/warp_core/src/cli_agent_protocol.rs (`CLI_AGENT_NOTIFICATION_SENTINEL`) |
| Missing | A recorder example, `tests/fixtures/claude_code/` (new), and an input-verification report | none |

## Proposed change
- ORC-03.1: `examples/record_session.rs` (new). Joins `/sessions/join/{id}?client=recorder` as an
  ordinary viewer, with the bearer token read from `SYNTH_RC_TOKEN` (the server's own variable, `Args`
  in `src/launch.rs`; owner to confirm) and never written to output. It sends `Initialize` and
  `Ping` only: no `ReportTerminalSize`, no input, no role request. It writes each downstream message
  verbatim, one `viewer::DownstreamMessage` JSON per line, to the output file in
  `tests/fixtures/claude_code/` (new), plus a sidecar `<name>.meta.json` (new) with Claude Code version,
  Synth Term version and commit, capture mode, menu kind, OS, and the human's chosen option. Arguments
  mirror `fake_sharer` (`<host:port> <session_id> <out>`; owner to confirm). Uses the existing
  dev-dependency `tokio-tungstenite` only; it stores PTY bytes still LZ4-framed, as received.
- ORC-03.2 (human, Windows): in a disposable workspace, capture a Claude Code permission menu and an
  AskUserQuestion menu, each in two modes: share-before-launch (share, start the recorder, launch
  Claude Code) and attach-mid-command (Claude Code already at the menu, then share and record). Keep
  recording through the human's answer so the next state is in the fixture. Also capture one idle
  share (Claude Code not running, prompt waiting) to settle the ORC-01 idle-shape question. Then answer a real menu
  from the web app and record what reached it. Write
  `docs/reports/remote-control-input-verification-<date>.md` (new) and add a pointer from the
  2026-11-05 report's "not verified" row to it. The new report uses the same front matter as the
  2026-11-05 report (`type`, `title`, `description`, `resource`, `tags`, `timestamp`, `okf_version`).
- The report records, per capture: `/api/sessions` `run_state` (checks ORC-01 on a real client), the
  last block's `did_execute` / `start_ts` / `completed_ts` and the init UDI
  `long_running_command_agent_interaction` (settles the ORC-01 open question), whether the OSC 777
  sentinel appears, and which keys select and which submit.

## Scope / non-goals
- In: recorder, four fixtures with metadata, one report, a one-line pointer in the old report.
- Out: macOS capture (DESIGN §8 M0 names Windows and macOS; `spec.md` defers macOS, owner to confirm),
  phone or LAN tests, an observer route (ORC-04 is in-process only), recognizer code (ORC-06).

## Acceptance criteria
- **ORC-03/AC-1**: When `record_session` joins a session shared by `fake_sharer --running`, the recorder shall write `JoinedSuccessfully` as its first line and each later downstream message as one JSON line that `viewer::DownstreamMessage::from_json` parses.
- **ORC-03/AC-2**: While it records, the recorder shall send no upstream message other than `Initialize` and `Ping`, so the fake sharer receives no `ViewerTerminalSizeReported`, `WriteToPtyRequested` or `CommandExecutionRequested` from it.
- **ORC-03/AC-3**: The recorder shall never write the server token to its output, sidecar or log.
- **ORC-03/AC-4**: When the human capture run completes on Windows, `tests/fixtures/claude_code/` (new) shall hold permission-menu and AskUserQuestion fixtures in both share-before-launch and attach-mid-command modes plus one idle-prompt share.
- **ORC-03/AC-5**: Each fixture's sidecar shall record the Claude Code version, the Synth Term version and commit, the capture mode, the menu kind and the option the human chose.
- **ORC-03/AC-6**: When the web app sends keys to a real running Claude Code menu in each mode, the verification report shall state whether the intended option was selected, with the observed next screen.
- **ORC-03/AC-7**: The verification report shall record, per capture, the `/api/sessions` `run_state`, the last block's `did_execute`, `start_ts` and `completed_ts`, the init UDI long-running interaction, and whether the `warp://cli-agent` OSC 777 sentinel appears in the stream.
- **ORC-03/AC-8**: Before fixtures are committed, the verifier shall review them for private paths, tokens and unrelated terminal content and note the review in the report.

## Test plan
| Platform | How |
|---|---|
| Windows | ORC-03.1: `cargo build --examples`; run the server, `fake_sharer --running`, then `record_session`; check AC-1..AC-3 by reading the JSONL and the fake sharer's stdout. ORC-03.2: live Synth Term share on Windows 11 with the tray or console server, recorder runs as above, web app input from Chromium. |
| macOS | Deferred (owner to confirm); the recorder builds and runs unchanged against the console server. |
| Linux | Recorder smoke against `fake_sharer` only. |

DoD for ORC-03.1: `cargo fmt --check`; `cargo clippy --all-targets -- -D warnings`; `cargo nextest run --no-fail-fast`.

## Security / license review
- No change to auth, hub roles or server routes; the recorder is a client example holding a Full-role
  viewer seat it never uses. No threat-model review needed for ORC-03.1 unless it gains input.
- The fixtures are captured terminal content: run in a disposable workspace, review before commit
  (AC-8), and never commit tokens. The report must not quote the token.
- No new crates (`tokio-tungstenite` is already a dev-dependency in `Cargo.toml`).

## Overlaps and existing work
- Replaces warpr-pdz0k (T-15) in epic warpr-oh1rg.
- Roadmap task INT-02.1 (docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/INT-02.md) re-verifies
  self-hosted remote control including mid-command input (INT-02/AC-3). One run should serve both:
  INT-02.1 cites this report for INT-02/AC-3 and keeps its client-side ACs (AC-1, AC-2, AC-6) and doc
  drift. Not a ticket, so no "Covered by" row.
- Feeds ORC-05 (screen fixture), ORC-06 (recognizer and key mapping) and ORC-15 (positive control).
- Uses ORC-01 so mid-command input routes to the PTY in the web app.

## Open questions (owner to confirm)
- macOS capture now (DESIGN §8 M0) or deferred with the tray (recommended: defer, record the gap).
- Token source for the recorder: `SYNTH_RC_TOKEN` env (recommended) or the tray's token file.
- Add a third mode, a late recorder join to an already-shared session (cheap, tests replay).

## References
- `specs/decision-orchestrator/DESIGN.md` §2, §8; `specs/decision-orchestrator/spec.md` (old AC-28, AC-29, risks).
- `docs/reports/remote-control-verification-2026-11-05.md`; `examples/fake_sharer.rs`.
