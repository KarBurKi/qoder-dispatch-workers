---
name: dispatch-workers
description: Orchestrate independent Qoder conversations as full-tool workers - spawn a session for a task, hand a task to an already-open conversation, let it read files and run commands on its own, supervise its turn, collect and verify its output, then follow up or abort. Use when the user asks to dispatch/parallelize work to separate agents or conversations, run several investigations or refactors concurrently, build a multi-session pipeline or cast, relay a task to a specific existing session, or resume a previously spawned worker.
---

# Dispatch Workers

Version 0.1.1 — adds the Prerequisites note and Relay mode.

## Prerequisites (read first: failure here is silent)

This skill is a document, not an engine. The eight orchestration primitives below are supplied by the
**client product form**, not by the skill and not by any installable package.

Verified working: the standalone **Qoder Desktop App** (Electron client). This machine's fingerprints:
`productVersion 0.4.11`, `buildTime 2026-10-23T04:33:29Z`, `Electron 43.1.1`, `appId com.qoder.app`;
its `resources/product.json` has 12 keys and **none of the VSCode baseline fields**.

Verified NOT working: **Qoder IDE** (the VSCode-derived line) — `1.21.2`, VSCode base `1.106.3`,
built 2026-08-02, Windows; a separate user measured all eight primitives missing there. The two
product lines do not share a version numbering scheme, so there is no "upgrade to X and it works"
answer. **Do not debug this as a version problem.**

Self-check, in order of reliability:

1. Hardest test: look for `create_chat_session` and friends in the tool list of your conversation. The
   client exposes them through its built-in `builtin` MCP server (this machine lists 10: the eight
   below plus `fork_chat_session` and `create_team_task`). If your MCP directory only holds things like
   `browser-use` / `genui` / `schedule`, nothing you can install will add them — the client simply
   does not provide them.
2. If you want to scan the package, scan `resources/app.asar`, **not `Qoder.exe`**. On this fully
   working machine `Qoder.exe` returns **zero hits** for all eight names, while the same machine's
   `app.asar` (180 MB) hits 5 of them (`create`=3, `list`=3, `send`=3, `read`=5, `wait`=3). The
   remaining three (`respond_`, `stop_`, `list_agents`) are nowhere on this disk — they are registered
   at runtime. So "zero string hits across the install" does **not** prove the capability is absent;
   that inference returns a false negative even on a machine where the skill runs.

Passing structural validation, appearing in `/skills list`, or invoke returning this document all mean
nothing about callability. If the primitives are absent, neither spawn nor relay can execute: treat
this skill as methodology only, and fall back to disk-mediated handoff (conversations read and write
agreed files in a shared directory).

## Overview

Turn the current conversation into a **director**: it never does the heavy work itself, it drives
independent Qoder conversations that carry the full toolset (Read / Write / Edit / Bash / Grep /
MCP), assigns each a self-contained task, waits for their turns, and verifies what landed on disk.
A worker is either a session it opened for the job (spawn) or a conversation already running that it
hands the job to (relay).

The director holds the plan, the roster, and the verdicts. Workers hold the execution. No scripting
language, no plugin process — every primitive is already a built-in MCP tool.

## Tool contract

| Role | Qoder tool | Notes |
| --- | --- | --- |
| Open a worker | `create_chat_session` | Pass `callbackMode: "supervise"` unless you truly want fire-and-forget. |
| Discover cast | `list_chat_sessions` | Returns ids + state, not transcripts. Use before read/wait/message. |
| Send next cue | `send_message_to_chat_session` | Also takes `callbackMode: "supervise"`. Never target yourself. This is also the whole of Relay mode. |
| Collect output | `read_chat_session` | The only way to get reply text; `turnLimit`, `includeToolOutputs`. |
| Status / barrier | `wait_chat_sessions` | ≤8 targets, `timeoutMs` ≤ 60000; returns when **any** target settles. |
| Answer a gate card | `respond_chat_session_interaction` | Needs the `interactionRef` from a supervise callback. `allow`/`deny` or `indexedAnswers`. |
| Abort | `stop_chat_session` | Discard a worker whose output is wrong. |
| Persona selection | `list_agents` | Only `ready` agents are selectable; on a bare install just `agent:default` exists. |

## Two dispatch modes: spawn vs relay

The kernel is the same; only the origin of the worker differs.

**Spawn (default)** — `create_chat_session` opens a blank conversation and makes it the worker. Use it
to fan out a batch of unrelated tasks: cheap, and the boundaries are clean because the new session
holds nothing but the prompt you hand it.

**Relay** — no new session. `send_message_to_chat_session` delivers the task to a conversation that is
**already open**, usually one doing other work or one you kept alive. Use it to *pull a conclusion back*
rather than to hire another pair of hands. The message necessarily enters the target's existing history,
so a relay prompt must carry three things a spawn prompt can get away without:

1. **A declared messenger header** — open by stating that you are another conversation (the director),
   that this message comes from an agent and not from the human. Measured: the target honoured it — it
   first answered the director's recall question (naming the file and the string its previous task had
   written) and then took the new task.
2. **A self-contained task** — absolute paths, deliverable shape, reply budget, all stated; the target
   does not know the director's context and will not know it later.
3. **A reply budget** — "answer in ≤5 lines: path, key figures, command", otherwise the callback body is large.

Register relay workers in the roster with `dispatchMode: "relay"` and `preexisting: true`. That flag is
what tells a later turn that the director did not create this session and therefore must not
`stop_chat_session` it on a hunch.

Honest split between what was measured and what was not:

- **Measured**: sending to an `idle` conversation drives a full turn with no human message in between;
  the artifact lands on disk; the director can read the result back. Storage semantics are confirmed —
  the relay message is stored in the target as `role: "user"` with `turnId` prefixed `agent-message:`,
  i.e. **it occupies the target's user slot**.
- **Not measured**: the target refusing because of a role conflict (it is mid-task for the human, or its
  persona contradicts the assignment). Entirely plausible; we never produced that scenario, so it stays
  a known unknown rather than a claim.
- **Structural limit**: since it lands in the `user` slot, the target cannot distinguish an agent
  handoff from the human speaking by role alone — it has only the declared header. Relaying into a
  conversation you do not control is effectively writing into its user input. Don't.

## Lifecycle

1. **Plan the cast.** Decide worker roles, one bounded deliverable each. Write
   `assets/roster.template.json` shape into `roster.json` in the project directory before spawning;
   record `name`, `role`, `sessionId`, `dispatchMode`, `task`, `expectedArtifact`, `status`.
2. **Dispatch.** Spawn: one `create_chat_session` per worker. Relay: locate the existing session with
   `list_chat_sessions`, then `send_message_to_chat_session`. Either way the prompt follows the rules
   below. Record the returned `sessionId` into `roster.json` immediately — it is only a dispatch
   receipt, not a result.
3. **Supervise.** Do not poll in a loop. Supervise mode delivers callbacks for permission requests,
   user questions, and turn completion. Use `wait_chat_sessions` only as a barrier or a one-shot
   `timeoutMs: 0` state snapshot.
4. **Gate.** When a worker raises a question or a permission request, answer it with
   `respond_chat_session_interaction` if it is within the director's authority; otherwise surface
   the exact human decision required. A `needs_attention` state is never a completion.
5. **Collect and verify.** Read the worker, then independently confirm its artifact on disk —
   file exists, contents parse, numbers recompute. Only then mark `status: "done"`.
6. **Follow up or tear down.** Continue a worker with `send_message_to_chat_session` (it keeps its
   history, so short cues work — that is relay in miniature, and a relay cue still needs the messenger
   header if the gap since your last message is long). `stop_chat_session` a failed *spawned* one and
   respawn with a corrected task; never stop a `preexisting` one without the human's word.
7. **Synthesize in the director.** Cross-worker judgement, ordering, and the final write-up belong
   here. Workers should not be asked to evaluate each other.

## Prompt rules for a worker

A new conversation receives **none** of this conversation's history. Make each prompt self-contained:

- **Line 1 = a short task label** (e.g. `数据核对工人：汇总分渠道营收`) so the raw transcript is scannable.
  Do **not** expect it to be the sidebar name — see the title note under Verified constraints.
- Absolute paths for the project directory, every input, and the required output path.
- The concrete deliverable shape (file name, JSON keys, section headings).
- **Require execution evidence**: the verbatim command it ran, embedded in the artifact.
- An explicit reply budget ("reply in ≤5 lines: path, key figures, command") so the callback stays small.
- Boundaries: which files it must not touch, so concurrent workers do not collide.
- Do not instruct a worker to spawn another conversation.

## Verified constraints

Learned from two live runs (a two-worker spawn batch, then a spawn-to-idle-to-relay handoff on one
worker); treat these as facts, not guesses.

- A worker's shell starts in its **own** directory (`Documents/Qoder/<date>/<sessionId>`), not the
  director's. Relative paths in your prompt resolve against that. Always hand over absolute paths.
- `wait_chat_sessions`' `lastAssistantText` is **not dependable**: on the same session it returned the
  body on the first turn and `null` on the relay turn. Always follow a `turn_completed` with
  `read_chat_session`.
- A relay message is stored in the target as `role: "user"`, with `turnId` prefixed `agent-message:`.
  It takes the target's user slot, so the target cannot tell agent handoff from human speech by role —
  only by your declared header.
- Relaying to a session sitting `idle` needs no human message in between; the turn runs and the
  artifact lands. (Refusal on role conflict was never reproduced — see Relay mode.)
- `turn_completed` / `idle` describe execution state, **not** acceptance of the work. Verify the artifact.
- `node` may be absent from the Bash PATH even when Node exists on the machine. Prefer `awk`, or give
  the interpreter's absolute path; workers will otherwise improvise.
- `wait_chat_sessions` caps at 8 targets and excludes the current conversation.
- The worker inherits this conversation's model and persistent permission mode, while running at the
  target Workspace's location — set `workspaceId` when the work belongs elsewhere.
- **Conversation titles are not yours to control.** A session initially shows its first prompt line, but
  an asynchronous summarizer may rewrite it at any point (three observations across the two runs: a
  worker prompted as `你是一个被派发的工人会话…` became `解析CSV生成报告`; the relay worker became
  `创建并读取文件`; the third kept its prompt line). Identity must therefore live in `roster.json`,
  keyed by `sessionId` — never by title, and never by re-searching titles to find a worker again.
- `respond_chat_session_interaction` can answer a supervised worker's question, but the director cannot
  pre-answer its own; a genuinely human-only decision must still reach the user.

## Not reproducible

If asked for these, say so rather than faking them:

- **No GUI follow.** Nothing opens or focuses a conversation in the app UI on command, so the dsh
  "auto-switch to the current speaker" behaviour has no equivalent. Keep the stage in the director.
- **No rename.** There is no tool to set a conversation title and the app overwrites it with a generated
  summary anyway. Name workers in `roster.json` instead; do not promise the user named sidebar sessions.
- **No per-session voice binding.** No TTS hook exists.
- **No raw event stream.** Turn boundaries arrive as supervise callbacks / `wait` results, so there is
  no `session/event` to hand-roll `turn/start → turn/end` matching against.

## Resources

- `assets/roster.template.json` — cast list schema. Copy it to the project as `roster.json` and keep it
  updated through the lifecycle; it is the durable memory that lets later turns resume workers.
