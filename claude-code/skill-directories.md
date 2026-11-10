# Claude Code 读取 skill 的全部目录

> **定位**：skill 从磁盘哪些位置被扫描加载的完整清单，含优先级、去重规则和本机实测路径。
>
> **依据**：claude-code-best v2.8.4 源码（`src/skills/loadSkillsDir.ts`、`src/skills/bundledSkills.ts`、`src/utils/plugins/loadPluginCommands.ts`、`src/commands.ts`），关键行为均在 2026-11-10 实测验证。
>
> **入口**：`getSkillDirCommands(cwd)`（loadSkillsDir.ts:638，memoized）负责磁盘扫描；`getSkills(cwd)`（commands.ts:444）并发聚合；`getBundledSkills()`（bundledSkills.ts:106）同步注册内置。

## 一、六类来源总表

| # | 来源 | 路径 | source 标记 | 开关/条件 |
|---|---|---|---|---|
| 1 | 用户级 | `~/.claude/skills/`（每 skill 一个子目录，含 `SKILL.md`） | `userSettings` | `CLAUDE_CONFIG_DIR` 可改根；settings 里禁用 userSettings 则跳过 |
| 2 | 项目级 | 从 **cwd 向上走到 git root**（不在 git 仓库则走到 HOME），每级的 `.claude/skills/` | `projectSettings` | `getProjectDirsUpToHome('skills', cwd)`；HOME 本身不算（它已是用户级）；projectSettings 被策略禁用则跳过 |
| 3 | 托管/策略级 | macOS: `/Library/Application Support/ClaudeCode/.claude/skills/`<br>Windows: `C:\Program Files\ClaudeCode\.claude\skills\`<br>Linux: `/etc/claude-code/.claude/skills/` | `policySettings` | 企业管控下发；`CLAUDE_CODE_DISABLE_POLICY_SKILLS=1` 可关闭 |
| 4 | 捆绑内置 | 代码内注册（`src/skills/bundled/*.ts` 调 `registerBundledSkill`），**不扫磁盘** | `bundled` | 永远加载；本仓库现有 19 个（remember/verify/loop/debug/dream/batch/update-config/keybindings/claude-api/skillify/use-artifacts/simplify/ultracode/stuck/schedule-remote-agents/cron 系列/lorem-ipsum） |
| 5 | 插件 | `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/skills/`（`installed_plugins.json` 记录 installPath） | `plugin` | 仅**已启用**插件；命名 `插件名:skill名`（如 `glm-plan-usage:usage-query-skill`） |
| 6 | 兼容/扩展 | legacy `commands/` 目录（旧版命令即 skill）＋ `--add-dir` 指定目录下的 `.claude/skills/` | `commands_DEPRECATED` / `projectSettings` | `--bare` 模式下**只**加载 `--add-dir`，跳过 1/2/3/6 的自动发现 |

## 二、本机实测（gatesma@macOS，2026-11-10）

```
用户级     ~/.claude/skills/                    → 空（无自装 skill）
项目级     ~/project/claude/claude-code-best/.claude/skills/
           ├── interview/
           └── teach-me/
插件       ~/.claude/plugins/cache/zai-coding-plugins/
           ├── glm-plan-usage/0.0.1/skills/usage-query-skill/
           └── glm-plan-bug/0.0.1/skills/case-feedback-skill/
捆绑内置   （代码注册 19 个，如 update-config —— 即调试时 skill_listing 里看到的那批）
托管级     /Library/Application Support/ClaudeCode/.claude/skills/ → 未使用
MCP        经 getMcpSkillCommands 从 AppState.mcp.commands 并入（不在 getCommands 主链，见下）
```

## 三、加载与合并规则

**扫描细节**（loadSkillsDir.ts）：

- 每个 skill = 一个子目录 + `SKILL.md`（YAML frontmatter：name/description/when-to-use/allowed-tools 等）；
- 六类来源在 `getSkills` 里 `Promise.all` 并发扫描，单类失败降级为空数组不拖垮整体（commands.ts:451-464）；
- **去重按文件真实身份**（realpath，防 symlink 重复），顺序先行先得；顺序 = bundled → builtinPlugin → skillDir → workflow → pluginCommands → pluginSkills（loadAllCommands，commands.ts:551）；
- 项目级向上走的**停止边界是 git root**——仓库外的父目录 skill 不会漏进来（markdownConfigLoader.ts:234-268）。

**与命令体系的关系**：skill 就是 `type: 'prompt'` 的 Command。全链：

```
getSkillDirCommands（扫盘） → getSkills → loadAllCommands(memoize by cwd) → getCommands（新鲜过滤可用性）
  → getSkillToolCommands（prompt 子集，skill 工具可见范围）
    → getSkillIndex（TF-IDF 索引，供 skill_discovery 相似度匹配）
getMcpSkillCommands：MCP skill 不走 getCommands，需调用方自行并入
```

## 四、常见坑

1. **`CLAUDE_CONFIG_DIR` 改根**：用户级路径跟着变，且 `getClaudeConfigHomeDir` 是 memoize——测试/进程内改环境变量后拿的是新值，但旧缓存不失效的场景要留意；
2. **monorepo 祖先目录**：只要在 git 仓库内，**任意祖先** `.claude/skills/` 都会被扫到——找不到某 skill 来源时往上层目录找；
3. **EXPERIMENTAL_SKILL_SEARCH 改变 listing 内容**：开启后 `getSkillListingAttachments` 把清单收敛为 **bundled + MCP only**（attachments.ts:2794-2800），用户/项目/插件 skill 不再出现在全量清单里，改由 `skill_discovery` 按意图匹配出现——这不是丢了，是设计；
4. **插件 skill 的双注册**：一个插件可同时注册 commands 和 skills（如 glm-plan-bug 的 `case-feedback` 与 `case-feedback-skill`），名字带 `插件:` 前缀；
5. **缓存三层**：`getSkillDirCommands` → `loadAllCommands` → `getSkillIndex` 各自 memoize，改 skill 文件后不生效时按此顺序排查（运行期 `skillChangeDetector` 监听变更会自动清）。

## 五、相关笔记

- WebStorm 调试方法：[webstorm-debug.md](webstorm-debug.md)
- 调试注入链路时的观察点：`loadSkillsDir.ts:644` 的 `Loading skills from: …` 日志直接打印 managed/user/project 三个路径（`/debug` + `tail -F ~/.claude/debug/latest` 可见）。
