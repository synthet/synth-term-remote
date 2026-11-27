---
id: ORC-12
title: Host extension channel (schema v1, server route, host client)
priority: P0
score: TBD
effort: L
depends_on: [ORC-04]
milestone: M2
status: proposed
baseline: main 58e796c (2026-11-27)
---

# [ORC-12] Host extension channel (schema v1, server route, host client)

## Problem
DESIGN §3 makes the Synth Term host the owner of "the last check before applying a terminal answer", and
§5 defines `observe`, `decision.opened`, `answer`, `decision.updated/closed` and `take_over` as new
logical operations. The pinned `session-sharing-protocol` rev cannot gain messages, so the orchestrator
needs a separate, authenticated, per-session channel to the host. Without it Delegate mode (M2) has no
safe write path; the only write path today is a viewer's raw `WriteToPty` with broad authority (§2 "Input").

## Current state (58e796c)
| Status | What | Where |
|---|---|---|
| Exists | WS routes `/sessions/create`, `/sessions/{session_id}/resume`, `/sessions/join/{session_id}`; no `/synth/` route | `src/server.rs` `router` |
| Exists | Upgrade auth: origin then bearer/cookie | `src/server.rs` `check_ws_auth`; `src/auth.rs` `is_same_origin_or_native`, `is_authorized` |
| Exists | Constant-time compare (length leaks, content does not) | `src/auth.rs` `constant_time_eq` |
| Exists | Socket pattern: split, `spawn_writer` (closes when every sender drops), `next_message` (silently drops unparseable frames) | `src/server.rs` `run_sharer`, `spawn_writer`, `next_message` |
| Exists | Per-session `reconnect_token`, sent only to the sharer in `SessionInitialized` | `src/hub.rs` `Session`, `Hub::create_session` |
| Conflict | `ReconnectToken` has no string accessor; `Display`/`Debug` print `***`; only serde (`#[serde(transparent)]`) and `FromStr` reach the value; `Hub::resume_session` compares with derived `!=` | protocol rev f86afe1 `src/sharer.rs` `ReconnectToken` |
| Exists | Sessions end at three sites: `Up::EndSession`, the size-limit path in `handle_sharer_message`, `Hub::reap_abandoned` | `src/hub.rs` |
| Exists | Viewer input relayed to the sharer: `WriteToPty`, `ExecuteCommand`, `SendAgentPrompt`, `SendControlAction` | `src/hub.rs` `Hub::handle_viewer_message` |
| Exists | Child-module pattern: `#[path = "hub_tests.rs"] mod tests;` | `src/hub.rs` (end of file) |
| Partial | `fake_sharer` ignores the reconnect token in `SessionInitialized`; no host-link mode | `examples/fake_sharer.rs` `report`, `parse_options` |
| Missing | Wire schema, fixtures, hub link state, host client | `src/synth_ext.rs` (new), `src/hub_host_ext.rs` (new) |
| Exists (warp) | Env family `SYNTH_WARP_REMOTE_CONTROL_*`, `connect_headers` (bearer + `x-synth-host`) | warp `app/src/terminal/shared_session/self_hosted.rs` |
| Exists (warp) | Sharer stores `reconnect_token: Option<ReconnectToken>` | warp `app/src/terminal/shared_session/sharer/network.rs` |
| Exists (warp) | Share-established hook point | warp `app/src/terminal/local_tty/terminal_view_adaptor.rs` `NetworkEvent::SharedSessionCreatedSuccessfully` arm |

## Proposed change
1. **Schema** `src/synth_ext.rs` (new), `pub mod synth_ext` in `src/lib.rs`, tests in `src/synth_ext_tests.rs` (new),
   one golden fixture per type in `tests/fixtures/synth_ext/v1/` (new). Envelope `{"v":1,"type":...}`, compact JSON.
   - host → server: `hello` (session_id, host_id, execution_id, stream_epoch, capabilities), `state`
     (host_generation, last_event_no, run state, CLI agent status, window size), `answer.result`
     (attempt_id, status rejected{reason}/submitted/confirmed/uncertain), `decision.closed` (prompt_id, reason
     human_input/take_over/expired/prompt_changed/process_exited), `take_over` (origin host).
   - server → host: `observe.refresh`, `decision.opened` (prompt_id, generation, host_generation, policy_revision,
     options [{option_id, index, label}], kind, expires_at), `answer` (the DESIGN §5 example fields), `take_over`
     (origin tray/remote).
   - `answer` has no bytes or key field; the host owns key encoding (DESIGN §3 "Adapter").
2. **Hub link state** `src/hub_host_ext.rs` (new), declared in `src/hub.rs` with `#[path = "hub_host_ext.rs"] mod host_ext;`
   so it can read private `Session` fields. Adds `Session.host_ext` (new) holding `(ConnId, HostExtTx)` (new) and one
   in-process subscriber. Proposed API (names (new)): `Hub::attach_host_ext`, `Hub::handle_host_ext_message`,
   `Hub::host_ext_disconnected`, `Hub::subscribe_host_ext` returning `HostExtLink` (new): an event receiver
   (host message, remote input, link down, session ended) plus a sender for server → host messages.
   The link is not a participant: no `ViewerState`, no role, no entry in `participant_list`.
   Token check: serialize `session.reconnect_token` with `serde_json::to_value` and compare the inner string with
   `auth::constant_time_eq` (no accessor exists; verify at pinned rev).
3. **Route** `/synth/v1/sessions/{session_id}/host` (new) in `src/server.rs` `router`; handler runs `check_ws_auth`,
   requires `x-synth-reconnect-token` (new), then `run_host_ext` (new) on the `run_sharer`/`spawn_writer` pattern.
   A newer link replaces the old one, like `Hub::resume_session` replaces `sharer_conn` (owner to confirm).
4. **fake_sharer** `--ext` (new): keeps the token from `SessionInitialized`, opens the host link, sends `hello`/`state`,
   prints received ext frames as JSON lines. Without `--ext` nothing changes (web-tests spawn it via `web-tests/tests/harness.js`).
5. **Host client** in synthet/warp: covered by warp-i9ils (`app/src/terminal/shared_session/synth_host_ext/` (new)).

## Scope / non-goals
- In: schema, fixtures, route, hub link state, remote-input events, fake_sharer `--ext`.
- Out: the answer gate (ORC-13), the orchestrator's delegate logic (ORC-14), any change to the pinned protocol rev,
  an HTTP API for the orchestrator (it uses `HostExtLink` (new) in-process only), a browser UI for the link.

## Acceptance criteria
- **ORC-12/AC-1**: When each v1 golden fixture under `tests/fixtures/synth_ext/v1/` (new) is parsed and re-serialized, `synth_ext` (new) shall produce output byte-identical to the fixture.
- **ORC-12/AC-2**: If a host-link frame has an unknown `v` or `type` or is not valid JSON, `synth_ext` (new) shall return a parse error without panicking.
- **ORC-12/AC-3**: If an `answer` frame carries a `bytes`, `keys` or any other field outside the v1 schema, `synth_ext` (new) shall reject it.
- **ORC-12/AC-4**: If an upgrade to `/synth/v1/sessions/{id}/host` (new) has no valid bearer or cookie token, the server shall refuse it with 401.
- **ORC-12/AC-5**: If an upgrade to the host route comes from a foreign Origin, the server shall refuse it with 403.
- **ORC-12/AC-6**: If the `x-synth-reconnect-token` header is missing, or does not match the session's reconnect token, or the session does not exist, the server shall refuse the upgrade without creating a host link.
- **ORC-12/AC-7**: The hub shall compare the `x-synth-reconnect-token` header against the session's serialized reconnect token with `auth::constant_time_eq`.
- **ORC-12/AC-8**: When a second host link is accepted for a session, the hub shall close the previous link and deliver a link-down event to the subscriber before any frame from the new link (owner to confirm).
- **ORC-12/AC-9**: When a session ends through `EndSession`, the size limit or `reap_abandoned`, the hub shall close that session's host link and deliver a session-ended event to the subscriber.
- **ORC-12/AC-10**: When the hub relays a viewer's `WriteToPty`, `ExecuteCommand`, `SendAgentPrompt` or `SendControlAction` to the sharer of a session that has a `HostExtLink` (new) subscriber, the hub shall deliver a remote-input event to that subscriber.
- **ORC-12/AC-11**: While a host link is attached, the hub shall leave `participant_list()` and the `viewer_count` in `/api/sessions` unchanged.
- **ORC-12/AC-12**: The hub shall allow at most one in-process `HostExtLink` (new) subscriber per session and expose it through no HTTP or WebSocket route.
- **ORC-12/AC-13**: If a frame on an established host link fails to parse, the server shall close that link and deliver a link-down event to the subscriber (owner to confirm).
- **ORC-12/AC-14**: When `fake_sharer` runs with `--ext` (new), it shall open the host link with the reconnect token from `SessionInitialized` and send a v1 `hello` and `state`.
- **ORC-12/AC-15**: When `fake_sharer` runs without `--ext` (new), it shall keep its 58e796c stdin/stdout behavior so the web-tests Playwright suite passes unchanged.
- **ORC-12/AC-16**: If `SYNTH_WARP_REMOTE_CONTROL_DECISIONS` (new) or the remote-control URL is unset, Synth Term shall open no host-extension socket.
- **ORC-12/AC-17**: When Synth Term publishes `hello` and `state`, its frames shall match the v1 golden fixtures in `tests/fixtures/synth_ext/v1/` (new).
- **ORC-12/AC-18**: When CLI agent status, the active block, a PTY write other than a focus report, or the window size changes, Synth Term shall re-send `state` with a higher `host_generation`.
- **ORC-12/AC-19**: Synth Term shall build the `x-synth-reconnect-token` header from the token's serialized string, never from its masked `Display` output.
- **ORC-12/AC-20**: When a PR adds the host route or hub link state, the PR shall record a review with `.claude/skills/threat-modeling-agentic-tools`.

## Test plan
| Platform | How |
|---|---|
| Windows | `cargo nextest run synth_ext hub`; `cargo nextest run -E 'binary(websocket_e2e)'`; web-tests `npm test` with `fake_sharer` unchanged |
| macOS | Same cargo commands in CI (server is cross-platform); warp-i9ils `self_hosted` tests |
| Linux | Same cargo commands in CI |

Unit: fixture round-trip and rejects in `src/synth_ext_tests.rs` (new); hub link lifecycle (replace, three end sites,
remote-input, participant list unchanged) in `src/hub_tests.rs`. E2E: refusal matrix (no token, wrong token, foreign
Origin, missing/wrong reconnect token, unknown session) in `tests/websocket_e2e.rs`, following
`upgrades_without_token_or_from_foreign_origins_are_refused`.

## Security / license review
Touches auth and the remote-control surface (new upgrade route, new per-session secret check, a new input-adjacent
channel): needs the `.claude/skills/threat-modeling-agentic-tools` review (ORC-12/AC-20). Threats to cover: a browser
viewer holding the cookie opening the host link (blocked by the reconnect token, which viewers never receive), token
leakage in logs (`Display` masks it; never log the header), frame floods (bounded by `configure` `max_message_bytes`;
a smaller cap is an option, owner to confirm). No new crates: serde, serde_json, axum `ws`, tokio-tungstenite (dev) are
already in `Cargo.toml`. The warp host client reuses its existing WebSocket client (verify in warp-i9ils).

## Overlaps and existing work
- Replaces warpr-gcfh8 (T-4, wire schema) and warpr-bzxpe (T-6, route and hub extension state).
- Host client and state publisher: warp-i9ils (ready). Its body still cites warpr-gcfh8/warpr-bzxpe, spec.md "H-1" and
  PR #3; lead to re-point it at ORC-12.1 and ORC-12.2. Its ACs are mirrored as ORC-12/AC-16..AC-18; AC-19 is new.
- warp-htojy also edits `self_hosted.rs`; land the `SYNTH_WARP_REMOTE_CONTROL_DECISIONS` (new) constant after or rebased on it.
- Depends on ORC-04 (observer attach; also edits `src/hub.rs`). Roadmap: docs/synth-term-roadmap:specs/SYNTH-TERM-ROADMAP/items/INT-02.md
  (INT-02.1 re-verification; scoped capabilities are INT-02's broader goal, this channel is one narrow capability).

## Open questions (owner to confirm)
- Second link: recommend newest replaces oldest (matches `resume_session`); alternative refuse while one is live (risks a
  half-open socket blocking reconnects).
- Bad frame on an established link: recommend close (fail closed); alternative drop and continue like `next_message`.
- Unknown fields: recommend strict `deny_unknown_fields` on all v1 types, new fields mean v2; alternative strict only on `answer`.
- Flag name: `SYNTH_WARP_REMOTE_CONTROL_DECISIONS` (new) matches the current `SYNTH_WARP_REMOTE_CONTROL_*` env family in
  `self_hosted.rs`; no `SYNTH_TERM_` env exists on warp master. Rename only together with the whole family.

## References
- DESIGN.md §3, §5 ("Host response semantics", "Key mapping"), §8 M2.
- `src/server.rs`, `src/hub.rs`, `src/auth.rs`, `examples/fake_sharer.rs`, `tests/websocket_e2e.rs`.
- warp `app/src/terminal/shared_session/self_hosted.rs`, `sharer/network.rs`, `local_tty/terminal_view_adaptor.rs`.
