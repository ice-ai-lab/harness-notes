# 2. 工具 · 机制详情

> 对应 [../harness-matrix.md](../harness-matrix.md) 第 2 层信号矩阵。字段为行、harness 为列。

---

## 默认工具集

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | `getAllBaseTools()` 列全 → `getTools()` 过滤（deny → REPL 隐藏 → `isEnabled()`）→ `assembleToolPool()` 合并 MCP（内置优先去重、排连续前缀保 prompt cache）→ 每回合现算 | — | — | — |
| **值** | 无条件 ~25 + 条件默认真 ~16 + 环境/flag 特定 ~14（见下清单） | — | — | — |
| **位置** | `CCB: src/tools.ts:218` / `:304` / `:378` / `:334` | — | — | — |
| **注** | ① MCP 资源类工具被 specialTools 剔除，按连接状态动态注入；② SIMPLE 模式只剩 Bash/Read/Edit；③ 工具搜索开启时非 CORE_TOOLS 白名单工具改经 SearchExtraTools + ExecuteExtraTool；④ ant 构建内嵌 bfs/ugrep 时 Glob/Grep 不注入 | — | — | — |

### 工具清单（精简版）

**A. 无条件注入**：`Agent`（子代理）· `TaskOutput` / `TaskStop`（后台任务）· `Bash` · `Read` / `Edit` / `Write` / `NotebookEdit` · `Glob`* / `Grep`* · `ExitPlanMode` / `EnterPlanMode` · `artifact` · `WebFetch` · `WebSearch` · `TodoWrite` · `AskUserQuestion` · `Skill` · `LocalMemoryRecall` · `VaultHttpFetch` · `EnterWorktree` / `ExitWorktree` · `SendMessage` · `TeamCreate` / `TeamDelete` · `CronCreate` / `CronDelete` / `CronList` · `DiscoverSkills` · `SearchExtraTools` · `ExecuteExtraTool`（* 条件默认真）

**B. 条件包含（默认真）**：Tasks V2 四件套（`TaskCreate/Get/Update/List`，headless 需 env）· `Workflow` · `Sleep` · `RemoteTrigger` · `Monitor` · `SendUserFile` · `PushNotification` · `Goal` · `SendUserMessage`（仅 bridge/remote）

**C. 环境/flag 特定（默认不开）**：`Config` / `Tungsten` / `REPL` / `SuggestBackgroundPR`（USER_TYPE=ant）· `PowerShell`（Windows）· `LSP` · `VerifyPlanExecution` · `TestingPermission` · `WebBrowser` / `OverflowTest` / `SubscribePR` / `Snip` / `CtxInspect` / `ReviewArtifact`（feature 未启用）

---

## 工具输出过长

> 核对日期：2026-11-11，基于本机 CCB 复原源码默认行为；env 与实验配置可覆盖。

三级处理：**工具内限制 → 通用单结果落盘 → 单消息聚合预算**。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 工具先自行限范围/报错 → 正常 `tool_result` 通用大小处理 → 发请求前查单消息聚合预算 | 待填 | 待填 | 待填 |
| **值** | Read 256 KiB（无 limit 时）/ 25,000 token / >100,000 字节快速拒绝 / 粗估 >6,250 才调计数 API；通用单结果 `min(maxResultSizeChars, 50,000)` 字符落盘；聚合预算 200,000 字符；预览 2,000 字符 | 待填 | 待填 | 待填 |
| **位置** | Read `FileReadTool.ts:1028–1039` / `:754–780`；通用 `toolResultStorage.ts:55–78` / `:272–333` / `:769–868` | 待填 | 待填 | 待填 |
| **注** | 字节≠字符≠token；通用层过大存盘而非报错；落盘失败保留原结果，故聚合预算非严格硬上限 | 待填 | 待填 | 待填 |

**Read 关键逻辑**：粗估 `round(len/4)`（json 类 `len/2`）≤6,250 直接通过，否则调计数 API，失败回退粗估；最终 >25,000 报错，不自动截断。`limit` 只限行数不限单行长度。粗估参数名叫 `bytesPerToken` 实为字符串长度；「默认 2,000 行上限」只是提示词说法，实现没有。

**通用落盘层**：>阈值全文存 `tool-results/`（.txt/.json），模型收 `<persisted-output>` + 路径 + 短预览；`maxResultSizeChars=Infinity` 不落盘；图片跳过。预览生成：截前 2,000，若其中最后换行 index >1,000 截到换行。Bash/PowerShell 走 Shell 输出控制（默认回读 30,000 字节、大输出留路径、持久化上限 64 MiB）；Grep 250 条、Glob 100 个、MCP 另有 25,000 token 预算（超限存文本或裁剪，落盘失败报错，与通用层不同）。

**计数 API**：正文包成单条 user 消息调 `POST /v1/messages/count_tokens`；Gemini 分支本地估、Bedrock 专用接口、OpenAI 兼容模式无分支。⚠️ 2026-11-11 本机快照（OpenAI 兼容→DeepSeek 经 tt-switch）计数路径 404，Read 回退粗估——不代表所有配置如此。

**一句话**：工具内部管范围/报错，通用层管落盘+预览，计数不可用就粗估；阈值不能混同一个精确 token 硬上限。

---

## Read 重复读取去重

> 核对日期：2026-11-11，CCB v2.1.888。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 会话级 `readFileState` 记录 `{content, mtime, offset, limit}`；命中「同路径+同区间+mtime 未变」回 stub，让模型引用上文已有 `tool_result` | 待填 | 待填 | 待填 |
| **值** | 四道闸门：记录存在 ∧ 非自动注入且真实 Read ∧ offset/limit 相同 ∧ `floor(mtimeMs)` 逐毫秒相等；stub ≈30 token；LRU 100 条/25MB；动机：~18% Read 同文件碰撞 | 待填 | 待填 | 待填 |
| **位置** | 判定 `FileReadTool.ts:538–588` · stub `prompt.ts:7–8` · 记录 `:1041–1046` · FileStateCache `fileStateCache.ts:30–59` | 待填 | 待填 | 待填 |
| **注** | ① 纯客户端，killswitch `tengu_read_dedup_killswitch`；② 判 mtime 非内容哈希，`touch -d` 回拨会误判；③ 图片/PDF 不去重；④ 省的是"第二份全文"，stub 有效性依赖旧结果还在历史里；⑤ 换区间覆写记录，软链接不同键；⑥ Magic Docs 先删记录再 Read 绕过 | 待填 | 待填 | 待填 |

**动机**：省重复 token（第二份全文每轮都背着）、延后压缩（少触发高信息损失的摘要回填）、减少版本歧义（改前/改后两份全文）、代价极低（只多一次 stat，可随时关闭）。

**存储形态**：进程内存 LRU，键为归一化绝对路径；每会话一实例，print/SDK 从 `initialMessages` 重建（resume 后 Edit 仍受先读后写约束）。`content` 存整份文本 = 25MB 上限原因；同一本账兼作 `getChangedFiles` 与 Edit/Write「先读后写」数据源。

**失效时机**：

| 场景 | 结果 |
|---|---|
| 被任何写者改过 | mtime 变 → 全量读 |
| 换 offset/limit | 全量读并覆写记录 |
| Edit/Write 后读 | 写记录 `offset: undefined` → 闸门②永不成立 |
| 自动注入 CLAUDE.md / 图片 PDF / LRU 淘汰 / 子 agent | 不写记录或记录缺失 → 全量读 |
| mtime 被外部还原 | **误判未变**返回 stub（无内容哈希兜底） |
| full/partial compact | 快照→clear→回填最近 5 文件（各 ≤5k token）；不清会悬空 stub；没回填的文件 Edit 报"被修改"（实为被压缩掉） |
| microcompact | 只清 tool_result 内容**不碰 readFileState**——唯一"stub 指向已清空结果"的缺口 |

**与 Edit 先读后写**：共用一本账但判定独立——去重要求真实 Read，校验接受任何条目；去重失效不影响校验正确性。

---

## 工具延迟加载（Tool Search）

> 核对日期：2026-11-12，官方 npm v2.1.88 sourcemap 还原 + CCB v2.1.888。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 原版：`defer_loading` + `tool_reference`（API 服务端展开，命中工具**会**进 tools 数组）；CCB：自研 SearchExtraTools + ExecuteExtraTool（纯客户端，schema **永不**进 tools 数组） | 四档 exposure（默认 codemode 不声明）+ `tool_search`(BM25) 命中后 schema 注入下一轮 | 全量注册无延迟；规模靠 PTC（`run_code` + SDK 进 system prompt）换形态 | 待填 |
| **值** | 原版：MCP 默认延迟、三模式 tst/tst-auto(阈值=上下文×10%)/standard；CCB：CORE_TOOLS 白名单 ~29 个，关键词×0.4+TF-IDF×0.6 | exposure ∈ {direct, codemode(默认), deferred, hidden}；BM25 name×3/namespace×0.5；`mcp_servers` 预算 4096 字符；默认 limit 8 | — | — |
| **位置** | 原版 `restored-src/src/tools/ToolSearchTool/ToolSearchTool.ts` · 进 tools 数组 `claude.ts:1158` · CCB CORE_TOOLS `constants/tools.ts:137` · 无条件排除 `claude.ts:1188` · TF-IDF `toolIndex.ts:80` | `pi: mcp-servers.ts:61` · `tool-search/tool.ts:300` | `deepseek-harness: mcp-client/src/tools.ts:126` · `tool-executor-ptc/src/index.ts:45` | — |
| **注** | 原版无 ExecuteExtraTool；CCB 里 defer_loading/tool_reference 是化石（零调用） | 命中后 schema 真进下一轮声明；激活记录进 transcript，resume 保留 | PTC 下模型只见 `run_code`——省往返不省 schema | — |

**自配模型（DeepSeek 等）不会开延迟加载**：`ANTHROPIC_BASE_URL` 指向非 anthropic.com 且未设 `ENABLE_TOOL_SEARCH` 时，`isToolSearchEnabledOptimistic` 在注册闸（`tools.ts:249`）直接不注册 ToolSearchTool，后续终审必拦——MCP 工具全量内联。模型名黑名单只有 `['haiku']`，从不拦自配模型；拦的是端点。第三方网关通常不转发 tool_reference，"宁可没有延迟加载，不发会炸的请求"。强开 `ENABLE_TOOL_SEARCH=true` 会表现为"第一问正常、一搜工具就断"，不会安静降级。

**原版心智模型**：`可见 = 非 defer_loading 工具 ∪ { T | 历史中存在 tool_reference(T) }`。四件套：**藏**（defer_loading 换出）→ **知**（`<available-deferred-tools>` 名单）→ **取**（ToolSearchTool 产出 tool_reference，全系统唯一产生点）→ **放**（API 服务端展开 + 强校验）。schema 三层状态不同：HTTP 请求体里有 → API 缓冲区有 → 模型上下文没有（直到展开）；"不参与 KV cache"指模型侧，官方注释明确。

**双通道（最常见误读点）**：命中后 ① API 服务端把 schema 展开进上下文（模型看见）；② 客户端扫历史 tool_reference 把工具加回下一轮 tools 数组（调用合法）。代价：发现过的工具使缓存前缀失效；compact 边界携带 `preCompactDiscoveredTools` 免重搜；Haiku 不支持 → 降级全内联。

**三代算法**：

| 实现 | 算法 | 细节 |
|---|---|---|
| 原版 | 位置加权布尔匹配（非 TF-IDF） | 词+12/+10、字段+6/+5、searchHint+4、描述词边界+2；`+term` AND 预过滤；与词频长度无关 |
| CCB | 关键词×0.4 + TF-IDF 余弦×0.6 | 三路加权 TF ×IDF；`discover:` 返回 schema 纯文本；中文 bigram |
| pi | Okapi BM25（k1=1.2/b=0.75） | 词频饱和+长度归一；精确名置顶；中文 Intl.Segmenter |

三代全停在经典 IR，无人用 embedding——查询是 LLM 写的，语义桥接在查询端完成。推论：排序对**命名极敏感**；词法天花板："美国总统"vs"特朗普"恒 0 分；BM25 270+ 工具 top-1 掉到 14%。

**CCB 演化**：继承原版 → 双轨分流 → `8c157f07` 统一自研 → `c14b7ead` 改无条件排除（分水岭）。动因：tool_reference 仅 Anthropic 支持、命中后动态注入破坏 prompt cache、跨 7 provider 统一。CCB 执行走 ExecuteExtraTool 代理（发现门 + zod 预校验 + 防死循环），cache 恒定不失效。

**社区流派**：网关+BM25（MCPProxy）· 平台+embedding（Speakeasy，省 token 96% 但调用 ×2-3）· Code mode（Cloudflare，中间结果不进上下文）· 客户端内置（官方/CCB/pi）· 硬上限（Cursor 40 / Copilot 128）。共识：词法匹配是默认起点，演进方向混合检索与 code mode。

---

## Web 能力（搜索与抓取）

> 核对日期：2026-11-12，官方 v2.1.88 sourcemap + CCB 分叉单列。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | WebSearch=客户端外壳，`call()` 内再发一次完整 LLM 请求，注入服务端工具 `web_search_20250305`，服务端在请求内跑搜索循环；WebFetch=axios 裸抓→turndown→Haiku 提炼，只回答案 | — | — | — |
| **值** | 内层 `tools:[]`、`max_uses:8`；WebFetch URL ≤2000、60s/10MiB/10 跳、MD 100k、缓存 15min·50MiB、预批准域名 89 条 | — | — | — |
| **位置** | 注册 `tools.ts:207–209` · WebSearch 嵌套 `WebSearchTool.ts:254–291` · extraToolSchemas 合并 `claude.ts:1385–1396` · Haiku 提炼 `WebFetchTool/utils.ts:484–529` | — | — | — |
| **注** | 两者 `shouldDefer:true`，默认模式要先 ToolSearch 一轮；WebSearch isEnabled 只看 provider 不认模型——第三方代理下工具启用但服务端能力不存在，**静默失败**；WebFetch 返回的是 Haiku 的答案，抓 JSON API 应改用 curl | — | — | — |

**三种模式**：A 混合式（服务端内联，如 `advisor`）· B 客户端确定性（Bash/Read/CCB tavily）· C 客户端外壳+内部 LLM（WebSearch/WebFetch）。C 内部跑的就是 A——用一次嵌套请求借来 A 的能力再降维。

**WebSearch 为何绕一圈**：`allowed/blocked_domains` 是 per-call 参数（混合式要求会话级固定）；校验与降维省 token。`page_age`/`encrypted_content`/citations 全丢，只留 title+url。

**WebFetch 四阶段**：校验 → 域名预检（`domain_info?can_fetch`）→ axios（手动跟随重定向）→ turndown + Haiku（`thinking:disabled`）。唯一跳过 Haiku：预批准域名 + 响应已是 markdown + <100k，极少命中。

**CCB 改造**：WebSearch isEnabled 恒 true，搜索默认 tavily HTTP（模式 B，不调模型，另有 bing/brave/exa/api）；WebFetch 默认 tavily `/extract` 直出 MD（跳过 turndown 和 Haiku）、删域名预检；`api` adapter 在第三方 provider 下失效（三条 dispatch 提前 return）。DeepSeek + WebSearch/WebFetch 在官方路径下均 ❌、CCB 下均 ✅。

**待验证**：⚠️ `pause_turn` 未实现（长搜索截断不续接）；⚠️ `citations_delta` 丢弃；⚠️ `tavily.claude-code-best.win` 是项目自建域名，query/URL 会经过第三方服务器。
