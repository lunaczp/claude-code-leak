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
> 文中路径用相对链接（`../src/...#L`），只链文件；勿用前缀 `/`（会被当成系统根目录）。

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

- [`getSystemPrompt`](../src/constants/prompts.ts#L444)／[`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`](../src/constants/prompts.ts#L114) — [`src/constants/prompts.ts`](../src/constants/prompts.ts)
- [`buildEffectiveSystemPrompt`](../src/utils/systemPrompt.ts#L41) — [`src/utils/systemPrompt.ts`](../src/utils/systemPrompt.ts)
- [`appendSystemContext`](../src/utils/api.ts#L437)／[`prependUserContext`](../src/utils/api.ts#L449)／[`splitSysPromptPrefix`](../src/utils/api.ts#L321) — [`src/utils/api.ts`](../src/utils/api.ts)
- 调用点：[`query.ts#L450`](../src/query.ts#L450)（append）、[`query.ts#L660`](../src/query.ts#L660)（prepend）
- [`fetchSystemPromptParts`](../src/utils/queryContext.ts#L44) — [`src/utils/queryContext.ts`](../src/utils/queryContext.ts)
- [`QueryEngine` 拼装](../src/QueryEngine.ts#L292) — [`src/QueryEngine.ts`](../src/QueryEngine.ts)
- REPL：[`getSystemPrompt` + `buildEffectiveSystemPrompt`](../src/screens/REPL.tsx#L2535) — [`src/screens/REPL.tsx`](../src/screens/REPL.tsx)

注意：[`src/context/`](../src/context/QueuedMessageContext.tsx) 是 React UI Context，**不是** prompt 上下文；动态上下文在 [`src/context.ts`](../src/context.ts)（[`getSystemContext`](../src/context.ts#L116)、[`getUserContext`](../src/context.ts#L155)）。

---

## 分层

| 层 | 内容 | 注入 |
|---|---|---|
| L1 主 system | 身份／行为／工具使用规范 | `string[]` → API `system` |
| L2 覆盖／追加 | agent、`--system-prompt`、append、coordinator、override | [`buildEffectiveSystemPrompt`](../src/utils/systemPrompt.ts#L41) |
| L3 system context | git status、cacheBreaker 等 | [`appendSystemContext`](../src/utils/api.ts#L437) |
| L4 user context | CLAUDE.md 树、currentDate 等 | [`prependUserContext`](../src/utils/api.ts#L449) |
| L5 工具 description | 各 tool 的 `prompt.ts` | tool schema |
| L6 旁路 LLM | compact／memory／suggestion 等 | 各自独立调用 |

静态／动态分界：[`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`](../src/constants/prompts.ts#L114)（`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`）。boundary 前可走 global／org cache；后段为会话动态（见 [`prompts.ts` 注释](../src/constants/prompts.ts#L105)、[`splitSysPromptPrefix`](../src/utils/api.ts#L321)）。

Registry 机制：[`systemPromptSection`](../src/constants/systemPromptSections.ts#L20)／[`DANGEROUS_uncachedSystemPromptSection`](../src/constants/systemPromptSections.ts#L32)／[`resolveSystemPromptSections`](../src/constants/systemPromptSections.ts#L43) — [`src/constants/systemPromptSections.ts`](../src/constants/systemPromptSections.ts)；[`clearSystemPromptSections`](../src/constants/systemPromptSections.ts#L65) 在 `/clear`、`/compact` 时清缓存。

---

## `buildEffectiveSystemPrompt` 优先级

（源码注释，[`utils/systemPrompt.ts#L28–L40`](../src/utils/systemPrompt.ts#L28)）

0. `overrideSystemPrompt` — 整段替换（如 loop mode）— [`#L56`](../src/utils/systemPrompt.ts#L56)  
1. Coordinator（`CLAUDE_CODE_COORDINATOR_MODE`）— [`#L62`](../src/utils/systemPrompt.ts#L62) → [`getCoordinatorSystemPrompt`](../src/coordinator/coordinatorMode.ts#L111)  
2. Agent system prompt — proactive 时**追加**到 default（[`#L103`](../src/utils/systemPrompt.ts#L103)）；否则**替换**（[`#L115`](../src/utils/systemPrompt.ts#L115)）  
3. `customSystemPrompt`（`--system-prompt`）  
4. `defaultSystemPrompt`（[`getSystemPrompt`](../src/constants/prompts.ts#L444)）  
+ 除 override 外，可再附 `appendSystemPrompt`

---

## 主 system prompt 段落

### 静态段（boundary 前）

来自 [`getSystemPrompt`](../src/constants/prompts.ts#L444) 返回数组前半（[`#L560–L576`](../src/constants/prompts.ts#L560)）：

1. [`getSimpleIntroSection`](../src/constants/prompts.ts#L175) — 身份 + cyber risk + URL 规则  
2. [`getSimpleSystemSection`](../src/constants/prompts.ts#L186) — `# System`  
3. [`getSimpleDoingTasksSection`](../src/constants/prompts.ts#L199) — `# Doing tasks`（可被 output style 关掉）  
4. [`getActionsSection`](../src/constants/prompts.ts#L255) — `# Executing actions with care`  
5. [`getUsingYourToolsSection`](../src/constants/prompts.ts#L269) — `# Using your tools`  
6. [`getSimpleToneAndStyleSection`](../src/constants/prompts.ts#L430) — `# Tone and style`  
7. [`getOutputEfficiencySection`](../src/constants/prompts.ts#L403) — 输出效率／对用户沟通  
8. （可选）[`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`](../src/constants/prompts.ts#L114) — 插入于 [`#L573`](../src/constants/prompts.ts#L573)

### 动态 registry

注册于 [`#L491–L555`](../src/constants/prompts.ts#L491)；`/clear`、`/compact` 清缓存：

| name | 来源 |
|---|---|
| `session_guidance` | [`#L492`](../src/constants/prompts.ts#L492) |
| `memory` | [`loadMemoryPrompt`](../src/memdir/memdir.ts#L419)（[`memdir/`](../src/memdir/memdir.ts)） |
| `ant_model_override` | [`#L496`](../src/constants/prompts.ts#L496) |
| `env_info_simple` | [`#L499`](../src/constants/prompts.ts#L499) |
| `language` | [`#L502`](../src/constants/prompts.ts#L502) |
| `output_style` | [`#L505`](../src/constants/prompts.ts#L505) → [`getOutputStyleSection`](../src/constants/prompts.ts#L151) |
| `mcp_instructions` | **uncached** [`#L513`](../src/constants/prompts.ts#L513)（或改走 delta attachment） |
| `scratchpad` | [`#L521`](../src/constants/prompts.ts#L521) |
| `frc` | [`#L522`](../src/constants/prompts.ts#L522) |
| `summarize_tool_results` | [`#L523`](../src/constants/prompts.ts#L523) |
| `numeric_length_anchors` | ant-only [`#L531`](../src/constants/prompts.ts#L531) |
| `token_budget` | feature `TOKEN_BUDGET` [`#L545`](../src/constants/prompts.ts#L545) |
| `brief` | feature KAIROS／BRIEF [`#L553`](../src/constants/prompts.ts#L553) |

### 特殊短路

- `CLAUDE_CODE_SIMPLE`：一行身份 + CWD + Date — [`#L450–L453`](../src/constants/prompts.ts#L450)  
- PROACTIVE／KAIROS 激活：另一套简短 autonomous prompt — [`#L466–L488`](../src/constants/prompts.ts#L466)

---

## 工具 description（36 个 `prompt.ts`）

路径模式：`src/tools/<Name>/prompt.ts`（链接到文件开头附近的导出）。

| 工具 | 文件 |
|---|---|
| Agent | [`AgentTool/prompt.ts`](../src/tools/AgentTool/prompt.ts#L15) |
| AskUserQuestion | [`AskUserQuestionTool/prompt.ts`](../src/tools/AskUserQuestionTool/prompt.ts#L3) |
| Bash | [`BashTool/prompt.ts`](../src/tools/BashTool/prompt.ts#L27) |
| Brief | [`BriefTool/prompt.ts`](../src/tools/BriefTool/prompt.ts#L1) |
| Config | [`ConfigTool/prompt.ts`](../src/tools/ConfigTool/prompt.ts#L9) |
| EnterPlanMode | [`EnterPlanModeTool/prompt.ts`](../src/tools/EnterPlanModeTool/prompt.ts#L4) |
| EnterWorktree | [`EnterWorktreeTool/prompt.ts`](../src/tools/EnterWorktreeTool/prompt.ts#L1) |
| ExitPlanMode | [`ExitPlanModeTool/prompt.ts`](../src/tools/ExitPlanModeTool/prompt.ts#L4) |
| ExitWorktree | [`ExitWorktreeTool/prompt.ts`](../src/tools/ExitWorktreeTool/prompt.ts#L1) |
| FileEdit | [`FileEditTool/prompt.ts`](../src/tools/FileEditTool/prompt.ts#L4) |
| FileRead | [`FileReadTool/prompt.ts`](../src/tools/FileReadTool/prompt.ts#L5) |
| FileWrite | [`FileWriteTool/prompt.ts`](../src/tools/FileWriteTool/prompt.ts#L3) |
| Glob | [`GlobTool/prompt.ts`](../src/tools/GlobTool/prompt.ts#L1) |
| Grep | [`GrepTool/prompt.ts`](../src/tools/GrepTool/prompt.ts#L4) |
| ListMcpResources | [`ListMcpResourcesTool/prompt.ts`](../src/tools/ListMcpResourcesTool/prompt.ts#L1) |
| LSP | [`LSPTool/prompt.ts`](../src/tools/LSPTool/prompt.ts#L1) |
| MCP | [`MCPTool/prompt.ts`](../src/tools/MCPTool/prompt.ts#L2)（常由服务端填） |
| NotebookEdit | [`NotebookEditTool/prompt.ts`](../src/tools/NotebookEditTool/prompt.ts#L1) |
| PowerShell | [`PowerShellTool/prompt.ts`](../src/tools/PowerShellTool/prompt.ts#L18) |
| ReadMcpResource | [`ReadMcpResourceTool/prompt.ts`](../src/tools/ReadMcpResourceTool/prompt.ts#L1) |
| RemoteTrigger | [`RemoteTriggerTool/prompt.ts`](../src/tools/RemoteTriggerTool/prompt.ts#L1) |
| ScheduleCron | [`ScheduleCronTool/prompt.ts`](../src/tools/ScheduleCronTool/prompt.ts#L6) |
| SendMessage | [`SendMessageTool/prompt.ts`](../src/tools/SendMessageTool/prompt.ts#L3) |
| Skill | [`SkillTool/prompt.ts`](../src/tools/SkillTool/prompt.ts#L21) |
| Sleep | [`SleepTool/prompt.ts`](../src/tools/SleepTool/prompt.ts#L3) |
| TaskCreate／Get／List／Stop／Update | [`TaskCreate`](../src/tools/TaskCreateTool/prompt.ts#L3)／[`Get`](../src/tools/TaskGetTool/prompt.ts#L1)／[`List`](../src/tools/TaskListTool/prompt.ts#L3)／[`Stop`](../src/tools/TaskStopTool/prompt.ts#L1)／[`Update`](../src/tools/TaskUpdateTool/prompt.ts#L1) |
| TeamCreate／Delete | [`TeamCreate`](../src/tools/TeamCreateTool/prompt.ts#L1)／[`TeamDelete`](../src/tools/TeamDeleteTool/prompt.ts#L1) |
| TodoWrite | [`TodoWriteTool/prompt.ts`](../src/tools/TodoWriteTool/prompt.ts#L3) |
| ToolSearch | [`ToolSearchTool/prompt.ts`](../src/tools/ToolSearchTool/prompt.ts#L9) |
| WebFetch | [`WebFetchTool/prompt.ts`](../src/tools/WebFetchTool/prompt.ts#L1) |
| WebSearch | [`WebSearchTool/prompt.ts`](../src/tools/WebSearchTool/prompt.ts#L3) |

另：`REPLTool`／`TaskOutputTool` 等可能无独立 `prompt.ts`；部分工具在 feature 门控下条件加载。

---

## 旁路／子 Agent prompts

### 旁路二次 LLM

| 模块 | 路径 |
|---|---|
| Compact 摘要 | [`BASE_COMPACT_PROMPT`](../src/services/compact/prompt.ts#L61) — [`services/compact/prompt.ts`](../src/services/compact/prompt.ts) |
| SessionMemory | [`DEFAULT_SESSION_MEMORY_TEMPLATE`](../src/services/SessionMemory/prompts.ts#L11) — [`SessionMemory/prompts.ts`](../src/services/SessionMemory/prompts.ts) |
| 抽记忆 | [`buildExtractAutoOnlyPrompt`](../src/services/extractMemories/prompts.ts#L50) — [`extractMemories/prompts.ts`](../src/services/extractMemories/prompts.ts) |
| MagicDocs | [`buildMagicDocsUpdatePrompt`](../src/services/MagicDocs/prompts.ts#L98) — [`MagicDocs/prompts.ts`](../src/services/MagicDocs/prompts.ts) |
| PromptSuggestion | [`SUGGESTION_PROMPT`](../src/services/PromptSuggestion/promptSuggestion.ts#L258) |
| 工具短标签 | [`TOOL_USE_SUMMARY_SYSTEM_PROMPT`](../src/services/toolUseSummary/toolUseSummaryGenerator.ts#L15) |
| autoDream | [`buildConsolidationPrompt`](../src/services/autoDream/consolidationPrompt.ts#L10) |
| 会话搜索 | [`SESSION_SEARCH_SYSTEM_PROMPT`](../src/utils/agenticSessionSearch.ts#L15) |
| 选相关 memory | [`SELECT_MEMORIES_SYSTEM_PROMPT`](../src/memdir/findRelevantMemories.ts#L18) |
| Prompt／Agent hooks | [`execPromptHook`](../src/utils/hooks/execPromptHook.ts#L21)／[`execAgentHook`](../src/utils/hooks/execAgentHook.ts#L36) |
| shell prefix／权限解释 | [`prefix.ts`](../src/utils/shell/prefix.ts#L92)／[`permissionExplainer`](../src/utils/permissions/permissionExplainer.ts#L43) |

### 内置子 Agent system prompts

目录：[`tools/AgentTool/built-in/`](../src/tools/AgentTool/built-in/generalPurposeAgent.ts)

| Agent | 锚点 |
|---|---|
| general-purpose | [`getGeneralPurposeSystemPrompt`](../src/tools/AgentTool/built-in/generalPurposeAgent.ts#L19) |
| Explore | [`getExploreSystemPrompt`](../src/tools/AgentTool/built-in/exploreAgent.ts#L13) |
| Plan | [`getPlanV2SystemPrompt`](../src/tools/AgentTool/built-in/planAgent.ts#L14) |
| Verification | [`VERIFICATION_SYSTEM_PROMPT`](../src/tools/AgentTool/built-in/verificationAgent.ts#L10) |
| Claude Code Guide | [`getClaudeCodeGuideBasePrompt`](../src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts#L23) |
| Statusline setup | [`STATUSLINE_SYSTEM_PROMPT`](../src/tools/AgentTool/built-in/statuslineSetup.ts#L3) |

另：

- [`DEFAULT_AGENT_PROMPT`](../src/constants/prompts.ts#L758)／[`enhanceSystemPromptWithEnvDetails`](../src/constants/prompts.ts#L760)
- Teammate：[`TEAMMATE_SYSTEM_PROMPT_ADDENDUM`](../src/utils/swarm/teammatePromptAddendum.ts#L8)
- Coordinator：[`getCoordinatorSystemPrompt`](../src/coordinator/coordinatorMode.ts#L111)

---

## 用户可编辑／可注入

| 类型 | 机制 | 进模型方式 |
|---|---|---|
| CLAUDE.md 层级 | Managed／`~/.claude`／项目／local／rules | [`getClaudeMds`](../src/utils/claudemd.ts#L1153) → [`getUserContext`](../src/context.ts#L155) → [`prependUserContext`](../src/utils/api.ts#L449) |
| Auto／Team memory | `MEMORY.md` + topic files | system section `memory` ← [`loadMemoryPrompt`](../src/memdir/memdir.ts#L419) |
| Skills | `.claude/skills/**/SKILL.md` 等 | Skill 工具列表；调用时 body 展开 — [`SkillTool/prompt.ts`](../src/tools/SkillTool/prompt.ts#L21) |
| Agents | `.claude/agents/*.md`、plugins | `getSystemPrompt()` 替换／追加 |
| Output styles | 内置 + 用户／插件 | section `output_style` — [`getOutputStyleSection`](../src/constants/prompts.ts#L151) |
| CLI flags | `--system-prompt`／`--append-system-prompt` | custom／append（见 [`buildEffectiveSystemPrompt`](../src/utils/systemPrompt.ts#L41)） |
| MCP prompts | `prompts/list` + `get` | slash → 消息块 |
| MCP instructions | `client.instructions` | section 或 delta attachment |
| Hooks `type: prompt`／`agent` | settings | [`execPromptHook`](../src/utils/hooks/execPromptHook.ts#L21)／[`execAgentHook`](../src/utils/hooks/execAgentHook.ts#L36) |
| Magic Docs 模板 | `~/.claude/magic-docs/prompt.md` | [`loadMagicDocsPrompt`](../src/services/MagicDocs/prompts.ts#L66) |

加载 CLAUDE.md：[`src/utils/claudemd.ts`](../src/utils/claudemd.ts)（[`getMemoryFiles`](../src/utils/claudemd.ts#L790)、[`getClaudeMds`](../src/utils/claudemd.ts#L1153)）。Memory：[`src/memdir/`](../src/memdir/memdir.ts)。

---

## QueryEngine 注入点

[`QueryEngine.ts#L284–L324`](../src/QueryEngine.ts#L284)（源码）：

1. [`fetchSystemPromptParts`](../src/utils/queryContext.ts#L44) → defaultSystemPrompt／userContext／systemContext  
2. userContext 再 merge coordinator user context  
3. `systemPrompt = custom ?? default` +（条件）[`loadMemoryPrompt`](../src/memdir/memdir.ts#L419) + `appendSystemPrompt`  
4. 传入 `query({ systemPrompt, userContext, systemContext, ... })`  
5. [`query.ts`](../src/query.ts)：[`appendSystemContext`](../src/query.ts#L450)；[`prependUserContext`](../src/query.ts#L660)

交互路径（[`REPL.tsx#L2535`](../src/screens/REPL.tsx#L2535)）用 [`buildEffectiveSystemPrompt`](../src/utils/systemPrompt.ts#L41)，再同样走 `query`。

---

## 和心智模型的关系

[00-mental-model](./00-mental-model.md) 的主循环是 [`QueryEngine`](../src/QueryEngine.ts) → [`query()`](../src/query.ts) → tools。  
提示词挂在**进循环之前**：拼好 system／context，再进 `while`；循环内旁路（compact、抽记忆、suggestion）另起调用，不替换主 system 前缀（fork／缓存策略另议）。

---

## 还没搞懂

1. [`splitSysPromptPrefix`](../src/utils/api.ts#L321)／global cache 与 boundary、MCP delta 的精确拆分规则。  
2. 各旁路调用是否刻意共享主 prompt 前缀（cache 友好）还是完全独立。  
3. Skill 展开为 user 消息时的包装格式与权限边界。  
4. Coordinator／teammate／fork subagent 三套 system 的互斥与叠加。  
5. 下一刀待定：深挖 [`getSystemPrompt`](../src/constants/prompts.ts#L444) 静态段原文，或 [`prependUserContext`](../src/utils/api.ts#L449) + CLAUDE.md 合并顺序。
