---
name: dispatch-workers
description: Orchestrate independent Qoder conversations as full-tool workers - spawn a session for a task, let it read files and run commands on its own, supervise its turn, collect and verify its output, then follow up or abort. Use when the user asks to dispatch/parallelize work to separate agents or conversations, run several investigations or refactors concurrently, build a multi-session pipeline or cast, or resume a previously spawned worker.
---

# Dispatch Workers

## Overview

Turn the current conversation into a **director**: it never does the heavy work itself, it opens
independent Qoder conversations that carry the full toolset (Read / Write / Edit / Bash / Grep /
MCP), assigns each a self-contained task, waits for their turns, and verifies what landed on disk.

The director holds the plan, the roster, and the verdicts. Workers hold the execution. No scripting
language, no plugin process — every primitive is already a built-in MCP tool.

## Tool contract

| Role | Qoder tool | Notes |
| --- | --- | --- |
| Open a worker | `create_chat_session` | Pass `callbackMode: "supervise"` unless you truly want fire-and-forget. |
| Discover cast | `list_chat_sessions` | Returns ids + state, not transcripts. Use before read/wait/message. |
| Send next cue | `send_message_to_chat_session` | Also takes `callbackMode: "supervise"`. Never target yourself. |
| Collect output | `read_chat_session` | The only way to get reply text; `turnLimit`, `includeToolOutputs`. |
| Status / barrier | `wait_chat_sessions` | ≤8 targets, `timeoutMs` ≤ 60000; returns when **any** target settles. |
| Answer a gate card | `respond_chat_session_interaction` | Needs the `interactionRef` from a supervise callback. `allow`/`deny` or `indexedAnswers`. |
| Abort | `stop_chat_session` | Discard a worker whose output is wrong. |
| Persona selection | `list_agents` | Only `ready` agents are selectable; on a bare install just `agent:default` exists. |

## Lifecycle

1. **Plan the cast.** Decide worker roles, one bounded deliverable each. Write
   `assets/roster.template.json` shape into `roster.json` in the project directory before spawning;
   record `name`, `role`, `sessionId`, `task`, `expectedArtifact`, `status`.
2. **Spawn.** One `create_chat_session` per worker, prompt built with the rules below. Record the
   returned `sessionId` into `roster.json` immediately — it is only a dispatch receipt, not a result.
3. **Supervise.** Do not poll in a loop. Supervise mode delivers callbacks for permission requests,
   user questions, and turn completion. Use `wait_chat_sessions` only as a barrier or a one-shot
   `timeoutMs: 0` state snapshot.
4. **Gate.** When a worker raises a question or a permission request, answer it with
   `respond_chat_session_interaction` if it is within the director's authority; otherwise surface
   the exact human decision required. A `needs_attention` state is never a completion.
5. **Collect and verify.** Read the worker, then independently confirm its artifact on disk —
   file exists, contents parse, numbers recompute. Only then mark `status: "done"`.
6. **Follow up or tear down.** Continue a worker with `send_message_to_chat_session` (it keeps its
   history, so short cues work). `stop_chat_session` a failed one and respawn with a corrected task.
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

Learned from a live two-worker run; treat these as facts, not guesses.

- A worker's shell starts in its **own** directory (`Documents/Qoder/<date>/<sessionId>`), not the
  director's. Relative paths in your prompt resolve against that. Always hand over absolute paths.
- `wait_chat_sessions` returns `lastAssistantText: null` even when the turn completed — always follow
  with `read_chat_session` to get the text.
- `turn_completed` / `idle` describe execution state, **not** acceptance of the work. Verify the artifact.
- `node` may be absent from the Bash PATH even when Node exists on the machine. Prefer `awk`, or give
  the interpreter's absolute path; workers will otherwise improvise.
- `wait_chat_sessions` caps at 8 targets and excludes the current conversation.
- The worker inherits this conversation's model and persistent permission mode, while running at the
  target Workspace's location — set `workspaceId` when the work belongs elsewhere.
- **Conversation titles are not yours to control.** A session initially shows its first prompt line, but
  an asynchronous summarizer may rewrite it at any point (observed within ~2 minutes on one of two
  workers: a worker prompted as `你是一个被派发的工人会话…` became `解析CSV生成报告`; the other kept its
  prompt line). Identity must therefore live in `roster.json`, keyed by `sessionId` — never by title,
  and never by re-searching titles to find a worker again.
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
