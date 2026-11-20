# 7. 多 Agent 与编排 · 机制详情

> 对应 [../harness-matrix.md](../harness-matrix.md) 第 7 层信号矩阵。核对日期：2026-11-20。

## 多 Agent 范式

各家支持哪些多 Agent 形态、怎么启用。三种范式：

- **子代理**：主 Agent 派生一次性执行者，完成后回传结果；
- **协调器**：主 Agent 变纯编排者，只派发不执行（星形）；
- **团队**：多个常驻 Agent 共存，点对点通信（网状）。

| 范式 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **子代理** | 有 · Agent 工具，`subagent_type` 选型，prompt 自包含，fork 型可继承父上下文 · [🟦](claude-code-sourcemap) | 无 · 无多 Agent 工具面，session fork 由编排层用 | 有 · `agent` 工具：立即返回 + 完成通知 + `send_message` 可继续 · [🟦](deepseek-harness/packages/subagent/tool-subagent/src/index.ts) | 有 · `spawn_agent`：继承创建回合的 provider/sandbox/cwd · [🟦](codex/codex-rs/core/src/tools/handlers/multi_agents_spec.rs) |
| **协调器** | 有 · ant-only：`CLAUDE_CODE_COORDINATOR_MODE=1` + 编译期门控（外部构建已 DCE 剔除）· [🟦](claude-code-sourcemap/restored-src/src/coordinator/coordinatorMode.ts) | 无 | 无 | 无 |
| **团队** | 有 · Swarm：外部需 `--agent-teams` 启用；对话会话即 lead（队内名 `team-lead`），队友常驻（独立进程），SendMessage 信箱 + 共享任务列表 · [🟦](claude-code-sourcemap/restored-src/src/utils/agentSwarmsEnabled.ts) | 无 | 半 · team 包提供任务板（team_task_*），但无独立常驻队友——"teammates are subagents" · [🟦](deepseek-harness/packages/team/tool-agent-team/src/index.ts) | 无 |

各格标注：有/无/半（形态不完整）；🟦 为源码锚点。启用条件、工具清单、提示词差异等细节不在此展开，需要时按锚点回源码。

## Agent 间通信

| 家 | 机制 |
|---|---|
| ClaudeCode | 同步=tool_result 回传；异步=`<task-notification>` XML 投父级下轮（去重）；队友=SendMessage 信箱（点对点/广播）· [🟦](claude-code-sourcemap/restored-src/src/utils/swarm/teammatePromptAddendum.ts) |
| PI | 待填 |
| DSH | `send_message`：运行中转向 / 结束后继续对话 · [🟦](deepseek-harness/packages/subagent/tool-subagent-control/src/index.ts) |
| Codex | send_input / send_message / followup_task / wait_agent 多动词族 · [🟦](codex/codex-rs/core/src/tools/handlers/multi_agents_spec.rs) |

## Worktree 隔离

| 家 | 机制 |
|---|---|
| ClaudeCode | Agent 参数 `isolation:"worktree"`：独立工作树，slug 校验防路径穿越，无变更自动清理 · [🟦](claude-code-sourcemap/restored-src/src/utils/worktree.ts) |
| PI / DSH / Codex | 待填 |

## 待研究

- PI session fork 的编排细节；Codex multi_agent_v2 与 v1 差异；DSH team 任务板与子代理收件箱对接；CC fork 的 prompt cache 经济学。
