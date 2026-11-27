---
id: ORC-11
title: Headless tray UX: sessions, Enroll, Take over, balloons
priority: P1
score: 6
effort: M
depends_on: [ORC-07, ORC-10]
milestone: M1
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-11] Headless tray UX: sessions, Enroll, Take over, balloons

## Problem
The owner chose a no-browser UX: the tray window lists sessions, an Enroll submenu binds a profile
from `orchestrator.toml`, Take over is one click, and recommendations and escalations raise
balloons. DESIGN.md §7 ("Several agents ask questions at once") requires per-row host, workspace,
task, question, age, mode and status, and §3 says taking over "invalidates any outstanding automatic
decision". §7 also forbids picking a target by title, working directory or tab position, so every
UI action must carry a session id. Today the tray only shows a status label and a log.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Exists | Window with a status label, three buttons, a read-only log box, a popup tray menu (Show, Open, Copy token, Quit) | `src/tray/app.rs` `App::build`, `App::handle_event` |
| Exists | UI refresh on `nwg::Notice` from other threads | `src/tray/app.rs` `App::refresh`, `E::OnNotice` |
| Exists | Tray icon and tip; balloons via `TrayNotification::show(text, title, flags, icon)` under the enabled `tray-notification` feature | `src/tray/app.rs` `App::tray`; native-windows-gui 1.0.13 src/controls/tray_notification.rs |
| Missing | `list-view` feature: the `nwg::ListView` control is behind it (the `OnListViewRightClick` event variant is not, but its event data is) | `Cargo.toml` native-windows-gui features; nwg 1.0.13 src/controls/mod.rs and src/events.rs `cfg(feature = "list-view")` |
| Exists | Submenus: the `Menu` docs allow "a top level window, a menu" as parent; `Menu` and `MenuItem` destroy their handle on drop (rebuilding items at runtime: verify at pinned version) | nwg 1.0.13 src/controls/menu.rs `Menu` docs, `impl Drop for MenuItem` |
| Missing | `timer` / `animation-timer` features (a periodic age refresh would need one) | `Cargo.toml` |
| Exists | Rule: tray logic testable without Windows goes in `src/tray_support.rs`; `src/tray/app.rs` is in the tray binary, not the library | `AGENTS.md` Rules; `src/tray/main.rs` `mod app` |
| Missing | UI model, session list, Enroll, Take over, balloons | - |

## Proposed change
1. **UI model** in `src/orchestrator/ui.rs` (new), tests in `src/orchestrator/ui_tests.rs` (new),
   compiled on all platforms like `src/tray_support.rs`. Pure functions over
   `OrchestratorHandle::snapshot()` (ORC-10.2):
   - `rows(&[SessionView], now) -> Vec<RowText>` with the columns Host, Folder, Task, Mode, Status,
     Question, Age. Question is one line, capped (cap owner to confirm). Unenrolled sessions show
     mode "-".
   - `enroll_entries(profiles) -> Vec<MenuEntry>` in profile-name order.
   - `action(menu_choice, captured: SessionId) -> UiAction` with the variants
     `Enroll { session_id, profile }`, `Unenroll { session_id }`, `TakeOver { session_id }`,
     `Resume { session_id }` and `ReloadConfig`. The session id is captured when the context menu
     opens, never looked up again by row index.
   - `balloon(&Notice) -> Option<(title, text)>` for recommendation and escalation notices: host,
     profile, question and option label, capped (the shell's balloon length limit: verify at pinned
     version). No excerpt text.
   - `status_line(Option<&OrchestratorHandle>)` gives "Orchestrator off" when the config is missing or invalid.
2. **Rendering** in `src/tray/app.rs`:
   - Add the `list-view` feature to native-windows-gui in `Cargo.toml`.
   - Add a detailed-style `nwg::ListView` below the buttons; the log box shrinks or moves (layout
     owner to confirm).
   - Right-click a row: a popup with an "Enroll as" submenu (one item per profile), Unenroll, Take
     over and Resume; items are enabled from the row state.
   - Tray menu: add "Reload orchestrator config" (calls `reload_config`, rebuilds the Enroll items).
   - `App::refresh` (already run on notice) re-renders rows, drains notices and calls
     `self.tray.show(..)` once per notice.
   - The handle comes from the `Arc<Mutex<Option<OrchestratorHandle>>>` slot set in ORC-07.3.
   - Age updates on each notice and when the window is revealed. A periodic tick needs a new nwg
     feature (owner to confirm).

## Scope / non-goals
- In: UI model and tests, list view, context and tray menu entries, balloons, the off state.
- Not: the web dashboard (no `web/` change), a macOS tray, Delegate controls (ORC-14), editing
  `orchestrator.toml` from the UI, balloon click-through actions.

## Acceptance criteria
- **ORC-11/AC-1**: When ui.rs receives a snapshot, it shall produce one row per live session with host, folder, task, mode, status, question and age.
- **ORC-11/AC-2**: When the Enroll submenu is built, ui.rs shall list exactly the profiles from the loaded orchestrator.toml in name order.
- **ORC-11/AC-3**: When the user picks a profile for a row, ui.rs shall emit Enroll with that row's session id and profile name and no pwd, title or tab.
- **ORC-11/AC-4**: If the session list changes between opening the context menu and choosing an item, then ui.rs shall target the session id captured when the menu opened.
- **ORC-11/AC-5**: When the user picks Take over for a row, the tray shall call OrchestratorHandle::take_over with that row's session id.
- **ORC-11/AC-6**: When a recommendation or escalation notice arrives, ui.rs shall produce bounded balloon text naming the host, the question and the outcome.
- **ORC-11/AC-7**: While the orchestrator is off, the tray shall show that state and disable Enroll, Take over and Resume.
- **ORC-11/AC-8**: While the tray runs on Windows, it shall show the session list, the Enroll submenu and balloons without opening a console window.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run -E 'test(orchestrator::ui)'`; manual: run `synth-remote-control-tray` with a valid config and two `fake_sharer` sessions, enroll one, trigger a recommendation through the stub decider, see the balloon, Take over, Reload; record screenshots in the PR (AC-8) |
| macOS | `cargo nextest run -E 'test(orchestrator::ui)'` (model only; the tray is Windows-only) |
| Linux | Same as macOS |

The manual check can use win-gui-mcp screenshots. Pattern for tests: `src/tray_support_tests.rs`.

## Security / license review
The tray is a local control, not a network surface, and it does not touch auth or hub roles. It is
the only M1 entry point for enrollment and take-over (ORC-07/AC-10), so AC-3 and AC-4 guard the
§7 rule against targeting by title or position. Balloon text comes from untrusted terminal output:
it is capped, single-line, and carries no excerpts or tokens. A threat-model review is not required
for this item alone; ORC-07 and ORC-10 carry it. No new crates: `list-view` is a feature of the
existing native-windows-gui 1.0.13 (MIT).

## Overlaps and existing work
- Replaces warpr-7tsb0 (T-12).
- ORC-07.3 provides the handle slot and `reload_config`; ORC-10.2 provides `snapshot`,
  `drain_notices`, `take_over` and `resume`.
- ORC-15 uses this UI for the real-host positive control; ORC-14 adds Delegate status and remote take-over.
- warp SEC-01 (docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/SEC-01.md) owns Synth Term's own approval UI; this tray is the server's.

## Open questions (owner to confirm)
- Placement: `src/orchestrator/ui.rs` (new; recommended, keeps the orchestrator's view model with its
  types) or `src/tray_support.rs` as AGENTS.md suggests for testable tray logic. Either way it stays
  out of `src/tray/app.rs`.
- Layout: the list above a shorter log (recommended) or a tab between Sessions and Log (needs the `tabs` feature).
- Periodic age refresh: add an nwg timer feature (verify which at pinned version) or refresh on notice only (recommended for M1).

## References
- specs/decision-orchestrator/DESIGN.md §3 (desktop row), §7 (first, second and fourth rows).
- specs/decision-orchestrator/spec.md (old AC-23, AC-24), tasks.md (T-12).
