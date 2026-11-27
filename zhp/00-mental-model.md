# Claude Code 心智模型：入口 → 循环 → 扩展

| | |
|---|---|
| 状态 | 学习中 |
| 日期 | 2026-11-27 |
| 基线 | 本仓 `zhp` 分支；对照 commit `b983a3b`（从 `main` 分出） |
| 证据 | 源码；`docs/architecture.md`／`docs/subsystems.md`（二手） |

> 本仓为 **leaked source**；本文只做机制学习。
> 证据约定：**实测** = 本机跑过；**源码** = 读仓库确认；**二手** = 仓内 docs；**推断** = 尚未验证。

## 一句话

**Claude Code 是 Bun + React/Ink 的产品单体：入口经 Commander／init 进 REPL，核心是 `QueryEngine.submitMessage` 包住 `query()` 的 `while` 循环（stream → tool_use → `runTools` → 续写），工具／权限／transcript／hooks／plugins／skills／MCP／bridge 全挂在这条链上。**（源码）

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

### 分层（源码，勿被 docs 行数误导）

| 层 | 职责 | 锚点 |
|---|---|---|
| `QueryEngine` | 会话编排 + 持久化；调用 `query`、`recordTranscript` | [`../src/QueryEngine.ts`](../src/QueryEngine.ts) 类注释／`submitMessage` |
| `query` | 真正的 agent loop：`while (true)` → stream → `runTools` → continue／terminal | [`../src/query.ts`](../src/query.ts) `while (true)`；`runTools` |
| 文档管道图 | User Input → Query Engine → Tool Loop | `docs/architecture.md`（二手） |

---

## 建议阅读顺序

1. **入口链**：`cli.tsx` → `main.tsx` 开头预取 + `init.ts` → 何时进 REPL
2. **会话引擎**：`QueryEngine.submitMessage` — 拼 system／tools、调 `query`、调 `recordTranscript`
3. **Query 循环**：`query.ts` 的 `while`：流式事件 → `tool_use` → `runTools` → continue／terminal（优先深挖）
4. **工具契约**：`Tool.ts` 的 `Tool`／`buildTool` + `tools.ts` 的 `getTools`／`assembleToolPool`
5. **权限＋沙箱**：`hooks/toolPermission/` + `utils/permissions/`；Bash 侧 sandbox／adapter
6. **Transcript**：`sessionStorage.getTranscriptPath`／`recordTranscript`／resume（`conversationRecovery`）
7. **扩展面**：hooks 事件名 → plugins／skills 加载 → `services/mcp/` ↔ `entrypoints/mcp.ts`
8. **IDE bridge**：`bridge/bridgeMain.ts` + permission 回调（`BRIDGE_MODE`）

---

## 还没搞懂

1. `query` 循环各 `continue`／terminal 分支与 compact／budget 的精确触发条件。
2. transcript 字段语义与 resume 边界。
3. bridge 协议与权限代理的完整消息面。
4. hooks／plugins／skills／MCP 的加载时序与优先级。
5. 下一刀待定：query 循环 或 BashTool（会话锚点）。
