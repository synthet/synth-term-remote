# Headless decision orchestrator: tasks

Spec: [spec.md](spec.md). Each task is one PR. Unit tests go in sibling `*_tests.rs` files.

| ID | Task | Owner role | Files | Depends on | ACs | Verification |
|---|---|---|---|---|---|---|
| T-1 | Orchestrator scaffold and dependencies | implementer | Cargo.toml, src/lib.rs, src/orchestrator/mod.rs | – | AC-1 | cargo nextest run |
| T-2 | Running state from share scrollback (mid-command idle fix) | implementer | src/hub.rs, src/hub_tests.rs, web/app.js, web-tests/tests/regressions.spec.js | – | AC-3, AC-4, AC-32 | cargo nextest run; npm test in web-tests |
| T-3 | Passive observer attach in Hub | implementer (security) | src/hub.rs, src/hub_tests.rs | T-2 | AC-5, AC-6, AC-32 | cargo nextest run hub |
| T-4 | Host extension wire schema v1 | implementer | src/synth_ext.rs, src/synth_ext_tests.rs, tests/fixtures/synth_ext/v1/ | – | AC-7 | cargo nextest run synth_ext |
| T-5 | Config, enrollment, launch and tray startup wiring | implementer | src/orchestrator/config.rs, src/orchestrator/enroll.rs, src/launch.rs, src/tray/app.rs, config/orchestrator.example.toml | T-1 | AC-8, AC-9 | cargo nextest run orchestrator |
| T-6 | Host extension route and hub extension state | implementer (security) | src/server.rs, src/hub_host_ext.rs, src/hub.rs, examples/fake_sharer.rs, tests/websocket_e2e.rs | T-3, T-4 | AC-10, AC-12, AC-32 | cargo nextest run websocket_e2e |
| T-7 | Observer, VT screen (emulator crate choice), OSC 777 scanner | implementer | src/orchestrator/observer.rs, src/orchestrator/screen.rs, src/orchestrator/osc.rs | T-1, T-3, T-15 | AC-2, AC-13, AC-14 | cargo nextest run orchestrator |
| T-8 | Claude Code recognizer and prompt identity tracker | implementer | src/orchestrator/recognize/mod.rs, src/orchestrator/recognize/claude_code.rs | T-1, T-15 | AC-15, AC-16 | cargo nextest run recognize |
| T-9 | Deterministic policy, audit journal, pending store | implementer | src/orchestrator/policy.rs, src/orchestrator/journal.rs | T-5 | AC-17, AC-18 | cargo nextest run orchestrator |
| T-10 | Evidence builder and Jev decider with contract stub | implementer (security) | src/orchestrator/evidence.rs, src/orchestrator/jev.rs, examples/jev_stub.rs | T-5 | AC-19, AC-20, AC-31, AC-32 | cargo nextest run orchestrator |
| T-11 | Decision pipeline: Observe and Recommend (M1) | implementer | src/orchestrator/decision.rs, src/orchestrator/mod.rs, tests/orchestrator_e2e.rs | T-7, T-8, T-9, T-10 | AC-21, AC-22 | cargo nextest run orchestrator_e2e |
| T-12 | Tray UX: session list, Enroll profile submenu, Take over, balloons | implementer | src/orchestrator/ui.rs, src/orchestrator/ui_tests.rs, src/tray/app.rs, Cargo.toml | T-11 | AC-23, AC-24 | cargo nextest run ui; manual tray check |
| T-13 | Delegate mode (M2) with take-over and recovery, against simulated host | implementer (security) | src/orchestrator/decision.rs, src/orchestrator/journal.rs, examples/fake_sharer.rs, tests/orchestrator_delegate_e2e.rs | T-6, T-11, T-16 | AC-25, AC-26, AC-30, AC-32 | cargo nextest run orchestrator_delegate_e2e |
| T-14 | Real-host positive control and release gate | verifier | docs/reports/decision-orchestrator-verification-<date>.md, config/orchestrator.example.toml | T-12, T-13 | AC-27 | manual run with Synth Term (needs warp H-2) |
| T-15 | M0 real transport: capture Claude Code menu streams and verify real input | verifier (human, Windows) | examples/record_session.rs, tests/fixtures/claude_code/, docs/reports/remote-control-input-verification-<date>.md | T-2 | AC-28, AC-29 | cargo run --example record_session against a live Synth Term share |
| T-16 | Resume accepts the client's empty secret when the reconnect token matches | implementer (security) | src/hub.rs, src/hub_tests.rs, tests/websocket_e2e.rs | T-2 | AC-11, AC-32 | cargo nextest run hub websocket_e2e |

Every ticket body also carries the Pattern file. T-2 follows `last_scrollback_pwd`, T-3 `join_session`/`replay_events`, T-5 `Args::configured_token`/`load_or_create_token`, T-6 `run_sharer`/`spawn_writer`, T-9 `LogTail`, T-10 `examples/fake_sharer.rs`, T-11 `tests/websocket_e2e.rs`, and T-12 `tray_support_tests.rs`. Security tickets (T-2 lightly, T-3, T-6, T-10, T-13) also need the `.claude/skills/threat-modeling-agentic-tools` review as a criterion. Each ticket's DoD: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo nextest run --no-fail-fast`.

Parallel waves: {T-1, T-2, T-4} → {T-3, T-5, T-15, T-16} → {T-6, T-7, T-8, T-9, T-10} → T-11 → {T-12, T-13} → T-14. T-2, T-3 and T-16 each depend on the one before, because they all edit `hub.rs`. T-7 and T-8 must use T-15's real captures, never hand-written menus. T-1 creates `src/orchestrator/mod.rs` with the `ScreenSnapshot` and `RunState` stubs.

Host-side tickets live in synthet/warp's tracker. H-1 is the extension client and state publisher; H-2 is the conditional answer gate, manual priority and take-over. T-14 cannot finish until H-2 is done.
