# NOLF2 简体中文汉化开发与技术记录

> [!NOTE]
> 本文档按开发阶段记录了《无人永生 2》简体中文汉化的完整逆向、试错与定型过程：
> - **前半部分（2026-07-12 至 2026-07-14）**：当前活跃的**运行时自绘中文渲染器**架构、Modernizer `CSHELL.DLL` 逆向偏移与 vtable 表、打磨记录，以及 v1.0 / v1.1 整合包发布方案。
> - **后半部分（2026-07-10 至 2026-07-12，第 1–24 轮）**：早期探索与已弃用的 `CRES.DLL` / 载体字节分页字体路线归档，保留作技术参考。

## 项目要求

- 最终目标：实现尽可能完整、稳定、可加载的简体中文汉化。
- 所有汉化相关工作文件放在游戏根目录的 `_work` 文件夹内。
- 每次修改均记录：修改内容、采用技术、遇到的问题、验证结果、下一步。
- 新步骤确认无误后，旧步骤产生的无用文件归档到 `_work/archive`，不直接删除。

## 2026-07-13 回退到稳定可玩版（放弃窗口化全屏，撤销双钩）

### 经过

- 先尝试修复窗口化全屏：`run_phase1_renderer.bat` 的分辨率检测失效（`^|` 在双引号 `-Command` 内非法 → `+ScreenWidth/Height` 为空 → 游戏回退 `DISPLAY.CFG` 的 640×480 小窗）。改为 `for /f` + `.Where()` + `-join [char]32` 稳健检测；代理里也曾尝试加入 `BorderlessThread` 去边框铺满。
- 实测发现：**继续游戏、载入完成后闪退。** 因此决定放弃窗口化全屏，先回到稳定可玩版本，专注汉化推进。

### 崩溃根因（关键）

`runtime.log` 显示当前代理处于「双钩」模式：`patched CreatePolyString + CreateFormattedPolyString` + `patched plain CUIPolyString::Render` + `patched CUIFont::DrawString`。崩溃现象「载入完成后闪退」与早前「打磨 8 回退」记录的完全一致 —— **挂钩 `CreatePolyString`（`0x38`）会在载入结束时闪退**。此前已定为「只钩格式 poly 的稳定版」，本次重新打开双钩后复现了崩溃，窗口/分辨率并非主因。

### 修改内容（回退）

- `_work/proxy_cshell_renderer/cshell_proxy_renderer.cpp`：
  - **移除无边框全屏逻辑**（`BorderlessThread` / `EnumProc` / `CreateThread`）。载入时引擎会重置 D3D 设备，代理再调用 `SetWindowLong` / `SetWindowPos` 抢占窗口会触发崩溃。回退后代理完全不干预窗口（确认清理干净）。
  - **撤销双钩，回到只钩 `CreateFormattedPolyString` 的稳定版**：`TryPatchFontManager` 不再 patch `VF_CreatePoly`；`CnCreateFormattedHook` 不再调用 `PatchFontDrawString`（仅作诊断探针，无翻译作用）。运行时仅安装：格式 poly 创建钩 + 对应的 `Render` 钩。`CnCreatePolyHook` / `CnDrawStringHook` / `PatchFontDrawString` 保留为死代码（永不安装），不影响运行。
  - 日志标记改为 `patched CreateFormattedPolyString (formatted-only stable)`。
  - 代价：仅滚动长帮助文本（ID `11016`）保持英文，可接受。

### 分辨率（保留安全窗口配置）

- `run_phase1_renderer.bat` 调用 `_work/tools/select_safe_window_resolution.ps1` 选择「安全」窗口分辨率（上限 1920×1200，避开已知会崩溃的高模式如 2800×1800、2500×1406），兜底 1280×720。
- 该脚本支持通过环境变量 `NOLF2_CN_WIDTH` / `NOLF2_CN_HEIGHT`，或在游戏根目录放置 `NOLF2_CN_Window.cfg`（`width=...` / `height=...`）进行覆盖。若重建后仍崩，优先把窗口分辨率调低测试。

### Windows 重建与验证步骤

1. 重建（重新编译代理）：
   ```powershell
   python _work\tools\build_phase1_renderer.py
   ```
2. 启动测试：
   ```powershell
   .\_work\run_phase1_renderer.bat
   ```
3. 预期 `runtime.log` 只出现 `patched CreateFormattedPolyString (formatted-only stable)` 与 `patched formatted CUIPolyString::Render`，**不再出现** `plain` / `CreatePolyString +` / `DrawString` 行。继续游戏载入后不再闪退，汉化显示正常。
4. 若仍崩溃，则说明另有主因（优先怀疑窗口分辨率），可通过 `NOLF2_CN_Window.cfg` 降至 1280×720 或更低复测，并查看 `runtime.log` 崩溃前最后一行。

### 结果与真正根因（已确认，2026-07-13）

- 中途曾出现误判：低分辨率启动能玩、高分辨率载入崩，一度怀疑是“引擎无法在高分辨率下载入关卡”，甚至在“原版”上也复现过崩溃 —— **但那个“原版”实际上是被早期实验污染过的安装**（`Game\CSHELL.DLL` 被早期 90 KB 代理覆盖等），并不纯净，因此“引擎老毛病”的结论是**错误的**。
- **纯净重装**（原版 1.3 + Modernizer 2 Beta 2c + Beta 2c release2 mp update）后实测：
  - 原版进游戏调 4K、载入关卡，**完全正常**。
  - 装回渲染器汉化（REZ 覆盖 + 代理），进游戏调 4K、重新开始，**同样完全正常**。
- **真正根因：文件损坏**（早期实验把 `Game\CSHELL.DLL` 改坏），与引擎、汉化、分辨率**均无关**。当前渲染器汉化不修改基础文本文件（仅通过 REZ 覆盖 + 新建 `CSHELL_MODERNIZER_ORIG.DLL` 供代理加载），不会再造成污染。
- 代理 Modernizer 版本核对：`_work/extract_modernizer/CSHELL.DLL`（2.25 MB）的代码切片与重装后的 `MODERNIZER.REZ` 逐字节吻合，硬编码偏移继续有效。
- **启动方式改回默认 + 修复“每次重置设置”根因**：
  - 现象：去掉分辨率强制后，游戏仍每次以窗口 1024×768 启动，游戏内设置的全屏/分辨率无法保存。
  - 根因：`NOLF2.exe` 启动器的 **Advanced Options → “Restore Defaults”** 复选框曾被勾选，往 `launchcmds.txt` 写入了 `+RestoreDefaults 1`。源码 `ProfileMgr.cpp:1565` 中 `if (GetConsoleInt("RestoreDefaults",0)) m_profile.RestoreDefaults(PROFILE_ALL)` 会导致**每次启动都将 profile 重置回出厂默认**（窗口 1024×768、默认按键/音效），清掉玩家在 `autoexec.cfg` 中的设置（`Windowed 0` / 3840×2160）。
  - 第一次修复尝试（无效）：仅剥离该 flag、不附加 display 参数，寄希望于引擎读取 `autoexec.cfg` 自行决定窗口/全屏 —— **结果仍窗口化启动**。
  - **纠正**：LithTech 引擎在**创建初始渲染窗口**时，窗口/全屏与分辨率只看**命令行** `+Windowed` / `+ScreenWidth` / `+ScreenHeight`，**不读 `autoexec.cfg`**（`autoexec.cfg` 的 `Windowed` 仅被游戏 DLL 用于选项菜单）。命令行缺省时引擎默认小窗。
  - 尝试全屏启动（失败）：新增 `_work/tools/build_launch_cmdfile.ps1`，从 `autoexec.cfg` 读取设置并显式拼接 `+Windowed 0 +ScreenWidth 3840 +ScreenHeight 2160`。**实测 `+Windowed 0`（独占全屏）在启动时直接闪退**，印证了“独占全屏在部分现代 GPU 上启动崩溃”的限制。
  - **当前落地（安全窗口模式）**：`build_launch_cmdfile.ps1` 改为：① 剥离 `+RestoreDefaults` 及残留 display flag（避免每次重置按键与设置）；② 固定附加 `+Windowed 1 +ScreenWidth 1280 +ScreenHeight 720 +BitDepth 32` + 汉化 REZ。窗口 720p 稳定启动、中文正常，进入游戏后再自行调整分辨率。`run_phase1_renderer.bat`（重写以清除此前写入残留的 NUL 尾字节）与 `build_phase1_renderer.py` 均调用该脚本。
  - 全屏诉求暂缓：启动期独占全屏会崩溃；日后如需全屏，应走“无边框窗口”方向（早期 borderless helper 崩溃发生在污染安装期，纯净版可另行验证），而非 `+Windowed 0`。
  - `select_safe_window_resolution.ps1` / `NOLF2_CN_Window.cfg` 不再被调用（保留备用）。
- 工作文件在重装期间整体备份到根目录 `NOLF2_CN_WORK_BACKUP_20260713/`（含 README 与 `game_dll_backups/`），重装后已将 `_work`、`chinese_note.md` 迁回根目录，备份文件夹保留作安全副本。

### 归档清理（2026-07-13）

确认自绘渲染器路线稳定可玩后，将早期载体 / CRES / 逐屏探针路线的历史产物统一归档到 `_work/archive/20260713_pre_ch2_cleanup/`（未删除）：

- `builds/`：26 个旧构建输出目录（smoke、encoded、menu_stage、各设置页探针、subtitle 载体、`build_full_import_modernizer` 等）。
- `proxies/`：`proxy_cshell`（早期代理）、`proxy_cshell_modernizer`（载体方案，已弃用）、`borderless`（独立无边框 helper EXE，已弃用）、`extract_lfp`。
- `runbats/`：27 个旧 `run_*.bat`（保留 `run_phase1_renderer.bat`）。
- `tools/`：23 个载体 / CRES / 探针构建脚本（各 `*_pack.py`、`build_encoded_font_smoke.py`、`build_full_import_modernizer.py`、`build_menu_stage1_encoded.py`、`build_cres_resdll.ps1`、`patch_pe_stringtable.py` 等）。
- `misc/`：`cn_all_strings_pages.tsv`、`cn_subtitles_ascii.tsv`、`borderless_runtime.log`、`patch/`（CRES 翻译模板）。

**当前活跃管线（保留）**：`proxy_cshell_renderer/`、`build_cn_renderer/`（图集 DTX / MET）、`build_phase1_renderer/`（输出 REZ）、`extract_modernizer/CSHELL.DLL`、`translation/`（词表源）、`run_phase1_renderer.bat`；工具保留 `build_phase1_renderer.py`、`build_cn_glyph_atlas.py`、`build_cn_dict.py`、`select_safe_window_resolution.ps1`，以及翻译提取脚本 `extract_translation_tables.py`、`extract_remaining_after_chapter01.py`、`extract_pe_stringtable.py`。`analysis/`、`source/`、`backup_before_modernizer_*` 作为参考保留。

### 开源备份 + 代码清理（2026-07-14）

- 整理源码仓库 `NOLF2_CN_source_repo/`（及同名 zip）用于 GitHub 备份与后续开发：按 `_work/` 布局收录代理源码、全部 `.py` / `.ps1` 工具、`translation/*.tsv`、`chinese_note.md`、发布声明、`README.md` 与 `.gitignore`。**排除**游戏本体、Modernizer 文件、构建生成物、`_work/source`（官方源码）、`archive`、`analysis` 等（出于版权、体积与可重生成考虑）。
- README 写明构建前置：NOLF2 1.3 + Modernizer 2 Beta 2c、MSVC（x86）、Python + Pillow、SimHei、LithRez，以及需自备 `_work/extract_modernizer/CSHELL.DLL`（Modernizer 原始 CSHELL，因第三方版权不入库）。
- 代码卫生修复：`cshell_proxy_renderer.cpp` 末尾存在 98 个 NUL 尾字节（此前工具写入残留，位于最后一个 `}` 之后，编译器未报错），已 `rstrip` 清除（30213 → 30115 字节），工作区与仓库副本均已清理干净。
- 版权提示：`translation/*.tsv` 含游戏英文原文，备份时请留意相关版权风险。

### 发布包 v1.1：`NOLF2_CN` 结构 + 集成 Modernizer（2026-07-14）

改进目标：① 发布包使用干净的 `NOLF2_CN\` 目录（不再带 `_work` 残留）；② 集成 Modernizer 2 核心（玩家只需纯净版 1.3）。

- **修改代理硬编码路径**（`cshell_proxy_renderer.cpp`，需重编）：MET / STRINGS / `runtime.log` / `CSHELL_ORIG` 四处路径从 `_work\...`、`Game\` 迁至 `<root>\NOLF2_CN\`。DTX 仍通过 REZ 加载（`Interface\Fonts\NOLF2CN_ATLAS.DTX`），保持不变。
- **构建脚本**（`build_phase1_renderer.py`）：`stage()` 额外生成 `<root>\NOLF2_CN\{NOLF2CN_ATLAS.MET, NOLF2CN_STRINGS.bin, CSHELL_MODERNIZER_ORIG.DLL}`；开发启动脚本移除 `Game\` 复制步骤，日志路径改为 `NOLF2_CN\runtime.log`。
- **Modernizer 核心文件识别**（按日期 2020+）：`Lithtech.exe`、`SDL2.dll`、`NOLF2Srv.exe`、`osslicenses.txt`（2020-12-24）、`Update_v1x3_GUI.REZ`（2021）、`ModernizerReadme.txt`、`JServerDir.dll`、`JServerInfo.txt`（2026 mp update）、`Custom\Mods\Modernizer\MODERNIZER.REZ`。**不含** 4.4 GB Upscale 高清包。基础大 REZ（`GAME*.REZ` 等 2002–2003 文件）来自玩家纯净版 1.3，不纳入打包。
- **集成打包器**（`_work/tools/make_cn_release.py`，Windows 运行）：组装 Modernizer 核心 + `NOLF2_CN\`（REZ / MET / STRINGS / `CSHELL_ORIG`）+ 预生成 `launchcmds_cn.txt`（从 `launchcmds.txt` 截去 `+RestoreDefaults` 起的默认段，保留 rez / mod / gameplay，追加窗口 1280×720 与汉化 REZ，玩家无需再跑 `NOLF2.exe` 选模组）+ `启动汉化版.bat` + `安装说明.txt` → 输出整合包。启动行静态验证通过：`+mod Modernizer` 存在、无 `+RestoreDefaults` / `+SetPerformanceLevel`、窗口 720p + 汉化 REZ。
- **发布目录结构**（覆盖纯净版 1.3）：游戏根目录放 Modernizer 核心 exe / dll / rez、`启动汉化版.bat`、`安装说明.txt`、`Custom\Mods\Modernizer\` 以及 `NOLF2_CN\`（全部汉化文件收拢于此）。
- 构建与出包流程（Windows）：
  ```powershell
  python _work\tools\build_phase1_renderer.py
  .\_work\run_phase1_renderer.bat
  python _work\tools\make_cn_release.py
  ```

### 发布包 v1.0（2026-07-13）

- 使用 maphole 与通关存档**从头测到尾，除少量漏译外无问题**，决定发布 v1.0。
- 打包 `NOLF2_简体中文汉化_v1.0.zip`（游戏根目录与输出目录各一份），供覆盖纯净版测试。
- 内容为当前已测二进制原样打包（**未重编**），包含代理运行时依赖的全部文件：
  - `NOLF2_CN_PHASE1.rez`（代理 CSHELL + 图集 DTX）
  - `Game\CSHELL_MODERNIZER_ORIG.DLL`（代理 `LoadLibrary` 目标）
  - `_work\build_cn_renderer\stage\Interface\Fonts\NOLF2CN_ATLAS.MET` 和 `NOLF2CN_STRINGS.bin`（代理硬编码路径读取）
  - `启动汉化版.bat` + `build_launch_cmdfile.ps1`（剥离 `+RestoreDefaults`、窗口 1280×720、追加 REZ）+ `安装说明.txt`
- v1.0 已知遗留：① 代理从 `_work\...` 硬编码路径读取 MET / BIN，导致发布包必须带 `_work` 子目录（已在 v1.1 迁至 `NOLF2_CN\`）；② 少量漏译；③ 不支持独占全屏启动。

### 进度 + Steam 启动说明（2026-07-13）

- **进度**：全量文本已导入，实测**游玩至第四章无问题**，汉化本体稳定。
- **Steam 启动（方便截图）**：不要直接把 `run_phase1_renderer.bat` 添加进 Steam —— 批处理末尾的 `start "" Lithtech.exe` 会将游戏甩成独立进程，导致 Steam 钩不住、覆盖层 / F12 截图失效并显示“已停止”。
  - 正确做法：把 **`Lithtech.exe` 本体**添加为非 Steam 游戏，启动选项设为 `-cmdfile "C:\Games\No One Lives Forever 2\_work\build_phase1_renderer\launchcmds_phase1.txt"`，起始位置设为游戏根目录。
  - 前置：`CSHELL_MODERNIZER_ORIG.DLL` 已就位（静态）、cmdfile 已生成；仅当基础 `launchcmds.txt` 变动时才需重跑批处理重新生成 cmdfile。
  - 仍保持窗口模式（独占全屏启动会崩），窗口模式下 Steam 截图正常。

## 2026-07-12 全量 Modernizer 导入包

### 修改内容

- 新增 Modernizer 专用 CSHELL 代理：
  - `_work/proxy_cshell_modernizer/cshell_proxy_modernizer.cpp`
  - `_work/proxy_cshell_modernizer/build_proxy.ps1`
- 新增全量构建脚本：
  - `_work/tools/build_full_import_modernizer.py`
- 生成全量文本分页表：
  - `_work/cn_all_strings_pages.tsv`
- 生成全量测试包：
  - `_work/build_full_import_modernizer/NOLF2_CN_FULL_IMPORT_MODERNIZER.rez`
- 生成启动脚本：
  - `_work/run_cn_full_modernizer.bat`
- 生成分析记录：
  - `_work/analysis/full_import_modernizer.md`

### 采用的技术

- 合并 `_work/translation/chapter01_priority.tsv` 和 `_work/translation/remaining_after_chapter01.tsv`，以 TSV 译文为准生成 6355 个唯一字符串 ID。
- 使用外部十六进制文本表绕开 Windows ACP 转换，避免中文在 `LoadStringA`/CP932 路径中变成问号或乱码。
- 使用 `0x80-0xFF` 高字节 carrier 加未占用 ASCII carrier 做分页字体映射。
- 为 5389 条含中文的文本生成 194 个 SimHei 子集字体页，布局字体索引为 `FontFace5` 到 `FontFace198`，未超过 255。
- 代理 DLL 加载 `Game/CSHELL_MODERNIZER_ORIG.DLL`，全局替换 Modernizer `LoadTempString`，并 hook CUI formatted text 创建和 `SetText`，在渲染前切换到对应分页字体。
- 同时从生成的 `ClientRes.cn_full.rc` 重编 `CRES.DLL`，让直接 `LoadString` 的代码路径也尽可能拿到译文；`IDS_GAME_LANGUAGE` 保持英文。

### 验证结果

- `_work/cn_all_strings_pages.tsv` 静态核验：
  - entries: 6355
  - unique IDs: 6355
  - min ID: 100
  - max ID: 33600
  - paged entries: 5389
  - max page: 193
  - max font index: 198
- `_work/proxy_cshell_modernizer/bin/CSHELL.DLL` 编译成功，并导出 `SetMasterDatabase`。
- `_work/build_full_import_modernizer/NOLF2_CN_FULL_IMPORT_MODERNIZER.rez` 打包成功，大小约 10.3 MB。
- 已复制 Modernizer 原始 `CSHELL.DLL` 到 `Game/CSHELL_MODERNIZER_ORIG.DLL`，供代理运行时加载。

### 下一步

- 用 `_work/run_cn_full_modernizer.bat` 进入游戏测试开头载入页、菜单、任务页和对白。
- 如果某个界面仍走直接 `LoadString` 并出现乱码或缺字，再针对该路径补 hook。

### 追加修正：弃用预加载 194 字体页

- 用户实测首版全量包启动后无响应。
- 日志显示代理已加载、`LoadTempString` 已 patch、文本表已读入，卡死点不在文本表解析。
- 判断首版路线不稳：把 194 个中文字体页全部注册进 `LAYOUT.TXT` 会导致启动期一次性初始化大量 CUI 字体，且全局 CUI 文本 hook 风险过高。
- 已改为按需字体方案：
  - `LAYOUT.TXT` 恢复 Modernizer 原版字体段，不再注册 `NOLF2CN_PAGE_*`。
  - 194 个 `NOLF2CN_PAGE_*.TTF` 仍打包进 REZ。
  - CSHELL 代理在某页文本实际出现时调用 `ILTFontManager::CreateFont("Interface\\Fonts\\NOLF2CN_PAGE_xxx.TTF", "SimHei", 60, 33, 255)` 动态创建并缓存字体。
  - 修复字幕 call site：原来误用 JMP 替换 CALL，已改为相对 CALL。
- 新版静态核验：
  - `_work/build_full_import_modernizer/stage/ATTRIBUTES/LAYOUT.TXT` 不含 `NOLF2CN_PAGE` 或新增中文 `FontFace`。
  - 字体页数量仍为 194。
  - REZ 已重建：`_work/build_full_import_modernizer/NOLF2_CN_FULL_IMPORT_MODERNIZER.rez`。

### 运行反馈修复：缺字漏字 + 标题乱码 + 首次点击卡顿

用户实测按需字体全量包：游戏能进，但很多选项缺字漏字（如显示设置“垂直同步”显示成“垂直步”、“抗锯齿”显示成“抗”），二级菜单标题显示成 ASCII 乱码（如单人子菜单标题显示 `=("!`），且每个按钮首次点击进二级菜单会卡顿一下。

#### 根因定位

- 缺字漏字（主要问题）：
  - `build_full_import_modernizer.py` 里 `HIGHBYTE_CARRIERS = list(range(0x80, 0x100))`，把 `0x80-0xFF` 全部当中文载体。
  - 但当前系统 ACP 为 `932`，`0x81-0x9F` 和 `0xE0-0xFC` 是 Shift-JIS 前导字节（lead byte）。任何 MBCS 文本路径都会把“前导字节 + 后一字节”合并成一个双字节字符，于是相邻两个载体被吞成一个，丢字。
  - 证据：`analysis/full_import_modernizer_char_pages.csv` 中缺失的字确实落在前导区，例如 `同=0x9E/0x8B/0x98`、`直=0xE0/0xE4`。
  - 之前所有逐屏包都只用 `0xA1-0xDF`（CP932 单字节半角区），所以从不丢字。全量脚本扩大到 `0x80-0xFF` 才引入这个问题。
- 二级菜单标题乱码：
  - 标题渲染走的是 pointer 匹配失败的路径。代理靠 `CnCreateFormattedPolyStringHook` 用 buffer 指针匹配 `LoadTempString` 返回槽来选分页字体。
  - 菜单正文项把返回指针直接传给 CreateFormattedPolyString，指针命中，正常显示中文。
  - 但标题控件（和主菜单高亮项）会先把字符串拷进自己的缓冲区再建 poly string，指针不再相等，命中失败，回退默认字体，于是把载体字节按默认字体画出来。
  - 证据：id 579“单人游戏”编码为 `3d 28 22 21` = `=("!`，正是截图里单人子菜单标题显示的乱码。
- 首次点击卡顿：
  - `GetPageFont` 首次用到某页时才 `CreateFont` 运行时栅格化该子集字体，同步发生在主线程；结果缓存在 `g_page_fonts`，每页每次会话只卡一次。

#### 修改内容

- `_work/tools/build_full_import_modernizer.py`：
  - `HIGHBYTE_CARRIERS` 从 `range(0x80, 0x100)` 改为 `range(0xA1, 0xE0)`（即 `0xA1-0xDF`，CP932 保证单字节），彻底避免前导字节吞字。ASCII 载体 `0x21-0x7E` 保留。
- `_work/proxy_cshell_modernizer/cshell_proxy_modernizer.cpp`：
  - 新增 `FindReturnPageByContent`：pointer 匹配失败时，按“最近优先”对返回槽做精确内容比较（`lstrcmpA`），命中则套用该页字体。载体字节组合对普通英文极不可能误命中，且只在 `page>=0` 的槽里找，不影响未翻译英文。
  - `CnCreateFormattedPolyStringHook` 在 pointer 匹配失败后追加调用内容匹配，修复标题/高亮项拷贝缓冲区导致的乱码。
  - 字体页缓存数组从固定 `256` 改为常量 `kMaxFontPages = 1024`，并把 `page >= 256` 上限判断改为 `page >= kMaxFontPages`。原因见下：载体收窄后页数升到约 295，超过 256 的页在旧代码里拿不到字体会整屏乱码。
  - `CreateFont` 栅格化范围从 `33..255` 收窄到 `33..0xDF`（载体最高只到 `0xDF`），小幅减少每页栅格化开销，缓解首次卡顿。

#### 静态验证（Linux 沙箱内，未编译）

- 用两张 TSV 复算新载体方案的分页：
  - 含汉字的译文行：5390。
  - 单条译文最多不同汉字：146，低于单页容量约 157（`(94 ASCII − 保护) + 63 高位`），因此 **oversized = 0**，没有任何译文因收窄载体而退回英文。
  - 估算字体页：约 295（旧的 `0x80-0xFF` 版是 194）。已据此把代理页数组扩到 1024。
- 本环境是 Linux 沙箱，无法运行 Windows 的 `build_proxy.ps1` / `build_cres_resdll.ps1` / `LithRez.exe`，因此代理 DLL、CRES、REZ 未在本轮实际重建，需用户在 Windows 上重跑构建。

#### 下一步（需用户在 Windows 执行）

1. 重建全量包（会重新编译代理、CRES、生成字体页和 REZ）：
   - `python _work\tools\build_full_import_modernizer.py`
2. 运行：
   - `_work\run_cn_full_modernizer.bat`
3. 重点复检：
   - 显示/声音/控制/性能等设置页是否还缺字（预期已修复，如“垂直同步”“抗锯齿”完整）。
   - 各二级菜单标题是否从乱码变成中文（预期内容匹配已修复；若仍乱码，说明标题走的是 poly string 的 SetText 更新路径而非创建路径，下一轮需再加 SetText 钩子）。
   - 首次进二级菜单的卡顿是否可接受（页数变多，单屏可能触及更多页；如仍明显，下一轮可按“同屏字符串聚到同页”优化打包以减少每屏字体创建次数）。

## 2026-07-12 转向：放弃载体方案，改做自绘中文渲染器

### 崩溃定位（上一版重建后闪退）

- 按上述改动重建后，游戏一闪即退。
- 根因：载体收窄到 `0xA1-0xDF` 后，大量字符串改用 ASCII 载体，结果有 1735 条含 `%`、1759 条含 `&`。
- `CUIFontManager::CreateFormattedPolyString` 是 printf 风格格式化函数；载体 `%` 后跟一个字节 → 非法格式符 → 读取不存在的可变参数 → 开局第一条此类字符串就触发崩溃（例如 ID `589` 被编码成 `%-`）。
- 旧的高位版依赖 `0x80-0xFF`，开局串刚好避开了 `%`，因此没有崩溃、仅表现为缺字。

### 为什么“载体字节 + 位图字体”路线对简体中文受众不可行

- 受众几乎全部使用简体中文系统（ACP = `936`）。CP936 中 `0x81-0xFE` 几乎全是 GBK 双字节前导字节，任何高位载体都会被 MBCS 合并吞字，唯一能安全穿过的只有 ASCII（`0x00-0x7F`）。
- 但 ASCII 载体存在硬伤：
  - 与格式符/标记冲突（`%` `&` `\` `"` 必须排除，只剩约 90 个可用载体）。
  - 单页最多容纳约 90 个不同汉字；全量 5390 条译文中有 114 条长文本（94 条长情报 + 14 条 mission + 6 条对白）单条不同汉字数超过 90，单个位图字体装不下。
  - 需要约 458 个动态字体页，卡顿更重且稳定性存疑。
- 结论：载体方案只能做到“短文本可玩”，天生装不下长情报文档且依赖系统区域设置。**决定彻底放弃，改做自绘渲染器。**
- 说明：`build_full_import_modernizer.py` 与 `cshell_proxy_modernizer.cpp` 的载体方案自此弃用（不再重建），仅保留作历史参考。

### 新路线：CSHELL 代理内自绘中文（locale 无关、无分页、无载体）

核心思路：不再把中文塞进 `uint8` 位图字体，而是在代理中利用引擎自身的 2D 绘制机制，从字形图集纹理采样并绘制中文。中文文本在代理内查表取得真实 Unicode，彻底绕开 `char*` 经 MBCS 转换被破坏的问题。

已从官方源码确认引擎具备所需全部能力：

- `g_pLTClient`（`ILTClient`）→ `GetDrawPrim()` 获取 `ILTDrawPrim`，`GetTexInterface()` 获取 `ILTTexInterface`。
- `ILTTexInterface::CreateTextureFromName(hTex, "Interface\\Fonts\\NOLF2CN_ATLAS.DTX")` 从 DTX 加载图集纹理（或使用 `CreateTextureFromData` 从内存像素创建）。
- `ILTDrawPrim`：`SetTransformType(SCREEN)` → `SetColorOp(MODULATE)` → `SetAlphaBlendMode(BLEND_MOD_SRCALPHA)` → `SetTexture(atlas)` → `DrawPrim(LT_POLYGT4*, n)`。源码 `interfacebutton.cpp` 等处已有现成范例。
- `CUIPolyString` 是桥接类，`Render()`、`GetText()`、`GetPolys()`（返回 `LT_POLYGT4*`，每字一个带 UV 的四边形）、`GetPosition()`、`GetCharScreenHeight()`、`GetLength()` 均为虚函数，可挂钩 `Render` 并读取坐标、尺寸与颜色。
- 图集采用白色 RGB + 逐像素 alpha 覆盖，通过 `MODULATE` 配合原顶点色即可自然还原任意文字颜色。

### Phase 0 已完成：字形图集生成器

- 新增 `_work/tools/build_cn_glyph_atlas.py`。
- 汇总两张 TSV 中所有非 ASCII 字形（2345 个），在 Windows 下使用 `SimHei` 渲染进图集（离线 Linux 环境下用 Noto CJK 验证逻辑，字形清晰无误）。
- 输出位置（`_work/build_cn_renderer/stage/Interface/Fonts/`）：
  - `NOLF2CN_ATLAS.DTX`：2048×2048 未压缩 BGRA8888，cell = 40 px，容量 2601 > 2345。
  - `NOLF2CN_ATLAS.MET`：二进制度量表（magic `CNMA` + 每字 `codepoint, u0, v0, u1, v1, advance`），供代理运行时查 UV。
  - `NOLF2CN_ATLAS_preview.png`：图集预览图。
- 在 Windows 上运行此脚本即可用本机 `simhei.ttf` 生成正式图集。
- 依赖：仅需 `Pillow`（自带 FreeType）；已移除原先的 `freetype-py` / `numpy` 依赖，避免 Windows 环境缺包。若报 `No module named PIL`，执行 `pip install Pillow` 即可。

### 分阶段里程碑

1. **Phase 1（最高风险，先验证）**：在代理中定位 `g_pLTClient` 全局 RVA，用 `CreateTextureFromName` 加载图集，验证自绘链路可行性。
2. **Phase 2**：挂钩 `CUIPolyString::Render`（定位 vtable / impl），命中中文串时按 `GetPosition` / `GetCharScreenHeight` / 顶点色使用图集渲染整串，英文串走原路径。
3. **Phase 3**：字幕路径（`CHUDSubtitles`）接入同一自绘器，处理换行、居中与自动换行布局。
4. **Phase 4**：全量文本（含 114 条长情报）统一接入自绘，无分页无载体。

## 2026-07-12 Phase 1 逆向：CSHELL 接口全局与 vtable 偏移（已核验）

对 `_work/extract_modernizer/CSHELL.DLL`（image base `0x10000000`）进行静态反汇编（`objdump`），定位自绘渲染所需的全部符号，均经多点交叉验证。

### 接口全局指针（相对 CSHELL image base 的 RVA）

从 `define_holder` 生成的 `CAPIHolder` 构造代码中（`movl $0, <addr>` 初始化 `ptr_var`）提取：

| 符号 | 类型 | RVA |
| --- | --- | --- |
| `g_pDrawPrim` | `ILTDrawPrim*` | `0x20E8F8` |
| `g_pFontMgr`（已有） | `ILTFontManager*` | `0x20E8FC` |
| `g_pTexInterface` | `ILTTexInterface*` | `0x20E900` |
| `g_pLTClient` | `ILTClient*` | `0x20E904` |

四者在 `.data` 段中连续排列；这些指针在模块 DB 解析完成后（即 `SetMasterDatabase` 之后）才被填充，与 `g_pFontMgr` 同一时机可用。

### `ILTDrawPrim` vtable 偏移（`IBase` 占 1 槽，已核验）

核验方式：统计全二进制对 `g_pDrawPrim` 的调用偏移分布，渲染状态组每项各约 24 次（每次 UI 绘制调用一次），与头文件顺序完全吻合。

| 方法 | vtable 偏移 | 备注 |
| --- | --- | --- |
| `BeginDrawPrim` / `EndDrawPrim` | `0x04` / `0x08` | — |
| `SetTexture` | `0x10` | 48 次引用 |
| `SetTransformType` / `SetColorOp` / `SetAlphaBlendMode` | `0x14` / `0x18` / `0x1C` | 各约 24 次 |
| `SetZBufferMode` / `SetAlphaTestMode` / `SetClipMode` / `SetFillMode` | `0x20` / `0x24` / `0x28` / `0x2C` | 各约 24 次 |
| `DrawPrim(LT_POLYGT4*, count)` | `0x48` | 带纹理四边形 |
| `DrawPrim(LT_POLYF4*, count)` | `0x54` | 纯色块（59 次，多数 UI 用 POLYF4） |
| `SetXYWH(POLYGT4*)` / `SetRGBA(POLYGT4*)` | `0xA0` / `0xD0` | 带纹理路径 |
| `SetXYWH(POLYF4*)` / `SetRGBA(POLYF4*)` | `0xAC` / `0xDC` | 纯色路径（交叉印证 `IBase+1` 模型） |

### `ILTTexInterface` vtable 偏移（`IBase` 占 1 槽，已核验）

| 方法 | vtable 偏移 | 备注 |
| --- | --- | --- |
| `CreateTextureFromName` | `0x08` | — |
| `GetTextureDims` | `0x18` | 调用处传 `htexture` + 两个输出局部变量，吻合 |

### `CUIPolyString` vtable 偏移（已核验）

依据：既有代理已用 `vt[11]` 作为 `SetFont` 且工作正常，反推整表（`index 0` = 虚析构，`index 1` = `GetClassName`）：

| 方法 | vtable 偏移 | 备注 |
| --- | --- | --- |
| `SetColor` | `0x0C` | — |
| `SetPosition` | `0x28` | — |
| `SetFont` | `0x2C` | `vt[11]`，印证 |
| `GetText` | `0x30` | — |
| `GetPolys` | `0x34` | 返回 `LT_POLYGT4*` |
| `GetFont` | `0x38` | — |
| `GetPosition` | `0x40` | — |
| `GetLength` | `0x58` | — |
| **`Render`** | **`0x60`** | **`index 24`** |
| `RenderClipped` | `0x64` | — |

### 绘制所需枚举值（头文件定义）

- `TransformType`: `SCREEN = 1`
- `ColorOp`: `MODULATE = 1`
- `BlendMode`: `BLEND_MOD_SRCALPHA = 3`
- `ZBuffer`: `NOZ = 2`
- `AlphaTest`: `NOALPHATEST = 0`
- `Clip`: `NOCLIP = 0`
- `Fill`: `FILL = 1`

### Phase 1 落地方案（hello-glyph）

1. 代理转发 `SetMasterDatabase`（与既有代理一致）。
2. 挂钩字体管理器 `CreateFormattedPolyString`（`g_pFontMgr` vtable 偏移 `0x3C`，已知），首次捕获 `CUIPolyString` 实例后读取其类 vtable，将 `Render`（偏移 `0x60`）patch 为自绘 hook（仅 patch 一次）。
3. Render hook：先调用原 `Render` 绘制正常文本，再用 `g_pTexInterface->CreateTextureFromName` 加载图集 DTX（仅一次），按既定绘制序列设置状态并绘制单个 `LT_POLYGT4`。
4. 验证链路成立后，在 Phase 2 将同一 Render hook 升级为查表自绘整串中文。

### Phase 1 产出文件

- `_work/proxy_cshell_renderer/cshell_proxy_renderer.cpp`：渲染器代理。转发 `SetMasterDatabase`，patch 字体管理器 `CreateFormattedPolyString` 捕获 poly string 并 patch 其 `Render`（偏移 `0x60`）；加载图集 DTX 并按已核验 vtable 偏移操作。
- `_work/proxy_cshell_renderer/build_proxy.ps1`：MSVC 编译代理（x86，基址 `0x50000000`，导出 `SetMasterDatabase`）。
- `_work/tools/build_phase1_renderer.py`：编译代理 + 打包 REZ（CSHELL 代理 + `Interface\Fonts\NOLF2CN_ATLAS.DTX`）+ 生成 `run_phase1_renderer.bat`。
- hello 字形测试直接读取 `_work/build_cn_renderer/stage/Interface/Fonts/NOLF2CN_ATLAS.MET` 的第一条记录（在 `runtime.log` 打印对应 `U+xxxx`）。

### 编译修正

- 首次编译报 `C3646` / `C2059` / `C2078`：`LT_VERTRGBA` 使用了 LithTech 的 `uint8` 拼写但未定义。已在文件头补充 `typedef uint8_t uint8;`。

### Phase 1 首次运行结果：管线打通

`runtime.log` 全部通过：`renderer proxy attached` → `patched CUIFontManager::CreateFormattedPolyString` → `patched CUIPolyString::Render` → `atlas metrics: cp=U+00B7` → **`atlas texture created`**。

- 关键结论：手写的最小 BGRA DTX 头**被引擎 `CreateTextureFromName` 正常接受**（`rc=0`，成功拿到 `HTEXTURE`），最大风险点已通过。
- Render hook 确实在渲染帧内执行，但首测屏幕未见明显字形是因为度量表第一条为 `U+00B7`（间隔号「·」），在 40 px 格中几乎全空；随后改为绘制整张图集（UV `0..1`，左上 480×480 方块）作无歧义验证。
- 注意：自绘仅在有 formatted poly 渲染时触发，主菜单每帧渲染，最适合验证。

### Phase 1 关键突破：直接接管引擎文字渲染

多轮实测结论：

- `CreateTextureFromName` 接受自制 BGRA DTX（`atlas texture created`），纹理路径畅通。
- 但**代理内自行发起的 `g_pDrawPrim->DrawPrim`（无论带纹理、纯色红块或剔除设置）均不显示也不崩溃** —— 引擎文字绘制位于特定 `Begin` / `End` 批次内，批次外补画的图元会被丢弃。
- 改用 **v3 方案：在引擎绘制前，直接修改 poly string 自身的字形四边形（`GetPolys` 偏移 `0x34`、`GetLength` 偏移 `0x58`），再交由引擎正常绘制。** 实测将所有顶点染红后，**菜单 / 标签 / HUD 所有文字均变红**（截图确认）。
- 意义：
  1. `CUIPolyString::Render`（偏移 `0x60`）hook 能拦截绝大多数文字。
  2. 引擎不会在 `Render` 内重建四边形，修改会保留并直接绘制。
  3. 架构定型：**不再另起 `DrawPrim`，而是“改写引擎的四边形 UV/坐标 + 临时切换本次绘制的字体贴图”**，彻底绕开 `DrawPrim` 上屏问题。

### 里程碑达成：中文从图集渲染到屏幕（2026-07-12）

实测将每条 poly string 的字体贴图临时替换为图集、四边形 UV 全部指向「中」（`U+4E2D`）后，**菜单所有文字均渲染为 SimHei 的「中」字，颜色正确（选中项红色、其余深色），稳定运行不崩溃（swap 跑至 1801+ 帧）**。

新增已核验的 vtable 偏移：

- `CUIPolyString`：`GetPolys` `0x34`、`GetFont` `0x38`、`GetLength` `0x58`、`Render` `0x60`。
- `CUIFont`：`GetTexture` `0x10`、`SetTexture(HTEXTURE, uint16*, uint8)`（proportional 重载）`0x48`。
- 核心手法：在 `Render` hook 调用 `g_orig_render` 前执行 `font->SetTexture(atlas)` 并改写 UV，绘制完毕后立即 `font->SetTexture(origTex)` 还原（同步绘制，swap-around 安全，不影响其它串）。
- `MODULATE` 配合原顶点色，文字颜色自动保留。

### 里程碑达成：真实译文正确渲染（2026-07-12）

使用 14 词演示表（英文 → 中文 codepoint）测试，在 hook 中按 poly 的英文原文匹配，命中则重排汉字与图集 UV，未命中保持英文。实测载入关卡后的 ESC 菜单**全部正确显示中文**（系统 / 任务 / 装备 / 物品 / 情报 / 玩家 / 继续游戏 / 载入游戏 / 保存游戏 / 选项 / 离开游戏 / 退出 / 游戏已暂停），位置、颜色与排版均正常。

**最终架构确定（locale 无关、无需修改 CRES）：**

1. 图集 DTX（全部字形）+ MET（`codepoint → UV`）。
2. 运行时英文 → 中文对照表（由 TSV 生成），按英文原文匹配。
3. 渲染器代理：挂钩 `CUIPolyString::Render`，命中则临时切换字体贴图为图集 + 汉字重排 UV，绘制后还原。

关键实现细节：

- 通过 `GetText`（偏移 `0x30`）读取英文原文，纯 ASCII 在任何 locale 下均不乱码。
- 汉字排版：读取 `quad[0]` 的 `x` / `y` 与字高，按字高步进重排前 `M` 个 quad，多余 quad 的顶点 alpha 置 `0` 隐藏。
- `.cpp` 源码需保持纯 ASCII 或在编译时传入 `/utf-8`（否则 MSVC 按 CP932 误读中文注释吃行）；已在 `build_proxy.ps1` 中添加 `/utf-8`。

### 里程碑：全量汉化跑通（2026-07-12）

接入完整对照表（4522 条，由 TSV 生成）与包含 ASCII 的图集后，实测**整个游戏完成中文化**：主菜单（单人游戏 / 继续游戏 / 多人游戏(互联网) / 多人游戏(局域网) / 选项 / 档案 / 退出 + 帮助行）、加载页（第1章：凯特·阿彻必须死！ / Tip）、场景字幕（H.A.R.M. 秘密地下巢穴 位置不明）、游戏内目标（与羽鸟先生会合 / 交谈 / 留意他留下的信息）全部正常显示。

产物与命令：

- `_work/tools/build_cn_dict.py`：从两张 TSV 生成 `NOLF2CN_STRINGS.bin`（英文 → 中文 codepoint，按英文排序，4522 条）。
- `build_cn_glyph_atlas.py`：图集补齐 ASCII `0x21-0x7E`（支持混排数字与英文缩写），字形左对齐并记录 `advance`。
- `build_phase1_renderer.py`：一键执行 atlas + dict + 编译代理 + 打包 REZ。
- 代理启动时加载 MET + BIN，在 hook 内按英文二分查表，命中后重排 UV 并切换贴图渲染。

### 打磨 1：比例排版 + 换行（2026-07-12）

- 等宽方块排版观感不佳 → 图集字形改为左对齐、按 `advance` 裁剪 UV；代理按每字 `advance` 排布（汉字近方块，ASCII 变窄）。
- 自动换行 → 以引擎原英文（已 word-wrap）的四边形横向范围作为行宽，对中文按字断行；译文 `\n` 触发换行，空格/缺字保留半角间隙。

### 已知待优化（非阻塞）

- 居中/右对齐界面按原 `x0` 起排，未按中文宽度重新居中（略有偏移，后在打磨 5 中修复）。
- 重复英文原文取首条译文，个别上下文可能存在歧义。
- 含 `%s` / `%d` 的动态串运行时英文已被替换，精确匹配不到（后在打磨 3 中支持）。
- 长文本换行按行宽估算，复杂排版可能与原生略有差异。

### 打磨 2：匹配与换行修正（2026-07-12）

针对部分英文串（机械八哥提示、帮助文本）未匹配，以及对话选项等定高列表换行溢出的问题：

- **匹配修复（Fix B）**：`build_cn_dict.py` 此前对英文执行了 `.strip()`，但运行时 `GetText` 返回的是**原始 CRES 字符串，常带尾随空格**（如 `...Mission Status screen. `），导致 key 不一致。改为对英文**只 unescape 不 strip**，精确匹配运行时串；词条数从 4522 增至 4528。
- **换行修复（Fix A）**：代理改为仅在**原英文本身即为多行**时才对中文换行（判据：原始四边形 `y` 跨度 > 0.5 倍字高），单行标签与定高列表项保持单行不溢出。

### 打磨 3：格式串（`%1!d!` 等）支持（2026-07-12）

- 问题：技能点、弹药、拾取等提示使用 LithTech 定位格式 `%1!d!`，运行时数字已填入（如 `You received 20 skill points...`），与模板 key 不一致。
- 方案：`build_cn_dict.py` 将含 `%` 的 163 条拆为 **FMT 模板表**（`en` 与 `zh` 均保留格式符），主表 4365 条继续精确匹配；`.bin` 格式升级为 `CNS2`（`main` + `fmt` 两段）。
- 代理：精确匹配失败时线性扫描 FMT 模板，用英文模板匹配运行时文本并捕获参数，再按定位参数号回填至中文模板渲染。已用 Python 原型验证（如 `获得20点技能点`、`（可用技能点：60）`）。
- 支持 `%N!T!` 定位式与 `%[flags]T` 简单式及 `%%` 转义，按参数号回填可处理中英文参数顺序差异。

### 打磨 4：格式串误匹配 + 情报 `@` 拆分（2026-07-12）

- 便签被 `%1!d! of %2!d!` 误匹配：该数字模板把便签中 `Director **of** Gadgetry` 的 `" of "` 当作锚点，导致整段英文被套进模板重绘。
  - 修复：代理匹配器记录每个格式符的类型，**数字类 `%d` / `%i` / `%u` / `%x` / `%o` 的捕获必须为纯数字**（可带正负号），否则不匹配；`build_cn_dict.py` 同时丢弃字面锚点少于 3 个字符的退化模板（如 `%1!s!!`）。
- 情报条目在词表中以 `标题@正文` 形式存储，而游戏分别渲染标题（情报列表）与正文（便签），与完整 `标题@正文` 对不上：
  - 修复：`build_cn_dict.py` 对含 `@` 的条目**额外拆出“标题”和“正文”两条**。主表由 4365 增至 4900，情报列表标题与便签正文均可正常匹配。
- 说明：便签本身是**文本渲染在纸张贴图上**（并非整张图片），因此可正常汉化；只有 `LOADING` 大字、纸张底纹与部分手写信件贴图属于真正的图片文字。

### 仍为英文的少量项说明

- 武器专有名（如 `Vindicator Sportsman's Crossbow`、`Lipstick Spy Camera`）与个别情报标题多为 TSV 中未填 `zh`，属翻译数据缺口，补齐译文即可。
- 个别目标串（如 `Recover six pages...(optional)`）运行时会追加 ` (optional)` 后缀（已在打磨 5 处理）。

### 打磨 5：拼接串 + 嵌入名 + 字幕对齐（2026-07-12）

第一章通关无问题。针对剩余的运行时拼接英文（物品名本身已译）：

- 格式串中 `%s` 捕获的内容若本身在词表中，自动替换为其译文（如难度名 `Amateur` → `业余`、嵌入物品名等）。
- 拾取提示 `<名称>!` → 查名称译文 + `!`（如 `武士刀!`、`密码破译器!`）。
- `Select <名称>` / `Select <名称> : n/m` → `选择` + 名称译文 + 尾部（如 `选择弩箭 : 10/20`）。
- 目标 `<正文> (optional)` → 正文译文 + `（可选）`。
- 字幕偏左修正：代理读取 `GetPosition`（偏移 `0x40`）锚点，按原文对齐比例 `f = (px - minX) / 宽` 对单行中文重新定位（左/中/右自适应），居中字幕不再左偏。

上述拼接与后缀处理均为数据驱动，仅在精确匹配失败时触发。

### 打磨 6：丢字隐患审计（2026-07-12）

系统排查可能导致中文截断的隐患：

- 格式串：最多 6 个占位符、最长字面段 193 字符，均在代理上限内（8 / 255），无因超限回退英文的情况。
- 缓冲区：最长译文 223 字 < 512，长情报文档无截断。
- 标签（如 `等级:` 等）为独立文本分别匹配，非拼接，无问题。
- **唯一真实隐患**：引擎按英文字符数分配四边形，若译文的“可见字形数 > 英文字符数”会截断尾部。全表共命中 8 条短串。
- 修复：缩短这 8 条译文使 `drawn(zh) <= len(en)`：
  - 5 条 FMJ 弹药：`全金属被甲弹` → `全金属弹`
  - `ASSS` → `抗超血清`、`CIA?` → `中情局?`、`So?` → `所以?`
  - 复查：全表 0 条截断。
- **通用规则**：**译文的可见字形数不能超过英文原文字符数**（否则超出部分会被引擎预分配的四边形数量截断），后续补充译文时需注意。

### 打磨 7：HUD 弹药拼接 + 情报标题数据修复 + 滚动长文本（2026-07-12）

- 武器 HUD 弹药 `(cur/max) <name>`（如 `(8/20) Bolt`）由代码拼接，无 CRES 模板。代理新增 `(N/M) <名称>` 处理：名称命中词表则翻译（如 `(8/20) 弩箭`），保留计数前缀。
- 情报标题 `：装备界面`（ID `25014`）为**译文漏字**（英文 `Reminder: Equipment screen`），已补全为 `提醒：装备界面`（纯数据修复，无需重编）。
- 超长帮助字幕（ID `11016` `"The Equipment screen shows you..."`）词表已翻译，但游戏采用**滚动/逐段显示**，运行时 `GetText` 可能只返回露出前缀，与完整英文不等而未匹配。属滚动文本边角，暂记录未修。

### 打磨 8：同时钩普通 poly（2026-07-12，实验后回退）

- 推测滚动长提示（ID `11016`）走的是**普通 `CUIPolyString`**（`CreatePolyString`），而此前仅挂钩了 `CreateFormattedPolyString`。
- 字体管理器 vtable（`IBase+1`）：`CreatePolyString` = `index 14` = `0x38`（16 次调用），`CreateFormattedPolyString` = `index 15` = `0x3C`（32 次调用），`DestroyPolyString` = `0x40`（63 次调用）。
- 先后尝试过单 Render 钩判断 vtable 与双独立 Render 钩（`CnRenderHookFmt` / `CnRenderHookPlain` + 未匹配 >80 字长串日志），但实测**挂钩 `CreatePolyString`（`0x38`）均会在载入结束时触发闪退**。
- 最终已按 2026-07-13 结论回退为**仅挂钩 `CreateFormattedPolyString` 的稳定版**，保留弹药 HUD、情报标题、居中对齐等全部其他改进。

### 便签 / 信件类“图片文字”说明

游戏中 Cate 捡到的纸质便签与手写信件部分是将文字**烘焙进 DTX 贴图**的图片，并非 poly string 文本，渲染器无法拦截，仍显示英文；如需汉化需单独重绘对应 DTX 贴图。情报“日志”界面中的文字条目属于文本，可正常汉化。

### 历史备忘（Phase 1 / 2 早期设想与风险验证）

- 早期 Phase 2 曾考虑通过 CRES ASCII 占位字节配合 `SetTexture` swap-around，后演进为更简洁的“不改 CRES、直接按英文原文查表 + swap-around 贴图”方案。
- Phase 1 重点验证的两大风险点（最小 DTX 头能否被 `CreateTextureFromName` 接受、菜单文本是否走 formatted poly 的 vtable）均已实测通过。

> [!NOTE]
> 以下第 1–24 轮为早期探索与已弃用的 `CRES.DLL` / 载体字节分页字体路线开发归档，保留作逆向与历史参考；当前正式方案见文档前半部分的**运行时自绘中文渲染器**章节。

## 2026-07-10 第 1 轮：项目初始化与资源确认

### 修改内容

- 创建汉化工作区：
  - `_work/analysis`
  - `_work/patch`
  - `_work/archive`
  - `_work/tools`
- 创建本记录文件 `chinese_note.md`。

### 采用的技术

- 使用游戏自带官方工具包中的 `Tools/Bin/LithRez.exe` 作为后续 REZ 解包/打包工具。
- 先分析现有已解包目录 `Game`，避免重复解包大型资源。

### 已确认事实

- 游戏根目录存在主要资源包：`GAME.REZ`、`GAME2.REZ`、`GAMEDLL.REZ`、`SOUND.REZ`、`Update_v1x3.rez`。
- `Game` 目录已经包含解包后的资源，包括 `ATTRIBUTES`、`INTERFACE`、`Worlds`、`VOICE`、`SND` 等目录。
- `Tools/Bin/unrez.bat` 的官方解包顺序为：
  1. `GAME.rez`
  2. `GAME2.rez`
  3. `GAMEDLL.rez`
  4. `SOUND.rez`
  5. `Update_v1x3.rez`
- `Tools/Bin/LithRez.exe` 存在，可作为后续生成汉化 REZ 的基础工具。

### 问题

- 早期本地沙箱环境在读取/创建部分游戏目录时出现 `windows sandbox: helper_unknown_error: setup refresh had errors`，改为在常规 Windows 环境下完成只读盘点和 `_work` 创建。

### 验证结果

- 已确认 `_work` 工作区创建成功。
- 已确认 `Game` 解包资源目录可用。

### 下一步

- 盘点 `Game` 中所有文本、字幕、界面、字体相关资源。
- 找出首批可以安全汉化且可打包覆盖的资源。
- 确认简体中文字体/编码方案，避免游戏内乱码或缺字。

## 2026-07-10 第 2 轮：本地化文本主资源与字体风险确认

### 修改内容

- 生成资源盘点文件：
  - `_work/analysis/extension_counts.txt`
  - `_work/analysis/game_file_inventory.csv`
  - `_work/analysis/text_candidate_files.csv`
  - `_work/analysis/key_directories.txt`
- 新增 PE 字符串表抽取工具：`_work/tools/extract_pe_stringtable.py`。
- 从 `Game/CRES.DLL` 抽取 6313 条 RT_STRING 字符串到 `_work/analysis/CRES_strings.csv`。
- 确认 `Game/SRES.DLL` 不含 RT_STRING 字符串，输出 `_work/analysis/SRES_strings.csv` 作为验证记录。
- 生成翻译工作表 `_work/patch/CRES_zh_template.csv`。
- 生成属性表字符串 ID 引用清单 `_work/analysis/attribute_string_id_refs.csv`。
- 解压官方源码到 `_work/source`，用于分析字符串和字体渲染管线。

### 采用的技术

- 使用自写 Python 脚本解析 PE 资源目录中的 `RT_STRING`，按 Windows STRINGTABLE 规则还原字符串 ID：`(block_id - 1) * 16 + index`。
- 使用 PowerShell 生成资源清单和属性表字符串 ID 引用清单。
- 使用官方源码核对字符串读取和字体创建逻辑。

### 已确认事实

- `Game/ATTRIBUTES/MISSIONS.TXT` 明确说明任务名、任务说明、目标文本等 ID 存储在 `CRes.dll`。
- `Game/CRES.DLL` 是当前最重要的本地化字符串库，包含菜单、任务、目标、物品、提示等 6313 条英文字符串。
- 抽样验证：
  - ID 2500 = `CHAPTER 1. Cate Archer Must Die!`
  - ID 2750 = `Cate travels to Japan to investigate a rumored international crime convention.`
  - ID 5001 = `Rendezvous with Hatori-san.`
  - ID 4503 = `Dead batteries`
  - ID 8000 = `You found a letter containing classified information.`
- `Game/VOICE/*.TXT` 抽样后确认不是字幕文本，而是口型/面部动画曲线数据。
- `Game/ATTRIBUTES/LAYOUT.TXT` 的 `[Fonts]` 段定义 4 个界面字体：
  - `FontFace0 = "Square 721 Blk"`, `FontFile0 = "Interface\Fonts\SQR721KN.TTF"`
  - `FontFace1 = "Expo"`, `FontFile1 = "Interface\Fonts\EXPON.TTF"`
  - `FontFace2 = "Typist"`, `FontFile2 = "Interface\Fonts\TYPIST.TTF"`
  - `FontFace3 = "BethHand"`, `FontFile3 = "Interface\Fonts\BETHHAND.TTF"`
- 源码 `ClientUtilities.cpp` 使用 ANSI `LoadString(...)` 从 `cres_hinstance` 读取字符串到 `char*` 缓冲区。
- 源码 `InterfaceResMgr.cpp` 创建字体时调用 `CreateFont(pszFontFile, pszFontFace, ptSize, 33, 255)`，只覆盖 33-255 字符范围。

### 问题

- 仅替换 `CRES.DLL` 字符串为中文，不足以实现完美汉化：当前客户端文本管线是 ANSI/单字节倾向，且字体纹理只生成 33-255 范围，中文 Unicode 字符大概率无法正确显示。
- 完整简体中文汉化需要进一步解决：
  1. 字符串读取：改造 `LoadStringA`/`char*` 管线，或确认可行的本地代码页方案。
  2. 字体生成：扩大字体 glyph 范围或改成按需生成中文 glyph。
  3. 字体文件：引入可覆盖简体中文的 TTF/TTC，并确认 LithTech 字体管理器是否支持对应格式。
- `_work/tools/extract_pe_stringtable.py` 当前对“无资源目录 DLL”会报错；这不影响 `CRES.DLL` 抽取，但后续应修复为返回 0 条。

### 验证结果

- `CRES_strings.csv` 成功抽取 6313 条字符串。
- `CRES_zh_template.csv` 已包含 `id/lang/source/zh/status/note` 字段，可作为后续翻译主表。
- `attribute_string_id_refs.csv` 显示属性表引用量最高的文件为：
  - `WEAPONS.TXT`: 216 条
  - `SEARCHITEMS.TXT`: 173 条
  - `KEYITEMS.TXT`: 82 条
  - `SURFACE.TXT`: 53 条
  - `MISSIONS.TXT`: 45 条

### 归档情况

- 本轮没有旧步骤产生的无用文件需要归档。
- 空的 `SRES_strings.csv` 保留为验证证据，不归档。

### 下一步

- 修复 `extract_pe_stringtable.py` 对无资源 DLL 的处理。
- 编写 `patch_pe_stringtable.py`，先在 `CRES.DLL` 副本上测试少量中文字符串写回。
- 同步研究客户端 DLL 是否可从源码重编译，重点改造 `ClientUtilities.cpp` 和 `InterfaceResMgr.cpp` 的中文显示限制。
- 选择本地可用中文字体进行测试，优先考虑 `NotoSansSC-VF.ttf`、`simhei.ttf` 或 `Deng.ttf`。

### 追加验证

- 已修复 `_work/tools/extract_pe_stringtable.py`：当 DLL 没有资源目录时返回 0 条字符串，而不是报错。
- 重新抽取 `Game/CSHELL.DLL`，结果为 0 条 RT_STRING，输出 `_work/analysis/CSHELL_strings.csv`。

## 2026-07-10 第 3 轮：CRES 写回烟测包与中文显示风险验证

### 修改内容

- 新增 `_work/tools/patch_pe_stringtable.py`，用于在不改变 PE 资源块大小的前提下写回 RT_STRING。
- 新增 `_work/patch/CRES_smoke_replacements.csv`，包含 6 条主菜单测试中文：
  - 502 = 单人游戏
  - 504 = 选项
  - 505 = 新游戏
  - 506 = 载入游戏
  - 507 = 保存游戏
  - 524 = 多人游戏
- 生成测试 DLL：`_work/build_smoke/CRES.DLL`。
- 重新抽取测试 DLL 到 `_work/analysis/CRES_smoke_strings.csv`，确认中文已写入资源表。
- 生成测试 REZ：`_work/build_smoke/NOLF2_CN_SMOKE.rez`。
- 新增测试启动脚本：`_work/run_cn_smoke.bat`。该脚本读取现有 `launchcmds.txt`，并在最后追加测试 REZ；不会修改原启动配置。

### 采用的技术

- 当前写回策略为“等长资源块内写回”：替换文本长度不能超过原字符串长度，较短文本自动用空格补齐，以避免重排 PE 资源目录。
- 使用 `LithRez.exe cv` 将测试版 `CRES.DLL` 打包成独立 REZ。
- 使用 `LithRez.exe v` 验证测试 REZ 内容，确认包内只有根目录下的 `CRES.DLL`。
- 读取 Windows NLS 设置，确认当前系统 ANSI 代码页。

### 已确认事实

- `patch_pe_stringtable.py` 可成功将中文写入 `CRES.DLL` 副本。
- 重新抽取 `_work/build_smoke/CRES.DLL` 后，ID 502/504/505/506/507/524 均显示为中文，说明 PE 资源写回层面可行。
- 当前系统语言环境：`ja-JP`。
- 当前 ANSI/OEM 代码页：`932`。
- 在 CP932 下，部分简体中文会转换为 `?`，例如：
  - `选项` -> `3f3f`
  - `单人游戏` 中的 `单` 和 `戏` 无法按 CP932 表示。

### 问题

- 游戏源码使用 ANSI `LoadString(...)` 读取 `CRES.DLL` 字符串，因此当前日文代码页环境下，资源中的简体中文即使写入成功，也可能在进入游戏 `char*` 缓冲区时变成 `?`。
- `CUI_MAX_BITMAPFONT_CHARS = 255`，且 `CUIFont` 的 map 是 `uint8*`；原 UI 字体系统不是完整 Unicode 字体系统，无法直接承载完整简体中文字库。
- 当前测试 REZ 只能验证“资源覆盖/字符串写回链路”，不能代表最终中文显示可用。

### 验证结果

- `NOLF2_CN_SMOKE.rez` 打包成功，大小约 696 KB。
- `LithRez v` 显示包内资源为：根目录 `CRES.DLL`，类型 `DLL`，大小 696320。
- 当前无需归档旧文件；本轮生成文件都是下一轮验证证据或工具。

### 下一步

- 若需要肉眼确认资源覆盖链路，可运行 `_work/run_cn_smoke.bat`，预期主菜单部分文字可能显示为问号或缺字。
- 继续研究可行显示方案：
  1. 尝试将系统/进程 ANSI 代码页问题绕开，避免 `LoadStringA` 破坏中文。
  2. 评估是否能改造 UI 文本渲染为动态 `StringToTexture(uint32*)` 或其他 Unicode 路线。
  3. 如果不能改造引擎/UI 层，则只能做有限字符集的伪汉化，无法达到“完美简体中文汉化”。

## 2026-07-10 第 4 轮：CRES 资源 DLL 可重建性验证

### 修改内容

- 新增 `_work/build_resdll/include/afxres.h`，作为本地 MFC `afxres.h` shim，仅用于现代 Windows SDK 编译官方资源脚本。
- 使用官方源码中的英文资源脚本 `_work/source/Game/ClientRes/TO2/Lang/EN/ClientRes.rc` 编译生成 `_work/build_resdll/ClientRes_EN.res`。
- 使用现代 VS BuildTools `link.exe /dll /noentry /machine:x86` 生成 `_work/build_resdll/CRes_rebuilt_EN.dll`。
- 新增自动构建脚本 `_work/tools/build_cres_resdll.ps1`。
- 用构建脚本再次生成 `_work/build_resdll/CRes_rebuilt_EN_script.dll`。
- 抽取重建 DLL 字符串到：
  - `_work/analysis/CRES_rebuilt_EN_strings.csv`
  - `_work/analysis/CRES_rebuilt_EN_script_strings.csv`

### 采用的技术

- 使用 Windows SDK `rc.exe` 编译 `.rc` 资源脚本。
- 使用 VS BuildTools 32 位目标 `link.exe` 以 `/NOENTRY` 方式生成资源 DLL，避免依赖 VC6 或运行库。
- 为缺失的 MFC `afxres.h` 提供本地 shim，内容只包含 `windows.h` 和 `winres.h`。
- 添加 include 路径以解析官方共享资源脚本：
  - `_work/source/Game/ClientRes/Shared/Lang/EN`
  - `_work/source/Game/Shared/Lang/EN`

### 已确认事实

- 官方源码中存在完整 ClientRes 多语言资源工程：`_work/source/Game/ClientRes/TO2`。
- 英文 `ClientRes.rc` 可用现代 `rc.exe` 编译。
- 重建出的英文资源 DLL 可由现有 `extract_pe_stringtable.py` 正常抽取。
- 重建 DLL 与原始 `Game/CRES.DLL` 的字符串数量一致，均为 6313 条。
- 完整字符串 diff 结果为 0，说明重建资源 DLL 的 RT_STRING 内容与原始 CRES 完全一致。

### 问题

- 资源 DLL 可重建只解决“字符串长度/资源写回”问题；仍未解决游戏运行时 `LoadStringA` 和 CUI 255 字形限制。
- 现代工具可重建 CRES，但旧客户端 DLL/引擎显示中文仍需要单独处理。

### 验证结果

- `_work/tools/build_cres_resdll.ps1` 已验证可用。
- `_work/build_resdll/CRes_rebuilt_EN_script.dll` 字符串抽取 6313 条。
- 与原始 `CRES_strings.csv` 对比差异数为 0。

### 归档情况

- 本轮没有旧步骤产生的无用文件需要归档。

### 下一步

- 基于官方 `ClientRes.rc` 或抽取 CSV 生成可维护的中文资源脚本。
- 同时继续攻克显示管线：当前关键阻塞仍是 ANSI `LoadStringA` 和 CUI 字体系统最多 255 字形。

### 追加归档

- 第 4 轮脚本化重建确认无误后，已将手工重建阶段的冗余产物归档到 `_work/archive/2026-07-10_resdll_manual_artifacts`：
  - `_work/build_resdll/ClientRes_EN.res`
  - `_work/build_resdll/CRes_rebuilt_EN.dll`
  - `_work/build_resdll/RCa27492`

## 2026-07-10 第 5 轮：单字节载体 + 编码字体 smoke 包

### 修改内容

- 新增编码字体 smoke 构建脚本：`_work/tools/build_encoded_font_smoke.py`。
- 将 `fontTools` 安装到工作区局部依赖目录：`_work/tools/pydeps`，仅用于生成测试字体。
- 新增主菜单 6 条 smoke 替换表：`_work/patch/CRES_encoded_smoke_replacements.csv`。
- 新增字符载体映射表：`_work/analysis/encoded_smoke_char_map.csv`。
- 新增分析说明：`_work/analysis/encoded_font_smoke.md`。
- 生成编码字体 smoke 包：`_work/build_encoded_smoke/NOLF2_CN_ENCODED_SMOKE.rez`。
- 新增启动脚本：`_work/run_cn_encoded_smoke.bat`。

### 采用的技术

- 不直接把中文写入 `CRES.DLL`，而是写入稳定 ASCII 载体，避免当前 CP932 环境下 `LoadStringA` 将简体中文转换成 `?`。
- 使用 `simhei.ttf` 作为源字体，通过 `fontTools` 修改 cmap，让 ASCII 载体 `A-L` 指向对应中文字形：
  - `A=单`, `B=人`, `C=游`, `D=戏`, `E=选`, `F=项`
  - `G=新`, `H=载`, `I=入`, `J=保`, `K=存`, `L=多`
- 修改 smoke 包内的 `ATTRIBUTES/LAYOUT.TXT`，将 4 个 UI 字体槽全部指向测试字体 `Interface\Fonts\NOLF2CN_SMOKE.TTF`，字体面名为 `SimHei`。
- 使用 `LithRez.exe cv` 打包独立 REZ 覆盖包。

### 已确认事实

- 生成的 `CRES.DLL` 中目标 ID 回读结果为 ASCII 载体：
  - ID 502 = `ABCD`，预期显示 `单人游戏`
  - ID 504 = `EF`，预期显示 `选项`
  - ID 505 = `GCD`，预期显示 `新游戏`
  - ID 506 = `HICD`，预期显示 `载入游戏`
  - ID 507 = `JKCD`，预期显示 `保存游戏`
  - ID 524 = `LBCD`，预期显示 `多人游戏`
- 测试字体已通过 glyph ID 验证：载体 `A-L` 在测试字体中指向的 glyph ID 与原 `simhei.ttf` 中对应中文字的 glyph ID 一致。
- `LithRez v` 确认 smoke 包只包含：
  - 根目录 `CRES.DLL`
  - `ATTRIBUTES\LAYOUT.TXT`
  - `INTERFACE\FONTS\NOLF2CN_SMOKE.TTF`

### 问题

- 这个方案是技术 smoke，不是最终翻译包；由于载体占用了普通 ASCII 字母 `A-L`，未编码的英文 UI 中这些字母可能显示成中文字形。
- 每个字体上下文可用的稳定单字节载体有限。若 smoke 成功，完整汉化需要按菜单、HUD、任务文本等上下文拆分字库和编码表。
- 仍未证明运行时 `CreateFont` 能接受这个修改后的 `simhei.ttf`；需要实际启动游戏肉眼验证。
- 追加发现：`NOLF2.exe` 是启动器，不是实际游戏客户端。启动器源码会将参数写入 `launchcmds.txt`，再执行 `lithtech.exe -cmdfile launchcmds.txt`。

### 验证结果

- `_work/tools/build_encoded_font_smoke.py` 可一键重建 smoke 包。
- `_work/analysis/CRES_encoded_smoke_strings.csv` 已确认目标字符串为 ASCII 载体。
- `_work/build_encoded_smoke/NOLF2_CN_ENCODED_SMOKE_listing.txt` 已确认 REZ 内容正确。
- 已修正 `_work/run_cn_encoded_smoke.bat` 和 `_work/run_cn_smoke.bat`：现在直接调用 `Lithtech.exe -cmdfile`，不再打开 `NOLF2.exe` 启动器。

### 归档情况

- 本轮没有旧步骤产生的无用文件需要归档。

### 下一步

- 运行 `_work/run_cn_encoded_smoke.bat` 肉眼验证主菜单是否显示中文。
- 若显示成功，继续做“菜单级有限字库”方案：统计首批菜单文本所需汉字，生成稳定载体编码表、编码后的 `CRES.DLL` 和匹配 TTF。
- 若显示失败，回到客户端构建路线，优先处理 `CSHELL.DLL` 重建依赖和 CUI 字体管线改造。

## 2026-07-10 第 6 轮：菜单阶段一编码包离线生成

### 修改内容

- 新增菜单阶段一候选翻译表：`_work/patch/menu_stage1_candidates.csv`。
- 新增候选字频统计：`_work/analysis/menu_stage1_source_charfreq.csv`。
- 新增菜单阶段一构建脚本：`_work/tools/build_menu_stage1_encoded.py`。
- 新增菜单阶段一编码替换表：`_work/patch/CRES_menu_stage1_encoded_replacements.csv`。
- 新增菜单阶段一载体映射表：`_work/analysis/menu_stage1_encoded_char_map.csv`。
- 新增菜单阶段一分析说明：`_work/analysis/menu_stage1_encoded.md`。
- 生成菜单阶段一 REZ：`_work/build_menu_stage1/NOLF2_CN_MENU_STAGE1.rez`。
- 新增启动脚本：`_work/run_cn_menu_stage1.bat`。

### 采用的技术

- 选择 `CRES.DLL` 中 ID `502-583` 作为第一阶段菜单/UI 标签范围，共 82 条。
- 使用简体中文草译后统计字形需求：阶段一需要 153 个中文字形。
- ASCII 可打印载体只有 94 个，不足以覆盖阶段一；因此加入当前 CP932 环境下可产生单字节的半角区载体 `0xA1-0xDF`。
- 生成字体 `NOLF2CN_MENU1.TTF`，将载体字节对应的字体 codepoint 映射到简体中文字形。
- 生成编码后的 `CRES.DLL`，资源字符串中保存的是单字节载体字符，而非直接中文。

### 已确认事实

- 完整 502-650 菜单候选范围共 149 条，独立中文字形约 226 个，超过单字体稳定容量，不适合一次性塞入一个 8 位字库。
- 较小的 502-583 范围共 82 条，独立中文字形 153 个；使用 ASCII + CP932 半角单字节载体后可以覆盖。
- 阶段一翻译中已移除裸 ASCII（如 `NOLF2`、`F9`、`[空槽]`、`-`），避免和载体字节冲突。

### 验证结果

- `_work/tools/build_menu_stage1_encoded.py` 成功生成阶段一 REZ。
- 重新抽取 `_work/build_menu_stage1/stage/CRES.DLL` 到 `_work/analysis/CRES_menu_stage1_encoded_strings.csv`，确认 ID `502-583` 均为载体字符串。
- `LithRez v` 确认阶段一 REZ 包内只有根目录 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_MENU1.TTF`。
- 字体 glyph ID 抽样验证通过，低位载体 `0x21-0x34` 和高位载体 `0xD2-0xDB` 都指向对应中文 glyph。

### 问题

- 阶段一仍未做运行时验证。
- 高位载体 `0xA1-0xDF` 依赖当前 CP932 ANSI 转换路径；如果运行时字体构建或 CUI 索引处理高位字节失败，需要拆成更小的 ASCII-only 字库上下文。
- 启动测试会打开游戏客户端，因此后续需要在不影响当前游戏时手动运行。

### 归档情况

- 本轮没有旧步骤产生的无用文件需要归档。

### 下一步

- 在方便打断游戏时运行 `_work/run_cn_menu_stage1.bat` 验证阶段一菜单显示。
- 若阶段一显示成功，继续按界面上下文拆分 584-650、选项子菜单、任务/物品文本。
- 若阶段一高位载体显示异常，则降级为 ASCII-only 小包，按更细界面上下文拆分。

## 2026-07-10 第 7 轮：ASCII-only 菜单探针包

### 修改内容

- 新增 ASCII-only 菜单探针构建脚本：`_work/tools/build_menu_ascii_probe.py`。
- 新增 ASCII-only 编码替换表：`_work/patch/CRES_menu_ascii_probe_replacements.csv`。
- 新增 ASCII-only 字符映射表：`_work/analysis/menu_ascii_probe_char_map.csv`。
- 新增 ASCII-only 探针说明：`_work/analysis/menu_ascii_probe.md`。
- 生成 ASCII-only 探针 REZ：`_work/build_menu_ascii_probe/NOLF2_CN_MENU_ASCII_PROBE.rez`。
- 新增启动脚本：`_work/run_cn_menu_ascii_probe.bat`。

### 采用的技术

- 选择 `CRES.DLL` 中 ID `502-541` 作为保守探针范围，共 40 条核心菜单/UI 标签。
- 该范围需要 85 个简体中文字形，可完全放入 ASCII 可打印载体 `0x21-0x7E`，不依赖 CP932 高位半角区。
- 与阶段一包相同，资源字符串写入载体字符，字体 `NOLF2CN_ASCII.TTF` 将载体 codepoint 映射到中文 glyph。

### 已确认事实

- `502-541` 范围是当前最适合做纯 ASCII 字体验证的核心菜单集合。
- 纯 ASCII 探针包可以作为阶段一包的 A/B 对照：
  - ASCII 探针成功、阶段一失败：高位载体 `0xA1-0xDF` 可能不可靠。
  - ASCII 探针失败：编码字体路线本身可能被引擎字体构建器阻断。

### 验证结果

- `_work/tools/build_menu_ascii_probe.py` 成功生成探针 REZ。
- 重新抽取 `_work/build_menu_ascii_probe/stage/CRES.DLL` 到 `_work/analysis/CRES_menu_ascii_probe_strings.csv`，确认目标字符串为载体文本。
- `LithRez v` 确认探针 REZ 包内只有根目录 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_ASCII.TTF`。
- 字体 glyph ID 抽样验证通过，载体 `0x21-0x75` 均正确指向对应中文 glyph。
- 运行验证追加：
  - 原始资源、窗口化参数、CRES-only 覆盖均可启动。
  - 使用完整 `simhei.ttf` 生成的 9.7 MB 字体会导致客户端 `0xC0000005` 崩溃。
  - 将字体裁剪为只含所需 glyph 后，ASCII 探针包可启动并显示中文字形。
  - `NOLF2CN_ASCII.TTF` 裁剪后大小为 22,844 字节，`NOLF2_CN_MENU_ASCII_PROBE.rez` 大小约 780 KB。
  - `NOLF2_CN_MENU_STAGE1.rez` 使用裁剪字体后也可启动，未再崩溃。

### 归档情况

- 本轮没有旧步骤产生的无用文件需要归档。

### 下一步

- 推荐运行测试顺序：
  1. `_work/run_cn_menu_ascii_probe.bat`
  2. 若成功，再运行 `_work/run_cn_menu_stage1.bat`
- 根据 A/B 结果决定继续使用高位载体扩大单包范围，还是拆分为 ASCII-only 小上下文字库。

### 追加结论

- 编码字体路线成立，但必须使用子集化 TTF；完整中文字体会让老引擎崩溃。
- 当前首屏停在性能设置界面，该界面很多字符串未纳入 ASCII 探针/阶段一包，因此英文被全局 ASCII 字体映射成随机中文。这是编码字体方案的固有限制，不是 glyph 映射失败。
- 后续可用包必须按“可见界面上下文”纳入全部同屏字符串，或改造客户端字体管线以避免全局 ASCII 重映射。

## 2026-07-10 第 8 轮：主菜单高位载体探针

### 修改内容

- 新增并迭代主菜单探针构建脚本：`_work/tools/build_main_menu_probe.py`。
- 新增主菜单编码替换表：`_work/patch/CRES_main_menu_probe_replacements.csv`。
- 新增主菜单载体映射表：`_work/analysis/main_menu_probe_char_map.csv`。
- 新增主菜单探针说明：`_work/analysis/main_menu_probe.md`。
- 生成主菜单 REZ：`_work/build_main_menu_probe/NOLF2_CN_MAIN_MENU_PROBE.rez`。
- 新增启动脚本：`_work/run_cn_main_menu_probe.bat`。

### 采用的技术

- 先用 ASCII-only 载体验证主菜单文字可以显示，但发现未覆盖的英文 `Continue game` 会被载体字体误映射成随机中文。
- 改为 CP932 高位半角单字节载体 `0xA1-0xDF`，资源字符串保存半角字符，运行时经 `LoadStringA` 回到单字节。
- 字体 `NOLF2CN_MAIN.TTF` 将 `0xA1-0xDF` 载体 codepoint 映射到中文 glyph，同时保留普通 ASCII `0x20-0x7E`，避免版本号和未覆盖英文被破坏。
- 主菜单探针仅需 54 个中文字形，能完整放入 `0xA1-0xDF` 的 63 个稳定载体中。
- 为适配 SimHei 的行高，探针包内的 `LAYOUT.TXT` 将 `MainPos` 从 `(128,390)` 调到 `(128,300)`，将通用菜单 `FontSize` 从 `22` 调到 `16`，并将 `ScreenMain` 的 `ItemSpace` 调到 `2`。

### 已确认事实

- `CScreenMain::Build()` 实际使用的首屏条目是：
  - `IDS_SINGLEPLAYER` / `IDS_CONTINUE_GAME`
  - `IDS_MULTIPLAYER` / `IDS_MULTIPLAYER_LAN`
  - `IDS_OPTIONS` / `IDS_PROFILE`
  - `IDS_EXIT`
- 主菜单相关帮助文本需要同步覆盖：`IDS_HELP_SINGLEPLAYER`、`IDS_HELP_CONTINUE_GAME`、`IDS_HELP_MULTIPLAYER`、`IDS_HELP_MULTIPLAYER_LAN`、`IDS_HELP_OPTIONS`、`IDS_HELP_PROFILE`、`IDS_HELP_EXIT`。
- `IDS_CREDITS` 在 TO2 主菜单源码中未被使用；旧候选里把 582 当成主菜单项是不准确的。
- `ScreenMain` 还会动态显示版本字符串，不能用 ASCII-only 载体方案破坏 ASCII。

### 验证结果

- `_work/tools/build_main_menu_probe.py` 成功生成主菜单 REZ。
- `_work/analysis/CRES_main_menu_probe_strings.csv` 已确认 17 个目标 ID 全部替换为高位半角载体字符串。
- `LithRez v` 确认 REZ 包内只有根目录 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_MAIN.TTF`。
- 运行截图 `_work/analysis/runtime_main_menu_probe_high_v2.png` 已确认：
  - “单人游戏、继续游戏、互联网多人游戏、局域网多人游戏、选项、玩家档案、退出”均可读。
  - 底部帮助文本“开始载入或保存单人游戏。”可读。
  - 版本号 `v1.3 - LivesForeverPlus::2009` 仍保持 ASCII 可读。
- 测试结束后已关闭 `Lithtech.exe`，未保留游戏进程。

### 问题

- 高位载体单字体最多稳定覆盖 63 个中文字形；复杂界面仍需按屏幕上下文拆包或进一步改造客户端字体管线。
- 探针里调整了主菜单布局，说明后续正式汉化不能只替换字符串，还需要为中文字体逐屏校正字号、行距和坐标。

### 下一步

- 以“可见界面上下文”为单位继续推进：
  1. 单人游戏子菜单：`ScreenSingle.cpp` 中的新游戏、载入、章节、自定义关卡及难度项。
  2. 选项主菜单：`ScreenOptions.cpp` 中显示、声音、控制、游戏、性能。
  3. 性能界面：基于源码完整收集 `ScreenPerformance.cpp` 同屏 ID，再用高位载体或拆分字体包验证。
- 每个界面先统计中文字形数量；小于等于 63 的优先使用高位载体保留 ASCII，大于 63 的再拆屏或考虑客户端改造。

## 2026-07-10 第 9 轮：单人游戏菜单上下文包

### 修改内容

- 新增单人游戏菜单构建脚本：`_work/tools/build_single_menu_probe.py`。
- 新增单人菜单编码替换表：`_work/patch/CRES_single_menu_probe_replacements.csv`。
- 新增单人菜单载体映射表：`_work/analysis/single_menu_probe_char_map.csv`。
- 新增单人菜单说明：`_work/analysis/single_menu_probe.md`。
- 生成单人菜单 REZ：`_work/build_single_menu_probe/NOLF2_CN_SINGLE_MENU_PROBE.rez`。
- 新增启动脚本：`_work/run_cn_single_menu_probe.bat`。

### 采用的技术

- 单人游戏子菜单单独作为一个“可见界面上下文”，不强行和主菜单中文字库合并。
- 该上下文覆盖 `ScreenSingle.cpp` 的主条目、返回项、难度列表和对应帮助文本。
- 使用 CP932 高位半角单字节载体 `0xA1-0xDF`，并保留 ASCII glyph，避免主菜单未翻译英文被破坏。
- 探针包内同样保留中文字体布局修正：通用菜单字号下调，`ScreenSingle` 的 `ListFontSize` 下调到 `12`。

### 已确认事实

- 单人菜单自身需要 63 个中文字形，刚好等于高位载体容量上限。
- 若把主菜单中文字形也合并进同一个高位载体字体，总字形数会达到 98，超过 `0xA1-0xDF` 容量。
- 因此“每个 REZ 覆盖多个中文界面”的做法会很快撞上 8 位字体容量；后续需要按界面拆包，或改造客户端字体/编码路径。

### 验证结果

- `_work/tools/build_single_menu_probe.py` 成功生成单人菜单 REZ。
- `_work/analysis/CRES_single_menu_probe_strings.csv` 已确认 19 个目标 ID 全部替换为高位半角载体字符串。
- `LithRez v` 确认 REZ 包内只有根目录 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_SINGLE.TTF`。
- 运行自动化已能启动该 REZ，截图 `_work/analysis/runtime_single_menu_probe_v1.png` 和 `_work/analysis/runtime_single_menu_probe_v2.png` 均显示主菜单英文保持正常，说明 ASCII 未被破坏。
- 自动化的 Enter/鼠标点击没有触发 `CMD_SINGLE_PLAYER`，所以单人子菜单的运行时截图仍待人工点击确认；这属于输入自动化问题，不是离线补丁失败。
- 用户人工运行截图 `QQ_1783690310053.png` 已确认单人子菜单运行时显示中文：
  - “单人游戏、新游戏、载入游戏、返回”可读。
  - 底部帮助文本“载入已保存的游戏。”可读。
- 测试结束后已关闭 `Lithtech.exe`，未保留游戏进程。

### 下一步

- 手动运行 `_work/run_cn_single_menu_probe.bat`，点击 `Single player`，确认单人子菜单是否显示：
  - “单人游戏、新游戏、载入游戏、章节列表、自定义关卡、返回”
  - 难度列表：“简单、普通、困难、超级间谍”
- 若单人菜单运行时确认通过，继续做 `ScreenOptions.cpp` 选项主菜单上下文。
- 若单人菜单出现缺字或半角符号，优先检查 `single_menu_probe_char_map.csv` 的 `0xA1-0xDF` 映射和 `NOLF2CN_SINGLE.TTF` cmap。

## 2026-07-10 第 10 轮：选项主菜单上下文包

### 修改内容

- 新增选项主菜单构建脚本：`_work/tools/build_options_menu_probe.py`。
- 新增选项菜单编码替换表：`_work/patch/CRES_options_menu_probe_replacements.csv`。
- 新增选项菜单载体映射表：`_work/analysis/options_menu_probe_char_map.csv`。
- 新增选项菜单说明：`_work/analysis/options_menu_probe.md`。
- 生成选项菜单 REZ：`_work/build_options_menu_probe/NOLF2_CN_OPTIONS_MENU_PROBE.rez`。
- 新增启动脚本：`_work/run_cn_options_menu_probe.bat`。

### 采用的技术

- 目标界面为 `CScreenOptions::Build()`，覆盖标题、五个子菜单入口、返回按钮和对应帮助文本。
- 继续使用 CP932 高位半角单字节载体 `0xA1-0xDF`，同时保留 ASCII glyph。
- 探针包保留中文字体布局修正：`MainPos=(128,300)`、`HelpSize=16`、通用 `FontSize=16`。

### 已确认事实

- `ScreenOptions.cpp` 实际使用的子菜单入口 ID 是：
  - `IDS_DISPLAY` / `IDS_SOUND` / `IDS_CONTROLS`
  - `IDS_GAME_OPTIONS` / `IDS_PERFORMANCE`
- `IDS_CONTROLS` 的真实 ID 是 `877`，不是早期粗查时误命中的 `IDS_CONTROLS_CROSSHAIR=613`。
- 选项主菜单上下文仅需 25 个中文字形，远低于 63 个高位载体上限。

### 验证结果

- `_work/tools/build_options_menu_probe.py` 成功生成选项菜单 REZ。
- `_work/analysis/CRES_options_menu_probe_strings.csv` 已确认 13 个目标 ID 全部替换为高位半角载体字符串。
- `LithRez v` 确认 REZ 包内只有根目录 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_OPTIONS.TTF`。
- 运行时截图仍待人工从主菜单点击 `Options` 确认；该包保留 ASCII，所以主菜单英文不会随机映射。

### 下一步

- 手动运行 `_work/run_cn_options_menu_probe.bat`，点击 `Options`，确认选项菜单是否显示：
  - “选项、显示、声音、控制、游戏、性能、返回”
  - 底部帮助文本随选中项显示中文。
- 若选项菜单确认通过，下一步进入 `ScreenPerformance.cpp`，完整收集性能界面同屏 ID 并评估是否需要拆成多个上下文。

## 2026-07-11 第 11 轮：启动菜单合并包

### 修改内容

- 新增启动菜单合并构建脚本：`_work/tools/build_startup_menus_pack.py`。
- 新增启动菜单合并替换表：`_work/patch/CRES_startup_menus_replacements.csv`。
- 新增启动菜单合并载体映射表：`_work/analysis/startup_menus_char_map.csv`。
- 新增启动菜单合并说明：`_work/analysis/startup_menus_pack.md`。
- 生成合并 REZ：`_work/build_startup_menus/NOLF2_CN_STARTUP_MENUS.rez`。
- 新增启动脚本：`_work/run_cn_startup_menus.bat`。

### 采用的技术

- 将已验证的主菜单、单人游戏菜单、选项主菜单合并为一个启动菜单包。
- 为了放入单个 `0xA1-0xDF` 高位载体字体，对帮助文本和部分标签采用更短、更一致的译法。
- 合并包继续保留 ASCII glyph，因此版本号和未覆盖英文不会被随机映射。
- 字体仍使用子集化 SimHei，避免完整字体导致老引擎崩溃。

### 覆盖范围

- 主菜单：
  - “单人游戏、继续游戏、网络联机、局域网联机、选项、档案、退出”
- 单人游戏菜单：
  - “新游戏、载入游戏、章节、自定关卡、返回”
  - 难度：“简单、普通、困难、超级”
- 选项主菜单：
  - “显示、声音、控制、游戏、性能、返回”
- 对应的短帮助文本也已纳入同一字库。

### 验证结果

- `_work/tools/build_startup_menus_pack.py` 成功生成合并 REZ。
- `_work/analysis/CRES_startup_menus_strings.csv` 已确认 45 个目标 ID 全部替换为高位半角载体字符串。
- `LithRez v` 确认 REZ 包内只有根目录 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_STARTUP.TTF`。
- 合并包共使用 53 个中文字形，低于 63 个高位载体上限。
- 运行截图 `_work/analysis/runtime_startup_menus_main_v1.png` 已确认主菜单启动页中文显示正常，版本号 `v1.3 - LivesForeverPlus::2009` 仍保持 ASCII 可读。
- 测试结束后已关闭 `Lithtech.exe`，未保留游戏进程。

### 下一步

- 推荐优先使用 `_work/run_cn_startup_menus.bat` 验证启动菜单合并包。
- 人工确认进入单人菜单和选项菜单时仍显示正常后，再继续做更深一层：
  1. 显示设置页 `ScreenDisplay.cpp`
  2. 声音设置页 `ScreenAudio.cpp`
  3. 控制设置页 `ScreenControls.cpp`
  4. 性能设置页 `ScreenPerformance.cpp`

## 2026-07-11 第 12 轮：显示设置页上下文包

### 修改内容

- 新增显示设置页构建脚本：`_work/tools/build_display_settings_pack.py`。
- 新增显示设置页替换表：`_work/patch/CRES_display_settings_replacements.csv`。
- 新增显示设置页载体映射表：`_work/analysis/display_settings_char_map.csv`。
- 新增显示设置页说明：`_work/analysis/display_settings_pack.md`。
- 生成显示设置页 REZ：`_work/build_display_settings/NOLF2_CN_DISPLAY_SETTINGS.rez`。
- 新增启动脚本：`_work/run_cn_display_settings.bat`。

### 覆盖范围

- 覆盖显示设置标题、分辨率、伽马、垂直同步、硬件光标、开关项、返回、确定、取消和硬件光标确认文本。
- 共替换 18 个 `RT_STRING` ID。
- 共使用 54 个中文字形，低于 `0xA1-0xDF` 的 63 个高位载体上限。

### 验证结果

- 构建脚本成功生成 REZ。
- `LithRez v` 确认 REZ 内只有 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_DISPLAY.TTF`。
- 运行时截图仍待人工通过 Options -> Display 验证。

## 2026-07-11 第 13 轮：声音设置页上下文包

### 修改内容

- 新增声音设置页构建脚本：`_work/tools/build_sound_settings_pack.py`。
- 新增声音设置页替换表：`_work/patch/CRES_sound_settings_replacements.csv`。
- 新增声音设置页载体映射表：`_work/analysis/sound_settings_char_map.csv`。
- 新增声音设置页说明：`_work/analysis/sound_settings_pack.md`。
- 生成声音设置页 REZ：`_work/build_sound_settings/NOLF2_CN_SOUND_SETTINGS.rez`。
- 新增启动脚本：`_work/run_cn_sound_settings.bat`。

### 覆盖范围

- 覆盖声音设置标题、音效、音乐、语音音量、音质、开关项、返回、确定、取消和高质量音频确认文本。
- 共替换 23 个 `RT_STRING` ID。
- 共使用 36 个中文字形，低于 `0xA1-0xDF` 的 63 个高位载体上限。

### 验证结果

- 构建脚本成功生成 REZ。
- `LithRez v` 确认 REZ 内只有 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_SOUND.TTF`。
- 运行时截图仍待人工通过 Options -> Sound 验证。

## 2026-07-11 第 14 轮：控制设置页上下文包

### 修改内容

- 新增控制设置页构建脚本：`_work/tools/build_controls_settings_pack.py`。
- 新增控制设置页替换表：`_work/patch/CRES_controls_settings_replacements.csv`。
- 新增控制设置页载体映射表：`_work/analysis/controls_settings_char_map.csv`。
- 新增控制设置页说明：`_work/analysis/controls_settings_pack.md`。
- 生成控制设置页 REZ：`_work/build_controls_settings/NOLF2_CN_CONTROLS_SETTINGS.rez`。
- 新增启动脚本：`_work/run_cn_controls_settings.bat`。

### 覆盖范围

- 覆盖控制设置标题、按键设置、鼠标、键盘、使用手柄、手柄设置、恢复默认、开关项、返回、确定、取消、恢复默认确认文本和无手柄提示。
- 共替换 21 个 `RT_STRING` ID。
- 共使用 51 个中文字形，低于 `0xA1-0xDF` 的 63 个高位载体上限。

### 验证结果

- `_work/tools/build_controls_settings_pack.py` 成功生成 REZ。
- `_work/analysis/CRES_controls_settings_strings.csv` 已确认 21 个目标 ID 全部替换为高位半角载体字符串。
- `LithRez v` 确认 REZ 内只有 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_CONTROLS.TTF`。
- 运行时截图仍待人工通过 Options -> Controls 验证。

## 2026-07-11 第 15 轮：性能设置页上下文包

### 修改内容

- 新增性能设置页构建脚本：`_work/tools/build_performance_settings_pack.py`。
- 新增性能设置页替换表：`_work/patch/CRES_performance_settings_replacements.csv`。
- 新增性能设置页载体映射表：`_work/analysis/performance_settings_char_map.csv`。
- 新增性能设置页说明：`_work/analysis/performance_settings_pack.md`。
- 生成性能设置页 REZ：`_work/build_performance_settings/NOLF2_CN_PERFORMANCE_SETTINGS.rez`。
- 新增启动脚本：`_work/run_cn_performance_settings.bat`。

### 覆盖范围

- 覆盖 `ScreenPerformance.cpp` 的主性能列表，以及 Display 和 Special Effects 两个内嵌列表。
- 包含性能、纹理精度、预载、显示、特效、测试、动态光照、阴影、水面凹凸、水面反射、环境凹凸、三线过滤、各向异性、曳光弹、弹壳、环境细节、效果和开关档位。
- 共替换 52 个 `RT_STRING` ID。
- 共使用 63 个中文字形，刚好达到 `0xA1-0xDF` 的高位载体上限。

### 验证结果

- `_work/tools/build_performance_settings_pack.py` 成功生成 REZ。
- `_work/analysis/CRES_performance_settings_strings.csv` 已确认 52 个目标 ID 全部替换为高位半角载体字符串。
- `LithRez v` 确认 REZ 内只有 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_PERFORMANCE.TTF`。
- 由于性能页已用满 63 个中文字形，本轮没有纳入性能测试结果和推荐长文本；后续若要汉化测试流程，需要单独拆上下文包或改客户端字体路径。
- 运行时截图仍待人工通过 Options -> Performance 验证。

## 2026-07-11 第 16 轮：游戏设置页上下文包

### 修改内容

- 新增游戏设置页构建脚本：`_work/tools/build_game_settings_pack.py`。
- 新增游戏设置页替换表：`_work/patch/CRES_game_settings_replacements.csv`。
- 新增游戏设置页载体映射表：`_work/analysis/game_settings_char_map.csv`。
- 新增游戏设置页说明：`_work/analysis/game_settings_pack.md`。
- 生成游戏设置页 REZ：`_work/build_game_settings/NOLF2_CN_GAME_SETTINGS.rez`。
- 新增启动脚本：`_work/run_cn_game_settings.bat`。

### 覆盖范围

- 覆盖 `ScreenGame.cpp` 的可见游戏设置项：准星、字幕、血腥、总是奔跑、难度、视角晃动、武器摆动、提示时间、自动换武器、载入提示、雪车轮廓和对应帮助文本。
- 难度四档“简单、普通、困难、超级”也纳入本包。
- `ScreenGame.cpp` 中 HUD 布局控件已被注释，运行路径不会显示，所以本轮不纳入 `IDS_HUDLAYOUT`。
- 共替换 34 个 `RT_STRING` ID。
- 共使用 59 个中文字形，低于 `0xA1-0xDF` 的 63 个高位载体上限。

### 验证结果

- `_work/tools/build_game_settings_pack.py` 成功生成 REZ。
- `_work/analysis/CRES_game_settings_strings.csv` 已确认 34 个目标 ID 全部替换为高位半角载体字符串。
- `LithRez v` 确认 REZ 内只有 `CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、`INTERFACE\FONTS\NOLF2CN_GAME.TTF`。
- 运行时截图仍待人工通过 Options -> Game 验证。

## 2026-07-11 第 17 轮：字幕路径和 CSHELL 代理补丁

### 背景结论

- 启动页和 Options 页的上下文包已经证明“单字节载体 + 子集字体”能显示中文。
- 但启动页和五个 Options 页的中文字形并集已经超过 63 个，剧情文本会远远超过这个规模。
- 因此后续不能继续只堆页面 REZ 包，必须进入客户端代码/二进制层面。

### 源码和二进制证据

- `CInterfaceResMgr::CreateFont` 源码里已经调用 `CreateFont(..., 33, 255)`，说明字体创建范围本来就到 `0xFF`。
- `CUIFont` 头文件明确说明字体系统主要基于 `uint8`，只改 `CUI_MAX_BITMAPFONT_CHARS` 不能让字体超过 255 项。
- `CHUDSubtitles::Show` 的字幕路径是 `LoadTempString(nStringId)` -> `m_pText->SetText(pStr)`。
- `CCharacterFX::PlayLipSyncSound` 中 `SUBTITLE_STRINGID_OFFSET` 为 `0`，所以对白声音 ID 和字幕 `RT_STRING` ID 相同。
- 原版 `Game\CSHELL.DLL` 的字幕 `LoadTempString` call site 已定位：
  - image base: `0x10000000`
  - call VA: `0x100A58C4`
  - call RVA: `0x000A58C4`
  - 原始字节：`E8 F7 0A FA FF`
  - 目标函数：`0x100463C0`

### 新增文件

- 新增对白/字幕索引脚本：`_work/tools/build_dialogue_subtitle_index.py`。
- 生成索引：`_work/analysis/dialogue_subtitle_index.csv`。
- 新增代理 DLL 源码：`_work/proxy_cshell/cshell_proxy.cpp`。
- 新增构建脚本：`_work/proxy_cshell/build_proxy.ps1`。
- 新增安装脚本：`_work/proxy_cshell/install_proxy.ps1`。
- 新增卸载脚本：`_work/proxy_cshell/uninstall_proxy.ps1`。
- 新增说明：`_work/analysis/subtitle_proxy_patch.md`。

### 代理补丁状态

- 代理 DLL 已成功编译到 `_work/proxy_cshell/bin/CSHELL.DLL`。
- 代理是 x86 DLL，基址设为 `0x50000000`，避免占用原版 `CSHELL.DLL` 的固定基址 `0x10000000`。
- 代理只导出 `SetMasterDatabase`，与原 DLL 的公开导出面一致。
- 代理静态链接 CRT，导入表只依赖 `KERNEL32.dll`。
- 已安装代理：
  - 原版 DLL 备份为 `Game\CSHELL_ORIG.DLL`。
  - 额外时间戳备份为 `Game\CSHELL.before_cn_proxy.20260711_100531.DLL`。
  - 当前 `Game\CSHELL.DLL` 是代理 DLL。
- 新增外部覆盖表示例：`_work\cn_subtitles_ascii.tsv`，当前覆盖 ID `32000` 为 ASCII 钩子测试文本。

### 验证结果

- `_work/tools/build_dialogue_subtitle_index.py` 生成 4,242 条候选字符串。
- 其中 997 条能直接追溯到 `voice\数字.wav` 引用。
- 样例确认：
  - `32000` 对应 `voice\32000.wav` 和字幕 `You won't be so happy when I sound the alarm!`
  - `32001` 对应 `voice\32001.wav` 和字幕 `Time to sound the alarm, I think!`
- 安装代理后用窗口模式启动 `Lithtech.exe`，8 秒后进程仍存活，随后手动结束。
- 这说明代理加载原 DLL 并进入启动流程没有直接崩溃。

### 下一步

- 用 `TestSubtitle 32000` 或实际触发 ID `32000` 的对白验证 `_work\cn_subtitles_ascii.tsv` 是否能在屏幕字幕中覆盖原文。
- 覆盖链路确认后，再把 ASCII 覆盖升级为中文载体字节，并为字幕做专用字体页。
- 完整剧情汉化仍需要解决动态中文字形分页，或者更深层地替换字幕渲染为纹理/宽字符路径。

## 2026-07-11 第 18 轮：中文字幕载体冒烟包

### 修改内容

- 新增中文字幕冒烟构建脚本：`_work/tools/build_subtitle_chinese_smoke_pack.py`。
- 生成中文字幕测试 REZ：`_work/build_subtitle_chinese_smoke/NOLF2_CN_SUBTITLE_CHINESE_SMOKE.rez`。
- 生成字幕专用字体：`_work/build_subtitle_chinese_smoke/stage/Interface/Fonts/NOLF2CN_SUBTITLE.TTF`。
- 生成外部覆盖表：`_work/cn_subtitles_ascii.tsv`。
- 新增启动脚本：`_work/run_subtitle_chinese_smoke.bat`。
- 新增说明：`_work/analysis/subtitle_chinese_smoke.md`。
- 新增字形映射表：`_work/analysis/subtitle_chinese_smoke_char_map.csv`。

### 覆盖内容

- 测试 ID：`32000`。
- 中文文本：`字幕覆盖测试成功。`
- 使用 9 个中文字形，映射到 raw carrier bytes `0xA1-0xA9`。
- `_work/cn_subtitles_ascii.tsv` 当前内容为二进制载体行：
  - `32000<TAB>A1 A2 A3 A4 A5 A6 A7 A8 A9`

### 字体和布局

- `NOLF2CN_SUBTITLE.TTF` 基于 `C:\Windows\Fonts\simhei.ttf` 子集生成。
- `LAYOUT.TXT` 使用 `LivesForeverPlus.rez` 提取版为基底。
- 仅改 `Font3`：
  - `FontFace3 = "SimHei"`
  - `FontFile3 = "Interface\Fonts\NOLF2CN_SUBTITLE.TTF"`
- 仅改 `[Subtitle]`：
  - `Font = 3`
  - `FontSize = 16`

### 验证结果

- `NOLF2_CN_SUBTITLE_CHINESE_SMOKE.rez` 中只有：
  - `ATTRIBUTES\LAYOUT.TXT`
  - `INTERFACE\FONTS\NOLF2CN_SUBTITLE.TTF`
- 带 `NOLF2_CN_CSHELL_PROXY.rez` 和 `NOLF2_CN_SUBTITLE_CHINESE_SMOKE.rez` 启动 10 秒，进程仍存活。
- `runtime.log` 确认代理加载并完成三个补丁：
  - `CHUDSubtitles::Show` 的 `LoadTempString` call。
  - `CHUDSubtitles::Render` 的 `Subtitles` 开关 gate。
  - `CHUDSubtitles::Update` 的主线程 hook。

### 下一步

- 运行 `_work/run_subtitle_chinese_smoke.bat` 进入关卡，确认屏幕字幕显示 `字幕覆盖测试成功。`
- 若显示正常，后续把单行测试扩展成多 ID 外部翻译表，并为每条字幕动态选择/生成载体字形页。

## 2026-07-11 第 19 轮：正式字幕代理和真实对白 batch1

### 验证结果

- 用户截图确认 `字幕覆盖测试成功。` 已在游戏内正常显示。
- 这证明以下链路已经打通：
  - REZ 覆盖 `LivesForeverPlus.rez` 里的 `CSHELL`。
  - `CHUDSubtitles::Show` 的 `LoadTempString` 调用可转向外部表。
  - 外部 raw carrier bytes 可通过字幕专用字体显示中文。

### 代理调整

- 从 `_work/proxy_cshell/cshell_proxy.cpp` 移除测试用的 `CHUDSubtitles::Update` 主线程自动触发 hook。
- 正式代理现在只保留：
  - `CHUDSubtitles::Show` 的 `LoadTempString` call hook。
  - `CHUDSubtitles::Render` 的 `Subtitles` 开关 gate 绕过。
- 重建正式代理 REZ：`_work/proxy_cshell/NOLF2_CN_CSHELL_PROXY.rez`。
- 启动冒烟确认正式代理加载后进程仍存活，日志只包含两个正式补丁。

### 真实对白 batch1

- 新增构建脚本：`_work/tools/build_dialogue_batch1_pack.py`。
- 新增真实对白 REZ：`_work/build_dialogue_batch1/NOLF2_CN_DIALOGUE_BATCH1.rez`。
- 新增启动脚本：`_work/run_dialogue_batch1.bat`。
- 新增说明：`_work/analysis/dialogue_batch1.md`。
- 新增翻译表：`_work/analysis/dialogue_batch1_translations.csv`。
- 新增字形映射表：`_work/analysis/dialogue_batch1_char_map.csv`。

### batch1 覆盖范围

- 覆盖 8 条常见警卫对白：
  - `32000`：等我拉响警报你就笑不出来了！
  - `32001`：我想该拉警报了！
  - `32002`：抓住那个间谍！
  - `32003`：帮我干掉那个间谍！
  - `32004`：有人来了，我们可没邀请他！
  - `32005`：别偷懒，帮我搜索！
  - `32006`：你看到什么可疑的东西了吗？
  - `32007`：当心危险，我的朋友！
- 共使用 55 个中文字形，低于 `0xA1-0xDF` 的 63 字形上限。
- batch1 的外部覆盖表写入 `_work/cn_subtitles_ascii.tsv`。
- `NOLF2_CN_DIALOGUE_BATCH1.rez` 中只有：
  - `ATTRIBUTES\LAYOUT.TXT`
  - `INTERFACE\FONTS\NOLF2CN_DIALOGUE1.TTF`

### 重要限制

- 继续扩展到 20 条时会达到 92 个中文字形，超过当前单个载体字体页 63 字形上限。
- 后续大规模剧情汉化需要动态字体页、多上下文包切换，或更深的渲染替换方案。

## 2026-07-11 第 20 轮：开场加载页和 Yamata-san 起始对白

### 用户修正优先级

- 当前优先级改为：
  - 先翻译最开头的 Chapter 1 加载屏幕。
  - 再翻译开场桥边 Yamata-san 给出的起始任务对白。
- 不再优先处理 `32000-32007` 那批通用警卫对白。

### 开场加载页

- 确认 `MISSIONS.TXT` 的 Mission0/Level1 使用：
  - `NameId = 2500`
  - `Level1NameId = 2551`
  - `Level1BriefingId = 2700`
  - `Level1HelpId = 3101`
  - `BriefingLayout = "LoadScreenC01"`
- 新增构建脚本：`_work/tools/build_opening_loading_pack.py`。
- 新增独立加载页 REZ：`_work/build_opening_loading/NOLF2_CN_OPENING_LOADING.rez`。
- 新增独立启动脚本：`_work/run_opening_loading_cn.bat`。
- 加载页中文译文：
  - `2500`：第1章 凯特必须死!
  - `2551`：直觉而已.
  - `2700`：UNITY派凯特到日本调查犯罪会议, 并暗中拍下与会者.\n\n先找Hatori取得情报. 他在村里等候.
  - `3101`：提示: 不知下一步时按Tab看目标. 右上罗盘会指路.
- 加载页使用 57 个中文字形，低于 `0xA1-0xDF` 的 63 字形上限。
- 注意：截图里的大号 `LOADING` 是加载页贴图，不是 CRES 字符串，本轮未改贴图。

### Yamata-san 起始对白

- 通过 `C01S01.TXT` 声音表和源码资源脚本确认桥边开场可触发对白为 `11001-11004`：
  - `11001`：Hatori-san is waiting for you somewhere in the village.
  - `11002`：Look for messages from him.
  - `11003`：Find Hatori-san. He can help you with your mission.
  - `11004`：Good luck!
- 这些台词内容是 Yamata-san 对 Cate 给出的 Hatori 接头指引，因此中文文本中会出现 Hatori/服部。
- 新增合并构建脚本：`_work/tools/build_opening_yamata_pack.py`。
- 新增合并 REZ：`_work/build_opening_yamata/NOLF2_CN_OPENING_YAMATA.rez`。
- 新增合并启动脚本：`_work/run_opening_yamata_cn.bat`。
- 合并包同时包含：
  - 打了开场加载页字符串补丁的 `CRES.DLL`。
  - 加载页字体 `NOLF2CN_LOADING.TTF`。
  - 字幕字体 `NOLF2CN_YAMATA.TTF`。
  - 同时覆盖加载页字体和字幕字体的 `LAYOUT.TXT`。
- 合并包需配合正式代理：
  - `_work/proxy_cshell/NOLF2_CN_CSHELL_PROXY.rez`
  - `_work/build_opening_yamata/NOLF2_CN_OPENING_YAMATA.rez`
- Yamata 起始对白中文译文：
  - `11001`：服部先生在村里某处等你.
  - `11002`：找找他留下的信息.
  - `11003`：找到服部先生. 他能帮你完成任务.
  - `11004`：祝你好运!
- Yamata 起始对白使用 28 个中文字形，低于 63 字形上限。
- `_work/cn_subtitles_ascii.tsv` 当前已被合并脚本写入 `11001-11004` 的 raw carrier bytes。

### 静态验证

- `NOLF2_CN_OPENING_YAMATA.rez` 列表确认包含 4 个资源：
  - `CRES.DLL`
  - `ATTRIBUTES\LAYOUT.TXT`
  - `INTERFACE\FONTS\NOLF2CN_LOADING.TTF`
  - `INTERFACE\FONTS\NOLF2CN_YAMATA.TTF`
- 读回 `CRES.DLL` 时显示半角载体字符，符合预期。
- `Format-Hex _work\cn_subtitles_ascii.tsv` 确认当前只有 `11001-11004` 四行。

### 下一步

- 运行 `_work/run_opening_yamata_cn.bat` 从开头验证：
  - Chapter 1 加载页标题/关卡名/说明/Tip 是否显示中文。
  - 进入 C01S01 后与桥边 Yamata-san 交互，确认 `11001-11004` 字幕中文显示。

## 2026-07-11 第 21 轮：全量文本提取表

### 验证结果

- 用户截图确认：
  - 开场加载页的标题、关卡名、说明、Tip 均能显示中文。
  - 进入 C01S01 后，桥边开场字幕 `11001` 可显示中文。
- 这证明当前方案足够进入批量翻译表阶段：
  - CRES 字符串可用单字节载体 + 子集字体显示中文。
  - 字幕代理可用 `_work/cn_subtitles_ascii.tsv` + 字幕字体显示中文。

### 全量提取

- 新增提取脚本：`_work/tools/extract_translation_tables.py`。
- 输出目录：`_work/translation`。
- 主输出：
  - `_work/translation/all_text_for_translation.tsv`
  - `_work/translation/all_text_for_translation.csv`
  - `_work/translation/all_text_raw_multiline.csv`
  - `_work/translation/README.md`
- 分类输出：
  - `dialogue.tsv/csv`
  - `mission.tsv/csv`
  - `mission_objective.tsv/csv`
  - `mission_failure.tsv/csv`
  - `intel.tsv/csv`
  - `menu.tsv/csv`
  - `equipment.tsv/csv`
  - `name.tsv/csv`
  - `reward.tsv/csv`
  - `other.tsv/csv`
- 第一章优先表：
  - `_work/translation/chapter01_priority.tsv`
  - `_work/translation/chapter01_priority.csv`

### 提取来源

- 主文本源：
  - `_work/source/Game/ClientRes/TO2/Lang/EN/ClientRes.rc`
  - `_work/source/Game/ClientRes/Shared/Lang/EN/ClientResShared.rc`
  - `_work/source/Game/ClientRes/TO2/ClientRes.h`
  - `_work/source/Game/ClientRes/Shared/ClientResShared.h`
- 完整性对照：
  - `_work/analysis/CRES_strings.csv`
- 合并策略：
  - 以 CRES 数字 ID 为全集，保证 6313 条运行时字符串都在表里。
  - 从源码 `.rc/.h` 追加 `IDS_*` 符号名和来源行，便于人工定位。

### 导出统计

- 总行数：6313。
- RC 字符串行：6230。
- CRES 字符串行：6313。
- 已预填中文：8 行。
- 第一章优先表：1222 行。
- 分类：
  - `dialogue`: 2773
  - `mission`: 378
  - `mission_objective`: 222
  - `mission_failure`: 32
  - `intel`: 982
  - `menu`: 917
  - `equipment`: 228
  - `name`: 18
  - `reward`: 301
  - `other`: 462

### 翻译约定

- 用户可直接填 `zh` 列。
- `all_text_for_translation.tsv/csv` 中换行已转成字面量 `\n`，便于表格软件逐行编辑。
- `all_text_raw_multiline.csv` 保留真实换行，仅用于需要查看原始段落时参考。
- 翻译时必须保留：
  - `%s`、`%d`、`%1!d!` 等格式占位符。
  - `@` 分隔符，例如情报标题和正文之间的分隔。
  - 必要的 `\n` 段落换行。

### 下一步

- 用户开始在 `_work/translation/chapter01_priority.tsv` 或 `_work/translation/dialogue.tsv` 填 `zh` 列。
- 收到翻译后的表后，下一步是写一个导入/构建脚本：
  - 读取 `zh` 列。
  - 按当前 63 字形限制分包或按章节生成字体页。
  - 生成 CRES 补丁 REZ、字幕 TSV、字幕字体和启动脚本。

## 2026-07-11 第 22 轮：CSV 插件兼容和 SCMD 判断

### CSV 插件问题

- 用户用 edit csv 插件打开 CSV 时报错：
  - `Trailing quote on quoted field is malformed on line 434`
- 定位到 `_work/translation/all_text_for_translation.csv` 第 434 行：
  - `IDS_FILTER_COOP`
  - 英文原文为 `"H.A.R.M. vs. UNITY" only`
- 该行原本使用标准 CSV 双引号转义，Python `csv.reader` 可正常读完整表。
- 但部分编辑插件对嵌套双引号兼容性差，因此调整导出脚本：
  - CSV 所有字段使用 `csv.QUOTE_ALL`。
  - 编辑用 CSV/TSV 中的英文原文双引号显示为全角 `＂`。
  - `all_text_raw_multiline.csv` 仍保留原始精确文本作为参考。
- 已重新导出 `_work/translation` 全部表。
- 当前第 434 行显示为：
  - `＂H.A.R.M. vs. UNITY＂ only`

### SCMD 是否需要翻译

- `SCMD` 相关字符串包括：
  - `SCMD login successful.`
  - `Admin already logged into SCMD.`
  - `Incorrect SCMD password.`
  - `SCMD Commands:`
  - `HELP - List all available SCMD commands.`
  - `LOGIN <password> - Logs in as SCMD Administrator.`
  - `IDS_SCMDPASSWORD`
  - `IDS_ALLOW_SCMD_COMMANDS`
- 判断：这是服务器/远程管理命令和管理员登录相关文本，不是单机剧情主线文本。
- 建议：单机剧情汉化第一阶段可跳过 SCMD；等菜单、剧情、任务、情报文本完成后再统一处理。

## 2026-07-12 第 23 轮：尖括号资源标记

### 判断

- 用户询问类似 `<CHAPTER 12 & 13 OBJECTIVES>` 的文本是否应该翻译。
- 查证来源：
  - `IDS_MISSION_OBJ_5150`
  - 同类还有 `<CHAPTER ... OBJECTIVES>`、`<CHAPTER ... REWARDS>`、`<COOPERATIVE MISSION OBJECTIVES>`。
- 其它官方语言资源中这些尖括号行也基本保持英文原样。
- 判断：整条文本都是 `<...>` 的这类项目是资源分组标记或占位符，不应作为玩家正文翻译。

### 翻译规则

- 整条都是尖括号包住的标记，例如：
  - `<CHAPTER 12 & 13 OBJECTIVES>`
  - `<CHAPTER 1 REWARDS>`
  - `<COOPERATIVE MISSION OBJECTIVES>`
  - `<empty>`
- 建议 `zh` 留空，或原样复制英文；构建时应跳过，不写入汉化补丁。
- 如果尖括号只是一句文本中的参数，例如：
  - `LOGIN <password> - Logs in as SCMD Administrator.`
- 则可翻译外部说明，但 `<password>` 必须原样保留。

### 脚本调整

- 已更新 `_work/tools/extract_translation_tables.py`：
  - 对整条匹配 `^<[^<>]+>$` 的文本标记 `status = skip`。
  - `note = angle-bracket marker or placeholder; keep original`。
- 暂未重新导出表，以免覆盖用户已经填写的 `zh` 列。

## 2026-07-12 第 24 轮：以 chapter01_priority.tsv 为准导入测试

### 用户要求

- 用户确认已翻译 `_work/translation/chapter01_priority.tsv`。
- 要求：
  - 把 `_work/translation/chapter01_priority.tsv` 之外的其它文本导出表归档或删除。
  - 后续以 `_work/translation/chapter01_priority.tsv` 为准。
  - 测试没问题后，再导出剩余所有文本为一个 TSV。

### 归档操作

- 已把 `_work/translation` 中除 `chapter01_priority.tsv` 外的所有文件移动到：
  - `_work/translation_archive/20260712_084925`
- 当前 `_work/translation` 只保留：
  - `_work/translation/chapter01_priority.tsv`

### 翻译表状态

- `chapter01_priority.tsv`：
  - 总行数：1222
  - 已填 `zh`：1201
  - 不同非 ASCII 字形：1569
- 这已经远超当前已验证的单字体页 63 字形限制，因此不能作为一个单包直接全部导入。

### 导入测试脚本

- 新增：
  - `_work/tools/build_chapter01_import_test_pack.py`
- 输出：
  - `_work/build_chapter01_import_test/NOLF2_CN_CHAPTER01_IMPORT_TEST.rez`
  - `_work/run_chapter01_import_test.bat`
  - `_work/analysis/chapter01_import_test_report.csv`
  - `_work/analysis/chapter01_import_test_char_maps.csv`
  - `_work/analysis/chapter01_import_test.md`

### 本轮导入范围

- 按不同字体槽拆分开场测试：
  - `title`: `2500`, `2551`
  - `briefing`: `2700`
  - `help`: `3101`
  - `subtitle`: `11001-11004`
- 实际导入结果：
  - included：7
  - skipped：1
  - patched CRES：3
  - subtitle TSV：4 行
- 字体页统计：
  - title：19 字形
  - help：56 字形
  - subtitle：36 字形
- `2700` 被跳过：
  - 用户译文单条 briefing 需要 77 个不同汉字。
  - 超过当前 CP932 单字节载体 `0xA1-0xDF` 的 63 字形上限。
  - 因此构建脚本保留原英文，不强行写入。

### 读回验证

- `NOLF2_CN_CHAPTER01_IMPORT_TEST.rez` 包含：
  - `CRES.DLL`
  - `ATTRIBUTES\LAYOUT.TXT`
  - `INTERFACE\FONTS\NOLF2CN_C01_TITLE.TTF`
  - `INTERFACE\FONTS\NOLF2CN_C01_HELP.TTF`
  - `INTERFACE\FONTS\NOLF2CN_C01_SUB.TTF`
- CRES 读回确认：
  - `2500` 已替换为载体字符。
  - `2551` 已替换为载体字符。
  - `3101` 已替换为载体字符。
  - `2700` 仍为英文原文。
- `_work/cn_subtitles_ascii.tsv` 当前由导入测试脚本写入 `11001-11004`。

### 下一步

- 用户运行 `_work/run_chapter01_import_test.bat` 验证：
  - 加载页标题和副标题应使用用户译文。
  - 加载页 Tip 应使用用户译文。
  - 加载页长说明仍为英文，这是预期的当前限制。
  - Yamata/Hatori 开场字幕 `11001-11004` 应使用用户译文。
- 后续要让长说明 `2700` 也进包，有两个方向：
  - 压缩该条译文到 63 个以内的不同汉字。
  - 继续推进更深的字体/渲染方案，突破单字体页 63 字形限制。

### 后续修正：扩展载体池解决 2700

- 用户指出译文没有修改空间，不能靠压缩文本解决。
- 调整 `_work/tools/build_chapter01_import_test_pack.py`：
  - 每个字体槽自动保护该槽中需要原样显示的 ASCII 字符，例如 `UNITY`、`Tab`。
  - 其余 ASCII 可打印字符 `0x21-0x7E` 加上半角区 `0xA1-0xDF` 都可作为中文载体。
  - 这样单个字体槽不再局限于 63 个半角载体。
- 重新构建结果：
  - included：8
  - skipped：0
  - patched CRES：4
  - title：19 字形
  - briefing：77 字形
  - help：56 字形
  - subtitle：36 字形
- `2700` 已成功导入，不再保持英文。
- 当前测试脚本仍是：
  - `_work/run_chapter01_import_test.bat`

### 后续修正：ASCII 扩容不能污染全局字体槽

- 用户截图验证上一版包存在严重问题：
  - 加载页出现中英混杂。
  - HUD 左下角文字被载体字形污染。
  - 存档/读档菜单英文和中文混杂。
- 根因：
  - `FontFace0-3` 是全局 UI 字体槽，会被加载屏、HUD、菜单共同使用。
  - 上一版把 ASCII 字符也作为中文载体挂到这些全局字体槽上，导致未翻译英文被错误显示成中文。
- 修正 `_work/tools/build_chapter01_import_test_pack.py`：
  - 保留原始 `FontFace0-3` 不动。
  - 新增专用字体槽：
    - `FontFace4`: 加载屏标题/关卡名。
    - `FontFace5`: 加载屏 briefing。
    - `FontFace6`: 加载屏 Tip。
    - `FontFace7`: 字幕。
  - `LoadScreenDefault` 改用 4/5/6 号字体，`Subtitle.Font` 改用 7 号字体。
  - 只有 `briefing` 槽使用 protected ASCII carriers，以容纳 `2700` 的 77 个不同汉字。
  - `title`、`help`、`subtitle` 回到半角载体，避免污染未翻译 ASCII。
- 重新构建：
  - `_work/build_chapter01_import_test/NOLF2_CN_CHAPTER01_IMPORT_TEST.rez`
  - `_work/run_chapter01_import_test.bat`
- 构建结果：
  - included：8
  - skipped：0
  - patched CRES：4
  - title：19 字形
  - briefing：77 字形
  - help：56 字形
  - subtitle：36 字形

### 运行截图反馈：鸟相关字幕缺失

- 用户截图验证：
  - 加载屏中文正常。
  - 开头 Yamata/Hatori 字幕正常。
  - HUD/存档菜单污染问题消失。
  - 射击机械八哥时出现英文字幕：`Oh, target practice with the bird. Very, very funny.`
- 定位结果：
  - 该句是 `IDS_DIALOGUE_33462`，来自 `SERVERSND.TXT`。
  - 相关 `33460-33465` 未包含在原 `chapter01_priority.tsv` 中。
- 已追加到 `_work/translation/chapter01_priority.tsv` 尾部，`zh` 留空待翻译：
  - `33460`: `Don't shoot the bloody bird.`
  - `33461`: `Please don't shoot the bird.`
  - `33462`: `Oh, target practice with the bird. Very, very funny.`
  - `33463`: `Shooting the bird is a waste of money. This is an expensive bird!`
  - `33464`: `I and the bird are here to help you. Don't shoot us!`
  - `33465`: `Oh, so glad you could find it in your heart to shoot the bloody bird.`
- 已调整 `_work/tools/build_chapter01_import_test_pack.py`：
  - `subtitle` 作用域包含 `33460-33465`。
  - 字幕专用字体槽也允许 protected ASCII carriers，因为它不再污染全局 UI 字体。
- 当前重建：
  - included：8
  - skipped：6
  - skipped 全部是 `33460-33465`，原因是 `no zh translation`。
  - 用户填好这 6 行后，重跑脚本即可导入。

### Modernizer 安装后的适配

- 用户安装 NOLF2 Modernizer 后，当前 `launchcmds.txt` 已切换为：
  - `+mod Modernizer`
  - `-rez Custom\Mods\Modernizer\MODERNIZER.REZ`
- 安装后新增/覆盖的根目录文件：
  - `JServerDir.dll`
  - `ModernizerReadme.txt`
  - `JServerInfo.txt`
  - `controls.cfg`
  - `Lithtech.exe`
  - `NOLF2Srv.exe`
  - `osslicenses.txt`
  - `SDL2.dll`
- 安装前备份：
  - `_work/backup_before_modernizer_20260712_092243`
- Modernizer REZ 内容确认：
  - 自带 `CRES.DLL`、`CSHELL.DLL`、`SRES.DLL`、`OBJECT.LTO`、`ATTRIBUTES\LAYOUT.TXT`。
  - 因此不能把旧汉化包里的旧 `CRES.DLL` 直接加载在 Modernizer 后面，否则会回退 Modernizer 的 CRES。
- 已解包：
  - `_work/extract_modernizer`
- 已调整 `_work/tools/build_chapter01_import_test_pack.py`：
  - 支持环境变量 `NOLF2_CN_MODERNIZER=1`。
  - Modernizer 模式基于 `_work/extract_modernizer\CRES.DLL` 和 `_work/extract_modernizer\ATTRIBUTES\LAYOUT.TXT` 构建。
  - Modernizer 原布局已有 `FontFace4 = FiraCode`，CN 专用字体槽改用：
    - `FontFace5`: 加载屏标题/关卡名。
    - `FontFace6`: 加载屏 briefing。
    - `FontFace7`: 加载屏 Tip。
    - `FontFace8`: 字幕。
  - Modernizer 测试包不加载旧 `CSHELL_PROXY`，先用直接 patch CRES 字幕的方式验证。
- 当前 Modernizer 专用输出：
  - `_work/build_chapter01_import_modernizer/NOLF2_CN_CHAPTER01_IMPORT_MODERNIZER.rez`
  - `_work/run_chapter01_import_modernizer.bat`
  - `_work/analysis/chapter01_import_modernizer.md`
  - `_work/analysis/chapter01_import_modernizer_report.csv`
  - `_work/analysis/chapter01_import_modernizer_char_maps.csv`
- 当前 Modernizer 构建结果：
  - included：14
  - skipped：0
  - patched CRES：14
  - title：19 字形
  - briefing：77 字形
  - help：56 字形
  - subtitle：78 字形
- 本轮包含：
  - 加载屏 `2500`, `2551`, `2700`, `3101`
  - 开头 Yamata/Hatori 字幕 `11001-11004`
  - 机械八哥射击字幕 `33460-33465`
- 下一步用户运行：
  - `_work/run_chapter01_import_modernizer.bat`

### Modernizer 运行反馈：机械八哥乱码

- 用户截图验证：
  - Modernizer 下加载屏中文正常。
  - 开头 Yamata/Hatori 字幕正常。
  - 机械八哥字幕出现严重中英混杂乱码。
- 源码定位：
  - `Speaker::DoDialogueSubtitles()` 返回 `LTTRUE`。
  - `CCharacterFX::PlayLipSyncSound()` 根据声音文件名取 `nStringId`，再调用 `g_pSubtitles->Show(nStringId, ...)`。
  - `SUBTITLE_STRINGID_OFFSET` 为 `0`。
- 乱码原因：
  - 上一版把 `33460-33465` 与 `11001-11004` 放进同一个字幕字体页。
  - 78 个不同汉字超过 63 个半角载体，所以字幕字体使用了 ASCII carriers。
  - 机械八哥路径出现英文/载体回退时，ASCII 被字幕字体重映射成汉字，造成乱码。
- 已修正稳定版：
  - `_work/tools/build_chapter01_import_test_pack.py` 默认字幕范围只包含 `11001-11004`。
  - 字幕字体不再使用 ASCII carriers，只使用半角区。
  - Modernizer 稳定包重建：
    - `_work/build_chapter01_import_modernizer/NOLF2_CN_CHAPTER01_IMPORT_MODERNIZER.rez`
    - `_work/run_chapter01_import_modernizer.bat`
  - 结果：
    - included：8
    - skipped：0
    - patched CRES：8
    - subtitle：36 字形
  - 预期：
    - 加载屏和 `11001-11004` 中文。
    - 机械八哥先回英文，但不再乱码。
- 已新增机械八哥探针：
  - 环境变量：`NOLF2_CN_BIRD_PROBE=1`
  - 输出：
    - `_work/build_chapter01_bird_probe_modernizer/NOLF2_CN_CHAPTER01_BIRD_PROBE_MODERNIZER.rez`
    - `_work/run_chapter01_bird_probe_modernizer.bat`
  - 结果：
    - included：10
    - patched CRES：10
    - subtitle：51 字形
  - 预期：
    - 机械八哥 `33460-33465` 如果读 CRES，会显示中文。
    - `11001-11004` 在此探针中保持英文。
  - 这个探针用于判断后续是“按场景分页”即可，还是必须做 Modernizer 版字幕钩子。

### 2026-07-12 Modernizer 路线确认与剩余文本导出

- 用户已测试：
  - 稳定 Modernizer 包：加载屏、Yamata/Hatori 字幕正常。
  - 机械八哥探针包：`33460-33465` 机械八哥字幕正常。
- 结论：
  - 当前“CRES 文本改写 + 专用字体槽 + 按范围/场景控制字形页”的技术路线可用。
  - 后续全量导入时不能把全游戏字幕塞进同一个字体页，需要按菜单/加载屏/字幕批次控制 carrier 字形范围，或再做字幕钩子分页。
- 已生成剩余文本 TSV：
  - `_work/translation/remaining_after_chapter01.tsv`
  - `_work/translation/remaining_after_chapter01_summary.md`
- 导出规则：
  - 来源为 Modernizer CRES 字符串表：`_work/analysis/CRES_modernizer_strings.csv`。
  - 排除 `_work/translation/chapter01_priority.tsv` 中已有的 1228 个 ID。
  - 剩余导出 5127 行。
  - 表头仍为：`id`, `symbols`, `category`, `source`, `source_lines`, `english`, `zh`, `status`, `note`。
  - 为避免表格插件的 quoted-field 报错，TSV 使用纯制表符分隔：
    - 无普通双引号 `"`。
    - 英文参考中的 `"` 显示为全角 `＂`。
    - 换行显示为字面量 `\n`。
- 导出校验：
  - 物理行：5128。
  - 数据行：5127。
  - 列数异常：0。
  - 普通双引号行：0。
  - 重复 ID：0。
  - 与 `chapter01_priority.tsv` 重叠 ID：0。
- 分类计数：
  - `dialogue`: 1854
  - `equipment`: 228
  - `intel`: 922
  - `menu`: 951
  - `mission`: 221
  - `mission_failure`: 2
  - `mission_objective`: 202
  - `other`: 410
  - `reward`: 291
  - `server_admin`: 46

### 2026-07-12 用户完成剩余 TSV 后的导入状态

- 用户已完成 `_work/translation/remaining_after_chapter01.tsv` 的主要翻译。
- 校验结果：
  - `chapter01_priority.tsv`: 1228 行，1207 行有 `zh`。
  - `remaining_after_chapter01.tsv`: 5127 行，4213 行有 `zh`。
  - 合并后唯一 ID：6355 行，5420 行有 `zh`。
  - 合并后中文非 ASCII 不重复字形：2345 个。
- 当前字体载体方案限制：
  - 半角高位载体安全容量约 63 个字形。
  - 如果使用 protected ASCII carriers，单页最多约 157 个字形。
  - protected ASCII carriers 不能用于全局字体，否则未翻译/回退英文会被同一字体映射成乱码。
  - 全量 CRES 一次性导入为单页字体不可行；需要动态字体分页或 Modernizer 版字幕/UI 钩子。
- 已生成可行性报告：
  - `_work/analysis/full_import_feasibility.md`
  - `_work/analysis/full_import_feasibility_by_category.csv`
- 已重建当前安全 Modernizer 起步测试包：
  - `_work/build_chapter01_import_modernizer/NOLF2_CN_CHAPTER01_IMPORT_MODERNIZER.rez`
  - 包含资源：`CRES.DLL`、`ATTRIBUTES\LAYOUT.TXT`、4 个 `NOLF2CN_C01_*.TTF`。
  - 当前包含已验证范围：
    - 加载屏 `2500`, `2551`, `2700`, `3101`
    - 开头 Yamata/Hatori 字幕 `11001-11004`
  - 字幕槽只使用半角载体，保留英文 fallback 可读，避免全局/后续对白乱码。
- 已重建机械八哥探针包：
  - `_work/build_chapter01_bird_probe_modernizer/NOLF2_CN_CHAPTER01_BIRD_PROBE_MODERNIZER.rez`
  - 仅用于验证机械八哥字幕（原始日志末尾截断，后续路线已于 2026-07-12 转向运行时自绘渲染器）。
�