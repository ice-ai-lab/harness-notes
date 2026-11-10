# WebStorm 调试 Claude Code（Bun launch 方案）

> **定位**：在 WebStorm 里断点调试本仓库 CLI（`query.ts` 循环、skill 注入等）的可复制的 Debug 配置，以及配置过程中踩过的四个坑的根因与修法。
>
> **适用**：claude-code-best v2.8.4 · WebStorm 2025.3（Bun 插件内置）· Bun 1.4.4
>
> **结论先行**：Bun 类型 Run Configuration **直接指向 `src/entrypoints/cli.tsx`**，管道模式 `-p` 触发，feature 用 `--feature` 传，defines 走 `bunfig.toml` 的 `[define]`。

---

## 1. 最终可用的配置

**Run | Edit Configurations → + → Bun**，不要用 npm 类型（原因见 §2 坑 ①）：

| 字段 | 值 | 说明 |
|---|---|---|
| **File** | `src/entrypoints/cli.tsx` | 直接指 CLI 入口，**不要指 `scripts/dev.ts`**（坑 ①） |
| **Application parameters** | `-p "你好"` | 管道模式：跑一轮完整请求后退出，断点链路完整、可反复复现；想交互式 TUI 见 §4 |
| **Bun parameters** | `--feature EXPERIMENTAL_SKILL_SEARCH --feature EXPERIMENTAL_SEARCH_EXTRA_TOOLS` | 研究哪条链路就开对应 feature；全量清单的再生成见 §5 |
| **Environment variables** | （管道模式留空） | TUI 模式才需要 `CLAUDE_CODE_FORCE_INTERACTIVE=1`（坑 ③） |
| **Working directory** | 项目根 | 默认即可 |

defines **一个都不用传**：`bunfig.toml` 已补 `[define]` 块（7 个 MACRO），对静态和动态 import 都生效（`scripts/dev.ts` 头注释"bunfig 不传导"已过时，实测 2026-11 Bun 1.4.4 传导）。

> **⚠️ 别把 `process.env.NODE_ENV` 放进 bunfig [define]**——它是转译期文本替换，`bun test` 下也会把源码里的 `process.env.NODE_ENV` 硬替换成 `"production"`，击穿 `NODE_ENV !== 'test'` 守卫（实测打穿 `bootstrap/state.ts` 的 `resetStateForTests`，2 个集成测试挂掉）。需要 NODE_ENV 时用运行时环境变量（WebStorm Environment variables 填 `NODE_ENV=production`），首次填过 `CLAUDE_CODE_FORCE_INTERACTIVE=1` 的同样走这里。

## 2. 四个坑（每个都实测复现过）

### 坑 ①：调试器在父进程，代码跑在子进程 → 断点永不命中

npm 配置执行 `bun run dev` → `scripts/dev.ts` `spawnSync` 又拉子进程跑 CLI。`ps -p <pid> -o ppid,command` 可验证：WebStorm 附着的是父进程（它只执行"拼参数 + spawn"，永远不经过 `query.ts`），`query.ts` 的断点打了个寂寞。

**修法**：Bun 配置 File 直接指 `cli.tsx`，被调试进程 = 跑代码的进程。

### 坑 ②：IDE 参数框剥双引号 → `-d` 的 JSON 值解析失败

`bun -d` 的值按 JSON 解析（`--help` 原话 "Values are parsed as JSON"），`MACRO.VERSION` 必须收到带引号的 `"2.8.4"`。终端里引号由 shell 传递没问题；WebStorm parameters 框自己做 shell 式分词，**把双引号剥掉**，Bun 收到裸 `2.8.4` 报：

```
error: Unexpected 2.8.4
    at defines.json:1:1
```

`defines.json` 是 Bun 给 define 解析失败临时起的虚拟文件名，仓库里并不存在——看到它就条件反射想到"参数里的引号被 IDE 吃了"。

**修法**：defines 全部挪进 `bunfig.toml [define]`（toml 里引号天经地义），IDE 参数零引号需求。注意 `-d` CLI 参数与 `Bun.build({ define })` 优先级更高，`bun run dev` / build 流程不受 bunfig 影响。

### 坑 ③：Process Console 不是 TTY → 自动进入管道模式又没输入

WebStorm 的 Process Console 是管道，`process.stdout.isTTY === false`，CLI 在 `src/main.tsx:931`（`!forceInteractive && !process.stdout.isTTY`）自动按非交互处理；若此时没传 prompt 就报 `Error: Input must be provided either through stdin or as a prompt argument when using --print`（`src/cli/print.ts:794`）退出。

**修法（二选一）**：
- **管道模式（推荐）**：Application parameters 给 `-p "你的提问"`，单轮跑完即退，复现最稳；
- **TUI 模式**：Environment variables 加 `CLAUDE_CODE_FORCE_INTERACTIVE=1`（`src/entrypoints/cli.tsx:41-55` 的 dev 垫片，把三个流的 isTTY 硬改 true）。两种互斥，别同时上。

### 坑 ④：`VOICE_MODE` 关闭时 TUI 渲染崩溃（已修复的仓库 bug）

只开少量 feature 时 TUI 崩：`undefined is not an object (evaluating 'VoiceContext.Provider')`，Boundary: RootREPLBoundary。

根因：`src/state/AppState.tsx:17-31` 的 noop 分支（`VOICE_MODE` 关时兜底）依赖 `require('../context/voice.js').VoiceContext`，但 `voice.tsx` 从未导出 `VoiceContext`——实测任何开关下该导出都是 `undefined`。管道模式（`-p`）不渲染 TUI 所以不触发；TUI 模式必崩。

**修法**：`src/context/voice.tsx` 的 `const VoiceContext` 已加 `export`（2026-11-10）。

## 3. 断点工作流

- **断点**：编辑器行号槽点击。`query.ts` 在首轮请求时才加载，启动时断点空心是正常的，发一轮消息后变实心。
- **页签**：Process Console（TUI/输出在这）· Threads & Variables（暂停时看栈帧和变量）· Debugger Console（暂停帧上下文里求值）· Scripts（模块树，基本用不上）。
- **常用定位**：`F8` 步过 / `F7` 步入 / `Shift+F8` 步出 / `F9` 放行。
- **条件断点**：右键断点 → Condition，如 `result.score >= 0.30`（skill 自动载入线）。
- **日志断点（最好用）**：右键断点 → 取消 Suspend、勾 Evaluate and log，填如 `` `[skill] ${result.name} score=${result.score.toFixed(3)}` ``——进程不停，输出进 Debugger Console，等于不改代码的 `logForDebugging`。
- **注意**：断点暂停时在途的 API 流式请求还在等，看变量别磨蹭；改代码后重启 Debug 会话（Bun 无热重载）。

## 4. skill 注入观察点（配合本方案）

| 通道 | 断点/日志位置 | 观察什么 |
|---|---|---|
| 全量清单 `skill_listing` | `src/utils/attachments.ts:2814` | `Sending N skills… (initial/dynamic)` |
| 相似度匹配 `skill_discovery` | `src/services/skillSearch/localSearch.ts:428`（0.10 展示线）、`src/services/skillSearch/prefetch.ts:141`（0.30 载入线） | 逐个 score 与两道门槛的比较 |
| 显式调用 `dynamic_skill` | `src/utils/attachments.ts:2658` | SKILL.md 全文展开 |

环境变量可改门槛做对照实验：`SKILL_SEARCH_DISPLAY_MIN_SCORE` / `SKILL_SEARCH_AUTOLOAD_MIN_SCORE`。

## 5. feature 参数再生成

`defines.ts` 的 feature 列表变化后，重出全量参数：

```bash
bun -e 'const {DEFAULT_BUILD_FEATURES}=await import("./scripts/defines.ts");console.log(DEFAULT_BUILD_FEATURES.map(f=>`--feature ${f}`).join(" "))'
```

只粘 `--feature` 部分；`-d` 参数永远不需要（见坑 ②）。漏开 feature 时 `feature()` 返回 false、对应代码被 DCE 裁掉——表现为"断点空心不命中"或行为缺失，先查 feature 再怀疑别的。

## 6. 备选方案

| 方案 | 适用 | 说明 |
|---|---|---|
| **本方案（WebStorm launch）** | WebStorm 日常调试 | 见上 |
| `bun run dev:inspect` + 浏览器 | 零配置、真终端 TUI | 开 banner 里的 `debug.bun.sh` 链接；WebKit Inspector，Sources 树大时卡 |
| `dev:inspect` + VS Code attach | VS Code 用户 | `.vscode/launch.json` 已配（`type: bun, request: attach`），扩展 `oven.bun-vscode` |
| 无断点日志 | 只看链路是否在跑 | `/debug` 激活 → `tail -F ~/.claude/debug/latest`；代码里用 `logForDebugging`（勿用 `console.log`，会打花 TUI） |
| ~~WebStorm Attach to Process~~ | 不可行 | JetBrains attach 走 V8 协议，Bun 说 WebKit 协议，接不上 |

## 7. 相关文件

| 文件 | 说明 |
|---|---|
| `bunfig.toml` | 本次新增 `[define]` 块（仅 MACRO，故意不含 NODE_ENV，原因见 §1 警告框） |
| `src/context/voice.tsx` | 本次导出 `VoiceContext`（修坑 ④） |
| `scripts/dev.ts` | 终端开发入口（feature/defines 注入的参考实现） |
| `src/entrypoints/cli.tsx:41` | `CLAUDE_CODE_FORCE_INTERACTIVE` 垫片 |
| `../debug/` 系列 | `logForDebugging` 日志链路笔记 |

## 8. 本次改动后的验证状态（2026-11-10）

- `bun src/entrypoints/cli.tsx --version` → `2.8.4 (Claude Code)`（bunfig defines 生效）
- `bun test ./tests/integration/` → 68/68 通过（撤掉 NODE_ENV define 后恢复）
- `bun test ./src/services/skillSearch/` → 43/43 · `./src/services/searchExtraTools/` → 12/12
- 预存问题（与本次无关，stash 验证过）：`bun run typecheck` 报 `scripts/dev.ts:9`、`build.ts:5` 两处 unused `@ts-expect-error`（TS2578）
