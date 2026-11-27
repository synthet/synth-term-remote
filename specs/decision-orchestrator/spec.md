# Headless decision orchestrator: spec

Status: planned 2026-11-27. Design source: [DESIGN.md](DESIGN.md) (proposal v0.1). Tasks: [tasks.md](tasks.md).

## Context
The orchestrator watches coding-agent terminals shared to this server, recognizes one pending question per session, asks Jev for a choice, and either recommends it or applies it through a host-validated conditional answer. It is **headless**: it runs in-process in the Windows tray app and in the console server, attaches to the `Hub` as its own passive participant, and needs no browser UI. Enrollment comes from `orchestrator.toml` plus tray controls. Recommendations and escalations appear as tray balloons, as rows in the tray window, and in an audit journal.

Scope: doc milestones M0–M2 (Observe, Recommend, Delegate) for Claude Code CLI menus. Host-side work (the extension client and the conditional answer gate) is tracked in synthet/warp as H-1 and H-2.

## Architecture
- **Passive attach.** Add `Hub::attach_observer`, `detach_observer` and `resync_observer`. Observers live in a separate `Session.observers` map and receive the same `JoinedSuccessfully`, replay and broadcasts as viewers. They have **no upstream path**, so they cannot write or resize, and they are not in `participant_list()` or `viewer_count` (that keeps Warp's single-uid viewer-driven sizing working). The dashboard gets `observed: bool`.
- **Core module.** `src/orchestrator/` consumes a `viewer::DownstreamMessage` stream. It decodes LZ4, runs a VT screen (vt100, confirmed by a spike) and an OSC 777 `warp://cli-agent` scanner, then the Claude Code recognizer, deterministic policy, evidence, a `Decider` (Jev stdio MCP client or stub), and the per-session decision state machine. It keeps a JSONL journal and an atomic `pending.json` under `<data_dir>/orchestrator/`.
- **Host extension channel.** The pinned `warpdotdev` protocol can't change, so add a separate WebSocket `/synth/v1/sessions/{id}/host`. It uses bearer and origin auth plus an `x-synth-reconnect-token` constant-time match, and allows one link per session. JSON `{"v":1,"type":…}` messages: `hello`, `state`, `observe.refresh`, `decision.opened`, `answer`, `answer.result`, `decision.closed`, `take_over`. `answer` carries no raw bytes. The host owns key encoding, generation (`host_generation` + `last_event_no`), and the final gate. A client without the extension never connects, so delegation stays unavailable (fails closed). The orchestrator reaches the link only through an in-process `HostExtLink`, never over HTTP.
- **Enrollment rule.** The tray's Enroll submenu lists the profiles in `orchestrator.toml`. Choosing one binds it to that live session id. A re-shared session (new id) has to be enrolled again; nothing matches on pwd, title or tab.
- **Decider.** `[decider.jev]` sets the command and args. The provider key comes from the environment only, and the child is spawned with `CREATE_NO_WINDOW`.
- **Startup.** `launch.rs` gets `--orchestrator-config` and `--data-dir`, and the tray passes its `data_dir()`. A missing config means no orchestrator task; an invalid one means an error is logged and the orchestrator stays off. Tray UI logic goes in a testable `orchestrator/ui.rs`, and `src/tray/app.rs` only renders it.
- **Prerequisite fix.** Derive `run_state: running|idle|unknown` from the share's scrollback (`did_execute`, `completed_ts`, `is_alt_screen_active`) next to `last_scrollback_pwd`, and have `web/app.js onJoined` use it. This fixes the mid-command idle bug. Also fix resume: accept the empty secret when the reconnect token matches, so enrollment survives a dropped socket.

## Out of scope
- M3 native requests (ACP, Codex app-server).
- The reviewer step. Jev decides directly behind the `Decider` trait, and a reviewer can implement the same contract later.
- A macOS tray. The console server runs on macOS, but macOS positive control is deferred.
- A WebSocket transport for the orchestrator. The core consumes a `viewer::DownstreamMessage` stream, so one can be added later.
- Changes to the pinned `session-sharing-protocol` rev.

## Acceptance criteria
- AC-1: WHEN `cargo build --all-targets` runs on Windows with the new deps (vt100, sha2, lz4_flex as a regular dep, tokio `process`/`io-util`) THEN it succeeds and clippy is clean.
- AC-2: WHEN a T-15 real-capture Claude Code menu fixture is fed to the screen THEN `ScreenSnapshot.rows` contains its option lines. The PR records the emulator crate choice and why.
- AC-3: WHEN the last scrollback block has `did_execute=true` and `completed_ts=null`, or alt-screen is active, THEN `run_state` is `running`. WHEN the block JSON is missing or unparseable THEN it is `unknown`.
- AC-4: WHEN the `--running` Playwright regressions in `web-tests/tests/regressions.spec.js` run with a pinned `@playwright/test` version (today it is the placeholder `"PLAYWRIGHT_VERSION"`) THEN they pass. `web/app.js` `onJoined`, which hard-codes `running = false` at line 387, takes the server's state from the `/api/sessions` `run_state` field.
- AC-5: WHEN an observer attaches THEN it receives `JoinedSuccessfully` plus every event, AND the sharer receives nothing, AND `participant_list()` and `viewer_count` are unchanged.
- AC-6: WHEN the session ends THEN the observer gets `SessionEnded` and its sender is dropped. WHEN `resync_observer(after=N)` is called THEN only events after N are replayed.
- AC-7: WHEN each v1 golden fixture under `tests/fixtures/synth_ext/v1/` is parsed and re-serialized THEN the output is byte-identical. WHEN `v` or `type` is unknown THEN parsing fails without a panic. `answer` has no raw-bytes field.
- AC-8: WHEN `orchestrator.toml` is absent THEN the server behaves as today. WHEN it is invalid THEN an error is logged and the orchestrator is off. WHEN it changes THEN `policy_revision` changes.
- AC-9: WHEN a session is shared THEN it is not enrolled until `enroll(session_id, profile)` is called.
- AC-10: WHEN the host-extension upgrade has a wrong or missing reconnect token, a foreign Origin, or no bearer token THEN it is refused. WHEN the session is reaped THEN the extension socket closes.
- AC-11: WHEN a resume carries the client's empty secret and the correct reconnect token THEN it succeeds. WHEN the token is wrong, or a non-empty secret is wrong, THEN it fails.
- AC-12: WHEN a viewer's `WriteToPty` is relayed on a session with an extension link THEN subscribers get a remote-input notice.
- AC-13: WHEN bytes are LZ4-framed THEN the screen matches the expected grid. WHEN the frames are malformed THEN `RunState::unknown` with no panic.
- AC-14: WHEN events are replayed THEN no detection runs. WHEN an event gap appears THEN a resync is requested and the session is ineligible for automatic answers. An OSC 777 split across events yields one signal.
- AC-15: WHEN captured permission and AskUserQuestion menus are scanned THEN their options and order are found. WHEN a log prints "1/2/3" or old transcript text contains yes/no THEN nothing is recognized.
- AC-16: WHEN 20 cosmetic redraws occur THEN there is one prompt_id and one generation. WHEN option text or order changes THEN the generation increments.
- AC-17: WHEN the grant is allow-once THEN the "don't ask again" option is ineligible. WHEN a permission lacks operation details THEN it escalates.
- AC-18: WHEN pending records are written THEN the write is atomic (temp file then rename). WHEN the orchestrator restarts THEN dispatched-but-unconfirmed records become `uncertain` and are never replayed.
- AC-19: WHEN an evidence path escapes its root through a symlink or junction THEN it is rejected. WHEN excerpts are not opted in THEN the packet holds metadata only. The token never appears in a packet.
- AC-20: WHEN the decider returns `ask_user`, an option that wasn't offered, `ok:false`, or times out THEN the outcome is Escalate with a reason. WHEN the doc §6 fixture is sent to the stub THEN `read_old_save_new` is accepted.
- AC-21: WHEN sessions A and B show the same menu and A's decider is slow THEN B's recommendation lands first and each stays with its own session.
- AC-22: WHEN the mode is Observe or Recommend THEN the session task is never given a `HostExtLink`, so no write path exists by construction (checked by its type or construction). WHEN it is Observe THEN the decider is never called. Every recommendation or escalation writes a journal row.
- AC-23: WHEN `ui.rs` gets model state THEN its rows show host, pwd, mode, status, question and age. The Enroll submenu lists the profiles from `orchestrator.toml` and binds the chosen profile to that session id only, never by pwd, title or tab. A Take over action reaches `OrchestratorHandle`. A recommendation or escalation produces balloon text.
- AC-24: WHEN the tray runs on Windows (manual check) THEN the session list, the Enroll toggle and balloons work with no console.
- AC-25: Against the simulated host: WHEN a human answers first THEN 0 extra keys are written. WHEN the prompt changed or the process was replaced THEN no stale answer is applied.
- AC-26: WHEN the socket drops before or after dispatch THEN the attempt is recovered by attempt_id or marked uncertain, and never sent twice. WHEN a grant is revoked or an excerpt changes mid-review THEN the decision is rejected or re-evaluated.
- AC-27: WHEN a real Claude Code menu is answered on Windows THEN the intended option is chosen and linked to its confirmation, including share-before-launch and attach-mid-command. WHEN the adapter version is not in `delegate_versions` THEN delegation is refused.
- AC-28: WHEN `examples/record_session.rs` joins a real Synth Term share (Windows) during a Claude Code permission menu and an AskUserQuestion menu THEN their downstream JSONL streams, both from share-before-launch and from attach-mid-command, are saved under `tests/fixtures/claude_code/` with the Claude Code version.
- AC-29: WHEN the web app sends `WriteToPty` to that real running menu THEN the intended key reaches it. The result is recorded in a short report, replacing the "not verified" entry in the 2026-11-05 report.
- AC-30: WHEN `take_over` comes from the tray, the remote side or the host THEN automatic writes for that session stop and outstanding decisions are cancelled and journaled, AND delegation resumes only after a fresh `hello` and observation.
- AC-31: WHEN the Jev decider starts THEN its command and args come from `[decider.jev]` in `orchestrator.toml`, the provider API key is inherited from the environment and never stored in the toml, the journal or an evidence packet, and the child process starts with `CREATE_NO_WINDOW`, so the tray opens no console.
- AC-32: WHEN a change touches auth, hub roles or the remote-control surface THEN the PR records a review with `.claude/skills/threat-modeling-agentic-tools`.

## Risks and open questions
- Claude Code key behavior (whether a digit selects and submits) and menu stability across versions have to be confirmed from fixtures captured from the installed version (T-8, T-14).
- A mid-command attach may give a partial frame until the next full redraw, so a session stays ineligible until the frame has been stable for a quiet period.
- Check the native-windows-gui list-view and balloon APIs and the vt100 API against their docs during T-1 and T-12.
- macOS positive control is deferred (the tray is Windows-only).
