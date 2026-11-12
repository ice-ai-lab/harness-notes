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
| **位置** | [Read 入口:1028](<../../claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L1028-L1039>)<br>[token 校验:754](<../../claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L754-L780>)<br>[通用阈值:55](<../../claude-code-best/src/utils/toolResultStorage.ts#L55-L78>)<br>[通用落盘:272](<../../claude-code-best/src/utils/toolResultStorage.ts#L272-L333>)<br>[聚合预算:769](<../../claude-code-best/src/utils/toolResultStorage.ts#L769-L868>) | 待填 | 待填 | 待填 |
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

- `content.length` 是 UTF-16 code unit 数，不是 UTF-8 字节数；粗估函数的参数虽叫 `bytesPerToken`，实际仍按字符串长度计算。[粗估实现:215–254](<../../claude-code-best/src/services/tokenEstimation.ts#L215-L254>)
- 默认预算可被调用上下文、`CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` 或 GrowthBook 覆盖；100,000 和 6,250 会随有效预算变化。[默认配置](<../../claude-code-best/packages/builtin-tools/src/tools/FileReadTool/limits.ts#L18-L91>)
- 普通 Read 没有实际的默认 2,000 行上限；提示词与实现有差异。`@文件` 附件层在超限后可重试前 2,000 行，compact 模式则只给文件引用，不是普通 Read 自动重试。[附件分支:3234–3299](<../../claude-code-best/src/utils/attachments.ts#L3234-L3299>)

### 通用结果层：何时落盘，何时报错

1. **作用范围**：统一执行框架正常返回的工具结果都会进入该处理层，但不是所有结果都能落盘。只有字符串或纯文本块数组能通过这套文本落盘逻辑保存。
2. **单结果条件**：无配置覆盖时，字符数 >`min(tool.maxResultSizeChars, 50_000)` 才落盘。Bash 的声明阈值是 30,000，Grep 是 20,000，Read 的有效阈值是 50,000。`tengu_satin_quoll` 可按工具覆盖阈值；`Infinity` 是硬退出，不受该覆盖影响。
3. **落盘形式**：整个结果保存到会话的 `tool-results` 目录；字符串存 `.txt`，纯文本块数组 JSON 序列化后存 `.json`。模型收到 `<persisted-output>`、输出路径、短预览及省略标记，而不是 AI 摘要。
4. **例外**：`maxResultSizeChars = Infinity` 不落盘；含图片结果跳过这层落盘；其他非文本内容无法保存时保留原结果。空结果会补充“工具完成但没有输出”的标记。
5. **报错规则**：通用层不会仅因输出过大而报错；正常落盘失败时保留原结果，不自动截断、不改为工具失败。后续模型请求仍可能因上下文过大失败。工具直接抛出的错误由执行框架包装成错误 `tool_result`，不经过这一正常成功结果分支。
6. **聚合条件**：同一 API 消息中可处理的工具结果合计 >200,000 字符，会优先把最大的、尚未固定处理决策的结果落盘替换成预览。之前的处理决策会冻结以保持缓存前缀稳定；`Infinity` 工具退出该预算，保存失败也保留原结果，所以不保证严格压到上限。

入口：[toolExecution.ts:1478–1490](<../../claude-code-best/src/services/tools/toolExecution.ts#L1478-L1490>)；聚合调用：[query.ts:593–613](<../../claude-code-best/src/query.ts#L593-L613>)。`MAX_TOOL_RESULT_TOKENS = 100_000` 及其派生常量不能理解为所有工具都会执行的精确 token 硬拒绝；正常单结果路径主要使用字符落盘阈值。[通用常量](<../../claude-code-best/src/constants/toolLimits.ts#L5-L49>)

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

`PREVIEW_SIZE_BYTES` 虽带 `BYTES`，实际按 JS 字符串长度截取，不是 UTF-8 字节或 token；2,000 只限制预览正文，不包括路径、提示、标签等包装文字。[generatePreview():339–355](<../../claude-code-best/src/utils/toolResultStorage.ts#L339-L355>)

### Bash 读文件与其他工具

- **Bash / PowerShell**：不调用 Read 的文件大小/token 校验，改走 Shell 输出控制。普通 Bash 的 stdout/stderr 直接写任务输出文件，结束后默认只读回前 30,000 字节；成功的大输出保留路径，模型收到约 2,000 字符预览。`BASH_MAX_OUTPUT_LENGTH` 可调，配置上限 150,000；成功返回时的大输出持久化最多保留 64 MiB，超过会截掉。这个 64 MiB 不是运行期间的磁盘写入硬上限；失败和后台任务另有路径。[前缀读取](<../../claude-code-best/src/utils/task/TaskOutput.ts#L297-L311>) · [配置](<../../claude-code-best/src/utils/shell/outputLimits.ts#L1-L14>) · [64 MiB 处理](<../../claude-code-best/packages/builtin-tools/src/tools/BashTool/BashTool.tsx#L870-L895>)
- **Grep**：搜索正文，默认最多返回 250 行/条，支持 `offset`；`head_limit: 0` 可取消条数限制，通用大小预算仍生效。匹配行自身很长时，条数限制也不能控制字节数。[实现](<../../claude-code-best/packages/builtin-tools/src/tools/GrepTool/GrepTool.ts#L101-L124>)
- **Glob**：按文件名/路径模式找文件，默认最多 100 个，超出提示缩小范围，不检查正文。[实现](<../../claude-code-best/packages/builtin-tools/src/tools/GlobTool/GlobTool.ts#L151-L168>)
- **WebFetch**：进入副模型处理的路径时，输入先裁到 100,000 字符，再按 prompt 提取/总结；被裁掉的部分副模型也没看过。[实现](<../../claude-code-best/packages/builtin-tools/src/tools/WebFetchTool/utils.ts#L543-L573>)
- **MCP**：另有默认 25,000-token 计数预算；检测超限后通常存文本，含图或禁用大输出落盘时走裁剪。专用层的计数失败可放行；其落盘失败会返回错误说明，与通用层“保留原结果”不同。[计数](<../../claude-code-best/src/utils/mcpValidation.ts#L151-L177>) · [超限分支](<../../claude-code-best/src/services/mcp/client.ts#L2771-L2827>)

**区分三种行为**：按 `limit` 选范围 ≠ 超限报错 ≠ 存盘后只展示预览。脚本读过全文、输出已存盘，也不等于模型看过全文；Shell 输出保护不保证脚本自身的 CPU、内存和磁盘消耗安全。

### 计数 API 与本机诊断记录

- **直连 Anthropic**：`countTokensWithAPI()` 将所选正文包装成一条 `user` 消息、`tools: []`，按主循环模型调用 `POST /v1/messages/count_tokens`，使用返回的 `input_tokens`。不生成回答，不包含 Read 随后添加的行号，也不是整个会话的计数。官方将服务端返回值定义为估计值。[调用链:131–210](<../../claude-code-best/src/services/tokenEstimation.ts#L131-L210>) · [官方说明](<https://platform.claude.com/docs/en/build-with-claude/token-counting>)
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
| **位置** | [去重判定:538](<../../claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L538-L588>)<br>[stub 文案](<../../claude-code-best/packages/builtin-tools/src/tools/FileReadTool/prompt.ts#L7-L8>)<br>[file_unchanged 映射:701](<../../claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L701-L706>)<br>[FileStateCache](<../../claude-code-best/src/utils/fileStateCache.ts#L30-L59>)<br>[Read 写记录:1041](<../../claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L1041-L1046>)<br>[killswitch:551](<../../claude-code-best/packages/builtin-tools/src/tools/FileReadTool/FileReadTool.ts#L551-L554>) | 待填 | 待填 | 待填 |
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

---

## 工具延迟加载（Tool Search）

> 核对日期：2026-11-12（第二版，经官方 npm 包 sourcemap 还原源码逐行验证）。原版依据 [claude-code-sourcemap](../../claude-code-sourcemap/README.md)（官方 `@anthropic-ai/claude-code` v2.1.88 的 `cli.js.map` sourcesContent 还原）；CCB 行为基于本机复原源码（v2.1.888）+ git 历史（`7be08f53` → `8c157f07`）；DeepSeek 兼容性依据 [DeepSeek 官方文档](https://api-docs.deepseek.com/guides/anthropic_api)。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 原版：`defer_loading` + `tool_reference`（API 服务端机制，命中工具**会**进 tools 数组）；CCB 替换为自研 SearchExtraTools + ExecuteExtraTool（纯客户端，schema **永不**进 tools 数组） | 四档 exposure（默认 `codemode` 不声明）+ `tool_search`（BM25）命中后 `setActiveTools` 把 schema 注入下一轮声明 | 全量注册、schema 无差别进 tools 数组，无延迟机制；规模靠 PTC（`run_code` + SDK 进 system prompt）换形态 | 待填 |
| **值** | 原版：MCP 默认延迟、`alwaysLoad` 豁免、`shouldDefer` 显式标记；三模式 tst（未设即默认）/ tst-auto（阈值=上下文×10%）/ standard；CCB：CORE_TOOLS 白名单 ~29 个，搜索=关键词×0.4+TF-IDF×0.6 | exposure ∈ {direct, codemode(默认), deferred, hidden}，`toolExposure` 按名/通配符覆盖；BM25 name×3/namespace×0.5；`mcp_servers` 段预算 4096 字符；tool_search 默认 limit 8 | — | — |
| **位置** | 原版：[ToolSearchTool](<../../claude-code-sourcemap/restored-src/src/tools/ToolSearchTool/ToolSearchTool.ts>) · [isDeferredTool 散特判](<../../claude-code-sourcemap/restored-src/src/tools/ToolSearchTool/prompt.ts#L62>) · [discovered 进 tools 数组](<../../claude-code-sourcemap/restored-src/src/services/api/claude.ts#L1158>) · [tool_reference 产出](<../../claude-code-sourcemap/restored-src/src/tools/ToolSearchTool/ToolSearchTool.ts#L461>) · [关键词打分器](<../../claude-code-sourcemap/restored-src/src/tools/ToolSearchTool/ToolSearchTool.ts#L186>)<br>CCB：[CORE_TOOLS](<../../claude-code-best/src/constants/tools.ts#L137>) · [无条件排除](<../../claude-code-best/src/services/api/claude.ts#L1188>) · [TF-IDF 索引](<../../claude-code-best/src/services/searchExtraTools/toolIndex.ts#L80>) · [发现门](<../../claude-code-best/packages/builtin-tools/src/tools/ExecuteTool/ExecuteTool.ts#L88>) | [exposure](<../../pi/packages/coding-agent/src/core/mcp-servers.ts#L61>) · [tool_search](<../../pi/packages/coding-agent/src/extensions/tool-search/tool.ts#L300>) | [syncTools](<../../deepseek-harness/packages/mcp/mcp-client/src/tools.ts#L126>) · [PTC](<../../deepseek-harness/packages/core/tool-executor-ptc/src/index.ts#L45>) | — |
| **注** | 原版无 ExecuteExtraTool（tools/ 40 个目录，全仓库无 CallTool 代理）；CCB 里 defer_loading/tool_reference 是化石（`api.ts` deferLoading 选项、`betas.ts` beta header 零调用） | 命中后 schema 真进下一轮声明；激活记录进 transcript，resume/fork 保留 | PTC 下模型只见 `run_code`，其余以 SDK 文本进 system prompt——省往返与中间结果，不省 schema | — |

### 自配模型（DeepSeek 等）会开启延迟加载吗：不会，拦在注册闸

**结论：`ANTHROPIC_BASE_URL` 指向非 anthropic.com 端点且未显式设 `ENABLE_TOOL_SEARCH` 时，延迟加载整体关闭，MCP 工具全量内联、一切正常可用。** 模型名检查（黑名单只有 `['haiku']`）对自配模型**从不拦截**——拦它的是端点检查。

调用链（官方 v2.1.88，三步）：

```
① tools.ts:249  getAllBaseTools()（会话启动，唯一决策点）
   └─ isToolSearchEnabledOptimistic()
        └─ toolSearch.ts:302  isFirstPartyAnthropicBaseUrl()
             ANTHROPIC_BASE_URL host ∉ ['api.anthropic.com'] → false
             三个条件同时成立才拦：未设 ENABLE_TOOL_SEARCH ∧ firstParty ∧ 非官方 host
             ⇒ optimistic = false → ToolSearchTool 不注册
② isToolSearchEnabled (claude.ts:1120) 每请求终审
   └─ toolSearch.ts:428  isToolSearchToolAvailable(tools) → 找不到 ToolSearchTool
        ⇒ return false（日志理由 mcp_search_unavailable）
③ useToolSearch = false → 无 defer_loading 字段、无 beta header、无名单公告
   ⇒ 所有 MCP 工具 schema 全量进 tools 数组，模型直接调用
```

要点拆解：

- **`isToolSearchEnabled` 内部没有任何 URL 检查**。URL 否决权在 `isToolSearchEnabledOptimistic`（L270）的 L298-310：`!ENABLE_TOOL_SEARCH ∧ firstParty ∧ !isFirstPartyAnthropicBaseUrl()` 三条同时成立才拦；`isFirstPartyAnthropicBaseUrl` 白名单只有 `api.anthropic.com`（ant 用户加 `api-staging`）。它拦的理由写在注释里（L286-296，issue #30912）：第三方网关通常不转发 tool_reference，模型一搜工具下一轮就 400——所以提前关门，"宁可没有延迟加载，不发会炸的请求"。
- **模型名门（L419 `modelSupportsToolReference`）对自配模型恒放行**：负向黑名单只有 `['haiku']`（可被 GrowthBook `tengu_tool_search_unsupported_models` 在线扩充），`deepseek-chat` 等不命中 → true。设计意图（注释）："ensures new models work by default"——对官方新模型安全，对第三方端点的模型名则完全测不准。
- **`isToolSearchEnabledOptimistic` 共 7 个调用点**，全部是注册闸的下游一致性维护：①tools.ts:249 注册（决策点）②ToolSearchTool 自身 isEnabled ③④normalizeMessagesForAPI 剥历史里的 tool_reference/caller（L2104/L2206）⑤deferred_tools_delta 附件 ⑥chrome 提示 ⑦盲调教学提示（L587）。主流程只需记 ①：它不注册，②-⑦ 全部跟随关闭，L428 必然拦。
- **两个函数分工**：optimistic = 会话级粗筛（同步、便宜，管注册/清洗/提示），`isToolSearchEnabled` = 请求级终审（算模型/阈值，claude.ts:1120 请求 + compact.ts:1265 压缩 + analyzeContext 统计）。合法中间态：optimistic 放行但终审否决（tst-auto 低于阈值时每轮如此）。
- **强开 `ENABLE_TOOL_SEARCH=true` 的后果**：绕过 URL 闸（注释明说显式配置 = 用户断言网关支持），四道检查全过 → defer_loading/tool_reference 照发 → DeepSeek 端点要么 422/400、要么丢弃字段退化 standard；模型一调 ToolSearchTool，下一轮历史带 tool_reference（DeepSeek content 类型表没有的类型）大概率 400。表现为"第一问正常、一搜工具就断"，**不会安静降级**。
- **验证**：`--debug` 日志可见两条——启动时 `[ToolSearch:optimistic] disabled: ANTHROPIC_BASE_URL=... is not a first-party Anthropic host`（L307），每轮 `Tool search disabled: ToolSearchTool is not available`（L430）。

### 原版机制：一个心智模型 + 四件套

**一句话：`defer_loading: true` 的工具对模型默认不可见；唯一让它重新可见的方式，是消息历史里出现指向它的 `tool_reference` 块。**

```
可见 = 非 defer_loading 工具 ∪ { T | 消息历史中存在 tool_reference(T) }
```

四件套：**藏**（defer_loading 标记，API 剥离出模型可见区）→ **知**（`<available-deferred-tools>` 名单，只有名字+描述）→ **取**（ToolSearchTool：发现入口 + 产出凭据 + 客户端记账数据源）→ **放**（API 服务端：tool_reference 触发展开 + 按 schema 强校验调用）。

关键辨析——schema 在三层的状态不同：**HTTP 请求体里有**（全程，展开的原料只能来自请求体，API 无状态不落盘）→ **API 服务端缓冲区有** → **模型上下文没有**（直到 tool_reference 展开）。官方缓存注释（claude.ts:1460）："the API strips them from the prompt, so they never affect the actual cache key"。第三方科普（[腾讯云文章](https://developer.cloud.tencent.com/article/2684265)）称"延迟工具只发名称不发 schema"是错的。

#### 调用链路（简化版报文时序，一次 MCP 调用最少两轮）

```
【请求1】tools[] = [核心工具…, ToolSearchTool, mcp__slack__send_message{完整schema, defer_loading:true}]
         messages 末尾附 <available-deferred-tools> 名单
【响应1】模型无 schema（只见名字+描述），凭常识发起 tool_use → ToolSearchTool({query:"select:mcp__slack__send_message"})
【客户端】纯本地关键词搜索（不经 LLM）→ 回填 tool_result:
         content:[{type:"tool_reference", tool_name:"mcp__slack__send_message"}]  ← 只有名字
【请求2】历史含 tool_reference → ① API 服务端展开 schema 进模型上下文（非客户端）
                                    ② 客户端把该工具加回 tools[]（discovered 过滤分支）
【响应2】模型直接原生调用 → tool_use{ name:"mcp__slack__send_message", input:{channel,text} }
【客户端】解析 mcp__ 前缀 → JSON-RPC tools/call{ name:"send_message" } 转发 MCP server → 结果回填 tool_result
```

#### 双通道（源码实锤，最常见误读点）

命中后 schema 通过**两条独立通道**生效：

- **通道 A（模型看见）**：tool_reference 由 **API 服务端**展开进上下文——客户端全程不碰被延迟工具的 schema；
- **通道 B（调用合法）**：客户端 `extractDiscoveredToolNames` 扫历史里的 tool_reference（唯一来源），把工具**加回下一轮 tools 数组**（claude.ts:1158 `return discoveredToolNames.has(tool.name)`）——API 只接受对已声明工具的调用，光读到 schema 不够。

双通道代价：发现过的工具进 tools 数组 → 缓存前缀从那一点失效（"不参与 KV Cache"只对未发现的成立）。compact 边界携带 `preCompactDiscoveredTools`，压缩后不用重搜。模型兼容守门：Haiku 不支持 tool_reference（负向名单 toolSearch.ts:227），整条机制降级全内联。

#### ToolSearchTool 原理

普通 zod 工具（`isDeferredTool` 特判永不延迟）+ 三步：① 候选集 = `tools.filter(isDeferredTool)`（进程内存）；② 打分（见下节算法表）；③ **构造凭据**——返回 tool_reference（全系统唯一产生点），下游三方消费：API 展开（让模型看见）、客户端记账（让调用合法）、compact 携带（跨压缩）。定位一句话：**defer_loading 管"藏"（换出），ToolSearchTool 管"取"（缺页处理器），API 管"放"（展开+强校验）**——只有换出没有缺页处理，被换出的页永久不可访问。官方建议 >10K tokens 或 >10 工具启用；实测 58 工具 ~55K tokens 省 85%，Opus 4 准确率 49%→74%。

### 算法：三代实现，原版不是 TF-IDF

| 实现 | 算法 | 细节 |
|---|---|---|
| 原版 v2.1.88 | **位置加权布尔匹配**（非 TF-IDF、非相似度） | 权重表硬编码：词=名字段 +12(MCP)/+10；字段含词 +6/+5；整名含词 +3；searchHint +4；词边界命中描述 +2。`+term` 必选词 AND 预过滤；精确名/`mcp__server` 前缀快速路径。与词频/长度/稀有度无关——同词 5 次与 1 次同分 |
| CCB | 关键词×0.4 + **TF-IDF 余弦**×0.6 | 三路加权 TF（name 3.0/hint 2.5/desc 1.0）×IDF，进程内存索引；新增 `discover:`（返回 schema 纯文本）、中文 bigram 门槛 |
| pi | **Okapi BM25**（k1=1.2/b=0.75） | 词频饱和+长度归一；name×3 IDF 压倒描述任意次提及、namespace×0.5、精确名置顶；中文走 Intl.Segmenter |

演进：权重表 → 统计检索 → 带归一化的统计检索。三代全停在经典 IR 无人用 embedding——查询是 LLM 写的，语义桥接在查询端完成（"给团队发消息"会被模型写成 "slack send message"），检索端只处理词面。推论：原版排序对**命名极敏感**（名字命中一次压倒描述五次），searchHint 与 MCP 命名规范因此重要。词法天花板："美国总统"vs"特朗普"零交集恒 0 分；BM25 270+ 工具时 top-1 掉到 14%（top-5 87%），大规模需混合 embedding。

### CCB 演化与现行机制

演化：fork 初期继承原版 → F001（`7be08f53`）双轨（`modelSupportsToolReference()` 分流）→ `8c157f07`（2026-05-09）统一自研：删分流/defer_loading/beta header/tool_reference，输出改纯文本 → `c14b7ead` **删掉原版 filter 里的 discovered 分支改无条件排除**（与原版的分水岭）。动因（spec 三条局限）：① tool_reference 仅 Anthropic 支持（DeepSeek 端点 `anthropic-beta: Ignored`）；② 命中后动态注入破坏 prompt cache；③ 跨 7 provider 统一。

| 维度 | 原版 | CCB |
|---|---|---|
| 延迟判定 | `shouldDefer`/`isMcp`/`alwaysLoad` 散特判（无 CORE_TOOLS） | CORE_TOOLS 白名单制 |
| 凭据形态 | `tool_reference` 协议块 | 一行名字文本（自家正则解析） |
| schema 进 tools 数组 | **是**（discovered 进入） | **永不**（无条件排除，数组恒定） |
| 执行 | 模型直接调用（API 强校验） | ExecuteExtraTool 代理（发现门+zod 预校验+防死循环；`buildSchemaNotSentHint` 教学，埋点 `tengu_deferred_tool_schema_not_sent`） |
| 依赖 | beta header，仅 Anthropic | 无，7 provider 通用 |
| cache | 未发现稳定，发现后失效 | 恒定不失效 |

公告机制两版相同：`<available-deferred-tools>` 报名字（ant 用户走 `deferred_tools_delta` 持久附件），diff 后仅新增时注入。CCB `extractDiscoveredToolNames` 的 docstring"用于加入后续 API 请求"是过时注释，现仅用于发现门与 compact 边界。

### 社区方案四流派（+硬上限）

数据来源：[MCPProxy 对比文](https://mcpproxy.app/blog/2026-03-19-bm25-vs-embeddings-vs-lua/)（2026-03）。

| 流派 | 代表 | 思路 | 代价 |
|---|---|---|---|
| 网关 + BM25 | MCPProxy | 代理藏起全部工具，只暴露 `search_tools` 元工具，零配置 | 词法天花板（270+ 工具 top-1 14%），向混合检索演进 |
| 平台 + embedding | Speakeasy Dynamic Toolsets | search/describe/execute 三元工具 + 语义检索，省 token 96% | 需 embedding 模型；调用 ×2-3，耗时 +50% |
| Code mode | [Cloudflare Code Mode](https://blog.cloudflare.com/code-mode/)、my-cool-proxy（Lua） | 工具翻译成 TS/Lua API，模型写代码编排，中间结果不进上下文 | 维护脚本/运行时 |
| 客户端内置 | 官方 Claude Code、CCB、pi | 本节所述方案 | 各家实现，不可移植 |
| （硬上限） | Cursor 40 / Copilot 128 | 超限让用户自己裁 | 粗暴 |

共识：客户端本地词法匹配是默认起点；演进方向是词法+embedding 混合与 code mode；API 层 `defer_loading` 因 provider 锁定可能是过渡形态。腾讯云文章"如果模型没有 defer_loading API"一节的 CallTool 代理假想方案即 CCB ExecuteExtraTool 的现实版本，文章同时点出其代价（失去 API 层强校验）——与 CCB 的 zod 兜底互为印证。

---

## Web 能力（搜索与抓取）

> 核对日期：2026-11-12。依据 [claude-code-sourcemap](../../claude-code-sourcemap/README.md)（官方 `@anthropic-ai/claude-code` v2.1.88 sourcemap 还原源码），CCB 分叉单列。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 两者都默认开启：`getAllBaseTools()` 静态列入数组（注册层不门控），门在 `getTools()` 的 `isEnabled()`。<br>**WebSearch** = 客户端工具外壳，`call()` 内部再发一次完整 LLM 请求，把服务端工具 `web_search_20250305` 注入该请求，服务端在**这次请求内部**跑搜索循环，客户端消费完整个流后压成纯文本回灌。<br>**WebFetch** = axios 裸抓 HTTP → turndown 转 Markdown → Haiku 按 prompt 二次提炼，只回传答案 | — | — | — |
| **值** | WebSearch 内层：`tools: []`、`extraToolSchemas: [web_search_20250305]`、`max_uses: 8`、`systemPrompt` 单行固定、`querySource: 'web_search_tool'`<br>WebFetch 入参：`url` + `prompt`，两者均必填<br>WebFetch：URL ≤2000 字符、HTTP 60s / 10 MiB / 重定向 10 跳、Markdown 100k 字符上限、缓存 15min·50MiB、预批准域名 89 条 | — | — | — |
| **位置** | [注册](<../../claude-code-sourcemap/restored-src/src/tools.ts#L207-L209>) · [isEnabled 过滤](<../../claude-code-sourcemap/restored-src/src/tools.ts#L325>) · [WebSearch isEnabled](<../../claude-code-sourcemap/restored-src/src/tools/WebSearchTool/WebSearchTool.ts#L168-L193>) · [WebSearch 嵌套请求](<../../claude-code-sourcemap/restored-src/src/tools/WebSearchTool/WebSearchTool.ts#L254-L291>) · [extraToolSchemas 合并](<../../claude-code-sourcemap/restored-src/src/services/api/claude.ts#L1385-L1396>) · [WebFetch 模型分叉](<../../claude-code-sourcemap/restored-src/src/tools/WebFetchTool/WebFetchTool.ts#L259-L278>) · [axios 抓取](<../../claude-code-sourcemap/restored-src/src/tools/WebFetchTool/utils.ts#L262-L329>) · [Haiku 提炼](<../../claude-code-sourcemap/restored-src/src/tools/WebFetchTool/utils.ts#L484-L529>) | — | — | — |
| **注** | ① 两者都 `shouldDefer: true`，ToolSearch 默认模式下首次使用要先 ToolSearch 一轮。<br>② WebSearch 的 `isEnabled` 只看 provider（firstParty/foundry 开、vertex 看模型名、bedrock 关），**不认模型是不是 Claude**；用 `ANTHROPIC_BASE_URL` 接第三方代理仍判 firstParty → 工具启用但服务端能力不存在，静默失败。<br>③ WebFetch 的原始 HTML 只在中间存在，`tool_result` 里没有，`result` 是 Haiku 的答案。 | — | — | — |

### 模式分辨：谁执行 + 是否又开模型

| 模式 | 执行者 | 额外 LLM | 代表 |
|---|---|---|---|
| A 混合式（服务端内联） | API 服务端 | 否 | `advisor` |
| B 客户端确定性执行 | 客户端 | 否 | `Bash` / `Read` / CCB 的 tavily adapter |
| C 客户端外壳 + 内部 LLM | 客户端发起，服务端执行 | 是，1 次 | `WebSearch` / `WebFetch` |

关键：**C 内部跑的就是 A**。WebSearch 内层请求把服务端工具放进 `tools`，那次请求内部就进入了 A 模式。C = 用一次嵌套请求借来 A 的能力，再把结果降维回外层。

### 注册与门控

`getAllBaseTools()` 里两者都是静态 import + 直接列入数组，没有任何 `feature()` 包裹。真正的门在 `getTools()` 末尾：

| 工具 | `isEnabled()` |
|---|---|
| `WebFetch` | **未定义**，走 `buildTool` 默认 `() => true`，恒启用 |
| `WebSearch` | 自定义，按 provider 判：`firstParty`/`foundry` → true；`vertex` → 模型名含 `claude-opus-4`/`sonnet-4`/`haiku-4`；其余 → **false** |

`getAPIProvider()` 只读 `CLAUDE_CODE_USE_BEDROCK`/`VERTEX`/`FOUNDRY` 三个环境变量，其余一律 `firstParty`。

### WebSearch：内层其实是一次完整模型请求

`call()` 内部（WebSearchTool.ts:254-291）再发一次 `queryModelWithStreaming`：

```ts
messages: [userMessage],          // 1 条："Perform a web search for the query: …"
systemPrompt: ['You are an assistant for performing a web search tool use'],
tools: [],                        // 客户端工具全部不给
extraToolSchemas: [toolSchema],   // → [{type:'web_search_20250305', name:'web_search', max_uses:8, allowed_domains, blocked_domains}]
model: useHaiku ? getSmallFastModel() : mainLoopModel,
```

`useHaiku` 读 GrowthBook `tengu_plum_vx3`（默认 false）。**默认走主模型 + 继承 thinking**；实验组走小模型 + 禁 thinking + 强制 `tool_choice`。

服务端在那次请求内部循环（最多 `max_uses: 8` 次）：`server_tool_use` → `web_search_tool_result` → … → 最终 text + citations。

客户端在流里只做两件服务 UI 的事：正则抠半截 JSON 的 query、数结果条数。流结束后 `makeOutputFromSearchResponse()` 压成 `{query, results, durationSeconds}`，再拼成纯文本 `tool_result`。

**信息损失**：`page_age`、`encrypted_content`、`citations_delta`（主流里是 `// TODO: handle citations`）全丢，只留 title + url。

### 为什么 WebSearch 要绕这一圈

同一个仓库里 `advisor` 也是服务端工具，但走的是模式 A——`extraToolSchemas.push({type:'advisor_20260301', …})` 直接进**主请求**的 tools，主流 streaming parser 只是旁听，一次请求搞定。

所以**协议上不要求单独请求**。WebSearch 选嵌套式的真实理由：

1. `allowed_domains` / `blocked_domains` 是 **per-call 参数**，`makeToolSchema(input)` 用模型本次入参构造工具定义。混合式下工具定义必须会话级固定，模型就没法每次调用改变域名过滤。
2. `validateInput` 要检查两者不能同时出现，混合式下这个校验只能交给 API。
3. 结果降维省 token，`encrypted_content` 不进主上下文。

（"保 prompt cache" 不是理由——`advisor` 注释明说追加在带 `cache_control` 的 `toolSchemas` 之后即可。旁证：全仓 `grep pause_turn` = 0 处，混合式必需的续接机制缺失。）

### WebFetch：真抓 HTTP，但返回的是 Haiku 的答案

入参 `{url, prompt}`，**两者均必填**，没有"有 prompt 才调模型"这种分支。

四阶段：

1. **校验**：URL ≤2000、拒绝 `user:pass@`、hostname 至少两段
2. **域名预检**：`GET api.anthropic.com/api/web/domain_info?domain=…` 查 `can_fetch`（可被 `skipWebFetchPreflight` 跳过）
3. **axios GET**：`maxRedirects: 0` 手动跟随、只允许同 host 或 www 增删、10 跳上限、10 MiB / 60s
4. **转 Markdown + Haiku**：`text/html` → turndown → Markdown；其他类型原样

Haiku 调用（`queryHaiku`）：`model = getSmallFastModel()`、`thinking: disabled`、`tools: []`、`cache: false`。

**唯一跳过 Haiku 的快路径**：预批准域名 + 响应本身是 `text/markdown` + <100k，三条件同时成立。实践极少命中。

**返回给主循环的只有 `result`（Haiku 的答案）**，HTML 不返回。所以不能用它抓 JSON API——Haiku 会"理解"那段 JSON 然后给你散文，要精确抓取应该用 `Bash` + `curl`。

### CCB 的改造

主线：**把"只能连 Anthropic"的部分换成自建 HTTP 服务或纯客户端实现**。

| 维度 | 官方 | CCB |
|---|---|---|
| `WebSearch.isEnabled()` | provider 门控 | **恒 `return true`**，判断整体删除 |
| 搜索后端 | 服务端 `web_search_20250305` | adapter 五选一，**默认 tavily** |
| 默认搜索是否调模型 | 是 | **否**（纯 HTTP） |
| WebFetch 抓取 | axios 直连目标站 | 默认 Tavily `/extract` 代抓 |
| WebFetch 域名预检 | 有 | **完全删除** |
| `webFetchAdapter` 分支 | 无 | `tavily`（默认）/ `http` |

**WebSearch 默认请求** `POST https://tavily.claude-code-best.win/search`，body `{query, search_depth, max_results, include_domains, exclude_domains}`。其余可选：`bing`（抓 `www.bing.com/search` HTML 正则解析）、`brave`（`api.search.brave.com/res/v1/llm/context`，需 key）、`exa`（`mcp.exa.ai/mcp` JSON-RPC）、`api`（官方路径）。

**WebFetch 默认请求** `POST https://tavily.claude-code-best.win/extract`，body `{urls:[url]}`，返回的 `raw_content` 已是 Markdown——**跳过 turndown 和 Haiku**。

`api` adapter 虽保留了官方嵌套逻辑，但**在第三方 provider 下失效**：`claude.ts:1343/1358/1371` 三条 dispatch 提前 return，早于 `:1485` 的 `extraToolSchemas` 合并点，且三个第三方文件都不读该字段。这就是默认设成 tavily 而非 api 的原因。

CCB 还有一个行为松动：`prompt` 虽仍必填，但 `call()` 里加了 `if (prompt && prompt.trim())`，模型传空串即可直接拿 Markdown 原文、跳过 Haiku。

| 场景 | 官方 | CCB |
|---|---|---|
| DeepSeek + WebSearch | ❌ 判定 firstParty → 工具启用但服务端能力不存在 | ✅ 走 tavily HTTP |
| DeepSeek + WebFetch | ❌ 域名预检请求 Anthropic 失败 | ✅ 无预检 |
| Bedrock + WebSearch | ❌ `isEnabled()` false | ✅ 恒启用 |

### 待验证

- ⚠️ `pause_turn` 未实现（全仓 0 处），长搜索被截断后不续接。
- ⚠️ `citations_delta` 被丢弃（`claude.ts:2085` TODO）。
- ⚠️ `tavily.claude-code-best.win` 是项目自建域名（非官方 `api.tavily.com`），仓库内无该代理实现；默认配置下 query 和 URL 会经过第三方服务器。
