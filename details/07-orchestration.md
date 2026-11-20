# 7. 多 Agent 与编排 · 机制详情

> 对应 [../harness-matrix.md](../harness-matrix.md) 第 7 层信号矩阵。核对日期：2026-11-20。以 ClaudeCode（CC）为例记录三种多 Agent 范式；各家支持情况见文末表。

## 三种多 Agent 范式（CC）

**① 子 Agent 模式**：独立的子 Agent，有自己的 tools 列表（按 `subagent_type` 选型，内建 general-purpose / Explore / Plan，或 `.claude/agents/*.md` 自定义）；支持 frontmatter `context` 字段决定执行形态——`inline`（缺省，子 Agent 在主对话内联执行）或 `fork`（fork 一个子 Agent 独立跑，继承完整上下文，主对话只收最终结果）。

**② 协调器模式**（ant-only）：主 Agent 的提示词和工具都会改——工具收窄到 Agent / SendMessage / TaskStop，**Bash、Read、Edit、Write 这些执行类工具全部拿掉**，主 Agent 只能派发不能动手；提示词也换成专门的编排者版（角色、派发规范、综合要求），更简单聚焦。

**③ Swarm 模式**：提示词同样变化（lead 与队友各有一套）；并且会提供 Agent 间通信工具，比如 SendMessage（点对点/广播信箱）、TeamCreate、TaskCreate 系共享任务列表——队友之间靠这些工具通信，纯文本输出对方看不见。

## 各家支持情况

| 范式 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **子 Agent** | 有 | 无（session fork 留给编排层） | 有 · `agent` 工具 | 有 · `spawn_agent` |
| **协调器** | 有 · ant-only | 无 | 无 | 无 |
| **Swarm 团队** | 有 · `--agent-teams` 启用 | 无 | 半 · team 包只有任务板，无常驻队友 | 无 |

## 待研究

- PI session fork 的编排细节；Codex multi_agent_v2 与 v1 差异；DSH team 任务板与子代理收件箱对接；CC fork 的 prompt cache 经济学。
