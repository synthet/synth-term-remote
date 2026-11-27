---
id: ORC-01
title: Authoritative run state for shared sessions (running/idle/unknown)
priority: P0
score: tbd
effort: M
depends_on: []
milestone: M0
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-01] Authoritative run state for shared sessions (running/idle/unknown)

## Problem
DESIGN.md §2 ("Real-client verification") and §8 M0: a terminal shared mid-command reports `idle`, so
the web viewer routes keystrokes to the command bar and a waiting agent never gets them. §5
`observe(session)` asks for "authoritative running state or `unknown`", and §7 says unknown state
must keep the session ineligible for automatic input. Today there is no unknown state at all: the
server guesses `false`. This is the main phone-workflow blocker recorded in
`docs/reports/remote-control-verification-2026-11-05.md` ("A terminal shared mid-command shows idle").

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Exists | `Session.command_running: bool` starts `false` and flips only on `CommandExecutionStarted` / `CommandExecutionFinished` | `src/hub.rs` `Hub::create_session`, `Hub::handle_sharer_message` |
| Exists | Share-time scrollback is parsed once for `pwd` from the client's `SerializedBlock` JSON | `src/hub.rs` `last_scrollback_pwd` |
| Exists | `SessionSummary.command_running` feeds the dashboard pill | `src/hub.rs` `Hub::list_sessions`; `web/app.js` (pill in the session list) |
| Missing | Any `unknown` state; any use of `Scrollback.is_alt_screen_active` or block `did_execute` / `start_ts` / `completed_ts` | `src/hub.rs` |
| Conflict | `onJoined` hard-codes `this.running = false`, ignoring the snapshot it was just given | `web/app.js` `RemoteTerminal.onJoined` |
| Exists | Playwright regressions for the mid-command cases (they cannot pass today) | `web-tests/tests/regressions.spec.js` ("a terminal shared mid-command shows running and takes input", "the dashboard shows a terminal shared mid-command as running") |
| Exists | Fake sharer emits a running block (`did_execute`, `start_ts`, `completed_ts: null`) and `--alt-screen` | `examples/fake_sharer.rs` `active_block`, `parse_options` |
| Conflict | `@playwright/test` is the literal placeholder `"PLAYWRIGHT_VERSION"`, so `npm install` in web-tests fails | `web-tests/package.json` |
| Exists | Host side: `did_execute` is true for `Executing`, `DoneWithExecution` and `Static` blocks; hidden active blocks are dropped from share scrollback | synthet/warp app/src/terminal/model/block/serialized_block.rs (`impl From<&Block> for SerializedBlock`), app/src/terminal/shared_session/mod.rs (`SharedSessionScrollbackType::to_scrollback`) |
| Exists | Host side candidate signal: the share's `InitPayload.universal_developer_input_context.long_running_command_agent_interaction` is `Some` when `active_block.is_active_and_long_running()` held at share creation, unless a cached UDI update replaced it (`cached_latest_state`); that is the predicate the host uses to accept `WriteToPty` | synthet/warp app/src/terminal/local_tty/terminal_view_adaptor.rs (UDI build before `Network::new`; `WriteToPtyRequested` arm), app/src/terminal/shared_session/sharer/network.rs (`InitPayload` build) |

## Proposed change
- `src/hub.rs`: add `pub enum RunState { Running, Idle, Unknown }` (serde `lowercase`) and
  `pub fn scrollback_run_state(scrollback: &Scrollback) -> RunState` beside `last_scrollback_pwd`,
  reading the same `ScrollbackBlock.raw` JSON. Rule for the share-time snapshot:
  alt screen active -> Running; else parse the **last** block: `did_execute` true with non-null
  `start_ts` and null `completed_ts` -> Running; `did_execute` false -> Idle; anything else (no blocks,
  bad JSON, missing `did_execute`, last block already completed, which means the active block was
  hidden) -> Unknown. Requiring `start_ts` keeps a trailing `Static` block from reading as running.
- Replace `Session.command_running` with `run_state: RunState`; `CommandExecutionStarted` sets
  Running and `CommandExecutionFinished` sets Idle from any state.
- `SessionSummary` gains `run_state`; `command_running` stays as `run_state == Running` for
  existing consumers (owner to confirm; drop it later).
- `web/app.js`: `onJoined` derives the initial state from `body.scrollback` with the same rule (mirror of
  `scrollback_run_state`), then the replayed events move it on as today. Recommended over fetching
  `/api/sessions`, because the snapshot plus ordered replay is self-consistent and a fetch races the
  replay (owner to confirm). The status pill and the dashboard pill gain an `unknown` label.
- Unknown in the web viewer keeps today's idle routing (command bar and `ExecuteCommand`) and only
  changes the pill label. Real idle shares may yet derive as unknown (first open question), so routing
  keys to the PTY on unknown could break idle typing that works today (owner to confirm).
- `examples/fake_sharer.rs`: add a `--no-scrollback` option (proposed) so Playwright can exercise Unknown.
- `web-tests/package.json`: pin `@playwright/test` to an exact version (verify at pinned version).
- `RunState` and `scrollback_run_state` are `pub` so ORC-05 imports them instead of defining its own.

## Scope / non-goals
- In: share-time derivation, event transitions, dashboard and viewer display, Playwright pin.
- Out: `pwd` updates after share (separate report bug; the `test.fixme` stays), `is_agent` for CLI
  agents, any protocol change (the `session-sharing-protocol` rev is pinned), using the UDI signal
  before ORC-03 captures confirm it.

## Acceptance criteria
- **ORC-01/AC-1**: When a sharer initializes a session whose `Scrollback.is_alt_screen_active` is true, the Hub shall set that session's run state to `running`.
- **ORC-01/AC-2**: When a sharer initializes a session whose last scrollback block has `did_execute` true, a non-null `start_ts` and a null `completed_ts`, the Hub shall set the run state to `running`.
- **ORC-01/AC-3**: When a sharer initializes a session whose last scrollback block has `did_execute` false and alt screen is off, the Hub shall set the run state to `idle`.
- **ORC-01/AC-4**: If the share-time scrollback has no blocks, or its last block is unparseable, lacks `did_execute`, or has a non-null `completed_ts`, then the Hub shall set the run state to `unknown`.
- **ORC-01/AC-5**: When the Hub stores a `CommandExecutionStarted` or `CommandExecutionFinished` event, the Hub shall set the run state to `running` or `idle` respectively, whatever the previous state was.
- **ORC-01/AC-6**: The Hub shall report `run_state` as one of `running`, `idle` or `unknown` in every `/api/sessions` row, with `command_running` true only when `run_state` is `running`.
- **ORC-01/AC-7**: When the web viewer joins a session shared mid-command, the web viewer shall send typed keys as `WriteToPty` before any `CommandExecutionStarted` event arrives.
- **ORC-01/AC-8**: When the web viewer joins a session whose snapshot yields unknown, the web viewer shall show `unknown` in the status pill.
- **ORC-01/AC-9**: While a session with a connected sharer has run state `unknown`, the dashboard shall show an `unknown` pill for it.
- **ORC-01/AC-10**: When `npm test` runs in `web-tests` after building the server and examples, the Playwright suite shall install a pinned `@playwright/test` and pass every test in `web-tests/tests/regressions.spec.js` except the existing `test.fixme`.
- **ORC-01/AC-11**: Where a PR changes how the web viewer routes input, the PR shall record a review with `.claude/skills/threat-modeling-agentic-tools`.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run -E 'test(hub)'`: new `hub_tests` cases per AC-1..AC-6 (alt screen, running block, idle block, empty / bad / field-less / completed block, Started then Finished over Unknown); update `dashboard_summary_reflects_session_state` to assert `run_state`. `cargo build --bins --examples` then `npm test` in `web-tests` (AC-7..AC-10), plus a new regression using `--no-scrollback` for AC-8 / AC-9. |
| macOS | Same commands; Playwright Chromium only (WebKit not installed per the 2026-11-05 report). |
| Linux | Same cargo and Playwright commands. |

DoD: `cargo fmt --check`; `cargo clippy --all-targets -- -D warnings`; `cargo nextest run --no-fail-fast`.

## Security / license review
- Touches the remote-control surface: it changes where browser keystrokes go (command bar vs PTY) and
  adds a field to the authenticated `/api/sessions` response. No auth or hub-role change. Needs the
  `.claude/skills/threat-modeling-agentic-tools` review (AC-11), scoped to input routing on unknown.
- Unknown must never be treated as eligible for automatic input by later items (ORC-08, ORC-14).
- No new crates. The Playwright pin is Apache-2.0 upstream (verify at pinned version); dev-only.

## Overlaps and existing work
- Replaces warpr-me1nx (T-2) in epic warpr-oh1rg.
- Report bug "A terminal shared mid-command shows idle" and the `onJoined` note in
  `docs/reports/remote-control-verification-2026-11-05.md`; the report's proposed fix is this item.
- Roadmap INT-02/AC-3 (docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/INT-02.md, task INT-02.1)
  asks for the same mid-command routing on the client side; this item is the server/web half.
- ORC-03 captures confirm the real block fields; ORC-05 reuses `RunState`; ORC-04 and ORC-02 edit
  `src/hub.rs` after this item (ordering ORC-01 -> ORC-04 -> ORC-02).

## Open questions (owner to confirm)
- Shape of a real idle share is unverified: `fake_sharer` always sends a `did_execute: false` block,
  but `Block::should_hide_block` (synthet/warp app/src/terminal/model/block.rs) can drop the active block
  and the 2026-11-05 report saw blank `pwd` for terminals with no finished blocks. If ORC-03 shows idle
  shares ending on a completed block or no block, AC-3 and AC-4 change before ORC-01.1 merges.
- Initial state in `onJoined`: derive from `body.scrollback` (recommended) or fetch `/api/sessions`
  `run_state` (simpler code, but races replayed events).
- Add the UDI `long_running_command_agent_interaction` signal as the first rule once ORC-03 shows it in
  real `InitPayload`s (recommended), or keep scrollback-only.
- Keep `command_running` in `SessionSummary` for one release (recommended) or remove it now.
- Web routing on unknown: idle routing (recommended until ORC-03 settles the idle shape) or `WriteToPty`.

## References
- `specs/decision-orchestrator/DESIGN.md` §2, §5, §7, §8; `specs/decision-orchestrator/spec.md` (old AC-3, AC-4).
- `docs/reports/remote-control-verification-2026-11-05.md`; `specs/self-hosted-remote-control/TECH.md`.
