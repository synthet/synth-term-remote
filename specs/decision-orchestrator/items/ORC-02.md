---
id: ORC-02
title: Resume accepts the client's empty secret when the reconnect token matches
priority: P0
score: tbd
effort: S
depends_on: [ORC-04]
milestone: M0
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-02] Resume accepts the client's empty secret when the reconnect token matches

## Problem
DESIGN.md §4 (Observe) and §7 ("session is re-shared under a new ID ... cancel the old pending
decision"): enrollment is bound to a live session id, so any sharer socket drop that forces a re-share
throws away the enrollment and every pending decision. Today every Synth Term reconnect against this
server fails: the client sends an empty session secret and the hub rejects it with `WrongPassword`, so
the terminal re-shares under a new id. ORC-14 (Delegate recovery) and the §8 "Reconnect and crash"
test need a resume that keeps the id.

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Exists | Resume checks `payload.session_secret != session.secret` first (`WrongPassword`), then the reconnect token (`WrongReconnectionToken`), with plain `!=` | `src/hub.rs` `Hub::resume_session` |
| Exists | `/sessions/{id}/resume` already requires bearer or cookie auth and a same-origin or native upgrade | `src/server.rs` `sharer_resume`, `check_ws_auth`; `src/auth.rs` `is_authorized`, `is_same_origin_or_native` |
| Exists | The session secret is checked nowhere else: viewers join on the server token alone | `src/hub.rs` `Hub::join_session` |
| Conflict | The client discards the secret from `SessionInitialized` (the arm destructures with `..`) and reconnects with `session_secret: Default::default()` | synthet/warp app/src/terminal/shared_session/sharer/network.rs (`SessionInitialized` arm; reconnect task building `ReconnectPayload`) |
| Exists | `SessionSecret(String)` derives `Default` (empty string) and `PartialEq`; `ReconnectToken(String)` has no byte accessor at rev f86afe1 | session-sharing-protocol rev f86afe1: src/common/session_params.rs, src/sharer.rs (cargo git checkout) |
| Partial | `resume_requires_secret_and_token_and_reports_last_event` covers a wrong token and the right pair; no case for an empty or a wrong non-empty secret | `src/hub_tests.rs` |
| Missing | Any resume over the real router | `tests/websocket_e2e.rs` (only `sharer_and_viewer_exchange_messages_end_to_end` and `upgrades_without_token_or_from_foreign_origins_are_refused`) |
| Exists | `constant_time_eq(a: &[u8], b: &[u8])` | `src/auth.rs` |

## Proposed change
- `src/hub.rs` `Hub::resume_session`: treat `payload.session_secret == SessionSecret::default()` as
  "no secret supplied". A non-empty secret must still equal `session.secret` (`WrongPassword`
  otherwise). The reconnect-token check is unchanged in meaning and stays mandatory in both cases.
- Keep the existing check order (secret, then token) so error reasons stay stable for old clients.
- Optional hardening (owner to confirm): compare the token with `auth::constant_time_eq` over its
  serde string form, since `ReconnectToken` has no accessor (verify at pinned version).
- No client change. The alternative is a synthet/warp change that keeps `session_secret` from
  `SessionInitialized` and sends it on reconnect.

## Scope / non-goals
- In: hub resume rule, unit tests, one end-to-end resume test.
- Out: the host-extension reconnect-token header (ORC-12), sharer replay de-duplication (already done
  in `handle_sharer_message`), changes to the pinned protocol, any change to viewer join.

## Acceptance criteria
- **ORC-02/AC-1**: When a sharer's `Reconnect` carries `SessionSecret::default()` and the session's reconnect token, the Hub shall resume the session and send `SessionReconnected` with the last stored event number.
- **ORC-02/AC-2**: If a `Reconnect` carries a non-empty session secret that differs from the session's secret, then the Hub shall reply `FailedToReconnect` with `WrongPassword` and not resume.
- **ORC-02/AC-3**: If a `Reconnect` carries the empty secret and a reconnect token that differs from the session's token, then the Hub shall reply `FailedToReconnect` with `WrongReconnectionToken` and not resume.
- **ORC-02/AC-4**: If a resume attempt fails, then the Hub shall leave the session's sharer connection and disconnect time unchanged, so `Hub::reap_abandoned` still ends it after the grace period.
- **ORC-02/AC-5**: When a sharer's socket drops and it resumes over `/sessions/{id}/resume` with the empty secret, the server shall keep the same session id and deliver its next `OrderedTerminalEvent` to a viewer that stayed joined.
- **ORC-02/AC-6**: Where a change relaxes a resume credential, the PR shall record a review with `.claude/skills/threat-modeling-agentic-tools`.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run -E 'test(hub) or binary(websocket_e2e)'`: extend `hub_tests` (empty secret + right token, wrong non-empty secret, empty secret + wrong token, failed attempt followed by `reap_abandoned`); new resume case in `tests/websocket_e2e.rs` that drops the sharer socket, resumes, sends an event and reads it on the viewer. |
| macOS | Same commands. |
| Linux | Same commands. |
| Manual (Windows) | Optional, once ORC-02.1 has landed: during a live share like ORC-03.2, drop the network or restart the server's listener briefly and confirm the Synth Term pane keeps its session id on the dashboard. |

DoD: `cargo fmt --check`; `cargo clippy --all-targets -- -D warnings`; `cargo nextest run --no-fail-fast`.

## Security / license review
- Touches auth on the remote-control surface: it drops one of two resume credentials. Needs the
  `.claude/skills/threat-modeling-agentic-tools` review (AC-6).
- Residual credential: the server token (bearer or cookie, checked by `check_ws_auth`) plus the
  reconnect token, a UUID v4 string sent only to the sharer in `SessionInitialized`. Anyone holding the
  server token can already join and type, so the secret added no separation between token holders.
- A hijacked resume replaces the live sharer connection (`sharer_conn` is overwritten); the review
  should confirm that the reconnect token never reaches viewers, logs or `/api/sessions`.
- No new crates.

## Overlaps and existing work
- Replaces warpr-jmobk (T-16) in epic warpr-oh1rg.
- ORC-12 (host extension route) authenticates with the same reconnect token; keep one comparison
  helper for both if the constant-time option is taken.
- ORC-14 (Delegate recovery) depends on this so a dropped sharer socket does not orphan an enrollment.
- Hub ordering: lands after ORC-04.1, which lands after ORC-01.1 (all edit `src/hub.rs`).

## Open questions (owner to confirm)
- Server-side acceptance (recommended: no client release needed) or a synthet/warp client fix that
  stores and resends the secret.
- Constant-time token comparison (recommended, cheap) or keep `!=`.

## References
- `specs/decision-orchestrator/DESIGN.md` §4, §7, §8 ("Reconnect and crash").
- `specs/decision-orchestrator/spec.md` (old AC-11); `specs/self-hosted-remote-control/TECH.md`.
