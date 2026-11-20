# 6. 技能与扩展 · 机制详情

> 对应 [../harness-matrix.md](../harness-matrix.md) 第 6 层信号矩阵。字段为行、harness 为列，同字段跨 harness 对齐。

---

## 渐进式披露

> 核对日期：2026-11-20。共同骨架（用户定稿的两步）：**① system-reminder 注入技能摘要清单（名+描述，不含正文）；② 提供按名加载工具，传入 name 即获取 SKILL 正文**。正文均由客户端展开注入，模型从不自己 Read SKILL.md。

| 字段 | ClaudeCode | DSH |
|---|---|---|
| **做法** | 两步同构：① `skill_listing` 附件渲染进 system-reminder（名字+截断描述，delta 增量去重）；② 模型调 `Skill(skill, args)`，客户端从内存缓存取 SKILL.md 全文 → 参数替换/目录头替换 → 作为 `isMeta` user 消息注入（tool_result 只回 "Launching skill"） | 两步同构：① 首请求前注入持久 `<available_skills>` 目录消息（名字+≤500 字符描述，digest 变化才全量替换）；② 模型调 `skill(name)`，每次重读文件，返回 `<skill_content>`（资源指引+正文）作为普通工具结果 |
| **值** | 描述截断按 token 预算动态均摊（bundled 保全文）；正文无大小限制记录 | `catalogDescriptionMaxLength=500`；正文无大小上限（提供方责任） |
| **位置** | 🟦 `claude-code-sourcemap/restored-src/src/tools/SkillTool/prompt.ts:189`（清单位置声明）· `claude-code-sourcemap/restored-src/src/utils/attachments.ts:2743-2750`（skill_listing）· `claude-code-sourcemap/restored-src/src/tools/SkillTool/SkillTool.ts:291-298`（inputSchema）· `claude-code-sourcemap/restored-src/src/skills/loadSkillsDir.ts:344-398`（客户端展开） | 🟦 `deepseek-harness/packages/skill/tool-skill/README.zh.md`（目录模板/工具结果模板/模型体验）· `deepseek-harness/packages/skill/skill/README.zh.md`（按需加载、定义不缓存）· 🟦实测 本会话即可观察（目录 + skill 工具） |
| **注** | ① 第 3 层两家一致：SKILL 正文引用的辅助文件由模型后续用 Read/Bash 自取（靠注入的 Base directory 头解析路径）。② 骨架下的实现分叉：目录维护 CC=delta 增量 / DSH=digest 全量替换+空目录停用旧名；调用入口 DSH 多一条 `/name` 用户手势直注入（免工具）；加载副作用 CC 有 `!`cmd`` 内联执行与 `context:fork` 派子 Agent，DSH 纯文本渲染 | ① 同左第 3 层。② DSH 的 `disable-model-invocation` 技能从目录和工具 schema 双侧隐藏，`/name` 是唯一入口；目录结尾防双载规则叮嘱模型「遵循注入块，勿再调工具」。③ 两家差异详见会话内对比，骨架层面的两步概括对两家均成立 |

---

## Inline/Fork 双模

> 核对日期：2026-11-20。共同背景：两家技能加载的缺省形态都是 inline（SKILL 展开正文注入主对话，主 Agent 亲自执行）；差异在是否支持技能**自声明**改走子 Agent。

| 字段 | ClaudeCode | DSH |
|---|---|---|
| **做法** | frontmatter `context: inline（缺省）/fork`；fork 时 SKILL 展开正文（+$ARGUMENTS 替换）包成一条 user 消息作为子 Agent 的全部初始上下文，`runAgent` 跑独立 query()，主对话只收最后一条 assistant 文本；frontmatter 的 `agent`（代理类型，缺省 general-purpose）、`allowed-tools`（子 Agent 临时白名单）、`model`、`effort` 随行生效 | **不支持技能自声明执行形态**——frontmatter 无 context 字段，加载即注入（`<skill_content>` 作为工具结果或 `/name` 直注入），执行地点由使用方决定：模型/用户可自行要求派子 Agent 处理，但这是编排侧行为，SKILL.md 无法声明 |
| **值** | 两条入口汇合：斜杠（`processSlashCommand.tsx:727-728` → `executeForkedSlashCommand`）与 Skill 工具（`SkillTool.ts:622-632` → `executeForkedSkill`）共用 `prepareForkedCommandContext`；子 Agent 看不到主对话聊天历史 | — |
| **位置** | 🟦 `claude-code-sourcemap/restored-src/src/skills/loadSkillsDir.ts:260`（context 解析）· `claude-code-sourcemap/restored-src/src/utils/forkedAgent.ts:191-232`（fork 公共准备）· `claude-code-sourcemap/restored-src/src/utils/processUserInput/processSlashCommand.tsx:229-241`（runAgent 调用） | 🟦 `deepseek-harness/packages/skill/skill/src/index.ts:56-100`（SkillDefinition 类型无执行形态概念）· `skill-filesystem/README.zh.md:36`（frontmatter 契约无 context）· `deepseek-harness/packages/skill/skill-filesystem/src/index.ts:1015-1019`（解析仅识别调用面布尔键） |
| **注** | ① inline 下 SKILL 全文留在主对话历史、工具输出占主上下文；fork 下主对话不被子 Agent 的中间工具噪音污染，代价是子 Agent 丢失对话上下文（SKILL 全文即其完整世界）。② KAIROS（assistant 模式）下 fork 例外为后台并行（`processSlashCommand.tsx:90-102`），常规用户调用仍同步 | ① DSH **有 fork 能力但不在 skill 层声明**——fork 是会话/Agent 层概念（`agents.create` 派子会话），由编排侧决定；准确表述是「无技能自声明执行形态」，不是「无 fork」。② `metadata` 自由字段可塞任意键但无消费者，写 `context: fork` 进去只是死数据 |

---

## 压缩后技能持久性（长会话）

> 核对日期：2026-11-20。口径：技能正文以普通消息进上下文后随轮次漂移、压缩时被摘要替换；考察各 harness 是否有机制保证「本会话用过的技能」在压缩后仍可用。结论分三档：确定性回注（CC）／仅目录重建（DSH）／无（PI、Codex）。

| 字段 | ClaudeCode | PI | DSH | Codex |
|---|---|---|---|---|
| **做法** | 有·开 · 加载即登记进 harness 状态注册表（不占上下文），压缩边界自动构造 `invoked_skills` 附件回注本 agent 用过的技能；`--resume` 时从 transcript 重建注册表 | 无 · 压缩管道（core/compaction/）无技能保留逻辑；技能正文本就靠模型用 Read 按需读文件，压缩后重读一次即恢复（自然恢复，非机制保证） | 半 · 正文不回注——README 明示正文随压缩失效；但目录持久消息被压缩隐藏后下一轮自动重建（replace 操作重发当前目录快照），「再触发」有机制保证 | 无 · compact.rs / skills.rs 均无技能内容回注逻辑；compact.rs 注释中的 "skill budgets" 指压缩策略不改变既有预算，非保留机制 |
| **值** | 每技能截断保留头部 5,000 token（注释：技能文件开头通常是最关键的指令），总预算 25,000（约容 5 个技能），按最近使用排序、超预算丢最久未用；fork 技能执行完即从注册表清除（finally 块），不回注 | — | — | — |
| **位置** | 🟦 `claude-code-sourcemap/restored-src/src/tools/SkillTool/SkillTool.ts:963,1088`（登记，注释明言 survives compaction）· `bootstrap/state.ts:1510-1565`（注册表增删查）· `services/compact/compact.ts:122-129`（预算常量）· `:1489-1544`（回注构造）· `:200-220`（摘要器剥离 skill_listing）· `:525-529`（目录不重发注释）· `utils/conversationRecovery.ts:391-399`（resume 重建） | 🟦 `pi/packages/coding-agent/src/core/compaction/compaction.ts`（全文件无 skill 引用） | 🟦 `deepseek-harness/packages/skill/tool-skill/README.md:222`（"resent … until compaction"）· `tool-skill/tests/tool-skill.spec.ts:688`（目录压缩后重建测试） | 🟦 `codex/codex-rs/core/src/compact.rs:66` · `core/src/skills.rs`（无压缩相关处理） |
| **注** | ① 回注是 harness 确定性动作，不依赖模型重调 Skill 工具——把「压缩后约束还在吗」从概率题变成确定题。② 历史正文无豁免，压缩时照常被摘要替换；复活靠注册表里的另一份副本，两份状态独立。③ 摘要器输入主动剥离 skill_listing/skill_discovery（省 token + 防陈旧建议污染摘要）；目录本身不在压缩边界重发，改由 resetSentSkillNames 清记录后下一轮懒重发。④ 回注内容随后继续随轮次漂移——CC 解「存续」，不解「注意力衰减」。⑤ 结论核于 v2.1.88 还原快照，CCB 复原源码同构 | ① PI 技能加载即模型 Read SKILL.md（官方文档明说），正文只是普通工具结果，无任何压缩豁免 | ① 目录重建的语义是「当前注册表快照」（名字+描述），不是用过的正文。② 与 CC 的架构分野：DSH 把「何时重载」留给模型每轮判断（概率性），CC 用注册表 + 边界回注把压缩失效变成机制保证 | ① plugins 路径（skill_snapshot_tests.rs）只测快照复用，未排查压缩路径；「无」结论基于 compact.rs + skills.rs 两文件，标待验证 |

---

## Frontmatter 字段支持（CC 15 字段清单对照）

> 核对日期：2026-11-20。来源：CC 官方文档 frontmatter 清单截图，逐项对照各仓库源码核验。结论先行：**CC 15/15 全支持，DSH 5/15，PI ≈8/15，Codex 3/15**——缺失的字段不是遗漏，是各架构「技能=纯指令文本」程度的投影。逐字段位置行号均指各仓库源码。

| 字段（CC 拼写） | Claude Code（2.1.88 快照） | DSH | PI | Codex |
|---|---|---|---|---|
| `name` | ✅ 必填（覆盖目录名） | ✅ 必填（kebab-case 强校验） | ✅（缺省回退父目录名，不强制与目录一致，[skills.md:83](pi/packages/coding-agent/docs/skills.md#L83)） | ✅ 必填（≤64 字符，缺省用目录名兜底，parser.rs:64-69） |
| `description` | ✅ 必填（缺省从正文首段提取） | ✅ 必填（缺失整个 skill 丢弃） | ✅ 必填（缺失则不加载，skills.ts:330） | ✅ 必填（缺失报 MissingField，parser.rs:83-85） |
| `when_to_use` | ✅ **下划线拼写确认**（`loadSkillsDir.ts:252`） | ⚠️ 同概念，拼写是 `whenToUse` | ❌（官方文档无此字段；CC 专属路由补充） | ❌（unknown 字段经 YAML 修复逻辑仅保标量，无消费者） |
| `argument-hint` | ✅ | ❌ | ✅（:163，用于 `/` 自动补全展示） | ⚠️ 仅 YAML 修复测试提及，非 skill 字段（属 prompt-template 体系） |
| `arguments` | ✅ 命名参数 → `$name` 逐个替换 | ❌（只有 `$ARGUMENTS` 整串替换） | ❌（PI 的参数走 `/skill:name args` 追加为 user 请求） | ❌ |
| `context` | ✅ `inline` / `fork` | ❌ | ❌ | ❌ |
| `allowed-tools` | ✅ | ❌ | ✅（官方文档「实验性预批工具清单」，skills.md:78） | ❌（Codex 的 skill-creator 校验脚本认识它，运行时 parser.rs 不消费） |
| `model` | ✅（`"inherit"`=缺省继承父级） | ❌ | ❌ | ❌ |
| `effort` | ✅ `quick`/`standard`/整数 | ❌ | ❌ | ❌（仅 external-agent-migration 的 agent frontmatter 有 effort，与 skill 无关） |
| `agent` | ✅ fork 时的代理类型 | ❌ | ❌ | ❌ |
| `shell` | ✅（`!`cmd`` 内联块的 shell） | ❌ | ❌ | ❌ |
| `paths` | ✅ gitignore 风格条件激活 | ❌ | ❌（claude-rules 扩展示例提 paths，是 rules 体系非 skill） | ❌ |
| `user-invocable` | ✅ | ✅ 同名同义（`:1019`） | ⚠️ 无此键；等价能力是 `disable-model-invocation` + `/skill:name` 命令面 | ❌ |
| `disable-model-invocation` | ✅ | ✅ 同名同义（`:1018`） | ✅ 同名同义（skills.ts:341；技能从系统提示词清单剔除，仅 `/skill:name` 可达） | ❌（memories 模板建议正文提及，运行时 parser.rs 不解析） |
| `hooks` | ✅（技能级 Hook，随技能加载注册为 session hook） | ❌（等价诉求走插件事件机制） | ❌ | ❌ |
| 截图未列 | `version` ✅（`:253`） | `metadata` ✅ 不透明自由对象 | ⚠️ `license`/`compatibility`/`metadata` 按 Agent Skills 规范文档收录（skills.md:75-77），运行时只强校验 name/description | ⚠️ `metadata.short-description` ✅（parser.rs:16-20；UI 元数据另走 `agents/openai.yaml`） |

> **做法**：CC 三组字段（执行语义+路由元数据+身份调用面）全解析生效；DSH/PI/Codex 均只消费身份+调用面附近字段——三家都是「加载即注入」架构，无执行语义可配置；PI 多出 argument-hint（路由展示）与 allowed-tools（实验性），Codex 只认 name/description/metadata.short-description 三键。
>
> **位置**：🟦 `claude-code-sourcemap/restored-src/src/skills/loadSkillsDir.ts:190-264` · 🟦 `deepseek-harness/packages/skill/skill-filesystem/src/index.ts:1015-1019` · 🟦 `pi/packages/coding-agent/src/core/skills.ts:280-345`（解析）+ [docs/skills.md:67-85](pi/packages/coding-agent/docs/skills.md#L67-L85)（字段契约）· 🟦 `codex/codex-rs/skills/src/parser.rs:1-92`（SkillFrontmatter 结构与校验）
>
> **注**：① `when_to_use` 下划线是 CC 方言，DSH 用 `whenToUse`，PI/Codex 无此键——跨家写 SKILL.md 只有 name/description 可移植。② PI 与 Codex 的渐进式披露形态与 DSH 同构（清单进 system prompt + 按需读正文），但 PI 的加载动作是模型用 read 工具读 SKILL.md（官方文档明说），不是专用工具。③ Codex 的 `metadata.short-description` 是 UI 短描述（顶层 UI 元数据另走 `agents/openai.yaml`），不是路由字段。④ DSH 调用面布尔键语法严格（非法值丢弃整个技能）；PI 契约相反「多数字段警告而非阻断」（skills.md:7）。⑤ `allowed-tools` 只有 CC 运行时真正消费（fork 白名单）；PI 文档收录为实验性、Codex 仅校验脚本认识。

> PI、Codex：渐进式披露与 frontmatter 字段已核（PI 官方源码，Codex 官方源码）；本文件其余机制的 PI/Codex 列仍未研究。
