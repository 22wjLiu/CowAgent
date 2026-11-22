# CLAUDE.md

CowAgent 是一个可通过 Web、桌面端、移动端及即时通信渠道使用的 AI 助手。渠道将消息转成统一上下文，Python 后端路由到对应 Agent，调用模型和工具，再将文本、流式事件或文件交回渠道。

## 仓库边界

这是一个 Git 仓库。Python 后端、桌面端、移动端和文档站都从根目录管理。共享接口的服务端与客户端改动可以放在同一个提交中。

`AGENTS.md` 是指向本文件的软链接，项目指令统一维护在 `CLAUDE.md`，不要把软链接替换成独立副本。

| 路径 | 技术栈 | 职责 |
| --- | --- | --- |
| `app.py`、`config.py`、`cli/` | Python、Click | 服务启动、配置加载与 `cow` 命令 |
| `channel/`、`bridge/`、`models/` | Python、web.py | 渠道接入、消息路由、模型适配与 Web API |
| `agent/`、`common/` | Python | Agent 运行、工具、技能、记忆、会话与运行时路径 |
| `desktop/` | Electron、React、TypeScript | 桌面客户端与 Python 后端打包 |
| `mobile/` | React Native、TypeScript | Android、iOS、HarmonyOS 客户端、内嵌后端与浏览器预览 |
| `tests/`、`docs/` | pytest、Node.js、Mintlify | 后端与 Web 回归测试、用户文档站 |

## 容易漏改的边界

1. **改消息执行、取消或持久化时，检查 `AgentBridge` 和 `ChatService` 两条入口。** 两者都使用 `AgentStreamExecutor`，只改一条入口会让渠道会话与 OpenAI 兼容接口表现不一致。见 [后端与运行时](docs/ai/backend-runtime.md)。
2. **改会话标识或文件落盘时，沿用 Agent 作用域和路径解析。** 会话 ID 只在所属 Agent 内唯一，忽略这个范围可能让一次取消或写入影响另一 Agent。见 [后端与运行时](docs/ai/backend-runtime.md)。
3. **改 Web API 或 SSE 字段时，核对 Web、桌面端和移动端的调用方。** 会话恢复依赖请求 ID 和事件游标，只更新服务端可能导致旧客户端重放或漏掉消息。见 [客户端与共享协议](docs/ai/clients-and-api.md)。
4. **改后端依赖或随包资源时，检查桌面 PyInstaller 和移动端内嵌内核的清单。** 根目录能运行的改动仍可能因漏打包而在安装后的客户端失效。见 [客户端与共享协议](docs/ai/clients-and-api.md)。

## 提交规范

按 [CONTRIBUTING.md](CONTRIBUTING.md) 尽量使用英文撰写 issue、PR、提交说明、代码注释和 docstring。提交与 PR 标题使用简短的祈使句，推荐 Conventional Commits。每个 PR 聚焦一个逻辑改动，描述改动内容与原因。

## 文档索引

| 文件 | 什么时候读 |
| --- | --- |
| [docs/ai/repos-and-commands.md](docs/ai/repos-and-commands.md) | 安装依赖、启动服务、运行测试、构建客户端或预览文档站时，查运行时、工作目录与命令。 |
| [docs/ai/backend-runtime.md](docs/ai/backend-runtime.md) | 改渠道启动、Agent 路由、模型或工具执行、配置、会话持久化或工作区路径时。 |
| [docs/ai/clients-and-api.md](docs/ai/clients-and-api.md) | 改 Web API、鉴权、聊天流，或桌面端与移动端的客户端、内嵌后端和打包资源时。 |
