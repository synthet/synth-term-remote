---
id: ORC-07
title: orchestrator.toml, enrollment and startup wiring
priority: P1
score: 7
effort: M
depends_on: [ORC-05]
milestone: M1
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-07] orchestrator.toml, enrollment and startup wiring

## Problem
DESIGN.md §1 ("Operating modes") makes mode, workspace, file access, decision classes and provider
access explicit per enrollment: "Starting a terminal or sharing a pane does not enroll it." §4 step 1
binds an enrollment to one session, and §7 forbids recovering a write target "by title, working
directory, or tab position". The owner chose a no-browser UX: an `orchestrator.toml` in the tray
data dir plus tray controls. Today the server has no config file, no enrollment concept, and
`launch::serve` owns the `Hub` privately, so nothing else in the process can observe sessions.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Exists | Shared CLI and env settings, all `SYNTH_RC_*` env names; fields are private except `bind` | `src/launch.rs` `Args` |
| Exists | Token-file pattern to follow for a config path | `src/launch.rs` `Args::configured_token`; `src/tray_support.rs` `load_or_create_token` |
| Partial | `serve` builds `Arc<Hub>` and `AppState` itself and returns only on shutdown, so no handle leaves it | `src/launch.rs` `serve` |
| Exists | Both binaries call `bind` then `serve` | `src/main.rs` `main`; `src/tray/app.rs` `spawn_server` |
| Exists | Tray data dir `%LOCALAPPDATA%\SynthRemoteControl` holding `token` and `server.log` | `src/tray/app.rs` `data_dir`, `start` |
| Exists | Cross-thread pattern: `Arc<Mutex<Option<Status>>>` plus `nwg::NoticeSender` | `src/tray/app.rs` `spawn_server`, `App::refresh` |
| Exists | Live session ids and labels | `src/hub.rs` `Hub::list_sessions`, `SessionSummary` |
| Partial | `toml` 1.2.1 is in `Cargo.lock` only as a dependency of the build-time crate `embed-resource`; it is not a direct dependency | `Cargo.lock`, `Cargo.toml` |
| Missing | Config schema, loader, policy revision, enrollment registry, startup flags | - |

## Proposed change
1. **Config** in `src/orchestrator/config.rs` (new), tests in `src/orchestrator/config_tests.rs` (new).
   Add `toml` as a direct dependency (verify at pinned version; the lock already has 1.2.1).
   Types: `OrchestratorConfig { orchestrator, decider, profiles }` with
   `#[serde(deny_unknown_fields)]` on every table, so a stray `api_key` or `token` field fails to load.
   - `[orchestrator]`: `max_concurrent_decisions` (consumed by ORC-10; default owner to confirm).
   - `[decider.jev]`: `command`, `args`, `timeout_secs` (consumed by ORC-09). No key field exists.
   - `[profiles.<name>]`: `mode` (`observe` / `recommend` / `delegate`), `task` (goal text),
     `workspace_root` (absolute), `attachments` (paths relative to the root), `send_excerpts`
     (default `false`), `decision_classes` (names owned by ORC-08), `budget` (bytes, seconds).
   - `delegate_versions` (list of adapter, recognizer and Claude Code versions): parsed here so
     `deny_unknown_fields` accepts it; ORC-15.1 enforces it, and the example file lists none.
   - Validation: absolute `workspace_root`, a known decider, non-empty profile names.
   - `policy_revision: u64` is taken from the first 8 bytes of the SHA-256 of the file bytes (sha2
     arrives with ORC-05.1), so it is the same across restarts and changes whenever the content
     changes. It is computed at load time; reload happens on start and on an explicit
     `OrchestratorHandle::reload_config` (tray menu in ORC-11). No file watcher (owner to confirm).
   - `config/orchestrator.example.toml` (new) documents every field, with a comment that provider
     keys stay in the environment.
2. **Enrollment** in `src/orchestrator/enroll.rs` (new), tests in `src/orchestrator/enroll_tests.rs` (new).
   `EnrollmentRegistry` maps `SessionId` to `Enrollment { profile, mode, policy_revision }`.
   `enroll(session_id, profile, live: &[SessionSummary])` fails with `UnknownProfile`,
   `SessionNotLive` or `DelegateUnavailable` (M1 refuses delegate until ORC-14 lifts it).
   `unenroll` and `session_ended` remove the row. No function takes a pwd, title or tab, and nothing
   enrolls automatically. Enrollments are runtime-only: sessions live in memory in `Hub`, so a
   restart ends them anyway. A reload drops enrollments whose profile disappeared and re-stamps the
   rest with the new revision (owner to confirm).
3. **Startup** in `src/launch.rs`, tests in `src/launch_tests.rs` (new). New `Args` fields
   (names owner to confirm, following the `SYNTH_RC_*` convention): `--orchestrator-config` /
   `SYNTH_RC_ORCHESTRATOR_CONFIG` and `--data-dir` / `SYNTH_RC_DATA_DIR`. Add
   `Args::orchestrator_paths(default_dir)` returning the config and state paths (state is
   `<data_dir>/orchestrator/`); the console server uses the config's parent when `--data-dir` is unset.
   `serve` gains `on_orchestrator: impl FnOnce(OrchestratorHandle) + Send + 'static`; it builds the
   `Hub` as today, then starts the orchestrator only when a valid config loads (owner to confirm;
   alternative: a new `launch::build_hub` so the caller owns the `Arc<Hub>`). `src/main.rs` passes
   a no-op. `OrchestratorHandle` (defined here in `src/orchestrator/mod.rs` (new, ORC-05.1)) starts
   with `profiles()`, `enroll`, `unenroll`, `reload_config` and `policy_revision()`; ORC-10.2 adds the
   session pipeline, `snapshot` and `take_over`.
4. **Tray** in `src/tray/app.rs`: `start` passes `data_dir()` as the default dir, so the config is
   `<data_dir>/orchestrator.toml`. `spawn_server` stores the handle in an `Arc<Mutex<Option<..>>>`
   and calls `notice.notice()`, mirroring `Status`.

## Scope / non-goals
- In: schema, loader, revision, registry, flags, handle skeleton, tray and console wiring, example file.
- Not: policy evaluation (ORC-08), evidence and the decider (ORC-09), the session pipeline (ORC-10),
  tray menus (ORC-11), delegate enablement (ORC-14), an HTTP enrollment API, a file watcher.

## Acceptance criteria
- **ORC-07/AC-1**: If no orchestrator config file exists at the resolved path, then the server shall start and serve exactly as at 58e796c with no orchestrator task.
- **ORC-07/AC-2**: If the config file fails to parse or validate, then the server shall log one error naming the file and keep serving with the orchestrator off.
- **ORC-07/AC-3**: If the config contains a field outside the schema, including any key, token or secret field, then the loader shall reject the file.
- **ORC-07/AC-4**: When the config content changes and is loaded again, the loader shall produce a different policy_revision.
- **ORC-07/AC-5**: While the config content is unchanged, the loader shall produce the same policy_revision across process restarts.
- **ORC-07/AC-6**: When a session is shared, the orchestrator shall keep it unenrolled until enroll is called with that session id.
- **ORC-07/AC-7**: When enroll is called with a live session id and a known profile, the registry shall bind that profile to that session id only.
- **ORC-07/AC-8**: When a session ends or its terminal re-shares under a new id, the registry shall drop the old enrollment and leave the new id unenrolled.
- **ORC-07/AC-9**: If enroll names a delegate-mode profile, then the M1 registry shall refuse it with DelegateUnavailable.
- **ORC-07/AC-10**: The orchestrator shall accept enrollment only through the in-process OrchestratorHandle, with no HTTP or WebSocket route.
- **ORC-07/AC-11**: When the tray starts without --orchestrator-config, it shall load orchestrator.toml from its data_dir and keep orchestrator state under data_dir/orchestrator.
- **ORC-07/AC-12**: When the ORC-07.2 and ORC-07.3 pull requests open, each shall record a threat-modeling-agentic-tools review.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run -E 'test(orchestrator::config) or test(orchestrator::enroll) or test(launch)'`; tray started by hand with no config, an invalid config and a valid one, reading `server.log` |
| macOS | Same nextest filter on the console server build; `--orchestrator-config` pointed at a temp file |
| Linux | Same as macOS |

Registry tests build `SessionSummary` rows directly. A session-ended test drives `Hub::create_session`
and the sharer end path, following `src/hub_tests.rs`. DoD: `cargo fmt --check`,
`cargo clippy --all-targets -- -D warnings`, `cargo nextest run --no-fail-fast`.

## Security / license review
Enrollment is the authority grant for every later decision, so this touches the remote-control
surface indirectly: run `.claude/skills/threat-modeling-agentic-tools` on ORC-07.2 and ORC-07.3.
Auth (`src/auth.rs`) and hub roles are not changed. Fail closed: an invalid config means no
orchestrator. Keys never enter the toml (AC-3). New direct crate: `toml` (MIT OR Apache-2.0, verify
at pinned version), compatible with AGPL-3.0-only.

## Overlaps and existing work
- Replaces warpr-slzc1 (T-5), which stays open until the new import cancels it.
- warp SEC-01.3 (workspace trust store, docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/SEC-01.md):
  same concept (explicit, revocable, per-workspace grants), different layer. Reuse its vocabulary;
  do not depend on it.
- warp AGT-07.1 (local task queue, docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/AGT-07.md): a different layer (Synth Term's own agents).
- Consumers: ORC-08 (`decision_classes`, revision), ORC-09 (`[decider.jev]`, attachments), ORC-10.2
  (handle), ORC-11 (profiles and reload in the tray).

## Open questions (owner to confirm)
- Flag names `--orchestrator-config` and `--data-dir`. Recommendation: keep them with `SYNTH_RC_` env twins.
- Revision as a content digest (recommended, stable across restarts) or a counter bumped on each changed load.
- Handle hand-off: an `on_orchestrator` callback in `serve` (recommended, smallest diff) or `launch::build_hub`.
- Reload removing a profile: drop its enrollments (recommended) or keep them on the old revision until unenrolled.

## References
- specs/decision-orchestrator/DESIGN.md §1, §4 step 1, §6 (first paragraph), §7.
- specs/decision-orchestrator/spec.md (old AC-8, AC-9), tasks.md (T-5).
