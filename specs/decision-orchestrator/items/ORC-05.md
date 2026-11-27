---
id: ORC-05
title: Orchestrator scaffold, stream observer, VT screen and OSC 777 scanner
priority: P0
score: 9
effort: L
depends_on: [ORC-01, ORC-03, ORC-04]
milestone: M1
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-05] Orchestrator scaffold, stream observer, VT screen and OSC 777 scanner

## Problem
DESIGN.md §4 step 2 (Observe) requires live state rebuilt from a snapshot plus ordered events, with detection suppressed during replay and automatic answers suspended when the initial state or an event range is missing. §2 says the watcher reuses the existing wire protocol and decodes LZ4-framed PTY output like the web viewer. Nothing in the server reconstructs a screen today: the hub only relays bytes, and every later stage (ORC-06 recognizer, ORC-08 policy, ORC-10 pipeline) needs a trustworthy screen and a per-session "eligible" flag. Without it, Synth Term's orchestrator cannot tell a live menu from old output (§8 "False prompt and redraw").

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Missing | No orchestrator module; `src/lib.rs` exports auth, hub, launch, server, tray_support, web only | `src/lib.rs` |
| Partial | `lz4_flex = "0.11"` is a dev-dependency only (0.11.6 in Cargo.lock); `examples/fake_sharer.rs` `event_for` frames output with `lz4_flex::block::compress_prepend_size` | `Cargo.toml` `[dev-dependencies]`; `examples/fake_sharer.rs` `event_for` |
| Conflict | `lz4_flex::block::decompress_size_prepended` (0.11.6) reads a 4-byte LE size and calls `Vec::with_capacity` with it before decoding, so an untrusted prefix can request a ~4 GiB allocation | `~/.cargo/registry/.../lz4_flex-0.11.6/src/block/decompress.rs` `decompress` |
| Exists | Viewer stream type: `ViewerTx = UnboundedSender<viewer::DownstreamMessage>` | `src/hub.rs` `ViewerTx` |
| Exists | Join sends `JoinedSuccessfully` with `scrollback`, `latest_event_no` and `window_size: session.initial_window_size`, then `replay_events(&viewer_id, None)` replays the full event log (Resize events included) | `src/hub.rs` `Hub::join_session`, `Session::replay_events` |
| Exists | Sharer event numbers are a contiguous counter (`EventNumber::advance`); the hub drops only `event_no <= latest` and does not check contiguity | warp `app/src/terminal/shared_session/sharer/network.rs`; `src/hub.rs` `Hub::handle_sharer_message` |
| Exists | Shared PTY bytes are the raw bytes read from the PTY; synchronized-output framing (DEC mode 2026) is left to viewers | warp `app/src/terminal/model/terminal_model.rs` (PtyBytesRead send); warp `crates/warp_terminal/src/model/ansi/mod.rs` (`\x1b[?2026h`/`l`) |
| Exists | Web viewer pattern: `onJoined` seeds from scrollback via `writeScrollbackBlock` (`stylized_command`, `stylized_output`); `onEvent` skips `event_no <= lastEventNo`; `decompressPtyBytes` | `web/app.js` |
| Exists | OSC 777 wire form `ESC ] 777;notify;warp://cli-agent;<json> BEL`; warp's parser joins `params[3..]` with `;` so the JSON body may contain `;` | warp `crates/warp_core/src/cli_agent_protocol.rs` `CLI_AGENT_NOTIFICATION_SENTINEL`, `CLIAgentNotification`; warp `crates/warp_tui/src/cli_agent_osc_event_publisher.rs` `publish`; warp `crates/warp_terminal/src/model/ansi/mod.rs` (`b"777"` arm) |
| Partial | Claude Code hooks are not auto-installed on Windows (`can_auto_install` is `!cfg!(windows)`); `is_installed` checks `windows_user_hooks_installed`, so OSC 777 appears only after a manual hook install | warp `app/src/terminal/cli_agent_sessions/plugin_manager/claude.rs` |
| Missing | No VT emulator crate in Cargo.lock (`vt100`, `alacritty_terminal`, `vte` absent) | `Cargo.lock` |
| Missing | Run state from scrollback is ORC-01's job: `pub enum RunState` and `pub fn scrollback_run_state` (proposed, ORC-01.1) beside `last_scrollback_pwd` | `src/hub.rs` `last_scrollback_pwd`; ORC-01.1 |

## Proposed change
Four PRs; the core is a pure, synchronous state machine so it can be unit tested without a Hub or socket.
1. **Scaffold (ORC-05.1).** `src/orchestrator/mod.rs` (new) and `pub mod orchestrator;` in `src/lib.rs`. Move `lz4_flex` from `[dev-dependencies]` to `[dependencies]` at the same `0.11` requirement. Shared types only: `ScreenSnapshot` (rows as text, cursor, alt-screen flag, size, `last_event_no`, `replay: bool`, `stable: bool`), `OscSignal`, `Eligibility` (eligible, or ineligible with a reason). Also add `sha2` as a direct dependency (verify at pinned version; absent from Cargo.lock), because ORC-07.1 computes `policy_revision` with it and ORC-09 digests evidence. tokio `process`/`io-util` belong to ORC-09.2. No run-state type here: ORC-05.4 imports ORC-01's.
2. **VT screen and emulator spike (ORC-05.2).** `src/orchestrator/screen.rs` (new) wraps the chosen crate behind `Screen::feed(&[u8])`, `resize`, `snapshot`. Spike: compare `vt100` and `alacritty_terminal` at the versions to be pinned (APIs, license, Windows build, alt-screen, resize, scrollback); verify at pinned version, name no API in this spec. The wrapper itself tracks `\x1b[?2026h`/`\x1b[?2026l` so a snapshot is never taken inside a synchronized-output frame, whatever the crate does. The PR records the choice and evidence.
3. **OSC 777 scanner (ORC-05.3).** `src/orchestrator/osc.rs` (new): a small byte state machine (no crate) that survives event boundaries, accepts `ESC ]` and BEL or `ESC \\` terminators, requires `777;notify;warp://cli-agent;`, keeps the rest as the body, caps the buffered length, and parses the body into a local serde mirror of the `CLIAgentNotification` fields it needs (`v`, `agent`, `event`, `session_id`, `tool_name`, `tool_input`, `summary`). warp crates are not dependencies of this repo.
4. **Stream observer (ORC-05.4).** `src/orchestrator/observer.rs` (new): `Observer::handle(viewer::DownstreamMessage) -> Vec<ObserverOutput>` plus a thin async loop over an `UnboundedReceiver`. On `JoinedSuccessfully`: reset the screen, size it from `window_size`, seed it from scrollback blocks as `web/app.js` `writeScrollbackBlock` does (owner to confirm), take run state from `hub::scrollback_run_state` (proposed, ORC-01.1) as `hub::RunState`, with no second run-state type, and record `latest_event_no` as the replay boundary. Events with `event_no <= boundary` are fed but marked replay. Check the 4-byte LZ4 prefix against a frame cap before decoding. A non-contiguous `event_no` emits `ObserverOutput::ResyncRequested { after }` (the caller invokes ORC-04's `Hub::resync_observer(session_id, observer, after)`) and makes the session ineligible until the resync replay ends. The hub stores whatever it received, so if the resync replays the same gap the session stays ineligible for its lifetime and no second resync is requested. `CommandExecutionStarted`/`Finished` update run state; `Resize` resizes; `SessionEnded` closes. A snapshot is `stable` once no live bytes arrived for a quiet period (value from ORC-03 capture timings, owner to confirm).

## Scope / non-goals
- No recognition (ORC-06), policy (ORC-08), Hub changes (ORC-04) or decisions (ORC-10).
- No write path: the observer has no sender toward the sharer.
- Not a full terminal for display; text rows only. tmux passthrough wrapping of OSC is out of scope.

## Acceptance criteria
- **ORC-05/AC-1**: When `cargo build --all-targets` and `cargo clippy --all-targets -- -D warnings` run on Windows after the scaffold lands, the build shall succeed with `lz4_flex` and `sha2` as regular dependencies and no new warnings.
- **ORC-05/AC-2**: When an ORC-03 Claude Code menu capture is replayed through the screen, the final `ScreenSnapshot` rows shall contain every option label listed in that capture's sidecar `options` field.
- **ORC-05/AC-3**: The spike PR shall record the chosen emulator crate, its pinned version and license, and test evidence for alt-screen, resize and synchronized-output handling.
- **ORC-05/AC-4**: While bytes between `\x1b[?2026h` and `\x1b[?2026l` are being fed, the screen shall return the last snapshot taken before the frame started.
- **ORC-05/AC-5**: When a `Resize` event arrives, the screen shall adopt the event's `num_rows` and `num_cols`.
- **ORC-05/AC-6**: If a `PtyBytesRead` frame declares a decompressed size above the frame cap, then the observer shall skip decoding it and mark the session ineligible.
- **ORC-05/AC-7**: If a `PtyBytesRead` frame fails LZ4 decoding, then the observer shall mark the session ineligible without panicking.
- **ORC-05/AC-8**: When an OSC 777 `warp://cli-agent` sequence is split across two `PtyBytesRead` events, the scanner shall emit exactly one signal.
- **ORC-05/AC-9**: When an OSC 777 body contains `;`, the scanner shall parse the whole remainder after the sentinel as the body.
- **ORC-05/AC-10**: If the OSC title is not `warp://cli-agent` or the body is not valid JSON, then the scanner shall emit no signal.
- **ORC-05/AC-11**: If an OSC sequence exceeds the length cap before a terminator, then the scanner shall discard it and resume scanning at the next byte.
- **ORC-05/AC-12**: While the observer processes events with `event_no` at or below the `JoinedSuccessfully` `latest_event_no`, the observer shall mark every snapshot and OSC signal as replay.
- **ORC-05/AC-13**: When an event's `event_no` is not exactly one more than the previous one, the observer shall emit a resync request after the last contiguous `event_no`.
- **ORC-05/AC-14**: While a resync is outstanding or a resync replayed the same gap, the observer shall report the session as ineligible for automatic answers.
- **ORC-05/AC-15**: While the session's `hub::RunState` is `Unknown`, the observer shall report the session as ineligible for automatic answers.
- **ORC-05/AC-16**: When `JoinedSuccessfully` carries scrollback blocks, the observer shall seed the screen from each block's `stylized_command` and `stylized_output` as `web/app.js` `writeScrollbackBlock` does.
- **ORC-05/AC-17**: When no live bytes arrive for the quiet period, the observer shall emit one snapshot marked stable.
- **ORC-05/AC-18**: When `SessionEnded` arrives, the observer shall close the session and emit no further snapshots.
- **ORC-05/AC-19**: When an observer attached through ORC-04's `Hub::attach_observer` joins a session that already holds events, the observer shall mark the replayed events as replay and the first later event as live.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run --no-fail-fast orchestrator`; unit tests in `src/orchestrator/screen_tests.rs` (new), `src/orchestrator/osc_tests.rs` (new), `src/orchestrator/observer_tests.rs` (new); AC-2 replays ORC-03.2 captures from `tests/fixtures/claude_code/` (new, ORC-03); AC-19 uses a real `Hub` with frames built by `lz4_flex::block::compress_prepend_size` as in `examples/fake_sharer.rs`; DoD `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` |
| macOS | Same nextest run (core is platform-neutral; no tray) |
| Linux | Same nextest run |

## Security / license review
- Does not touch `src/auth.rs`, hub roles or the HTTP/WebSocket surface, and adds no write path, so the threat-modeling review is not mandatory. PTY bytes and OSC bodies are agent-controlled input: the frame cap, OSC length cap and no-panic decoding are the mitigations. Because later stages gate automatic input on this item's "eligible" flag, a light `.claude/skills/threat-modeling-agentic-tools` pass alongside ORC-08 is recommended (owner to confirm).
- `lz4_flex` 0.11.6 is MIT (its Cargo.toml). The emulator crate's license must be checked against this repo's AGPL-3.0-only at the pinned version (verify at pinned version).

## Overlaps and existing work
- Replaces warpr-vh8nh (T-1) and warpr-66a37 (T-7).
- Consumes ORC-04 (`Hub::attach_observer`, `resync_observer`, ORC-04.1) and ORC-01 (`hub::RunState`, ORC-01.1); uses ORC-03.2 captures, never hand-written menus. AC-2 needs ORC-03's sidecar to list the option labels in screen order (`options`), a requested addition to ORC-03/AC-5.
- warp's own OSC 777 parser and CLI-agent status (`crates/warp_terminal/src/model/ansi/mod.rs`, `app/src/terminal/cli_agent_sessions/event/v1.rs`) are references for format only.
- AGT-07.1 (docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/AGT-07.md) is client-side orchestration, a different layer; not duplicated.

## Open questions (owner to confirm)
- Emulator crate: decided by the ORC-05.2 spike; recommendation is whichever passes AC-2 to AC-5 with fewer transitive deps.
- Mid-command attach seeding: recommend seeding from scrollback (AC-16) but keeping the session non-stable until a live redraw; alternative is a blank screen until the first full redraw.
- Frame cap: recommend a fixed per-frame cap well under `HubConfig::max_session_bytes`; alternative is reusing that limit.
- Quiet period: set from ORC-03 timings, configurable later in `orchestrator.toml` (ORC-07).

## References
- DESIGN.md §2, §4 (Observe), §8 (False prompt and redraw); `specs/self-hosted-remote-control/TECH.md`.
- Previous spec ACs AC-1, AC-2, AC-13, AC-14 in `specs/decision-orchestrator/spec.md`.
