# 个人微信群机器人交接

CowAgent 的 `weixin` 通道只收私聊（[weixin_message.py](../../channel/weixin/weixin_message.py) 固定 `is_group = False`）。普通微信群改由 Windows 虚拟机里的 wechat4-vision-bot 处理：截图识别 + 模拟键鼠，没有协议级接口。目标是先在测试群跑通收发，再接回 CowAgent。

## 代码位置

- 上游源码原样放在 [third_party/wechat4-vision-bot](../../third_party/wechat4-vision-bot)，对应 [上游提交 `3bf7b905`](https://github.com/HUTAOLIUYINGWEITA/wechat4-vision-bot/tree/3bf7b90580d4e467434ae2d39c52af2a6937db9a)。`third_party/.gitattributes` 让 git 不转换换行，`.cmd` 必须保持 CRLF。
- 我们的改动是 [只读测试补丁](patches/wechat4-vision-bot-read-test.patch)，没有应用到这份副本。
- `watcher.py` 负责轮询检测和调度，`reply.py` 负责打开会话、读消息和发送，`genreply.py` 调模型，`sendguard.py` 是发送护栏。watcher 每次读或发都会新起一个 `reply.py` 子进程。

## 当前状态

- 只读读群已在虚拟机实测通过。`reply.py --read-msg --chat Test` 从其他会话自动搜到并打开测试群，读到 1 条消息，耗时 11.56 秒；目标会话已打开时耗时 5.75 秒。
- 补丁做了三件事：
  - 单独识别搜索下拉框里的本地分区（群聊、联系人、最常使用、最近使用），只接受唯一的精确匹配，网络结果不参与。
  - 点开后按单行识别标题，只接受精确匹配。
  - 只读流程失败时返回非零退出码，收尾不再按 Esc。
- 离线回归覆盖 100% 缩放下的「最常使用」和 125% 缩放下的「群聊」。
- 虚拟机之后做了显卡直通，显示输出可能已变化。补丁只在原布局下验证过，需要重跑只读测试。
- 视觉模型生成、自动发送、图片视频、任务结果回传都还没验证。

## 截图路线的固有限制

- 不识别发送者：只按气泡左右位置区分对方和自己，头像列在识别前被裁掉，输出里没有发送者字段。
- 没有 @ 触发：白名单群里每条新消息都会走回复，只受限流约束。
- 只读聊天区下部最近 3 条对方消息（最多可调到 5 条）。刷屏时会漏，也拿不到原图和视频。
- 生成层把回复压成一行，截断到 `maxReplyChars`（默认 80 字），并禁止链接和文件，不适合回传任务结果。
- 所有动作共用一个桌面，只能串行执行。

## 下一步：CPU 直通 + RapidOCR

已排除改写成 C 或 C#、用显卡跑 OCR 两个方向。耗时主要来自界面等待、每次识别都启动一次 `tesseract.exe` 并重新加载模型、同一区域多遍识别，换语言或上显卡都解决不了这些。

1. 用虚拟机只读日志里各阶段的时间戳，算出 OCR、等待和界面操作各占多少耗时。
2. 宿主机是 fnOS，没查到它的虚拟机界面是否提供 CPU 直通选项。可以在宿主机查看 qemu 进程的 `-cpu` 参数，`host` 表示已直通。改好后在虚拟机运行 `tesseract --version`，输出应有 `Found AVX2` 和 `Found FMA`，再重跑只读测试对比耗时。
3. 在 Python 里把 Tesseract 换成 RapidOCR + PP-OCRv5 轻量版，用 CPU 版 `onnxruntime`。需要改的调用点：
   - `watcher.py`：[`_tess_run`](../../third_party/wechat4-vision-bot/watcher.py#L1160)、`_ocr_row_name_candidates`、`_ocr_row_preview`。
   - `reply.py`：`ocr_candidates`、`_ocr_tsv_lines`、`_ocr_words`、`_ocr_row_text_psm7`、`_ocr_bubble_text_precise`，以及补丁新增的 `ocr_chat_title`。

   现有的放大、多 PSM、反色和匹配阈值都是按 Tesseract 调的，替换后要重新标定。
4. 替换前先做离线对比：用同一批失败截图、同样的裁剪区域，分别记录两种引擎的识别结果、首次调用耗时和后续平均耗时。每次读取都是新起的 `reply.py` 进程，模型首次加载的时间也会计入每次读取。
5. 收发跑通后再配置视觉模型。在 `.credentials.json` 填 `baseUrl`、模型 ID 和 API Key，接口需要接受 `image_url`。运行 `watcher.py --vision-test --chat "Test"`，看 `gen.text` 和 `gen.latencyMs`。默认超时 25 秒，输出上限 120 tokens。Qwen3.8-27B 部署在哪台机器还没定，要确认能在超时内返回。
6. 接回 CowAgent 时：
   - 普通对话、编程任务和工作任务分开路由，保留来源群和任务标识。
   - 任务结果需要单独的发送路径，不能走 genreply 的截断。
   - 任务结果和聊天回复排进同一个串行动作队列。

## 虚拟机环境

- 运行条件：微信 4.x Windows 版，桌面保持登录、不锁屏，微信可见且最大化。从 SSH 会话启动的进程看不到桌面。
- 项目目录是 `C:\WorkPlace\wechat4-vision-bot`，Python 3.12 venv 用 uv 创建，Tesseract 在 PATH 中，装有 `chi_sim`。
- 虚拟机不能联网下载。依赖要在别处下载 Python 3.12、win_amd64 的 wheel，再用 `uv pip install --offline --no-index --find-links <目录>` 安装；RapidOCR 的模型文件也要提前拷入。
- 测试入口是 `C:\WorkPlace\cmd\wechat4-vision-test.cmd`：选 `1` 跑 OCR 只读测试，选 `2` 生成视觉模型草稿。日志在同目录的 `wechat4-vision-test-last.log`。
- 全新部署时，把 `third_party/wechat4-vision-bot` 复制到虚拟机，在该目录执行 `git apply <CowAgent>\docs\ai\patches\wechat4-vision-bot-read-test.patch`。
