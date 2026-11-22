# 个人微信群机器人交接

CowAgent 的个人微信 `weixin` 通道只接收私聊。普通微信群目前由 Windows 虚拟机里的 `wechat4-vision-bot` 做桌面识别。先在测试群验证收发，再考虑接回 CowAgent。编程和工作任务的分流、后台执行与结果回群仍待实现。[CowAgent 微信消息适配](../../channel/weixin/weixin_message.py) · [wechat4-vision-bot 源码](https://github.com/HUTAOLIUYINGWEITA/wechat4-vision-bot/tree/3bf7b90580d4e467434ae2d39c52af2a6937db9a)

## 当前验证结果

- 虚拟机的项目目录是 `C:\WorkPlace\wechat4-vision-bot`，基于上游提交 `3bf7b90580d4e467434ae2d39c52af2a6937db9a`。用 `uv` 创建了 Python 3.12 环境并离线安装了 `requirements.txt`。Tesseract 通过 Windows `PATH` 调用，已安装 `chi_sim`。OCR 使用虚拟机 CPU。
- 最终实机从尚未打开 `Test` 的状态执行 `reply.py --read-msg --chat Test`，自动搜索并点开「最常使用」里的唯一精确匹配，核对 `Test (3)` 后读到 **1 条入站消息**，总耗时 **11.56 秒**。结束时微信仍可见且最大化。已打开 `Test` 的直接读取也曾通过，耗时 **5.75 秒**。
- 上游在全窗口 OCR 时可能把低饱和度群头像判为 `unknown`，或因标题裁剪过宽而读空。只读测试补丁单独扫描左侧搜索区域，在「群聊」「联系人」「最常使用」或「最近使用」中使用真实 OCR 候选做精确匹配。扫描在「更多」等分隔处结束，网络结果不会进入候选。同名会话会中止，未识别到分隔时交回原有选择逻辑。点开后仍检查会话标题。只读流程失败时返回非零退出码，收尾跳过 Esc。
- 离线回归已用原始失败截图验证 **100% 缩放的「最常使用」**与 **125% 缩放的「群聊」**。验证了排除网络同名结果、拒绝多个本地同名结果以及拒绝只有 OCR 合成候选或缺少结果边界的情形。实机测试覆盖这台虚拟机的当前布局。复查源码用 [只读测试补丁](patches/wechat4-vision-bot-read-test.patch)。
- 虚拟机上可双击 `C:\WorkPlace\cmd\wechat4-vision-test.cmd`。选 `1` 测 OCR，选 `2` 测视觉模型草稿。最近一次完整只读日志在虚拟机的 `C:\WorkPlace\cmd\wechat4-vision-test-last.log`。模型调用、真实自动发送、媒体和 Agent 任务回传均未验收。

## 在另一台机器复现

先从 CowAgent 的 `origin` 切到 `feat/weixin-agent-bot`。在要运行微信的 Windows 机器上单独检出上游仓库。以下示例假设 CowAgent 已检出到 `C:\WorkPlace\CowAgent`。

```cmd
git clone https://github.com/HUTAOLIUYINGWEITA/wechat4-vision-bot.git C:\WorkPlace\wechat4-vision-bot
cd /d C:\WorkPlace\wechat4-vision-bot
git switch --detach 3bf7b90580d4e467434ae2d39c52af2a6937db9a
git apply --check C:\WorkPlace\CowAgent\docs\ai\patches\wechat4-vision-bot-read-test.patch
git apply C:\WorkPlace\CowAgent\docs\ai\patches\wechat4-vision-bot-read-test.patch
uv venv --python 3.12 .venv
uv pip install --python ".venv\Scripts\python.exe" -r requirements.txt
copy config.example.json config.json
tesseract --list-langs
```

Tesseract 语言列表应出现 `chi_sim`。如果目标虚拟机无法下载，先在能联网的机器下载源码、中文数据以及匹配 Python 3.12、Windows x64 的 wheel，再用 SCP 传到虚拟机，用 `uv pip install --offline --no-index --find-links <wheel目录> --python ".venv\Scripts\python.exe" -r requirements.txt` 安装。[Tesseract Windows 安装包](https://github.com/UB-Mannheim/tesseract/wiki) · [官方简体中文识别数据](https://github.com/tesseract-ocr/tessdata_fast/blob/main/chi_sim.traineddata)

测试时保持 Windows 桌面已登录，微信可见并最大化。在桌面的 CMD 运行这条命令，它会核对当前会话，必要时自动搜索目标。可从另一个会话或空白会话区开始，验证完整搜索链路。仅在 SSH 会话里启动程序时，进程通常看不到已登录的微信桌面。

```cmd
".venv\Scripts\python.exe" reply.py --read-msg --chat "Test"
```

用 `ok` 判断成功与否，核对 `bubbles` 里的文本和 `elapsed`。有 `ok: true` 但读到零条时，检查可见区内是否有对方发的消息。

## 继续工作的顺序

1. 配置本地 Qwen3.8-27B 的 OpenAI 兼容服务。在 `wechat4-vision-bot` 的 `.credentials.json` 填虚拟机可达的 `baseUrl`、实际服务端模型 ID 和 API Key。本地服务无鉴权时可填占位值 `local`。模型接口需接受 `image_url`。先运行 `".venv\Scripts\python.exe" watcher.py --vision-test --chat "Test"`，核对 `gen.text` 和 `gen.latencyMs`。默认模型请求超时是 25 秒，输出最多 120 tokens，优先用非思考模式验证短回复。[Qwen3.8-27B 模型卡](https://huggingface.co/Qwen/Qwen3.8-27B)
2. 群消息收发验证完成后，把普通对话、编程任务和工作任务接到各自的处理入口，保留来源群和任务标识。后台任务完成后把结果送回同一群，并验证执行期间仍能继续聊天。图片预览和视频播放分别验收。

已在原 WSL 上为虚拟机的 `liu` 账户配置公钥登录。另一台机器需要自己的 SSH 密钥和到虚拟机的网络连接。模型凭据和本地聊天数据在各自的运行环境配置。
