# Harness 横向对比矩阵

> **用途**：把几个 agent harness 的机制按层拆开，逐行对比。
>
> **分层依据**：参照 [how-claude-code-works](https://github.com/Windy3f3f3f3f/how-claude-code-works) 的章节体系（核心架构 / 能力系统 / 运行逻辑 / 设计哲学 / 运行观测），整理为 10 层。
>
> **结构**：两层。本文件是第一层——每层一张**信号矩阵**，格内只放「有/默认 + 一句话做法 + 证据锚点」；具体数字、公式、代码位置、坑，全部放 `details/` 下对应层的**详情表**（字段为行、harness 为列）。
>
> **示例基准**：除特别说明，所有算例统一 **ContextWindow = 1M**、输出预留取默认 8k。
>
> **证据标注**：✅ 官方文档 · 🟦 源码锚点 · ⚠️ 待验证 · `—` 无此机制

## 两层结构怎么读

1. 在本文件扫信号：哪几个 harness 有此机制、默认开没开，点锚点跳详情。
2. 去详情块看内容：`做法` / `值` / `位置` / `注` 四字段为行、harness 为列，同字段跨 harness 对齐。
3. 详情文件按层编号：`details/01-context.md` … `details/10-observability.md`。

## 10 层总览

| 层 | 名称 | 覆盖范围（对应 how-claude-code-works 章节） | 详情 |
|---|---|---|---|
| 1 | 上下文 | 上下文工程、压缩策略（03） | [details/01-context.md](details/01-context.md) |
| 2 | 工具 | 工具系统、代码编辑策略（04、05） | [details/02-tools.md](details/02-tools.md) |
| 3 | 主循环 | 系统主循环、概述（02、01） | [details/03-agent-loop.md](details/03-agent-loop.md) |
| 4 | 安全 | 权限与安全（11） | [details/04-security.md](details/04-security.md) |
| 5 | 扩展 | Hooks、技能系统（06、09） | [details/05-extensibility.md](details/05-extensibility.md) |
| 6 | 记忆 | 记忆系统（08） | [details/06-memory.md](details/06-memory.md) |
| 7 | 多 Agent | 多 Agent、Agent Teams、后台舰队（07、20、21） | [details/07-multi-agent.md](details/07-multi-agent.md) |
| 8 | 任务与计划 | Plan 模式、任务系统、goal/loop（10、15、17） | [details/08-task-plan.md](details/08-task-plan.md) |
| 9 | 提示词与 UX | 系统提示词设计、用户体验（14、12） | [details/09-prompt-ux.md](details/09-prompt-ux.md) |
| 10 | 观测 | 可观测性 Metrics/Trace（16） | [details/10-observability.md](details/10-observability.md) |

---

## 1. 上下文

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **自动压缩阈值** | 有·开 · 阈值=有效窗口−缓冲；原版固定 13k（1M→967k），CCB 三档缓冲（1M→942k）· [🟦:72](details/01-context.md#自动压缩阈值) | 有·开 · `tokens > 窗口−16,384` 一行判定 · [🟦:161](details/01-context.md#自动压缩阈值) | 有·开 · `totalTokens ≥ min(窗口×0.8, pressureBudget)` · [🟦:132](details/01-context.md#自动压缩阈值) | 待填 |
| **token 计量方式** | 有·开 · 最后一条 usage 精确 + 其后消息 `4字符=1token` 粗估 · [🟦:226](details/01-context.md#token-计量方式) | 有·开 · 同构：usage 精确 + 尾部粗估，自述保守（高估） · [🟦:104](details/01-context.md#token-计量方式) | 有·开 · 独立 `ctx.tokenMeter` 服务，非 4:1 粗估 · [🟦:793](details/01-context.md#token-计量方式) | 待填 |
| **预测性压缩** | 原版：无 · CCB：有·开 · 发请求前按增长预估，1M 阈值 969k（第二道防线） · [🟦:883](details/01-context.md#预测性压缩) | 无 · 发前/响应后同阈值，无增长预估 · [🟦:279](details/01-context.md#预测性压缩) | 无 · 单层 pressure + overflow 强制压 · [🟦:932](details/01-context.md#预测性压缩) | 待填 |

---

## 2. 工具

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **默认工具集** | 有·开 · 无条件 ~25 + 条件默认真 ~16 + 环境/flag 特定 ~14，见清单 · [🟦:218](details/02-tools.md#默认工具集) | 待填 | 待填 | 待填 |
| **工具输出过长** | 有·开 · 工具内限制 → 单结果落盘 → 单消息聚合预算 200k 字符 · [🟦:55](<details/02-tools.md#工具输出过长>) | 待填 | 待填 | 待填 |
| **Read 重复读取去重** | 有·开 · 同路径+同区间+mtime 未变 → 回 stub（≈30 token）；compact 清账回填，microcompact 有缺口 · [🟦:538](<details/02-tools.md#read-重复读取去重>) | 待填 | 待填 | 待填 |
| **工具延迟加载** | 原版：有·开 · defer_loading + ToolSearchTool，API 服务端展开 · CCB：有·开 · SearchExtraTools+ExecuteExtraTool 纯客户端，schema 永不进 tools 数组 · [🟦:1158](<details/02-tools.md#工具延迟加载-tool-search>) | 有·开 · 四档 exposure + tool_search(BM25) 命中注入下一轮 · [🟦:61](<details/02-tools.md#工具延迟加载-tool-search>) | 无 · schema 全量进 tools 数组；PTC 换形态兜底 · [🟦:126](<details/02-tools.md#工具延迟加载-tool-search>) | 待填 |
| **Web 能力（搜索/抓取）** | 有·开 · WebSearch=嵌套请求借服务端搜索；WebFetch=抓取→MD→Haiku 提炼；CCB 改 tavily 纯 HTTP · [🟦:209](details/02-tools.md#web-能力搜索与抓取) | 待填 | 待填 | 待填 |

---

## 3. 主循环

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| （待填） | | | | |

---

## 4. 安全

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| （待填） | | | | |

---

## 5. 扩展

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| （待填） | | | | |

---

## 6. 记忆

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| （待填） | | | | |

---

## 7. 多 Agent

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| （待填） | | | | |

---

## 8. 任务与计划

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| （待填） | | | | |

---

## 9. 提示词与 UX

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| （待填） | | | | |

---

## 10. 观测

| 机制 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| （待填） | | | | |

---

## 填表约定

### 字段含义

| 字段 | 含义 | 写在哪 |
|---|---|---|
| `有` | 该 harness 是否有此机制：有 / 无 / 部分 | 本文件矩阵格 |
| `默认` | 默认是否生效：开 / 关 / N/A | 本文件矩阵格 |
| `做法` | 一句话概括实现思路 | 详情表 |
| `值` | 具体数字、公式、常量（一行一项） | 详情表 |
| `位置` | `CCB: 路径:行号` + 符号名 | 详情表 |
| `注` | 反常点、坑、易误解处 | 详情表 |

### 规则

1. **两层各管各的**：本文件格子只写「信号 + 一句话 + 证据锚点」，**禁止换行**；做法/值/位置/注一律写进对应详情块。
2. **矩阵格格式**：`有·默认 · 一句话做法 · 证据锚点`；无此机制写 `—`；未验证加 `⚠️` 前缀。
3. **详情表内一格一行**（用 `<br>` 分行），不挤成一段；写具体数字和公式，不写形容词。
4. **算例统一 1M 窗口 + 输出预留 8k**；随窗口变化的在详情块内用 markdown 表格逐行列出。
5. **必须给可跳转代码位置**，写在详情表 `位置` 行。
6. **机制语义不同时**，在详情表 `做法` 行各自点明定义。
7. 空格写 `待填`；新增机制：先加矩阵行，再建详情块（四字段暂写 `—`）。
