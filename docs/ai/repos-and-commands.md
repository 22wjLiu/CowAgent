# 项目与开发命令

根目录是 Git 工作区与 Python 服务入口。`desktop/` 和 `mobile/` 各自有 `package.json` 和 `package-lock.json`，从对应目录安装 npm 依赖。以下命令来自仓库的脚本、文档、测试文件或 CI 配置。

## Python 后端

从仓库根目录执行。以下命令中的 `python` 指向当前开发环境的解释器，系统只提供 `python3` 时可用该命令。`pyproject.toml` 的 `requires-python >=3.7` 属于 `cow` CLI 包的元数据。日常后端验证可用 Python 3.11 或 3.13，对应 [后端测试 CI](../../.github/workflows/test.yml) 的版本矩阵。后端依赖在 `requirements.txt`，可选集成依赖在 `requirements-optional.txt`。

```bash
python -m pip install -r requirements.txt
python -m pip install -e .
```

按接入渠道和功能选择是否安装 `requirements-optional.txt`。源码开发可以从 `config-template.json` 创建本地 `config.json` 并配置模型和渠道，配置位置与环境变量覆盖规则见 [后端与运行时](backend-runtime.md)。

| 命令 | 用途 |
| --- | --- |
| `python app.py` | 前台运行 Python 服务。默认启动 Web 控制台，默认端口 `9899`。 |
| `cow start -f` | 安装 CLI 后前台运行同一个 `app.py`。 |
| `cow start` / `cow stop` | 通过 CLI 启停后台服务。显式只使用终端渠道时，`cow start` 会切到前台。 |
| `python -m pip install pytest` | 安装后端测试工具。 |
| `python -m pytest tests/ -q` | 运行完整后端测试，与 Linux CI 的命令一致。定向验证时把 `tests/` 换成对应的测试文件。 |
| `ruff check <改动的 Python 文件> --select F --ignore F403,F405` | 安装 Ruff 后，复现 CI 对改动 Python 文件的检查范围。 |
| `node --test tests/*.test.cjs` | 单独运行 Web 前端 Node 测试，覆盖聊天滚动和消息操作。 |

Windows shell 选择、设置和流式输出的验证命令见 [Windows Bash 测试 CI](../../.github/workflows/test-windows-bash.yml)。源码安装与 Docker Compose 的运行方式见 [手动安装](../guide/manual-install.mdx)。Compose 文件挂载工作区 `./cow` 和用户数据 `./cow-data`，后者由容器中的 `COW_DATA_DIR` 指定。见 [docker-compose.yml](../../docker/docker-compose.yml)。

## 桌面端

桌面 [发布 CI](../../.github/workflows/release.yml) 使用 Node.js 20 和 Python 3.11。`desktop/package-lock.json` 里的 `png-to-ico` 要求 Node.js ≥ 20。下表中的命令都从 `desktop/` 运行。

| 命令 | 用途 |
| --- | --- |
| `npm ci` | 按桌面端锁文件安装依赖。 |
| `npm run dev` | 先构建渲染进程和主进程，再启动 Electron。主进程从仓库根目录启动 Python 后端，因此还需要安装后端依赖。 |
| `npm run dev:renderer` / `npm run dev:main` | 分别在两个终端运行 Vite 和 Electron 主进程开发入口。 |
| `npm run build` | 运行 Vite 构建及 `tsc -p tsconfig.main.json`。 |
| `node --test tests/chat-scroll.test.mjs` | 安装依赖后验证桌面聊天滚动。 |
| `npm run dist` / `npm run dist:mac` / `npm run dist:win` | 通过 electron-builder 打包并写入 `release/`。先准备 `build/dist/cowagent-backend/`。 |

从仓库根目录运行 `bash desktop/build/build-backend.sh` 可构建桌面 Python 后端，脚本优先使用 Python 3.11，支持通过 `PYTHON` 指定解释器。它会创建构建虚拟环境、安装依赖并通过 PyInstaller 生成后端目录。Windows 7 兼容包使用独立的 [发布 CI](../../.github/workflows/release-win7.yml)，其中使用 Python 3.8 和 Electron 22。

## 移动端

从 `mobile/` 运行。`package.json` 要求 Node.js ≥ 20.19.4，README 推荐 Node.js 22。使用 Node.js 22 时选择 22.13.0 或更新的补丁版，以满足锁文件中 Vite 与 ESLint 依赖声明。`npm ci` 按锁文件安装，并通过 `postinstall` 执行 `patch-package`。

| 命令 | 用途 |
| --- | --- |
| `npm start` | 启动 Metro。 |
| `npm run android` / `npm run ios` | 运行对应原生应用。Android 需要 JDK 17、Android SDK/NDK 与 Python 3.13。iOS 需要 macOS、Xcode、CocoaPods 与 Python 3.13，首次运行前按 README 安装 Pods。 |
| `npm run web` | 在 `localhost:8090` 预览界面，默认代理到 `http://127.0.0.1:9899`，可用 `COW_BACKEND` 指定后端。 |
| `npm run typecheck` / `npm run lint` / `npm test` | 分别运行 TypeScript 检查、ESLint 和 Jest。 |
| `npm run bundle:harmony:release` | 生成 HarmonyOS 发布包使用的 Hermes bundle。签名、原生环境与调试步骤见移动端 README。 |

完整前置条件与各平台说明见 [移动端 README](../../mobile/README.md)。

## 文档站

从 `docs/` 运行 `mint dev`。根据 [文档站 README](../README.md)，需要 Node.js ≥ 20.17.0，并先用 `npm i -g mint` 安装 CLI。导航由 [docs.json](../docs.json) 管理，首次预览会将框架下载到 `~/.mintlify/`。
