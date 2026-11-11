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

---

## 工具输出过长

> 核对日期：2026-11-11。以下是本机 CCB 复原源码的默认行为，不保证所有官方 Claude Code 版本一致；环境变量、调用上下文及实验配置可覆盖默认值。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 工具先自行限制读取范围或输出 → 正常返回的 `tool_result` 经通用大小处理 → 发模型请求前检查单消息聚合预算 | 待填 | 待填 | 待填 |
| **值** | Read 文件字节限制：256 KiB（仅无 `limit`）<br>Read 内容预算：25,000 tokens<br>Read 快速拒绝：内容 >100,000 UTF-8 字节<br>尝试计数 API：粗估 >6,250 tokens<br>通用单结果阈值：`min(tool.maxResultSizeChars, 50_000)`<br>单消息聚合预算：200,000 字符<br>预览上限：2,000 个 JS 字符串长度单位 | 待填 | 待填 | 待填 |
| **位置** | [Read 入口:1028](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L1028-L1039>)<br>[token 校验:754](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L754-L780>)<br>[通用阈值:55](</Users/gatesma/project/claude/claude-code-best/src/utils/toolResultStorage.ts#L55-L78>)<br>[通用落盘:272](</Users/gatesma/project/claude/claude-code-best/src/utils/toolResultStorage.ts#L272-L333>)<br>[聚合预算:769](</Users/gatesma/project/claude/claude-code-best/src/utils/toolResultStorage.ts#L769-L868>) | 待填 | 待填 | 待填 |
| **注** | 字节、字符和 token 不是同一单位；工具自身超限可以报错，通用层过大通常存盘而非报错；落盘失败会保留原结果，因此通用预算不是严格硬上限 | 待填 | 待填 | 待填 |

### Read 工具

1. **文件大小检查**：不传 `limit` 时启用 0.25 MiB（256 KiB）限制；超过就报错，提示模型传 `offset/limit` 或搜索。普通小文件通过 `stat` 提前拒绝，大文件流式路径按累计字节拒绝。传了 `limit` 后跳过这道限制。
2. **内容 token 检查**：读取的 `content` 按默认 25,000-token 预算校验。先检查 UTF-8 字节数：超过 `maxTokens * 4`（默认 100,000 字节）直接报错，不走计数 API。这是读取后的检查，与前面的 256 KiB 限制不同。
3. **粗估与 API 回退**：未被快速拒绝时，普通文件用 `round(content.length / 4)`；`json/jsonl/jsonc` 用 `round(content.length / 2)`。粗估 ≤`maxTokens / 4`（默认 6,250）直接通过，否则尝试计数 API；认证、请求失败或返回值无效时使用原粗估。最终采用的数值 >25,000 才报错，不自动截断到 token 上限。
4. **通用结果层**：内容通过校验后加行号，再检查结果字符数。Read 声明 `maxResultSizeChars: 100_000`，默认被通用阈值压到 50,000；超过就全文存盘，给模型短预览和路径。token 校验通过的内容仍可能触发落盘。
5. **超长单行**：`limit` 只限制行数，不限制单行长度；`offset` 也是行号。即使 `limit: 1`，所选行仍可能超限。当前实现先读取再检查 token，选中的极长单行还可能造成缓冲内存增长。应改用字节/字符切片或结构化提取，而不是继续减小行数。

关键回退代码：

```ts
const tokenEstimate = roughTokenCountEstimationForFileType(content, ext)
if (!tokenEstimate || tokenEstimate <= effectiveMaxTokens / 4) return

const tokenCount = await countTokensWithAPI(content)
const effectiveCount = tokenCount ?? tokenEstimate

if (effectiveCount > effectiveMaxTokens) {
  throw new MaxFileReadTokenExceededError(effectiveCount, effectiveMaxTokens)
}
```

- `content.length` 是 UTF-16 code unit 数，不是 UTF-8 字节数；粗估函数的参数虽叫 `bytesPerToken`，实际仍按字符串长度计算。[粗估实现:215–254](</Users/gatesma/project/claude/claude-code-best/src/services/tokenEstimation.ts#L215-L254>)
- 默认预算可被调用上下文、`CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` 或 GrowthBook 覆盖；100,000 和 6,250 会随有效预算变化。[默认配置](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/FileReadTool/limits.ts#L18-L91>)
- 普通 Read 没有实际的默认 2,000 行上限；提示词与实现有差异。`@文件` 附件层在超限后可重试前 2,000 行，compact 模式则只给文件引用，不是普通 Read 自动重试。[附件分支:3234–3299](</Users/gatesma/project/claude/claude-code-best/src/utils/attachments.ts#L3234-L3299>)

### 通用结果层：何时落盘，何时报错

1. **作用范围**：统一执行框架正常返回的工具结果都会进入该处理层，但不是所有结果都能落盘。只有字符串或纯文本块数组能通过这套文本落盘逻辑保存。
2. **单结果条件**：无配置覆盖时，字符数 >`min(tool.maxResultSizeChars, 50_000)` 才落盘。Bash 的声明阈值是 30,000，Grep 是 20,000，Read 的有效阈值是 50,000。`tengu_satin_quoll` 可按工具覆盖阈值；`Infinity` 是硬退出，不受该覆盖影响。
3. **落盘形式**：整个结果保存到会话的 `tool-results` 目录；字符串存 `.txt`，纯文本块数组 JSON 序列化后存 `.json`。模型收到 `<persisted-output>`、输出路径、短预览及省略标记，而不是 AI 摘要。
4. **例外**：`maxResultSizeChars = Infinity` 不落盘；含图片结果跳过这层落盘；其他非文本内容无法保存时保留原结果。空结果会补充“工具完成但没有输出”的标记。
5. **报错规则**：通用层不会仅因输出过大而报错；正常落盘失败时保留原结果，不自动截断、不改为工具失败。后续模型请求仍可能因上下文过大失败。工具直接抛出的错误由执行框架包装成错误 `tool_result`，不经过这一正常成功结果分支。
6. **聚合条件**：同一 API 消息中可处理的工具结果合计 >200,000 字符，会优先把最大的、尚未固定处理决策的结果落盘替换成预览。之前的处理决策会冻结以保持缓存前缀稳定；`Infinity` 工具退出该预算，保存失败也保留原结果，所以不保证严格压到上限。

入口：[toolExecution.ts:1478–1490](</Users/gatesma/project/claude/claude-code-best/src/services/tools/toolExecution.ts#L1478-L1490>)；聚合调用：[query.ts:593–613](</Users/gatesma/project/claude/claude-code-best/src/query.ts#L593-L613>)。`MAX_TOOL_RESULT_TOKENS = 100_000` 及其派生常量不能理解为所有工具都会执行的精确 token 硬拒绝；正常单结果路径主要使用字符落盘阈值。[通用常量](</Users/gatesma/project/claude/claude-code-best/src/constants/toolLimits.ts#L5-L49>)

### 约 2,000 字符预览如何生成

1. 长度 ≤2,000：完整返回。
2. 长度更大：查看前 2,000 内最后一个换行符；若其索引 >1,000，就截到该换行前，否则硬截前 2,000。
3. 有后续内容时设置 `hasMore: true`，展示时在预览后加 `...`。

```ts
const truncated = content.slice(0, 2000)
const lastNewline = truncated.lastIndexOf('\n')
const cutPoint = lastNewline > 1000 ? lastNewline : 2000
const preview = content.slice(0, cutPoint)
```

`PREVIEW_SIZE_BYTES` 虽带 `BYTES`，实际按 JS 字符串长度截取，不是 UTF-8 字节或 token；2,000 只限制预览正文，不包括路径、提示、标签等包装文字。[generatePreview():339–355](</Users/gatesma/project/claude/claude-code-best/src/utils/toolResultStorage.ts#L339-L355>)

### Bash 读文件与其他工具

- **Bash / PowerShell**：不调用 Read 的文件大小/token 校验，改走 Shell 输出控制。普通 Bash 的 stdout/stderr 直接写任务输出文件，结束后默认只读回前 30,000 字节；成功的大输出保留路径，模型收到约 2,000 字符预览。`BASH_MAX_OUTPUT_LENGTH` 可调，配置上限 150,000；成功返回时的大输出持久化最多保留 64 MiB，超过会截掉。这个 64 MiB 不是运行期间的磁盘写入硬上限；失败和后台任务另有路径。[前缀读取](</Users/gatesma/project/claude/claude-code-best/src/utils/task/TaskOutput.ts#L297-L311>) · [配置](</Users/gatesma/project/claude/claude-code-best/src/utils/shell/outputLimits.ts#L1-L14>) · [64 MiB 处理](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/BashTool/BashTool.tsx#L870-L895>)
- **Grep**：搜索正文，默认最多返回 250 行/条，支持 `offset`；`head_limit: 0` 可取消条数限制，通用大小预算仍生效。匹配行自身很长时，条数限制也不能控制字节数。[实现](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/GrepTool/GrepTool.ts#L101-L124>)
- **Glob**：按文件名/路径模式找文件，默认最多 100 个，超出提示缩小范围，不检查正文。[实现](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/GlobTool/GlobTool.ts#L151-L168>)
- **WebFetch**：进入副模型处理的路径时，输入先裁到 100,000 字符，再按 prompt 提取/总结；被裁掉的部分副模型也没看过。[实现](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/WebFetchTool/utils.ts#L543-L573>)
- **MCP**：另有默认 25,000-token 计数预算；检测超限后通常存文本，含图或禁用大输出落盘时走裁剪。专用层的计数失败可放行；其落盘失败会返回错误说明，与通用层“保留原结果”不同。[计数](</Users/gatesma/project/claude/claude-code-best/src/utils/mcpValidation.ts#L151-L177>) · [超限分支](</Users/gatesma/project/claude/claude-code-best/src/services/mcp/client.ts#L2771-L2827>)

**区分三种行为**：按 `limit` 选范围 ≠ 超限报错 ≠ 存盘后只展示预览。脚本读过全文、输出已存盘，也不等于模型看过全文；Shell 输出保护不保证脚本自身的 CPU、内存和磁盘消耗安全。

### 计数 API 与本机诊断记录

- **直连 Anthropic**：`countTokensWithAPI()` 将所选正文包装成一条 `user` 消息、`tools: []`，按主循环模型调用 `POST /v1/messages/count_tokens`，使用返回的 `input_tokens`。不生成回答，不包含 Read 随后添加的行号，也不是整个会话的计数。官方将服务端返回值定义为估计值。[调用链:131–210](</Users/gatesma/project/claude/claude-code-best/src/services/tokenEstimation.ts#L131-L210>) · [官方说明](<https://platform.claude.com/docs/en/build-with-claude/token-counting>)
- **provider 差异**：本仓库 Gemini 分支直接本地估算；Bedrock 使用专用计数接口；OpenAI 兼容模式没有对应计数分支，仍创建 Anthropic client，不使用 `OPENAI_BASE_URL/OPENAI_API_KEY`。
- **2026-11-11 本机配置快照**：正常生成使用 OpenAI 兼容模式，目标 DeepSeek；`ANTHROPIC_BASE_URL` 指向 `http://127.0.0.1:15721`，该端口由 `tt-switch` 监听。对普通和 `?beta=true` 的计数路径发送无敏感测试文本、未携带真实密钥，均返回 HTTP 404，没有取得计数。
- **实际影响**：按该快照，Read 进入计数分支后无法从已测路径得到计数，失败后回退字符长度粗估；不代表 Read 必然失败。认证不足也可能在 HTTP 请求前就失败。测试未追踪已有 Claude 进程的启动覆盖，不能把这次 404 推广为所有 TT Switch/DeepSeek 配置均不支持计数。

复现本地路径测试（不发送项目正文或真实密钥）：

```bash
curl --silent --show-error --noproxy '*' --max-time 10 \
  -X POST 'http://127.0.0.1:15721/v1/messages/count_tokens' \
  -H 'content-type: application/json' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"token-count-probe"}],"tools":[]}' \
  -w '\nHTTP_STATUS=%{http_code}\n'
```

**一句话**：工具内部负责范围限制或超限报错，通用层负责大文本结果落盘和短预览，计数服务不可用时 Read 继续使用粗估；这些阈值不能混同为一个精确 token 硬上限。

---

## Read 重复读取去重

> 核对日期：2026-11-11。行为基于本机 CCB 复原源码（v2.1.888）。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 会话级 `readFileState`（FileStateCache）记录每次真实读取的 `{content, mtime, offset, limit}`；下一次 Read 命中「同路径 + 同区间 + mtime 未变」时不再返回内容，改回一句固定 stub，让模型引用上文里已有的那次 `tool_result` | 待填 | 待填 | 待填 |
| **值** | 四道闸门（全部成立才 stub）：① 记录存在；② 非自动注入（`!isPartialView`）且 `offset !== undefined`（来自真实 Read）；③ `offset`/`limit` 与上次完全相同；④ 重新 stat，`Math.floor(mtimeMs)` 逐毫秒相等<br>stub 文案：`File unchanged since last read. …refer to that instead of re-reading.`（约 30 token）<br>命中输出类型：`{type: 'file_unchanged'}`，UI 显示 `Unchanged since last read`<br>记录容量：LRU 100 条 / 25MB（按 content UTF-8 字节计）<br>动机数据（源码注释）：约 18% 的 Read 是同文件碰撞，最高占 fleet cache_creation 2.64% | 待填 | 待填 | 待填 |
| **位置** | [去重判定:538](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L538-L588>)<br>[stub 文案](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/FileReadTool/prompt.ts#L7-L8>)<br>[file_unchanged 映射:701](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L701-L706>)<br>[FileStateCache](</Users/gatesma/project/claude/claude-code-best/src/utils/fileStateCache.ts#L30-L59>)<br>[Read 写记录:1041](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L1041-L1046>)<br>[killswitch:551](</Users/gatesma/project/claude/claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L551-L554>) | 待填 | 待填 | 待填 |
| **注** | ① 纯客户端机制，GrowthBook killswitch `tengu_read_dedup_killswitch`（默认 false=启用），Bedrock/Vertex 同样生效；命中上报 `tengu_file_read_dedup`。② 判据是 mtime 不是内容哈希：外部 `touch -d` 回拨 mtime 会误判"未变"。③ 图片/PDF 不写记录，永不去重。④ 命中省的是"第二份全文"，不缩短已有上下文；stub 的有效性依赖旧 `tool_result` 还在历史里（见下方"失效时机"）。⑤ 换区间读会覆写记录，只有"最近一次区间"可去重；软链接是不同键。⑥ Magic Docs / SessionMemory 需要真实内容，做法是先 `readFileState.delete(path)` 再调 Read 绕过 | 待填 | 待填 | 待填 |

### 为什么需要这个机制

1. **省重复 token**：同文件全文出现两份时，之后每一轮请求都背着两份；服务端 prompt cache 虽对已见前缀按缓存价读，但窗口占用不变，且追加内容时还要再付一次 cache_creation。stub 把第二份全文换成约 30 token。
2. **延后压缩**：上下文更慢到达压缩阈值，就更少触发"摘要 + 只回填 5 个文件 × 5k token"这种高信息损失的操作。
3. **减少版本歧义**：Read → Edit → Read 序列里，不去重会同时存在改前/改后两份全文；stub 明确说"用前面那份"。
4. **代价极低**：命中路径只多一次 `stat`；判断错了顶多退化为全量读，不影响正确性——所以敢做成随时可关的实验开关。

### 存储形态（readFileState 是什么）

- 纯进程内存对象，不落盘。`FileStateCache` 内部是 `LRUCache<string, FileState>`，键为 `path.normalize` 后的绝对路径（`expandPath` 先把 `~`/相对路径归一为绝对路径，所以 `./a.ts` 与 `/abs/a.ts` 命中同一条；软链接不解析，是不同键）。
- 每会话一个实例：REPL 用 `useState` 惰性创建一次 + `useRef` 持有（构造 LRUCache 约 170ms，不能每渲染新建）；QueryEngine 持同一引用；print/SDK 启动时从 `initialMessages` 用 `extractReadFilesFromMessages` 重建（`--resume` 后 Edit 仍受"先读后写"约束）。
- 生命周期：`/clear` 清空；compact 先 `cacheToObject` 快照 → `clear()` → 回填 → 置 `undefined` 释放内存（快照可能挂 25MB+）；子 agent `cloneFileStateCache` 隔离。
- 记录的 `content` 存整份文件文本——它本身就是一份上下文外的文件副本，这是 25MB 上限存在的原因；同时也是每轮 `getChangedFiles`（检测外部修改并注入 diff 附件）的数据源，以及 Edit/Write「先读后写」校验的数据源。一本账三个用途。

### 失效时机（stub 何时拿不到内容 / 去重何时不成立）

| 场景 | 结果 |
|---|---|
| 文件被任何写者改过（Edit/Write/NotebookEdit/linter/git checkout） | mtime 变 → 全量读（最常见出口，去重默认让位给新鲜度） |
| 换 offset/limit 读同一文件 | 区间不等 → 全量读，并覆写记录（只有最近区间可去重） |
| Edit/Write/NotebookEdit 成功后读 | 它们写记录时 `offset: undefined` → 闸门 ② 永不成立（防同毫秒写入 mtime 未变时命中旧内容） |
| 自动注入的 CLAUDE.md | `isPartialView: true` → 永不去重（模型看到的可能是剥注释/截断版） |
| 图片 / PDF | 不写记录 → 每次重发 |
| LRU 淘汰（>100 条或 >25MB） | 记录没了 → 全量读 |
| 子 agent / 新会话 / resume 重建条目（`offset: undefined`） | 闸门 ② 不成立 → 全量读 |
| mtime 被外部还原（`touch -d`、备份还原） | **误判为未变**，返回 stub——mtime 是唯一真相，无内容哈希兜底 |
| **full/partial compact** | 先快照 → `readFileState.clear()` → 按时间戳回填最近 5 个文件（各 ≤5k token，经真实 Read 重写记录）。清空是正确性要求：旧 `tool_result` 已被摘要掉，若记录残留，下次 Read 会说"用前面那份"而那份已不存在（悬空 stub）。清空后所有文件回到"未读"，没被回填的文件 Edit 直接报 `File has been unexpectedly modified`（实际是"被压缩掉"而非"被修改"，文案有误导） |
| **microcompact**（time-based 默认关，cached MC 需 feature+env） | 只把老 tool_result 内容换成 `[Old tool result content cleared]`，**不碰 readFileState**——当前实现里唯一可能造成"stub 指向已清空结果"的缺口 |
| compact 回填判定 | `collectReadToolFilePaths` 跳过 stub 的 Read，避免把"命中过 stub"误判成"内容还在上下文"，导致该回填的不回填 |

从历史重建 `readFileState` 的地方同样要防 stub：`queryHelpers.ts` 提取 Read 结果时跳过以 stub 开头的 `tool_result`，否则 stub 文本会被当成文件内容存进记录。

### 与 Edit「先读后写」的关系（共享同一本账，判定独立）

- Edit/Write 校验「文件是否被外部改过」用的也是 `readFileState` 的 `timestamp` vs 当前 mtime；去重失效（如 compact 清空）**不影响**这个校验的正确性——记录没了只会让 Edit 要求先 Read，不会让它编辑到旧内容上。
- 但两者对「什么算读过」定义不同：去重要求 `offset !== undefined`（真实 Read），校验接受任何条目。自动注入的 CLAUDE.md（`isPartialView: true`）能通过 Edit 校验，却不能去重。
- Edit 失败（`String to replace not found`，code 8）不改记录；随后对同一文件 Read 若命中 stub（"unchanged"），两条消息组合恰好证明文件没被改、是 old_string 本身不对。

### 关键代码骨架

```ts
// FileReadTool.call() 内，真实读取之前
const existingState = readFileState.get(fullFilePath)
if (existingState && !existingState.isPartialView && existingState.offset !== undefined) {
  const rangeMatch = existingState.offset === offset && existingState.limit === limit
  if (rangeMatch) {
    const mtimeMs = await getFileModificationTimeAsync(fullFilePath)
    if (mtimeMs === existingState.timestamp) {           // 逐毫秒比较
      return { data: { type: 'file_unchanged', file: { filePath: file_path } } }
    }
  }
}
// 未命中 → 全量读，然后覆写记录（下次同一区间才能命中）
readFileState.set(fullFilePath, { content, timestamp: Math.floor(mtimeMs), offset, limit })
```




