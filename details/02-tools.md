# 2. 工具 · 机制详情

> 对应 [../harness-matrix.md](../harness-matrix.md) 第 2 层信号矩阵。字段为行、harness 为列，同字段跨 harness 对齐。

---

## 默认工具集

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | `getAllBaseTools()` 列出全部可注册工具 → `getTools()` 过滤（deny 规则 → REPL 模式隐藏 → `isEnabled()`）→ `assembleToolPool()` 合并 MCP 工具（内置优先去重、内置排连续前缀保 prompt cache）→ 每回合 `computeTools()` 从 store 现算 | — | — | — |
| **值** | 无条件 ~25 个 + 条件包含 ~16 个（默认构建下多数为真）+ 环境/flag 特定 ~14 个，见下方清单 | — | — | — |
| **位置** | `CCB: src/tools.ts:218` `getAllBaseTools()` · 过滤 `src/tools.ts:304` `getTools()` · 合并 MCP `src/tools.ts:378` `assembleToolPool()` · 特殊工具剔除 `src/tools.ts:334` | — | — | — |
| **注** | ① `ListMcpResources`/`ReadMcpResources` 在 `getAllBaseTools` 里但被 `getTools` 的 specialTools 剔除，由 MCP 连接状态动态注入。② SIMPLE 模式（`CLAUDE_CODE_SIMPLE=1`）只剩 Bash/Read/Edit。③ 工具搜索开启时，不在 `CORE_TOOLS` 白名单（`src/constants/tools.ts:137`）的工具从 API 请求剔除，改经 SearchExtraTools + ExecuteExtraTool 按需调用。④ ant 原生构建内嵌 bfs/ugrep 时 Glob/Grep 不注入（开源构建不会发生） | — | — | — |

### 工具清单

**A. 无条件注入**（`getAllBaseTools()` 直写；`isEnabled` 均默认 true）：

| 工具 | 作用 |
|---|---|
| `Agent` | 启动子代理执行开放式搜索/多步任务（原名 Task） |
| `TaskOutput` | 读取后台任务（shell/agent）的输出流 |
| `Bash` | shell 命令执行，持久会话 |
| `ExitPlanMode` | 退出计划模式，提交计划开始执行 |
| `Read` | 读文件（文本/图片/PDF/Jupyter），带行号、offset/limit 分页 |
| `Edit` | 精确字符串替换编辑文件 |
| `Write` | 写/覆盖文件 |
| `NotebookEdit` | 编辑 Jupyter `.ipynb` 单元格（替换/插入/删除） |
| `artifact` | 上传 HTML/Markdown 到 cloud-artifacts，返回公开 URL |
| `WebFetch` | 抓取 URL 转 markdown，可按 prompt 提取 |
| `TodoWrite` | 维护任务清单（旧版 todo 面板） |
| `WebSearch` | 网络搜索 |
| `TaskStop` | 停止后台任务/代理 |
| `AskUserQuestion` | 向用户出选择题获取决策 |
| `Skill` | 执行技能（slash 命令展开为完整 prompt） |
| `EnterPlanMode` | 进入计划模式（只读探索） |
| `LocalMemoryRecall` | 只读检索用户跨会话笔记（仅主线程） |
| `VaultHttpFetch` | 用 vault 存储的密钥发认证 HTTPS 请求（仅主线程） |
| `SendMessage` | 向其他 agent 发消息 |
| `TeamCreate` / `TeamDelete` | 创建/删除 agent 团队 |
| `CronCreate` / `CronDelete` / `CronList` | 定时任务（cron）管理 |
| `ExecuteExtraTool` | 调用经 SearchExtraTools 发现的延迟工具 |

**B. 条件包含 — 默认构建/运行时通常为真**：

| 工具 | 注入条件 | 作用 |
|---|---|---|
| `Glob` | `!hasEmbeddedSearchTools()`，开源构建恒真 | `rg --files --glob` 按文件名模式找文件，按修改时间排序，上限 100 |
| `Grep` | 同上 | ripgrep 内容正则搜索，content/files/count 三种输出模式 |
| `TaskCreate` / `TaskGet` / `TaskUpdate` / `TaskList` | `isTodoV2Enabled()`：交互式会话恒真；headless 需 `CLAUDE_CODE_ENABLE_TASKS=1` | Tasks V2 结构化任务管理 |
| `EnterWorktree` / `ExitWorktree` | `isWorktreeModeEnabled()` 恒 true | 创建隔离 git worktree 并切入/退出返回原目录 |
| `Workflow` | `feature('WORKFLOW_SCRIPTS')` 默认开 | 运行 `.claude/workflows/` 工作流脚本（多代理编排） |
| `Sleep` | `feature('PROACTIVE') \|\| feature('KAIROS')` 默认开 | 定时睡眠等待（proactive 模式节奏控制） |
| `RemoteTrigger` | `feature('AGENT_TRIGGERS_REMOTE')` 默认开 | 远程触发 agent 会话 |
| `Monitor` | `feature('MONITOR_TOOL')` 默认开 | 流式监控后台进程输出 |
| `SendUserFile` | `feature('KAIROS')` 默认开 | 向用户发送文件 |
| `PushNotification` | `feature('KAIROS')` 默认开 | 推送通知 |
| `DiscoverSkills` | `feature('EXPERIMENTAL_SKILL_SEARCH')` 默认开（运行时开关另控） | 技能语义搜索发现 |
| `Goal` | `feature('GOAL')` 默认开 | 持久目标 + 自动续跑 |
| `SearchExtraTools` | `isSearchExtraToolsEnabledOptimistic()` 默认开 | TF-IDF 工具搜索，发现延迟工具后转 ExecuteExtraTool 调用 |
| `SendUserMessage`（BriefTool） | `isEnabled() = isBridgeEnabled()`，仅 bridge/remote 模式 | 向用户发消息的主输出通道（assistant 模式必用） |

**C. 条件包含 — 环境/flag 特定（默认不开）**：

| 工具 | 注入条件 | 作用 |
|---|---|---|
| `Config` / `Tungsten` / `REPL` / `SuggestBackgroundPR` | `USER_TYPE=ant`（Anthropic 内部） | 配置管理 / 内部监控 / VM 内聚合执行原语 / 后台 PR 建议 |
| `PowerShell` | Windows 且未 `CLAUDE_CODE_USE_POWERSHELL_TOOL=0` | Windows PowerShell 执行 |
| `LSP` | `ENABLE_LSP_TOOL=1` | LSP 诊断/定义跳转 |
| `VerifyPlanExecution` | `CLAUDE_CODE_VERIFY_PLAN=true` | 计划执行验证 |
| `TestingPermission` | `NODE_ENV=test` | 测试用权限工具 |
| `WebBrowser` / `OverflowTest` / `SubscribePR` | 对应 feature 未入默认构建列表 | 浏览器 / 压测 / PR 订阅 |
| `Snip` / `CtxInspect` / `ReviewArtifact` | feature 已禁用（HISTORY_SNIP / CONTEXT_COLLAPSE / REVIEW_ARTIFACT） | — |


1. builtInTools 返回的内建Tool列表
const builtInTools = getTools(permissionContext)

Agent
TaskOutput
Bash
Glob
Grep
ExitPlanMode
Read
Edit
Write
NotebookEdit
artifact
WebFetch
TodoWrite
WebSearch
TaskStop
AskUserQuestion
Skill
EnterPlanMode
LocalMemoryRecall
VaultHttpFetch
EnterWorktree
ExitWorktree
SendMessage
TeamCreate
TeamDelete
CronCreate
CronDelete
CronList
DiscoverSkills
SearchExtraTools
ExecuteExtraTool

2. commands.ts 的 loadAllCommands



