# Headless decision orchestrator

Spec set for an in-process orchestrator that watches Synth Term terminals shared to this server, recognizes pending Claude Code questions, asks Jev, and recommends or (through a host-validated conditional answer) applies a choice. Baseline: `main` 58e796c (2026-11-27).

| File | Content |
|---|---|
| [DESIGN.md](DESIGN.md) | The owner's design proposal v0.1 (source; private link removed) |
| [spec.md](spec.md) | Direction, baseline table, backlog, corrections the code forced, architecture, milestones, spec-level ACs, verification matrix, open decisions, risks |
| [items/](items/) | One spec per backlog item, ORC-01 to ORC-15 |
| [tasks.md](tasks.md) | AC traceability and one task per PR (`ORC-nn.n`) grouped by milestone |
| [tasks-import.md](tasks-import.md) | Generated table for agent-flow's importer (`T-n`, global `AC-n`); do not edit by hand |

## Items by milestone
- **M0 Real transport**: [ORC-01](items/ORC-01.md), [ORC-02](items/ORC-02.md), [ORC-03](items/ORC-03.md), [ORC-04](items/ORC-04.md)
- **M1 Recommend**: [ORC-05](items/ORC-05.md), [ORC-06](items/ORC-06.md), [ORC-07](items/ORC-07.md), [ORC-08](items/ORC-08.md), [ORC-09](items/ORC-09.md), [ORC-10](items/ORC-10.md), [ORC-11](items/ORC-11.md)
- **M2 Delegate**: [ORC-12](items/ORC-12.md), [ORC-13](items/ORC-13.md), [ORC-14](items/ORC-14.md), [ORC-15](items/ORC-15.md)

## Overlap map
| Existing work | Where | Relation |
|---|---|---|
| Epic `warpr-oh1rg`, tickets T-1..T-16 | this repo's agent-flow | Replaced by this re-key; cancel after the new import is confirmed (mapping in spec.md backlog) |
| `warp-i9ils` host extension client | synthet/warp agent-flow | Covers ORC-12.3; body needs updating to ORC IDs and to own the PTY write counter |
| `warp-m0pj2` host answer gate | synthet/warp agent-flow | Covers ORC-13.1; body lacks ORC-13/AC-6, AC-7, AC-10, AC-11, AC-14 to AC-16 |
| `warp-htojy` gate session sharing on a configured server | synthet/warp agent-flow (LOCAL-FIRST-CLEANUP T-4) | Also edits `self_hosted.rs`; sequence with warp-i9ils |
| INT-02.1 re-verify self-hosted remote control | warp branch `docs/synth-term-roadmap`, `specs/SYNTH-TERM-ROADMAP/items/INT-02.md` | ORC-03 does the mid-command input check against this server; INT-02.1 cites its report instead of repeating it |
| AGT-07 orchestration bridge | same roadmap, `items/AGT-07.md` | Different layer (in-client local task queue); not duplicated |
| SEC-01.3 workspace trust, SEC-01.4 fail-closed approval evaluator | same roadmap, `items/SEC-01.md` | Same principles, client side; ORC-08 shares no code |
| Mid-command idle bug, Playwright plan | `docs/reports/remote-control-verification-2026-11-05.md`, `web-tests/` | Fixed by ORC-01 |

## Conventions
- Paths are `path` + symbol at the baseline commit. Proposed files are marked "(new)". Paths under `app/` and `crates/` are synthet/warp paths (verified at warp `master` 0e6fd3ac); roadmap files exist only on warp branch `docs/synth-term-roadmap`.
- Item ACs are `<item>/AC-n`, one line each, EARS with one "shall". Spec-level ACs are `SPEC/AC-n`.
- Recommendations are marked "owner to confirm". Versions and APIs of new crates are "verify at pinned version".
