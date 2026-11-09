# Claude Code `queryLoop` 全流程拆解

> **定位**：把 `queryLoop()` 的一次迭代从头到尾拆开讲——先给一张总表，再逐段展开每一步"到底干了什么、影响什么"。
>
> **源码锚点**：`CCB: src/query.ts`，锚点基于 claude-code-best v2.1.888（Claude Code 的工程化复原项目，行为与官方高度一致但非逐字源码）。
> 行号会随代码演进漂移，按函数名/段落注释检索更稳。
>
> **在读之前需要建立的三个认知**：
> 1. `queryLoop` 是 **`while (true)` + `continue`**，不是递归。每次"续跑"都是 `state = next; continue`。
> 2. 一个"回合（turn）"不是一次 API 请求——**模型每次调用工具都会让循环再转一圈**。`turnCount` 统计的是后者。
> 3. 循环只有一个继续信号：**流里出现过 `tool_use` 块**（`needsFollowUp`）。`stop_reason` 字段不可靠，源码注释明确说了不要用它判断。

---

## 0. 总表：一次迭代的八个大段

> **读法**：加粗行是八个大段（`// ===== 一、… =====` 这样的段落注释），缩进行是段内的二级小节。
> 行号为该小节起始注释所在行。带 `feature()` 标记的小节是构建期门控，未启用时整段不存在。

| 段 / 小节 | 行号 | 干什么 |
|---|---|---|
| **一、初始化** | 457 | 解构 params、建跨轮 `state`、按 feature 加载可选模块、发起记忆预取 |
| 　1.1 解构 params | 458 | 拆出整场查询不变的 8 个参数 |
| 　1.2 跨轮状态 `State` | 471 | 建立 10 个字段的可变状态对象 |
| 　1.3 依赖注入 `QueryDeps` | 469 | 4 个可替换的 I/O 依赖（`callModel` 等） |
| 　1.4 记忆预取 | 496 | `using` 管理生命周期，后台查记忆 |
| **二、消息预处理管道** | 569 | ① ~ ⑦ 逐层收缩上下文，交给 API |
| 　① 压缩边界切片 | 571 | 只保留最后一个压缩边界之后的消息 |
| 　② 剥离 `toolUseResult` | 573 | 删 UI 用的原始负载，防内存涨 |
| 　③ 工具结果预算 | 601 | 超大工具结果换成引用并持久化 |
| 　④ Snip 按条裁剪 | 615 | `HISTORY_SNIP`：剪掉可牺牲的整条旧消息 |
| 　⑤ 微压缩 | 629 | 旧工具结果清成占位符 |
| 　⑥ 上下文折叠 | 656 | `CONTEXT_COLLAPSE`：折叠视图投影（本仓库为 stub） |
| 　组装系统提示词 | 679 | 系统提示 + `systemContext` 每轮重拼 |
| 　⑦ 自动压缩 | 684 | 唯一需要真调模型的一步 |
| **三、响应收集器** | 786 | 建容器、选执行器、选模型、做阻塞/预测性检查 |
| 　3.1 三个容器 + 一个标志 | 789 | `assistantMessages` / `toolResults` / `toolUseBlocks` / `needsFollowUp` |
| 　3.2 选执行器与选模型 | 798 | 流式执行器开关、运行时模型决策 |
| 　3.3 请求体转储 | 821 | 仅 `isAnt`，只留最近一份防 500MB 泄漏 |
| 　3.4 硬性阻塞上限 | 833 | 自动压缩关闭时给手动 `/compact` 留空间 |
| 　3.5 预测性自动压缩 | 896 | 估算本轮增长会顶爆窗口就提前压 |
| **四、调模型并流式消费** | 939 | `callModel()` 起流，处理降级、暂扣错误、边流边执行工具 |
| 　4.1 流式降级 | 1002 | 半成品作废、孤儿消息墓碑化 |
| 　4.2 回填 tool_use 输入 | 1038 | 在克隆消息上做，不污染历史 |
| 　4.3 可恢复错误暂扣 | 1098 | 413 / 媒体超限 / 输出截断先不外抛 |
| 　4.4 收集与边流边执行 | 1139 / 1166 | 收到 `tool_use` 就提前开工 |
| 　4.5 缓存式微压缩收尾 | 1186 | 用 API 实报的删除量补边界消息 |
| 　4.6 模型降级重试 | 1216 | `FallbackTriggeredError` 切备用模型整轮重放 |
| 　4.7 调用失败兜底 | 1281 | 补齐缺失的 `tool_result` |
| 　4.8 post-sampling 钩子 | 1355 | 异步不阻塞，挂记忆提取等 |
| 　4.9 流式中断 | 1368 | 消费完剩余结果再收尾，防泄漏 |
| 　4.10 上轮工具摘要 | 1415 | Haiku 生成，纯 UI 增强 |
| **五、回合结束分支** | 1423 | `!needsFollowUp` 时的全部收尾与恢复 |
| 　5.1 恢复 413 / 媒体超限 | 1428 | 先折叠排水，再响应式压缩，各一次 |
| 　5.2 输出截断恢复 | 1551 | 8k→64k 升级重试，再多轮恢复（上限 3） |
| 　5.3 API 错误收尾 | 1626 | 跳过 Stop 钩子 |
| 　5.4 Stop 钩子 | 1640 | 可拦截结束，也可注入阻塞错误重跑 |
| 　5.5 Token 预算续跑 | 1683 | `TOKEN_BUDGET`：未达预算时注入推进元消息 |
| **六、工具执行** | 1737 | 消费流式执行器剩余结果，或走 `runTools()` |
| 　6.1 统一出口 | 1758 | 两条路径共用一个消费循环 |
| 　6.2 结果合并 | 1762 | `message` / `contextModifier` / 钩子叫停 |
| 　6.3 工具使用摘要 | 1789 | 异步发起，不阻塞下次 API 调用 |
| 　6.4 执行期中断 | 1869 | 清理 CU 状态、下发中断消息 |
| 　6.5 压缩后轮次埋点 | 1911 | `turnCounter++`，供 `turnsSincePreviousCompact` |
| **七、排队命令与附件注入** | 1936 | 捞队列命令、消费预取、刷新工具 |
| 　7.1 队列命令 | 1957 | 进程级单例，只捞发给自己的 |
| 　7.2 记忆预取消费 | 2004 | 零等待，没就绪下轮再试 |
| 　7.3 技能 / 额外工具发现 | 2030 / 2044 | 延迟工具的"兑现场" |
| 　7.4 队列认领 | 2058 | 只移除真正被消费的命令 |
| 　7.5 附件规模埋点 | 2077 | 注入前后对比统计 |
| 　7.6 轮间刷新工具 | 2092 | 新连的 MCP server 无需重启即可用 |
| 　7.7 后台任务摘要 | 2115 | `BG_SESSIONS`：仅主线程生成 |
| 　7.8 轮次上限 | 2139 | 超 `maxTurns` 以 `max_turns` 结束 |
| **八、组装下一轮** | 2150 | `messages = 历史 + 助手 + 工具结果`，重写 `state`，`continue` |

**控制流只有三种出口**：

| 出口 | 类型 | 触发点 |
|---|---|---|
| `return { reason: ... }` | `Terminal`（10 种） | 正常完成、被拦、出错、超轮次 |
| `continue` | `Continue`（7 种） | 工具续跑、各类恢复重试、预算续跑 |
| 落到段八 | —— | 唯一一条"正常前进"的路径 |

`Terminal` 与 `Continue` 的全集在 `CCB: src/query/transitions.ts:1-20`，值得单独看一眼——**它们合起来就是这个循环完整的状态机**。

**附录两节**（不属于循环本体，但与读代码相关）：

| 节 | 行号 | 内容 |
|---|---|---|
| 9. 三个容易踩的坑 | 561 | UI≠模型视野、`query()` 与循环的边界、恢复路径的顺序约束 |
| 10. 关联文档 | 579 | 指向 `mechanisms.md` 的压缩家族 / 工具管线 / Hooks |

---

## 1. 第一大段：初始化（457）

### 1.1 解构 params

`query()` 的入参 `QueryParams`（`CCB: src/query.ts:266`）分成两类：

- **整场查询不变**：`systemPrompt`、`userContext`、`systemContext`、`canUseTool`、`fallbackModel`、`querySource`、`maxTurns`、`skipCacheWrite`。源码注释写着 `Immutable params — never reassigned during the query loop.`
- **跨轮可变**：装进 `state` 对象。

**影响**：`systemPrompt` 在整场查询里是常量，所以如果你在会话中途改了系统提示词（比如切 output style），那是**下一个 `query()` 调用**才生效，不是下一次循环。

### 1.2 跨轮状态 `State`

```ts
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

（`CCB: src/query.ts:291`）

几个字段的用途值得单独记：

- **`transition`** — 记录"上一轮是因为什么继续的"。它是**防死循环的关键**：比如 413 恢复时，如果上一轮已经试过折叠排水（`collapse_drain_retry`），这一轮就不能再试，必须往下走响应式压缩。没有这个字段，两个恢复路径会互相踢皮球转不出来。
- **`maxOutputTokensRecoveryCount`** — 输出截断的多轮恢复计数，上限 3 次。
- **`hasAttemptedReactiveCompact`** — 响应式压缩只有一次机会，这个布尔量就是那个"一次"。
- **`stopHookActive`** — Stop 钩子已经拦过一次，防止"钩子拦截 → 重跑 → 钩子又拦"的螺旋。

### 1.3 依赖注入 `QueryDeps`

`CCB: src/query/deps.ts:21`。只有 4 个：`callModel`、`microcompact`、`autocompact`、`uuid`。

**为什么要这层**：注释写得很直白——`callModel` 和 `autocompact` 各自在 6~8 个测试文件里被 `spyOn`，样板代码太多。改成 `deps` 注入后，测试直接传假实现。范围刻意做窄（就 4 个），是个"试点"，后续 PR 再扩。

**影响**：读测试代码时如果看到 `deps: { autocompact: ... }`，那是测试替身，不是生产路径。

### 1.4 记忆预取

```ts
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(state.messages, state.toolUseContext)
```

（`CCB: src/query.ts:496`）

用 `using`（TS 的显式资源管理）保证生成器的**所有退出路径**都会正确释放——包括中途 `return`、抛异常、被外部 `return()` 提前终止。预取结果在第七段消费。

**影响**：这是一个"用时间换 token"的设计——在模型流式输出期间后台查记忆，等真正要用时零等待。没就绪就不消费，下轮再试。

---

## 2. 第二大段：消息预处理管道（569）

这一段是整条链路里最容易讲不清的部分。它的本质是**七道筛子，越往下越贵**：

| 步骤 | 行号 | 成本 | 一句话 |
|---|---|---|---|
| ① 压缩边界切片 | 571 | 零 | 只留最后一次压缩之后的消息 |
| ② 剥离 toolUseResult | 573 | 零 | 删 UI 用的原始负载，防内存涨 |
| ③ 工具结果预算 | 601 | 零 | 超大工具结果换成引用 |
| ④ Snip 按条裁剪 | 615 | 本地 | 剪掉可牺牲的整条旧消息 |
| ⑤ 微压缩 | 629 | 本地 | 旧工具结果清成占位符 |
| ⑥ 上下文折叠 | 656 | 读时投影 | 旧区段折成摘要送上请求 |
| ⑦ 自动压缩 | 684 | **要调模型** | 前六步都不够才真总结 |

关键关系：**前六步只改内存里的数据结构，一个 token 都不发给模型；第 ⑦ 步的阈值判断必须扣掉前六步已经省下的量**——否则会误判"还没到阈值"从而该压不压。

### ① 压缩边界切片

```ts
let messagesForQuery = getMessagesAfterCompactBoundary(messages)
```

`CCB: src/utils/messages.ts:5083`。找到最后一个"压缩边界"标记，把之前的消息整段丢掉。开了 `HISTORY_SNIP` 时还会应用 `projectSnippedView()`，把 snip 标记过的消息投影掉。

**影响**：**你在 UI 里看到的历史 ≠ 模型看到的历史。** REPL 保留完整对话用于滚动回看和 `/resume`，但发给模型的只有边界之后的部分。这是设计如此，不是 bug。

### ② 剥离 `toolUseResult` 原始负载

工具执行完后，一条用户消息上会挂两份数据：

- `message.content` 里的 `tool_result` 块 —— **发给模型的**
- `message.toolUseResult` —— **给 UI 渲染的原始对象**

这一步把后者删掉。**必须是浅拷贝后删，不能原地删**（`CCB: src/query.ts:573` 处的注释解释了原因）：

> 消息与 UI 状态共享引用。原地 `delete` 会把正在渲染的工具结果行变成空白——因为下一轮 query 可能在 tool_result 创建后**几毫秒内**就启动，早于 React 的 commit。

**影响**：单个 `Read` 返回 400KB 文件的话，这份内容若一直挂在 `mutableMessages` 上，长会话内存会无限涨。这一步是内存曲线的关键拐点。

### ③ 工具结果预算

`applyToolResultBudget()`（`CCB: src/utils/toolResultStorage.ts:925`）。单条消息里工具结果体积超预算时，把大块内容替换成引用，**原文持久化到会话记录**（jsonl），需要时能读回。

**顺序讲究**：它排在微压缩**之前**。原因在 `CCB: src/query.ts:593` 的注释里——缓存式微压缩**纯按 `tool_use_id` 操作，从不检查内容**，所以内容替换对它完全透明，两者可以干净组合。

**影响**：长会话里看到的"内容已持久化到 transcript"提示来自这里。另外只有 `repl_main_thread*` 和 `agent:*` 会持久化替换记录（因为只有它们会在 resume 时读回），临时的 fork agent 不持久化。

### ④ Snip 按条裁剪（`HISTORY_SNIP` 门控）

`snipCompactIfNeeded()`（`CCB: src/services/compact/snipCompact.ts:83`）。跟 ① 的区别是：① 按**边界**切（一次切一大段），④ 按**条**挑（逐条挑可牺牲的），在裁剪位置留一个 `snip_boundary` 消息记录删了哪些 UUID。

**影响**：产出 `snipTokensFreed`，一路传到 ⑦。这是必须的——token 估算读的是 usage 字段，而 snip 删掉的内容在 usage 里看不出来，不扣除就会误判"还没到阈值"。

### ⑤ 微压缩

`microcompactMessages()`（`CCB: src/services/compact/microCompact.ts:257`）。**只动工具结果，不动对话本身**：把旧的读文件、命令输出换成 `[Old tool result content cleared]`。

两种触发：
- **时间触发优先** —— 距上条助手消息太久，服务端缓存已冷，反正整段前缀都要重写，那不如现在就把它变小
- **量触发** —— 按工具结果累计体积

**作用范围**：只有白名单工具（`Read`、`Bash`/shell 系、`Grep`、`Glob`、`WebSearch`、`WebFetch`、`Edit`、`Write`）。

**影响**：模型会看不见早期的工具输出，但对话主线完整保留。这是"小步快跑"式的省 token。

### ⑥ 上下文折叠（`CONTEXT_COLLAPSE` 门控）

`applyCollapsesIfNeeded()`（`CCB: src/services/contextCollapse/index.ts:52`）。理念跟其他压缩器不同：**原始消息不删**，只把旧区段折成摘要"投影"给请求，本地仍保留原文，需要时能展开。本质是"读时投影"——摘要不在 REPL 数组里，而在折叠存储里，每次进入都重放提交日志。

**为什么排在 ⑦ 之前**：如果折叠已经把上下文降到阈值以下，⑦ 就是空操作，从而保住细粒度上下文，而不是被压成一条摘要。

> ⚠️ **本文档的实测提醒**：在 claude-code-best 仓库里，这个模块是 **stub**——`CCB: src/services/contextCollapse/index.ts` 文件开头就写着 `Auto-generated stub — replace with real implementation`，`applyCollapsesIfNeeded` 原样返回入参。所以本地跑起来这一步是空操作。官方版本里它是完整实现。

### 组装系统提示词

```ts
const fullSystemPrompt = asSystemPrompt(appendSystemContext(systemPrompt, systemContext))
```

（`CCB: src/query.ts:679`）

**位置值得注意**：它在管道中间、⑦ 之前。因为 `systemContext`（git 状态、日期等）每轮都可能变，必须每轮重拼。

### ⑦ 自动压缩

`deps.autocompact()` → `autoCompactIfNeeded()`（`CCB: src/services/compact/autoCompact.ts:270`，已补中文注释）。

两条路径：先试 `trySessionMemoryCompaction()`（记忆型裁剪，保留细粒度），不行再走 `compactConversation()`（传统全量摘要）。

**熔断器**：连续失败 3 次后不再尝试。状态不放在函数里，而是通过返回值的 `consecutiveFailures` 交给调用方写回 `tracking`（`CCB: src/query.ts:770`），因为函数本身无状态。

**压缩成功后的动作**（`CCB: src/query.ts:704`）：
- 上报 `tengu_auto_compact_succeeded` 埋点
- 重置 `tracking`（`consecutiveFailures: 0`、`turnCounter: 0`、换新 `turnId`）
- `buildPostCompactMessages()` 转成新消息数组，逐条 `yield` 给 UI
- **用压缩后的消息继续本轮请求**

**影响**：压缩后上一轮的工具结果、文件读取内容会让模型"失忆"。摘要只保留结论性信息。这就是为什么长会话里模型有时会重复读同一个文件。

---

## 3. 第三大段：响应收集器（786）

### 3.1 三个容器 + 一个标志

```ts
const assistantMessages: AssistantMessage[] = []
const toolResults: (UserMessage | AttachmentMessage)[] = []
const toolUseBlocks: ToolUseBlock[] = []
let needsFollowUp = false
```

`needsFollowUp` 是**唯一的循环继续信号**：`CCB: src/query.ts:792` 的注释明确写着 `stop_reason === 'tool_use' is unreliable`。

**影响**：这是理解控制流的关键。判断"要不要继续"只看"流里有没有出现过 `tool_use` 块"，不看 API 返回的 `stop_reason`。

### 3.2 选执行器与选模型

```ts
const useStreamingToolExecution = config.gates.streamingToolExecution
let streamingToolExecutor = useStreamingToolExecution ? new StreamingToolExecutor(...) : null
```

（`CCB: src/query.ts:800`）

`currentModel = getRuntimeMainLoopModel({ permissionMode, mainLoopModel, exceeds200kTokens })`。

**影响**：模型不是固定的——`plan` 模式下如果最近一条助手消息超过 200k tokens，会切到别的模型。这是运行时决策。

### 3.3 请求体转储

`createDumpPromptsFetch()`，仅 `config.gates.isAnt` 生效。

**为什么要"只创建一次"**：注释说每次调用都会捕获请求体闭包，逐请求创建会让整个会话的请求体（长会话约 500MB）全都留在内存里。只创建一次就只保留最近一份（约 700KB）。

### 3.4 硬性阻塞上限（仅自动压缩关闭时）

为手动 `/compact` 预留空间，超限直接下发 prompt-too-long 并 `return { reason: 'blocking_limit' }`。

跳过条件（`CCB: src/query.ts:833` 注释）：刚压缩过 / snip 刚省出空间（旧 usage 已过期会误判）/ `compact` 和 `session_memory` 这类派生 agent（越压越少，阻塞会死锁）。

### 3.5 预测性自动压缩

```ts
if (!compactionResult && isAutoCompactEnabled()) {
  const estimatedGrowth = estimateMaxTurnGrowth(model)
  const predictiveThreshold = getEffectiveContextWindowSize(model) - estimatedGrowth
  if (currentTokens > predictiveThreshold) { /* 提前压 */ }
}
```

（`CCB: src/query.ts:896`）

**意图**：抢在请求发出前腾空间，避免撞 413。用 `effectiveContextWindow` 直接算而不走 `getAutoCompactThreshold`，是为了避免和已经减过 buffer 的阈值重复扣减。

**影响**：这一处是"花了钱但省了更大的钱"——主动压一次，好过被 413 打回来再压。

---

## 4. 第四大段：调模型并流式消费（939）

这一段最长（约 480 行），但主干只有一条：`deps.callModel()` 返回异步流，`for await` 逐事件处理。

请求体组装的细节（`CCB: src/query.ts:952`）：

```ts
messages: prependUserContext(messagesForQuery, userContext),
systemPrompt: fullSystemPrompt,
tools: toolUseContext.options.tools,
signal: toolUseContext.abortController.signal,
```

**注意 `messages` 用的是局部变量 `messagesForQuery`，不是 `toolUseContext.messages`** —— 后者要到 `CCB: src/query.ts:783` 才被同步成前者的副本，纯粹给工具用。

### 4.1 流式降级（1002）

`streamingFallbackOccured` 为真时，之前的半成品全部作废：
- 已产出的助手消息 `yield { type: 'tombstone' }` 让 UI 移除（**因为部分消息里的 thinking 块签名无效，留着会触发 "thinking blocks cannot be modified" 报错**）
- 清空收集器、丢弃未完成工具结果、重建执行器

### 4.2 回填 tool_use 输入（1038）

在**克隆消息**上调用 `backfillObservableInput()`，再下发。

**影响**：不改原始消息，UI 与 API 看到的是两个版本。派生字段回填不会污染历史。

### 4.3 可恢复错误暂扣（1098）

三类错误先**扣住不外抛**，给恢复逻辑留机会：

- 413 prompt-too-long
- 媒体超限（图片/PDF）
- 输出截断 `max_output_tokens`

**为什么叫"暂扣"**：如果直接 `yield` 出去，UI 会显示错误，且第五段没机会介入恢复。

### 4.4 收集与边流边执行（1139 / 1166）

每收到一个 `tool_use` 块就置 `needsFollowUp = true`，并在**未中止**时立刻交给流式执行器提前开工；流式执行器里已完成的工具结果**立即 yield 并入列**。

**影响**：这是"边流边执行"的来源——用户能在模型还没说完时就看到工具结果。

### 4.5 缓存式微压缩收尾（1186）

用 API **实报的** `cache_deleted_input_tokens` 补发边界消息。

**为什么要等 API 回报**：本地估算删了多少不准，埋点得用真实数字。

### 4.6 模型降级重试（1216）

`FallbackTriggeredError` → 切备用模型，**整轮重放**。

**影响**：用户可能看到模型名悄悄变了（Opus → Sonnet）。这是外层五层降级瀑布的最后一层，触发条件是连续 529 达到上限。

### 4.7 调用失败兜底（1281）

补齐缺失的 `tool_result`。

**为什么必须补**：API 要求每个 `tool_use` 都有对应的 `tool_result`，缺一个整段对话就结构非法，下一次请求直接报错。

### 4.8 post-sampling 钩子（1355）

`executePostSamplingHooks()`，**不阻塞主流程**（`void` 调用）。

**挂在这里的功能**：记忆提取（`extractSessionMemory`）、技能学习观测（`skillLearning`）、技能改进（`skillImprovement`）。这也是内部那个"可注册回调数组"的唯一对外近似物。

### 4.9 流式中断（1368）

`toolUseContext.abortController.signal.aborted` 时最先处理：消费完流式执行器剩余结果（**不消费会泄漏**），清理 CU 状态，下发中断消息，`return { reason: 'aborted_streaming' }`。

### 4.10 上轮工具摘要（1415）

上一轮异步发起的 Haiku 摘要（约 1s）此刻应已就绪，顺路 `yield` 出去。

**影响**：纯 UI 增强，不参与模型上下文。

---

## 5. 第五大段：回合结束分支（1423）

**只有 `!needsFollowUp` 时才进这里**——模型没再调工具，说明它想结束。但"想结束"不等于"能结束"，这一段是五道闸依次检查：

### 5.1 恢复 413 / 媒体超限（1428）

顺序**固定**：

1. 先试**折叠排水** `contextCollapse.recoverFromOverflow()` —— 便宜，保留细粒度
2. 再试**响应式压缩** —— 全摘要，`hasAttemptedReactiveCompact` 保证只有一次

各只有一次机会。仍失败则 `yield` 错误、跑 `executeStopFailureHooks()`、`return`。

**关键设计**：失败后**不往下走 Stop 钩子**。注释解释了原因（`CCB: src/query.ts:1537`）：

> 模型根本没产出有效响应，钩子没有可评估的东西。在 prompt-too-long 上跑 Stop 钩子会造成死亡螺旋：错误 → 钩子拦截 → 重试 → 错误……**钩子每轮还往里注入更多 token**。

### 5.2 输出截断恢复（1551）

两级：
1. **升级重试** —— 8k 打到上限就用 64k 重放**同一个请求**，不加元消息、不走多轮
2. **多轮恢复** —— 注入"从中断处继续写，别道歉别回顾"的元消息，上限 `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT`（3）

升级重试只在 `tengu_otk_slot_v1` 开、且用户没自己设 `CLAUDE_CODE_MAX_OUTPUT_TOKENS` 时生效；3P（Bedrock/Vertex）默认关闭因为没验证过。

### 5.3 API 错误收尾（1626）

跳过 Stop 钩子——理由同 5.1。

### 5.4 Stop 钩子（1640）

`handleStopHooks()` 两个出口：

- `preventContinuation` → `return { reason: 'stop_hook_prevented' }`
- `blockingErrors` → 注入错误消息，`continue`（`transition: 'stop_hook_blocking'`，`stopHookActive: true`）

**一个已修复的坑**（`CCB: src/query.ts:1670` 注释）：注入阻塞错误时**不能把 `hasAttemptedReactiveCompact` 重置为 false**。否则会形成 `压缩 → 还是太长 → 报错 → Stop 钩子拦截 → 又压缩 → …` 的无限循环，烧掉成千上万次 API 调用。

### 5.5 Token 预算续跑（1683，`TOKEN_BUDGET` 门控）

`checkTokenBudget()` 三种结局：`continue`（注入推进元消息接着跑）、`completionEvent`（记录完成，含"边际递减"早停）、两者都无 → `return { reason: 'completed' }`。

**注意**：`return { reason: 'completed' }` 是这一段**唯一**的正常结束出口。走到这里说明所有恢复逻辑都认输了，模型正常结束。

---

## 6. 第六大段：工具执行（1737）

### 6.1 统一出口

```ts
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

两条路径，**同一个消费循环**。流式执行器里没跑完的在这里收尾，非流式路径就是常规编排。

### 6.2 结果合并

每个 `update` 里可能带：`message`（yield 给 UI）、`contextModifier`（修改 `toolUseContext`）、`hook_stopped_continuation`（钩子叫停）。

**影响**：工具的 `contextModifier` 能改上下文——这是工具影响后续行为的唯一正当通道。

### 6.3 工具使用摘要（1789）

这批发起的摘要**异步跑、不阻塞**下一次 API 调用，下次循环 4.10 处收。

### 6.4 执行期中断（1869）

清理 CU 状态、下发中断消息，按需上报，`return { reason: 'aborted_tools' }`。

### 6.5 压缩后轮次埋点（1911）

`tracking?.compacted` 为真时 `turnCounter++` 并上报 `tengu_post_autocompact_turn`。

**影响**：纯统计，用来算"压缩后还能撑几轮"。`autoCompactIfNeeded` 的 `recompactionInfo.turnsSincePreviousCompact` 就是从这个计数来的。

---

## 7. 第七大段：排队命令与附件注入（1936）

这一段做的事：**把各种异步产物在进入下一轮前塞进消息列表**。

### 7.1 队列命令

```ts
const sleepRan = toolUseBlocks.some(b => b.name === SLEEP_TOOL_NAME)
const isMainThread = querySource.startsWith('repl_main_thread') || querySource === 'sdk'
```

队列是**进程级单例**，协调者和所有 in-process 子 agent 共享。每个循环只捞**发给自己的**：主线程捞 `agentId === undefined`，子 agent 捞自己的 id。

**几个排除规则**：
- **斜杠命令排除** —— 必须走回合后的 `processSlashCommand`，不能当文本发给模型
- **用户 prompt 只给主线程** —— 子 agent 永远看不到 prompt 流
- **Bash 模式命令已在更上游排除**

### 7.2 记忆预取消费（2004）

只消费**已就绪且从未消费过**的，零等待。没就绪就下轮再试。

### 7.3 技能发现（2030）/ 额外工具发现（2044）

把 `skill_discovery` / `tool_discovery` 附件注入下一轮上下文。

**影响**：这是"延迟工具按需加载"的兑现点——模型在本轮表达了需要某工具，下一轮这个工具的描述就出现在上下文里。

### 7.4 队列认领（2058）

从队列移除**真正被消费的**命令，回调生命周期 `started`。

**为什么"真正被消费"才移除**：可能构造了附件但最终没用上，那命令不能丢。

### 7.5 附件规模埋点（2077）

`tengu_query_before_attachments` / 附件注入后的对比统计。

### 7.6 轮间刷新工具（2092）

```ts
if (updatedToolUseContext.options.refreshTools) {
  const refreshedTools = updatedToolUseContext.options.refreshTools()
  ...
}
```

**影响**：本轮期间新连接的 MCP server 下一轮就能用，**不需要重启会话**。

### 7.7 后台任务摘要（2144，`BG_SESSIONS` 门控）

仅主线程（`!toolUseContext.agentId`）生成，`claude ps` 看到的"正在做什么"就来自这里。子 agent 和 fork 不参与。

### 7.8 轮次上限（2139）

```ts
if (maxTurns && nextTurnCount > maxTurns) { ... return { reason: 'max_turns', turnCount: nextTurnCount } }
```

**兜底机制**，防止工具调用无限循环把预算烧光。

---

## 8. 第八大段：组装下一轮（2150）

```ts
const next: State = {
  messages: messagesForQuery.concat(assistantMessages, toolResults),
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,        // 重置
  hasAttemptedReactiveCompact: false,     // 重置
  pendingToolUseSummary: nextPendingToolUseSummary,
  maxOutputTokensOverride: undefined,     // 重置
  stopHookActive,
  transition: { reason: 'next_turn' },
}
state = next
```

**三个"重置"字段的语义**：它们都是"单回合只允许一次"的守卫，跨回合必须清零。`maxOutputTokensRecoveryCount`（截断恢复）、`hasAttemptedReactiveCompact`（响应式压缩）、`maxOutputTokensOverride`（64k 升级）。

**注意 `stopHookActive` 没有重置** —— 它是被显式传递的，因为 Stop 钩子可能需要跨多轮才能解除。

`transition: { reason: 'next_turn' }` 是唯一一个"正常前进"的 transition 值，其余六个都是某种恢复。

---

## 9. 三个容易踩的坑

### 9.1 UI 看到的消息 ≠ 模型看到的消息

链条是：`messages`（REPL 完整历史）→ ①边界切片 → ②剥负载 → ③预算 → ④snip → ⑤微压缩 → ⑥折叠 → ⑦压缩 → `messagesForQuery`。

任一步都可能删掉内容。**排查"模型为什么不记得"时，要按这条链从后往前找。**

### 9.2 循环会转很多圈，但只有一次 `query()` 调用

`/compact`、`/clear` 都是在**下一个 `query()` 调用**生效，不在下一次循环生效。因为它们在段一的 `Immutable params` 之外。

### 9.3 恢复路径才是复杂度所在

八个大段里，段五（回合结束）的代码量不到 330 行，却塞了五条恢复路径，且**顺序不可交换**、**各自有一次性守卫**、**失败后不能往下走 Stop 钩子**。要改这个循环，先读 `CCB: src/query/transitions.ts` 里的 `Terminal` / `Continue` 全集——那 27 行就是整个状态机的宪法。

---

## 10. 与本文档其他内容的关联

- **压缩家族的完整设计**（auto / micro / snip / reactive / sessionMemory 五种分工）：见本目录 `mechanisms.md §2.2`
- **工具执行管线**（权限 → 沙箱 → 执行 → 后处理）：见 `mechanisms.md §3.1`
- **Stop 钩子与 Hooks 体系**：见 `mechanisms.md §5.1`
- **自动压缩的函数级中文注释**：`CCB: src/services/compact/autoCompact.ts:270`
- **Hook 事件机制**（27 种事件 / 6 种实现类型 / 4 个来源）：`CCB: docs/extensibility/hooks.mdx`
