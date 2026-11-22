# 后端与运行时

## 启动与渠道路由

`app.py:run()` 先加载配置和执行启动迁移，再由 `ChannelManager` 调用 `channel/channel_factory.py` 启动渠道。`channel_type` 支持单个名称、逗号分隔字符串或列表。`web_console` 默认启用，Web 渠道缺席时会自动加入启动列表。

改渠道配置或绑定时，同时检查 `app.py` 的 `_resolve_startup_channels()`、[channel_instances.py](../../channel/channel_instances.py) 和 [team.py](../../agent/team.py)。旧配置从 `config.json` 的 `channel_type` 启动。`team.json` 中的渠道实例可以携带各自的凭据和 Agent 绑定。当某个支持多实例的渠道已有显式实例时，启动逻辑会过滤同类型的旧入口，避免同一个机器人重复连接。可多开的渠道以 `MULTI_INSTANCE_READY` 为准。

## 消息、模型与工具

| 要改的行为 | 入口与关联层 |
| --- | --- |
| 入站消息转换和渠道回复 | 各渠道的消息适配类、`channel/chat_channel.py`、`channel/channel.py`，统一类型是 `bridge/context.py` 和 `bridge/reply.py`。 |
| Agent 会话执行 | `bridge/bridge.py` 进入 `bridge/agent_bridge.py`，再由 `agent/protocol/agent.py` 调用 `AgentStreamExecutor`。`agent` 配置控制渠道的 Agent 模式，普通聊天路径见 `Bridge.fetch_reply_content()`。 |
| OpenAI 兼容聊天 | `channel/web/api/openai_compat.py` 直接调用 `agent/chat/service.py` 的 `ChatService`。检查两条执行路径各自的事件转换、取消和持久化。 |
| 流式模型、工具调用和结果 | `agent/protocol/agent_stream.py` 负责执行循环。模型工厂和模型目录分别在 `models/bot_factory.py` 和 `models/model_catalog.py`。工具发现和可用性判断在 `agent/tools/tool_manager.py`，内置技能在 `skills/`，运行时技能加载在 `agent/skills/`。 |

改事件字段时，继续核对 [客户端与共享协议](clients-and-api.md) 中的消费者。调整工具依赖时也要核对客户端打包范围。

## Agent 作用域与会话持久化

`agent/registry.py` 解析 Agent 与各自的工作区，`agent/routing.py` 负责入站路由。会话 ID 在所属 Agent 内唯一，取消和临时追加指令沿用 `AgentBridge.scoped_session_key()` 的作用域。查改会话历史时，从 `agent/chat/session_service.py` 与 [conversation_store.py](../../agent/memory/conversation_store.py) 进入。注册表可用时，会话存放在默认 Agent 的共享 `index.db`，由 `agent_id` 隔离，默认 Agent 沿用历史行的空标识。直接按工作区拼数据库路径会绕过这层隔离和兼容逻辑。

工作区文件使用 [state_dir.py](../../common/state_dir.py) 的路径助手。在已有 `RuntimeIdentity` 的路径中保留身份上下文，跨线程或后台任务按现有调用方式传递身份。`state_root()` 解析 Agent 工作区，`shared_root()` 指向默认 Agent 的共享资源根。无 Agent 身份的启动任务会解析到默认 Agent，显式指定的无效 ID 会报错。固定写入 `~/cow` 会把其他 Agent 的文件放进默认工作区。

## 配置与部署模式

- `config.py:load_config()` 从 `get_data_root()` 读取 `config.json`。源码运行通常与仓库同目录，设置 `COW_DATA_DIR` 后使用其指向的目录。缺少用户配置时加载 `config-template.json`。非空的已知环境变量可以覆盖加载值，Compose 中的空值占位会跳过。
- `agent_workspace` 及 Agent 实例的 `workspace` 决定工作区。`COW_DATA_DIR` 用于配置等应用数据。改默认值或新增配置时，核对 [config.py](../../config.py)、[config-template.json](../../config-template.json) 与相关 Web 配置接口。
- `common/deployment.py` 根据 `COW_DEPLOYMENT` 区分源码、桌面、移动、Docker 和托管模式，也识别旧的 `COW_DESKTOP` 与 `COW_DOCKER` 环境变量。改启动、依赖或资源定位时，检查 [桌面和移动端的打包入口](clients-and-api.md)。
