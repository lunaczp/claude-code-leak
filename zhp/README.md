# zhp/

本目录收录 Claude Code 源码学习笔记（机制／结构）。分支：`zhp`。

协作约定：[AGENTS.md](./AGENTS.md)。

| 文件 | 内容 | 一句话 |
|---|---|---|
| [AGENTS.md](./AGENTS.md) | 协作约定 | 中文、证据分级、自动 commit+push |
| [00-mental-model.md](./00-mental-model.md) | 心智模型 | 入口→Query 循环→工具／权限／扩展地图 |
| [01-prompts.md](./01-prompts.md) | 提示词体系 | 主 system／context／工具 description／旁路／用户可编辑 |
| [02-prompt-example.md](./02-prompt-example.md) | 完整 prompt 例子 | 一次会话里 system + meta user + tools 长什么样 |

> 本仓为 **leaked source**（见根 README）；笔记只谈机制，不作法律意见，不讨论翻版／规避授权。

建议入口：[00-mental-model.md](./00-mental-model.md)。下一刀由 lux 指定。
