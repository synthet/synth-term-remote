---
id: ORC-04
title: Passive observer attach in Hub
priority: P0
score: tbd
effort: M
depends_on: [ORC-01]
milestone: M0
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-04] Passive observer attach in Hub

## Problem
DESIGN.md §2 ("Implement a passive headless viewer ... A watcher must not change terminal size or claim
interactive focus") and §4 step 2 (reconstruct from a snapshot plus ordered events). Owner decision: the
orchestrator runs in-process and is its own passive participant on `Hub`, not a browser viewer. A
normal viewer seat is wrong for it: it appears in the participant list, raises `viewer_count`, holds a
Full role and has an upstream path that relays input and size reports to the sharer.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Exists | Every joining viewer gets `config.viewer_role` (default `Role::Full`) and the shared owner uid, is inserted into `Session.viewers`, and triggers `broadcast_participant_list`, which also messages the sharer | `src/hub.rs` `Hub::join_session`, `Session::broadcast_participant_list`, `HubConfig` |
| Exists | Viewer upstream: `ReportTerminalSize` is relayed to the sharer whatever the role; `WriteToPty` / `ExecuteCommand` need `role.can_execute()` | `src/hub.rs` `Hub::handle_viewer_message` |
| Exists | Snapshot + replay: `JoinedSuccessfully` then `Session::replay_events(id, after)` | `src/hub.rs` `Hub::join_session`, `Session::replay_events` |
| Exists | All live fan-out (events, prompt, UDI, presence, role and participant-list updates, `SessionEnded`) goes through one function; targeted replies use `send_viewer` | `src/hub.rs` `Session::broadcast_viewers`, `Session::send_viewer` |
| Exists | All three session-end paths broadcast `SessionEnded` and then drop the `Session` | `src/hub.rs` `handle_sharer_message` (`EndSession`, size limit), `Hub::reap_abandoned` |
| Exists | `viewer_count` counts connected entries of `Session.viewers`; `participant_list()` is built from the same map | `src/hub.rs` `Hub::list_sessions`, `Session::participant_list` |
| Exists | Host sizing needs exactly one distinct present viewer uid | synthet/warp app/src/terminal/view/shared_session/view_impl.rs (`is_viewer_driven_sizing_eligible`) |
| Missing | Any observer type, map or API; `Hub` is re-exported for in-process use | `src/lib.rs` (`pub use hub::{Hub, HubConfig}`) |

## Proposed change
- `src/hub.rs`: `Session.observers: BTreeMap<ConnId, ViewerTx>`, keyed by a fresh id from
  `Hub::new_conn_id`. No `ParticipantInfo`, no role.
- `pub fn attach_observer(&self, session_id: SessionId, tx: ViewerTx) -> Option<ConnId>`: sends
  `JoinedSuccessfully` built like `join_session` (fresh `ParticipantId` and input replica id that are
  never inserted into `viewers`; `participant_list` without the observer), then every stored event.
  It does **not** call `broadcast_participant_list` and does not `touch()` the session.
- `pub fn detach_observer(&self, session_id: SessionId, observer: ConnId)` removes and drops the sender.
- `pub fn resync_observer(&self, session_id: SessionId, observer: ConnId, after: Option<usize>)`:
  `Some(n)` replays events with `event_no > n`; `None` sends a fresh `JoinedSuccessfully` and all events.
- `Session::broadcast_viewers` also sends to every observer (the `except` argument names viewers only),
  so the three session-end paths deliver `SessionEnded` and then drop observer senders with the
  `Session`. `send_viewer` stays viewer-only, so targeted replies never reach observers.
- `SessionSummary.observed: bool` (true while any observer is attached) for the tray (ORC-11; owner to
  confirm).
- No upstream path by construction: there is no observer variant in `handle_viewer_message`, no
  `handle_observer_message`, and no route in `src/server.rs`. The orchestrator holds only a receiver.

## Scope / non-goals
- In: hub API, fan-out, end-of-session cleanup, summary flag, unit tests.
- Out: an HTTP or WebSocket observer route, any write or answer path (ORC-12 host extension), the
  orchestrator consumer (ORC-05), session discovery notifications (ORC-05 polls `list_sessions`; owner
  to confirm), bounded channels or back-pressure.

## Acceptance criteria
- **ORC-04/AC-1**: When `Hub::attach_observer` is called for a live session, the Hub shall send that observer `JoinedSuccessfully` followed by every stored `OrderedTerminalEvent` in `event_no` order.
- **ORC-04/AC-2**: When an observer attaches or detaches, the Hub shall send no message to the sharer.
- **ORC-04/AC-3**: While an observer is attached, the Hub shall leave it out of `participant_list()` and out of `SessionSummary.viewer_count`.
- **ORC-04/AC-4**: While an observer is attached, the Hub shall deliver to it every message it broadcasts to viewers, including live `OrderedTerminalEvent`s, `ActivePromptUpdated` and `ParticipantListUpdated`.
- **ORC-04/AC-5**: The Hub shall give observers no upstream entry point, so no observer can cause `WriteToPtyRequested`, `CommandExecutionRequested` or `ViewerTerminalSizeReported` at the sharer.
- **ORC-04/AC-6**: The server router shall expose no route that attaches an observer.
- **ORC-04/AC-7**: When a session ends by `EndSession`, the size limit or `reap_abandoned`, the Hub shall send each observer `SessionEnded` and drop its sender.
- **ORC-04/AC-8**: When `resync_observer` is called with `after` set to N, the Hub shall replay only the stored events whose `event_no` is greater than N.
- **ORC-04/AC-9**: When `resync_observer` is called with `after` unset, the Hub shall send a fresh `JoinedSuccessfully` followed by all stored events.
- **ORC-04/AC-10**: When `detach_observer` is called, the Hub shall drop that observer's sender and deliver nothing more to it.
- **ORC-04/AC-11**: If `attach_observer` names a session that does not exist, then the Hub shall return `None` and keep no observer state.
- **ORC-04/AC-12**: While at least one observer is attached to a session, the Hub shall report `observed` true in that session's `SessionSummary`.
- **ORC-04/AC-13**: Where a change adds a participant class to the hub, the PR shall record a review with `.claude/skills/threat-modeling-agentic-tools`.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run -E 'test(hub)'`: new `src/hub_tests.rs` cases reusing `fixture`, `join`, `drain`, `pty_event`: attach after events (AC-1), sharer receiver empty after attach/detach (AC-2), `participant_list` / `viewer_count` with a viewer and an observer (AC-3, AC-12), live event and prompt update reach the observer (AC-4), `EndSession` / size cap / reap close the observer receiver (AC-7), resync with and without `after` (AC-8, AC-9), detach (AC-10), unknown session (AC-11). AC-5: a hub test sends `WriteToPty` and `ReportTerminalSize` through `handle_viewer_message` under the observer's `JoinedSuccessfully` `viewer_id` and asserts the sharer receives nothing. AC-6: review of `src/server.rs` `router`. |
| macOS | Same command. |
| Linux | Same command. |

DoD: `cargo fmt --check`; `cargo clippy --all-targets -- -D warnings`; `cargo nextest run --no-fail-fast`.

## Security / license review
- Touches hub roles and the remote-control surface: a new participant class that receives the full
  terminal stream. Needs the `.claude/skills/threat-modeling-agentic-tools` review (AC-13).
- Read-only by construction (AC-5, AC-6); it must stay in-process. Adding a network route later is a
  new review, because the stream contains everything typed and printed.
- An unbounded `ViewerTx` per observer matches viewers; memory is already capped by
  `max_session_bytes` for stored events, not for an observer that stops reading (owner to confirm).
- No new crates.

## Overlaps and existing work
- Replaces warpr-sczw3 (T-3) in epic warpr-oh1rg.
- Consumed by ORC-05 (observer stream); the ORC-12 server route depends on this item (lead's
  dependency hint).
- Hub ordering: after ORC-01.1, before ORC-02.1 (all edit `src/hub.rs`).
- Report item "Viewer counts look high" (`docs/reports/remote-control-verification-2026-11-05.md`) is
  unrelated; observers must not add to it.

## Open questions (owner to confirm)
- `observed` in `SessionSummary` now (recommended, the tray needs it) or added in ORC-11.
- Observer that stops reading: accept unbounded growth like viewers (recommended for M0) or bound it
  and force a `resync_observer(None)` on overflow.
- Should a sharer `UpdateAllRolesToReader` also detach observers? Recommended no: observers hold no role.

## References
- `specs/decision-orchestrator/DESIGN.md` §2, §3, §4; `specs/decision-orchestrator/spec.md` (Passive attach, old AC-5, AC-6).
- `specs/self-hosted-remote-control/TECH.md`.
