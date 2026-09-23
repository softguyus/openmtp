# OpenMTP 功能改进与 Mac ARM64 构建汇总报告

本报告汇总了在本项目（[softguyus/openmtp](https://github.com/softguyus/openmtp)，Fork 自 [ganeshrvel/openmtp](https://github.com/ganeshrvel/openmtp)）中完成的核心功能改造、Mac ARM 架构适配以及 GitHub Actions CI 自动化构建全过程。

---

## 一、核心功能改进：“跳过已存在文件，仅传输新文件”

### 1. 业务背景
在原版 OpenMTP 中，文件传输遇到目标目录同名文件时缺乏细粒度的冲突处理，不支持仅传输增量新文件的需求。

### 2. 实现方案与核心模块修改
1. **多数据源层检测 (`checkFilesExist`)**：
   - [`app/data/file-explorer/data-sources/FileExplorerLocalDataSource.js`](app/data/file-explorer/data-sources/FileExplorerLocalDataSource.js)：通过 Node.js `fs.existsSync` 高效检测本地目标文件/目录是否存在。
   - [`app/data/file-explorer/data-sources/FileExplorerKalamDataSource.js`](app/data/file-explorer/data-sources/FileExplorerKalamDataSource.js)：利用 Kalam MTP 底层接口检测 Android 存储设备中同名条目。
   - [`app/data/file-explorer/data-sources/FileExplorerLegacyDataSource.js`](app/data/file-explorer/data-sources/FileExplorerLegacyDataSource.js)：为 Legacy MTP 驱动补充存在性检测。
   - [`app/data/file-explorer/repositories/FileExplorerRepository.js`](app/data/file-explorer/repositories/FileExplorerRepository.js) 与 [`FileExplorerController.js`](app/data/file-explorer/controllers/FileExplorerController.js)：串联底层数据源并暴露给前端 UI。

2. **传输交互与冲突处理逻辑**：
   - [`app/containers/HomePage/components/FileExplorer.jsx`](app/containers/HomePage/components/FileExplorer.jsx)：
     - 在执行粘贴/拖拽传输前，主动调用 `checkFilesExist` 比对待传队列与目标路径。
     - 若发现冲突，弹出冲突处理对话框，提供 **“全部替换 (Replace All)”**、**“跳过已存在 (Skip Existing)”** 与 **“取消 (Cancel)”** 三个选项。
     - 选择“跳过已存在”时，系统自动过滤队列中已存在的文件，仅将新文件/文件夹加入传输队列；若所有文件均已存在，则提示无需传输并安全终止。

3. **设置项策略支持 (`FILE_CONFLICT_POLICY`)**：
   - [`app/enums/index.js`](app/enums/index.js)：新增 `FILE_CONFLICT_POLICY` 枚举（`ask` 询问、`skip` 自动跳过、`replace` 自动覆盖）。
   - [`app/containers/Settings/reducers.js`](app/containers/Settings/reducers.js) 与 [`selectors.js`](app/containers/Settings/selectors.js)：状态管理持久化。
   - [`app/containers/Settings/components/SettingsDialog.jsx`](app/containers/Settings/components/SettingsDialog.jsx)：在设置面板中提供全局文件冲突策略配置项。

---

## 二、Mac ARM64 (Apple Silicon) 原生架构适配

### 1. 二进制与打包配置
- [`electron-builder-config.js`](electron-builder-config.js)：
  - 完善架构检测逻辑：识别 `process.arch === 'arm64'`、`process.env.TARGET_ARCH === 'arm64'` 或 CLI 参数 `--arm64`。
  - 自动从 `build/mac/bin/arm64` 中打包原生架构的命令行与驱动二进制。
  - 支持免证书代码签名打包模式（`forceCodeSigning: process.env.CSC_IDENTITY_AUTO_DISCOVERY !== 'false'`）。

### 2. 构建脚本扩展
- [`package.json`](package.json)：
  - 新增 `package-mac-arm64`: `yarn build && cross-env TARGET_ARCH=arm64 dotenv -- electron-builder --config electron-builder-config.js build --mac --arm64 --publish never`
  - 新增 `package-mac-arm64-without-notarize`: 免公证打包模式。
  - 新增 `package-mac-arm64-without-notarize-no-verify`: 直接打包已生成的构建产物。

---

## 三、GitHub Actions CI 自动化构建与踩坑解决

为了在无 macOS 物理机的情况下完成原生 ARM64 构建，我们配置了基于 Apple Silicon 虚拟机的 GitHub Actions 流水线 ([`.github/workflows/main.yml`](.github/workflows/main.yml))。构建过程中排查并解决了以下关键兼容性问题：

### 1. 问题与修复列表
| 环节 | 报错现象 | 根本原因 | 解决方案 |
| :--- | :--- | :--- | :--- |
| **npm 全局包** | `ReadableStream is not defined` | npm 全局安装拉取了最新的 `@sentry/cli@3.8.0`（依赖 Node>=18），在 Node 16 下崩溃 | 移除无用的全局 npm 安装步骤，统一使用本地锁定版本 |
| **Node.js OpenSSL** | `--openssl-legacy-provider is not allowed in NODE_OPTIONS` | Node 16 自带 OpenSSL 1.1.1，该选项仅在 Node 17+ (OpenSSL 3.0) 中存在 | 移除 Node 16 下的 `--openssl-legacy-provider` 环境变量 |
| **npm 版本校验** | `Error: This project requires npm version >=6.x <=8.16.0` | GitHub Actions macOS 镜像自带 npm 8.19.4，被项目 `CheckYarn.js` 拦截 | 1. CI 中显式安装 `npm@8.16.0`<br>2. 优化 `CheckYarn.js`，在 CI 环境下免除拦截 |
| **node-gyp 编译** | `ModuleNotFoundError: No module named 'distutils'` | macOS-14 镜像自带 Python 3.12+，Python 3.12 彻底移除了标准库 `distutils` | 在 workflow 中引入 `actions/setup-python@v5` 锁定 Python 3.11 |
| **代码格式检查** | `Expected 1 empty line after require statement` | `yarn build` 内置触发 `yarn lint`，修改后的 `CheckYarn.js` 缺少规范空行 | 严格按照 ESLint 规范补齐格式空行，本地复测 0 errors |

---

## 四、安装包下载与 macOS 运行指南

### 1. 构建产物下载
- 构建运行记录：[GitHub Actions Run #35844439063](https://github.com/softguyus/openmtp/actions/runs/35844439063)
- 产物名称：`OpenMTP-mac-arm64`（体积约 232.20 MB）
- 适配架构：Apple Silicon（M1 / M2 / M3 / M4 芯片 MacBook Air / Pro / Mac mini / iMac 等）

### 2. 绕过 Gatekeeper 提示（“OpenMTP 已损坏，无法打开。你应该将它移到废纸篓”）
因开源版本未经付费 Apple 开发者证书签名与公证，macOS 会自动添加隔离属性。请按以下方法解决：

**终端一行命令解除隔离（推荐）**：
1. 将下载解压出的 `OpenMTP.app` 拖入 `/Applications`（应用程序目录）；
2. 打开 Mac 终端，执行：
   ```bash
   sudo xattr -rd com.apple.quarantine /Applications/OpenMTP.app
   ```
3. 输入 Mac 密码回车即可正常打开运行。
