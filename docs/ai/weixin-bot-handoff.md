# 个人微信群机器人交接

CowAgent 的 `weixin` 通道只收私聊（[weixin_message.py](../../channel/weixin/weixin_message.py) 固定 `is_group = False`）。普通微信群改由 Windows 虚拟机里的 wechat4-vision-bot 处理：截图识别 + 模拟键鼠，没有协议级接口。目标是先在测试群跑通收发，再接回 CowAgent。

## 代码位置

- 上游源码原样放在 [third_party/wechat4-vision-bot](../../third_party/wechat4-vision-bot)，对应 [上游提交 `3bf7b905`](https://github.com/HUTAOLIUYINGWEITA/wechat4-vision-bot/tree/3bf7b90580d4e467434ae2d39c52af2a6937db9a)。`third_party/.gitattributes` 让 git 不转换换行，`.cmd` 必须保持 CRLF。
- 按顺序应用三个补丁：[只读测试](patches/wechat4-vision-bot-read-test.patch)、[RapidOCR](patches/wechat4-vision-bot-rapidocr.patch)、[读群共用常驻 OCR](patches/wechat4-vision-bot-rapidocr-worker.patch)。仓库中的上游副本保留原样。
- `watcher.py` 负责轮询检测和调度，`reply.py` 负责打开会话、读消息和发送，`genreply.py` 调模型，`sendguard.py` 是发送护栏。watcher 每次读或发都会新起一个 `reply.py` 子进程。

## 当前状态

- 显卡直通后，已在虚拟机应用 RapidOCR 补丁并用 Tesseract 重跑只读测试。100% 缩放下，从其他会话搜索打开 `Test` 和直接读取已打开的 `Test` 都成功。两次各读到 1 条对方消息，未发送消息，识别文字还没逐字核对。
- 只读测试补丁做了三件事：
  - 单独识别搜索下拉框里的本地分区（群聊、联系人、最常使用、最近使用），只接受唯一的精确匹配，网络结果不参与。
  - 点开后按单行识别标题，只接受精确匹配。
  - 只读流程失败时返回非零退出码，收尾不再按 Esc。
- 离线回归覆盖 100% 缩放下的「最常使用」和 125% 缩放下的「群聊」。
- 2026-11-28 已在 Virtual-1 离线安装 RapidOCR 3.10.0 和 ONNX Runtime 1.32.0，使用 CPU 执行。内置 PP-OCRv6 small 在两张基线截图上都通过了机器人对 Test 群的标题精确校验，各识别出 1 条消息，文字与同图的 Tesseract 结果一致。默认引擎为 Tesseract。补丁包含以下改动。
  - 新增 `ocr_engine.py`。`config.json` 的 `ocr.engine` 选 `tesseract` 或 `rapidocr`，环境变量 `WECHAT4_OCR_ENGINE` 可临时覆盖。所有 OCR 调用点都经过它。
  - 启用 RapidOCR 后，`watcher.py` 在第一次读群时启动常驻 OCR 进程。后续 `reply.py --read-msg` 子进程把截图区域交给它，复用同一个模型。常驻进程失效时，本次读群在自己的进程内识别，下次读群重建常驻进程。直接运行 `reply.py --read-msg` 时使用该进程自己的模型。
  - `reply.py --read-msg` 输出的 JSON 增加 `timing`（OCR、模型加载、界面等待、截图、其余各占多少毫秒）和 `scale`。加 `--save-shot` 会把整窗截图存到 `screenshots\`。
  - 新增 `tools/ocr_bench.py`：在截图上逐个引擎对比识别结果和耗时，每个引擎单独起一个进程，第一张图的耗时包含模型加载。
- 视觉模型生成、自动发送、图片视频、任务结果回传都还没验证。
- Virtual-1 的离线回归用两个独立读群子进程处理基线截图。第二次模型初始化为 0 毫秒，同图的标题校验和聊天区识别从独立进程的 5.46 秒降到 3.52 秒。两张图的标题校验和消息结果均一致。模拟常驻进程退出后，本次识别回退成功，下次读取成功重建。实际打开微信窗口的耗时待复测。

## Tesseract 基线

2026-11-25 在虚拟机桌面运行。耗时取自 `timing`，单位为毫秒。总耗时从读取流程开始计时，Python 模块导入耗时另计。

| 路径 | 总耗时 | OCR（次数） | 界面等待 | 截图 | 其他 | 模块导入 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 搜索并打开 `Test` | 14,233 | 10,005（22） | 2,310 | 938 | 979 | 2,750 |
| 直接读取已打开的 `Test` | 6,312 | 5,236（14） | 561 | 219 | 295 | 375 |

截图在虚拟机机器人目录的 `screenshots\` 下，按上表顺序为 `reply-20261125-000028-read.png` 和 `reply-20261125-000035-read.png`。原始日志在 `logs\cow-wechat-baseline-20261125-000012\` 下，为 `read-1.log` 和 `read-2.log`。

在虚拟机桌面的 CMD 运行下面的命令。先从其他会话开始，成功打开 `Test` 后再跑一次。

```cmd
cd /d C:\WorkPlace\wechat4-vision-bot
set "WECHAT4_OCR_ENGINE=tesseract"
.venv\Scripts\python.exe reply.py --read-msg --chat "Test" --save-shot
```

## 截图路线的固有限制

- 不识别发送者：只按气泡左右位置区分对方和自己，头像列在识别前被裁掉，输出里没有发送者字段。
- 没有 @ 触发：白名单群里每条新消息都会走回复，只受限流约束。
- 只读聊天区下部最近 3 条对方消息（最多可调到 5 条）。刷屏时会漏，也拿不到原图和视频。
- 生成层把回复压成一行，截断到 `maxReplyChars`（默认 80 字），并禁止链接和文件，不适合回传任务结果。
- 所有动作共用一个桌面，只能串行执行。

## 下一步验证 RapidOCR

已排除改写成 C 或 C#、用显卡跑 OCR 两个方向。耗时主要来自界面等待、每次识别都启动一次 `tesseract.exe` 并重新加载模型、同一区域多遍识别，换语言或上显卡都解决不了这些。

1. 用同一批基线截图离线对比。运行 `.venv\Scripts\python.exe tools\ocr_bench.py --variants tesseract,rapidocr,v5mobile,v6tiny --scale 1.0 screenshots\reply-20261125-000028-read.png screenshots\reply-20261125-000035-read.png`。看识别是否正确，再看首张耗时和后续平均耗时。
2. 选出模型后，把 `ocr.engine` 改成 `rapidocr`，在 `ocr.rapidocr` 里填模型路径。在桌面的 CMD 执行 `set "WECHAT4_OCR_ENGINE="` 清除环境变量覆盖，再运行 `.venv\Scripts\python.exe reply.py --read-msg --chat "Test" --save-shot`，分别验证搜索打开和直接读取。匹配阈值和裁剪区域是按 Tesseract 标定的，识别偏差要在这一步调整。
3. 收发跑通后再配置视觉模型。在 `.credentials.json` 填 `baseUrl`、模型 ID 和 API Key，接口需要接受 `image_url`。运行 `watcher.py --vision-test --chat "Test"`，看 `gen.text` 和 `gen.latencyMs`。默认超时 25 秒，输出上限 120 tokens。Qwen3.8-27B 部署在哪台机器还没定，要确认能在超时内返回。
4. 接回 CowAgent 时：
   - 普通对话、编程任务和工作任务分开路由，保留来源群和任务标识。
   - 任务结果需要单独的发送路径，不能走 genreply 的截断。
   - 任务结果和聊天回复排进同一个串行动作队列。

## 虚拟机环境

- 运行条件：微信 4.x Windows 版，桌面保持登录、不锁屏，微信可见且最大化。从 SSH 会话启动的进程看不到桌面。
- 项目目录是 `C:\WorkPlace\wechat4-vision-bot`，Python 3.12 venv 用 uv 创建，Tesseract 在 PATH 中，装有 `chi_sim`。
- 虚拟机不能联网下载。依赖需从其他机器下载并拷入 Python 3.12、win_amd64 的 wheel。
- 重装 RapidOCR 和 ONNX Runtime 时，在机器人目录执行 `uv pip install --offline --no-index --find-links <wheel目录> --python .venv\Scripts\python.exe -r requirements-rapidocr.txt --overrides overrides-rapidocr.txt`。必须带上 overrides 文件，否则会多装一份 `opencv-python-headless`，和现有的 `opencv-python` 抢 `cv2`。wheel 目录用 `uv pip compile --python-platform windows --python-version 3.12` 解析同样两个文件后，逐个从 PyPI 下载。rapidocr 的 wheel 自带 PP-OCRv6 small。PP-OCRv5 mobile 和 PP-OCRv6 tiny 从 RapidOCR v3.10.0 模型仓库下载，放到 `models\`。
- 测试入口是 `C:\WorkPlace\cmd\wechat4-vision-test.cmd`：选 `1` 跑 OCR 只读测试，选 `2` 生成视觉模型草稿。日志在同目录的 `wechat4-vision-test-last.log`。
- 全新部署时，把 `third_party/wechat4-vision-bot` 复制到虚拟机，在该目录按文首顺序用 `git apply` 应用三个补丁。已应用前两个补丁的环境只需应用第三个。
