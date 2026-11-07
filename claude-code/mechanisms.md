# Claude Code Harness 机制全景

> **定位**：记录 Claude Code 在各种情况下的决策机制（正常路径 + 异常路径），按层组织、按机制分章。
> 每个机制一张"决策解剖卡"：**触发条件 → 决策流程 → 数据与呈现 → 策略参数 → 边界与降级 → 源码位置 → 证据强度**。
> 后续将以此卡为模板，横向对比 pi / dsh / codex（见 §7）。
>
> **源码锚点说明**：本文 `CCB:` 前缀的路径指向本机 `claude-code-best/` 仓库——它是 Claude Code 的工程化复原项目，锚点行号基于 v2.1.888，与官方行为高度一致但非逐字官方源码，关键结论需按 §6 清单验证。

## 证据强度图例

| 标记 | 含义 |
|---|---|
| ✅ | Anthropic 官方文档/官方博客确认 |
| 🟦 | CCB 源码锚点确认（复原源码，可信度高，含 file:line） |
| ⚠️ | 待验证（社区分析或推断，列入 §6 验证清单） |

**分层参考**：沿用 CCB 官方文档站（https://ccb.agent-aura.top/ ，源码 `claude-code-best/docs/`）的分组——对话循环 / 工具 / 上下文工程 / 多 Agent / 可扩展性 / 安全权限 / 内部机制，在此之上增加"异常决策"视角。

---

## 0. 机制全景一页表

| 层 | 机制 | 触发 | 核心决策（一句话） |
|---|---|---|---|
| L1 循环 | 1.1 主循环 | 每轮对话 | stop_reason 分支：tool_use → 工具管线 → 回环；否则结束回合 |
| L1 循环 | 1.2 API 错误重试与降级 | 429/529/5xx/断流 | 按 retry-after 退避重试；fast mode 降级；流式→非流式 fallback；模型 fallback（Opus→Sonnet） |
| L1 循环 | 1.3 紧急压缩 | prompt too long | 边压缩边重试，从尾部剥离最早对话组换 token 空间 |
| L1 循环 | 1.4 用户中断 | Esc / 拒绝工具 | abort + 注入 UserInterruptionMessage 让 AI 理解发生了什么 |
| L1 循环 | 1.5 消息排队 Steering | AI 运行中用户输入 | 不打断，排队，回合边界注入 |
| L1 循环 | 1.6 计划模式 | Shift+Tab | 工具层硬拦截写操作 + 换规划系统提示词 |
| L2 上下文 | 2.1 上下文组装 | 每次 API 调用 | 身份提示词 + 环境块 + CLAUDE.md 层级合并 + 工具 schema |
| L2 上下文 | 2.2 Compaction 家族 | 阈值/预警/紧急 | auto / micro / snip / reactive / sessionMemory 五种压缩器分工 |
| L2 上下文 | 2.3 会话持久化 | 每条消息 | jsonl 追加落盘，resume/fork 重建状态 |
| L2 上下文 | 2.4 记忆体系 | 跨会话 | CLAUDE.md 层级 + 自动记忆提取 |
| L3 工具 | 3.1 工具执行管线 | 每个 tool_use | 权限 → 沙箱 → 执行 → 结果后处理 → hook |
| L3 工具 | 3.2 超时后台化 | 命令超时 | 不直接杀：转后台继续跑 + 通知 |
| L3 工具 | 3.3 后台任务通知 | 后台任务完成 | task-notification XML 注入下一轮上下文 |
| L3 工具 | 3.4 并行工具调用 | 多个 tool_use | 并发执行 + 读写冲突串行化 |
| L3 工具 | 3.5 子 Agent | Task/Agent 工具 | 独立上下文窗口，只回传最终摘要 |
| L3 工具 | 3.6 结果后处理 | 输出过大 | 截断/分页/图片降级 |
| L4 安全 | 4.1 权限模型 | 每个工具调用 | deny > ask > allow 规则求值 + 审批 UI 状态机 |
| L4 安全 | 4.2 沙箱 | 已批准命令 | OS 级文件系统/网络约束，沙箱内默认放行 |
| L4 安全 | 4.3 Bash 解析 | Bash 调用 | 复合命令拆分、注入检测、读写路径预判 |
| L4 安全 | 4.4 注入防护 | 不可信内容 | 网页/工具输出当数据不当指令 |
| L5 扩展 | 5.1 Hooks | 生命周期事件 | 确定性护栏：exit 2 / JSON decision 阻断并回灌 |
| L5 扩展 | 5.2 Skills/MCP/Plugins | 按需 | 渐进式披露：元数据常驻，正文按需加载 |

---

## 1. L1 循环控制层（对话是如何运转的）

> CCB 文档站对应章节：`docs/conversation/`（the-loop / streaming / multi-turn）

### 1.1 主循环状态机 🟦

- **触发**：每条用户消息（或 queued 消息注入）。
- **决策流程**：
  1. 组装请求（system + messages + tools）→ 调 API 流式返回；
  2. 消费 SSE 事件流，聚合 assistant 消息；
  3. `stop_reason === 'tool_use'` → 进入工具执行管线（§3.1）→ tool_result 追加回历史 → **回到第 1 步**；
  4. `end_turn` / `max_tokens` → 结束回合，交还 REPL；
  5. 每次回环前检查上下文余量（§2.2）与 token 预算。
- **源码锚点**：`CCB: src/query.ts`（主查询循环，yield 消息流）；`CCB: src/QueryEngine.ts`（上层编排：压缩、快照、回合记账）；`CCB: src/query/transitions.ts`、`CCB: src/query/stopHooks.ts`（回合边界与停止钩子）。
- **边界**：OpenAI 兼容路径流结束无 `message_stop` 时有 post-loop safety fallback 兜底产出半截消息（`CCB: src/services/api/openai/index.ts`）。

### 1.2 API 错误 → 重试与降级 🟦（⭐ 面试高频）

- **触发**：429（限流）/ 529（overloaded）/ 5xx / 401 / 网络断流。
- **决策流程**（`CCB: src/services/api/withRetry.ts`）：
  1. **可否重试判定**：服务器 `x-should-retry` 头优先遵循；overloaded_error 消息体识别；429 对 Max/Pro 与 Enterprise 订阅类型分别处理；
  2. **退避**：优先取 `retry-after` 头（`getRetryAfterMs`），否则指数退避（`getRetryDelay(attempt, retryAfter)`）；
  3. **fast mode 降级**：429/529 时若 retry-after 很短 → 原模型短等待重试（保 prompt cache）；retry-after 长/未知 → 进入 cooldown，降回标准速度模型（`DEFAULT_FAST_MODE_FALLBACK_HOLD_MS`）；
  4. **529 计数预算**：连续 529 达到 `MAX_529_RETRIES` → 触发 `FallbackTriggeredError`（`Model fallback triggered: X -> Y`），换 fallback 模型重试（如 Opus→Sonnet，`tengu_api_opus_fallback_triggered`）；
  5. **流式 → 非流式 fallback**：流中途断（stream idle abort）或流结束无内容块 → 用非流式请求补一发（带 ~5min 单次上限超时防止挂死，`tengu_nonstreaming_fallback_started/error`），并把 529 计数跨路径续算（`initialConsecutive529Errors`）；
  6. **不可重试**：`CannotRetryError` 直接抛出——非前台调用（摘要/标题/分类器）在容量雪崩时**立即放弃**，避免每个后台请求重试放大 3-10 倍网关流量（源码注释原文语义）；
  7. **无人值守模式**：`CLAUDE_CODE_UNATTENDED_RETRY` 下 429/529 无限重试 + 更长退避 + 周期 keep-alive（⭐ 面试点：**有人值守 vs 无人值守采用完全不同的重试策略**）。
- **错误呈现**：429 剥离状态码前缀后转成面向模型的错误文本（`API_ERROR_MESSAGE_PREFIX: Request rejected (429)…`），并提示可用 `/model` 切换 fallback 模型（`CCB: src/services/api/errors.ts:467-560`）。
- **边界与降级**：401 → OAuth 刷新令牌后重试（`withOAuth401Retry`）；max_tokens 超上限 → 自动调低 thinking budget 或以 64k 重试一次。
- **面试金句**：重试不是单一策略，是**五层降级瀑布**：退避重试 → fast mode cooldown → 模型 fallback → 流式转非流式 → 前台/后台差异化放弃。

### 1.3 紧急压缩（Prompt Too Long）→ 边压缩边重试 🟦

- **触发**：API 返回 prompt too long（上下文爆窗），或压缩本身请求也爆窗（二阶问题）。
- **决策流程**（`CCB: src/services/compact/compact.ts`）：
  1. PTL（Prompt Too Long）重试循环**从历史尾部剥离最早对话组**（grouping），给压缩请求腾出空间，加合成标记 `[earlier conversation truncated for compaction retry]`；
  2. 重试前剥离上一次的合成标记，防止"只删标记、零进展"死循环；
  3. 压缩流式请求失败 → `tengu_compact_streaming_retry`，最多 `MAX_COMPACT_STREAMING_RETRIES` 次（feature flag `tengu_compact_streaming_retry` 控制）；
  4. 代价标注：PTL retry 打破 prompt cache 前缀，但"解锁用户优先"（源码注释 `CC-1180`）。
- **面试金句**：压缩请求自己也可能爆窗——**递归降级**是 harness 设计里容易被忽略的暗角。

### 1.4 用户中断 → UserInterruptionMessage 🟦

- **触发**：Esc 打断流式/工具执行；或在权限确认时拒绝工具。
- **决策流程**：
  1. AbortController 取消 API 流 / 工具执行；
  2. **区分两种中断语义**：`createUserInterruptionMessage({ toolUse: false })`（打断文本生成）与 `{ toolUse: true }`（打断工具使用），分别生成 `INTERRUPT_MESSAGE` / `INTERRUPT_MESSAGE_FOR_TOOL_USE` 两种文案（`CCB: src/utils/messages.ts:548`）；
  3. 合成消息以 user 消息身份注入历史（带唯一 uuid），让模型下一轮理解"刚才发生了什么"；
  4. 中断点分布在三处：主循环（`CCB: src/query.ts:1334,1796`）、停止钩子（`CCB: src/query/stopHooks.ts:300`）、本地命令处理（`CCB: src/utils/processUserInput/processSlashCommand.tsx:972`、`processBashCommand.tsx:150`）。
- **设计意图**：中断不是简单丢弃——**把控制流事件翻译成模型可读的数据流事件**，保持对话叙事连续。
- ⚠️ 待验证：子 agent 被中断时父会话收到什么；中断时已产出部分 tool 输出如何保留。

### 1.5 运行中输入 → 排队与 Steering ⚠️

- **触发**：AI 工作期间用户输入新消息。
- **决策流程**（社区分析 + 版本演进）：默认不打断——输入进入队列；回合边界（工具轮次之间/当前回合结束）注入为下一条 user 消息；Esc 才是显式打断（走 §1.4）。
- **源码锚点**：`CCB: src/screens/REPL.tsx`（输入状态与队列）⚠️ 具体入队时机待定位。
- **面试点**：steering 让"多轮协作"变成可能——用户可以中途纠偏而不用停机重启。

### 1.6 计划模式 ⚠️

- **触发**：Shift+Tab 切换到 plan mode；或模型调用 EnterPlanModeTool。
- **决策流程**：权限层把写工具（Edit/Write/变更类 Bash）硬拦截为 deny；系统提示词切换为"研究 → 出计划 → ExitPlanMode 提交审批"流程；用户批准计划后才允许实施（常伴随切到 acceptEdits 模式）。
- **源码锚点**：`CCB: packages/builtin-tools/src/tools/`（EnterPlanModeTool / ExitPlanModeV2Tool / VerifyPlanModeTool）；`CCB: docs/safety/plan-mode` 文档。
- **面试点**：只读约束放在**工具权限层**而非 prompt 约束——prompt 可被绕过，权限层不可。

---

## 2. L2 上下文层（上下文工程）

> CCB 文档站对应章节：`docs/context/`（system-prompt / project-memory / compaction / token-budget）

### 2.1 上下文组装 🟦

- **触发**：每次 API 调用前。
- **决策流程**：身份/行为系统提示词 → 环境信息块（cwd、git 状态、平台、日期）→ CLAUDE.md 层级内容合并 → 工具 schema 数组 → beta headers。
- **源码锚点**：`CCB: src/context.ts`（构建 system/user context）；`CCB: src/utils/claudemd.ts`（CLAUDE.md 层级发现与加载：企业级 → 项目根 → 子目录 → 用户级 `~/.claude/CLAUDE.md`，支持 `@path` import）。

### 2.2 Compaction 家族 🟦（⭐ 本仓库最完整的机制族）

- **触发**：token 用量逼近阈值 / 用户 `/compact` / API 爆窗（§1.3）。
- **五种压缩器分工**（`CCB: src/services/compact/`）：

| 压缩器 | 文件 | 职责 |
|---|---|---|
| autoCompact | `autoCompact.ts` | 阈值触发的整段历史压缩；**断路器**：连续失败 N 次停止重试，避免每轮空转 |
| microCompact / apiMicrocompact / cachedMicrocompact | 同目录 | 细粒度清理：清除陈旧工具输出保留近期，缓存友好版本避免打爆 prompt cache |
| snipCompact | `snipCompact.ts` + `snipProjection.ts` | 剪切片段式压缩（保留投影） |
| reactiveCompact | `reactiveCompact.ts` | PTL 之后被动触发，一次跳过多组历史 |
| sessionMemoryCompact | `sessionMemoryCompact.ts` | 会话记忆视角的压缩 |

- **预警与阈值**（`autoCompact.ts:57-143`）：`WARNING_THRESHOLD_BUFFER_TOKENS = 20_000`、`ERROR_THRESHOLD_BUFFER_TOKENS = 20_000`——阈值减 2 万 token 提前预警（compactWarningHook），再逼近才自动压缩；`CLAUDE_CODE_AUTO_COMPACT_WINDOW` 可覆盖窗口，`DISABLE_AUTO_COMPACT` 可关闭；剩余百分比实时展示给用户。
- **决策流**：达到 error 阈值 → 自动压缩 → 压缩结果替换历史 → 继续当前任务；压缩 prompt 见 `CCB: src/services/compact/prompt.ts`（⚠️ 摘要结构化章节待摘录）。
- **面试金句**：CC 的上下文管理不是"一个压缩按钮"，而是**预警 → 微压缩 → 剪切 → 全量压缩 → 被动压缩**的五级流水线，每级都在"信息损失 vs token 预算 vs cache 命中率"之间做不同取舍。

### 2.3 会话持久化与恢复 ⚠️

- jsonl 追加式落盘每条消息；`--resume`/`--continue` 重建；远程 ingress 有 409 冲突解决（采用服务器 lastUuid 后重试，`CCB: src/services/api/sessionIngress.ts:58-180`）。
- ⚠️ 待验证：本地存储路径与目录 hash 规则；fork 语义。

### 2.4 记忆体系 ⚠️

- CLAUDE.md 层级（§2.1）+ 自动记忆提取（feature `EXTRACT_MEMORIES`，穷鬼模式下跳过）+ SessionMemory（`CCB: src/services/SessionMemory/`）+ `/dream` 记忆整理。
- 面试点：**分层记忆 = 项目事实（CLAUDE.md）/ 会话记忆 / 自动提取记忆**三种生命周期。

---

## 3. L3 工具执行层（AI 的双手）

> CCB 文档站对应章节：`docs/tools/`；工具实现在 `CCB: packages/builtin-tools/src/tools/`（60 个工具目录）

### 3.1 工具执行管线 🟦

- **决策流**：模型产出 tool_use → 权限求值（§4.1）→ PreToolUse hook → 沙箱包装（§4.2）→ 执行 → 结果后处理（§3.6）→ PostToolUse hook → tool_result 追加 → 回主循环。
- **源码锚点**：`CCB: src/Tool.ts`（Tool 接口：`findToolByName`/`toolMatchesName`）；`CCB: src/tools.ts`（注册表，按 feature flag 条件加载）；`CCB: src/constants/tools.ts`（`CORE_TOOLS` 38 个白名单，配合 SearchExtraToolsTool 实现**延迟工具按需加载**——schema 不全量进上下文，省 token）。

### 3.2 超时 → 自动后台化 🟦（⭐ 你的四个机制之一）

- **触发**：Bash 命令执行超过 timeout。
- **决策流程**（`CCB: packages/builtin-tools/src/tools/BashTool/BashTool.tsx:1009-1134`）：
  1. `timeoutMs = timeout || getDefaultTimeoutMs()`，上限 `getMaxTimeoutMs()`；
  2. 超时**不是直接杀掉**——auto-backgrounding：`startBackgrounding('tengu_bash_command_timeout_backgrounded', backgroundFn)`，命令转入后台继续跑；
  3. 前台立即返回"已转后台"结果，AI 可以继续做别的事或轮询（§3.3）；
  4. 也可以主动 `run_in_background: true` 显式后台化。
- **面试金句**：超时处理的最优解常常不是"失败"，是**降级为异步**——把同步等待变成通知驱动。

### 3.3 后台任务 + task-notification 通知机制 🟦

- **触发**：后台任务完成/失败。
- **决策流程**：完成事件被包装成 `<task-notification>` XML（含 `task-id` / `tool-use-id` / `task-type` / `output-file` / `status` / `summary` 等字段，`CCB: src/constants/xml.ts:27-38`）注入到下一轮上下文，模型从而感知后台任务结果并决定是否收割（读输出文件/继续动作）。
- **设计意图**：通知是**数据流事件**而非控制流事件——不打断当前回合，只在回合边界进入模型视野。

### 3.4 并行工具调用 ⚠️

- 模型单轮可产出多个 tool_use；只读工具并发执行，写操作经文件状态跟踪串行化 + "read 后被修改"陈旧性检测。
- ⚠️ 待验证：CCB/官方并发度上限与冲突提示文案。

### 3.5 子 Agent（Task/Agent 工具）🟦

- **决策流**：AgentTool 派生独立上下文（子系统提示词 + 受限工具集）→ 子 agent 独立跑完循环 → **只回传最终摘要**给父会话（过程不占父上下文）；TaskCreate/TaskUpdate/TaskList/TaskGet 支撑任务清单协作；coordinator/swarm 支撑多 agent 编排（`CCB: docs/agent/coordinator-and-swarm`）。
- **面试点**：子 agent 本质是**上下文隔离的容量扩展**——用独立窗口换主窗口 token；代价是信息有损回传。

### 3.6 工具结果后处理 ⚠️

- 大输出截断（保头尾）、Read 分页、图片降采样；WebFetch 将 HTML 转 markdown 后由小模型按 prompt 提炼，原始网页内容不进主上下文。
- ⚠️ 待验证：各工具截断阈值常量。

---

## 4. L4 安全权限层

> CCB 文档站对应章节：`docs/safety/`（permission-model / sandbox / plan-mode / auto-mode）

### 4.1 权限模型与审批状态机 ⚠️

- **决策流**：每个工具调用 → 规则求值（deny > ask > allow；`Bash(git *)`、`Edit(docs/**)`、`WebFetch(domain:…)`、`mcp__*` 等模式）→ 权限模式修饰（default / acceptEdits / plan / bypassPermissions / auto-mode）→ ask 则弹出审批 UI（一次性允许 / 会话内允许 / 项目持久化 / 拒绝+反馈文本）。
- **源码锚点**：`CCB: src/types/permissions.ts`（Permission mode 与 result 类型）；`CCB: src/components/permissions/`（审批 UI）；`CCB: src/services/acp/permissions.ts`（ACP 远程权限管道 `createAcpCanUseTool`）。
- **面试点**：审批 UI 的每个按钮都是一条**规则生成器**——用户的一次选择被固化成未来会话的 allow 规则。

### 4.2 沙箱 ⚠️

- OS 级约束（macOS Seatbelt / Linux 对应机制）：工作区外只读、网络走代理白名单；沙箱内命令可默认放行（少弹窗），沙箱不可用时回退到权限询问（`CCB: packages/builtin-tools/src/tools/BashTool/shouldUseSandbox.ts`）。

### 4.3 Bash 命令解析与安全检查 🟦

- **文件清单**（`CCB: packages/builtin-tools/src/tools/BashTool/`）：`bashSecurity.ts`（注入检测）、`bashPermissions.ts`（权限规则匹配）、`commandSemantics.ts`（命令语义）、`sedEditParser.ts`/`sedValidation.ts`（sed 透写检测）、`destructiveCommandWarning.ts`（破坏性命令警告）、`pathValidation.ts`、`readOnlyValidation.ts`。
- **面试点**：把 shell 命令当**结构化输入解析**（复合命令拆分、重定向目标提取、子 shell 识别）来做权限预判，而不是当字符串。

### 4.4 Prompt Injection 防护 ⚠️

- 网页抓取内容/工具输出被标记为数据（caveat/system-reminder 包裹），WebFetch 经小模型转述隔离原始内容。
- ⚠️ 待验证：点式拒绝名单（shell 配置文件读取默认 ask 等）。

---

## 5. L5 扩展层

### 5.1 Hooks ⚠️

- 生命周期事件（PreToolUse / PostToolUse / UserPromptSubmit / Stop / PreCompact / SessionStart…）触发用户脚本；exit 2 = 阻断并把 stderr 回灌给模型；JSON 输出可结构化 allow/block。Hooks 是**确定性护栏**——不依赖模型自觉。
- 参考文档：`CCB: docs/extensibility/hooks`。

### 5.2 Skills / MCP / Plugins ⚠️

- Skills：SKILL.md 元数据常驻上下文，正文按需加载（渐进披露）；MCP 外部工具以 `mcp__server__tool` 接入；Plugins 打包以上能力 + hooks。
- 参考文档：`CCB: docs/extensibility/`（mcp-protocol / skills / custom-agents）。

---

## 6. 待验证清单（实验 + grep 计划）

| # | 机制 | 验证方式 | 关键词/入口 |
|---|---|---|---|
| 1 | 重试次数与退避曲线 | 读 `withRetry.ts` 常量 + mock 429 实验 | `getRetryDelay`、`MAX_529_RETRIES`、`tengu_api_retry` |
| 2 | 模型 fallback 触发条件 | grep + 触发 529 | `FallbackTriggeredError`、`fallbackModel` |
| 3 | compact 摘要结构 | 读 `src/services/compact/prompt.ts` | 八段式摘要章节 |
| 4 | autoCompact 具体阈值 | 读 `autoCompact.ts` + 观察 UI 百分比 | `getAutoCompactThreshold` |
| 5 | 中断文案双形态 | 读 `messages.ts` 常量 | `INTERRUPT_MESSAGE(_FOR_TOOL_USE)` |
| 6 | steering 入队时机 | 读 REPL.tsx 输入处理 | queue / pendingMessages |
| 7 | 抓包对照 | mitmproxy 挂官方 CLI | system prompt、beta headers、消息结构 |
| 8 | Bash 默认/最大超时常量 | grep `getDefaultTimeoutMs` 定义 | `BASH_DEFAULT_TIMEOUT_MS` 等环境变量 |

## 7. 横向对比占位（pi / dsh / codex）

> 对比表列 = 解剖卡维度（触发 / 决策 / 数据呈现 / 参数 / 边界降级 / 源码位置）。本机已有 checkout：`pi/`、`codex/`、`deepseek-harness/`。建议顺序：pi（最小参照系）→ dsh → codex。

| 机制 | Claude Code | pi | dsh | codex |
|---|---|---|---|---|
| API 重试与降级 | §1.2 五层降级瀑布 | 待填 | 待填 | 待填 |
| 工具超时/后台化 | §3.2 自动后台化 | 待填 | 待填 | 待填 |
| Compaction | §2.2 五级流水线 | 待填 | 待填 | 待填 |
| 用户中断 | §1.4 双形态合成消息 | 待填 | 待填 | 待填 |
| 权限/沙箱 | §4 | 待填 | 待填 | 待填 |

## 8. 参考

- CCB 架构文档站：https://ccb.agent-aura.top/ （本地 `claude-code-best/docs/`，Mintlify 配置 `docs.json`）
- Anthropic 工程博客《Effective harnesses for long-running agents》：https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- Claude Cookbook（context engineering）：https://platform.claude.com/cookbook/tool-use-context-engineering-context-engineering-tools
- 官方文档：permissions / hooks / sandboxing 章节

## 变更日志

- 2026-11-07：初版。建立分层骨架与 21 个机制解剖卡；L1/L2/L3 核心机制带 CCB 源码锚点；L4/L5 部分机制标记 ⚠️ 待验证。
