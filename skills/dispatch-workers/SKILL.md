---
name: dispatch-workers
description: Orchestrate independent Qoder conversations as full-tool workers - spawn a session for a task, hand a task to an already-open conversation, let it read files and run commands on its own, supervise its turn, collect and verify its output, then follow up or abort. Use when the user asks to dispatch/parallelize work to separate agents or conversations, run several investigations or refactors concurrently, build a multi-session pipeline or cast, relay a task to a specific existing session, or resume a previously spawned worker.
---

# Dispatch Workers

Version 0.1.3 — 0.1.1 added Prerequisites and Relay mode, 0.1.2 the measured CLI-driven fallback,
0.1.3 the measured limits of two-way conversation (call mode).

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
nothing about callability. If the primitives are absent, this MCP-primitive path cannot run — but the
orchestration itself can still be done: see **Fallback: CLI-driven dispatch** below. Only if that is also
unavailable should you fall back to disk-mediated handoff (conversations read and write agreed files in
a shared directory).

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

## Fallback: CLI-driven dispatch (no primitives needed)

The primitives belong to the Desktop App, but orchestration does not. `qodercli` carries session identity
and resume, so a director with a shell can run the same chain. Measured on `qodercli 1.1.78`:

```bash
# 1. Spawn — the director pre-assigns the id so it can go straight into the roster
qodercli -p --output-format json --session-id <uuid> \
  -w "C:\\absolute\\worker\\dir" --permission-mode accept_edits \
  "Write C:\\absolute\\worker\\dir\\probe1.txt containing CLI-PROBE-1. Reply with one line: path and content."
#    stdout is one JSON object: session_id / result / num_turns / duration_ms / total_credits
#    measured: 12.8 s, num_turns 2, 0.162 credits, the 11-byte file was really on disk

# 2. Relay — resume the same id; the messenger header is still mandatory
qodercli -p --output-format json --resume <same session id> --permission-mode accept_edits \
  "[Handoff, not a continuation] I am another conversation (the director). First answer: what file and string did your last task write? Second, new task: ..."
#    measured: the worker recalled probe1.txt / CLI-PROBE-1 correctly, then produced the new artifact

# 3. Cast discovery
qodercli --list-sessions
```

Costs and traps, all measured rather than assumed:

- CLI sessions do **not** appear in the App/IDE sidebar, and `--list-sessions` groups by workspace — its
  own header reads `Available sessions for this project`. Identity lives in the roster, as always.
- Calling the CLI from **inside** a Qoder conversation fails outright:
  `sdk_invalid_args: Agent SDK entrypoint env is set but required flags are missing`, because the host's
  `QODER_AGENT_SDK_ENTRYPOINT` and friends are inherited. Strip them and normal behaviour returns:

  ```bash
  env -u QODER_AGENT_SDK_ENTRYPOINT -u QODER_AGENT_SDK_DAEMON \
      -u QODER_AGENT_SDK_VERSION -u QODER_SDK_AUTH_PAYLOAD_FILE qodercli -p ...
  ```

  A human's own terminal never sees this.
- There is no supervise callback to answer, so permissions must be granted up front (`--permission-mode`,
  `--allowed-tools`); otherwise headless runs silently refuse — check `permission_denials`.
- Cost is observable per call; `total_credits` ranged from 0.02 to 0.16 for the same shape of task here.
- **Unverified**: whether the older `qodercli 1.0.34` bundled by an IDE install supports `--session-id`
  and `--list-sessions` (`-p` / `--resume` are baseline). The CLI has its own `update` subcommand, so it
  can be upgraded **without touching the IDE** — which is exactly what sidesteps a stale desktop build.
- Same-source corollary: grepping the 220 MB `qodercli` binary gives **zero hits** for all eight primitive
  names. Installing a CLI does not grant the primitives; it offers an equivalent chain instead.

Everything else transfers unchanged: roster keyed by `sessionId`, absolute paths, independent disk
verification, and no acceptance before the artifact is recomputed.

## Call mode: two-way conversation (measured)

A worker can talk **back**, so a director and one worker can hold a real conversation without the human
injecting a message at each step. Measured chain: director→worker (relay) → worker→director
(`send_message_to_chat_session` back to the delegator) → director→worker again. The inbound reply lands
in the director as an ordinary queued turn.

Two facts define the shape, both measured rather than assumed:

1. **The topology is a star, not a mesh.** Replying to the conversation that delegated the current turn
   is allowed. Forwarding to a *third* conversation is blocked by the platform, with this refusal
   returned to the worker verbatim:
   `由其他会话委派的任务必须直接执行，不能继续向其他会话转发消息。`
   So build conversations as director↔worker pairs; do not design a worker-to-worker relay, and do not
   chain casts through a worker.
2. **An inbound agent message carries no sender id.** It arrives as `role: "user"` with
   `turnId = agent-message:<hash>`; nothing in the payload names who sent it. A conversation that must
   reply therefore needs the address handed to it explicitly — put a `REPLY-TO=<sessionId>` line in the
   prompt (and require the worker to copy it verbatim into its reply). This works; guessing from
   `list_chat_sessions` does not, because a worker can see **every** conversation on the machine,
   including unrelated cold ones — so never let a worker pick its own reply target.

To hold a call: seed the worker with its reply address and a turn budget ("reply to `<director id>` with
exactly one message, then stop"), then answer it with `send_message_to_chat_session`. Keep the budget
explicit in every round — two agents with free rein will keep each other running, and only the human can
`stop_chat_session` a runaway.

Note `wait_chat_sessions` refuses the director's own id:
`不能等待当前 ChatSession；当前 Turn 会因此无法结束。` Use `list_chat_sessions` for a self snapshot.

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
- `wait_chat_sessions`' `lastAssistantText` is **not dependable**: for one session at the *same*
  `revision`, it was `null` right after the turn ended and fully populated in a `timeoutMs: 0` snapshot
  a few minutes later. It is written asynchronously, so `null` means "not yet", not "absent". To get the
  body, always call `read_chat_session`.
- **Supervise callbacks arrive late and can duplicate.** On one probe session the first turn's callback
  (completed 23:47:58) landed at 00:03 and the relay turn's (23:48:44) at 00:05 — over fifteen minutes
  late, delivered as a backlogged pair, in order. A callback is a signal to *go read*, not a new fact:
  compare `revision` and the artifact's mtime before concluding anything happened.
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
