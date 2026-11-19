# 1. 上下文 · 机制详情

> 对应 [../harness-matrix.md](../harness-matrix.md) 第 1 层信号矩阵。字段为行、harness 为列，同字段跨 harness 对齐。
>
> **算例基准**：统一 ContextWindow = 1M、输出预留默认 8k；随窗口变化的用表格逐行列出。

---

## 压缩流水线分级（四级）

> 核对日期：2026-11-16。原版依据：v2.1.88 还原源码（🟦）+ v2.1.325 抓包与本机二进制考证（🟦实测）；CCB 复原源码辅助（🟦）。宣传图（"Snip→Microcompact→Context Collapse→Autocompact"）描述的是 **ant 内部构建**；外部 npm/安装包构建经死代码剔除后只保留第 4 级（+reactive 兜底），见各级"外部可用性"。

每轮 API 请求前在 `query.ts` 主管道按固定顺序执行四级（v2.1.88 [query.ts:396-468](../../claude-code-sourcemap/restored-src/src/query.ts)）：**Snip → Microcompact → Context Collapse → Autocompact**。注释写明排序理由：collapse 排在 autocompact 之前，若折叠把 token 压到阈值下，autocompact 空转，保住细粒度上下文。

| 级 | 触发条件 | 执行机制 | 调 LLM？ | 外部构建（2.1.325 实测） |
|---|---|---|---|---|
| **1 Snip 剪裁** | 模型自主调用 Snip 工具（API 消息注入 `[id:xxxxxx]` 短标签供引用；每增长约 10k token 且无 snip 时注入 `context_efficiency` 轻推）；或 `/force-snip` 手动 | 每轮 `snipCompactIfNeeded()` 按边界消息的 `snipMetadata.removedUuids` **本地过滤**消息，UI 保留全量 | 否 | ❌ 不在构建里。2.1.325 二进制中 `HISTORY_SNIP`/`SnipTool`/`snipCompact` 等标识计数全为 0——ant-only（`excluded-strings.txt` + Bun DCE），非"默认关" |
| **2 Microcompact 微压缩** | ① 时间触发：距上条主线程 assistant 消息 >60min（GrowthBook `tengu_slate_heron`，**默认 `enabled:false`**）；② 缓存编辑触发（ant-only `CACHED_MICROCOMPACT`）：可压缩工具结果活跃数超阈值 10、保留最近 5 | ① 旧 tool_result 内容本地替换为 `[Old tool result content cleared]`；② API 请求附 `cache_edits` 块由服务端删除，本地消息不动 | 否 | ❌ ① 默认关，② ant-only。外部仅有 API 原生 `clear_thinking_20251015`（keep all）——2.1.325 抓包 `context_management` 字段实证 |
| **3 Context Collapse 折叠** | 以 effectiveWindow 为分母：**约 90% 提交折叠、约 95% 阻塞并派生**（v2.1.88 autoCompact.ts 注释明文）；真实 413 时先 `recoverFromOverflow` 排空暂存 | 读时投影：原始消息数组与 transcript 不动，替换指令（span 首尾 uuid + `<collapsed>摘要</collapsed>` 占位文本）存独立 store 并持久化（`marble-origami-commit` entry），`projectView()` 每轮重放 commit log 把 span 内 N 条消息顶替成 1 条占位。**摘要生成需 LLM**（fork ctx-agent `marble_origami` 写摘要入 staged 队列），**替换执行不需要**（~90% 时本地 splice 提交）。span 不按轮数，按 token 增长 + `risk` 评分动态选连续区段。启用期间抑制 autocompact（`shouldAutoCompact` 直接 false） | **是**（摘要生成；替换无） | ❌ 不在构建里。二进制无 `CONTEXT_COLLAPSE`/collapse 符号；CCB 中该 feature 在禁用列表、模块为 stub |
| **4 Autocompact 全量摘要** | `tokenCountWithEstimation(messages) − snipTokensFreed ≥ effectiveWindow − 13,000`；前置否决：`DISABLE_COMPACT`/`DISABLE_AUTO_COMPACT`/用户配置、fork 源递归守卫、collapse 激活、熔断器（连续失败 3 次停试） | 先试 session memory 压缩（不调 LLM，复用已抽取记忆）；失败则 fork compact agent（`maxTurns:1`，复用主线程 prompt cache）生成 9 节结构化摘要 | **是** | ✅ 唯一默认完整保留的一级；外加 reactive compact 兜底（真实 413 后触发，同级摘要） |

**外部用户的实际视图（2.1.325）**：主动压缩只有 Autocompact；413 兜底走 reactive compact；"微压缩"只剩 API 原生 thinking 清理。宣传图的四级是 ant 内部形态。

关键代码锚点：主管道顺序 [query.ts:396-468](../../claude-code-sourcemap/restored-src/src/query.ts) · snip 过滤 CCB [snipCompact.ts:83-147](../../claude-code-best/src/services/compact/snipCompact.ts) · microcompact 双路径 [microCompact.ts:253-293](../../claude-code-sourcemap/restored-src/src/services/compact/microCompact.ts) · collapse 抑制 autocompact [autoCompact.ts:201-223](../../claude-code-sourcemap/restored-src/src/services/compact/autoCompact.ts) · autocompact 阈值与 fork [autoCompact.ts:160-239](../../claude-code-sourcemap/restored-src/src/services/compact/autoCompact.ts) + [compact.ts:1188-1215](../../claude-code-sourcemap/restored-src/src/services/compact/compact.ts) · 摘要 prompt [prompt.ts:61-143](../../claude-code-sourcemap/restored-src/src/services/compact/prompt.ts)。

⚠️ 待验证：v2.1.325 中 collapse/snip 的 DCE 结论基于二进制字符串计数（0 命中），未反编译控制流；"87% 触发"非代码常量，200K 窗口按公式为 83.5%。

---

## 自动压缩阈值

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | `阈值 = 有效窗口 − 缓冲`<br>`有效窗口 = 上下文窗口 − min(模型输出上限, 20,000)`<br>原版：缓冲固定 13,000<br>CCB：缓冲按有效窗口三档 | `tokens > 窗口 − reserveTokens(16,384)`（`shouldCompact()`，一行判定）<br>另设 `keepRecentTokens: 20,000` 控制压缩后保留量 | `totalTokens ≥ min(窗口 × 0.8, pressureBudget)`<br>`pressureBudget = 消息预算 − reservedCompletion − headroom` | — |
| **值** | 原版：`AUTOCOMPACT_BUFFER_TOKENS = 13,000` 固定，不分档；1M 窗口阈值 967,000（96.7%）<br>CCB：三档 `有效窗口 ≥800k ? 50,000 : ≥400k ? 30,000 : 13,000`；1M 窗口阈值 942,000（94.2%）（见下算例） | `reserveTokens: 16,384`（默认，可配）；阈值 ≈ 窗口的 98.4%（固定 16k 留白，不随窗口分档） | `DEFAULT_THRESHOLD_RATIO = 0.8`（默认留 20% 余量）；`compactionRetries` 允许压后重试直至低于阈值 | — |
| **位置** | 原版：`src/services/compact/autoCompact.ts:72` `getAutoCompactThreshold()` · 缓冲常量 `:62` · 窗口 `:33`<br>CCB：`src/services/compact/autoCompact.ts:202` `getAutoCompactThreshold()`<br>窗口 `:75` · 缓冲 `:152` · 输出预留 `:34` | `pi: dist/core/compaction/compaction.js:161` `shouldCompact()`<br>默认值 `:74-78` `DEFAULT_COMPACTION_SETTINGS`<br>调用点 `dist/core/agent-session.js:279`（发前）`/ :1724`（响应后） | `dsh: @deepseek-ai/dsh-compaction-basic/lib/index.js:132`（阈值计算）`/ :949` `compactPressure()`<br>`:15` `DEFAULT_THRESHOLD_RATIO=0.8` | — |
| **注** | ①②共同：输出预留封顶 20k（输出上限 64k 时也只留 20k）；窗口可用 `CLAUDE_CODE_AUTO_COMPACT_WINDOW` 压低（只能调小）<br>②CCB 特有：缓冲分档有断崖，有效窗口 800k 处跳增 20k，窗口刚过 808k 时阈值反降<br>③原版固定 13k 缓冲，无断崖；1M 窗口留白仅 33k（3.3%），比 CCB 的 58k 更激进 | PI 计量与 CC 同构：usage 精确 + 尾部 `chars/4` 估算（`estimateContextTokens`）；estimate 注释自述 "conservative (overestimates)"——宁可高估早压，不撞墙；溢出后走 overflow-with-retry（删失败回复→压→重试该轮一次） | DSH 留白最激进（20%），单层阈值即可吸收单轮膨胀，**无预测层**；压完仍超线按 `compactionRetries` 重试；provider 确认溢出走 `context-overflow` trigger 强制压（绕过阈值与保留策略）；用 `ctx.tokenMeter` 统一计量，非 4:1 粗估 | — |

**原版算例**（固定缓冲 13k，输出预留取 20k）：

| 窗口 | 有效窗口 | 缓冲 | 阈值 | 占比 |
|---|---|---|---|---|
| 200k | 180,000 | 13,000 | **167,000** | 83.5% |
| 400k | 380,000 | 13,000 | **367,000** | 91.8% |
| **1M** | 980,000 | 13,000 | **967,000** | 96.7% |

**CCB 算例**（三档缓冲，输出预留取 8k）：

| 窗口 | 有效窗口 | 缓冲 | 阈值 | 占比 |
|---|---|---|---|---|
| 100k | 92,000 | 13,000 | **79,000** | 79.0% |
| 200k | 192,000 | 13,000 | **179,000** | 89.5% |
| 400k | 392,000 | 13,000 | **379,000** | 94.8% |
| 500k | 492,000 | 30,000 | **462,000** | 92.4% |
| 800k | 792,000 | 30,000 | **762,000** | 95.2% |
| **1M** | 992,000 | 50,000 | **942,000** | 94.2% |
| 2M | 1,992,000 | 50,000 | **1,942,000** | 97.1% |

**三方阈值哲学对比**（2026-11-11 源码核实）：

| 留白策略 | ClaudeCode 原版 | ClaudeCode (CCB) | PI | DSH |
|---|---|---|---|---|
| 1M 窗口实际留白 | 33k（缓冲 13k + 输出预留 20k）≈ 3.3% | 58k（缓冲 50k + 输出预留 8k）≈ 5.8% | 16,384 ≈ 1.6% | 200,000 ≈ 20% |
| 单层够不够 | 够（单层阈值，无预测层） | 不够 → 需第二道预测线 | 够（高估计量 + overflow 兜底） | 够（大留白 + retries + overflow trigger） |
| 自动触发点数 | 1（967k） | 2（常规 942k + 预测 969k） | 1 公式、2 检查时机（响应后 + 发前，同阈值） | 1（`pressure`），overflow 为独立事后 trigger |

> 原版留白最激进（3.3%）却只用一个阈值，靠事后 413 兜底；CCB 反而加大留白并加了预测线。DSH 留白最大（20%）只用一道线。留白越激进越依赖"预测/事后"补偿——这是架构选择联动，不是谁更先进。

---

## token 计量方式

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 精确值 + 增量估算（原版与 CCB 同构：尾扫最后一条 usage 作锚点，其后消息粗估） | 同构：`calculateContextTokens(usage)` + 尾部消息 `chars/4` 粗估 | 统一 `ctx.tokenMeter` 服务测量（非 4:1 字符估算） | — |
| **值** | `input + cache_creation + cache_read + output`<br>其后消息按 `4 字符 = 1 token` 粗估 | `usageTokens + trailingTokens`；图片按 4,800 字符折算；estimate 注释自述偏保守（高估） | meter 口径由实现决定，压力预算 = 消息预算 − 完成预留 − headroom | — |
| **位置** | 原版：`src/utils/tokens.ts:226` · CCB：`src/utils/tokens.ts:307` `tokenCountWithEstimation()`<br>求和 `:55` · 估算比例 `src/services/tokenEstimation.ts:215` | `pi: dist/core/compaction/compaction.js:104-158` `estimateContextTokens()` | `dsh: dsh-compaction-basic/lib/index.js:793` 起（依赖注入 `ctx.tokenMeter`） | — |
| **注** | 4:1 对中文偏低，中文会话会**提前**触发限制；usage 四项中 `getCurrentUsage` 对全零占位有跳过防御，但 `tokenCountWithEstimation` 锚点判定不看全零——第三方 provider 的 message_start 占位 usage 会让计数瞬间塌到只剩 output | PI 的 token 计数与压缩判定解耦清晰：`estimateContextTokens` 返回 `{tokens, usageTokens, trailingTokens, lastUsageIndex}`，压缩后靠 `lastUsageIndex` 时间戳防"刚压完误触发" | DSH 把计量做成独立服务（tokenMeter），压缩层不自带估算——计量口径可在 composition 层替换 | — |

---

## 预测性压缩

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 发请求**前**估算本轮增长，会顶爆就先压（定义：提前预判而非事后触发） | **无此机制**。发前检查（`_compactBeforeNextAssistantResponse`）与响应后检查共用同一阈值 16k 留白，不做增长预估 | **无此机制**。单层 `pressure` trigger（0.8 线）+ provider 溢出后 `context-overflow` 强制压 | — |
| **值** | `有效窗口 − (min(输出上限,20k) + 15,000)`<br>1M 窗口：`992,000 − 23,000 = 969,000` | — | — | — |
| **位置** | `CCB: src/query.ts:883`<br>估算 `src/services/compact/autoCompact.ts:169` `estimateMaxTurnGrowth()` | 对比点：`pi: dist/core/agent-session.js:279`（发前检查，同阈值无增长项） | 对比点：`dsh: dsh-compaction-basic/lib/index.js:932` `compactIfNeeded(trigger: 'pressure' \| 'context-overflow')` | — |
| **注** | 修正旧记录：预测阈值 969k 比常规 942k **晚** 27,000 触发（数字更大、更靠墙），是第二道防线而非"提前" | — | — | — |

**预测层是否必要（三方源码对比后的结论，2026-11-11）**：

- 预测层补偿的是 CC 自己的两个架构选择：**留白激进**（1M 只留约 5.8% = 缓冲 50k + 输出预留 8k，且窗口越大缓冲占比越低）+ **粗估增量**（4:1 字符估算，大工具结果误差放大）；
- DSH（20% 留白 + `ctx.tokenMeter` 精确计量）和 PI（自述"conservative"的高估型估算 + overflow 后压完重试一次）都用单层阈值覆盖同样风险，**没有第二道线**——单层模型在足够留白下成立；
- 预测层真正防的场景：常规判定（942k）之后、请求发出之前消息数组再次增长（迟到 tool_result、附件注入），叠加估算低估时直接冲破有效窗口 992k。这是"判定后增量窗口"的盲区，常规层时机上覆盖不到；
- 结论：删掉预测层 99% 会话无感，代价是极端会话（估算低估 + 判定后增量叠加）直接吃 413、只能靠 reactive compact 事后救。是否值得保留取决于愿意为 1% 场景付一道冗余检查的成本。
