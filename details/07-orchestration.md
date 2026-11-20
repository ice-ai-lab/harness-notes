# 7. 多 Agent 与编排 · 机制详情

> 对应 [../harness-matrix.md](../harness-matrix.md) 第 7 层信号矩阵。字段为行、harness 为列，同字段跨 harness 对齐。

---

## 多 Agent 范式（形态与启用）

> 核对日期：2026-11-20。口径：各家「多 Agent」不是单一机制，是几种拓扑形态的集合——**子代理（树状派生）、协调器（星形纯编排）、团队/信箱（常驻成员对等通信）**。本节记录每家支持哪些形态、入口是什么、用户怎么用。术语各家不互认：CC 的 teammate ≈ DSH 的 subagent（但常驻度不同），勿按名字直接对齐。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **形态清单** | ① 子代理（AgentTool，同步/异步/fork 四模式）② 协调器（ant-only）③ Swarm 团队（常驻队友+信箱） | 无内建多 Agent 工具面——核心 tools 目录只有文件/shell 类（read/grep/glob/bash 等），子代理靠 session fork（编排侧概念） | ① 子代理（`agent` 工具，独立受管、完成通知、可继续对话）② 队友协作（team 包：任务板+收件箱，"teammates are subagents"） | ① 协作工具面（`multi_agent_v1` namespace）：spawn_agent / send_input / send_message / followup_task / resume_agent / wait_agent / list_agents / close_agent / interrupt_agent，另有 multi_agent_v2 第二版面 |
| **子代理入口** | 🟦 Agent 工具（模型决定是否派生）：`{description, prompt, subagent_type, model, run_in_background, name, isolation}`；prompt 必须自包含（Worker 无对话上下文），fork 子代理除外（字节级继承父前缀保 prompt cache） | 🟦 无 AgentTool；`agent-session.ts` 提供 fork/派生 API，由扩展或编排层调用 | 🟦 `agent` 工具（`tool-subagent/src/index.ts:208`）：立即返回子代理名（不等完成），运行时自动投递完成通知；`send_message` 可在运行中转向、结束后继续对话；root agent 固定名 `lead` | 🟦 `spawn_agent`（`multi_agents_spec.rs`）：子代理从创建回合的 effective config 派生，继承 provider/approval/sandbox/cwd，可叠 role 配置；嵌套深度有上限（`exceeds_thread_spawn_depth_limit`） |
| **协调器** | 🟦 `isCoordinatorMode()`（`coordinatorMode.ts:36-40`）**双门控**：编译期 `feature('COORDINATOR_MODE')` ∧ 运行时 `CLAUDE_CODE_COORDINATOR_MODE=1`；外部构建 feature=false → 整棵协调器代码被 DCE 剔除（main.tsx:74 注释 "Dead code elimination"）——**ant 内部专属**。启用后系统提示词整体换编排者版、工具收窄到 Agent/SendMessage/TaskStop（`COORDINATOR_MODE_ALLOWED_TOOLS`），不能执行只能派发 | — | — | — |
| **团队/信箱** | 🟦 Swarm：门控 `isAgentSwarmsEnabled()`（`agentSwarmsEnabled.ts`）——ant 恒开；外部需 `--agent-teams` flag 或 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` + GrowthBook killswitch `tengu_amber_flint`（默认开）。开启后 TeamCreate/SendMessage 等解锁；**对话中的会话即 lead**（TeamCreate 时以固定名 `team-lead` 写进队成员表，`TEAM_LEAD_NAME` 常量），lead 同时只能管一个队；队友=独立 claude 进程（tmux/iTerm2 后端）或 InProcess 隔离运行，**常驻跨轮存活**；通信=SendMessage 点对点信箱（`to:"<name>"`/`to:"*"` 广播）+ 全队共享任务列表（Team=TaskList 一一对应） | — | 🟦 team 包（`team/tool-agent-team/`）：team_task_create/list/get/update 任务板；README 明示 "teammates are subagents, so subagent limits and inbox delivery apply"——DSH 无独立常驻队友进程，团队是子代理+任务板的组合 | — |
| **队友/子代理的工具与提示词** | 🟦 三层差异（v2.1.88）：① 普通子代理：极简系统提示词 + 工具过滤四层流水线（全局元工具黑名单→自定义类型加限→异步白名单→类型级 disallowedTools）② 队友：**主 Agent 同款完整系统提示词**（main.tsx:1384-1388 追加通信附言："只写文本别人看不见，必须用 SendMessage"）③ InProcess 队友：白名单 `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`（constants/tools.ts:77）= TaskCreate/Get/List/Update + SendMessage + Cron 系；**AgentTool 被禁**（防递归，注释明示——⚠️ 与 how-claude-code-works 07 章"进程内队友拿到 Agent 工具"相悖，以源码为准） | — | 🟦 子代理工具面独立注册；模型选择可选（provider/model/reasoning_effort，继承父值或 backend 路由默认） | 🟦 子代理从创建回合 config 派生 + role 配置叠加；工具面未见独立白名单（⚠️ 未深查） |
| **注** | ① 三形态共享同一套底层设施：AgentTool、ToolUseContext 隔离（readFileState 克隆、abort 单向传播、setAppState no-op 但 setAppStateForTasks 共享防僵尸进程、queryTracking depth 防无限递归）、`<task-notification>` 结果通知。② Worktree 隔离：`isolation:"worktree"` 参数 + `utils/worktree.ts`（slug 校验防路径穿越、无变更自动清理）。③ 用户交互统一为自然语言：Swarm 开启后不用学命令（可点名 team/swarm 强触发），协调器=启动方式即全部操作；「怎么用」的剧本写在系统提示词（协调器整体替换/队友附言）与工具 prompt（TeamCreate 工作流教学）里 | ① PI 的多 Agent 是「无内建工具面」形态：能力存在（session fork）但模型无专门工具可调 | ① `agent` 工具的「立即返回+完成通知+可 send_message 继续」三件套与 CC 异步模式同构；② 无 CC 式编译期门控——subagent/team 是常规包，profile 装配决定启用 | ① Codex 把整套协作面收进一个 namespace（v1）并演进 v2，与 CC 的「单 Agent 工具+参数分支」路线不同；② 通信动词族齐全（send_input/send_message/followup_task/wait_agent） |

---

## Agent 间通信

> 核对日期：2026-11-20。仅核了 CC 与 DSH 两家（有实作的两家），PI/Codex 未深查。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 两条通道：① `<task-notification>` XML（user 角色消息投递到父级下一轮，含 task-id/status/result/usage；原子 `notified` 标志去重防 stop+完成双发）② 队友信箱（SendMessage 点对点/广播，写进对方 mailbox 队列；队友附言强制「纯文本不可见」） | 待填 | 🟦 `send_message`（tool-subagent-control）：运行中转向 / 结束后继续；完成通知自动投递，若子代理结束前刚发过消息则通知不含最终答案（避免重复） | 🟦 `send_input` / `send_message` / `followup_task` / `wait_agent`（阻塞等待）多动词族 |
| **注** | ① 同步结果=子代理最后一条 assistant 文本包成 tool_result；异步结果=通知 XML。② 安全：`TRANSCRIPT_CLASSIFIER` 开启时子代理结果回传前过分类器（防借道子代理注入父对话） | — | — | — |

---

## Worktree 隔离

> 核对日期：2026-11-20。仅 CC 深查；其余家未研究。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 🟦 Agent 参数 `isolation:"worktree"`：在 `.claude/worktrees/<slug>/` 建独立工作树；slug 校验（≤64 字符、禁 `..`/绝对路径，`worktree.ts:66`）；完成后无变更（git diff 检测）自动删、有变更返回路径交用户决定；大目录（node_modules）用符号链接省磁盘 | 待填 | 待填（会话/编排层有 worktree 概念，未核是否达 Agent 参数级） | 待填 |
| **注** | 防多 Agent 并行写同一文件；与本工作区顶层 [create_worktree](../AGENTS.md) 工具同思路 | — | — | — |

---

## 待研究

- PI 的 session fork 编排细节、扩展如何暴露多 Agent 给模型（`06-skills.md` 已确认 PI 技能正文靠 Read 按需读，多 Agent 面未查）。
- Codex `multi_agent_v2` 与 v1 的演进差异、子代理工具面是否独立白名单。
- DSH team 包任务板与 subagent 收件箱的对接细节（journal/task-graph 未读）。
- CC fork 子代理的 `CacheSafeParams` 经济学（how-claude-code-works 07 章有分析，需回快照源码核验后再定稿）。
