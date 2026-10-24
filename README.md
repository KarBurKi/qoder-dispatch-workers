# qoder-dispatch-workers

Qoder 版的会话编排能力。参考 [KarBurKi/dsh-theater](https://github.com/KarBurKi/dsh-theater) 的内核，
把它在 Qoder Desktop 上能落地的部分做成一个可安装 Skill。

取的是 dsh-theater 的 **Host 半编排内核**：开独立会话 → 投任务 → 等回合 → 收产出 → 验收。
多角色剧场玩法、GUI 自动跟随当前说话人、GPT-SoVITS 音色绑定不在范围内，原因见下。

## 前置要求（先读这条，否则会静默失败）

本 skill 是说明书，不是引擎：它不带所驱动的会话编排原语，那些原语由**客户端形态**提供。

**门槛是形态，不是版本号。** 已验证可用的是 **Qoder 桌面 App**（Electron 独立客户端），本机指纹：
`productVersion 0.4.11`、`buildTime 2026-10-23T04:33:29Z`、`Electron 43.1.1`、`appId com.qoder.app`、
更新源 `download.qoder.com.cn/qoder-app/releases`；它的 `resources/product.json` 只有 12 个键，
**没有 VSCode 那套 `productVersion`／基线版本字段**。

已验证**不可用**的是 Qoder IDE（VSCode 派生那条线）：`1.21.2`、VSCode 基线 `1.106.3`、构建 2026-08-02，
Windows。该台由另一位使用者实测并报缺这 8 个原语。两条产品线的版本号不同源，
所以这里给不出「IDE 升到某版即可」这种答案——**别把它当版本问题排查**。

自检方法（比读文档可靠）：

1. 最硬的一条：直接看会话里有没有 `create_chat_session` 这类工具。它们由客户端自带的 `builtin` MCP
   server 暴露（本机 `list` 到 10 个：那 8 个原语加 `fork_chat_session`、`create_team_task`）。
   如果你的 MCP 目录里只有 `browser-use`、`genui`、`schedule` 这类，就没有替代装法能补上——
   **不是少装一个 server，是客户端不提供。**
2. 想扫包体验证，要扫 `resources/app.asar`，**别扫 `Qoder.exe`**：本机这台能跑通的机器上
   `Qoder.exe` 对 8 个名字也是**零命中**，而同一台的 `app.asar`（180 MB）命中其中 5 种
   （`create`=3、`list`=3、`send`=3、`read`=5、`wait`=3）。剩下 3 个（`respond_`、`stop_`、`list_agents`）
   在本机整套磁盘都搜不到——它们由运行时注册。所以「整包字符串零命中」推不出「代码里没这能力」，
   这条推理在有能力的机器上照样会给出否定结果。

结构校验通过、skill 出现在列表里、invoke 能返回文档，**都不代表原语可调用**。缺失时 spawn 与 relay
都跑不起来，本 skill 只能当方法论参考；跨会话协作请退回磁盘中转（例如各会话读写同一目录下的约定文件）。

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

## 两种派发模式：spawn 与 relay

同一个内核有两条投递路径，差别只在工人会话从哪来。

**spawn（默认）**：`create_chat_session` 新建一条空白会话，把它当工人。适合并行铺开一批互不相干的任务，
成本低、边界干净——新会话除了你给的 prompt 什么都没有，不会带上别处的历史。

**relay（接力）**：不新建会话，用 `send_message_to_chat_session` 把任务投给一条**已经开着**的会话——
通常是另一条正在干活的会话，或你之前留着的工人。适合「把结论要回来」而不是「再雇一个人」。
因为消息必然进入目标的既有历史，relay 的 prompt 必须自带三样东西：

1. **传话头声明**：开头就讲清「我是另一条会话（导演），这条消息来自 agent，不是主人本人」。
   实测目标会话认这个：它先回答了导演问的上一题（ recalling 自己写的文件名与内容串），再接新任务。
2. **自包含任务**：绝对路径、产出形态、回复预算一次说全，不要假设目标知道导演的上下文。
3. **回执预算**：要求「≤5 行：路径、关键数字、跑过的命令」，否则回调正文很大。

roster 里以 `dispatchMode: "relay"` + `preexisting: true` 登记（见 `assets/roster.template.json`），
这样后续回合知道这条工人不是本导演创建的，不能想当然地 `stop_chat_session` 它。

relay 的代价与未验证项，实测与推测分开写：

- **已实测**：对一条 `idle` 的会话直接投递即可驱动，中间不需要主人插一条消息；任务跑完、产出落盘、
  导演读得到。存储语义也确认了——relay 消息在目标历史里落成 `role: "user"`，`turnId` 前缀
  `agent-message:`，也就是说**它会占用目标的用户槽位**。
- **未实测**：目标因角色冲突拒单（比如它正被主人当成别的用途、或它的 persona 与任务矛盾）。
  行为上完全可能，但我们没造出这个场景，不编。
- **结构性限制**：既然落在 `user` 槽，目标无法从消息角色本身分辨这是 agent 传话还是主人说话，
  全靠传话头声明。对不可信的第三方会话传话，等于把 prompt 写进它的用户输入，慎用。

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
| 对**已在场**的角色说下一句 | `send_message_to_chat_session`（即 relay 模式） |

没有做成 MCP server：这些原语本来就是 Qoder 内置工具，再加一层常驻进程和安装步骤只是负担。

## 复刻不了的部分

- **GUI 跟随**：没有可注入的界面 bundle，也没有聚焦/打开会话的工具，所以「视图自动切到当前说话人」无对应实现。
- **改名**：没有设置会话标题的工具，且标题会被异步摘要器随时重写（实测约 2 分钟内发生），身份只能靠 `sessionId`。
- **音色**：没有 dsh-tts / `sessionVoices` 那样的每会话语音钩子。

## 实测约束

以下来自两次真实跑批（一次两工人并发 spawn，一次 spawn→idle→relay 接力），不是读代码推断：

- 工人 shell 的起始目录是它自己的 `Documents/Qoder/<date>/<sessionId>`，**不是派单方目录**，一律传绝对路径。
- `wait_chat_sessions` 的 `lastAssistantText` **不可依赖**：同一条会话上，首回合给了正文，
  relay 后的第二回合回 `null`。稳定做法是拿到 `turn_completed` 后再走一次 `read_chat_session`。
- `turn_completed` / `idle` 只描述执行状态，不等于工作被验收；必须独立复算磁盘产出。
- `node` 可能不在 Bash 的 PATH 中（实测 `command not found`），优先 `awk` 或给解释器绝对路径。
- `wait_chat_sessions` 上限 8 个目标，且不能包含当前会话。
- 会话标题不可控：异步摘要器会随时重写（本机三次观测——两工人分别变成
  「解析CSV生成报告」与「创建并读取文件」，另一个保持 prompt 首行）。身份只认 `sessionId`，
  别靠标题回头找人。

## 关于 0.1.1 的 Relay 模式

Relay 不是从别处的 diff 合进来的。另一台机器的使用者先按行为描述做出了 0.1.1，
本机按同一份描述重建，并把那句「该路径未经实测」补成了实测——上文的存储语义
（`role: "user"`、`turnId` 前缀 `agent-message:`、idle 会话免人工插话）就是这一轮拿到的。
剩下的未实测项（角色冲突拒单）保留原样，不假装验过。

## 状态

Skill 本体在 `skills/dispatch-workers/`，版本 `0.1.1`（前置要求 + Relay 模式）。
未附带 LICENSE 文件，需要的话按你的偏好补。
