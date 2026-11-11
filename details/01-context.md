# 1. 上下文 · 机制详情

> 对应 [../harness-matrix.md](../harness-matrix.md) 第 1 层信号矩阵。字段为行、harness 为列，同字段跨 harness 对齐。
>
> **算例基准**：统一 ContextWindow = 1M、输出预留默认 8k；随窗口变化的用表格逐行列出。

---

## 自动压缩阈值

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | `阈值 = 有效窗口 − 缓冲`<br>`有效窗口 = 上下文窗口 − min(模型输出上限, 20,000)` | `tokens > 窗口 − reserveTokens(16,384)`（`shouldCompact()`，一行判定）<br>另设 `keepRecentTokens: 20,000` 控制压缩后保留量 | `totalTokens ≥ min(窗口 × 0.8, pressureBudget)`<br>`pressureBudget = 消息预算 − reservedCompletion − headroom` | — |
| **值** | 缓冲分档：`有效窗口 ≥800k ? 50,000 : ≥400k ? 30,000 : 13,000`（见下算例） | `reserveTokens: 16,384`（默认，可配）；阈值 ≈ 窗口的 98.4%（固定 16k 留白，不随窗口分档） | `DEFAULT_THRESHOLD_RATIO = 0.8`（默认留 20% 余量）；`compactionRetries` 允许压后重试直至低于阈值 | — |
| **位置** | `CCB: src/services/compact/autoCompact.ts:202` `getAutoCompactThreshold()`<br>窗口 `:75` · 缓冲 `:152` · 输出预留 `:34` | `pi: dist/core/compaction/compaction.js:161` `shouldCompact()`<br>默认值 `:74-78` `DEFAULT_COMPACTION_SETTINGS`<br>调用点 `dist/core/agent-session.js:279`（发前）`/ :1724`（响应后） | `dsh: @deepseek-ai/dsh-compaction-basic/lib/index.js:132`（阈值计算）`/ :949` `compactPressure()`<br>`:15` `DEFAULT_THRESHOLD_RATIO=0.8` | — |
| **注** | 输出预留封顶 20k（输出上限 64k 时也只留 20k）；缓冲分档有断崖，有效窗口 800k 处跳增 20k，窗口刚过 808k 时阈值反降；窗口可用 `CLAUDE_CODE_AUTO_COMPACT_WINDOW` 压低（只能调小） | PI 计量与 CC 同构：usage 精确 + 尾部 `chars/4` 估算（`estimateContextTokens`）；estimate 注释自述 "conservative (overestimates)"——宁可高估早压，不撞墙；溢出后走 overflow-with-retry（删失败回复→压→重试该轮一次） | DSH 留白最激进（20%），单层阈值即可吸收单轮膨胀，**无预测层**；压完仍超线按 `compactionRetries` 重试；provider 确认溢出走 `context-overflow` trigger 强制压（绕过阈值与保留策略）；用 `ctx.tokenMeter` 统一计量，非 4:1 粗估 | — |

**算例**（`阈值 = 有效窗口 − 缓冲`，随窗口变化）：

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

| 留白策略 | ClaudeCode | PI | DSH |
|---|---|---|---|
| 1M 窗口实际留白 | 58k（缓冲 50k + 输出预留 8k）≈ 5.8% | 16,384 ≈ 1.6% | 200,000 ≈ 20% |
| 单层够不够 | 不够 → 需第二道预测线 | 够（高估计量 + overflow 兜底） | 够（大留白 + retries + overflow trigger） |
| 自动触发点数 | 2（常规 942k + 预测 969k） | 1 公式、2 检查时机（响应后 + 发前，同阈值） | 1（`pressure`），overflow 为独立事后 trigger |

> CC 留白最小（5.8%）却分了两道线；DSH 留白最大（20%）只用一道线。留白越激进越依赖"预测/事后"补偿——这是架构选择联动，不是谁更先进。

---

## token 计量方式

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 精确值 + 增量估算 | 同构：`calculateContextTokens(usage)` + 尾部消息 `chars/4` 粗估 | 统一 `ctx.tokenMeter` 服务测量（非 4:1 字符估算） | — |
| **值** | `input + cache_creation + cache_read + output`<br>其后消息按 `4 字符 = 1 token` 粗估 | `usageTokens + trailingTokens`；图片按 4,800 字符折算；estimate 注释自述偏保守（高估） | meter 口径由实现决定，压力预算 = 消息预算 − 完成预留 − headroom | — |
| **位置** | `CCB: src/utils/tokens.ts:251` `tokenCountWithEstimation()`<br>求和 `:55` · 估算比例 `src/services/tokenEstimation.ts:215` | `pi: dist/core/compaction/compaction.js:104-158` `estimateContextTokens()` | `dsh: dsh-compaction-basic/lib/index.js:793` 起（依赖注入 `ctx.tokenMeter`） | — |
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
