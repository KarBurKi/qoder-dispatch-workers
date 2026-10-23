# qoder-dispatch-workers

Qoder 版的会话编排能力。参考 [KarBurKi/dsh-theater](https://github.com/KarBurKi/dsh-theater) 的内核，
把它在 Qoder Desktop 上能落地的部分做成一个可安装 Skill。

取的是 dsh-theater 的 **Host 半编排内核**：开独立会话 → 投任务 → 等回合 → 收产出 → 验收。
多角色剧场玩法、GUI 自动跟随当前说话人、GPT-SoVITS 音色绑定不在范围内，原因见下。

## 安装

把 `skills/dispatch-workers/` 放进下列任一位置：

```
# 用户级（所有会话可用）
~/.qoder/skills/dispatch-workers/

# 项目级（仅该仓库可用）
<repo>/.qoder/skills/dispatch-workers/
```

然后 `/skills reload` 或重启会话，用 `/skills list` 确认出现 `dispatch-workers`。

## 用法

直接在对话里说「把这几件事拆给独立会话并行去做」，或显式调用：

```
/dispatch-workers
```

当前会话成为导演，每个工人是一个**独立的 Qoder 会话**，自带全套工具（Read / Write / Edit / Bash /
Grep / MCP），会自己读文件、跑命令、写盘。

## 与 dsh-theater 的对应关系

| dsh-theater | 本项目 |
| --- | --- |
| `theater_open` 建角色会话 | `create_chat_session`（`callbackMode: "supervise"`） |
| `theater_say` 投 cue、等回合 | `send_message_to_chat_session` + 监督回调 |
| 订阅 `session/event` 判 `turn/end` | 监督回调 / `wait_chat_sessions` |
| 取回台词原文 | `read_chat_session` |
| `userQuestions.ask` 放行卡 | `respond_chat_session_interaction` |
| `actors.json` 演员表 | `roster.json`（以 `sessionId` 为身份键） |
| `theater_cast` | `list_chat_sessions` + `roster.json` |

没有做成 MCP server：这些原语本来就是 Qoder 内置工具，再加一层常驻进程和安装步骤只是负担。

## 复刻不了的部分

- **GUI 跟随**：没有可注入的界面 bundle，也没有聚焦/打开会话的工具，所以「视图自动切到当前说话人」无对应实现。
- **改名**：没有设置会话标题的工具，且标题会被异步摘要器随时重写（实测约 2 分钟内发生），身份只能靠 `sessionId`。
- **音色**：没有 dsh-tts / `sessionVoices` 那样的每会话语音钩子。

## 实测约束

以上结论来自一次两个工人并发的真实跑批，不是读代码推断：

- 工人 shell 的起始目录是它自己的 `Documents/Qoder/<date>/<sessionId>`，**不是派单方目录**，一律传绝对路径。
- `wait_chat_sessions` 即使返回 `turn_completed`，`lastAssistantText` 仍为 `null`，正文要另走 `read_chat_session`。
- `turn_completed` / `idle` 只描述执行状态，不等于工作被验收；必须独立复算磁盘产出。
- `node` 可能不在 Bash 的 PATH 中（实测 `command not found`），优先 `awk` 或给解释器绝对路径。
- `wait_chat_sessions` 上限 8 个目标，且不能包含当前会话。

## 状态

Skill 本体在 `skills/dispatch-workers/`。未附带 LICENSE 文件，需要的话按你的偏好补。
