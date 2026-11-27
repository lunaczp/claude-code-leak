# Claude Code 提示词体系

| | |
|---|---|
| 状态 | 学习中 |
| 日期 | 2026-11-27 |
| 基线 | 本仓 `zhp` 分支；对照 commit `b983a3b`（从 `main` 分出） |
| 证据 | 源码；对照 [00-mental-model.md](./00-mental-model.md) |

> 本仓为 **leaked source**；本文只做机制学习。  
> 证据约定：**源码** = 读仓库确认；**二手** = 仓内 docs；**推断** = 尚未验证。  
> **不含** 根目录 [`prompts/`](../prompts/00-overview.md)（社区重建工程用的学习 prompt，不是产品运行时提示词）。  
> 文中链接只用相对文件路径（`../src/...`），**不写 `#L`**（Preview 会当成页内锚点打不开）；行号写在链接文字里（如 `· L444`）。

## 一句话

**主对话提示词 = 可缓存静态行为规范 + registry 动态段（memory／env／style／MCP…）+ userContext（CLAUDE.md 等）+ systemContext（git 等）+ 工具 description；旁路任务各自独立小 prompt。**（源码）

---

## 主路径

```
getSystemPrompt                          // constants/prompts.ts#L444
    ↓
buildEffectiveSystemPrompt               // utils/systemPrompt.ts#L41  （REPL）
  或 QueryEngine 自行拼装                // QueryEngine.ts#L292–L324   （headless／SDK）
    ↓
fetchSystemPromptParts                   // utils/queryContext.ts#L44
  → getUserContext / getSystemContext    // context.ts#L155 / #L116
    ↓
query()                                  // query.ts
  ├─ appendSystemContext                 // api.ts#L437 → #L450
  └─ prependUserContext                  // api.ts#L449 → #L660
    ↓
工具 schema.description                  // tools/*/prompt.ts
```

锚点：

- [`getSystemPrompt` · L444](../src/constants/prompts.ts)／[`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` · L114](../src/constants/prompts.ts) — [`src/constants/prompts.ts`](../src/constants/prompts.ts)
- [`buildEffectiveSystemPrompt` · L41](../src/utils/systemPrompt.ts) — [`src/utils/systemPrompt.ts`](../src/utils/systemPrompt.ts)
- [`appendSystemContext` · L437](../src/utils/api.ts)／[`prependUserContext` · L449](../src/utils/api.ts)／[`splitSysPromptPrefix` · L321](../src/utils/api.ts) — [`src/utils/api.ts`](../src/utils/api.ts)
- 调用点：[`query.ts` · L450](../src/query.ts)（append）、[`query.ts` · L660](../src/query.ts)（prepend）
- [`fetchSystemPromptParts` · L44](../src/utils/queryContext.ts) — [`src/utils/queryContext.ts`](../src/utils/queryContext.ts)
- [`QueryEngine` 拼装 · L292](../src/QueryEngine.ts) — [`src/QueryEngine.ts`](../src/QueryEngine.ts)
- REPL：[`getSystemPrompt` + `buildEffectiveSystemPrompt` · L2535](../src/screens/REPL.tsx) — [`src/screens/REPL.tsx`](../src/screens/REPL.tsx)

注意：[`src/context/`](../src/context/QueuedMessageContext.tsx) 是 React UI Context，**不是** prompt 上下文；动态上下文在 [`src/context.ts`](../src/context.ts)（[`getSystemContext` · L116](../src/context.ts)、[`getUserContext` · L155](../src/context.ts)）。

---

## 分层

| 层 | 内容 | 注入 |
|---|---|---|
| L1 主 system | 身份／行为／工具使用规范 | `string[]` → API `system` |
| L2 覆盖／追加 | agent、`--system-prompt`、append、coordinator、override | [`buildEffectiveSystemPrompt` · L41](../src/utils/systemPrompt.ts) |
| L3 system context | git status、cacheBreaker 等 | [`appendSystemContext` · L437](../src/utils/api.ts) |
| L4 user context | CLAUDE.md 树、currentDate 等 | [`prependUserContext` · L449](../src/utils/api.ts) |
| L5 工具 description | 各 tool 的 `prompt.ts` | tool schema |
| L6 旁路 LLM | compact／memory／suggestion 等 | 各自独立调用 |

静态／动态分界：[`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` · L114](../src/constants/prompts.ts)（`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`）。boundary 前可走 global／org cache；后段为会话动态（见 [`prompts.ts` 注释 · L105](../src/constants/prompts.ts)、[`splitSysPromptPrefix` · L321](../src/utils/api.ts)）。

Registry 机制：[`systemPromptSection` · L20](../src/constants/systemPromptSections.ts)／[`DANGEROUS_uncachedSystemPromptSection` · L32](../src/constants/systemPromptSections.ts)／[`resolveSystemPromptSections` · L43](../src/constants/systemPromptSections.ts) — [`src/constants/systemPromptSections.ts`](../src/constants/systemPromptSections.ts)；[`clearSystemPromptSections` · L65](../src/constants/systemPromptSections.ts) 在 `/clear`、`/compact` 时清缓存。

---

## `buildEffectiveSystemPrompt` 优先级

（源码注释，[`utils/systemPrompt.ts · L28–L40`](../src/utils/systemPrompt.ts)）

0. `overrideSystemPrompt` — 整段替换（如 loop mode）— [`#L56`](../src/utils/systemPrompt.ts)  
1. Coordinator（`CLAUDE_CODE_COORDINATOR_MODE`）— [`#L62`](../src/utils/systemPrompt.ts) → [`getCoordinatorSystemPrompt` · L111](../src/coordinator/coordinatorMode.ts)  
2. Agent system prompt — proactive 时**追加**到 default（[`#L103`](../src/utils/systemPrompt.ts)）；否则**替换**（[`#L115`](../src/utils/systemPrompt.ts)）  
3. `customSystemPrompt`（`--system-prompt`）  
4. `defaultSystemPrompt`（[`getSystemPrompt` · L444](../src/constants/prompts.ts)）  
+ 除 override 外，可再附 `appendSystemPrompt`

---

## 主 system prompt 段落

### 静态段（boundary 前）

来自 [`getSystemPrompt` · L444](../src/constants/prompts.ts) 返回数组前半（[`#L560–L576`](../src/constants/prompts.ts)）：

1. [`getSimpleIntroSection` · L175](../src/constants/prompts.ts) — 身份 + cyber risk + URL 规则  
2. [`getSimpleSystemSection` · L186](../src/constants/prompts.ts) — `# System`  
3. [`getSimpleDoingTasksSection` · L199](../src/constants/prompts.ts) — `# Doing tasks`（可被 output style 关掉）  
4. [`getActionsSection` · L255](../src/constants/prompts.ts) — `# Executing actions with care`  
5. [`getUsingYourToolsSection` · L269](../src/constants/prompts.ts) — `# Using your tools`  
6. [`getSimpleToneAndStyleSection` · L430](../src/constants/prompts.ts) — `# Tone and style`  
7. [`getOutputEfficiencySection` · L403](../src/constants/prompts.ts) — 输出效率／对用户沟通  
8. （可选）[`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` · L114](../src/constants/prompts.ts) — 插入于 [`#L573`](../src/constants/prompts.ts)

### 动态 registry

注册于 [`#L491–L555`](../src/constants/prompts.ts)；`/clear`、`/compact` 清缓存：

| name | 来源 |
|---|---|
| `session_guidance` | [`#L492`](../src/constants/prompts.ts) |
| `memory` | [`loadMemoryPrompt` · L419](../src/memdir/memdir.ts)（[`memdir/`](../src/memdir/memdir.ts)） |
| `ant_model_override` | [`#L496`](../src/constants/prompts.ts) |
| `env_info_simple` | [`#L499`](../src/constants/prompts.ts) |
| `language` | [`#L502`](../src/constants/prompts.ts) |
| `output_style` | [`#L505`](../src/constants/prompts.ts) → [`getOutputStyleSection` · L151](../src/constants/prompts.ts) |
| `mcp_instructions` | **uncached** [`#L513`](../src/constants/prompts.ts)（或改走 delta attachment） |
| `scratchpad` | [`#L521`](../src/constants/prompts.ts) |
| `frc` | [`#L522`](../src/constants/prompts.ts) |
| `summarize_tool_results` | [`#L523`](../src/constants/prompts.ts) |
| `numeric_length_anchors` | ant-only [`#L531`](../src/constants/prompts.ts) |
| `token_budget` | feature `TOKEN_BUDGET` [`#L545`](../src/constants/prompts.ts) |
| `brief` | feature KAIROS／BRIEF [`#L553`](../src/constants/prompts.ts) |

### 特殊短路

- `CLAUDE_CODE_SIMPLE`：一行身份 + CWD + Date — [`#L450–L453`](../src/constants/prompts.ts)  
- PROACTIVE／KAIROS 激活：另一套简短 autonomous prompt — [`#L466–L488`](../src/constants/prompts.ts)

---

## 工具 description（36 个 `prompt.ts`）

路径模式：`src/tools/<Name>/prompt.ts`（链接到文件开头附近的导出）。

| 工具 | 文件 |
|---|---|
| Agent | [`AgentTool/prompt.ts` · L15](../src/tools/AgentTool/prompt.ts) |
| AskUserQuestion | [`AskUserQuestionTool/prompt.ts` · L3](../src/tools/AskUserQuestionTool/prompt.ts) |
| Bash | [`BashTool/prompt.ts` · L27](../src/tools/BashTool/prompt.ts) |
| Brief | [`BriefTool/prompt.ts` · L1](../src/tools/BriefTool/prompt.ts) |
| Config | [`ConfigTool/prompt.ts` · L9](../src/tools/ConfigTool/prompt.ts) |
| EnterPlanMode | [`EnterPlanModeTool/prompt.ts` · L4](../src/tools/EnterPlanModeTool/prompt.ts) |
| EnterWorktree | [`EnterWorktreeTool/prompt.ts` · L1](../src/tools/EnterWorktreeTool/prompt.ts) |
| ExitPlanMode | [`ExitPlanModeTool/prompt.ts` · L4](../src/tools/ExitPlanModeTool/prompt.ts) |
| ExitWorktree | [`ExitWorktreeTool/prompt.ts` · L1](../src/tools/ExitWorktreeTool/prompt.ts) |
| FileEdit | [`FileEditTool/prompt.ts` · L4](../src/tools/FileEditTool/prompt.ts) |
| FileRead | [`FileReadTool/prompt.ts` · L5](../src/tools/FileReadTool/prompt.ts) |
| FileWrite | [`FileWriteTool/prompt.ts` · L3](../src/tools/FileWriteTool/prompt.ts) |
| Glob | [`GlobTool/prompt.ts` · L1](../src/tools/GlobTool/prompt.ts) |
| Grep | [`GrepTool/prompt.ts` · L4](../src/tools/GrepTool/prompt.ts) |
| ListMcpResources | [`ListMcpResourcesTool/prompt.ts` · L1](../src/tools/ListMcpResourcesTool/prompt.ts) |
| LSP | [`LSPTool/prompt.ts` · L1](../src/tools/LSPTool/prompt.ts) |
| MCP | [`MCPTool/prompt.ts` · L2](../src/tools/MCPTool/prompt.ts)（常由服务端填） |
| NotebookEdit | [`NotebookEditTool/prompt.ts` · L1](../src/tools/NotebookEditTool/prompt.ts) |
| PowerShell | [`PowerShellTool/prompt.ts` · L18](../src/tools/PowerShellTool/prompt.ts) |
| ReadMcpResource | [`ReadMcpResourceTool/prompt.ts` · L1](../src/tools/ReadMcpResourceTool/prompt.ts) |
| RemoteTrigger | [`RemoteTriggerTool/prompt.ts` · L1](../src/tools/RemoteTriggerTool/prompt.ts) |
| ScheduleCron | [`ScheduleCronTool/prompt.ts` · L6](../src/tools/ScheduleCronTool/prompt.ts) |
| SendMessage | [`SendMessageTool/prompt.ts` · L3](../src/tools/SendMessageTool/prompt.ts) |
| Skill | [`SkillTool/prompt.ts` · L21](../src/tools/SkillTool/prompt.ts) |
| Sleep | [`SleepTool/prompt.ts` · L3](../src/tools/SleepTool/prompt.ts) |
| TaskCreate／Get／List／Stop／Update | [`TaskCreate` · L3](../src/tools/TaskCreateTool/prompt.ts)／[`Get` · L1](../src/tools/TaskGetTool/prompt.ts)／[`List` · L3](../src/tools/TaskListTool/prompt.ts)／[`Stop` · L1](../src/tools/TaskStopTool/prompt.ts)／[`Update` · L1](../src/tools/TaskUpdateTool/prompt.ts) |
| TeamCreate／Delete | [`TeamCreate` · L1](../src/tools/TeamCreateTool/prompt.ts)／[`TeamDelete` · L1](../src/tools/TeamDeleteTool/prompt.ts) |
| TodoWrite | [`TodoWriteTool/prompt.ts` · L3](../src/tools/TodoWriteTool/prompt.ts) |
| ToolSearch | [`ToolSearchTool/prompt.ts` · L9](../src/tools/ToolSearchTool/prompt.ts) |
| WebFetch | [`WebFetchTool/prompt.ts` · L1](../src/tools/WebFetchTool/prompt.ts) |
| WebSearch | [`WebSearchTool/prompt.ts` · L3](../src/tools/WebSearchTool/prompt.ts) |

另：`REPLTool`／`TaskOutputTool` 等可能无独立 `prompt.ts`；部分工具在 feature 门控下条件加载。

---

## 旁路／子 Agent prompts

### 旁路二次 LLM

| 模块 | 路径 |
|---|---|
| Compact 摘要 | [`BASE_COMPACT_PROMPT` · L61](../src/services/compact/prompt.ts) — [`services/compact/prompt.ts`](../src/services/compact/prompt.ts) |
| SessionMemory | [`DEFAULT_SESSION_MEMORY_TEMPLATE` · L11](../src/services/SessionMemory/prompts.ts) — [`SessionMemory/prompts.ts`](../src/services/SessionMemory/prompts.ts) |
| 抽记忆 | [`buildExtractAutoOnlyPrompt` · L50](../src/services/extractMemories/prompts.ts) — [`extractMemories/prompts.ts`](../src/services/extractMemories/prompts.ts) |
| MagicDocs | [`buildMagicDocsUpdatePrompt` · L98](../src/services/MagicDocs/prompts.ts) — [`MagicDocs/prompts.ts`](../src/services/MagicDocs/prompts.ts) |
| PromptSuggestion | [`SUGGESTION_PROMPT` · L258](../src/services/PromptSuggestion/promptSuggestion.ts) |
| 工具短标签 | [`TOOL_USE_SUMMARY_SYSTEM_PROMPT` · L15](../src/services/toolUseSummary/toolUseSummaryGenerator.ts) |
| autoDream | [`buildConsolidationPrompt` · L10](../src/services/autoDream/consolidationPrompt.ts) |
| 会话搜索 | [`SESSION_SEARCH_SYSTEM_PROMPT` · L15](../src/utils/agenticSessionSearch.ts) |
| 选相关 memory | [`SELECT_MEMORIES_SYSTEM_PROMPT` · L18](../src/memdir/findRelevantMemories.ts) |
| Prompt／Agent hooks | [`execPromptHook` · L21](../src/utils/hooks/execPromptHook.ts)／[`execAgentHook` · L36](../src/utils/hooks/execAgentHook.ts) |
| shell prefix／权限解释 | [`prefix.ts` · L92](../src/utils/shell/prefix.ts)／[`permissionExplainer` · L43](../src/utils/permissions/permissionExplainer.ts) |

### 内置子 Agent system prompts

目录：[`tools/AgentTool/built-in/`](../src/tools/AgentTool/built-in/generalPurposeAgent.ts)

| Agent | 锚点 |
|---|---|
| general-purpose | [`getGeneralPurposeSystemPrompt` · L19](../src/tools/AgentTool/built-in/generalPurposeAgent.ts) |
| Explore | [`getExploreSystemPrompt` · L13](../src/tools/AgentTool/built-in/exploreAgent.ts) |
| Plan | [`getPlanV2SystemPrompt` · L14](../src/tools/AgentTool/built-in/planAgent.ts) |
| Verification | [`VERIFICATION_SYSTEM_PROMPT` · L10](../src/tools/AgentTool/built-in/verificationAgent.ts) |
| Claude Code Guide | [`getClaudeCodeGuideBasePrompt` · L23](../src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts) |
| Statusline setup | [`STATUSLINE_SYSTEM_PROMPT` · L3](../src/tools/AgentTool/built-in/statuslineSetup.ts) |

另：

- [`DEFAULT_AGENT_PROMPT` · L758](../src/constants/prompts.ts)／[`enhanceSystemPromptWithEnvDetails` · L760](../src/constants/prompts.ts)
- Teammate：[`TEAMMATE_SYSTEM_PROMPT_ADDENDUM` · L8](../src/utils/swarm/teammatePromptAddendum.ts)
- Coordinator：[`getCoordinatorSystemPrompt` · L111](../src/coordinator/coordinatorMode.ts)

---

## 用户可编辑／可注入

| 类型 | 机制 | 进模型方式 |
|---|---|---|
| CLAUDE.md 层级 | Managed／`~/.claude`／项目／local／rules | [`getClaudeMds` · L1153](../src/utils/claudemd.ts) → [`getUserContext` · L155](../src/context.ts) → [`prependUserContext` · L449](../src/utils/api.ts) |
| Auto／Team memory | `MEMORY.md` + topic files | system section `memory` ← [`loadMemoryPrompt` · L419](../src/memdir/memdir.ts) |
| Skills | `.claude/skills/**/SKILL.md` 等 | Skill 工具列表；调用时 body 展开 — [`SkillTool/prompt.ts` · L21](../src/tools/SkillTool/prompt.ts) |
| Agents | `.claude/agents/*.md`、plugins | `getSystemPrompt()` 替换／追加 |
| Output styles | 内置 + 用户／插件 | section `output_style` — [`getOutputStyleSection` · L151](../src/constants/prompts.ts) |
| CLI flags | `--system-prompt`／`--append-system-prompt` | custom／append（见 [`buildEffectiveSystemPrompt` · L41](../src/utils/systemPrompt.ts)） |
| MCP prompts | `prompts/list` + `get` | slash → 消息块 |
| MCP instructions | `client.instructions` | section 或 delta attachment |
| Hooks `type: prompt`／`agent` | settings | [`execPromptHook` · L21](../src/utils/hooks/execPromptHook.ts)／[`execAgentHook` · L36](../src/utils/hooks/execAgentHook.ts) |
| Magic Docs 模板 | `~/.claude/magic-docs/prompt.md` | [`loadMagicDocsPrompt` · L66](../src/services/MagicDocs/prompts.ts) |

加载 CLAUDE.md：[`src/utils/claudemd.ts`](../src/utils/claudemd.ts)（[`getMemoryFiles` · L790](../src/utils/claudemd.ts)、[`getClaudeMds` · L1153](../src/utils/claudemd.ts)）。Memory：[`src/memdir/`](../src/memdir/memdir.ts)。

---

## QueryEngine 注入点

[`QueryEngine.ts · L284–L324`](../src/QueryEngine.ts)（源码）：

1. [`fetchSystemPromptParts` · L44](../src/utils/queryContext.ts) → defaultSystemPrompt／userContext／systemContext  
2. userContext 再 merge coordinator user context  
3. `systemPrompt = custom ?? default` +（条件）[`loadMemoryPrompt` · L419](../src/memdir/memdir.ts) + `appendSystemPrompt`  
4. 传入 `query({ systemPrompt, userContext, systemContext, ... })`  
5. [`query.ts`](../src/query.ts)：[`appendSystemContext` · L450](../src/query.ts)；[`prependUserContext` · L660](../src/query.ts)

交互路径（[`REPL.tsx` · L2535](../src/screens/REPL.tsx)）用 [`buildEffectiveSystemPrompt` · L41](../src/utils/systemPrompt.ts)，再同样走 `query`。

---

## 和心智模型的关系

[00-mental-model](./00-mental-model.md) 的主循环是 [`QueryEngine`](../src/QueryEngine.ts) → [`query()`](../src/query.ts) → tools。  
提示词挂在**进循环之前**：拼好 system／context，再进 `while`；循环内旁路（compact、抽记忆、suggestion）另起调用，不替换主 system 前缀（fork／缓存策略另议）。

---

## 还没搞懂

1. [`splitSysPromptPrefix` · L321](../src/utils/api.ts)／global cache 与 boundary、MCP delta 的精确拆分规则。  
2. 各旁路调用是否刻意共享主 prompt 前缀（cache 友好）还是完全独立。  
3. Skill 展开为 user 消息时的包装格式与权限边界。  
4. Coordinator／teammate／fork subagent 三套 system 的互斥与叠加。  
5. 下一刀待定：深挖 [`getSystemPrompt` · L444](../src/constants/prompts.ts) 静态段原文，或 [`prependUserContext` · L449](../src/utils/api.ts) + CLAUDE.md 合并顺序。
