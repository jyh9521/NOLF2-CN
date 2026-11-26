# 《无人永生 2》简体中文汉化

*No One Lives Forever 2: A Spy in H.A.R.M.'s Way* 社区简体中文汉化项目。发布包为 **汉化 + Modernizer 整合包**，通过运行时中文渲染提供界面、字幕、任务与情报文本的简体中文支持。

## 当前版本与状态

- **当前版本**：**v1.1**
- **适用环境**：**NOLF2 1.3 + Modernizer 2 Beta 2c**
- **整合说明**：发布包已内置上述版本的 Modernizer 核心，玩家只需准备纯净的 NOLF2 1.3，**无需另外安装 Modernizer**，也无需自行构建中文渲染组件（从源码重建时需另行准备对应组件）。
- **测试与覆盖**：已完成全流程通关测试，可稳定游玩；仍有少量漏译、贴图文字（图片文字）与个别滚动长帮助文本未覆盖。v1.1 统一了句尾标点并修订了部分措辞。

> [!NOTE]
> 根目录的 [NOLF2_汉化_发布声明.md](NOLF2_汉化_发布声明.md) 保留了 v1.0 首发时的项目背景与技术说明，可供了解项目来历。

## 下载

- **整合包直链**：**[下载 v1.1 整合包（v1.1.zip）](https://github.com/jyh9521/NOLF2-CN/releases/download/1.1/v1.1.zip)**
- **发布页**：[v1.1 Release 页面](https://github.com/jyh9521/NOLF2-CN/releases/tag/1.1)

> [!IMPORTANT]
> 玩家请直接下载发布页附件 **`v1.1.zip`**，不要下载 GitHub 自动生成的 `Source code` 源码压缩包。

整合包内包含 Modernizer 核心文件（`Lithtech.exe`、`SDL2.dll`、`Custom/Mods/Modernizer/MODERNIZER.REZ` 等）、`NOLF2_CN` 中文运行目录以及 `启动汉化版.bat`。本源码仓库仅包含代理源码、构建工具与译文数据，不包含游戏本体或重建所需的第三方 Modernizer 文件。

## 安装与使用

1. 准备一份纯净的 **NOLF2 1.3** 安装，按包内说明准备好运行库；**覆盖前建议先备份原文件**。
2. 解压 `v1.1.zip`，进入解压后的版本文件夹，将其中的**全部内容**复制到游戏根目录（与 `GAME.REZ`、`NOLF2.exe` 同级），按提示合并 `Custom` 等文件夹并覆盖同名文件。
   - 不要只复制 `NOLF2_CN` 目录；
   - 也不要把整个版本文件夹作为一层子目录丢进游戏根目录。
3. 双击游戏根目录下的 `启动汉化版.bat`，脚本会自动加载 Modernizer 与简体中文，**无需在启动器中手动选择模组**。

> [!WARNING]
> - **请勿叠加其他版本的 Modernizer**：本汉化的运行时代理仅适配 **Modernizer 2 Beta 2c**，保留整合包配套文件即可，自行替换版本可能导致显示异常或崩溃。
> - **请保持窗口模式启动**：默认以 **1280×720 窗口模式**启动，进入游戏后可在显示设置中调整分辨率。部分现代显卡上以独占全屏启动会直接崩溃，请保留窗口模式。
> - **如何卸载还原**：中文渲染本身不修改基础文本资源，但整合包会替换 `Lithtech.exe`、`SDL2.dll` 等 Modernizer 文件。仅删除 `NOLF2_CN` 目录并不等于完整恢复原版；如需彻底还原，请恢复覆盖前的备份或重新安装纯净游戏。

## 技术原理简介

本项目通过 CSHELL 代理转发指定版本的 Modernizer 客户端，并挂钩引擎的文字渲染入口 `CUIPolyString::Render`：渲染时按文本的英文原文匹配运行时中文对照表，命中后直接从字形图集（SimHei）采样、重新排版并绘制。

该方案避开了原引擎的单字节文本管线与 255 字形 UI 字体上限，不依赖系统 ANSI 代码页。对照表、字形图集和运行时代理各自独立构建；详细的逆向偏移、调试过程与被否决的早期方案见 [chinese_note.md](chinese_note.md)。

## 从源码重建

### 环境依赖

- Windows 系统
- NOLF2 1.3 与 Modernizer 2 Beta 2c
- MSVC x86 工具链
- Python 3 与 `Pillow`
- 本机字体 `simhei.ttf`
- 官方打包工具 `LithRez.exe`

### 构建与打包步骤

> [!TIP]
> 构建脚本会根据自身路径推导游戏根目录，请将本仓库的 `_work/` 目录放到游戏根目录下操作，不要在缺少游戏资源的源码目录中直接运行。

1. 从对应版本的 `MODERNIZER.REZ` 中提取并准备 `_work\extract_modernizer\CSHELL.DLL`。
2. 按本机 Visual Studio 环境检查 `_work\proxy_cshell_renderer\build_proxy.ps1` 中的工具链路径。
3. 在游戏根目录运行以下命令，生成图集、词典、代理 DLL 与 REZ 覆盖包并启动测试：

   ```powershell
   python -m pip install Pillow
   python _work\tools\build_phase1_renderer.py
   .\_work\run_phase1_renderer.bat
   ```

4. 确认显示正常后，先通过 Modernizer 启动器生成 `launchcmds.txt`，再运行打包脚本组装发布整合包：

   ```powershell
   python _work\tools\make_cn_release.py
   ```

打包产物会输出到游戏根目录下的 `NOLF2_简体中文汉化_v1.1.zip`。组装整合包需要本地已有指定版本 Modernizer 的完整运行文件，脚本不会自动下载它们。修改译文后需要重新生成词典；若引入了新字形，还须同步重建字形图集。

## 仓库结构与文档

| 路径 | 说明 |
| --- | --- |
| `_work/proxy_cshell_renderer/` | CSHELL 运行时代理源码、逆向常量与 MSVC 构建脚本 |
| `_work/tools/` | 文本提取、词典构建、字形图集、启动与整合包打包脚本 |
| `_work/translation/` | 英文到中文的 TSV 译文数据 |
| `chinese_note.md` | 技术调查、逆向偏移与完整开发记录 |
| `NOLF2_汉化_发布声明.md` | v1.0 发布背景、技术简介与已知问题 |

相关入口：

- [技术与开发记录](chinese_note.md)
- [v1.0 发布背景与已知问题](NOLF2_汉化_发布声明.md)
- [整合包打包脚本](_work/tools/make_cn_release.py)

## 已知问题

- 部分武器专有名、少数情报标题与个别超长滚动帮助文本仍可能显示英文。
- 烘焙到贴图中的纸质便签、信件等图片文字不由文本渲染器处理，仍保持英文。
- 仅适配 **Modernizer 2 Beta 2c**，自行替换其他 Modernizer 版本可能导致异常。

## 致谢与版权声明

感谢原作开发商 Monolith Productions 与 Modernizer 作者。游戏本体、Modernizer 补丁与 SimHei 字体均各自保留原有版权。本仓库目前未设置统一的项目许可证，源码公开并不代表相关第三方资源可自由再分发。
