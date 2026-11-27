# Synth Term Remote decision orchestrator

**Design proposal v0.1 · November 27, 2026**  
**Primary targets:** Synth Term on Windows and macOS, the self-hosted Remote server, and Jev MCP.

## 1. Outcome

A developer starts several coding agents in Synth Term, attaches a task brief and selected files to each session, and delegates a class of decisions. When an agent blocks on a supported question, the orchestrator identifies the choices, gathers relevant evidence, optionally asks a reviewer to compare the alternatives, and asks Jev for a final choice. The originating host applies the choice if the same question is still pending. The developer can watch, answer, or take over from the desktop or phone.

The unit of work is **one pending question in one session**. A terminal can be running yet waiting for input. Silence, a list of options in old output, and a dashboard “running” badge are each insufficient evidence. Success means resolving the intended question and observing what happened afterward. A successful socket write alone establishes delivery to the transport.

### Operating modes

| Mode | Behavior |
|---|---|
| Observe | Detect and display questions. Replay recorded evidence to evaluate detectors without calling a provider or sending terminal input. |
| Recommend | Produce a recommendation with evidence and trade-offs. The developer applies an answer or takes over. |
| Delegate | Automatically apply eligible choices within the session’s configured scope. Escalate the remaining decisions to the developer. |

Mode, workspace, file access, eligible decision classes, and provider access are explicit per enrollment. Starting a terminal or sharing a pane does not enroll it.

## 2. Implementation baseline

| Component | Evidence in the current repositories | Design implication |
|---|---|---|
| Session discovery and observation | Remote implements `GET /api/sessions` and a viewer WebSocket at `/sessions/join/{session_id}`. Initial join and rejoin have different replay behavior. The web viewer decodes LZ4-framed PTY output and consumes ordered events and authoritative resize events. | Implement a passive headless viewer. Reuse the server and wire protocol. A watcher must not change terminal size or claim interactive focus. [Server](https://github.com/synthet/synth-term-remote/blob/main/src/server.rs) · [Protocol notes](https://github.com/synthet/synth-term-remote/blob/main/specs/self-hosted-remote-control/TECH.md) |
| Input | Remote forwards `WriteToPty` for a running program and supports a separate `ExecuteCommand` path for an idle shell. The existing viewer role can carry broad write authority. | Add a narrower, conditional answer capability. Never translate a failed answer into a shell command. [Viewer](https://github.com/synthet/synth-term-remote/blob/main/web/app.js) · [Hub](https://github.com/synthet/synth-term-remote/blob/main/src/hub.rs) |
| Real-client verification | The Windows verification report observed live output. Mid-command sharing could report idle, and browser-to-terminal input was left unverified. The README still lists the mid-command issue. | Exercise real input and late attachment before enabling delegation. Treat unknown host state as ineligible for automatic input. [Report](https://github.com/synthet/synth-term-remote/blob/main/docs/reports/remote-control-verification-2026-11-05.md) · [README](https://github.com/synthet/synth-term-remote/blob/main/README.md) |
| Existing orchestration | Synth Agent Remote describes a local bridge with a session registry, approval queue, event bus, audit log, and adapters for Cursor/VS Code. | Start with a terminal adapter and decision worker beside that bridge. Keep PTY ownership in Synth Term. Reuse queue and audit code where their contracts fit. [Bridge architecture](https://github.com/synthet/synth-agent-remote/blob/main/docs/architecture.md) |
| Decision engine | Jev MCP exposes `jev_choice(state, instructions, criteria)` and typed answers. Confidence can be unavailable. | Use named choices, including an explicit abstention. A general-purpose reviewer can supply evidence or implement the same choice contract when explicitly configured. [Jev MCP API](https://github.com/synthet/jev-mcp/blob/master/docs/api.md) |

In the existing Synth Term roadmap, the worker fits the orchestration bridge (AGT-07), the host and relay changes fit Remote (INT-02), and enrollment and approval scope fit workspace trust (SEC-01). The self-hosted Remote implementation above is the starting point.

## 3. Components and ownership

| Component | Owns |
|---|---|
| Synth Term host | Live terminal, foreground command instance, rendered screen, human input, and the last check before applying a terminal answer. Assigns a new execution identity when the process or share is replaced. |
| Remote server | Authenticated discovery, subscriptions, replay, and relay of bounded answer requests to an enrolled host. The worker receives a scoped credential. Existing human viewers keep their normal input path. |
| Orchestrator | Enrollment, per-session pending decisions, deadlines, review jobs, deterministic policy evaluation, and an audit journal. It can run alongside Remote or connect to multiple trusted hosts. |
| Adapter | Recognition and answer semantics for one native protocol or one CLI menu family. Declares its capabilities and supported versions. The host-side part of a PTY adapter owns key encoding. |
| Context resolver and reviewer | Read authorized files on their owning host and compare the options against the task. A reviewer returns evidence and trade-offs. It has no terminal credential or ability to answer a session. |
| Jev | Selects an eligible option or `ask_user` using a bounded evidence packet. The orchestrator validates the answer. |
| Desktop and Remote dashboard | Show the pending question, recommendation, evidence, and final outcome. Taking over a session invalidates any outstanding automatic decision for it. |

Use a native request ID when the orchestrator owns the agent connection and the harness exposes a structured question. A manually started CLI in an existing pane can use the terminal path independently. Deduplicate across paths by choosing one authoritative adapter per prompt. A rendered copy of a native request must never spawn a second decision. ACP covers tool permissions and Codex app-server exposes structured user-input and approval requests. Negotiate the installed harness’s capabilities. [ACP tool calls](https://agentclientprotocol.com/protocol/tool-calls) · [Codex app-server](https://developers.openai.com/codex/app-server/)

## 4. Decision lifecycle

1. **Enroll.** Bind the session to a host, execution instance, workspace root, task, attachments, delegation policy, and provider profile. Multiple sessions in the same repository may have different briefs or grants.
2. **Observe.** Reconstruct live state from a snapshot plus ordered events. Suppress detection during replay. If the initial state or an event range is missing, request a fresh snapshot and suspend automatic answers for that session.
3. **Recognize.** A native request supplies its question and options. A terminal adapter must establish an active supported menu using its layout, screen mode, foreground program, and input behavior. Track prompt identity across cosmetic redraws. A materially changed question, option, or input target creates a new generation. Generic yes/no detection may suggest a candidate for human inspection.
4. **Evaluate scope.** Separate project choices from tool permissions. Check the actual requested operation, target, and duration against the user’s grants. An “Allow once” label without verifiable operation details is insufficient for an automatic permission grant. Remove ineligible options before review and judgment.
5. **Gather and decide.** Assemble evidence, optionally invoke a reviewer, and call the configured decider. A clear direct choice can skip the reviewer. Missing facts, contradictory evidence, exhaustion of budget, and invalid provider responses lead to a human decision. Review jobs are isolated from the monitored session registry to prevent recursive review.
6. **Apply conditionally.** At the host or native request owner, validate the live request, policy, option, execution identity, and generation. Serialize the operation with local and remote human input. Give manual input priority while a verdict is pending.
7. **Confirm and journal.** Record whether the host rejected, submitted, or confirmed the answer. Confirmation requires an adapter-defined signal correlated with this request. A generic screen refresh or loss of the socket is insufficient. If the result is ambiguous, show “delivery uncertain” and re-observe.

One pending automatic write per session, with a bounded pool of independent review jobs across sessions. An unanswered question in session A cannot hold session B’s input lock. Persist pending records before dispatch. After a worker restart, reconcile each record with the live host before resuming observation.

## 5. Proposed host and worker contracts

These names describe new logical operations. The HTTP and WebSocket routes in §2 are the existing transport.

| Operation | Required behavior |
|---|---|
| `observe(session)` | Supplies host/execution identity, stream epoch and sequence, negotiated capabilities, and authoritative running state or `unknown`. A fresh snapshot anchors resumed observation. |
| `decision.opened` | Carries the pending question, stable option IDs and labels, decision kind, request reference, generation, evidence revision, and expiration. A native adapter may use the harness request ID. A PTY adapter mints a host-bound prompt ID. |
| `answer(request)` | Carries the exact decision and option, expected generation and policy revision, and a unique attempt ID. Only the owning adapter may translate a validated choice to a native response or terminal input. |
| `decision.updated` / `decision.closed` | Announce material changes, cancellation, manual handling, submission, confirmation, or expiry. Replayed events carry their original identity. |
| `take_over(session)` | Suspends automatic writes for that session and cancels outstanding decisions. Resuming delegation requires a fresh observation. |

Illustrative conditional answer:

```json
{
  "session_id": "s17",
  "host_id": "workstation",
  "execution_id": "run42",
  "stream_epoch": "e3",
  "prompt_id": "p8",
  "expected_generation": 4,
  "expected_policy_revision": 2,
  "option_id": "read_old_save_new",
  "attempt_id": "a91"
}
```

**Host response semantics.** Reject a stale generation, wrong execution, revoked capability, expired request, or invalid option without writing. Reuse of an attempt ID with identical content returns its recorded status. Conflicting reuse is an error. For an accepted PTY attempt, record write intent before dispatch. If the host crashes between intent and receipt, preserve uncertainty and prohibit automatic replay across that boundary. A PTY cannot provide an exactly-once transaction spanning a process crash.

**Key mapping.** The adapter owns mappings such as literal `y`, `2` plus Enter, or navigation in a highlighted menu. Some menus submit on the first key. For multi-step navigation, observe the selected option after moving and revalidate before the final Enter. Only enable automated submission where the supported adapter can distinguish a live menu and foreground command from their disappearance. Residual races in third-party TUIs make a native request preferable for sensitive actions.

## 6. Evidence, review, and Jev

Enrollment declares explicit file attachments or read-only workspace roots, the permitted reviewers/providers, and a size and time budget. Resolve paths on the session’s host, check the resolved target against the grant, and reject symlink or junction escapes. Attach bounded excerpts with relative path, line range, content digest, and workspace revision when available. An uncommitted file needs its own digest even if a Git commit is known. Recheck decision-relevant evidence before applying a verdict if it could have changed during review.

The evidence packet contains the task goal, current question, eligible options, relevant recent terminal context, and authorized excerpts. Files and terminal text supply evidence, including when they contain imperative language. Only enrollment and the host’s policy can grant authority. Local read access and permission to send an excerpt to an external provider are separate settings. Record which provider received which evidence references. Keep credentials and unrelated session transcripts out of the packet.

A reviewer returns a row for each option with its main benefit, drawback, evidence references, and missing facts. The orchestrator can use **Jev directly**, **reviewer → Jev**, or an explicitly selected **reviewer as decider**. The final contract is the same in each mode. A timeout does not silently switch provider.

**Worked example.** The agent asks when to migrate settings. The attached compatibility spec requires old settings to work on startup and prohibits writes until the user saves. The reviewer finds that the loader can already read both versions.

| Candidate | Benefit | Drawback | Evidence-based assessment |
|---|---|---|---|
| Migrate on startup | Immediate conversion to one format | Writes before the user saves | Conflicts with the startup requirement |
| Read old settings and convert on save | Preserves startup behavior and honors the write constraint | The reader must support two formats during transition | Satisfies both requirements |
| Require manual import | Simple normal startup path | Existing settings disappear until import | Conflicts with compatibility requirement |

Illustrative arguments to the existing `jev_choice` tool. The excerpt IDs below stand for the actual content supplied by the resolver.

```json
{
  "question_id": "migration_choice",
  "instructions": "Select the offered option supported by the task and evidence. Choose ask_user if evidence is insufficient or contradictory.",
  "state": {
    "goal": "Preserve settings during upgrade",
    "evidence": [
      {"id": "E1", "text": "Read old settings at startup. Write the new format only on save."},
      {"id": "E2", "text": "The loader already accepts both schema versions."}
    ]
  },
  "criteria": {
    "migrate_on_start": "Convert settings when the app opens",
    "read_old_save_new": "Read either format and convert on the next save",
    "manual_import": "Require an explicit import",
    "ask_user": "The supplied evidence cannot resolve this choice"
  }
}
```

For this fixture the expected choice is `read_old_save_new`. Validate the returned choice against the eligible set and the live prompt. `ask_user` is an orchestrator outcome and sends no keys. Confidence is optional telemetry. It does not expand delegated authority.

## 7. User controls and failure behavior

| Situation | Required behavior |
|---|---|
| Several agents ask questions at once | Show host, workspace, task, question, age, mode, and status per row. A reviewer finishing out of order still answers only its originating session. |
| Human answers from desktop or phone | Invalidate the in-flight verdict as soon as the host accepts human interaction. If automated submission has already started, report its state before permitting a second submission. |
| New persistent permission or unmatched operation | Surface the original operation and grant duration to the user. Eligibility for a one-time grant cannot make an “always allow” option eligible. |
| Prompt changes, process exits, or session is re-shared under a new ID | Cancel the old pending decision. Display a newly recognized question separately. Never recover a write target by title, working directory, or tab position. |
| Reviewer or Jev unavailable | Release the review slot, retain the live question for manual handling, and record the reason. A human can act immediately without waiting for a retry. |
| Unsupported CLI, password prompt, free-form answer, or unknown state | Keep interactive control available. The initial delegated feature covers recognized, single-choice questions with enumerable answers. |
| Missed acknowledgment after submission | Read the attempt status and inspect the current prompt. Mark uncertain if the adapter cannot establish the outcome. Never repeat a raw keystroke to discover whether the first one landed. |

The audit record includes session and execution IDs, prompt and option IDs, eligibility decision, mode, evidence references and digests, reviewer/provider version, selected answer, attempt status, and any manual takeover. Default to bounded metadata and a short rationale. Make transcript and excerpt retention configurable separately.

## 8. Delivery plan and acceptance tests

| Milestone | Deliverable | Exit condition |
|---|---|---|
| M0 · Real transport | Passive observer and real-client input test on an enrolled disposable session | On Windows and macOS, demonstrate live screen reconstruction, correct input to a running menu, and proper handling of a mid-command join. Unknown state disables automatic input. |
| M1 · Recommendation | Per-session queue, explicit attachments, one CLI recognizer, reviewer and Jev adapter, dashboard review | Two concurrent sessions produce correctly attributed recommendations. The worker has no write capability in recommend mode. |
| M2 · Delegated terminal answers | Host-side conditional answers, scoped credentials, local and remote takeover, audit and recovery | Pass all terminal race, duplicate-delivery, isolation, and recovery cases below on the supported CLI/version. |
| M3 · Native requests | One native harness with typed request IDs and cancellation | The same decision pipeline resolves a structured request, handles human resolution and cancellation, and emits no PTY input for that request. |

Run deterministic tests using the Remote fake sharer, a scripted reviewer, and a Jev contract stub. Use captured menus with known expected choices for offline evaluation. Keep real-provider evaluation separate from transport and host acceptance tests. [Fake sharer](https://github.com/synthet/synth-term-remote/blob/main/examples/fake_sharer.rs)

| Test | Pass condition |
|---|---|
| Simultaneous prompts | A and B display identical menus with different task briefs. A slow review in A cannot block B. Each answer reaches the correct process and follows its own evidence. |
| False prompt and redraw | A log prints “1/2/3” and a historical transcript contains “yes/no.” Neither produces a write. Cosmetic redraws of a real menu produce at most one pending decision. |
| Human wins race | Delay Jev, answer on the phone, then release the verdict. The process receives zero additional keys. Repeat with desktop input and manual takeover. |
| Changed target | Change option order or meaning, replace the process, or return to the shell between review and submission. No stale answer is applied. A staged arrow-key sequence cannot deliver Enter to a different prompt. |
| Reconnect and crash | Drop the socket before dispatch, after dispatch, and before receipt is journaled. Recover by attempt ID or mark uncertain. No case produces duplicate input. |
| Context and policy | Remove a file grant, change a decisive excerpt, or revoke delegation during review. Reject or re-evaluate the stale decision. The provider never receives an ungranted file. |
| Permission scope | Offer “allow once,” “always allow,” and “deny.” With a one-time grant, the persistent option is ineligible even if the reviewer proposes it. With insufficient operation metadata, escalate. |
| Inconclusive judgment | Simulate `ask_user`, invalid option, provider timeout, and contradictory files. Each case yields a visible pending human decision and zero keystrokes. |
| Positive control | For a real known menu, choose the intended option, capture its specific acknowledgment or expected next state, and link that observation to the attempt. Also test share-before-launch and attach-mid-command. |

**Release criterion.** Enable delegation per adapter and version only after its real-host positive control and stale-input tests pass. Track detection precision, missed questions, abstentions, latency, manual overrides, and uncertain deliveries by adapter version. The first supported slice is two sessions, one terminal menu family, one reviewer, and Jev Choice. Additional terminal adapters and native harnesses can adopt the same decision contract independently.
