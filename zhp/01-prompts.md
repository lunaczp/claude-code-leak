# Claude Code 提示词体系

| | |
|---|---|
| 状态 | 学习中 |
| 日期 | 2026-11-27 |
| 基线 | 本仓 `zhp` 分支；对照 commit `b983a3b`（从 `main` 分出） |
| 证据 | 源码；对照 `00-mental-model.md` |

> 本仓为 **leaked source**；本文只做机制学习。  
> 证据约定：**源码** = 读仓库确认；**二手** = 仓内 docs；**推断** = 尚未验证。  
> **不含** 根目录 `prompts/`（那是社区重建工程用的学习 prompt，不是产品运行时提示词）。

## 一句话

**主对话提示词 = 可缓存静态行为规范 + registry 动态段（memory／env／style／MCP…）+ userContext（CLAUDE.md 等）+ systemContext（git 等）+ 工具 description；旁路任务各自独立小 prompt。**（源码）

---

## 主路径

```
getSystemPrompt (constants/prompts.ts)
    ↓
buildEffectiveSystemPrompt (utils/systemPrompt.ts)   // REPL；可被 agent／CLI 覆盖
  或 QueryEngine 自行拼装                            // headless／SDK
    ↓
fetchSystemPromptParts (utils/queryContext.ts)
  → defaultSystemPrompt + getUserContext + getSystemContext (context.ts)
    ↓
query() (query.ts)
  ├─ appendSystemContext(systemPrompt, systemContext)  → API system
  └─ prependUserContext(messages, userContext)         → 首条 meta user（<system-reminder>）
    ↓
工具 schema.description（各 tools/*/prompt.ts）一并发送
```

锚点：

- [`src/constants/prompts.ts`](../src/constants/prompts.ts) — `getSystemPrompt`、`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`
- [`src/utils/systemPrompt.ts`](../src/utils/systemPrompt.ts) — `buildEffectiveSystemPrompt` 优先级
- [`src/utils/api.ts`](../src/utils/api.ts) — `appendSystemContext`／`prependUserContext`
- [`src/query.ts`](../src/query.ts) — 真正接到 API 调用
- [`src/QueryEngine.ts`](../src/QueryEngine.ts)／[`src/screens/REPL.tsx`](../src/screens/REPL.tsx) — 两套入口拼装

注意：`src/context/` 是 React UI Context，**不是** prompt 上下文；动态上下文在 `src/context.ts`。

---

## 分层

| 层 | 内容 | 注入 |
|---|---|---|
| L1 主 system | 身份／行为／工具使用规范 | `string[]` → API `system` |
| L2 覆盖／追加 | agent、`--system-prompt`、append、coordinator、override | `buildEffectiveSystemPrompt` |
| L3 system context | git status、cacheBreaker 等 | `appendSystemContext` |
| L4 user context | CLAUDE.md 树、currentDate 等 | `prependUserContext` |
| L5 工具 description | 各 tool 的 `prompt.ts` | tool schema |
| L6 旁路 LLM | compact／memory／suggestion 等 | 各自独立调用 |

静态／动态分界：`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`（`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`）。boundary 前可走 global／org cache；后段为会话动态（见 `prompts.ts` 注释与 `api.ts` 的 `splitSysPromptPrefix`）。

---

## `buildEffectiveSystemPrompt` 优先级

（源码注释，`utils/systemPrompt.ts`）

0. `overrideSystemPrompt` — 整段替换（如 loop mode）  
1. Coordinator（`CLAUDE_CODE_COORDINATOR_MODE`）  
2. Agent system prompt — proactive 时**追加**到 default；否则**替换**  
3. `customSystemPrompt`（`--system-prompt`）  
4. `defaultSystemPrompt`（`getSystemPrompt`）  
+ 除 override 外，可再附 `appendSystemPrompt`

---

## 主 system prompt 段落

### 静态段（boundary 前）

来自 `getSystemPrompt` 返回数组前半：

1. `getSimpleIntroSection` — 身份 + cyber risk + URL 规则  
2. `getSimpleSystemSection` — `# System`  
3. `getSimpleDoingTasksSection` — `# Doing tasks`（可被 output style 关掉）  
4. `getActionsSection` — `# Executing actions with care`  
5. `getUsingYourToolsSection` — `# Using your tools`  
6. `getSimpleToneAndStyleSection` — `# Tone and style`  
7. `getOutputEfficiencySection` — 输出效率／对用户沟通  
8. （可选）`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`

### 动态 registry

`systemPromptSection`／`DANGEROUS_uncachedSystemPromptSection`（`/clear`、`/compact` 清缓存）：

| name | 来源 |
|---|---|
| `session_guidance` | 会话级工具／skill 指引 |
| `memory` | `loadMemoryPrompt()`（`memdir/`） |
| `ant_model_override` | ant 模型后缀 |
| `env_info_simple` | `# Environment` |
| `language` | settings.language |
| `output_style` | `# Output Style: …` |
| `mcp_instructions` | **uncached**：MCP server instructions（或改走 delta attachment） |
| `scratchpad` | scratchpad 目录指引 |
| `frc` | Function Result Clearing |
| `summarize_tool_results` | 工具结果摘要提醒 |
| `numeric_length_anchors` | ant-only 字数锚点 |
| `token_budget` | feature `TOKEN_BUDGET` |
| `brief` | feature KAIROS／BRIEF |

### 特殊短路

- `CLAUDE_CODE_SIMPLE`：一行身份 + CWD + Date  
- PROACTIVE／KAIROS 激活：另一套简短 autonomous prompt（含 memory／env／proactive）

---

## 工具 description（36 个 `prompt.ts`）

路径：`src/tools/<Name>/prompt.ts`。

Agent、AskUserQuestion、Bash、Brief、Config、Enter／Exit Plan／Worktree、FileEdit／Read／Write、Glob、Grep、ListMcpResources、LSP、MCP（description 常由服务端填）、NotebookEdit、PowerShell、ReadMcpResource、RemoteTrigger、ScheduleCron、SendMessage、Skill、Sleep、TaskCreate／Get／List／Stop／Update、TeamCreate／Delete、TodoWrite、ToolSearch、WebFetch、WebSearch。

另：`REPLTool`／`TaskOutputTool` 等可能无独立 `prompt.ts`；部分工具在 feature 门控下条件加载。

---

## 旁路／子 Agent prompts

### 旁路二次 LLM

| 模块 | 路径 |
|---|---|
| Compact 摘要 | `services/compact/prompt.ts` |
| SessionMemory | `services/SessionMemory/prompts.ts` |
| 抽记忆 | `services/extractMemories/prompts.ts` |
| MagicDocs | `services/MagicDocs/prompts.ts` |
| PromptSuggestion | `services/PromptSuggestion/promptSuggestion.ts` |
| 工具短标签 | `services/toolUseSummary/toolUseSummaryGenerator.ts` |
| autoDream | `services/autoDream/consolidationPrompt.ts` |
| 会话搜索 | `utils/agenticSessionSearch.ts` |
| 选相关 memory | `memdir/findRelevantMemories.ts` |
| Prompt／Agent hooks | `utils/hooks/execPromptHook.ts`、`execAgentHook.ts` |
| shell prefix／权限解释 | `utils/shell/prefix.ts`、`permissions/permissionExplainer.ts` |

### 内置子 Agent system prompts

`tools/AgentTool/built-in/`：`generalPurposeAgent`、`exploreAgent`、`planAgent`、`verificationAgent`、`claudeCodeGuideAgent`、`statuslineSetup`。

另：`DEFAULT_AGENT_PROMPT`、`enhanceSystemPromptWithEnvDetails`（`constants/prompts.ts`）；teammate addendum（`utils/swarm/teammatePromptAddendum.ts`）；coordinator 整段（`coordinator/coordinatorMode.ts`）。

---

## 用户可编辑／可注入

| 类型 | 机制 | 进模型方式 |
|---|---|---|
| CLAUDE.md 层级 | Managed／`~/.claude`／项目／local／rules | `getUserContext().claudeMd` → `prependUserContext` |
| Auto／Team memory | `MEMORY.md` + topic files | system section `memory` |
| Skills | `.claude/skills/**/SKILL.md` 等 | Skill 工具列表；调用时 body 展开 |
| Agents | `.claude/agents/*.md`、plugins | `getSystemPrompt()` 替换／追加 |
| Output styles | 内置 + 用户／插件 | section `output_style` |
| CLI flags | `--system-prompt`／`--append-system-prompt` | custom／append |
| MCP prompts | `prompts/list` + `get` | slash → 消息块 |
| MCP instructions | `client.instructions` | section 或 delta attachment |
| Hooks `type: prompt`／`agent` | settings | 独立 LLM 判条件 |
| Magic Docs 模板 | `~/.claude/magic-docs/prompt.md` | 覆盖旁路模板 |

加载 CLAUDE.md：[`src/utils/claudemd.ts`](../src/utils/claudemd.ts)。Memory：[`src/memdir/`](../src/memdir/)。

---

## 和心智模型的关系

`00-mental-model` 的主循环是 `QueryEngine` → `query()` → tools。  
提示词挂在**进循环之前**：拼好 system／context，再进 `while`；循环内旁路（compact、抽记忆、suggestion）另起调用，不替换主 system 前缀（fork／缓存策略另议）。

---

## 还没搞懂

1. `splitSysPromptPrefix`／global cache 与 boundary、MCP delta 的精确拆分规则。  
2. 各旁路调用是否刻意共享主 prompt 前缀（cache 友好）还是完全独立。  
3. Skill 展开为 user 消息时的包装格式与权限边界。  
4. Coordinator／teammate／fork subagent 三套 system 的互斥与叠加。  
5. 下一刀待定：深挖 `getSystemPrompt` 静态段原文，或 `prependUserContext` + CLAUDE.md 合并顺序。
