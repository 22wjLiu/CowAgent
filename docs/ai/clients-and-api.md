# 客户端与共享协议

## Web API 消费入口

Python Web 渠道同时提供 Web 控制台和桌面端、移动端使用的接口。路由表及 handler 导入在 [web_channel.py](../../channel/web/web_channel.py)，各 handler 在 `channel/web/api/`，服务和聊天队列在 [core/channel.py](../../channel/web/core/channel.py)。web.py 按路由表顺序匹配。新增 API 时检查前面的正则路由，通用页面路由保留在后面，避免吞掉 API 路径。

| 客户端 | 改接口时查找的位置 |
| --- | --- |
| Web 控制台 | `channel/web/chat.html`、`channel/web/static/js/`，聊天发送与事件消费见 `chat/send.js`。 |
| 桌面 | `desktop/src/renderer/src/api/client.ts`、`desktop/src/renderer/src/types.ts`，跨 Electron 进程的调用还涉及 `desktop/src/main/preload.ts`。 |
| 移动 | `mobile/src/api/cowApi.ts`、`mobile/src/api/http.ts`、`mobile/src/api/sse.ts`、`mobile/src/protocol/` 与 `mobile/src/realtime/chatStream.ts`。 |

改鉴权时同时核对 `channel/web/api/auth.py`、`channel/web/core/_common.py` 和现有客户端。桌面和移动端会通过设备 token 访问 API，Web 浏览器也使用登录 cookie。移动端连接探测包含 `/api/health` 与 `/auth/check` 的回退路径，参见 [cowApi.ts](../../mobile/src/api/cowApi.ts)。

## 聊天与断线恢复

`POST /message` 发起一轮聊天，流式响应由 `GET /stream?request_id=…` 读取，取消走 `POST /cancel`。入口在 [api/chat.py](../../channel/web/api/chat.py)。SSE 帧使用单调增加的序号作为 `id`，重连可以带 `after_seq` 或 `Last-Event-ID`。服务端会话详情可返回活动请求和与持久化快照对应的游标，相关逻辑在 `channel/web/api/sessions.py` 和 `WebChannel.resumable_stream()`。

修改事件的类型、字段、终止条件或序号时，检查三端的增量处理，以及会话详情载入后的恢复流程。游标跳过了尚未持久化的序号会漏消息，游标落后于已显示的快照会重放消息。OpenAI 兼容接口使用 `channel/web/api/openai_compat.py` 的响应转换，执行入口见 [后端与运行时](backend-runtime.md)。

## 桌面与移动端的嵌入后端

- **桌面**的 Electron 主进程通过 `desktop/src/main/python-manager.ts` 管理 Python 进程。开发时从仓库运行 `app.py`，安装包运行 `resources/backend/` 下的 PyInstaller 后端。新增 Python 依赖、动态导入、模板或资源时，检查 [桌面依赖清单](../../desktop/build/requirements-desktop.txt) 和 [PyInstaller spec](../../desktop/build/cowagent-backend.spec)。安装包的资源目录由 [desktop/package.json](../../desktop/package.json) 配置。
- **移动**同时支持连接远程后端和使用内嵌后端。原生构建通过 [kernel/bundle.py](../../mobile/kernel/bundle.py) 复制核心源码，依赖单列在 [kernel/requirements.txt](../../mobile/kernel/requirements.txt)。增加顶层包或文件时核对复制清单，移动端内核会跳过 Web 控制台静态资源。`COW_DEPLOYMENT=mobile` 下后端只启动 Web 渠道并绑定回环地址，工具可用性由 `agent/tools/tool_manager.py` 判断。

启动、构建和定向验证命令见 [项目与开发命令](repos-and-commands.md)。
