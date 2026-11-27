# Claude Code 心智模型：入口 → 循环 → 扩展

| | |
|---|---|
| 状态 | 学习中 |
| 日期 | 2026-11-27 |
| 基线 | 本仓 `zhp` 分支；对照 commit `b983a3b`（从 `main` 分出） |
| 证据 | 源码；[`docs/architecture.md`](../docs/architecture.md)／[`docs/subsystems.md`](../docs/subsystems.md)（二手） |

> 本仓为 **leaked source**；本文只做机制学习。  
> 证据约定：**实测** = 本机跑过；**源码** = 读仓库确认；**二手** = 仓内 docs；**推断** = 尚未验证。  
> 文中路径用相对链接（`../src/...#L`），只链文件；勿用前缀 `/`（会被当成系统根目录）。

## 一句话

**Claude Code 是 Bun + React/Ink 的产品单体：入口经 Commander／init 进 REPL，核心是 [`QueryEngine.submitMessage`](../src/QueryEngine.ts#L209) 包住 [`query()`](../src/query.ts#L219) 的 [`while`](../src/query.ts#L307) 循环（stream → tool_use → [`runTools`](../src/services/tools/toolOrchestration.ts#L19) → 续写），工具／权限／transcript／hooks／plugins／skills／MCP／bridge 全挂在这条链上。**（源码）

---

## 目录地图

```
入口
  src/entrypoints/cli.tsx     # 启动分流（--version／MCP／主路径）
  src/main.tsx                # Commander + 预取副作用 → REPL
  src/entrypoints/init.ts     # config／telemetry／OAuth／policy
  src/entrypoints/mcp.ts      # 本进程当 MCP server
  src/entrypoints/sdk/        # Agent SDK 类型／协议面

主循环
  src/QueryEngine.ts          # 会话级：submitMessage、写 transcript、拼上下文
  src/query.ts                # 回合级：while(true) 流式 + 工具续写
  src/query/                  # query 配置／deps／transitions／stopHooks
  src/services/tools/         # runTools／StreamingToolExecutor

工具与权限
  src/Tool.ts                 # Tool／ToolUseContext／buildTool
  src/tools.ts                # 注册表 getAllBaseTools／getTools／assembleToolPool
  src/tools/*Tool/            # Bash／File*／Agent／Skill／MCP…
  src/hooks/toolPermission/   # 权限门
  src/utils/permissions/      # 规则匹配等

持久化
  src/utils/sessionStorage.ts # ~/.claude/projects/… append transcript

扩展
  src/utils/hooks/ + hooks 类型   # lifecycle hooks
  src/plugins/ + services/plugins # plugins／marketplace
  src/skills/                     # skills
  src/services/mcp/               # MCP 客户端
  src/bridge/                     # IDE bridge（BRIDGE_MODE）

壳
  src/screens/REPL.tsx、components/、state/AppState*
  src/commands.ts + commands/     # 斜杠命令（另一条用户入口）
```

可点击入口：

| 区域 | 链接 |
|---|---|
| CLI 分流 | [`src/entrypoints/cli.tsx`](../src/entrypoints/cli.tsx) |
| main | [`main()`](../src/main.tsx#L585) — [`src/main.tsx`](../src/main.tsx) |
| init | [`init`](../src/entrypoints/init.ts#L57) |
| MCP server | [`startMCPServer`](../src/entrypoints/mcp.ts#L35) |
| SDK | [`src/entrypoints/sdk/`](../src/entrypoints/sdk/controlSchemas.ts) |
| QueryEngine | [`class`](../src/QueryEngine.ts#L184)／[`submitMessage`](../src/QueryEngine.ts#L209) |
| query loop | [`query()`](../src/query.ts#L219)／[`while (true)`](../src/query.ts#L307) |
| query 子目录 | [`src/query/`](../src/query/config.ts) |
| 工具编排 | [`runTools`](../src/services/tools/toolOrchestration.ts#L19)／[`StreamingToolExecutor`](../src/services/tools/StreamingToolExecutor.ts#L40) |
| Tool 契约 | [`buildTool`](../src/Tool.ts#L783) — [`src/Tool.ts`](../src/Tool.ts) |
| 工具注册表 | [`getAllBaseTools`](../src/tools.ts#L193)／[`assembleToolPool`](../src/tools.ts#L345) — [`src/tools.ts`](../src/tools.ts) |
| 工具实现 | [`src/tools/`](../src/tools/utils.ts) |
| 权限门 | [`src/hooks/toolPermission/`](../src/hooks/toolPermission/PermissionContext.ts)／[`src/utils/permissions/`](../src/utils/permissions/PermissionMode.ts) |
| Transcript | [`getTranscriptPath`](../src/utils/sessionStorage.ts#L202)／[`recordTranscript`](../src/utils/sessionStorage.ts#L1408) |
| hooks | [`src/utils/hooks/`](../src/utils/hooks/AsyncHookRegistry.ts) |
| plugins | [`src/plugins/`](../src/plugins/builtinPlugins.ts)／[`src/services/plugins/`](../src/services/plugins/PluginInstallationManager.ts) |
| skills | [`src/skills/`](../src/skills/bundledSkills.ts) |
| MCP 客户端 | [`src/services/mcp/`](../src/services/mcp/InProcessTransport.ts) |
| bridge | [`bridgeMain`](../src/bridge/bridgeMain.ts#L1980) — [`src/bridge/`](../src/bridge/bridgeApi.ts) |
| REPL 壳 | [`src/screens/REPL.tsx`](../src/screens/REPL.tsx) |
| 斜杠命令 | [`src/commands.ts`](../src/commands.ts)／[`src/commands/`](../src/commands/advisor.ts) |

### 分层（源码，勿被 docs 行数误导）

| 层 | 职责 | 锚点 |
|---|---|---|
| `QueryEngine` | 会话编排 + 持久化；调用 `query`、`recordTranscript` | [`QueryEngine.ts#L184`](../src/QueryEngine.ts#L184) 类；[`submitMessage`](../src/QueryEngine.ts#L209)；[`recordTranscript` 调用](../src/QueryEngine.ts#L451) |
| `query` | 真正的 agent loop：`while (true)` → stream → tools → continue／terminal | [`query.ts#L219`](../src/query.ts#L219)；[`while`](../src/query.ts#L307) |
| 文档管道图 | User Input → Query Engine → Tool Loop | [`docs/architecture.md`](../docs/architecture.md)（二手） |

提示词拼装见 [01-prompts.md](./01-prompts.md)。

---

## 建议阅读顺序

1. **入口链**：[`cli.tsx`](../src/entrypoints/cli.tsx) → [`main.tsx`](../src/main.tsx#L585) 开头预取 + [`init.ts`](../src/entrypoints/init.ts#L57) → 何时进 REPL  
2. **会话引擎**：[`QueryEngine.submitMessage`](../src/QueryEngine.ts#L209) — 拼 system／tools、调 `query`、调 [`recordTranscript`](../src/utils/sessionStorage.ts#L1408)  
3. **Query 循环**：[`query.ts`](../src/query.ts#L307) 的 `while`：流式事件 → `tool_use` → tools → continue／terminal（优先深挖）  
4. **工具契约**：[`buildTool`](../src/Tool.ts#L783) + [`assembleToolPool`](../src/tools.ts#L345)／[`getAllBaseTools`](../src/tools.ts#L193)  
5. **权限＋沙箱**：[`hooks/toolPermission/`](../src/hooks/toolPermission/PermissionContext.ts) + [`utils/permissions/`](../src/utils/permissions/PermissionMode.ts)；Bash 侧 sandbox／adapter  
6. **Transcript**：[`getTranscriptPath`](../src/utils/sessionStorage.ts#L202)／[`recordTranscript`](../src/utils/sessionStorage.ts#L1408)／resume（`conversationRecovery`）  
7. **扩展面**：hooks 事件名 → plugins／skills 加载 → [`services/mcp/`](../src/services/mcp/InProcessTransport.ts) ↔ [`entrypoints/mcp.ts`](../src/entrypoints/mcp.ts#L35)  
8. **IDE bridge**：[`bridgeMain`](../src/bridge/bridgeMain.ts#L1980) + permission 回调（`BRIDGE_MODE`）

---

## 还没搞懂

1. `query` 循环各 `continue`／terminal 分支与 compact／budget 的精确触发条件。  
2. transcript 字段语义与 resume 边界。  
3. bridge 协议与权限代理的完整消息面。  
4. hooks／plugins／skills／MCP 的加载时序与优先级。  
5. 下一刀待定：query 循环 或 BashTool（会话锚点）。
