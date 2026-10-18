# 《无人永生 2》简体中文汉化

No One Lives Forever 2: A Spy in H.A.R.M.'s Way 的社区简体中文汉化，基于 Modernizer 2，以运行时中文渲染提供界面、字幕、任务与情报文本。

## 支持版本与当前状态

当前发布版为 **v1.1**，适用于 **NOLF2 1.3 + Modernizer 2 Beta 2c**。发布包已内置该 Modernizer 核心；源码重建则需另行准备对应组件。

已完整通关测试，仍有少量漏译、图片文字与个别长帮助文本未覆盖。v1.1 统一句尾标点并修订部分措辞；根目录的[发布声明](NOLF2_汉化_发布声明.md)保留 v1.0 背景与技术说明。

## 下载

[v1.1 发布页](https://github.com/jyh9521/NOLF2-CN/releases/tag/1.1)，附件为 `v1.1.zip`。源码仓库包含代理、工具与译文，不包含完整游戏或重建所需的第三方 Modernizer 文件。

## 安装 / 使用

1. 准备纯净的 NOLF2 1.3 安装，按包内说明准备运行库；覆盖前备份原文件。
2. 将汉化包全部内容复制到游戏根目录，与 `GAME.REZ`、`NOLF2.exe` 同级，合并并覆盖。
3. 双击 `启动汉化版.bat`，自动加载 Modernizer 与中文。

默认以 **1280×720 窗口模式**启动，可在游戏内调整分辨率。部分现代显卡上独占全屏启动会崩溃，请保留窗口模式。

中文渲染不改基础文本资源，但整合包会替换 `Lithtech.exe`、`SDL2.dll` 等 Modernizer 文件。只删除 `NOLF2_CN` 目录并不等于完整恢复原版；恢复时还原覆盖前备份，或重新安装干净游戏。

## 技术摘要

CSHELL 代理转发指定版本的 Modernizer 客户端，并挂钩 `CUIPolyString::Render`；按英文原文匹配中文对照表，使用字形图集重新排版与绘制。

该路径避免把中文塞入原引擎的单字节文本与 255 字形界面字体，不依赖系统 ANSI 代码页。对照表、字体图集和运行时代理分别构建；具体偏移与否决路线见 [chinese_note.md](chinese_note.md)。

## 从源码重建

需要 Windows、NOLF2 1.3 与 Modernizer 2 Beta 2c、MSVC x86、Python 3 / Pillow、本机 `simhei.ttf` 和 `LithRez.exe`。

脚本按自身路径推导游戏根目录，因此请把仓库的 `_work/` 放在游戏根目录下操作，不要在缺少游戏资源的源码目录直接运行：

1. 从对应 `MODERNIZER.REZ` 准备 `_work\extract_modernizer\CSHELL.DLL`。
2. 按本机 VS 环境检查 `_work\proxy_cshell_renderer\build_proxy.ps1` 的工具链路径。
3. 在游戏根目录运行：

   ```powershell
   python -m pip install Pillow
   python _work\tools\build_phase1_renderer.py
   .\_work\run_phase1_renderer.bat
   ```

4. 生成图集、词典、代理与 REZ 后测试显示。组装整合包前，先通过 Modernizer 启动器生成 `launchcmds.txt`，再运行：

   ```powershell
   python _work\tools\make_cn_release.py
   ```

组装输出为游戏根目录下的 `NOLF2_简体中文汉化_v1.1.zip`。打包还需要指定 Modernizer 的完整运行文件；工具不自动下载它们。修改译文后须同步词典；引入新字形时还须重建图集。

## 仓库结构与文档

| 路径 | 内容 |
| --- | --- |
| `_work/proxy_cshell_renderer/` | 代理源码、逆向常量与 MSVC 构建脚本 |
| `_work/tools/` | 文本提取、词典、图集、启动与打包工具 |
| `_work/translation/` | 英文到中文的 TSV 译文 |
| `chinese_note.md` | 技术调查与完整开发记录 |

- [技术与重建记录](chinese_note.md)
- [v1.0 发布背景与已知问题](NOLF2_汉化_发布声明.md)
- [当前整合包生成脚本](_work/tools/make_cn_release.py)

## 已知问题与来源

部分武器名、情报标题与超长帮助文本仍可能显示英文；烘焙到贴图里的便签 / 信件不由文本渲染器处理。仅适配指定 Modernizer 版本，自行替换版本可能导致异常。

感谢原作 Monolith Productions 与 Modernizer 作者。游戏、Modernizer 和 SimHei 字体各自保留原有版权；本仓库当前没有统一项目许可证，源码公开不代表相关第三方资源可自由再分发。
