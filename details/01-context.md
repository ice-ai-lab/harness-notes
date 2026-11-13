# 1. 上下文 · 机制详情

> 对应 [../harness-matrix.md](../harness-matrix.md) 第 1 层信号矩阵。字段为行、harness 为列。
>
> **算例基准**：统一 ContextWindow = 1M、输出预留默认 8k；随窗口变化的用表格逐行列出。

---

## 自动压缩阈值

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | `阈值 = 有效窗口 − 缓冲`；`有效窗口 = 窗口 − min(输出上限, 20k)`。原版固定缓冲；CCB 缓冲三档 | `shouldCompact()` 一行判定，`keepRecentTokens: 20,000` 控压缩后保留量 | `totalTokens ≥ min(窗口×0.8, pressureBudget)`；压后仍超线按 `compactionRetries` 重试 | — |
| **值** | 原版缓冲 13k 固定；CCB 三档 `≥800k?50k : ≥400k?30k : 13k` | `reserveTokens: 16,384`（可配），阈值 ≈ 窗口 98.4% | `DEFAULT_THRESHOLD_RATIO = 0.8`（留 20% 余量） | — |
| **位置** | 原版 `autoCompact.ts:72` `getAutoCompactThreshold()` · CCB `:202` | `pi: compaction.js:161` · 默认值 `:74-78` · 调用点 `agent-session.js:279` / `:1724` | `dsh-compaction-basic/lib/index.js:132`（阈值）`/ :949` `compactPressure()` · `:15` | — |
| **注** | 输出预留封顶 20k；CCB 三档有断崖（800k 处缓冲跳增，窗口刚过 808k 阈值反降）；原版 1M 留白仅 3.3% 更激进 | 计量与 CC 同构；estimate 自述保守（宁高估早压）；溢出走 overflow-with-retry | 留白最激进（20%），单层阈值即可；provider 确认溢出走 `context-overflow` trigger 强制压；`ctx.tokenMeter` 统一计量 | — |

**算例**（1M 窗口）：原版阈值 967,000（96.7%，留白 33k）；CCB 942,000（94.2%，留白 58k）；PI ≈ 983,616（98.4%）；DSH 800,000（80%）。

**三方阈值哲学对比**（2026-11-11 源码核实）：

| 留白策略 | CC 原版 | CC (CCB) | PI | DSH |
|---|---|---|---|---|
| 1M 实际留白 | 33k ≈ 3.3% | 58k ≈ 5.8% | 16,384 ≈ 1.6% | 200,000 ≈ 20% |
| 单层够不够 | 够（靠 413 事后兜底） | 不够 → 加预测线 | 够（高估计量 + overflow 兜底） | 够（大留白 + retries + overflow trigger） |
| 自动触发点数 | 1（967k） | 2（942k + 预测 969k） | 1 公式、2 检查时机 | 1（pressure），overflow 为独立事后 trigger |

> 留白越激进越依赖"预测/事后"补偿——架构选择联动，不是谁更先进。

---

## token 计量方式

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 尾扫最后一条 usage 作锚点，其后消息粗估（原版与 CCB 同构） | 同构：`calculateContextTokens(usage)` + 尾部 `chars/4` | 独立 `ctx.tokenMeter` 服务测量，非 4:1 估算 | — |
| **值** | `input + cache_creation + cache_read + output`；其后 `4字符=1token` | `usageTokens + trailingTokens`；图片按 4,800 字符折算 | meter 口径由实现决定 | — |
| **位置** | 原版 `tokens.ts:226` · CCB `tokens.ts:307` `tokenCountWithEstimation()` | `pi: compaction.js:104-158` `estimateContextTokens()` | `dsh: dsh-compaction-basic/lib/index.js:793` | — |
| **注** | 4:1 对中文偏低（中文会话提前触发）；第三方 provider 的 message_start 全零占位 usage 会让锚点计数塌到只剩 output | `estimateContextTokens` 返回结构化结果，压缩后靠 `lastUsageIndex` 防"刚压完误触发" | 计量做成独立服务，压缩层不自带估算，口径可在 composition 层替换 | — |

---

## 预测性压缩

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 发请求**前**估算本轮增长，会顶爆就先压 | 无此机制，发前/响应后共用同一阈值 | 无此机制，单层 pressure + overflow 事后强制压 | — |
| **值** | `有效窗口 − (min(输出上限,20k) + 15k)`；1M → 969,000 | — | — | — |
| **位置** | `CCB: src/query.ts:883` · `autoCompact.ts:169` `estimateMaxTurnGrowth()` | 对比点 `pi: agent-session.js:279` | 对比点 `dsh: lib/index.js:932` | — |
| **注** | 预测阈值 969k 比常规 942k **晚**触发（第二道防线，非"提前"） | — | — | — |

**预测层是否必要**（三方源码对比结论，2026-11-11）：

- 预测层补偿的是 CC 自己的选择：**留白激进**（1M 只留 ~5.8%）+ **粗估增量**（4:1，大工具结果误差放大）；DSH（20% 留白 + 精确 meter）和 PI（高估型估算 + overflow 重试）用单层覆盖同样风险；
- 预测层真正防的场景：常规判定后、请求发出前消息数组再次增长（迟到 tool_result、附件注入），常规层时机上覆盖不到；
- 结论：删掉预测层 99% 会话无感，代价是极端会话直接吃 413。
