# deepin-kwin 与原版 KDE KWin 差异审计报告

- 审计对象：`CharOfString/deepin-kwin` 的 master，HEAD 是 `664808264f5a49c306834565d0e0c631a8feda27`。这个 fork 和 linuxdeepin/deepin-kwin 官方 master 完全一致
- **对照基准："原版 KWin"，即 KDE 官方 KWin**。数据来自 GitHub 上的 KDE 官方镜像 `https://github.com/KDE/kwin`，因为本环境的网络代理不允许访问 invent.kde.org
  - 基线 tag：`v5.27.2`，对应 commit `09def9d0713954da8d72ae1e1f9be71d6622b9af`
  - 拿来比对的 KDE 分支：`master`（截至 2026-11-04）、`Plasma/5.27`、`Plasma/6.0`。v5.27.2 之后共有 **9,203** 个非 merge 提交
  - 拿来比对的 KDE 旧版本：KWin `v5.18.8`、`v5.21.5`、`v5.22.5`、`v5.26.5`，KDE kdecoration `v5.27.5`，kwayland-server `v5.20.5`、`v5.23.5`，kwayland `v5.90.0`
- 审计日期：2026-11-05
- 逐提交清单：[`commits-vs-kwin.csv`](./commits-vs-kwin.csv)，共 961 行，每行包含完整 commit id、分类、对应的 KDE commit 和说明

---

## 一、结论

**963 个提交里，只有 1 个是原版 KWin 本身，1 个是空提交，剩下 961 个全部是相对原版 KWin 的改动。**

| 分类 | 提交数 | 说明 |
|---|---:|---|
| 空提交 | 1 | `aa0a579806de69e35ccebcacc2f3d5d729f91137`，Gerrit 建仓时生成 |
| **原版 KWin** | **1** | `fb932a112a3fe280ec23f43c6d88d24af211e6cd`（init kwin from upstream v5.27.2）。源码和 KDE 官方 v5.27.2 **逐字节一致**，只多了一个 `debian/` 目录（32 个文件、2,210 行），是 Debian 的打包文件，不属于 KDE |
| A. 完整移植自 KDE KWin 后续版本 | 16 | 代码和 KDE 官方某个提交基本相同，内容算是原版 KWin 的，只是由 deepin 的人提交 |
| B. 部分含 KDE 代码，主体是 deepin 修改 | 11 | 部分参考或搬用了 KDE 代码 |
| C. 引入其他 KDE 库的源码 | 1 | `91e541428a` 把 KDE 的 kdecoration 5.27.5 源码整体放进了仓库。它是 KDE 官方代码，但不属于 KWin |
| **D. deepin 自研修改** | **933** | 原版 KWin 里没有，由 deepin/统信写的代码，另有少量翻译机器人和外部贡献者的提交 |

**按"不是原版 KWin"来算：961 个提交都不是原版 KWin。其中纯 deepin 自己的改动是 933 个，另外 11 个以 deepin 修改为主、夹带了少量 KDE 代码。**

### 按代码行数算（当前 HEAD，逐行 git blame）

统计范围：当前代码树中除翻译（po、.ts）、`debian/` 和图片以外的全部文本文件，共 **390,421 行**。

| 代码来源 | 行数 | 占比 |
|---|---:|---:|
| 原版 KWin 5.27.2（来自 `fb932a1`，没被改动过） | 334,717 | **85.73%** |
| deepin 自研（D 类） | 47,818 | **12.25%** |
| KDE kdecoration 5.27.5（C 类，整体引入的 KDE 另一个库） | 6,174 | 1.58% |
| B 类（部分含 KDE 代码） | 939 | 0.24% |
| A 类（移植自 KDE KWin 后续版本） | 773 | 0.20% |

直接和 KDE v5.27.2 做 diff：除去翻译和 `debian/` 后，共有 **903 个文件**改动，**新增 56,161 行，删除 4,734 行**。按文件数算，新增 600 个文件、修改 879 个、删除 17 个，这三个数字包含翻译文件。

### deepin 改动最集中的目录（deepin 行数占该目录总行数的比例）

| 目录 | 总行数 | deepin 行数 | 占比 |
|---|---:|---:|---:|
| `src/xdgshellv6window.*`、`src/windowstyle/`、`src/splitscreen/` | 5,310 | 5,310 | 100%（全新增） |
| `src/qml` | 1,440 | 1,145 | 79.5% |
| `cmake` | 1,421 | 845 | 59.5% |
| `src/wayland_server.cpp` | 1,239 | 404 | 32.6% |
| `src/plugins`（含 deepin-chameleon 窗口装饰） | 21,304 | 6,335 | 29.7% |
| `src/composite.cpp` | 1,544 | 427 | 27.7% |
| `src/workspace.cpp` | 4,137 | 1,065 | 25.7% |
| `src/effects`（含多任务视图、分屏等特效） | 45,773 | 10,342 | 22.6% |
| `src/wayland`（含 dde 私有 Wayland 协议） | 60,321 | 8,480 | 14.1% |
| `src/backends` | 26,599 | 2,414 | 9.1% |

---

## 二、deepin 相对原版 KWin 新增的主要内容

- **特效**：多任务视图 `multitaskview`、分屏 `splitscreen` / `splitswap`、`maximizeex`、`scissorwindow`（窗口圆角裁剪）、`alttabthumbnaillist`、`squash`、`fadingpopups` 等
- **分屏管理**：`src/splitscreen/`
- **窗口装饰和样式**：deepin-chameleon 窗口装饰插件（`b91d404`），`src/windowstyle/`（圆角、阴影）
- **deepin 私有 Wayland 协议**：ddeshell、ddeseat、ddekeyboard、ddekvm、ddesecurity、dderestrict、clientmanagement、strut、remote_access，以及 outputdevice_v1 / outputmanagement_v1
- **找回 KDE 已删除的功能**：XRender 合成后端（`a449132`，KDE 在 5.23 左右删除）、xdg-shell-unstable-v6（`8c31004`）
- **为老环境做的兼容**：2023-05 前后约 40 个提交把依赖降到 Qt 5.11、cmake 3.13、debhelper 12（UOS V20 环境），并加入一批 Find*.cmake 模块（`fb5d941`）
- **为 deepin V25 做的移植**：2025 年 Qt6 / KF6 构建支持（`39f55f1` 等）
- **其它**：日志系统改用 journald、截图和录屏限制、DRM 和多屏的大量修复、Alt+Tab 缩略图等

---

## 三、方法说明

1. **确认基线**：比较 `fb932a1` 和 KDE `v5.27.2` 的源码树，结果是只多了 `debian/` 目录。
2. **补丁指纹精确匹配**：对 961 个 deepin 提交和 9,203 个 KDE 提交分别计算 `git patch-id --stable`，完全相同的有 5 个。
3. **代码行比对**：取每个提交新增的、去掉空白后不少于 25 个字符的代码行，先排除 v5.27.2 里本来就有的行，再和 KDE 提交的新增行求交集，找出重合度高的候选。
4. **提交说明比对**：deepin 提交说明里的句子和 KDE 提交说明重合的，也列为候选。
5. **人工复核**：对 30 多个候选逐个比较两边的 diff，判断是完整移植、部分参考，还是 deepin 员工把自己的代码也提交给了 KDE（第 7.3 节）。
6. **溯源旧代码**：把 deepin 新增的行和 KWin 旧版本、kwayland-server、kwayland 比对，找出从 KDE 老代码复制回来的部分。
7. **逐行归属**：对当前 HEAD 做 `git blame`，统计每一行最后由哪个提交写入。

**局限**：
- 改写幅度很大的移植提交可能识别不出来。
- 不足 25 个字符的短行不参与比对。
- 只比对了上面列出的 KDE 分支和 tag。
- 领域分类是按文件路径自动归的。
- deepin 的私有 Wayland 协议实现和 KDE 旧版 kwayland 有 15%–30% 的行相同。复查后发现这些主要是许可证头和样板代码，说明是照着 KDE 的写法写的，不是复制，所以仍算 D 类。


## 四、A 类：完整移植自 KDE KWin 后续版本（16 个）

这些提交的代码和 KDE KWin 5.27.2 之后的某个官方提交基本相同，也就是 deepin 把原版 KWin 的新修复搬了回来。按"原版 KWin"的标准，它们的**内容是官方的**，只是由 deepin 的人提交。

| # | deepin commit | 日期 | 提交人 | 标题 | 对应 KDE 官方 commit | KDE 作者 / 日期 | 是否注明出处 |
|---:|---|---|---|---|---|---|---|
| 1 | `b724380c842903f43cdf86d6ef2553153a461056` | 2023-09-15 | huangzeming | fix: Simplify backing store buffer handling | `665e05d711f814b04845abc5833b8899242efd54`<br>plugins/qpa: Simplify backing store buffer handling | Vlad Zahorodnii / 2023-07-03 | 照抄 KDE 提交说明，未给链接 |
| 2 | `d338f610cdc3630424a6a8199b67573493f49ef1` | 2023-11-21 | liuzheng | fix: screencasting: on memfd, skip the QImage step | `5ac5251674d910e3402885144dc6ce9479a960eb`<br>screencasting: on memfd, skip the QImage step | Aleix Pol / 2023-03-03 | 照抄 KDE 提交说明，未给链接 |
| 3 | `39b05d97d15577a9499fffce5235b19d5e559559` | 2023-11-24 | huangzeming | plugins/blur: Rewrite it | `36960d8572221c696289232d0ab55c02ecfa5324`<br>plugins/blur: Rewrite it | Vlad Zahorodnii / 2023-08-10 | 照抄 KDE 标题和说明，未给链接、未保留作者 |
| 4 | `3765ef1b5e81a7eb00de29021adafd3aab40b24a` | 2024-05-07 | zhangyaning | fix: update keyboard focus when grabbing popup closed. | `9241252c6765738d9b5f0ac31f5e08b5ac1a5c3e`<br>update keyboard focus when grabbing popup closed | Liu Jie / 2023-04-26 | 未注明来源（原作者为麒麟 kylinos.cn 开发者） |
| 5 | `2bd16957fecd00aacae12c771d0136829947e7ac` | 2023-03-21 | Vlad Zahorodnii | feat: Refactor compositing teardown in X11Window and Unmanaged | `017906541b26307152442bf7ea14f03f4c427fad`<br>Refactor compositing teardown in X11Window and Unmanaged | Vlad Zahorodnii / 2023-03-21 | 保留了 KDE 原作者署名 |
| 6 | `49ce09fee4f2b657b036fb301e3d44e2d4c67fe7` | 2024-05-23 | dingzhixuan | fix: xdg popup position incorrect | `c1eca8019012a7c3e2e4b2832250d1b63a55ec66`<br>wayland: Remove implicit slide constraint adjustment | Vlad Zahorodnii / 2023-03-02 | 附了上游补丁标题 |
| 7 | `8da3eae7cf70a92763cd3b549bc989158d6a4a67` | 2024-07-16 | yangtian | fix: stop setting XCURSOR_SIZE | `0edf264947ef72341913daf472abc1dd37713d5c`<br>Stop setting XCURSOR_THEME and XCURSOR_SIZE envvars | Vlad Zahorodnii / 2023-04-04 | 未注明来源 |
| 8 | `fc18056d5e1d86f0d07c3da98ebe57618f0909eb` | 2024-07-10 | dingzhixuan | fix: glmark2 will change size when plugin and plugout screen | `008e5e5a108e310c352ff5ba9003dab15130077f`<br>workspace: reduce placeholder output size | Xaver Hugl / 2023-11-21 | 仅写 "by upstream" |
| 9 | `8b10d5c2afe4e916f69c47f7159c4d805ef4f2ec` | 2024-08-29 | dingzhixuan | fix: Improve Workspace::outputAt() | `a9ad59b32a67aced8cf2baf6182ad3ded94a5af7`<br>Improve Workspace::outputAt() | Vlad Zahorodnii / 2023-03-25 | 标题照搬，未给链接 |
| 10 | `d7193c643a4cba4edefeb78045c28af7ab3263a8` | 2025-02-18 | Lu YaNing | fix: Check GPU active state before handling change events | `8c412c229b9edfc9ed6e8de4c5ae1975e7dbcfc7`<br>backends/drm: track device active status per GPU | Xaver Hugl / 2023-04-27 | 给了 KDE commit 链接 |
| 11 | `57e48d350773b4edd47a4fd5243d043cc9f97fa4` | 2025-03-19 | rewine | fix: QQuickRenderControl can only sync/render when beginFrame() has been called | `9ea839eb03e7b81b01c4877ef4ca421b7c35e0c1`<br>libkwineffects: Add missing QQuickRenderControl::{beginFrame,endFrame} calls | Vlad Zahorodnii / 2023-02-25 | 未注明来源 |
| 12 | `a2e2fd539292255483cacc9e9b3b84f19fc35254` | 2025-03-21 | rewine | chore: do not call RHI QQuickRenderControl methods when using software rendering | `8c149bb6b711c00e0fa3cf2f3d06a09dcc57a1e4`<br>effects: Do not call RHI QQuickRenderControl methods when using software rendering | David Edmundson / 2024-06-04 | 给了 KDE commit 链接 |
| 13 | `f22b24202e159a3f526402224fd2823f333471b2` | 2025-03-21 | rewine | fix: adapt offscreenquickview to Qt6 changes | `ced95f536cf4119e2c3a7579636fe27290d94486`<br>Adapt offscreenquickview to Qt6 changes | David Edmundson / 2023-06-28 | 给了 KDE commit 链接 |
| 14 | `7a721b5603feb4646accbd1d24437d4213940445` | 2025-03-24 | Lu YaNing | fix: crash when hotplugging screens by updating window outputs | `02fbeeae78188c6e7aa4dd90c582faef1d678c01`<br>Make Workspace::desktopResized() reassign outputs of uninitialized windows | Vlad Zahorodnii / 2024-07-22 | 给了 KDE commit 链接 |
| 15 | `3291806e1f4260c0ed6ae2268b7266df362c2a8f` | 2025-04-18 | Lu YaNing | fix: Window disappears after un-maximizing in mirror mode | `dadf3f3a9774e930a50d002bb9f0daf2a806ac40`<br>window: move geometry restore to new output on electric border maximize | Xaver Hugl / 2024-09-17 | 给了 KDE commit 链接 |
| 16 | `09da202b2900630a06b9f2efbe4de2ba1953d503` | 2025-04-21 | YaoBing Xiao | chroe: increase GPU reset timeout again | `985f0edf886ba1d7a3061c65142fa66c04965e40`<br>platformsupport/scenes/openglbackend: increase GPU reset timeout again | Xaver Hugl / 2023-11-23 | 给了 KDE MR 链接 |

## 五、B 类：部分含 KDE 代码，主体是 deepin 修改（11 个）

| # | deepin commit | 日期 | 提交人 | 标题 | 相关 KDE commit | 说明 |
|---:|---|---|---|---|---|---|
| 1 | `75db2e63a571cb93fe546d184b20516c3e05d0e9` | 2023-11-10 | Chaojiang Luo | fix: Do not restart wayland compositing when config change | `49b84ddf8c21bc6abf6ccaad268fefa48aaee414`<br>Avoid destroying effects if animation speed changes（Vlad Zahorodnii） | 第 1 部分照搬 KDE（9/27 行） |
| 2 | `18d63bad13aa244eb9944c1ba69c993c0fa28b1e` | 2023-11-21 | liuzheng | fix: Fix the black screen issue with gles screencast | `e125f48ec1d64bd2ea8bbf1a3edd0439fc9b5276`<br>screencast: Base the frame skippin on max_framerate（Marco Martin） | 说明中引用 KDE 39ee72f3，部分代码相同 |
| 3 | `a363f443ae660428c7478f9621506a30667e0d2b` | 2023-11-23 | wangyu | fix: drag the icon in the Dock application area to display the titlebar window | `bcdceae2242eb2d9acbd89c12d58f68a1ad53f90`<br>Clean up how server side decorations are forced（Vlad Zahorodnii） | 9 行与 KDE 的 SSD 重构相同 |
| 4 | `2d3e05ce3d743b5a8b20e0872992fbdc2cba2de8` | 2024-01-25 | Lu YaNing | feat: protocol: add wl_pointer.warp event | `8a2d77e0a56f9e0917b8041cc65a87ee42f1958c`<br>Revert "wayland: Send wl_pointer leave before data_device enter"（David Edmundson） | 参考 KDE MR 3592/4480 与 wayland MR 340 自行实现 |
| 5 | `4568aae44b377b2572e84569813506fd37284d45` | 2024-08-28 | huangzhenyan | fix: sunlogin's image has flipped | `d02cf12f139e0f6adc380a57f2b9725e6ce8ebd0`<br>opengl: don't crash in GLTexture::toImage with OpenGL ES（Xaver Hugl） | 约 18/31 行与 KDE 相同 |
| 6 | `7a95a898992acf87504ad9eff1269cfd0c653fc5` | 2024-09-04 | huangzeming | fix: some crash | `276baa7dd592c2f9b272ffbbbfce2f4ba0cdb779`<br>Fix null dereference in Workspace::workspaceEvent()（Vlad Zahorodnii） | 3 处修复中 events.cpp 那处来自 KDE |
| 7 | `9f3f9a48c7257da29ee8729e45666c5e5fc1c62d` | 2025-03-20 | rewine | fix: outline.qml not work with qt6 | `569e0f8f6474e30fae3a2c5edb82d4424f0f2178`<br>Port to KSvg（Marco Martin） | Qt6 outline.qml，少量取自 KDE |
| 8 | `a78e4bae4c61533618e38afde907743314510eea` | 2025-04-14 | rewine | fix: kpackage install namespace in kf6 | `fe935c0dc86585991a205e0961547bcc5210f14d`<br>Adjust to new KPackage namespace for structures（Alexander Lohnau） | KF6 包路径调整，10/18 行与 KDE 相同 |
| 9 | `e6e3492dea0fee2c47f563873bd76cbdc0918da1` | 2025-04-14 | rewine | fix: adapt tabbox qml to qt6 | `3ca1fc800d7f7d8b36a3ec5436f644439c6f0146`<br>Make "Thumbnail Grid" the default Task Switcher and make it live here（Nate Graham） | Qt6 版 Alt+Tab QML，约 120/375 行取自 KDE |
| 10 | `416c04d5dfb360ba717d49130f2fe529cea2add0` | 2025-04-24 | rewine | chore(effects): rename all effects same with uos 20 | `6f622d3348f70203da8e910ce41eba4509d28255`<br>effects: Adapt to kcoreaddons_add_plugin plugin id changes（Vlad Zahorodnii） | 特效改名，命名方式与 KDE 同一改动一致（69/93 行） |
| 11 | `664808264f5a49c306834565d0e0c631a8feda27` | 2025-09-25 | Felix Yan | fix: build against Qt 6.9+ | `358def4ae30c8f896f891c4387ec45a0539f8148`<br>Rename QGenericUnixServices to QDesktopUnixServices（Carl Schwan） | 外部作者 Felix Yan 独立实现，与 KDE 的同类修复思路相同 |

## 六、C 类：引入其他 KDE 库源码（1 个）

| deepin commit | 日期 | 提交人 | 标题 | 改动 | 说明 |
|---|---|---|---|---|---|
| `91e541428a1de0744b9cd5a1d0627052e4d5b871` | 2025-03-11 | rewine | fix(v25): build vendor kdecoration2 on qt6 | +11995/-1844 | 引入 KDE 另一个库 kdecoration 5.27.5 源码（非 KWin），并做 Qt6/改名等约 140 行适配。已与 KDE 官方 kdecoration v5.27.5 逐文件比对：源码目录只有约 +142/-129 行差异，都是构建和 Qt6 适配 |

## 七、D 类：deepin 自研修改（933 个）

### 7.1 按功能领域统计

| 领域 | 提交数 | 增删行数（提交时） |
|---|---:|---:|
| 其它核心 (workspace/window 等) | 260 | 9141 |
| 构建/打包 | 172 | 5632 |
| 显示后端/DRM/多屏 | 94 | 5613 |
| 特效 | 84 | 6719 |
| Wayland 协议/服务端 (含 dde 私有协议) | 77 | 14043 |
| 多任务视图 multitaskview | 54 | 7470 |
| 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | 48 | 18804 |
| 分屏 splitscreen | 46 | 6238 |
| X11/Xwayland | 40 | 919 |
| Alt+Tab 切换器 | 22 | 975 |
| 输入 | 21 | 470 |
| 翻译 | 10 | 82485 |
| 录屏/截图/远程 | 4 | 85 |
| 测试 | 1 | 4 |

> 领域是按每个提交改动最多的文件路径自动归类的，仅供参考。"翻译"类的行数很大，是因为包含大量 .ts 翻译文件。

### 7.2 当前代码里留存行数最多的 deepin 提交（前 30）

"留存行数"指在当前 HEAD 中，`git blame` 仍归到这个提交名下的代码行数（不含翻译、打包和图片）。

| # | commit | 日期 | 作者 | 标题 | 留存行数 |
|---:|---|---|---|---|---:|
| 1 | `b91d4042605fae31127a1aa239fab6bacfb85592` | 2023-05-22 | Chaojiang Luo | feat: add decoration deepin-chameleon | 4994 |
| 2 | `b5e7de713b3042db192185e8a2e2310b0c9dab68` | 2023-09-05 | huangzeming | feat: add multitaskview | 4873 |
| 3 | `8c3100413b45d7a5b6ce13b41467d67b09a05028` | 2024-04-09 | dingzhixuan | fix: add xdg-shell-unstable-v6 | 4615 |
| 4 | `a44913259444357ec8a0713841c5fc032a1e6f30` | 2023-11-08 | xinbo wang | feat: add XRender backend support. | 1636 |
| 5 | `5fda0a90b0a57582604bc30b17000b99332d3084` | 2023-08-30 | zhangyu | feat: reconstruct split screen | 1225 |
| 6 | `9f1b334fa5e0d40e6a217b96ab26c3ca970d3fb7` | 2023-09-13 | Chaojiang Luo | feat: Add output device v1 support | 1074 |
| 7 | `7fc7e1f8044fbc9cf77ad80939dd020d77c20690` | 2022-04-08 | Chaojiang Luo | feat: add ddeseat protocol server implementation | 1010 |
| 8 | `39f55f1405fedc2a144d3902eb9dd106cc49e1c8` | 2025-03-03 | rewine | chore: support build with qt6 | 924 |
| 9 | `6229ee1931a27698edab7b668959d927e7f8f741` | 2022-10-09 | Chaojiang Luo | feat: Add dde security protection | 898 |
| 10 | `fb5d941cccdbd7d11ee224c9f651ccdf22cd4608` | 2023-05-05 | Chaojiang Luo | feat: Add cmake module support | 858 |
| 11 | `56600400d8d38536e76cd174087ffaa1172a3ede` | 2024-02-20 | zhangyu | fix: reconstruct window shadow | 843 |
| 12 | `f540eae1f37284f0472b6abe8a11a3630c90bf8e` | 2023-09-06 | xinbo wang | fead: add prohibit screenshot interface | 755 |
| 13 | `ebba44871654426eb0325509770d74a7044e2376` | 2024-03-14 | xinbo wang | feat: add kvm interface | 752 |
| 14 | `db5978574a283836b659158884ee400387469941` | 2024-01-04 | yangtian | feat: animation effect | 722 |
| 15 | `81b13d53ad698b4e5747342b40cab79cdbdbe07d` | 2023-09-05 | Chaojiang Luo | feat: Add remote access support | 649 |
| 16 | `b46ea1925fa062c098cead7f6dae8b66c81dc3ea` | 2022-04-01 | Chaojiang Luo | feat: Add ddeshell server implementation | 632 |
| 17 | `b4a9cf01cf0bf3bb910ca0a70f25d1ecc37c0577` | 2024-05-31 | huangzeming | feat: present thumbnail in tabbox | 570 |
| 18 | `1eb64176c2ef3196d6f5e237e8c49d32e545fc1d` | 2023-10-10 | zhang yu | feat: swap split window effect | 480 |
| 19 | `5e0a06bcc61d472c2ef2de36923aefc644403377` | 2023-09-12 | xinbo wang | fix: tabbox can't show UI. | 440 |
| 20 | `63258401c48d813453ff7079ff4a09bc1d755b0d` | 2023-05-25 | Chaojiang Luo | feat: Add effects scissorwindow | 428 |
| 21 | `1445a9be5ae577934a500f68190e2eac13f69d06` | 2024-03-25 | zhangyu | fix: add effect rules | 366 |
| 22 | `5aa62685fcbd49f33c23e4fdcda8a15de895d729` | 2023-09-05 | huangzeming | feat: add split menu interface | 362 |
| 23 | `7ef5d3e1a377ba4b134354c03719d5c3b81def60` | 2022-04-08 | ut004146 | fix: add strut protocol for supporting fashion mode | 335 |
| 24 | `2ef71691d81f7d7dfc00b5111eb29520aa98b344` | 2022-04-02 | ut004146 | feat: add clinet-management protocol | 295 |
| 25 | `81547fc351c4a0a7e4e3294806e1a6ab62e85bce` | 2025-03-19 | rewine | chore: make it build on qt5 again | 291 |
| 26 | `75a019cb04b77b5e0c4206e9a4a32687d569d529` | 2024-04-15 | zhangyu | fix: maxi effect | 289 |
| 27 | `f3fb544f5b7b42aee2d65fd2faa5af9a207400fd` | 2024-11-28 | huangzhenyan | fix: eyeprotection effect blink when dpms turns on | 278 |
| 28 | `0169f1e6265dbf914295b3fe091937654c2572a8` | 2025-04-17 | rewine | chore: reuse compliant | 263 |
| 29 | `ca4cd90038f8535f50078b6541acfc18b33c172f` | 2023-10-21 | xinbo wang | fix: when touch to move client,kwin must need client send events. | 262 |
| 30 | `77e6a3d3dea343ccc8c9d8723421ba4afcc9bf03` | 2024-10-17 | huangzhenyan | refactor: refactor drm blob: mode, gamma, ctm | 261 |

### 7.3 需要特别说明的 D 类提交

| commit | 日期 | 作者 | 标题 | 说明 |
|---|---|---|---|---|
| `fb5d941cccdbd7d11ee224c9f651ccdf22cd4608` | 2023-05-05 | Chaojiang Luo | feat: Add cmake module support | 66% 的有效代码行可在 KWin 5.18–5.26 旧版本中找到（旧版 cmake 模块） |
| `354102ebb968af4b57344940d4bb3268ef624391` | 2023-11-14 | Chaojiang Luo | fix: fix displaylink error | deepin 员工所写，同一改动也已提交并被 KDE 上游接受（KDE 对应提交 `1502fa9ab1f470877b466a4bb56dc32524696644`） |
| `a44913259444357ec8a0713841c5fc032a1e6f30` | 2023-11-08 | xinbo wang | feat: add XRender backend support. | 41% 的有效代码行来自 KWin 旧版本（5.22 及以前被 KDE 删除的 XRender 合成后端） |
| `823e26c3a5160e8c89e0c681aaeb0b6c568f4a97` | 2024-01-29 | luochaojiang | Revert "fix: remove some code about KWinKScreenIntegration and setting default display mode." | 回退 deepin 自己的删除，恢复 KWin 5.27.2 原有的 KScreen 集成代码 |
| `4992d94a9477313aaad07a6ed6205ade8279f06a` | 2024-02-26 | Lu YaNing | fix:  Revert "Xwayland:Update pointer state before ending the drag" | 回退（删除）一个 KDE 上游修复 |
| `8c3100413b45d7a5b6ce13b41467d67b09a05028` | 2024-04-09 | dingzhixuan | fix: add xdg-shell-unstable-v6 | 57% 的有效代码行与 KWin 5.27.2 现有 xdgshellwindow 代码相同（复制后改为 v6 协议） |
| `5871ecec40c4fa4e0aabd67f484868657f3e6b4c` | 2024-10-18 | huangzeming | fix: Avoid creating Unmanaged for reparented window | deepin 员工所写，同一改动也已提交并被 KDE 上游接受（KDE 对应提交 `2f4005cd67f23570c5aff3c330d5c69bc0a8af2f`） |
| `f3ef20f3b32f0443693599af90e5dabb3038be0e` | 2024-11-01 | Chaojiang Luo | backends/x11: fix colormap leak | deepin 员工所写，同一改动也已提交并被 KDE 上游接受（KDE 对应提交 `1747518f81d46db0230cb434f673bcedf565de8f`） |
| `c1767868987747b2e0edfb733a16e4c9e1749ae1` | 2024-11-29 | huangzhenyan | fix: delay done event after all changes are emitted | deepin 员工所写，同一改动也已提交并被 KDE 上游接受（KDE 对应提交 `5b5045a4022b0f572b888861bae4adf081969653`） |

---

## 附录：全部 961 个改动提交逐条清单（按时间顺序）

分类：A = 完整移植 KDE，B = 部分含 KDE 代码，C = 引入其他 KDE 库，D = deepin 自研。完整 40 位 commit id 和更多字段见 [`commits-vs-kwin.csv`](./commits-vs-kwin.csv)。

| # | commit | 日期 | 作者 | 类 | 领域 | +/- | 标题 |
|---:|---|---|---|:-:|---|---|---|
| 1 | `033587a4b790` | 2023-05-04 | Chaojiang Luo | D | 构建/打包 | +5/-5 | feat: download build tool deps |
| 2 | `1130419cb140` | 2023-05-04 | Chaojiang Luo | D | 构建/打包 | +8/-8 | feat: degrade Qt build deps |
| 3 | `f91264aaf813` | 2023-05-04 | Chaojiang Luo | D | 构建/打包 | +32/-32 | feat: degrade kde frameworks deps |
| 4 | `6f882023f685` | 2023-05-04 | Chaojiang Luo | D | 构建/打包 | +2/-2 | feat: degrade pipewire build deps |
| 5 | `4ffc14db37cf` | 2023-05-04 | Chaojiang Luo | D | 构建/打包 | +2/-2 | feat: change EGL build deps |
| 6 | `114dad053323` | 2023-05-05 | Chaojiang Luo | D | 构建/打包 | +1/-1 | feat: replace libxcb-util-dev with libxcb-util0-dev |
| 7 | `fb5d941cccdb` | 2023-05-05 | Chaojiang Luo | D | 构建/打包 | +859/-6 | feat: Add cmake module support |
| 8 | `234c88c13a1d` | 2023-05-05 | Chaojiang Luo | D | 构建/打包 | +297/-195 | feat: Adapt to qt 5.11 interface and linked lib |
| 9 | `46d6a0395420` | 2023-05-05 | Chaojiang Luo | D | 构建/打包 | +4/-4 | feat: link X11 without X11 workspace |
| 10 | `b31a2f49d12c` | 2023-05-05 | Chaojiang Luo | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +14/-16 | feat: replace Qt::endl with \n |
| 11 | `29a1e7a9a4a7` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +6/-5 | feat: replace QRecursiveMutex with QMutex |
| 12 | `be02ee926ef4` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +46/-1 | feat: Add missing head file memory |
| 13 | `1c4d10db3d9c` | 2023-05-05 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +14/-14 | feat: adapt cxx std::span |
| 14 | `43f440e1bb6f` | 2023-05-05 | Chaojiang Luo | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +2/-2 | feat: replace Qt::SkipEmptyParts with QString::SkipEmptyParts |
| 15 | `e9a866f29715` | 2023-05-05 | Chaojiang Luo | D | 特效 | +5/-5 | feat: adapt QStringView |
| 16 | `be48e66bccd5` | 2023-05-05 | Chaojiang Luo | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +52/-1 | feat: replace qmlRegisterAnonymousType with qmlRegisterType |
| 17 | `eedbdbae13cc` | 2023-05-05 | Chaojiang Luo | D | 特效 | +1/-1 | feat: replace convertTo with convertToFormat |
| 18 | `95e1e6bc0481` | 2023-05-05 | Chaojiang Luo | D | 特效 | +2/-3 | feat: adapt QList<QByteArray> |
| 19 | `6fbebfda44d2` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +1/-1 | feat: replace widget->screen with QGuiApplication::primaryScreen |
| 20 | `afac85bc4a5f` | 2023-05-05 | Chaojiang Luo | D | 特效 | +1/-1 | feat: redefine LEDs with typedef QFlags |
| 21 | `1b03f292f0fe` | 2023-05-05 | Chaojiang Luo | D | 输入 | +24/-12 | feat: adapt QWheelEvent |
| 22 | `ed010901863a` | 2023-05-05 | Chaojiang Luo | D | 特效 | +1/-1 | feat: fix std::map has not contains |
| 23 | `d4a544d9b969` | 2023-05-05 | Chaojiang Luo | D | 特效 | +1/-1 | feat:remove consteval |
| 24 | `6b8c56d5861d` | 2023-05-05 | Chaojiang Luo | D | 特效 | +7/-0 | feat: add missing head file |
| 25 | `629850c7fb88` | 2023-05-05 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +1/-1 | feat: replace Q_NAMESPACE_EXPORT with Q_NAMESPACE |
| 26 | `1a483bd23be9` | 2023-05-05 | Chaojiang Luo | D | 输入 | +1/-1 | feat: fix QString has no interface isUpper |
| 27 | `211f58c0a3f4` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +2/-0 | feat: add default constructor |
| 28 | `560fc04705c5` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +1/-1 | feat: fix cbegin and cend not found |
| 29 | `c609e2573655` | 2023-05-05 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +6/-6 | feat: change Qt::hex and Qt::dec to std |
| 30 | `b68a1353fc65` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +4/-4 | feat: fix QSet init error |
| 31 | `ca35f5dc7dd3` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +8/-3 | feat: fix vector and pair traverse |
| 32 | `05d8aa214f9e` | 2023-05-05 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +1/-1 | feat: replace loadRelaxed with load |
| 33 | `6cac9d823f6f` | 2023-05-05 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +5/-1 | feat: fix QHash init issue |
| 34 | `d0675279b57b` | 2023-05-05 | Chaojiang Luo | D | 录屏/截图/远程 | +6/-4 | feat: fix QImage format not supported |
| 35 | `9d21b6e7ef49` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +4/-1 | feat: fix Q_DISABLE_COPY_MOVE not found |
| 36 | `75938e790af4` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +2/-1 | feat: adapt AbstractSystemPoller_iid |
| 37 | `c4252a416eb9` | 2023-05-05 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +58/-0 | feat: remove throwError |
| 38 | `4a0ea22ba742` | 2023-05-06 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +5/-0 | feat:replace createTextureFromNativeObject with createTextureFromId |
| 39 | `935c17b2215b` | 2023-05-06 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +58/-8 | feat: adapt qpa QWindowSystemInterface interfaces |
| 40 | `b19e655cf1b4` | 2023-05-06 | Chaojiang Luo | D | 录屏/截图/远程 | +7/-1 | feat: adapt screencast |
| 41 | `6c3d5818c5d3` | 2023-05-06 | Chaojiang Luo | D | X11/Xwayland | +2/-0 | feat: delete Qt::AA_DisableSessionManager on 5.11 |
| 42 | `16d550401902` | 2022-04-06 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +82/-2 | feat: Add new plasma shell role server implementation |
| 43 | `52198fb91f49` | 2022-04-06 | ut004146 | D | 其它核心 (workspace/window 等) | +36/-0 | fix: add send window id in wayland server |
| 44 | `7c4d50a975b0` | 2022-04-07 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +86/-1 | feat: Add brightness wayland server implementation |
| 45 | `2ef71691d81f` | 2022-04-02 | ut004146 | D | Wayland 协议/服务端 (含 dde 私有协议) | +396/-0 | feat: add clinet-management protocol |
| 46 | `b46ea1925fa0` | 2022-04-01 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +641/-3 | feat: Add ddeshell server implementation |
| 47 | `7ef5d3e1a377` | 2022-04-08 | ut004146 | D | Wayland 协议/服务端 (含 dde 私有协议) | +344/-4 | fix: add strut protocol for supporting fashion mode |
| 48 | `7fc7e1f8044f` | 2022-04-08 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +1075/-0 | feat: add ddeseat protocol server implementation |
| 49 | `6e066f372ff9` | 2022-04-07 | ut004146 | D | Wayland 协议/服务端 (含 dde 私有协议) | +102/-0 | feat: add fakeinput axis event for capture |
| 50 | `fc87aef75deb` | 2023-05-18 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +10/-0 | feat: Add env to enable interfaces blacklist |
| 51 | `5d4b4383c598` | 2023-05-19 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +87/-2 | feat: support csd client use splitmenu |
| 52 | `b91d4042605f` | 2023-05-22 | Chaojiang Luo | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +6790/-0 | feat: add decoration deepin-chameleon |
| 53 | `53d1a9fba970` | 2023-05-24 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +47/-5 | feat: Add compatibility for windowId |
| 54 | `afb0b44093f5` | 2022-05-30 | Chaojiang Luo | D | 特效 | +156/-4 | feat: Add screenshot extend interface |
| 55 | `63258401c48d` | 2023-05-25 | Chaojiang Luo | D | 特效 | +528/-0 | feat: Add effects scissorwindow |
| 56 | `3923eef9c045` | 2023-07-11 | diguoliang | D | X11/Xwayland | +27/-0 | fix: the issue of slow dock startup |
| 57 | `b423fcefed1b` | 2023-07-13 | Chaojiang Luo | D | 特效 | +22/-2 | fix: Add scissor shader to render |
| 58 | `7ef13bb7de30` | 2023-07-14 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +1/-1 | fix: dispatch focus to onscreendisplay |
| 59 | `05ae7fe6a5f6` | 2023-07-14 | diguoliang | D | X11/Xwayland | +8/-0 | fix: the problem of minimizing DTK windows not working |
| 60 | `8516e11aac6e` | 2020-10-19 | luochaojiang | D | 其它核心 (workspace/window 等) | +16/-0 | feat: kwin:Reset position set flag if the window has been moved by seat |
| 61 | `93d571191a27` | 2023-05-06 | Chaojiang Luo | D | 构建/打包 | +7/-3 | workaround: fix ECM_GLOBAL_FIND_VERSION error |
| 62 | `34a3370f5525` | 2023-05-06 | Chaojiang Luo | D | 特效 | +1/-2 | workaround: delete qml setInitialProperties and createWithInitialProperties |
| 63 | `2881a8c9650d` | 2023-05-06 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +7/-4 | workaround: delete qmlRegisterSingletonInstance |
| 64 | `80693f55eabb` | 2023-05-06 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +3/-0 | workaround: delete QProcess::splitCommand |
| 65 | `f4a853e369e9` | 2023-05-12 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +133/-0 | workaround: Add QScopeGuard define in kwin |
| 66 | `2da503d5cd90` | 2023-06-13 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +9/-3 | workaround: fix dock and lunch position issue |
| 67 | `c37febbf7fe2` | 2023-08-23 | Chaojiang Luo | D | 构建/打包 | +6/-6 | fix: degrade graphics libs version |
| 68 | `715e51c01521` | 2023-08-23 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +228/-0 | feat: Adapt libdrm v99 |
| 69 | `bd9b0cb495cb` | 2023-08-23 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +37/-2 | feat: adapt libinput v1.12 |
| 70 | `af7036017949` | 2023-08-24 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +91/-64 | feat: send event to each client |
| 71 | `b032979ba71f` | 2023-07-14 | Chaojiang Luo | D | 特效 | +5/-4 | workaround: set default cornerRadius for all window |
| 72 | `d3704c6d20ed` | 2023-08-16 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +102/-1 | feat: Add ctm feature |
| 73 | `3b5f580805f2` | 2023-08-24 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +104/-0 | feat: adapt gbm missing interfaces |
| 74 | `589f20f39ee4` | 2023-08-24 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +99/-44 | fix: add drm support higher version |
| 75 | `69f2b7d407e3` | 2023-08-24 | Chaojiang Luo | D | 构建/打包 | +22/-0 | fix: add libinput support higher version |
| 76 | `5cdaf7be77eb` | 2023-08-25 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +8/-7 | fix: Fix gbm_bo_map can not found on mali platform |
| 77 | `ed8b305709fa` | 2023-08-28 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +20/-9 | fix: using mmap instand of gbm_bo_map |
| 78 | `ee7faea2a7c9` | 2023-08-30 | Chaojiang Luo | D | 构建/打包 | +145/-92 | fix: Fix autotests and tests build error |
| 79 | `6229ee1931a2` | 2022-10-09 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +911/-33 | feat: Add dde security protection |
| 80 | `1f9588f0de28` | 2022-10-24 | Chaojiang Luo | D | X11/Xwayland | +155/-9 | feat: init dde security protection |
| 81 | `5fda0a90b0a5` | 2023-08-30 | zhangyu | D | 分屏 splitscreen | +1472/-8 | feat: reconstruct split screen |
| 82 | `857e1168cdd8` | 2023-09-05 | zhangyu | D | 分屏 splitscreen | +43/-125 | feat: arrange split dev |
| 83 | `81b13d53ad69` | 2023-09-05 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +739/-0 | feat: Add remote access support |
| 84 | `b5e7de713b30` | 2023-09-05 | huangzeming | D | 多任务视图 multitaskview | +5669/-15 | feat: add multitaskview |
| 85 | `5aa62685fcbd` | 2023-09-05 | huangzeming | D | 分屏 splitscreen | +945/-98 | feat: add split menu interface |
| 86 | `fc919b1985af` | 2023-09-05 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +47/-92 | feat: arrange split cursor |
| 87 | `59a33d2e46e3` | 2023-09-05 | Chaojiang Luo | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.12-deepin12 |
| 88 | `91e6dabab404` | 2023-09-06 | zhang yu | D | 输入 | +15/-13 | fix: temp remove split cursor in wayland |
| 89 | `a237ab64ac1c` | 2023-09-06 | zhang yu | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.13-deepin13 |
| 90 | `f540eae1f372` | 2023-09-06 | xinbo wang | D | Wayland 协议/服务端 (含 dde 私有协议) | +913/-1 | fead: add prohibit screenshot interface |
| 91 | `0cc40241535a` | 2023-09-08 | zhang yu | D | 特效 | +27/-2 | feat: add interface about show minimized window in effects |
| 92 | `b6dbf4e9e493` | 2023-09-08 | zhang yu | D | 分屏 splitscreen | +27/-32 | feat: use tilemanage to resize window size |
| 93 | `605f9027b7f9` | 2023-09-11 | zhang yu | D | 输入 | +14/-11 | feat: solve split cursor in wayland |
| 94 | `e02b3b56b399` | 2023-09-09 | zhang yu | D | 其它核心 (workspace/window 等) | +12/-0 | feat: reset split window area |
| 95 | `b201a1c87b54` | 2023-09-07 | huangzeming | D | 多任务视图 multitaskview | +2/-0 | fix: dock did not respond to click event in multitaskview on x11 |
| 96 | `49ceae29915d` | 2023-09-08 | huangzeming | D | Wayland 协议/服务端 (含 dde 私有协议) | +12/-155 | fix: failed to show split menu on wayland |
| 97 | `9579da22c09b` | 2023-09-11 | zhang yu | D | 其它核心 (workspace/window 等) | +12/-10 | feat: temp disable 1/4 split and custom split |
| 98 | `656cf3b59c06` | 2023-09-11 | zhang yu | D | 其它核心 (workspace/window 等) | +2/-35 | feat: solve the issue of split window to split again from preview |
| 99 | `7a7e566157bd` | 2023-09-11 | xinbo wang | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.14-deepin14 |
| 100 | `5760e668fec7` | 2023-09-12 | xinbo wang | D | 特效 | +2/-3 | fix: change the dock preview effect in protected window. |
| 101 | `5e0a06bcc61d` | 2023-09-12 | xinbo wang | D | Alt+Tab 切换器 | +453/-0 | fix: tabbox can't show UI. |
| 102 | `39e97a952c29` | 2023-09-12 | zhang yu | D | 特效 | +15/-5 | fix: optimize screen info and add extended judgment |
| 103 | `55c48e93b30f` | 2023-09-13 | zhang yu | D | 特效 | +39/-0 | fix: solve the watermark display in multitaskview and splitscreen |
| 104 | `0032427ba76f` | 2023-09-14 | zhang yu | D | 其它核心 (workspace/window 等) | +52/-5 | fix: the problem of Win+D not being able to hide above window |
| 105 | `2debc40275a0` | 2023-09-14 | zhang yu | D | 其它核心 (workspace/window 等) | +1/-1 | fix: the problem of roll desktops when super + ->/<- |
| 106 | `1473e5781cd9` | 2023-09-14 | huangzeming | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +27/-5 | fix: press and hold maximize button, splitmenu disappears after button released |
| 107 | `c74999043402` | 2023-09-08 | huangzeming | D | 多任务视图 multitaskview | +8/-6 | fix: functions of close and top button are opposite in multitaskview |
| 108 | `162680391f63` | 2023-09-08 | huangzeming | D | 多任务视图 multitaskview | +5/-4 | fix: windows and add button were shown uncorrectly at the resolution of 1024*768 in multitaskview |
| 109 | `1b40837c6a78` | 2023-09-14 | huangzeming | D | 构建/打包 | +15/-0 | chore: Update version number for 5.27.2.15-deepin15 |
| 110 | `32d6c425e3a0` | 2023-09-15 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +1/-1 | feat: Disable DRM modifiers by default |
| 111 | `ce3ada6dee6a` | 2023-09-15 | Chaojiang Luo | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.16-deepin16 |
| 112 | `ca57fecad087` | 2023-09-15 | xinbo wang | D | 显示后端/DRM/多屏 | +181/-54 | fix: wayland can't use prohibit screenshot interface. |
| 113 | `f0f04a0bdcc7` | 2023-09-18 | zhang yu | D | 分屏 splitscreen | +2/-591 | fix: solve leave without hide after long press maxi button show splitmenu |
| 114 | `9f1b334fa5e0` | 2023-09-13 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +1211/-0 | feat: Add output device v1 support |
| 115 | `51b2236634be` | 2023-09-19 | Chaojiang Luo | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.17-deepin17 |
| 116 | `d946e50d11b5` | 2023-09-14 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +3/-2 | fix: cancel the limit of icon surface role for temporary. |
| 117 | `054bc28cbad3` | 2023-09-18 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +6/-0 | fix: when OutputConfiguration applied,reset value of applying. |
| 118 | `d312e94bf44f` | 2023-09-19 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +25/-0 | fix: Add remote access data lost |
| 119 | `c3c1ab742d63` | 2023-09-19 | Chaojiang Luo | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.18-deepin18 |
| 120 | `533727ceafcb` | 2023-09-19 | huangzeming | D | 多任务视图 multitaskview | +127/-69 | fix: multitaskview caused crash when switched 2d to 3d |
| 121 | `4059e31a75a7` | 2023-09-19 | huangzeming | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.19-deepin19 |
| 122 | `332328250c50` | 2023-09-20 | zhang yu | D | 其它核心 (workspace/window 等) | +41/-2 | fix: solve the split bar fillets |
| 123 | `6aceee29fe1d` | 2023-09-20 | zhang yu | D | 分屏 splitscreen | +102/-12 | fix: solve the split window switch screen and desktop at the same time |
| 124 | `1416576005a9` | 2023-09-21 | zhang yu | D | 分屏 splitscreen | +25/-2 | fix: kwin crash when out screen in wayland |
| 125 | `f7184bc931fb` | 2023-09-24 | zhang yu | D | 分屏 splitscreen | +32/-12 | fix: update minimized window in split manager |
| 126 | `16c36a00ebf7` | 2023-09-24 | zhang yu | D | 分屏 splitscreen | +3/-1 | fix: update fullscreen window in split manager |
| 127 | `32bf6910eda8` | 2023-09-25 | zhangyu | D | 分屏 splitscreen | +14/-1 | fix: solve the split windows order |
| 128 | `37d83df04ec7` | 2023-09-27 | zhangyu | D | 分屏 splitscreen | +18/-5 | fix: solve the split bar opacity in wayland |
| 129 | `ed5e7aab95ee` | 2023-09-26 | zhangyu | D | 分屏 splitscreen | +1/-1 | fix: solve the preview postion in multi screen in wayland |
| 130 | `6f20d1066f6f` | 2023-09-22 | zhang yu | D | 特效 | +20/-8 | fix: cancle the un resizeable window display in splitpreview |
| 131 | `d6a19125cf41` | 2023-09-27 | xinbo wang | D | 其它核心 (workspace/window 等) | +6/-0 | fix: Avoid crash with on alt+tab(tabbox) view on close it current window |
| 132 | `c9003cd97d80` | 2023-09-27 | xinbo wang | D | 其它核心 (workspace/window 等) | +4/-0 | fix: Save and load current desktop |
| 133 | `8a7cc983b667` | 2023-09-27 | xinbo wang | D | X11/Xwayland | +7/-0 | fix: Click on the parent window of the modal dialog box to play the prompt sound. |
| 134 | `73c959b54601` | 2023-09-27 | xinbo wang | D | X11/Xwayland | +8/-2 | fix: normal window with above hint is able to cover dock |
| 135 | `a952e10305c8` | 2023-09-27 | xinbo wang | D | 其它核心 (workspace/window 等) | +4/-1 | fix: dialog can show or hide with parent window when destop is toggled by win+d |
| 136 | `6a7351151352` | 2023-09-28 | xinbo wang | D | 其它核心 (workspace/window 等) | +49/-0 | fix: disable handle gesture events with kwin when locking screen |
| 137 | `b657c85da7d2` | 2023-09-28 | xinbo wang | D | 特效 | +1/-1 | fix: Change the screen shot file to jpg format to improve performance |
| 138 | `b91e03c71820` | 2023-10-07 | xinbo wang | D | 其它核心 (workspace/window 等) | +5/-1 | fix: empty caption show caption suffix. |
| 139 | `e5166f35191e` | 2023-10-07 | xinbo wang | D | 输入 | +1/-0 | fix: send mouse release event to client after moving window. |
| 140 | `ee6a52e6fb8a` | 2023-10-07 | xinbo wang | D | 其它核心 (workspace/window 等) | +7/-0 | fix: Realize the function of setOnAllDesktops in ddeshell protocol. |
| 141 | `a6db32612160` | 2023-10-07 | xinbo wang | D | 输入 | +1/-1 | fix: fix the problem that application cann't accept Poweroff event |
| 142 | `7012027ec9d6` | 2023-10-08 | zhang yu | D | 其它核心 (workspace/window 等) | +2/-0 | fix: active window frome split menu |
| 143 | `52a05026cc36` | 2023-09-26 | huangzeming | D | 分屏 splitscreen | +300/-20 | fix: tooltips were shown uncorrectly in splitmenu |
| 144 | `9e5136959e1d` | 2023-09-20 | huangzeming | D | 多任务视图 multitaskview | +9/-35 | fix: only the windows on selected workspace were shown in multitaskview |
| 145 | `c606dd509ac1` | 2023-09-25 | huangzeming | D | 多任务视图 multitaskview | +2/-0 | fix: monitor screen changed signals in multitaskview |
| 146 | `25ae7fa90082` | 2023-09-22 | huangzeming | D | 多任务视图 multitaskview | +2/-7 | fix: fix adding workspace shortcut in multitaskview |
| 147 | `997d1af86e76` | 2023-10-07 | zhangyu | D | 其它核心 (workspace/window 等) | +26/-7 | fix: update the window hover color in split preview effect |
| 148 | `89606261d61d` | 2023-10-08 | zhangyu | D | 分屏 splitscreen | +46/-11 | fix: optimize split bar in wayland |
| 149 | `b724380c8429` | 2023-09-15 | huangzeming | A | 其它核心 (workspace/window 等) | +23/-21 | fix: Simplify backing store buffer handling |
| 150 | `840159ab71f3` | 2023-09-25 | huangzeming | D | 特效 | +54/-79 | fix: add animation while switching workspace |
| 151 | `08562084bb9d` | 2023-09-20 | huangzeming | D | 多任务视图 multitaskview | +3/-3 | fix: deleting workspace caused crash in duplicate mode in multitaskview |
| 152 | `1f819dbc4888` | 2023-10-08 | zhang yu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.20 |
| 153 | `60f4bd52da36` | 2023-10-08 | zhang yu | D | 分屏 splitscreen | +20/-300 | Revert "fix: tooltips were shown uncorrectly in splitmenu" |
| 154 | `8166112ac2ea` | 2023-10-08 | huangzeming | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.21-deepin21 |
| 155 | `1eb64176c2ef` | 2023-10-10 | zhang yu | D | 分屏 splitscreen | +515/-7 | feat: swap split window effect |
| 156 | `1196c5a2e159` | 2023-10-11 | Chaojiang Luo | D | 构建/打包 | +2/-0 | fix: declare replaces of dde-kwin |
| 157 | `aa32cf6be2f9` | 2023-09-26 | huangzeming | D | 其它核心 (workspace/window 等) | +34/-3 | fix: lock screen shortcut didn't respond in multitaskview |
| 158 | `620eda883a8d` | 2023-10-10 | huangzeming | D | 多任务视图 multitaskview | +70/-17 | feat: support motion effect switch in multitaskview |
| 159 | `6fc167c4a426` | 2023-10-11 | zhijie geng | D | X11/Xwayland | +13/-3 | feat: boot usmserver |
| 160 | `3732ed8018f7` | 2023-10-11 | zhang yu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.22 |
| 161 | `69150cf0fb4d` | 2023-09-22 | huangzeming | D | 多任务视图 multitaskview | +16/-8 | fix: add window background hover in multitaskview |
| 162 | `07f7d3b0e828` | 2023-10-12 | huangzeming | D | 多任务视图 multitaskview | +1/-28 | fix: make color of hover frame in multitaskview consistent with active color |
| 163 | `899b956c37e2` | 2023-10-10 | huangzeming | D | 多任务视图 multitaskview | +5/-1 | fix: several shortcuts didn't respond in multitaskview |
| 164 | `8c43adcc82f5` | 2023-10-11 | huangzeming | D | 分屏 splitscreen | +302/-20 | fix: tooltips were shown uncorrectly in splitmenu |
| 165 | `29eb5524a92c` | 2023-10-13 | zhang yu | D | 分屏 splitscreen | +1/-1 | fix: active the maximize window frome split menu |
| 166 | `aba466a447e8` | 2023-10-13 | zhang yu | D | 特效 | +3/-2 | fix: split window don't show rounded corners |
| 167 | `03c1a0615efe` | 2023-10-13 | zhang yu | D | 其它核心 (workspace/window 等) | +38/-8 | fix: repair the split trigger effect color |
| 168 | `1c8d3dbda00b` | 2023-10-13 | huangzeming | D | 构建/打包 | +13/-0 | chore: Update version number for 5.27.2.23-deepin23 |
| 169 | `37fd3ca8ee93` | 2023-10-13 | zhang yu | D | 其它核心 (workspace/window 等) | +4/-3 | fix: active the maximize window frome split menu |
| 170 | `65d1f083cef7` | 2023-10-13 | zhang yu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.24 |
| 171 | `025b72062334` | 2023-10-17 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +31/-3 | fix: Add HuWei partial update support |
| 172 | `0cfddae79fd9` | 2023-10-17 | huangzeming | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.25 |
| 173 | `584444d59d8f` | 2023-10-16 | xinbo wang | D | 特效 | +13/-0 | fix: after reboot,the magiclamp effect invalid. |
| 174 | `c9cbe2790384` | 2023-10-18 | xinbo wang | D | Alt+Tab 切换器 | +16/-3 | feat: activate minimized if not composited |
| 175 | `08f85a1b8ae1` | 2023-10-18 | xinbo wang | D | Alt+Tab 切换器 | +13/-0 | fix: quick switch window and long press the shortcut key |
| 176 | `d7dc9da1462a` | 2023-10-18 | xinbo wang | D | Alt+Tab 切换器 | +17/-2 | fix: alt+tab can't keep minimized window minimized in 2D mode |
| 177 | `7165c6353294` | 2023-10-18 | xinbo wang | D | Alt+Tab 切换器 | +13/-0 | fix: when tabbox in 2D,keep first client minisize. |
| 178 | `1f4d395255c1` | 2023-10-18 | xinbo wang | D | Alt+Tab 切换器 | +31/-0 | fix: when tabbox in 2D,make all client keep old minimized. |
| 179 | `c1368d498615` | 2023-10-18 | xinbo wang | D | 其它核心 (workspace/window 等) | +1/-1 | fix: 按下caps键后，再按Alt+tab后切换窗口的顺序变为了反向 |
| 180 | `26827d874a1b` | 2023-10-18 | xinbo wang | D | 特效 | +2/-1 | fix: the bug about alt + ~ show non same window |
| 181 | `01df5a98fd04` | 2023-10-19 | xinbo wang | D | Alt+Tab 切换器 | +127/-49 | feat: alt+tab需求开发 |
| 182 | `df2f14d143e8` | 2023-10-17 | zhangyu | D | Wayland 协议/服务端 (含 dde 私有协议) | +64/-30 | fix: dtk split menu protocol |
| 183 | `487e552eaa08` | 2023-10-17 | zhangyu | D | 其它核心 (workspace/window 等) | +7/-1 | fix: add window minimum judgment in wayland |
| 184 | `c7bc95853c2e` | 2023-10-18 | zhang yu | D | 其它核心 (workspace/window 等) | +10/-0 | fix: disable launcher shortcut in effects |
| 185 | `1960923ba6e4` | 2023-10-17 | huangzeming | D | 多任务视图 multitaskview | +6/-0 | fix: windows were still painted while showing desktop in multitaskview |
| 186 | `dd3e0ece6789` | 2023-10-19 | huangzeming | D | 其它核心 (workspace/window 等) | +77/-45 | fix: add window fool-proofing |
| 187 | `2710c9a3dfbc` | 2023-10-20 | xinbo wang | D | 显示后端/DRM/多屏 | +30/-0 | feat: Add interface to disable multi-screens |
| 188 | `5452f61ef4a1` | 2023-10-20 | xinbo wang | D | 其它核心 (workspace/window 等) | +33/-1 | fix: Fix focus error in lock screen |
| 189 | `ac7c1115ea9c` | 2023-10-20 | xinbo wang | D | 输入 | +49/-1 | fix: add long touch press show right menu. |
| 190 | `59a5d8aadfe4` | 2023-10-20 | xinbo wang | D | 其它核心 (workspace/window 等) | +6/-2 | fix: show cursor when event is actually from libinput device. |
| 191 | `384a379a6bbd` | 2023-10-20 | xinbo wang | D | 输入 | +1/-1 | fix: Update value of LONGTOUCHPRESSIGNOREMOTION is 5,same as x11. |
| 192 | `9f477d514e9a` | 2023-10-20 | xinbo wang | D | 输入 | +1/-1 | fix: touch screen can't show user action menu. |
| 193 | `6c18da6c22f8` | 2023-10-20 | xinbo wang | D | 其它核心 (workspace/window 等) | +32/-0 | feat: add dbus interface setEnableTouchLongPress and isEnableTouchLongPress. |
| 194 | `58a9b2a1d059` | 2023-10-20 | wangboyu | D | 构建/打包 | +27/-0 | chore: Update version number for 5.27.2.26 |
| 195 | `ca4cd90038f8` | 2023-10-21 | xinbo wang | D | 其它核心 (workspace/window 等) | +266/-1 | fix: when touch to move client,kwin must need client send events. |
| 196 | `f60184d9617a` | 2023-10-23 | wangboyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.27 |
| 197 | `24742b186cdb` | 2023-10-20 | huangzeming | D | 多任务视图 multitaskview | +114/-38 | fix: drag tips and translations in multitaskview |
| 198 | `8a9737405dd2` | 2023-10-27 | huangzeming | D | 多任务视图 multitaskview | +1/-1 | fix: opening multitaskview caused crash while screen recorder is running |
| 199 | `7628075a1051` | 2023-10-27 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +7/-4 | fix: screen hot plug can not restore |
| 200 | `39b584075a77` | 2023-10-23 | zhijie geng | D | 构建/打包 | +0/-1 | fix: KDE system settings appear in the launcher when      updating from 1060 to 1070 |
| 201 | `c339dc357806` | 2023-10-27 | huangzeming | D | 构建/打包 | +0/-19 | fix: remove CAP_SYS_RESOURCE=+ep from kwin-wayland.postinst |
| 202 | `58c861078e92` | 2023-10-27 | xinbo wang | D | 其它核心 (workspace/window 等) | +21/-41 | fix: touchpad can't Execute corresponding functions |
| 203 | `585f40b134a3` | 2023-10-26 | zhangyu | D | 其它核心 (workspace/window 等) | +42/-29 | fix: 2d don't swap split window in x11 |
| 204 | `afb775007a5f` | 2023-10-23 | zhangyu | D | 分屏 splitscreen | +1/-1 | fix: repair the swap effect lag |
| 205 | `351795f9258f` | 2023-10-24 | zhang yu | D | 其它核心 (workspace/window 等) | +51/-6 | fix: repair the watermark display in alt+tab |
| 206 | `e19622a535d0` | 2023-10-25 | zhangyu | D | 其它核心 (workspace/window 等) | +13/-2 | fix: solve the menu don't dispaly in wayland |
| 207 | `4a3911857496` | 2023-10-29 | zhangyu | D | 其它核心 (workspace/window 等) | +247/-19 | feat: make the window hover fillet config in split preview and multitask view |
| 208 | `5a3ec8bc4eb3` | 2023-10-29 | zhangyu | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.28 |
| 209 | `01b3aa65de28` | 2023-10-30 | zhangyu | D | 其它核心 (workspace/window 等) | +34/-44 | fix: optimize about radius acquisition |
| 210 | `17325d43cac4` | 2023-10-30 | zhangyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.2.27.29 |
| 211 | `32e9a78697e7` | 2023-10-30 | zhangyu | D | 构建/打包 | +1/-1 | chore: Update version number for 5.27.2.29 |
| 212 | `8c1a8e31e6a0` | 2023-11-01 | liuzheng | D | 显示后端/DRM/多屏 | +14/-0 | fix: fix the issue where the mouse wheel cannot be used |
| 213 | `8164d8277b5e` | 2023-11-01 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +9/-3 | fix: reset screen dpms off at unplug |
| 214 | `994c260c3ba6` | 2023-10-31 | xinbo wang | D | 多任务视图 multitaskview | +1/-8 | fix: touchpad can't exit multitaskview in wayland. |
| 215 | `b68188cd29e4` | 2023-11-01 | xinbo wang | D | X11/Xwayland | +1/-1 | fix: modifier keys make x11-window type anything in wayland. |
| 216 | `dd567603f197` | 2023-10-31 | zhangyu | D | 分屏 splitscreen | +3/-0 | fix: solve the split window don't minimize |
| 217 | `7f20e1e9e769` | 2023-10-30 | huangzeming | D | 其它核心 (workspace/window 等) | +281/-2 | fix: add lightweight configuration |
| 218 | `f94fb2dd9cd6` | 2023-11-03 | huangzeming | D | 其它核心 (workspace/window 等) | +4/-0 | fix: maximized window didn't overspread while dock in smart-hide mode |
| 219 | `57bea664b8a8` | 2023-11-03 | huangzeming | D | 构建/打包 | +13/-0 | chore: Update version number for 5.27.2.30-deepin30 |
| 220 | `3a960be0f31c` | 2023-11-03 | zhijie geng | D | 其它核心 (workspace/window 等) | +1/-1 | fix: close window effects, cold start time is degraded      compared to historical baseline |
| 221 | `dc37af6dbcc0` | 2023-11-03 | huangzeming | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.31-deepin31 |
| 222 | `12bb6cfb3a1d` | 2023-11-08 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +1/-1 | feat: do not send event to none ddeshell window |
| 223 | `acd2aaa472ea` | 2023-11-06 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +13/-3 | fix: reset dpms only for unplugin screens |
| 224 | `656a30295430` | 2023-11-07 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +145/-57 | feat: rebuild remote access |
| 225 | `6cf0daf10bd1` | 2023-11-07 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +13/-3 | fix: Add displaylink support on mali gpu |
| 226 | `66401722336b` | 2023-11-06 | huangzeming | D | 其它核心 (workspace/window 等) | +6/-1 | fix: multitaskview effect was disabled by default |
| 227 | `da21b89d1f32` | 2023-11-08 | huangzeming | D | 特效 | +0/-6 | feat: optimize partial refresh |
| 228 | `48194031f63a` | 2023-11-08 | zhang yu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.32 |
| 229 | `2bbe63ed520b` | 2023-11-09 | wangyu | D | Wayland 协议/服务端 (含 dde 私有协议) | +3/-0 | fix: unable to close the window through the "x" button in the upper right corner of the Dock bar window preview |
| 230 | `cf65f792cce3` | 2023-11-09 | zhangyu | D | 特效 | +6/-6 | fix: solve the window opacity in alt + tab |
| 231 | `ffc6272589f0` | 2023-11-10 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +7/-0 | fix: clear config cache data |
| 232 | `52a7c440f99e` | 2023-11-13 | zhang yu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.33 |
| 233 | `75db2e63a571` | 2023-11-10 | Chaojiang Luo | B | 其它核心 (workspace/window 等) | +19/-20 | fix: Do not restart wayland compositing when config change |
| 234 | `cc8ea8b2a782` | 2023-11-13 | Chaojiang Luo | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.34 |
| 235 | `8514bdd49471` | 2023-11-13 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +9/-0 | fix: Fix nullptr error of m_shm_rp_buffer |
| 236 | `72d3900534c4` | 2023-11-13 | wangboyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.35 |
| 237 | `f536cc931a6c` | 2023-11-14 | wangyu | D | X11/Xwayland | +17/-2 | fix: clipboard is empty after copying content from wayland web page |
| 238 | `8295f2fda1f5` | 2023-11-14 | huangzeming | D | Wayland 协议/服务端 (含 dde 私有协议) | +3/-0 | fix: clipboard-ace worked uncorrectly on wayland |
| 239 | `a798b9ba5838` | 2023-11-14 | huangzeming | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.36-deepin36 |
| 240 | `13198cf36c16` | 2023-11-15 | wangyu | D | 显示后端/DRM/多屏 | +1/-1 | fix: wayland protocol under the mouse or touchpad rolling bar is too slow |
| 241 | `354102ebb968` | 2023-11-14 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +42/-5 | fix: fix displaylink error |
| 242 | `3c98c67a25c7` | 2023-11-15 | huangzeming | D | 输入 | +125/-6 | fix: add xwayland grab keyboard feature |
| 243 | `75b61fe00aa6` | 2023-11-15 | zhangyu | D | 其它核心 (workspace/window 等) | +62/-19 | fix: split window adapter light weight |
| 244 | `41a14fda75eb` | 2023-11-17 | zhijie geng | D | 特效 | +1/-1 | fix: desktop become grey after kwin boost |
| 245 | `abb4cba138a6` | 2023-11-17 | zhangyu | D | 其它核心 (workspace/window 等) | +11/-1 | fix: prefect qpa for theme |
| 246 | `4f671f6ec173` | 2023-11-17 | zhangyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.37 |
| 247 | `fde51815dfa8` | 2023-11-20 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +12/-2 | fix: fix wild pointer in remote access |
| 248 | `d338f610cdc3` | 2023-11-21 | liuzheng | A | 录屏/截图/远程 | +37/-34 | fix: screencasting: on memfd, skip the QImage step |
| 249 | `18d63bad13aa` | 2023-11-21 | liuzheng | B | 录屏/截图/远程 | +25/-5 | fix: Fix the black screen issue with gles screencast |
| 250 | `a9bec5dbfb29` | 2023-11-21 | zhangyu | D | 其它核心 (workspace/window 等) | +41/-24 | fix: c++ interface param about js call |
| 251 | `288d5f02f26a` | 2023-11-20 | huangzeming | D | 多任务视图 multitaskview | +19/-25 | fix: tips for desktops without window in multitaskview |
| 252 | `8fbf1208b2a2` | 2023-11-21 | huangzeming | D | 多任务视图 multitaskview | +6/-6 | fix: touch events weren't processed in multitaskview |
| 253 | `a363f443ae66` | 2023-11-23 | wangyu | B | 其它核心 (workspace/window 等) | +11/-11 | fix: drag the icon in the Dock application area to display the titlebar window |
| 254 | `bac694b676e9` | 2023-11-24 | wangboyu | D | 构建/打包 | +15/-0 | chore: Update version number for 5.27.2.38 |
| 255 | `39b05d97d155` | 2023-11-24 | huangzeming | A | 特效 | +475/-962 | plugins/blur: Rewrite it |
| 256 | `1943ebd47c2a` | 2023-11-24 | wangboyu | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.39 |
| 257 | `7a2c6f9de191` | 2023-11-28 | huangzeming | D | 多任务视图 multitaskview | +2/-1 | fix: switching workspace caused crash in duplicate mode in multitaskview |
| 258 | `ed0f2aa23261` | 2023-11-28 | wangboyu | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.40 |
| 259 | `d1bf81128a92` | 2023-11-30 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +0/-4 | fix: remove the limit what xwaylandInterfaces only allow in x11 application. |
| 260 | `061f91126477` | 2023-11-29 | zhijie geng | D | Alt+Tab 切换器 | +4/-0 | fix: set tabbox go back to the first window |
| 261 | `e6fccf1ccb6b` | 2023-11-27 | yangtian | D | 特效 | +61/-35 | fix: alpha blend in scissorWindow |
| 262 | `9a8503e339f1` | 2023-11-30 | huangzeming | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.41 |
| 263 | `b4e23b409f75` | 2023-11-30 | xinbo wang | D | 其它核心 (workspace/window 等) | +9/-1 | fix: replace from QApplication::exit to _exit in signhandler of kill. |
| 264 | `ad316e76de5d` | 2023-11-30 | xinbo wang | D | 其它核心 (workspace/window 等) | +228/-1 | fix: change kde default useraction menu to dde style. |
| 265 | `16ae50a7018c` | 2023-11-30 | huangzeming | D | 多任务视图 multitaskview | +4/-6 | fix: animations in multitaskview weren't controlled by window_animation field |
| 266 | `4d6b3324f090` | 2023-11-30 | huangzeming | D | 其它核心 (workspace/window 等) | +52/-0 | fix: desktop was black after upgrading from 1062 to 1070 |
| 267 | `82b5c343cc09` | 2023-11-29 | xinbo wang | D | 其它核心 (workspace/window 等) | +37/-3 | fix: prohibit window uneffect when minimize |
| 268 | `ea06e4c3b494` | 2023-11-27 | xinbo wang | D | 其它核心 (workspace/window 等) | +30/-1 | fix: add dbus interface to get led status of keyboard. |
| 269 | `c58e84a81602` | 2023-11-30 | zhangyu | D | 其它核心 (workspace/window 等) | +94/-2 | fix: solve the window transparent when preview from dock |
| 270 | `871cfabf07ba` | 2023-11-30 | zhangyu | D | 其它核心 (workspace/window 等) | +2/-2 | fix: repair the ineffective about effects switch |
| 271 | `22ec5431b4d5` | 2023-11-30 | liuzheng | D | 显示后端/DRM/多屏 | +5/-1 | fix: Fix the issue of KLV eye protection mode failure |
| 272 | `9407ea04f837` | 2023-12-01 | yangtian | D | 特效 | +0/-1 | fix: delay display of preview window. |
| 273 | `3b41eda0c822` | 2023-12-01 | zhangyu | D | 分屏 splitscreen | +21/-0 | fix: relayout window preview in wayland |
| 274 | `0679dd0a1dab` | 2023-11-30 | zhangyu | D | 分屏 splitscreen | +1/-0 | fix: finish swap timeline when split window exited |
| 275 | `28f082df5e6b` | 2023-12-01 | huangzeming | D | 构建/打包 | +19/-0 | chore: Update version number for 5.27.2.42 |
| 276 | `12160484d066` | 2023-12-01 | zhangyu | D | 特效 | +44/-1282 | fix: remove meta+a\meta+w functions |
| 277 | `11891dd6f46f` | 2023-12-01 | wangboyu | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.43 |
| 278 | `fadbbdb402da` | 2023-12-04 | liuzheng | D | 输入 | +13/-1 | fix: Filter the simulated touch devices and do not add them to the Xwayland event sending |
| 279 | `9d90b5f2b689` | 2023-12-04 | xinbo wang | D | Alt+Tab 切换器 | +3/-2 | fix: tabbox crash in app_opentab pressure testing script. |
| 280 | `fb269c22c1e6` | 2023-12-04 | wangyu | D | 其它核心 (workspace/window 等) | +6/-0 | fix: when playing a video, the video cannot continue to play after minimizing the window and then restoring it |
| 281 | `c5ab9f6bc3c1` | 2023-12-04 | wangboyu | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.44 |
| 282 | `5e0cc2eba1af` | 2023-12-05 | wangyu | D | 其它核心 (workspace/window 等) | +25/-5 | fix: wrong font and color for window menu options |
| 283 | `433682117102` | 2023-12-05 | zhangyu | D | 其它核心 (workspace/window 等) | +1/-28 | Revert "fix: Fix focus error in lock screen" |
| 284 | `73e513f796a8` | 2023-12-06 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +1/-1 | fix: Fix screen stuck while display mode changed |
| 285 | `2e1db0dfac93` | 2023-12-05 | yangtian | D | 特效 | +3/-1 | fix: collapse caused by surfaceitem setScissorAlpha |
| 286 | `d881ba143fd3` | 2023-12-07 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +6/-1 | feat: Remove CVT mode on Hisi devices |
| 287 | `f04e0133d61d` | 2023-12-07 | wangboyu | D | 构建/打包 | +12/-0 | chore: Update version number for 5.27.2.45 |
| 288 | `0e57aa437231` | 2023-12-08 | wangyu | D | 其它核心 (workspace/window 等) | +26/-2 | fix: there is no selected state in the menu bar |
| 289 | `fcb6877501ab` | 2023-12-08 | wangyu | D | 其它核心 (workspace/window 等) | +5/-3 | fix: the right-click menu actually responds to the inactive window |
| 290 | `e3dbcda14fe8` | 2023-12-08 | wangyu | D | 其它核心 (workspace/window 等) | +3/-0 | fix: launcher window can be dragged |
| 291 | `090fcbe6f8f3` | 2023-12-08 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +3/-7 | feat: round up to an integer for vrefresh |
| 292 | `445f899c2667` | 2023-12-09 | liuzheng | D | 显示后端/DRM/多屏 | +3/-1 | fix: Fixed the issue that the mouse was red during the KLV boot process |
| 293 | `873d036fedfc` | 2023-12-09 | Chaojiang Luo | D | 构建/打包 | +11/-0 | chore: Update version number for 5.27.2.46 |
| 294 | `d083de94d819` | 2023-12-11 | zhangyaning | D | 输入 | +5/-3 | fix: updateAvailableInputDevices when device is not fakeinput. |
| 295 | `5e535168ed8f` | 2023-12-11 | wangboyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.47 |
| 296 | `053578ae4b81` | 2023-12-12 | xinbo wang | D | X11/Xwayland | +1/-1 | fix: prohibit screen active when client change desktop |
| 297 | `c8b958c3dcf3` | 2023-12-14 | xinbo wang | D | Wayland 协议/服务端 (含 dde 私有协议) | +0/-3 | fix: remove selectionchanged/selectioncleared of primary selection. |
| 298 | `8e1d12f26dc1` | 2023-12-14 | xinbo wang | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +1086/-254 | fix: thirdpart titlebar icon don't change with system zoom |
| 299 | `6b5c13d222ce` | 2023-12-12 | zhangyaning | D | 输入 | +4/-0 | fix: greeter will be killed when the shortcut of alt+f4 is pressed. |
| 300 | `1c4f78aa0075` | 2023-12-01 | zhangyaning | D | 显示后端/DRM/多屏 | +114/-2 | fix: add function setTouchDeviceToScreenId and getTouchDeviceToScreenInfo. |
| 301 | `2a9644b117ea` | 2023-12-15 | wangboyu | D | 构建/打包 | +11/-0 | chore: Update version number for 5.27.2.48 |
| 302 | `b48ceeadab2c` | 2023-12-11 | liuzheng | D | 其它核心 (workspace/window 等) | +56/-0 | fix: Adapt to fps and windowdisplay interfaces |
| 303 | `220c5f1708fc` | 2023-12-18 | wangyu | D | 其它核心 (workspace/window 等) | +0/-3 | Revert "fix: launcher window can be dragged" |
| 304 | `40360db93a96` | 2023-12-18 | wangyu | D | 其它核心 (workspace/window 等) | +3/-0 | fix: launcher window can be dragged |
| 305 | `0d8971bccc5c` | 2023-12-18 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +158/-96 | feat: ignore output added and removed signal |
| 306 | `55659249ce68` | 2023-12-18 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +20/-0 | fix: Resolve inability to obtain screen size |
| 307 | `934c3a8c690c` | 2023-12-14 | zhangyu | D | 其它核心 (workspace/window 等) | +68/-16 | fix: fix the radius of unmanage window |
| 308 | `ffb895b45769` | 2023-12-19 | wangyu | D | 其它核心 (workspace/window 等) | +2/-2 | fix: the pop-up window of the self-developed Qt application cannot be moved |
| 309 | `cd6c3a8fa526` | 2023-12-18 | wangyu | D | 其它核心 (workspace/window 等) | +1/-0 | fix: the right-click menu position of all areas of the taskbar is displayed incorrectly |
| 310 | `c8a337fa208b` | 2023-12-18 | wangyu | D | 其它核心 (workspace/window 等) | +23/-1 | fix: the right-click menu should not be displayed after using the alt+space shortcut key in the launcher window |
| 311 | `03261d5ec4a8` | 2023-12-18 | huangzeming | D | Wayland 协议/服务端 (含 dde 私有协议) | +126/-7 | feat: replaced interface GetAllWindowStates with GetAllWindowStatesList |
| 312 | `9f117762302b` | 2023-12-19 | huangzeming | D | 构建/打包 | +18/-0 | chore: Update version number for 5.27.2.49 |
| 313 | `eff2364af16d` | 2023-12-18 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +214/-26 | feat: Add compatibility for v2 |
| 314 | `0a4efbc7fb61` | 2023-12-19 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +6/-3 | feat: fix static check error |
| 315 | `ba1fe465bf2d` | 2023-12-19 | Chaojiang Luo | D | 构建/打包 | +8/-0 | chore: add new version 4:5.27.2.50-deepin50 |
| 316 | `97ff9aff38ca` | 2023-12-18 | zhijie geng | D | 其它核心 (workspace/window 等) | +6/-0 | fix: x11 alt+space popup window can't disapear |
| 317 | `059417e94cad` | 2023-12-21 | zhangyu | D | 其它核心 (workspace/window 等) | +0/-3 | fix: repair the window can't show when close from dock preview |
| 318 | `899c0af667e8` | 2023-12-19 | xinbo wang | D | 输入 | +69/-1 | fix: x11 grab window cannot recieve pointer press |
| 319 | `0152d633d9d2` | 2023-12-21 | zhijie geng | D | 其它核心 (workspace/window 等) | +32/-2 | fix: update from 1040 to 1060, workspace's number      change from three to two |
| 320 | `5ddcb34a9381` | 2023-12-21 | xinbo wang | D | 其它核心 (workspace/window 等) | +15/-0 | fix: keypad shortcut donnot take effect |
| 321 | `a44913259444` | 2023-11-08 | xinbo wang | D | 显示后端/DRM/多屏 | +1667/-41 | feat: add XRender backend support. |
| 322 | `bb77dcaad101` | 2023-12-21 | huangzeming | D | 其它核心 (workspace/window 等) | +29/-58 | feat: add support for switching compositing backend |
| 323 | `4f931f063b20` | 2023-12-21 | huangzeming | D | 构建/打包 | +13/-0 | chore: Update version number for 5.27.2.50 |
| 324 | `5c47849f9779` | 2023-12-21 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +2/-1 | fix: Fix screen flash for damage error |
| 325 | `a4088a869608` | 2023-12-19 | zhangyaning | D | 其它核心 (workspace/window 等) | +24/-0 | fix: add dbus interface of disableGlobalShortcutsForClient according to kwin of 106x. |
| 326 | `fd52cdcbf75b` | 2023-12-22 | zhijie geng | D | 其它核心 (workspace/window 等) | +14/-9 | fix: right click third-part titlebar can't show menu |
| 327 | `c0b8a9820ce2` | 2023-12-22 | zhijie geng | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.52 |
| 328 | `776208bc7279` | 2023-12-26 | luochaojiang | D | 其它核心 (workspace/window 等) | +1/-2 | Revert "fix: Fix screen flash for damage error" |
| 329 | `26163b16ad5b` | 2023-12-26 | wangboyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.53 |
| 330 | `2a946cb923c6` | 2023-12-27 | zhangyaning | D | 其它核心 (workspace/window 等) | +0/-237 | fix: remove some code about KWinKScreenIntegration and setting default display mode. |
| 331 | `a4724cca63c2` | 2023-12-27 | xinbo wang | D | 其它核心 (workspace/window 等) | +8/-0 | fix: window to desktop will change active client. |
| 332 | `4549792e81dd` | 2023-12-28 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +5/-5 | fix: remove some log about remoteaccess. |
| 333 | `5a2c93f1421d` | 2023-12-28 | zhangyaning | D | 显示后端/DRM/多屏 | +1/-1 | fix: set fullpath for dmidecode. |
| 334 | `4c2335e06552` | 2023-12-26 | huangzeming | D | 其它核心 (workspace/window 等) | +169/-0 | feat: journal and debug config adaptation |
| 335 | `77a87c55c645` | 2023-12-27 | wangyu | D | 其它核心 (workspace/window 等) | +4/-0 | fix: when the text management search input box is activated, the text prompt after hovering the mouse to the Sogou input method icon cancels the search input box state |
| 336 | `08c4878dab84` | 2023-12-27 | zhang yu | D | 分屏 splitscreen | +2/-3 | fix: quit split from maxi in wayland |
| 337 | `26771d84ea01` | 2023-12-27 | zhang yu | D | 分屏 splitscreen | +35/-0 | fix: perfect the touch event in split preview |
| 338 | `b9fbe1c6153e` | 2023-12-27 | zhangyu | D | 特效 | +26/-5 | fix: repair the preview problem in dock |
| 339 | `b2719cdd8d94` | 2023-12-28 | huangzeming | D | 多任务视图 multitaskview | +8/-0 | fix: closing browser stuck in multitaskview |
| 340 | `c510b508a63d` | 2023-12-28 | huangzeming | D | 特效 | +1/-1 | fix: screenshot on wayland was not clear |
| 341 | `e020b859a9d2` | 2023-12-27 | huangzeming | D | 多任务视图 multitaskview | +5/-14 | fix: wheel events in multitaskview |
| 342 | `64f9a0539e6e` | 2023-12-28 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +1/-0 | fix: update shadow when radius change in wayland |
| 343 | `9d4f1b038043` | 2023-12-25 | zhangyaning | D | 其它核心 (workspace/window 等) | +10/-0 | fix: add dbus interface to know if x11 client grabbed keyboard. |
| 344 | `b6b3abd6f225` | 2023-12-29 | zhijie geng | D | 输入 | +23/-0 | fix: unable to use TabletTool device to move dtk client |
| 345 | `765c04750819` | 2023-12-01 | zhijie geng | D | Alt+Tab 切换器 | +8/-4 | fix: after showing desktop, type alt+tab and click      in empty area, windows don't restore |
| 346 | `28a58d625885` | 2023-12-28 | zhijie geng | D | 显示后端/DRM/多屏 | +1/-1 | fix: VGA screen become black in XRender compositor |
| 347 | `524bd463c52c` | 2023-12-29 | zhijie geng | D | 构建/打包 | +23/-0 | chore: Update version number for 5.27.2.54 |
| 348 | `4596e1bd2b0d` | 2023-12-27 | xinbo wang | D | 其它核心 (workspace/window 等) | +18/-1 | fix: add maximize/unmaximize window shortcut. |
| 349 | `507bd2fb7223` | 2024-01-03 | Chaojiang Luo | D | X11/Xwayland | +2/-0 | fix: remove output primary and order for xwayland |
| 350 | `4994577e46e8` | 2024-01-03 | wangboyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.55 |
| 351 | `d3ed96584efb` | 2024-01-02 | zhijie geng | D | Alt+Tab 切换器 | +34/-1 | fix: window switcher background is transparent in      xrender mode |
| 352 | `9df01ea58ccd` | 2024-01-02 | wangyu | D | 其它核心 (workspace/window 等) | +1/-1 | fix: pop-up animation appears when the window is minimized and closed through the dock |
| 353 | `21ae67c2062b` | 2023-12-29 | huangzeming | D | 其它核心 (workspace/window 等) | +169/-16 | feat: add effect type configuration |
| 354 | `e3a4b053ade2` | 2024-01-03 | wangyu | D | 其它核心 (workspace/window 等) | +1/-1 | fix: after minimizing the backup file management and log management windows, the main window is also minimized |
| 355 | `3a2914a5ed12` | 2024-01-03 | huangzeming | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.56 |
| 356 | `f915441340ed` | 2024-01-04 | wangyu | D | 其它核心 (workspace/window 等) | +5/-0 | fix: the window is minimized after ending the preview in the dock |
| 357 | `b2139d65c2ac` | 2024-01-02 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +97/-2 | feat: add dynamic load for gbm modify |
| 358 | `8d56fc864782` | 2024-01-02 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +9/-0 | feat: fix gbm_bo_get_stride_for_plane get stride failed |
| 359 | `3ac3f65ce40d` | 2024-01-04 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +18/-1 | feat: disable partial update for pgux |
| 360 | `4fd7acfaaafc` | 2024-01-04 | Chaojiang Luo | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.57 |
| 361 | `249c6c914347` | 2024-01-03 | zhijie geng | D | 特效 | +1/-0 | fix: in XRender, alt+tab can't preview minimized window |
| 362 | `6b00d857e22e` | 2024-01-05 | zhangyu | D | 特效 | +0/-13 | fix: split preview can show right angle |
| 363 | `364c6c3d1d97` | 2024-01-03 | huangzeming | D | 特效 | +3/-1 | fix: blur effect was broken on VGA screen |
| 364 | `1b6154addea3` | 2024-01-05 | huangzeming | D | 多任务视图 multitaskview | +8/-6 | fix: buttons in multitaskview didn't show using touchscreen |
| 365 | `db5978574a28` | 2024-01-04 | yangtian | D | 特效 | +760/-191 | feat: animation effect |
| 366 | `b4d8d7a60744` | 2024-01-05 | wangyu | D | 其它核心 (workspace/window 等) | +1/-1 | fix: the Always Visible Workspace button in the right-click menu cannot be clicked when the window is split |
| 367 | `0297bc9f50b4` | 2024-01-02 | dingzhixuan | D | 显示后端/DRM/多屏 | +13/-13 | fix: capslock led donnot reset when reboot |
| 368 | `3d7a04e01fb7` | 2024-01-02 | liuzheng | D | Wayland 协议/服务端 (含 dde 私有协议) | +4/-1 | fix: Fix the issue of failed simulation of mouse scrolling events |
| 369 | `694a41d7b5b2` | 2023-12-26 | zhijie geng | D | 其它核心 (workspace/window 等) | +16/-2 | fix: after turning off the window special effects switch      and then turning it on again, the taskbar transparency      adjustment value becomes 1 |
| 370 | `ccd9d48ef085` | 2024-01-05 | zhijie geng | D | 构建/打包 | +1/-17 | fix: kwin has invalid shortcuts |
| 371 | `dd3dc19c1b7a` | 2024-01-04 | zhangyaning | D | 其它核心 (workspace/window 等) | +18/-0 | fix: add dbus interface to set repeat and delay of keyboard. |
| 372 | `2c1e092aa3c2` | 2024-01-05 | huangzeming | D | 其它核心 (workspace/window 等) | +37/-9 | feat: dynamic response to DConfig effect type |
| 373 | `5df0379cf2d3` | 2024-01-05 | huangzeming | D | 构建/打包 | +18/-0 | chore: Update version number for 5.27.2.58 |
| 374 | `e8a6eb79c43a` | 2024-01-05 | xinbo wang | D | 其它核心 (workspace/window 等) | +2/-0 | fix: moveResize window without button press,      we also need update the relative position of the pointer. |
| 375 | `9975e3d0f91b` | 2024-01-08 | zhangyu | D | 其它核心 (workspace/window 等) | +1/-4 | fix: delete notification due to graphics reset |
| 376 | `36e78febf88c` | 2024-01-08 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +14/-17 | fix: modify window default radius |
| 377 | `e9e8c51ad57f` | 2024-01-08 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +13/-13 | fix: shadow at the top of window |
| 378 | `cede6675f09e` | 2024-01-08 | xinbo wang | D | 显示后端/DRM/多屏 | +6/-0 | fix: enable prohibit screenshot in sunloginClient |
| 379 | `f95b780e5cc7` | 2024-01-09 | yangtian | D | 特效 | +1/-1 | fix: dialog animation |
| 380 | `609a345eeed2` | 2024-01-09 | yangtian | D | 特效 | +2/-2 | fix: transparency of scissor effect |
| 381 | `a22ea1efead1` | 2024-01-06 | huangzeming | D | 多任务视图 multitaskview | +1/-1 | fix: window filled background didn't align with window after relayout |
| 382 | `e6adff4a71c5` | 2024-01-09 | yangtian | D | 构建/打包 | +14/-0 | chore: Update version number for 5.27.2.59 |
| 383 | `c4c7e9577bb5` | 2024-01-09 | wangyu | D | X11/Xwayland | +4/-0 | fix: show left triggered when window changes size |
| 384 | `3bc4eeabd9ff` | 2024-01-10 | wangyu | D | 其它核心 (workspace/window 等) | +9/-0 | fix: the window is not adapted to display after adjusting the position of the taskbar in the lower part of the screen |
| 385 | `96914bad7fc5` | 2024-01-10 | zhijie geng | D | 显示后端/DRM/多屏 | +6/-6 | fix: after adjusting the screen transform in xrender      mode, only half of screen is displayed |
| 386 | `8334cbe1ee89` | 2024-01-10 | huangzeming | D | 其它核心 (workspace/window 等) | +4/-0 | fix: window position in multitaskview on secondary screen was incorrect on wayland |
| 387 | `bef723272ae3` | 2024-01-11 | wangyu | D | 其它核心 (workspace/window 等) | +4/-4 | fix: the maximized window of the secondary screen does not cover the entire screen |
| 388 | `97a4cd4ced87` | 2024-01-10 | zhijie geng | D | X11/Xwayland | +1/-2 | fix: gtk window shows black border |
| 389 | `93d291573396` | 2024-01-11 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +4/-4 | fix: shadow of right click menu |
| 390 | `2c24ea3051c8` | 2024-01-11 | yangtian | D | 特效 | +9/-3 | fix: animation of shutdown menu |
| 391 | `7dbf357ae945` | 2024-01-11 | zhijie geng | D | 显示后端/DRM/多屏 | +3/-3 | fix: zx install error |
| 392 | `9e333873f160` | 2024-01-11 | zhijie geng | D | 构建/打包 | +15/-0 | chore: Update version number for 5.27.2.60 |
| 393 | `7e822ef3e014` | 2024-01-11 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +20/-1 | feat: Add alternative for EGL_LINUX_DMA_BUF_EXT |
| 394 | `d9c782cd45c1` | 2024-01-11 | wangboyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.61 |
| 395 | `b9b339f8e994` | 2024-01-12 | zhijie geng | D | 其它核心 (workspace/window 等) | +6/-3 | fix: rendering problem of wine app in cloud desktop      xrender mode |
| 396 | `f1444d33297c` | 2024-01-12 | yangtian | D | 其它核心 (workspace/window 等) | +9/-1 | fix: opacity blend of titlebar menu |
| 397 | `3ebfb0b26489` | 2024-01-09 | zhangyu | D | Alt+Tab 切换器 | +2/-4 | fix: show icon in xrender env |
| 398 | `72b6c804443a` | 2024-01-14 | wangboyu | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.62 |
| 399 | `ad260bd4892e` | 2024-01-15 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +1/-1 | fix: the variable of isAccepted using isEmpty insteadof isNull. |
| 400 | `0ca292c36ece` | 2024-01-16 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +4/-4 | fix: shadow radius of normal window |
| 401 | `93d64c3db9ca` | 2024-01-15 | Chaojiang Luo | D | 特效 | +3/-3 | fix: Fix screen flash black due to damage error |
| 402 | `21cd36226dd7` | 2024-01-16 | zhangyaning | D | 其它核心 (workspace/window 等) | +19/-8 | fix: performance optimization. |
| 403 | `bae18441940f` | 2024-01-16 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +8/-6 | fix: tooltip shadow |
| 404 | `950f1fc56972` | 2024-01-16 | wangboyu | D | 构建/打包 | +11/-0 | chore: Update version number for 5.27.2.63 |
| 405 | `b469af09fd98` | 2024-01-18 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +31/-23 | fix: when the output Changed once, only send done once. |
| 406 | `528d53dd93fa` | 2024-01-19 | huangzeming | D | 其它核心 (workspace/window 等) | +9/-7 | fix: kwin didn't record opacity if opacity was 1 |
| 407 | `bb50ddd18c0c` | 2024-01-20 | huangzeming | D | 其它核心 (workspace/window 等) | +31/-13 | fix: temporarily support the old version switch |
| 408 | `50c9031523c5` | 2024-01-21 | wangboyu | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.64 |
| 409 | `7c0b2583c0c5` | 2024-01-22 | yangtian | D | 特效 | +16/-6 | fix: the conflict between animation and dde effect |
| 410 | `716f86d03e7a` | 2024-01-23 | yangtian | D | 特效 | +18/-26 | fix: glsl version of scissor shader |
| 411 | `2d3e05ce3d74` | 2024-01-25 | Lu YaNing | B | Wayland 协议/服务端 (含 dde 私有协议) | +18/-6 | feat: protocol: add wl_pointer.warp event |
| 412 | `f447d58917cf` | 2024-01-20 | zhangyu | D | 其它核心 (workspace/window 等) | +376/-80 | fix: reconstruct window radius |
| 413 | `eb74563e42aa` | 2024-01-28 | wangboyu | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.65 |
| 414 | `13ae57a015e0` | 2024-01-26 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +238/-25 | fix: shadow and scissor refresh |
| 415 | `90935e79f1a3` | 2024-01-26 | zhangyu | D | 分屏 splitscreen | +28/-21 | fix: optimize split menu positon scale |
| 416 | `585fb69fee12` | 2024-01-26 | zhangyu | D | 其它核心 (workspace/window 等) | +22/-16 | fix: repair window's radius scale |
| 417 | `0f5bc5a73cc3` | 2024-01-29 | wangboyu | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.66 |
| 418 | `8a9bcdb4e338` | 2024-01-24 | Chaojiang Luo | D | Wayland 协议/服务端 (含 dde 私有协议) | +1/-3 | fix: Avoid double remove in QHash sentBuffers |
| 419 | `823e26c3a516` | 2024-01-29 | luochaojiang | D | 其它核心 (workspace/window 等) | +237/-0 | Revert "fix: remove some code about KWinKScreenIntegration and setting default display mode." |
| 420 | `b6b5351fc915` | 2024-01-29 | Chaojiang Luo | D | 其它核心 (workspace/window 等) | +11/-3 | feat: Fix screen position changed after external screen re-plugin |
| 421 | `a1306d2049ab` | 2024-01-22 | wangyu | D | Wayland 协议/服务端 (含 dde 私有协议) | +32/-7 | feat: optimize the clipboard seat process |
| 422 | `da42a6abb801` | 2024-01-30 | huangzeming | D | 其它核心 (workspace/window 等) | +8/-2 | fix: dock will flash after switching theme color |
| 423 | `2b9787a760fc` | 2024-01-31 | huangzeming | D | 其它核心 (workspace/window 等) | +6/-3 | fix: solve switch 2d to 3d black screen problem |
| 424 | `8ed25fc70dbe` | 2024-01-30 | zhangyu | D | 其它核心 (workspace/window 等) | +21/-3 | fix: shape window shadow |
| 425 | `96c00118b936` | 2024-01-23 | zhangyu | D | X11/Xwayland | +4/-0 | fix: check the minimize and maximize attribute when window create |
| 426 | `270eeb623586` | 2024-01-31 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +52/-52 | fix: blur region, scissor and shadow |
| 427 | `563546d9c79c` | 2024-01-31 | zhangyaning | D | 其它核心 (workspace/window 等) | +6/-1 | feat: remove the log of kwin writing to syslog for now. |
| 428 | `03aafbcaabb7` | 2024-01-31 | huangzeming | D | 构建/打包 | +16/-0 | chore: Update version number for 5.27.2.67 |
| 429 | `aa606f6a6bf5` | 2024-02-01 | huangzeming | D | 其它核心 (workspace/window 等) | +5/-4 | fix: unable to open x11 application on wayland |
| 430 | `dceeb6effe8a` | 2024-02-01 | dingzhixuan | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.68 |
| 431 | `03e4086f324a` | 2024-02-02 | huangzeming | D | 其它核心 (workspace/window 等) | +68/-10 | fix: raise window after button released |
| 432 | `a16d5c604c1c` | 2024-02-02 | zhangyu | D | X11/Xwayland | +1/-0 | fix: display the outline of window when show desktop |
| 433 | `13b1d28ba1bd` | 2024-02-02 | zhangyu | D | 其它核心 (workspace/window 等) | +15/-6 | fix: modify window radius scale |
| 434 | `1eff2a03691c` | 2024-02-03 | wangboyu | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.69 |
| 435 | `998c3a901285` | 2024-02-01 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +0/-6 | fix: only destroy the resource when remoteaccessManager is released. |
| 436 | `e974367a0527` | 2024-02-02 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +1/-1 | fix: dock panel shadow in wayland |
| 437 | `7ff80668ef45` | 2023-12-28 | zhangyaning | D | 显示后端/DRM/多屏 | +28/-0 | fix: send the buffer of output in remoteaccess when the output is longly not presented. |
| 438 | `f6d6a7521a39` | 2024-02-04 | yangtian | D | 特效 | +1/-1 | fix: cancel noise strength in blur |
| 439 | `298c57543096` | 2024-02-04 | zhijie geng | D | 输入 | +7/-6 | fix: kwin crashes on logout |
| 440 | `7161e2193aa6` | 2024-02-02 | yangtian | D | 特效 | +18/-12 | fix: cancel osd effect |
| 441 | `ace871644b54` | 2024-02-20 | huangzeming | D | 构建/打包 | +12/-0 | chore: Update version number for 5.27.2.70 |
| 442 | `762792984e1e` | 2024-02-20 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +1/-1 | fix:Revert "fix: the variable of isAccepted using isEmpty insteadof isNull." |
| 443 | `b79f89018ad0` | 2024-02-22 | zhijie geng | D | 其它核心 (workspace/window 等) | +1/-1 | fix: when prohibitscreenshot demo is moved to another      workspace, it still take effect in the origin      workspace |
| 444 | `c5f356a22d99` | 2024-02-04 | xinbo wang | D | 其它核心 (workspace/window 等) | +1/-1 | fix: dde-control can't read maximize/unmaximize shortcut |
| 445 | `e34d9171df86` | 2024-02-22 | xinbo wang | D | 特效 | +29/-2 | fix: prohibit window preview image error in dock. |
| 446 | `b01f6578d0e3` | 2024-02-22 | huangzeming | D | 其它核心 (workspace/window 等) | +8/-1 | feat: 'DraggingWithContent' is only supported in 2d |
| 447 | `b2010794f18b` | 2024-02-22 | huangzeming | D | 多任务视图 multitaskview | +14/-11 | fix: plugin on dock didn't respond if pointer moved in multitaskview |
| 448 | `1f1457bf1450` | 2024-02-21 | huangzeming | D | 其它核心 (workspace/window 等) | +1/-1 | fix: the minimum width of window on both side of screen is 160px |
| 449 | `9a879c6713c2` | 2024-02-05 | huangzeming | D | 其它核心 (workspace/window 等) | +46/-30 | fix: window restored before button released if DraggingWithContent is 0 |
| 450 | `df122a59ceef` | 2024-02-21 | zhangyaning | D | 其它核心 (workspace/window 等) | +1/-1 | fix: breaking the status of ShowingDesktop when the client keeped above. |
| 451 | `ae3a71767bf4` | 2024-02-22 | pengwenhao | D | 其它核心 (workspace/window 等) | +3/-0 | fix: The maximized window does not follow the changes with dock |
| 452 | `2fbaeb467938` | 2024-02-22 | zhijie geng | D | 其它核心 (workspace/window 等) | +156/-1 | feat: add event tracking |
| 453 | `2f5638942b23` | 2024-02-22 | zhangyu | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.71 |
| 454 | `51b8a1d407c4` | 2024-02-21 | pengwenhao | D | 其它核心 (workspace/window 等) | +3/-3 | fix: x apps can't be dragged to a negative position on the screen |
| 455 | `152046cdec63` | 2024-02-25 | xinbo wang | D | 其它核心 (workspace/window 等) | +11/-6 | fix: set prohibit window be preview in dock will be timeout. |
| 456 | `a6dd0fd55df7` | 2024-02-25 | zhijie geng | D | 其它核心 (workspace/window 等) | +1/-1 | fix: screen flash after user login in system |
| 457 | `80340ef5d5b6` | 2024-02-25 | huangzeming | D | 其它核心 (workspace/window 等) | +3/-1 | fix: window resized immediately when DraggingWithContent is 0 |
| 458 | `2564a9fe65e7` | 2024-02-23 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +12/-1 | fix: using clientResources instead of resourceMap. |
| 459 | `f5462af2990f` | 2024-02-23 | gerrit | D | 显示后端/DRM/多屏 | +5/-1 | fix: add judgment is on m_platform->renderBackend. |
| 460 | `841cd3bda740` | 2024-02-25 | zhangyu | D | 特效 | +8/-8 | fix: has decoration window can't transparent when move |
| 461 | `b650dde111c7` | 2024-02-22 | dingzhixuan | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +2/-2 | fix: wayland titlebar not scale |
| 462 | `253088f69156` | 2024-02-23 | dingzhixuan | D | 其它核心 (workspace/window 等) | +2/-6 | fix: dock strut error when change position |
| 463 | `276f7901cd17` | 2024-02-23 | dingzhixuan | D | 其它核心 (workspace/window 等) | +2/-2 | fix: force to read /etc/xdg/kwinrulesrc |
| 464 | `bf129798b2d7` | 2024-02-25 | dingzhixuan | D | Wayland 协议/服务端 (含 dde 私有协议) | +32/-0 | fix: plasmawindow virtual desktop is null |
| 465 | `d5381e4fe314` | 2024-02-25 | zhijie geng | D | 其它核心 (workspace/window 等) | +2/-6 | fix: can't install system |
| 466 | `69ba6b0db869` | 2024-02-23 | huangzhenyan | D | 翻译 | +124/-21 | fix: Add Traditional Chinese and Uyghur menu and multitasking view translation entries |
| 467 | `cde0a1b3f12b` | 2024-02-25 | zhangyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.72 |
| 468 | `42396f6e1c81` | 2024-02-23 | pengwenhao | D | 其它核心 (workspace/window 等) | +4/-0 | fix: maximiz the init need reset positonset |
| 469 | `773f62b9426c` | 2024-02-21 | huangzeming | D | 其它核心 (workspace/window 等) | +1/-2 | fix: window can be resized out of screen |
| 470 | `4992d94a9477` | 2024-02-26 | Lu YaNing | D | Wayland 协议/服务端 (含 dde 私有协议) | +2/-6 | fix:  Revert "Xwayland:Update pointer state before ending the drag" |
| 471 | `26124f678248` | 2024-02-26 | zhijie geng | D | 其它核心 (workspace/window 等) | +16/-20 | fix: there is no event tracking data in wayland |
| 472 | `4dfc5beb5f95` | 2024-02-26 | pengwenhao | D | 其它核心 (workspace/window 等) | +12/-1 | feat: add dde shell closeable request |
| 473 | `cb2e81b204eb` | 2024-02-26 | zhangyaning | D | 其它核心 (workspace/window 等) | +4/-0 | fix: When the useractionmenu appears, filter out the operation of maximizing and minimizing the window. |
| 474 | `56600400d8d3` | 2024-02-20 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +1051/-357 | fix: reconstruct window shadow |
| 475 | `b9844f3d2e92` | 2024-02-26 | dingzhixuan | D | X11/Xwayland | +68/-1 | fix: KuGou Music cause xwayland crash |
| 476 | `332abd82a078` | 2024-02-27 | dingzhixuan | D | 构建/打包 | +14/-0 | chore: Update version number for 5.27.2.73 |
| 477 | `70cdf4604eef` | 2024-02-27 | zhangyaning | D | 其它核心 (workspace/window 等) | +0/-4 | fix: Revert "fix: When the useractionmenu appears, filter out the operation of maximizing and minimizing the window." |
| 478 | `8a4f660ee874` | 2024-02-27 | zhangyaning | D | 其它核心 (workspace/window 等) | +5/-0 | fix: When the useractionmenu appears, filter out the operation of maximizing and minimizing the window. |
| 479 | `f83775bf595b` | 2024-02-27 | dingzhixuan | D | 其它核心 (workspace/window 等) | +50/-1 | feat: improve glmark2 score |
| 480 | `fa1545943abc` | 2024-02-27 | zhijie geng | D | 其它核心 (workspace/window 等) | +35/-0 | fix: there are a lot of xcb error BadAtom |
| 481 | `1d875fb1361a` | 2024-02-27 | zhijie geng | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.74 |
| 482 | `9080e554df77` | 2024-02-28 | zhijie geng | D | 其它核心 (workspace/window 等) | +1/-1 | fix: rendering problem of wine app in cloud desktop |
| 483 | `731b559d09bd` | 2024-02-29 | zhangyu | D | 多任务视图 multitaskview | +19/-3 | fix: add bo and ug language |
| 484 | `20fa96684a44` | 2024-02-29 | zhangyu | D | X11/Xwayland | +2/-0 | fix: fixed the issue that the laucher couldn't be fullscreen |
| 485 | `d3401bf957ac` | 2023-12-21 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +38/-0 | feat: add fault tolerance for remote buffer. |
| 486 | `eed0fbda6a15` | 2024-03-01 | zhijie geng | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.75 |
| 487 | `15e8eb172dd6` | 2024-03-01 | dingzhixuan | D | X11/Xwayland | +15/-2 | fix: wps checkbox cannot click for passthrough |
| 488 | `be93fe22217b` | 2024-03-01 | huangzeming | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +2/-2 | fix: border color of windows in 2d |
| 489 | `5213483ac7f4` | 2024-03-01 | huangzeming | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.76 |
| 490 | `f255bbb8b239` | 2024-03-01 | zhangyaning | D | 录屏/截图/远程 | +55/-9 | fix: using memory's copy instead of glReadPixels. |
| 491 | `9690cf1a019b` | 2024-03-01 | liuzheng | D | Wayland 协议/服务端 (含 dde 私有协议) | +6/-1 | fix: Modifying PrimarySelection Invalidation Issues |
| 492 | `ee992800cc1b` | 2024-02-29 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +76/-5 | fix: get radius from dtk gsettings, add lisence |
| 493 | `a704393bcb7e` | 2024-03-01 | huangzeming | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +2/-2 | fix: border color of dialog window in 2d |
| 494 | `d56b86059684` | 2024-03-01 | huangzeming | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.77 |
| 495 | `470a7092a446` | 2024-03-04 | zhangyaning | D | 其它核心 (workspace/window 等) | +36/-0 | fix: add sign for coredump. |
| 496 | `b257f0a9e7b4` | 2024-03-01 | dingzhixuan | D | Wayland 协议/服务端 (含 dde 私有协议) | +1/-6 | fix: dde-wloutput will show twich mode |
| 497 | `d7e5935ad626` | 2024-03-04 | zhangyu | D | 分屏 splitscreen | +27/-11 | fix: optimized the window swap function |
| 498 | `b579b28dbdb7` | 2024-03-05 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +6/-1 | fix: Fixed the issue that the rounded corners of the 3-square window were updated in real time |
| 499 | `a6027fbcc5b6` | 2024-03-05 | zhangyu | D | Alt+Tab 切换器 | +80/-4 | fix: alt tab window add blur |
| 500 | `07915fc2d157` | 2024-03-05 | zhangyu | D | 构建/打包 | +11/-0 | chore: Update version number for 5.27.2.78 |
| 501 | `1ed9e970ddc0` | 2024-03-06 | huangzeming | D | Wayland 协议/服务端 (含 dde 私有协议) | +12/-4 | fix: slowly respond in specificWindowStateRequest |
| 502 | `12d6b485924d` | 2024-03-06 | dingzhixuan | D | 其它核心 (workspace/window 等) | +1/-1 | fix: long touch cannot show menu on titlebar |
| 503 | `6dbbdbdf86ec` | 2024-03-06 | dingzhixuan | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.79 |
| 504 | `f610d7e1ffee` | 2024-03-05 | zhijie geng | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +28/-1 | feat: adjust titlebar height |
| 505 | `bd1d42b93f35` | 2024-03-06 | zhangyu | D | 其它核心 (workspace/window 等) | +265/-54 | fix: optimized config reader,alt tab window |
| 506 | `1f179c984c3e` | 2024-03-06 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +166/-74 | fix: optimized window border、shadowcolor、effects、scale |
| 507 | `816859554f37` | 2024-03-11 | zhangyu | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.80 |
| 508 | `59020ee828ac` | 2024-03-06 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +10/-1 | feat: add env to control hpRefresh and cvt mode |
| 509 | `145d54ca2f2b` | 2024-03-12 | zhangyu | D | 分屏 splitscreen | +18/-1 | fix: optimize the wayland radius update timing |
| 510 | `361f30812d50` | 2024-03-12 | zhangyu | D | 构建/打包 | +9/-1 | chore: Update version number for 5.27.2.81 |
| 511 | `257e27d89964` | 2024-03-14 | zhangyaning | D | 其它核心 (workspace/window 等) | +0/-3 | fix: remove Reading output's configuration from KScreen config. |
| 512 | `ee6da5bd35b7` | 2024-03-14 | zhijie geng | D | X11/Xwayland | +0/-3 | fix: color temperature doesn't take effect after reboot |
| 513 | `3af6104fbeaf` | 2024-03-14 | zhangyu | D | 多任务视图 multitaskview | +15/-3 | fix: optimized button hover area in multitaskview |
| 514 | `f96ab680a077` | 2024-03-14 | zhangyu | D | 分屏 splitscreen | +1/-0 | fix: fix swap split window out of position |
| 515 | `eca24fc306e3` | 2024-03-14 | huangzeming | D | 其它核心 (workspace/window 等) | +13/-0 | fix: add effectsEnabledChanged signal |
| 516 | `27e2b2d394bc` | 2024-03-14 | dingzhixuan | D | Wayland 协议/服务端 (含 dde 私有协议) | +41/-6 | fix: add workarea in dde-shell |
| 517 | `b75bdb726d63` | 2024-03-14 | dingzhixuan | D | 构建/打包 | +12/-1 | chore: Update version number for 5.27.2.82 |
| 518 | `fbcc6b53b718` | 2024-03-15 | wangyu | D | X11/Xwayland | +22/-2 | fix: Add identifying name to xwl clipboard |
| 519 | `e2785058d1af` | 2024-03-18 | liuzheng | D | 构建/打包 | +6/-0 | chore: add new tag |
| 520 | `e38bd1ab019b` | 2024-03-19 | zhijie geng | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +217/-269 | feat: change title bar icon |
| 521 | `53feff16c231` | 2024-03-15 | huangzeming | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +1/-1 | feat: support alpha channel in border color |
| 522 | `a532641afe02` | 2024-03-18 | huangzeming | D | 输入 | +1/-1 | fix: do not raise 'keepAbove' window while dragging |
| 523 | `ef9e20487fa1` | 2024-03-18 | zhangyu | D | 其它核心 (workspace/window 等) | +28/-25 | fix: display complete split window from maximized state |
| 524 | `f195441301fc` | 2024-03-15 | zhangyaning | D | 显示后端/DRM/多屏 | +7/-0 | fix: reset disabled outputs to off when kwin is reactivating. |
| 525 | `c664f7701061` | 2024-03-15 | zhijie geng | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +66/-27 | feat: implement dynamic adjustment of titlebar hegiht |
| 526 | `3025a199a38a` | 2024-03-19 | zhijie geng | D | 构建/打包 | +14/-1 | chore: Update version number for 5.27.2.84 |
| 527 | `b7d4d9e3642c` | 2024-03-21 | zhijie geng | D | 显示后端/DRM/多屏 | +4/-3 | fix: dde-control-center error when screen is repeatly plugged/unplugged |
| 528 | `f945e5d957dd` | 2024-03-21 | huangzeming | D | 多任务视图 multitaskview | +101/-96 | fix: screen flash when activating multitaskview |
| 529 | `b7ccc8eb8e8a` | 2024-03-22 | huangzeming | D | 其它核心 (workspace/window 等) | +1/-1 | fix: window disappeared after changing screen position |
| 530 | `604cab65e462` | 2024-03-20 | zhangyu | D | 特效 | +1/-4 | feat: optimize clipboard、uos-ai and osd(notification center) effects |
| 531 | `11af2976ba56` | 2024-03-14 | zhangyu | D | 特效 | +126/-118 | feat: showing desktop effects |
| 532 | `e1abfcbcf502` | 2024-03-20 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +2/-0 | fix: add judgment for output. |
| 533 | `c9d1ec6346e9` | 2024-03-21 | zhangyaning | D | X11/Xwayland | +29/-1 | fix: add logical filter code about wayland's app drag to x11's app. |
| 534 | `0a5e085a356f` | 2024-03-22 | wangboyu | D | 构建/打包 | +14/-0 | chore: Update version number for 5.27.2.85 |
| 535 | `0521f7a1b079` | 2024-03-26 | huangzeming | D | 显示后端/DRM/多屏 | +0/-7 | Revert "fix: reset disabled outputs to off when kwin is reactivating." |
| 536 | `c70f46e1ff3d` | 2024-03-26 | zhang yaning | D | X11/Xwayland | +12/-10 | fix: fix some crash point and add fd with O_CLOEXEC. |
| 537 | `4bc09fc6fcdb` | 2024-03-26 | wangboyu | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.86 |
| 538 | `9cfc024fcb93` | 2024-03-27 | zhang yaning | D | 显示后端/DRM/多屏 | +2/-2 | fix: disable common output modes by default. |
| 539 | `bff9f198eb76` | 2024-03-28 | wangboyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.87 |
| 540 | `8f303767b29f` | 2024-03-28 | wangyu | D | X11/Xwayland | +5/-0 | fix: the X window crashes during dragging and dragging again causes kwin coredump |
| 541 | `b8de7baedb27` | 2024-03-28 | wangyu | D | X11/Xwayland | +2/-29 | fix: drag the icon in the taskbar plug-in area to the X11 window without disappearing |
| 542 | `a893570dd931` | 2024-03-29 | huangzeming | D | 分屏 splitscreen | +174/-148 | fix: rewrite splitmenu |
| 543 | `cd52b28098ba` | 2024-03-15 | dingzhixuan | D | Wayland 协议/服务端 (含 dde 私有协议) | +4/-2 | fix: refix add workarea in dde-shell |
| 544 | `c652502488bf` | 2024-03-29 | huangzeming | D | 分屏 splitscreen | +3/-3 | fix: adjust position of split menu |
| 545 | `1445a9be5ae5` | 2024-03-25 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +493/-58 | fix: add effect rules |
| 546 | `730fe8145cdd` | 2024-03-29 | dingzhixuan | D | X11/Xwayland | +62/-75 | fix: tencent meeting cannot passthrough |
| 547 | `d3c8a38aafd7` | 2024-03-29 | dingzhixuan | D | Wayland 协议/服务端 (含 dde 私有协议) | +12/-4 | fix: cannot long screenshot on xwayland window |
| 548 | `5c4f00d60f4d` | 2024-03-29 | dingzhixuan | D | 显示后端/DRM/多屏 | +1/-1 | fix: cursor black margin on uos-update window |
| 549 | `74f92f014b3c` | 2024-03-29 | dingzhixuan | D | 特效 | +3/-1 | fix: dde-control-center is not full display when maximize immediately after start |
| 550 | `6289b7100955` | 2024-03-29 | wangboyu | D | 构建/打包 | +14/-0 | chore: Update version number for 5.27.2.88 |
| 551 | `23f1131ed6dc` | 2024-04-01 | dingzhixuan | D | X11/Xwayland | +0/-1 | fix: tencent meeting cannot passthrough |
| 552 | `d7614938ff39` | 2024-04-01 | dingzhixuan | D | 构建/打包 | +15/-8 | chore: Update version number for 5.27.2.89 |
| 553 | `47e7ec5afc9a` | 2024-04-01 | dingzhixuan | D | 构建/打包 | +1/-1 | chore: Update version number for 5.27.2.89 |
| 554 | `b31b0f37077b` | 2024-04-02 | zhangyu | D | 其它核心 (workspace/window 等) | +19/-13 | fix: optimized font family in menu |
| 555 | `526f29d43f05` | 2024-04-02 | zhangyu | D | 特效 | +2/-2 | feat: optimized timer in showing desktop |
| 556 | `2d155792a735` | 2024-04-02 | huangzeming | D | 分屏 splitscreen | +66/-96 | fix: update icons in split menu |
| 557 | `2314060268b5` | 2024-04-01 | zhijie geng | D | 其它核心 (workspace/window 等) | +21/-8 | fix: wine app render problem |
| 558 | `70aab0bbd415` | 2024-04-02 | huangzeming | D | 多任务视图 multitaskview | +1/-1 | fix: make background in multitaskview darker |
| 559 | `f9b59184dd29` | 2024-03-29 | zhangyaning | D | 其它核心 (workspace/window 等) | +5/-1 | fix: fix some crash point in pgx. |
| 560 | `84387ad4cfc4` | 2024-04-01 | zhangyu | D | 其它核心 (workspace/window 等) | +2/-2 | fix: optimized showing desktop in no compositing |
| 561 | `69e6babdce3d` | 2024-04-02 | huangzeming | D | 其它核心 (workspace/window 等) | +5/-1 | fix: desktop and dde-lock didn't spread over the screen |
| 562 | `1595a627201e` | 2024-04-02 | huangzeming | D | 构建/打包 | +14/-0 | chore: Update version number for 5.27.2.90 |
| 563 | `40d602029f9b` | 2024-04-07 | zhang yaning | D | 显示后端/DRM/多屏 | +2/-2 | Revert "fix: disable common output modes by default." |
| 564 | `ab5a6ed2967c` | 2024-04-07 | zhang yaning | D | 显示后端/DRM/多屏 | +1/-10 | Revert "feat: add env to control hpRefresh and cvt mode" |
| 565 | `c50f4d7b2452` | 2024-04-03 | zhang yaning | D | 显示后端/DRM/多屏 | +28/-1 | fix: support scaling modes. |
| 566 | `d138aa8686ca` | 2024-04-03 | zhangyu | D | 特效 | +4/-3 | fix: optimized unmanaged win effects |
| 567 | `f8b4992229c4` | 2024-04-08 | huangzeming | D | 多任务视图 multitaskview | +1/-0 | fix: pointer penetrates in multitaskview |
| 568 | `5e771ece2465` | 2024-04-08 | zhangyu | D | 其它核心 (workspace/window 等) | +30/-10 | fix: update window radius\shadow when switch compositor |
| 569 | `92ff62f281ab` | 2024-04-08 | dingzhixuan | D | Wayland 协议/服务端 (含 dde 私有协议) | +12/-0 | fix: wl_pointer.wrap send error |
| 570 | `2f4ef441e510` | 2024-04-08 | zhangyu | D | 构建/打包 | +12/-0 | chore: Update version number for 5.27.2.91 |
| 571 | `6897c7de1470` | 2024-04-07 | xinbo wang | D | 其它核心 (workspace/window 等) | +14/-5 | fix: prohibit screen picture display incompleted. |
| 572 | `9a1785eb75ed` | 2024-04-09 | huangzeming | D | 多任务视图 multitaskview | +8/-1 | fix: click event wasn't sent to notification tips in multitaskview |
| 573 | `8c3100413b45` | 2024-04-09 | dingzhixuan | D | Wayland 协议/服务端 (含 dde 私有协议) | +4631/-0 | fix: add xdg-shell-unstable-v6 |
| 574 | `89b638da869f` | 2024-04-09 | dingzhixuan | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.92 |
| 575 | `a248d40b127c` | 2024-04-10 | huangzeming | D | 其它核心 (workspace/window 等) | +9/-1 | fix: find the client error where the event is located |
| 576 | `aea001f3ee2b` | 2024-04-10 | huangzeming | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.93 |
| 577 | `85092547e219` | 2024-04-11 | zhijie geng | D | 其它核心 (workspace/window 等) | +2/-1 | fix: screen flashes black after user logs in |
| 578 | `bef42efc075e` | 2024-04-11 | huangzeming | D | 多任务视图 multitaskview | +10/-2 | fix: 2 desktops are added when press the shortcut only once |
| 579 | `a7442aad9f06` | 2024-04-12 | huangzeming | D | 多任务视图 multitaskview | +93/-79 | fix: unable to render hover frame |
| 580 | `57c01d02e944` | 2024-04-10 | zhangyu | D | 特效 | +1/-1 | fix: optimized blur area when system enable zoom |
| 581 | `6577330b08a0` | 2024-04-12 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +19/-7 | feat: round corner radius optimization |
| 582 | `56cc071959d3` | 2024-04-12 | huangzeming | D | 构建/打包 | +12/-0 | chore: Update version number for 5.27.2.94 |
| 583 | `3b76d0129579` | 2024-04-14 | zhangyu | D | 特效 | +2/-2 | fix: fix rendering anomalies |
| 584 | `cdbadabd0eab` | 2024-04-15 | zhangyu | D | 其它核心 (workspace/window 等) | +7/-3 | fix: add null pointer judgment |
| 585 | `75a019cb04b7` | 2024-04-15 | zhangyu | D | 特效 | +397/-7 | fix: maxi effect |
| 586 | `c8a4195411ba` | 2024-04-15 | dingzhixuan | D | 其它核心 (workspace/window 等) | +11/-1 | fix: popup window will show another screen |
| 587 | `dac670f53c3a` | 2024-04-15 | dingzhixuan | D | 其它核心 (workspace/window 等) | +4/-0 | fix: window show black border on second screen |
| 588 | `47671491a3af` | 2024-04-15 | huangzeming | D | 其它核心 (workspace/window 等) | +6/-4 | fix: menu of wechat can't show by right click on dock |
| 589 | `d04e1ea8f08c` | 2024-04-15 | huangzeming | D | 构建/打包 | +12/-0 | chore: Update version number for 5.27.2.95 |
| 590 | `fcae6eccc818` | 2024-04-16 | huangzeming | D | 其它核心 (workspace/window 等) | +17/-8 | fix: pointer penetrates the menu of wechat |
| 591 | `4269de500dfe` | 2024-04-16 | zhangyaning | D | 多任务视图 multitaskview | +1/-1 | fix: add null pointer judgment |
| 592 | `3c902dc25f47` | 2024-04-16 | dingzhixuan | D | 其它核心 (workspace/window 等) | +1/-8 | fix: refix popup window will show another screen |
| 593 | `4bef9242477c` | 2024-04-16 | dingzhixuan | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.96 |
| 594 | `fd1436a5564c` | 2024-04-15 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +42/-24 | fix: window corner |
| 595 | `482d8ee70c68` | 2024-04-17 | yangtian | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.97 |
| 596 | `c69ad7e9ffc2` | 2024-04-17 | zhangyaning | D | 显示后端/DRM/多屏 | +19/-17 | fix: update dmidecode's command about identify pgx's machine. |
| 597 | `cdf9db912861` | 2024-04-17 | zhijie geng | D | 构建/打包 | +8/-1 | chore: Update version number for 5.27.2.98 |
| 598 | `690dad23214f` | 2024-04-08 | huangzeming | D | 其它核心 (workspace/window 等) | +55/-31 | fix: dock flashes when changing effect type |
| 599 | `8cecb62ad4af` | 2024-04-18 | huangzeming | D | 其它核心 (workspace/window 等) | +95/-39 | feat: expand mouseInputAreaMargins to 10px |
| 600 | `5af3fa9f1987` | 2024-04-19 | zhangyu | D | 特效 | +1/-0 | fix: Limit the maximum frequency of operations |
| 601 | `d42a617d4991` | 2024-04-19 | zhangyu | D | 特效 | +1/-1 | feat: optimized effect time of maximize |
| 602 | `58f5f30ffa58` | 2024-04-19 | zhijie geng | D | 特效 | +1/-1 | fix: maximize/unmaximize ciname causes secondary screen to flicker |
| 603 | `c9ee3bedde4e` | 2024-04-19 | zhangyu | D | 其它核心 (workspace/window 等) | +34/-4 | feat: adjust split window exit rule |
| 604 | `5c752f50a540` | 2024-04-19 | zhangyu | D | 构建/打包 | +7/-0 | chore: Update number for 5.27.2.99 |
| 605 | `119f38737b12` | 2024-04-23 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +3/-3 | fix: adjust the rule of wayland window shadow |
| 606 | `8c8d69c7fd7c` | 2024-04-26 | zhijie geng | D | 其它核心 (workspace/window 等) | +2/-1 | fix: showing desktop don't work |
| 607 | `ebba44871654` | 2024-03-14 | xinbo wang | D | Wayland 协议/服务端 (含 dde 私有协议) | +759/-6 | feat: add kvm interface |
| 608 | `638e0f622b94` | 2024-05-09 | dingzhixuan | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.200 |
| 609 | `818eaef382c6` | 2024-05-10 | dingzhixuan | D | Wayland 协议/服务端 (含 dde 私有协议) | +13/-6 | fix: Prevent multiple hiding cursor. |
| 610 | `98cf8a066f9a` | 2024-05-10 | dingzhixuan | D | 构建/打包 | +8/-1 | chore: Update version number for 5.27.2.201 |
| 611 | `ec0366e6233e` | 2024-04-12 | zhangyu | D | 特效 | +8/-5 | fix: fix dde-lock effect |
| 612 | `a24563a47c7c` | 2024-04-23 | yangtian | D | 多任务视图 multitaskview | +2/-0 | fix: window movement effect when removing workspace |
| 613 | `d7e269f09538` | 2024-05-09 | yangtian | D | 多任务视图 multitaskview | +31/-12 | feat: multitaskview background blur optimization |
| 614 | `8932d6d2092c` | 2024-05-09 | yangtian | D | 分屏 splitscreen | +159/-3 | feat: splitpreview and splitswap effect background blur mask |
| 615 | `453d6c0ece1b` | 2024-05-11 | yangtian | D | 特效 | +3/-0 | fix: watermark display when alttab switching to desktop |
| 616 | `4ffa12bfbc28` | 2024-05-13 | yangtian | D | 特效 | +10/-5 | fix: fix launcher effect |
| 617 | `64c5f0f61293` | 2024-05-10 | yangtian | D | 其它核心 (workspace/window 等) | +33/-1 | fix: sliding workspace but overlapped other screens |
| 618 | `3283fdaa76e0` | 2024-04-23 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +18/-18 | fix: optimized window param |
| 619 | `015b5b473b12` | 2024-05-09 | chenjiafeng | D | Wayland 协议/服务端 (含 dde 私有协议) | +23/-3 | fix: kwin_wayland添加no-vt-switch参数 |
| 620 | `413a5684819b` | 2024-05-11 | zhangyu | D | 其它核心 (workspace/window 等) | +7/-1 | fix: repair about window can't minimized |
| 621 | `fe85e7f34f69` | 2024-05-14 | dingzhixuan | D | 其它核心 (workspace/window 等) | +1/-0 | fix: dde-lock won't show when preview windows |
| 622 | `c691022aa122` | 2024-05-14 | dingzhixuan | D | 其它核心 (workspace/window 等) | +8/-4 | fix: crash when maximize by using shortcut |
| 623 | `06d5d75570ee` | 2024-05-14 | dingzhixuan | D | 其它核心 (workspace/window 等) | +7/-1 | fix: remove dock extend border |
| 624 | `360e7dc921de` | 2024-05-15 | dingzhixuan | D | 构建/打包 | +19/-0 | chore: Update version number for 5.27.2.202 |
| 625 | `1c593d20a7af` | 2024-05-23 | chenjiafeng | D | 显示后端/DRM/多屏 | +24/-0 | feat: add function for select display output devices by outputlayoutrc file |
| 626 | `75e2e52a8769` | 2024-05-15 | chengong | D | 显示后端/DRM/多屏 | +80/-5 | feat: Support NV12 format dmabuf for direct rendering compositing in kwin |
| 627 | `5c3c670b8cc7` | 2024-05-24 | zhang yu | D | 分屏 splitscreen | +32/-4 | fix: fix about split preview crash |
| 628 | `3bde46ecb261` | 2024-05-29 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +1/-1 | fix: update special window with clipdata when the window geometrychanged |
| 629 | `e439056825b4` | 2024-05-28 | huangzeming | D | Wayland 协议/服务端 (含 dde 私有协议) | +4/-2 | fix: can't select dde-control-center in screenshot |
| 630 | `8db228ac7cdb` | 2024-05-27 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +4/-553 | fix:  clean up invalid processes |
| 631 | `3765ef1b5e81` | 2024-05-07 | zhangyaning | A | 输入 | +8/-0 | fix: update keyboard focus when grabbing popup closed. |
| 632 | `fd9185060f18` | 2024-04-30 | zhangyu | D | X11/Xwayland | +1/-2 | fix: cancel the opacity of desktop |
| 633 | `2633884a2f39` | 2024-04-29 | huangzeming | D | 其它核心 (workspace/window 等) | +1/-2 | fix: window is raised even it's below after dragging released |
| 634 | `2ad40539cde5` | 2024-04-24 | zhangyu | D | 特效 | +11/-11 | fix: optimized maxi effect pointer |
| 635 | `f27548f2e56d` | 2024-05-29 | dingzhixuan | D | 构建/打包 | +17/-0 | chore: Update version number for 5.27.2.203 |
| 636 | `b4a9cf01cf0b` | 2024-05-31 | huangzeming | D | 特效 | +956/-7 | feat: present thumbnail in tabbox |
| 637 | `6b8fc2dcc4fc` | 2024-05-31 | huangzeming | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.204 |
| 638 | `2bd16957fecd` | 2023-03-21 | Vlad Zahorodnii | A | X11/Xwayland | +30/-16 | feat: Refactor compositing teardown in X11Window and Unmanaged |
| 639 | `39ecad98ff61` | 2024-06-05 | zhangyu | D | X11/Xwayland | +1/-1 | fix: the override window set border in xrender |
| 640 | `4986a6081274` | 2024-05-31 | zhangyu | D | 特效 | +0/-7 | fix: fix maxieffect grab half of events |
| 641 | `31a7adea95fc` | 2024-05-29 | dingzhixuan | D | 多任务视图 multitaskview | +11/-6 | fix: tip show error in multitaskview |
| 642 | `dbb2bf366d4f` | 2024-06-05 | huangzeming | D | 特效 | +2/-1 | fix: tabbox window didn't adjust dock |
| 643 | `f0e5db738bf6` | 2024-06-05 | huangzeming | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +14/-0 | fix: blur region error after compositing toggled |
| 644 | `1f952b5f4b1b` | 2024-06-05 | huangzeming | D | 其它核心 (workspace/window 等) | +2/-0 | fix: button of multitaskview was shown on 2d mode |
| 645 | `0259963decf9` | 2024-06-07 | huangzeming | D | 特效 | +14/-13 | feat: modify ui parameter |
| 646 | `94aae3db6d3f` | 2024-05-24 | yangtian | D | X11/Xwayland | +3/-1 | fix: Cancel the maximum restore operation in split screen |
| 647 | `52a5894edffb` | 2024-05-30 | yangtian | D | 其它核心 (workspace/window 等) | +27/-2 | fix: add disablesplitpreview rule for workwx |
| 648 | `86f9cb969b39` | 2024-06-07 | yangtian | D | Alt+Tab 切换器 | +1/-1 | fix: switcherItem spacing |
| 649 | `445352448cb5` | 2024-05-24 | yangtian | D | 分屏 splitscreen | +5/-2 | fix: Adjusting the split screen rules for special windows |
| 650 | `bf4a2e2987b4` | 2024-06-02 | dingzhixuan | D | 其它核心 (workspace/window 等) | +41/-0 | feat: add icbc prohibitscreenshot |
| 651 | `c80efb16abec` | 2024-06-13 | zhangyu | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.205 |
| 652 | `fd80a5cab0ba` | 2024-06-14 | huangzhenyan | D | 显示后端/DRM/多屏 | +5/-1 | fix: Set HDMI screen rotation, causing mouse misalignment and screen blur |
| 653 | `38274d82ba3f` | 2024-04-24 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +1/-1 | fix: fix crash during frequent screen plugging and unplugging |
| 654 | `bdbc78fa728e` | 2024-06-05 | yangtian | D | 多任务视图 multitaskview | +93/-43 | fix: fix multitaskview calling WM signal |
| 655 | `ef48c70897c8` | 2024-06-13 | yangtian | D | 分屏 splitscreen | +17/-7 | fix: add splitmenu border |
| 656 | `10e2443f6570` | 2024-06-14 | yangtian | D | 多任务视图 multitaskview | +1/-1 | fix: launcher exit effects in multitaskview |
| 657 | `99f53fd93483` | 2024-06-04 | huangzeming | D | 其它核心 (workspace/window 等) | +54/-0 | fix: coredump if xorg exited before kwin |
| 658 | `22654b0399e4` | 2024-06-17 | huangzeming | D | 特效 | +291/-255 | fix: use texture instead of qml in thumbnaillist |
| 659 | `f8af8ac0e672` | 2024-06-18 | huangzeming | D | Alt+Tab 切换器 | +1/-0 | fix: incorrect layout in tabbox thumbnail list |
| 660 | `abff7fc78d56` | 2024-06-18 | huangzeming | D | 其它核心 (workspace/window 等) | +3/-2 | fix: number of desktop is 1 after upgrading from 1054 to 1071 |
| 661 | `e5806a88000e` | 2024-05-20 | zhangyu | D | 其它核心 (workspace/window 等) | +3/-0 | fix: Fix split window display exceeding screen issue |
| 662 | `e077a3fc5f3c` | 2024-06-14 | zhangyu | D | 分屏 splitscreen | +0/-1 | fix: fix issue about desktop's brightness when split window swap |
| 663 | `c33722fa565a` | 2024-06-19 | huangzeming | D | 构建/打包 | +18/-1 | chore: Update version number for 5.27.2.206 |
| 664 | `2b02a7e99cc0` | 2024-06-20 | huangzeming | D | 其它核心 (workspace/window 等) | +26/-24 | feat: add icbc prohibitscreenshot |
| 665 | `98de74c393fc` | 2024-06-20 | huangzeming | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.207 |
| 666 | `a138eb4b2f97` | 2024-06-24 | huangzhenyan | D | 显示后端/DRM/多屏 | +2/-0 | fix: Mode setting failed when plugging in monitor |
| 667 | `fca0f35cbcba` | 2024-06-18 | yangtian | D | 多任务视图 multitaskview | +75/-3 | fix: dconfig question |
| 668 | `5e6e3a785d8c` | 2024-06-20 | yangtian | D | 多任务视图 multitaskview | +1/-1 | fix: fix repetitive multitaskview wallpaper |
| 669 | `862be78ce77d` | 2024-06-25 | yangtian | D | 特效 | +3/-0 | fix: multitaskview grab status when two screenrecorder window closed |
| 670 | `cd410916f96b` | 2024-06-25 | yangtian | D | 多任务视图 multitaskview | +40/-20 | fix: dde-kwin get currently workspace background is not sync when kwin add new workspace |
| 671 | `70b3a262d39c` | 2024-06-21 | zhangyu | D | 分屏 splitscreen | +7/-7 | fix: fix error log |
| 672 | `49ac0590747a` | 2024-06-19 | zhangyu | D | 多任务视图 multitaskview | +12/-4 | fix: repair issue about dottedline when drop workspace in wayland |
| 673 | `01bd098fe60d` | 2024-06-26 | zhangyu | D | 输入 | +1/-0 | fix: fix issue about click dock tray no response |
| 674 | `14f1b053065e` | 2024-05-21 | zhangyu | D | 其它核心 (workspace/window 等) | +3/-0 | fix: right click menu follows mouse |
| 675 | `c9a441789341` | 2024-06-18 | huangzeming | D | 特效 | +39/-3 | fix: show dock in tabbox |
| 676 | `ae567871d24c` | 2024-06-20 | huangzeming | D | 特效 | +5/-1 | fix: reduce the brightness of desktop in tabbox |
| 677 | `3c9c6f33fca4` | 2024-06-24 | huangzeming | D | 其它核心 (workspace/window 等) | +13/-3 | fix: keep icon consistent with control center |
| 678 | `df721c03672f` | 2024-06-25 | huangzeming | D | Alt+Tab 切换器 | +14/-4 | fix: reduce respond frequency while pressing keys in tabbox |
| 679 | `8c4a69b0888b` | 2024-06-26 | yangtian | D | 构建/打包 | +19/-0 | chore: Update version number for 5.27.2.208 |
| 680 | `49ce09fee4f2` | 2024-05-23 | dingzhixuan | A | 其它核心 (workspace/window 等) | +11/-14 | fix: xdg popup position incorrect |
| 681 | `66abd38bb5e0` | 2024-07-03 | huangzeming | D | 特效 | +2/-1 | fix: tabbox window bottom margin |
| 682 | `ef2d77e849b7` | 2024-07-02 | huangzeming | D | Alt+Tab 切换器 | +6/-3 | fix: crash when toggling compositor |
| 683 | `9bb5476533ff` | 2024-07-03 | huangzeming | D | Alt+Tab 切换器 | +2/-1 | fix: delay key '~' in tabbox |
| 684 | `af756d986d36` | 2024-07-03 | huangzeming | D | 特效 | +5/-5 | fix: minimized windows flicker after tabbox hid |
| 685 | `27bb547a3853` | 2024-06-27 | yangtian | D | 多任务视图 multitaskview | +4/-0 | fix: dont active multitaskview effect when user move or resize window |
| 686 | `0e8f82c336ca` | 2024-06-26 | yangtian | D | 翻译 | +32/-0 | fix: Tibetan translation of kwin menu |
| 687 | `aaa10c8c3e3c` | 2024-07-03 | huangzhenyan | D | 构建/打包 | +14/-0 | chore: Update version number for 5.27.2.209 |
| 688 | `cc9418eb17e4` | 2024-06-18 | yangtian | D | 多任务视图 multitaskview | +3/-4 | fix: multitaskview showWorkspacePreview offset in wayland |
| 689 | `508e0542b094` | 2024-07-09 | yangtian | D | 分屏 splitscreen | +7/-1 | fix: add timer to exit splitpreview for client damage |
| 690 | `f8a84a74cf1a` | 2024-07-04 | yangtian | D | 特效 | +1/-3 | fix: the squash effect in quick previewwindow was not cleared |
| 691 | `ba5b5c755901` | 2024-07-08 | huangzeming | D | 特效 | +31/-1 | fix: watermark below the thumbnail in tabbox |
| 692 | `b1f1a214db61` | 2024-07-10 | huangzeming | D | 多任务视图 multitaskview | +2/-2 | fix: workspace close button on touchscreen in multitaskview |
| 693 | `0b0aea5ad087` | 2024-07-10 | huangzeming | D | Alt+Tab 切换器 | +5/-2 | fix: minimized window flickers if click outside the tabbox |
| 694 | `ba92882b7c07` | 2024-07-11 | yangtian | D | 构建/打包 | +12/-0 | chore: Update version number for 5.27.2.210 |
| 695 | `8da3eae7cf70` | 2024-07-16 | yangtian | A | 其它核心 (workspace/window 等) | +0/-3 | fix: stop setting XCURSOR_SIZE |
| 696 | `bfa77eb5ba58` | 2024-07-15 | huangzeming | D | 其它核心 (workspace/window 等) | +1/-1 | fix: kwin coredump after logging out |
| 697 | `bd825cdb079e` | 2024-07-15 | huangzeming | D | X11/Xwayland | +6/-5 | fix: wechat window raised when compositor toggled |
| 698 | `32beae350762` | 2024-07-16 | huangzeming | D | 其它核心 (workspace/window 等) | +4/-0 | fix: update icon when icon theme changed |
| 699 | `7159699967de` | 2024-07-16 | huangzeming | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +47/-43 | fix: keep color of titlebar consistent with theme |
| 700 | `4b67f52daa5a` | 2024-07-17 | zhangyaning | D | 其它核心 (workspace/window 等) | +3/-1 | fix: disconnect the connection when the WindowItem object is destroyed. |
| 701 | `b37fc2e54218` | 2024-07-17 | huangzeming | D | 构建/打包 | +12/-0 | chore: Update version number for 5.27.2.211 |
| 702 | `3992849fd039` | 2024-07-22 | huangzeming | D | 其它核心 (workspace/window 等) | +3/-0 | fix: add restrictions in 'sendWinodwToDesktop' |
| 703 | `8af0d7aa0cd9` | 2024-07-22 | huangzeming | D | 特效 | +230/-150 | feat: add scroll and select animation in tabbox |
| 704 | `009e24640f05` | 2024-07-22 | yangtian | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.212 |
| 705 | `2d82fd1c5b9b` | 2024-07-24 | zhangyu | D | 其它核心 (workspace/window 等) | +47/-55 | fix: repair about dock don't display when show unmanaged win |
| 706 | `199cb03bdbd0` | 2024-07-25 | huangzeming | D | 特效 | +3/-2 | fix: tabbox window bottom margin when dock on the top of screen |
| 707 | `5897a55c5997` | 2024-07-25 | huangzeming | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.213 |
| 708 | `f7ccfe7f9728` | 2024-08-13 | huangzeming | D | 构建/打包 | +3/-1 | fix: add libx11-6 depends in kwin-x11 |
| 709 | `c10c753848e1` | 2024-05-15 | zhijie geng | D | Wayland 协议/服务端 (含 dde 私有协议) | +5/-0 | fix: subsurface window's position can't update immediately |
| 710 | `00669d2f8bc7` | 2024-01-15 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +27/-4 | gbm_bo_create_with_modifiers2 and gbm_bo_get_fd_for_plane |
| 711 | `7aa514032de2` | 2024-05-30 | dingzhixuan | D | 显示后端/DRM/多屏 | +21/-20 | fix: remote access buffer tearing |
| 712 | `1c4b9cfca051` | 2024-06-07 | dingzhixuan | D | 显示后端/DRM/多屏 | +8/-2 | fix: pgx warmboot or coldboot crash |
| 713 | `282cd1049d1d` | 2024-07-04 | dingzhixuan | D | 显示后端/DRM/多屏 | +13/-4 | feat: support hw cursor on pgx |
| 714 | `fc18056d5e1d` | 2024-07-10 | dingzhixuan | A | 其它核心 (workspace/window 等) | +1/-1 | fix: glmark2 will change size when plugin and plugout screen |
| 715 | `d38121a7b7ee` | 2024-07-24 | dingzhixuan | D | 其它核心 (workspace/window 等) | +74/-4 | fix: add edge software cursor item in pgx |
| 716 | `09d6ad7a2cbd` | 2024-05-22 | zhangyaning | D | 显示后端/DRM/多屏 | +41/-4 | fix: add judgment of update modes. |
| 717 | `fd57f88d2bdf` | 2024-08-06 | dingzhixuan | D | 其它核心 (workspace/window 等) | +17/-2 | fix: soft cursor will blink on pgx |
| 718 | `dd8ab6cf8c4e` | 2024-08-07 | dingzhixuan | D | 显示后端/DRM/多屏 | +1/-12 | fix: remove virtual output |
| 719 | `5affbbc8976b` | 2024-08-13 | dingzhixuan | D | 构建/打包 | +16/-0 | chore: Update version number for 5.27.2.260 |
| 720 | `a7dcc60aee3f` | 2024-08-09 | huangzhenyan | D | 其它核心 (workspace/window 等) | +7/-4 | fix: window title bar have gap between screen edge |
| 721 | `84fa63616a57` | 2024-08-08 | xinbo wang | D | 其它核心 (workspace/window 等) | +1/-1 | fix: RRScreenChangNoitify make desktop and fullscreen type window can't keep old geometry |
| 722 | `737b30a7c6da` | 2024-08-09 | yangtian | D | 多任务视图 multitaskview | +1/-1 | fix: the judgement of workspace preview |
| 723 | `3fce292c5ad3` | 2024-08-09 | huangzeming | D | 特效 | +6/-0 | fix: maximize animation while dragging maximized window |
| 724 | `0511c33caa92` | 2024-08-09 | huangzeming | D | 其它核心 (workspace/window 等) | +14/-1 | fix: unable to raise window below shape window |
| 725 | `d1c1b233ad06` | 2024-08-12 | yangtian | D | X11/Xwayland | +10/-2 | fix: add timestamp for x11window |
| 726 | `442cd7a8d972` | 2024-08-13 | yangtian | D | 输入 | +16/-4 | fix: wayland cursor size based scale factor |
| 727 | `07e7206a1338` | 2024-08-14 | yangtian | D | 构建/打包 | +13/-0 | chore: Update version number for 5.27.2.261 |
| 728 | `381024a1fba9` | 2024-08-21 | dingzhixuan | D | 显示后端/DRM/多屏 | +4/-13 | Revert "feat: support hw cursor on pgx" |
| 729 | `1a45db78b2f3` | 2024-08-21 | dingzhixuan | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.262 |
| 730 | `29fa2c3f9312` | 2024-08-15 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +1/-1 | fix: fix issue about desktop has radius |
| 731 | `84cead079d1c` | 2024-08-15 | zhangyu | D | 多任务视图 multitaskview | +5/-5 | fix: don't show menu when quit multitask |
| 732 | `60cd3dcb5684` | 2024-08-15 | zhangyu | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +2/-2 | fix: keep color of titlebar consistent with theme |
| 733 | `8059037f02da` | 2024-08-15 | huangzhenyan | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +133/-168 | fix: atom read in xwayland get XCB_NONE |
| 734 | `12a8cd10ac86` | 2024-08-26 | huangzhenyan | D | X11/Xwayland | +3/-1 | fix: ignore grab/ungrab in x11 focus in |
| 735 | `914e356a1703` | 2024-08-26 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +15/-15 | fix: avoid capture hanging ref in unref |
| 736 | `074379ce7e98` | 2024-08-28 | dingzhixuan | D | 显示后端/DRM/多屏 | +13/-4 | Reapply "feat: support hw cursor on pgx" |
| 737 | `bdbc59ced22a` | 2024-08-29 | dingzhixuan | D | 显示后端/DRM/多屏 | +1/-1 | fix: [drm] donnot use hardware cursor if no cursor plane |
| 738 | `8b10d5c2afe4` | 2024-08-29 | dingzhixuan | A | 其它核心 (workspace/window 等) | +8/-9 | fix: Improve Workspace::outputAt() |
| 739 | `ae176d97cfd9` | 2024-08-28 | dingzhixuan | D | 分屏 splitscreen | +8/-2 | fix: [splitscreen] preview show under window |
| 740 | `7abffaed321d` | 2024-08-28 | dingzhixuan | D | 其它核心 (workspace/window 等) | +3/-0 | fix: restore window will blink other screen |
| 741 | `e4751bc1f803` | 2024-08-22 | dingzhixuan | D | 其它核心 (workspace/window 等) | +13/-2 | fix: add interface to perferred cursor show output |
| 742 | `6850fa02fb7d` | 2024-07-25 | dingzhixuan | D | 其它核心 (workspace/window 等) | +46/-6 | fix: screenshot donnot show cursor |
| 743 | `97e01dd82873` | 2024-08-01 | dingzhixuan | D | 输入 | +21/-4 | fix: refix wemeet donnot show cursor when screen record |
| 744 | `fb2416e529a9` | 2024-08-16 | dingzhixuan | D | 显示后端/DRM/多屏 | +19/-9 | fix: [drm] pgx displaylink show black screen |
| 745 | `486173a1b073` | 2024-08-22 | huangzhenyan | D | 构建/打包 | +20/-0 | chore: Update version number for 5.27.2.263 |
| 746 | `4568aae44b37` | 2024-08-28 | huangzhenyan | B | 特效 | +23/-14 | fix: sunlogin's image has flipped |
| 747 | `2a8772e3461a` | 2024-09-11 | dingzhixuan | D | 构建/打包 | +5/-2 | fix: print version info to log |
| 748 | `525c21c08098` | 2024-09-04 | dingzhixuan | D | 其它核心 (workspace/window 等) | +1/-0 | fix: cursor not update after output transform |
| 749 | `94709b9ea183` | 2024-09-06 | zhangyaning | D | 其它核心 (workspace/window 等) | +1/-1 | fix: remove dbus's interface of Scripting. |
| 750 | `40a955f8f9c1` | 2024-09-06 | zhangyu | D | 特效 | +49/-13 | fix: optimize window radius in multitask and splitpreview |
| 751 | `82e18bc3ddf2` | 2024-08-26 | huangzeming | D | 其它核心 (workspace/window 等) | +25/-14 | fix: raise when release only available for dde-file-manager |
| 752 | `7a95a898992a` | 2024-09-04 | huangzeming | B | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +39/-21 | fix: some crash |
| 753 | `411e735dfc89` | 2024-09-11 | dingzhixuan | D | 构建/打包 | +13/-0 | chore: Update version number for 5.27.2.264 |
| 754 | `dd75e46b4372` | 2024-09-12 | huangzhenyan | D | 特效 | +3/-1 | fix: wechat screenshot blink |
| 755 | `40ebf1594240` | 2024-09-15 | huangzhenyan | D | 其它核心 (workspace/window 等) | +64/-36 | feat: improve glmark2 score |
| 756 | `5268fc47e0f9` | 2024-08-09 | yangtian | D | 其它核心 (workspace/window 等) | +11/-22 | fix: cascade placement |
| 757 | `49a781635ce6` | 2024-09-13 | zhangyu | D | 其它核心 (workspace/window 等) | +124/-15 | feat: add log |
| 758 | `0db597e742ef` | 2024-09-19 | huangzhenyan | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.265 |
| 759 | `a2c2c9dfc6de` | 2024-09-26 | huangzhenyan | D | 显示后端/DRM/多屏 | +20/-20 | fix: memory stomping when unplugging output |
| 760 | `3f8ef6eeb40f` | 2024-09-26 | huangzeming | D | 其它核心 (workspace/window 等) | +9/-6 | fix: set dconfig user type base on backend |
| 761 | `590729cb7f3c` | 2024-09-26 | huangzeming | D | 显示后端/DRM/多屏 | +5/-1 | fix: coredump in eglDestroyImageKHR |
| 762 | `8ee24b6d75b6` | 2024-09-26 | huangzeming | D | 其它核心 (workspace/window 等) | +19/-2 | fix: avoid null pointer in windowItem |
| 763 | `a644244c5ad5` | 2024-09-26 | huangzeming | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +9/-2 | fix: decoration button lost hoverLeave event |
| 764 | `85f34ff4ff95` | 2024-09-26 | yangtian | D | 构建/打包 | +11/-0 | chore: Update version number for 5.27.2.266 |
| 765 | `0372da95d5e7` | 2024-09-30 | zhangyu | D | 其它核心 (workspace/window 等) | +11/-1 | fix: fix issue of failure to change the position of splitscreen windows |
| 766 | `7cbbf75a2775` | 2024-06-14 | zhangyu | D | 翻译 | +9245/-0 | fix: Improve language support |
| 767 | `f40eb0864d48` | 2024-09-29 | zhangyu | D | 其它核心 (workspace/window 等) | +226/-7 | feat: add interface to save debug pixmap of window |
| 768 | `0b4c929f11cd` | 2024-09-30 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +13/-8 | fix: screen blink when plugin in extend mode |
| 769 | `b9528b22d9b6` | 2024-10-11 | dingzhixuan | D | 其它核心 (workspace/window 等) | +38/-8 | fix: [composite] lock cannot cover desktop |
| 770 | `e51d513bac8d` | 2024-10-08 | xinbo wang | D | 其它核心 (workspace/window 等) | +3/-1 | fix: remove upstream Meta shortcus in X11. |
| 771 | `9f389c592671` | 2024-09-27 | yangtian | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +1/-1 | fix: Invisible buttons do not display split screen menu |
| 772 | `2f1958395df7` | 2024-10-08 | zhangyaning | D | Wayland 协议/服务端 (含 dde 私有协议) | +4/-0 | fix: add judgment of dataSource. |
| 773 | `4b352223d2f7` | 2024-10-12 | huangzeming | D | 构建/打包 | +14/-0 | chore: Update version number for 5.27.2.267 |
| 774 | `54f6f4836552` | 2024-10-17 | Liu Zheng | D | Alt+Tab 切换器 | +21/-1 | fix: the problem of frozen screen in tabbox pressure test |
| 775 | `3009799c1af0` | 2024-10-17 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +8/-13 | Revert "fix: screen blink when plugin in extend mode" |
| 776 | `80bc8eebce0f` | 2024-10-17 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +63/-59 | fix: x11perf+glmark2 crash |
| 777 | `b1fb8d273e7e` | 2024-10-17 | huangzhenyan | D | 构建/打包 | +8/-0 | feat: add memcheck for kwin |
| 778 | `5871ecec40c4` | 2024-10-18 | huangzeming | D | X11/Xwayland | +1/-1 | fix: Avoid creating Unmanaged for reparented window |
| 779 | `6d3c7a73b9ee` | 2024-10-18 | huangzeming | D | 多任务视图 multitaskview | +3/-0 | fix: disable scrolling if only 1 workspace in multitaskview |
| 780 | `f8e858631b34` | 2024-10-17 | yangtian | D | 其它核心 (workspace/window 等) | +6/-1 | fix: xdgpopup window movement is error |
| 781 | `a9eb2a072789` | 2024-10-18 | dingzhixuan | D | 显示后端/DRM/多屏 | +1/-3 | fix: [drm] remove update mode limit when mode changed |
| 782 | `3686b53ec24c` | 2024-10-18 | huangzhenyan | D | 构建/打包 | +13/-0 | chore: Update version number for 5.27.2.268 |
| 783 | `e1880ac6af59` | 2024-10-18 | huangzhenyan | D | 显示后端/DRM/多屏 | +78/-16 | fix: screen blink when resize/move/hotpluging |
| 784 | `20dd5e356420` | 2024-10-21 | huangzhenyan | D | 分屏 splitscreen | +50/-17 | fix: splitswap effect blocking |
| 785 | `170ba0529785` | 2024-10-23 | huangzeming | D | 其它核心 (workspace/window 等) | +10/-2 | fix: light scroll lock led |
| 786 | `56085a42e297` | 2024-10-24 | huangzhenyan | D | Alt+Tab 切换器 | +1/-21 | Revert "fix: the problem of frozen screen in tabbox pressure test" |
| 787 | `75a0eba72fb0` | 2024-10-22 | zhangyu | D | 其它核心 (workspace/window 等) | +5/-2 | fix: optimize debug pixmap bug |
| 788 | `386cb2e87245` | 2024-10-22 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +2/-2 | fix: glmark2's name can't be retrieved under root |
| 789 | `449fa440d5b8` | 2024-10-24 | dingzhixuan | D | 构建/打包 | +11/-0 | chore: Update version number for 5.27.2.269 |
| 790 | `846f3e9b033b` | 2024-10-25 | dingzhixuan | D | 显示后端/DRM/多屏 | +3/-3 | fix: [wayland] colorcurves init incorrect |
| 791 | `70edbc57e0d1` | 2024-10-25 | dingzhixuan | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.270 |
| 792 | `ad063264ed44` | 2024-10-22 | zhangyu | D | 其它核心 (workspace/window 等) | +50/-2 | fix: disable window moving or resizing when trigger other shortcut |
| 793 | `af08c9755c01` | 2024-10-28 | huangzhenyan | D | 其它核心 (workspace/window 等) | +4/-1 | fix: glmark2 blink |
| 794 | `7c2fca467d6e` | 2024-10-14 | xinbo wang | D | 其它核心 (workspace/window 等) | +2/-1 | fix: change split trigger effect in Xrender. |
| 795 | `7d3d29e17ee0` | 2024-10-29 | xinbo wang | D | 其它核心 (workspace/window 等) | +11/-0 | fix: when active window changed, we finish moveresize window. |
| 796 | `e048db06ba01` | 2024-10-30 | yangtian | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.271 |
| 797 | `83bcdea927d9` | 2024-11-01 | huangzeming | D | 其它核心 (workspace/window 等) | +64/-29 | fix: adjust window shadow rules |
| 798 | `902b95c471fe` | 2024-10-28 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +26/-17 | fix: deepin-screen-recorder recording GIF, kwin_wayland crashes |
| 799 | `f3ef20f3b32f` | 2024-11-01 | Chaojiang Luo | D | 显示后端/DRM/多屏 | +10/-3 | backends/x11: fix colormap leak |
| 800 | `20ba9a981bf0` | 2024-10-28 | yangtian | D | X11/Xwayland | +58/-4 | fix: dont set noborder property for _DEEPIN_FORCE_DECORATE in xrender |
| 801 | `12febc0bd34d` | 2024-11-06 | huangzeming | D | 构建/打包 | +10/-0 | chore: Update version number for 5.27.2.272 |
| 802 | `00fdee27c4b2` | 2024-11-27 | huangzeming | D | 其它核心 (workspace/window 等) | +21/-24 | fix: start prohibit screenshot dbus in subthread |
| 803 | `d289b6e01a2d` | 2024-11-22 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +2/-2 | fix: window caption contains "glmark2" will increase latency |
| 804 | `6fa5e6573803` | 2024-11-27 | huangzeming | D | 构建/打包 | +8/-0 | chore: Update version number for 5.27.2.273 |
| 805 | `c17678689877` | 2024-11-29 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +34/-9 | fix: delay done event after all changes are emitted |
| 806 | `4a541cb99948` | 2024-10-30 | huangzhenyan | D | 其它核心 (workspace/window 等) | +13/-6 | feat: add XWAYLAND_FORCE_SUPPORT_GLAMOR env |
| 807 | `a6326aa7eb1a` | 2024-12-03 | zhangyaning | D | 录屏/截图/远程 | +2/-1 | fix: setYInverted when GLTexture is already interted of Y. |
| 808 | `123397872925` | 2024-12-11 | yangtian | D | 构建/打包 | +9/-0 | chore: Update version number for 5.27.2.274 |
| 809 | `700dc3ad629c` | 2024-12-24 | huangzeming | D | 其它核心 (workspace/window 等) | +5/-4 | fix: cancel shadow for window with alpha channel |
| 810 | `835a2df206e1` | 2024-12-24 | huangzeming | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.275 |
| 811 | `25ef3ba8b365` | 2024-11-23 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +0/-6 | fix: mode id confilt |
| 812 | `9334f142ea3d` | 2024-12-31 | YaoBing Xiao | D | 构建/打包 | +7/-0 | chore: Update version number for 5.27.2.276 |
| 813 | `0582c3739cce` | 2025-01-03 | Yixue Wang | D | 分屏 splitscreen | +3/-2 | fix: crash in splitmenu event filter |
| 814 | `17a3bbc10290` | 2025-01-09 | justforlxz | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +1/-1 | fix: missing window alpha check |
| 815 | `a6055b2ac08b` | 2025-01-10 | pengwenhao | D | 显示后端/DRM/多屏 | +8/-8 | fix: remote access gbm memory leak |
| 816 | `22a79610224a` | 2025-01-10 | pengwenhao | D | 构建/打包 | +32/-23 | chore: Update version number for 5.27.2.277 |
| 817 | `228890d5dbda` | 2025-01-10 | justforlxz | D | 其它核心 (workspace/window 等) | +14/-6 | fix: missing libgbm.so.1 check |
| 818 | `26561792e887` | 2025-01-11 | Yixue Wang | D | Wayland 协议/服务端 (含 dde 私有协议) | +6/-0 | Revert "fix: mode id confilt" |
| 819 | `c9154b408258` | 2025-01-11 | Yixue Wang | D | 构建/打包 | +8/-0 | chore: update version number for 5.27.2.278 |
| 820 | `762bf4252198` | 2024-12-25 | huangzeming | D | 特效 | +2/-0 | fix: force blur in squash effect |
| 821 | `d7d293dfcc34` | 2025-01-15 | guoyao | D | 构建/打包 | +7/-0 | chore: update version number for 5.27.2.279 |
| 822 | `ec4203b68df0` | 2025-01-04 | YaoBing Xiao | D | 其它核心 (workspace/window 等) | +1/-43 | fix: Lag during composition after mode modification |
| 823 | `99b880036947` | 2025-01-17 | Liu Heng | D | 其它核心 (workspace/window 等) | +10/-2 | fix: The error password prompt window cannot be displayed |
| 824 | `5e6732af514e` | 2025-01-21 | Yixue Wang | D | 构建/打包 | +8/-0 | chore: update version number for 5.27.2.280 |
| 825 | `38450d90d486` | 2025-01-14 | YaoBing Xiao | D | 显示后端/DRM/多屏 | +2/-0 | feat: add output info log |
| 826 | `af03c303b8f6` | 2025-01-22 | Zhang Dingyuan | D | 构建/打包 | +7/-0 | chore: update version number for 5.27.2.281 |
| 827 | `d7193c643a4c` | 2025-02-18 | Lu YaNing | A | 显示后端/DRM/多屏 | +38/-75 | fix: Check GPU active state before handling change events |
| 828 | `4ccaaff927e3` | 2025-02-18 | pengwenhao | D | 其它核心 (workspace/window 等) | +1/-1 | fix: root window _NET_KDE_COMPOSITE_TOGGLING was not set success |
| 829 | `2a7c524fbd20` | 2025-02-14 | YaoBing Xiao | D | 其它核心 (workspace/window 等) | +4/-0 | fix: asan heap-use-after-free |
| 830 | `b62b6efbddb1` | 2025-02-18 | Zhang Dingyuan | D | 其它核心 (workspace/window 等) | +0/-1 | fix: crash when output added and removed |
| 831 | `8664c7b63aa7` | 2025-02-19 | Zhang Dingyuan | D | 构建/打包 | +16/-0 | chore: update version number to 5.27.2.282-deepin282 |
| 832 | `e300f8676aec` | 2025-02-25 | guoyao | D | Wayland 协议/服务端 (含 dde 私有协议) | +30/-8 | fix: use env to skip buffer |
| 833 | `9338c4c73b1e` | 2025-02-25 | guoyao | D | 构建/打包 | +7/-0 | chore: update version number to 5.27.2.283-deepin283 |
| 834 | `e4f24b8ffdf8` | 2025-03-02 | guoyao | D | Wayland 协议/服务端 (含 dde 私有协议) | +8/-7 | fix: windowId should't be used internalId |
| 835 | `df8ad4bcdfb3` | 2025-02-05 | YaoBing Xiao | D | 其它核心 (workspace/window 等) | +22/-0 | fix: asan heap-use-after-free |
| 836 | `caedab3f30a9` | 2025-02-11 | YaoBing Xiao | D | 特效 | +1/-1 | feat: Enable Debugging for GL Framebuffer |
| 837 | `6760f97497cd` | 2025-03-13 | rewine | D | 构建/打包 | +31/-9 | fix: support build on v25 |
| 838 | `f64fe2d113a2` | 2025-03-13 | rewine | D | X11/Xwayland | +10/-0 | fix(v25): avoid no authorization protocol specified in x11 |
| 839 | `bc7bff3baf0c` | 2025-03-13 | rewine | D | 其它核心 (workspace/window 等) | +75/-24 | fix: imageblur and appearance dbus use new api in v25 |
| 840 | `39f55f1405fe` | 2025-03-03 | rewine | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +3217/-1534 | chore: support build with qt6 |
| 841 | `91e541428a1d` | 2025-03-11 | rewine | C | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +11995/-1844 | fix(v25): build vendor kdecoration2 on qt6 |
| 842 | `670f2a37d3f4` | 2025-03-18 | rewine | D | 其它核心 (workspace/window 等) | +17/-3 | fix: minimized animation error in v25 |
| 843 | `5fb4488e8fc9` | 2025-03-18 | rewine | D | 构建/打包 | +1/-1 | fix: support build kscreenlocker on qt6 |
| 844 | `29452fa32a25` | 2025-03-18 | rewine | D | 构建/打包 | +29/-1 | fix: rename plugins file name to mate id |
| 845 | `1a28c601cc3d` | 2025-03-18 | rewine | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +2/-2 | fix: can't get windowRadius for dtk |
| 846 | `23571a243f7b` | 2025-03-19 | rewine | D | 分屏 splitscreen | +10/-14 | fix: adjust dbus imageblur to imageeffec adapter dde dbus |
| 847 | `57e48d350773` | 2025-03-19 | rewine | A | 特效 | +7/-1 | fix: QQuickRenderControl can only sync/render when beginFrame() has been called |
| 848 | `68d527f9ce80` | 2025-03-19 | rewine | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +18/-35 | chore: rename kdecoration to avoid conflict |
| 849 | `81547fc351c4` | 2025-03-19 | rewine | D | 构建/打包 | +320/-278 | chore: make it build on qt5 again |
| 850 | `3fb2ae8df883` | 2025-03-20 | rewine | D | 多任务视图 multitaskview | +116/-1 | fix: can't not find GraphicalEffects in qt6 |
| 851 | `7bc74e347a54` | 2025-03-20 | rewine | D | 多任务视图 multitaskview | +0/-1 | fix: attempted to overwrite a QRandomGenerator to global |
| 852 | `b96dfb9cf13c` | 2025-03-20 | rewine | D | 其它核心 (workspace/window 等) | +43/-21 | fix: optimize dbus monitor |
| 853 | `9f3f9a48c725` | 2025-03-20 | rewine | B | 其它核心 (workspace/window 等) | +131/-2 | fix: outline.qml not work with qt6 |
| 854 | `a76de61b9553` | 2025-03-20 | rewine | D | 构建/打包 | +234/-222 | chore: clear up |
| 855 | `a770dd1f2580` | 2025-03-20 | rewine | D | 输入 | +2/-9 | fix: use same QWheelEvent with kwin 5.27.80 |
| 856 | `bb8942f8a03f` | 2025-03-21 | rewine | D | X11/Xwayland | +7/-0 | fix: AA_DisableHighDpiScaling not work at qt6 |
| 857 | `a2e2fd539292` | 2025-03-21 | rewine | A | 特效 | +7/-3 | chore: do not call RHI QQuickRenderControl methods when using software rendering |
| 858 | `f22b24202e15` | 2025-03-21 | rewine | A | 特效 | +6/-1 | fix: adapt offscreenquickview to Qt6 changes |
| 859 | `c5b634b4daaf` | 2025-03-21 | rewine | D | 多任务视图 multitaskview | +6/-9 | fix: can't show hover text in qt6 |
| 860 | `0f30a314d77b` | 2025-03-25 | rewine | D | 构建/打包 | +7/-2 | fix: can't build translations in qt6 |
| 861 | `b4c6b5fd446a` | 2025-03-25 | rewine | D | 多任务视图 multitaskview | +42/-56 | chore: update dbus call in multitaskview |
| 862 | `1b4c9b11e955` | 2025-03-24 | rewine | D | 构建/打包 | +384/-665 | chore(debian): deb support in qt6 |
| 863 | `1be97c6b5ad5` | 2025-03-26 | rewine | D | 其它核心 (workspace/window 等) | +39/-41 | chore: format code |
| 864 | `c0081fec64cc` | 2025-03-28 | rewine | D | 构建/打包 | +27/-27 | chore: clear up KF version in cmake |
| 865 | `0c964ad07085` | 2025-03-28 | rewine | D | 构建/打包 | +62/-47 | chore: make it build on qt5 again |
| 866 | `2bad6bacebab` | 2025-04-01 | rewine | D | 构建/打包 | +4/-2 | chore: BUILD_ON_V25 should be an option |
| 867 | `618f9c070375` | 2025-04-03 | rewine | D | 构建/打包 | +1/-0 | fix: add kglobalacceld as depends |
| 868 | `5983e66d048a` | 2025-04-03 | rewine | D | 构建/打包 | +6/-0 | chore: update version to 6.0.1 |
| 869 | `f1b81267f6a4` | 2025-04-07 | deepin-admin-bot | D | 其它核心 (workspace/window 等) | +108/-0 | chore: Sync by https://github.com/linuxdeepin/.github/commit/0ff70d7d4d36ce0fae3f3cdb3ba6d7518af204a3 |
| 870 | `3e6cefaacb8d` | 2025-04-03 | rewine | D | 构建/打包 | +3/-2 | fix(debian): error links in debian |
| 871 | `a92dafc50aaa` | 2025-04-08 | rewine | D | X11/Xwayland | +1/-1 | fix: dxcb may not loaded in qt6 |
| 872 | `854c2cb6c56a` | 2025-04-09 | rewine | D | 构建/打包 | +7/-0 | chore(debian): update version to 6.0.2 |
| 873 | `49e5699bbfa4` | 2025-04-09 | rewine | D | 特效 | +6/-3 | fix: should not show radius when fullscreen or splited |
| 874 | `42ed20ffe38d` | 2024-07-04 | rewine | D | 特效 | +4/-0 | fix: schedule repaint after call takeScreenShot |
| 875 | `a78e4bae4c61` | 2025-04-14 | rewine | B | 构建/打包 | +25/-7 | fix: kpackage install namespace in kf6 |
| 876 | `e82293d6245a` | 2025-04-14 | rewine | D | Alt+Tab 切换器 | +19/-2 | chore(tabbox): Simplify loading switcher from kf6 KPackage |
| 877 | `e6e3492dea0f` | 2025-04-14 | rewine | B | Alt+Tab 切换器 | +559/-2 | fix: adapt tabbox qml to qt6 |
| 878 | `1a769758f87c` | 2024-07-02 | lbwtw | D | 其它核心 (workspace/window 等) | +2/-7 | fix: hoverleave dde-dock preview,window show error |
| 879 | `a833ad3b67d4` | 2025-04-16 | rewine | D | 分屏 splitscreen | +15/-8 | fix(splitbar): can't show when use qt6 |
| 880 | `53128083c4e3` | 2025-04-16 | rewine | D | 其它核心 (workspace/window 等) | +15/-5 | fix: com.deepin.wm dbus has changed in v25 |
| 881 | `7e4ecdd4b21b` | 2025-04-16 | rewine | D | 其它核心 (workspace/window 等) | +4/-1 | fix: access illegal memory in createTimespanDBusMessage |
| 882 | `ab6de38573f8` | 2025-04-16 | rewine | D | 构建/打包 | +13/-0 | chore(debian): update version to 6.0.3 |
| 883 | `0169f1e6265d` | 2025-04-17 | rewine | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +263/-842 | chore: reuse compliant |
| 884 | `cecace22cdb3` | 2025-04-18 | rewine | D | 构建/打包 | +1/-1 | fix(debain): kpackage has moved to kf6 |
| 885 | `9bec39ae2845` | 2025-04-18 | rewine | D | 构建/打包 | +7/-0 | fix(debain): update version to 6.0.4 |
| 886 | `7dd0fd631779` | 2025-04-18 | rewine | D | 构建/打包 | +2/-2 | fix: INSTALL_NAMESPACE error for windowswitcher |
| 887 | `3a8f3f71982e` | 2025-04-18 | rewine | D | 构建/打包 | +6/-0 | chore(debian): update version to 6.0.5 |
| 888 | `9e39994fe9d7` | 2025-04-21 | rewine | D | 分屏 splitscreen | +0/-1 | chore(splitscreen): remove unsed PlasmaCore |
| 889 | `38a2923be912` | 2025-04-21 | rewine | D | 其它核心 (workspace/window 等) | +13/-0 | fix: force launcher size to be set to screen size |
| 890 | `7b760da6701e` | 2025-04-23 | rewine | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +11/-2 | fix: add listener for scale change |
| 891 | `feb5f9bf2d22` | 2025-04-23 | rewine | D | 其它核心 (workspace/window 等) | +16/-0 | fix: add judge about mouseInputAreaMargin from dtk win in isResizable |
| 892 | `9f09114621ff` | 2025-04-23 | rewine | D | 构建/打包 | +3/-2 | chore(effects): rename kwin4_effect_magiclamp to magiclamp |
| 893 | `c49ec42db837` | 2025-04-23 | rewine | D | 构建/打包 | +10/-0 | chore(debian): update version to 6.0.6 |
| 894 | `416c04d5dfb3` | 2025-04-24 | rewine | B | 构建/打包 | +93/-93 | chore(effects): rename all effects same with uos 20 |
| 895 | `350883d2c1de` | 2025-05-09 | rewine | D | 其它核心 (workspace/window 等) | +14/-9 | fix: get an empty filteredProcessList from dbus |
| 896 | `c2243c9916e2` | 2025-05-09 | rewine | D | 构建/打包 | +7/-0 | chore(debain): update version to 6.0.7 |
| 897 | `95fe8b2135cb` | 2025-05-16 | rewine | D | 构建/打包 | +16/-11 | chore: optimize CMakeLists.txt |
| 898 | `f560645f87bc` | 2025-05-16 | rewine | D | 其它核心 (workspace/window 等) | +69/-23 | chore: Update README files for Deepin KWin Qt6/KF6 migration |
| 899 | `b5366ef84058` | 2025-03-06 | Zhang Dingyuan | D | 其它核心 (workspace/window 等) | +1/-0 | chore: sync output order to protocol interface |
| 900 | `b6b61ac0a235` | 2025-04-01 | Zhang Dingyuan | D | X11/Xwayland | +1/-1 | fix: correct skipTaskbar() call in window focus handling |
| 901 | `e4ea04b32626` | 2025-03-11 | Yixue Wang | D | 其它核心 (workspace/window 等) | +36/-2 | fix: client not raised without xinput |
| 902 | `7a721b5603fe` | 2025-03-24 | Lu YaNing | A | 测试 | +125/-0 | fix: crash when hotplugging screens by updating window outputs |
| 903 | `4af20ef3d7db` | 2025-04-11 | Zhang Dingyuan | D | 其它核心 (workspace/window 等) | +2/-0 | fix: add repaint call in recordShape to ensure proper rendering |
| 904 | `13419b0cc4c5` | 2025-04-09 | YaoBing Xiao | D | Wayland 协议/服务端 (含 dde 私有协议) | +22/-12 | fix: error next window geometry under qt opengl |
| 905 | `c382e7c362fa` | 2025-04-17 | pengwenhao | D | 其它核心 (workspace/window 等) | +6/-0 | fix: kwin no usage x11 WM_HINTS |
| 906 | `0c8b30986eca` | 2025-04-21 | YaoBing Xiao | D | Wayland 协议/服务端 (含 dde 私有协议) | +2/-1 | chore: hide viewporter dependency from surface_interface public header |
| 907 | `3291806e1f42` | 2025-04-18 | Lu YaNing | A | 测试 | +6/-2 | fix: Window disappears after un-maximizing in mirror mode |
| 908 | `77e6a3d3dea3` | 2024-10-17 | huangzhenyan | D | 显示后端/DRM/多屏 | +357/-219 | refactor: refactor drm blob: mode, gamma, ctm |
| 909 | `302d02da9fd0` | 2024-10-28 | huangzhenyan | D | 显示后端/DRM/多屏 | +214/-13 | feat: add colormode switching with refactoring |
| 910 | `8b1e31e08058` | 2024-11-08 | huangzhenyan | D | 显示后端/DRM/多屏 | +168/-155 | fix: screen frozen after switch session |
| 911 | `c25e984235df` | 2024-11-13 | huangzeming | D | 其它核心 (workspace/window 等) | +3/-10 | fix: network tip on lock screen |
| 912 | `cd20c8ae1ad9` | 2024-11-16 | huangzhenyan | D | 显示后端/DRM/多屏 | +7/-25 | fix: eyeprotection and colormode flick |
| 913 | `6404957cb958` | 2024-11-19 | huangzeming | D | 特效 | +102/-92 | fix: rewrite maximizeex |
| 914 | `dadba30c89ff` | 2024-12-02 | huangzeming | D | Wayland 协议/服务端 (含 dde 私有协议) | +14/-0 | fix: log the information of grab client |
| 915 | `277cc1e3a517` | 2024-12-02 | yangtian | D | 特效 | +2/-0 | fix: add effect log when changing status |
| 916 | `bbdc01988f1d` | 2024-12-02 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +19/-25 | fix: remote_access get invalid fd of a deleted remote_buffer |
| 917 | `c34889becaeb` | 2024-12-03 | dingzhixuan | D | 显示后端/DRM/多屏 | +20/-0 | fix: [drm] turn of outputs when prepare for sleep |
| 918 | `f3fb544f5b7b` | 2024-11-28 | huangzhenyan | D | 显示后端/DRM/多屏 | +280/-0 | fix: eyeprotection effect blink when dpms turns on |
| 919 | `40c7e9ba44c2` | 2024-12-04 | huangzhenyan | D | Wayland 协议/服务端 (含 dde 私有协议) | +75/-35 | fix: avoid deepin-screen-recorder crash |
| 920 | `3a325063c601` | 2024-12-05 | yangtian | D | 分屏 splitscreen | +1/-1 | fix: dont raise the minimized splitscreen window |
| 921 | `e5be7f72a435` | 2024-12-05 | huangzhenyan | D | 显示后端/DRM/多屏 | +1/-1 | fix: do not send rgb(0,0,0) to CTM |
| 922 | `33234e5281aa` | 2024-12-18 | huangzhenyan | D | 显示后端/DRM/多屏 | +2/-2 | fix: enable outputs appropriately |
| 923 | `f87521e2fad5` | 2024-12-23 | YaoBing Xiao | D | 显示后端/DRM/多屏 | +15/-0 | fix: multiple output initialization position are [0, 0] |
| 924 | `09667b0871dc` | 2024-12-27 | YaoBing Xiao | D | 其它核心 (workspace/window 等) | +1/-1 | fix: abnormal maximization window |
| 925 | `81db552bbee1` | 2024-12-30 | YaoBing Xiao | D | X11/Xwayland | +14/-0 | fix: some X11 app in Alt+TAB icon display errors |
| 926 | `d06629348801` | 2025-01-07 | YaoBing Xiao | D | Wayland 协议/服务端 (含 dde 私有协议) | +27/-7 | fix: high cpu usage when adjusting brightness or color temperature |
| 927 | `768e8c56b1bb` | 2025-02-08 | YaoBing Xiao | D | 显示后端/DRM/多屏 | +1/-10 | fix: delete udev timer |
| 928 | `a500d441a1a0` | 2025-02-17 | pengwenhao | D | Wayland 协议/服务端 (含 dde 私有协议) | +11/-10 | fix: retmoe access graphics stress test causes memory leak |
| 929 | `0c5e74293dba` | 2025-02-24 | Lu YaNing | D | 窗口装饰/标题栏 (deepin-chameleon/windowstyle/kdecoration) | +5/-0 | fix: Add null pointer check for atoms in WindowRadius::updateWindowRadius |
| 930 | `b46adb10c339` | 2025-02-26 | Lu YaNing | D | 其它核心 (workspace/window 等) | +19/-9 | fix:  Crashes when rapidly plugging/unplugging external displays |
| 931 | `c5492d91468d` | 2025-02-27 | zorowk | D | Wayland 协议/服务端 (含 dde 私有协议) | +9/-2 | fix: remote access manager release crash |
| 932 | `05bb426bcd0b` | 2025-03-03 | pengwenhao | D | 显示后端/DRM/多屏 | +3/-1 | fix: drm_restorer dead loop crash |
| 933 | `0195c83daa8e` | 2025-03-04 | YaoBing Xiao | D | 显示后端/DRM/多屏 | +4/-0 | fix: screen frozen |
| 934 | `1f2d47a6fc94` | 2025-03-04 | YaoBing Xiao | D | 显示后端/DRM/多屏 | +3/-1 | feat: Add all connector status output |
| 935 | `76c96ce70f32` | 2025-03-11 | pengwenhao | D | Wayland 协议/服务端 (含 dde 私有协议) | +1/-1 | fix: primary selection device resource crash when empty |
| 936 | `1f0a7b3e9a0e` | 2025-03-12 | guoyao | D | 其它核心 (workspace/window 等) | +0/-16 | refactor(main_wayland): not need signal handler |
| 937 | `09da202b2900` | 2025-04-21 | YaoBing Xiao | A | 显示后端/DRM/多屏 | +2/-2 | chroe: increase GPU reset timeout again |
| 938 | `e29d3095801b` | 2025-03-03 | guoyao | D | 其它核心 (workspace/window 等) | +55/-7 | refactor(log): use systemd-journald log by default. |
| 939 | `48e65b71ca63` | 2025-03-28 | guoyao | D | 其它核心 (workspace/window 等) | +111/-36 | refactor(logger): async call dbus method |
| 940 | `407a009554df` | 2025-04-10 | guoyao | D | 其它核心 (workspace/window 等) | +40/-10 | refactor(log): refactor `appendRules` method |
| 941 | `28ea60abfc5c` | 2025-02-21 | Liu Heng | D | Wayland 协议/服务端 (含 dde 私有协议) | +9/-0 | fix:client cannot obtain dataoffer |
| 942 | `aaffb130b561` | 2025-05-06 | rewine | D | 其它核心 (workspace/window 等) | +41/-0 | fix: build on qt 6.8 |
| 943 | `273923218486` | 2025-05-06 | rewine | D | 其它核心 (workspace/window 等) | +4/-2 | fix: disable unapply patch on v25 |
| 944 | `88a6a16295d2` | 2025-05-13 | xiaoyaobing | D | 其它核心 (workspace/window 等) | +0/-6 | Revert "fix: kwin no usage x11 WM_HINTS" |
| 945 | `7a4d17ab1085` | 2025-05-12 | liuheng | D | 其它核心 (workspace/window 等) | +34/-1 | fix: some window get return code -1 when call prohibitscreenshot |
| 946 | `4d5932885607` | 2025-05-16 | guoyao | D | X11/Xwayland | +107/-65 | Refactor(XwaylandLauncher): Move XwaylandLauncher operations to a separate thread |
| 947 | `6c006956e30e` | 2025-05-15 | Yixue Wang | D | 特效 | +4/-18 | fix: tearing in new window effect |
| 948 | `1afb737adeb1` | 2025-05-26 | YaoBing Xiao | D | 显示后端/DRM/多屏 | +3/-3 | chroe: add output mode set log |
| 949 | `925dc91a0363` | 2025-06-04 | rewine | D | 显示后端/DRM/多屏 | +1/-1 | fix: the typo: change "witch" to "which" in the comment. |
| 950 | `33d0ff866366` | 2025-06-04 | rewine | D | 测试 | +2/-2 | chore: the variable name tileOutputPositon  is misspelled |
| 951 | `305ad7b6ebc6` | 2025-06-04 | zccrs | D | 显示后端/DRM/多屏 | +5/-1 | chore: Guard against a null blob pointer before calling blob()->blobId() |
| 952 | `34e024ee3851` | 2025-06-05 | rewine | D | 构建/打包 | +6/-0 | chore(debian): update version to 6.0.8 |
| 953 | `013f8d53e5f1` | 2025-06-24 | rewine | D | 翻译 | +147/-0 | chore: configure transifex translation files |
| 954 | `48eaf3c1f764` | 2025-06-24 | transifex-integration[bot] | D | 翻译 | +38473/-14096 | i18n：Updates for project kwin (#367) |
| 955 | `89db151d57e8` | 2025-06-25 | rewine | D | 翻译 | +1/-4 | docs: fix plural forms header in Icelandic translation (#368) |
| 956 | `14632fab12b0` | 2025-06-26 | transifex-integration[bot] | D | 翻译 | +5956/-2667 | Updates for project deepin kwin (#369) |
| 957 | `6ae41c201362` | 2025-06-26 | rewine | D | 分屏 splitscreen | +73/-2 | chore: update translation file references |
| 958 | `4efd6681a07a` | 2025-06-27 | rewine | D | 翻译 | +31/-1 | chore: update translation file with window control strings |
| 959 | `af560bd312ac` | 2025-07-11 | transifex-integration[bot] | D | 翻译 | +7941/-3627 | Updates for project TreeLand (#371) |
| 960 | `e2123e36b161` | 2025-07-15 | rewine | D | 翻译 | +0/-119 | chore: remove unimportant translations files (#374) |
| 961 | `664808264f5a` | 2025-09-25 | Felix Yan | B | 其它核心 (workspace/window 等) | +11/-1 | fix: build against Qt 6.9+ |
