# zhp/ — Claude Code 学习协作约定

> 本文件是 lux 与 agent 在本仓学习 Claude Code 的长期协作记忆。新会话先读这里，再读 [README.md](./README.md)。

## 我们在做什么

1. **主线：陪 lux 学习本仓 Claude Code 源码**（`src/`）。
2. 学习笔记写进 `zhp/`，**一个主题一篇**；编号 `NN-<topic>.md`。
3. 本树 README 标明 **leaked source**——只做机制学习，不讨论翻版／规避授权。
4. 与 DeepSeek Harness 的对照摘要可参考他仓 `deepseek-harness/zhp-claude/`，**以本仓源码为准**。

## 怎么合作

- 全程中文；**解释优先于改代码**，除非 lux 明确要求改实现。
- 聊天与 `zhp/*.md` 凡提源码：用**相对路径**（`../src/...`），只链**文件**不链目录；知道行号写 `#L`（例：`[getSystemPrompt](../src/constants/prompts.ts#L444)`）。**不要**用前缀 `/`（会被当成系统根目录）。目录请改链到该目录下某个具体文件。
- 证据五级：**实测／源码／日志／二手／推断**；推断不得写成观测。
- 每篇固定头部：状态、日期、基线、证据；结尾留「还没搞懂」。
- 先读再答；给完一刀结论后停，等 lux 指定下一刀。

## Git

- 分支：`zhp`（本仓）。
- 笔记落盘后：**自动 `git commit` + `git push`**（`docs(zhp): …`）；只提交 `zhp/` 相关文件。
- 不询问是否 commit／push（本会话起已约定）。
