# 基于原版 KWin 6 重建 DKWin：in-place merge 与 patch 方案评估

- 评估对象：deepin-kwin `664808264f5a49c306834565d0e0c631a8feda27`，即 KWin 5.27.2 快照加上 961 个改动提交，详见 [deepin-kwin-vs-kwin-report.md](./deepin-kwin-vs-kwin-report.md)
- 目标基线：KDE 官方 KWin（GitHub 镜像 `KDE/kwin`）
  - `Plasma/6.8`，commit `7a8d309b7a7cf503bf77481c389579ee3568bce9`。最新稳定 tag 是 v6.8.2，**从 6.8 起只支持 Wayland**
  - `Plasma/6.7`，commit `c1f623739f4e819d291b1ddda50d5a57674a54f5`。这是最后一个还有 X11 会话的系列，KDE 已在 `KDE/kwin-x11` 仓库里声明 X11 部分不再维护，最后维护版本就是 Plasma/6.7
- 评估日期：2026-11-05

---

## 一、结论

**建议走 patch 路线**：以原版 KWin 6 为底，把 deepin 的功能拆成两部分。一部分做成独立的插件仓库，另一部分整理成一组尽量少的核心补丁，每次 KDE 发版时把补丁 rebase 到新 tag 上。

**不建议**把 KWin 6 直接合并进现有的 deepin-kwin 仓库（in-place merge）。主要原因有六条：

1. 两边没有共同的 git 历史，git 无法正常做三方合并。
2. 试合并会产生 **479 处非翻译冲突**，其中 307 个文件里共有 **950 段冲突代码**。
3. 合并没有冲突的部分也编译不过，因为 KWin 6 的插件接口变了。
4. 有不少 deepin 改动在 KWin 6 上已经没用了，merge 时却还得先给它们解冲突。
5. merge 完成后，deepin 的改动和 KDE 的历史混在一起，以后每次升级还得再做一次大合并。
6. KWin 6.8 删除 X11 会话的那批提交，本身就会和 deepin 的 X11 改动大面积冲突。

---

## 二、in-place merge 的试合并数据

### 2.1 两边没有共同历史

```text
git merge-base 6648082 kde/master   ->  （空）
```

deepin-kwin 是在 `fb932a1` 里把 KWin 5.27.2 的源码一次性导入的，没有带上 KDE 的提交历史。所以直接 `git merge` 会被当成两段不相关的历史，几乎每个文件都会冲突。要让三方合并能跑起来，只能人为把合并基准指定为 `v5.27.2`（用 `git replace --graft` 或 `git merge-tree --merge-base`）。下面的数据就是这样得到的。

### 2.2 KDE 那边的变化量

| 从 v5.27.2 到 | 非 merge 提交数 | 文件改动 |
|---|---:|---|
| Plasma/6.7 | 7,187 | 4,251 个文件，+681,926 / −624,753 |
| Plasma/6.8 | 7,823 | 4,306 个文件，+746,157 / −712,730 |

KWin 6 里影响最大的几项结构性变化：
- `src/effects/*` 整体搬到了 `src/plugins/*`
- libkwineffects 合并进了 libkwin，`src/libkwineffects` 改名为 `src/effect`
- 窗口装饰接口从 KDecoration2 升级到 KDecoration3
- 只支持 Qt6 / KF6
- 6.8 删除了 X11 独立会话和 `backends/x11/standalone`

### 2.3 试合并结果（合并基准指定为 v5.27.2，不含翻译文件）

| 冲突类型 | 合并到 6.7 | 合并到 6.8 | 含义 |
|---|---:|---:|---|
| 内容冲突（content） | 309 | 303 | 双方改了同一个文件 |
| 双方都新增了同名文件（add/add） | 4 | 4 | |
| deepin 改过、KDE 已删除 | 148 | 145 | KDE 删除了文件，或者改动太大、识别不出是重命名。这类冲突没法按文本合并，只能重写 |
| deepin 删除、KDE 还在改 | （含在上行） | 11 | |
| deepin 删除、KDE 移动了位置 | 13 | 13 | 例如 windowview 特效 |
| 文件位置冲突（file location）/ 目录重命名分裂 | 4 | 3 | 例如 `src/libkwineffects/kwineffectsex.*` 和 `backends/x11/standalone` |
| **合计** | **478** | **479** | 算上翻译文件，分别是 853 和 862 |
| 带冲突标记的文件 / 冲突代码段数 | 313 / 931 | 307 / **950** | |

**6.8 冲突最多的文件**（冲突代码段数）：`src/window.cpp` 41，`src/workspace.cpp` 35，`src/x11window.cpp` 29，`src/xdgshellwindow.cpp` 27，`src/window.h` 19，`src/input.cpp` 17，`src/useractions.cpp` 16，`src/CMakeLists.txt` 16，`src/wayland_server.cpp` 14，`src/wayland/seat.cpp` 14，`src/tabbox/tabbox.cpp` 14，`src/rules.cpp` 14。

当前代码中，冲突文件里的 deepin 代码合计约 **9,400 行**。被 KDE 删除的那 145 个文件里，deepin 代码合计约 **3,000 行**。逐个文件的明细见附录 A。

### 2.4 合并通过不等于能编译

deepin 约 70% 的代码（约 3.3 万行）在它自己新增的文件里，比如多任务视图、分屏、chameleon 标题栏、dde 私有 Wayland 协议。这些文件 merge 时**不会冲突**，但 KWin 6 的特效接口、场景接口、装饰接口和 Wayland 服务端接口都变了，它们照样编译不过。也就是说，这 3.3 万行**不管选哪条路都要按 KWin 6 的接口重新适配**。merge 并不能省掉这部分工作，只是把问题从解冲突推迟到了编译阶段。

### 2.5 要为马上就会删掉的代码解冲突

在 KWin 6 上，下面这些 deepin 改动大概率可以直接丢弃（逐个提交列在附录 B）：

| 类别 | 提交数 | 说明 |
|---|---:|---|
| 从 KDE 搬来的修复（审计报告 A 类） | 16 | KWin 6 里本来就有 |
| 为 Qt 5.11 / UOS V20 降级做的适配 | 31 | 集中在 2023-05，KWin 6 只支持 Qt6 |
| Qt6 / KF6 / V25 构建适配 | 26 | 和 KDE 自己的 Qt6 移植重叠，不含已列在其他类别的 `f22b242`、`91e5414`。其中 `53128083`、`bc7bff3` 等是 V25 的 DBus 接口调整，需要逐个确认 |
| 整体引入的 kdecoration 5.27.5 | 1 | `91e5414`，KWin 6 用系统里的 KDecoration3 |
| XRender 合成后端 | 1 | `a449132`。KWin 6 的软件渲染用 QPainter |

另外两项不一定能丢，需要评估：
- **xdg-shell-unstable-v6**（`8c31004`）：要看还有没有老应用依赖它。
- **outputdevice_v1 / outputmanagement_v1**（`9f1b334`）：KWin 6 已有 v2，要看 dde 客户端能不能改用 v2。

---

## 三、patch 方案

### 3.1 KWin 6 可以从外部扩展的地方

以下都已在 `Plasma/6.8` 源码中核实：

- **安装的头文件**：KWin 6 会把大量内部头文件安装到 `include/kwin`，包括：
  - 核心：`workspace.h`、`window.h`、`input.h`、`input_event_spy.h`、`plugin.h`、`pluginmanager.h`、`wayland_server.h`、`useractions.h`、`screenedge.h`、`rules.h` 等
  - 子目录：`effect/*`、`scene/*`、`opengl/*`、`core/*`、`utils/*`
  - Wayland 服务端：`wayland/*` 下 83 个头文件
- **CMake 导出**：导出 `KWin::kwin` 目标，供外部项目链接。
- **插件加载**：`PluginManager` 会从 Qt 插件目录下的 `kwin/plugins` 加载外部插件；C++ 特效从 `kwin/effects/plugins` 加载；QML 和脚本特效从 KPackage 的 `kwin/effects` 加载。
- **窗口圆角**：场景图原生支持圆角（`Item::setBorderRadius`，见 `scene/borderradius.h`），可以作为 deepin `windowstyle` 和 `scissorwindow` 的替代基础。
- **标准 Wayland 协议**：KWin 6 已经内置 layer-shell、plasma-window-management、plasma-shell、kde-output-device-v2 / output-management-v2、zkde-screencast、security-context-v1、EIS（输入捕获）等。部分 dde 私有协议也许能换成这些标准协议，**需要逐个评估**。

### 3.2 建议的组织方式

1. **底座用 KDE 真正的历史。** 直接 clone `KDE/kwin`，从某个 tag 拉出 `deepin/6.8` 分支（如果还要 X11，另拉一个基于 `Plasma/6.7` 的分支）。不要再一次性导入源码快照。
2. **独立插件仓库，不改 KWin 源码。** 新增文件里的代码约占 deepin 改动的 70%，按下表搬迁，各自对 KWin 6 的接口做一次适配：

   | deepin 模块 | 在 KWin 6 中的形态 |
   |---|---|
   | multitaskview、splitscreen、splitswap、maximizeex、scissorwindow、alttabthumbnaillist 等特效 | C++ 或 QML 特效插件 |
   | deepin-chameleon 标题栏 | KDecoration3 装饰插件 |
   | Alt+Tab 切换器 | KPackage QML 切换器 |
   | ddeshell / ddeseat / ddekeyboard / ddekvm / ddesecurity / dderestrict / clientmanagement / strut / remote_access | KWin 插件（基于 `kwin/plugins` 和安装出来的 `wayland/*.h`），或者换成标准协议 |
   | windowstyle（圆角、阴影） | 尽量改用 `Item::setBorderRadius` 和标准阴影 |

3. **核心补丁队列。** 直接改 KWin 核心文件的那约 1.4 万行（集中在 `workspace.cpp`、`window.cpp`、`input.cpp`、`xdgshellwindow.cpp`、`effects.cpp`、`wayland_server.cpp`），不逐个照搬原提交，而是按功能重新整理成少量补丁，主要是给插件留的钩子和 deepin 专有行为。
   - 每个补丁只做一件事，并标明上游状态：已提交 KDE / 不适合上游 / deepin 专有。
   - 通用的钩子优先提交给 KDE。以前已经有 deepin 员工的补丁被 KDE 接受过，例如 `1747518f81`、`2f4005cd67`、`5b5045a402`。
4. **升级流程**：KDE 每发一个版本，就执行一次 `git rebase --onto <新 tag> <旧 tag> deepin/6.x`，同时让插件仓库适配新接口。打 deb 包时可以把补丁导出成 `debian/patches`。
5. **验收**：用一份功能清单逐项对照，确认 deepin 的功能在新版上都能正常使用。清单来自审计报告的 D 类提交，按领域分组。

---

## 四、什么时候 in-place merge 更合适

只有当以下两个前提**同时**成立时：
- 只做这一次升级，以后不再持续跟进 KDE；
- 非常看重保留现有 deepin-kwin 的提交历史和 git blame。

既然目标是"用新的原版 KWin 6 重新实现 DKWin"，并且要长期跟着上游走，这两个前提都不成立。

---

## 五、复现方法

```bash
git remote add kde https://github.com/KDE/kwin
git fetch --no-tags kde '+refs/heads/Plasma/6.8:refs/remotes/kde/Plasma/6.8' '+refs/tags/v5.27.2:refs/tags/kde-v5.27.2'
git merge-base 6648082 kde/Plasma/6.8                      # 为空：没有共同历史
git -c merge.renameLimit=5000 merge-tree --write-tree --name-only \
    --merge-base=kde-v5.27.2 6648082 kde/Plasma/6.8 > merge.txt  # 试合并，不改动工作区
grep '^CONFLICT' merge.txt | grep -v '/po/\| po/' | sed -E 's/^CONFLICT \(([^)]*)\).*/\1/' | sort | uniq -c
git grep -c '^<<<<<<< ' "$(head -1 merge.txt)" -- . ':!po' ':!**/po/**'   # 每个文件的冲突代码段数
```

---

## 附录 A：试合并到 Plasma/6.8 的全部非翻译冲突（479 项）

"deepin 行数"是当前 HEAD 中，`git blame` 归到 deepin 改动（审计报告 B、D 类）名下的行数。

| # | 文件 | 冲突类型 | 冲突代码段数 | deepin 行数 |
|---:|---|---|---:|---:|
| 1 | `src/window.cpp` | content | 41 | 795 |
| 2 | `src/workspace.cpp` | content | 35 | 1065 |
| 3 | `src/x11window.cpp` | content | 29 | 204 |
| 4 | `src/xdgshellwindow.cpp` | content | 27 | 427 |
| 5 | `src/window.h` | content | 19 | 212 |
| 6 | `src/input.cpp` | content | 17 | 373 |
| 7 | `src/useractions.cpp` | content | 16 | 250 |
| 8 | `src/CMakeLists.txt` | content | 16 | 140 |
| 9 | `src/wayland_server.cpp` | content | 14 | 404 |
| 10 | `src/tabbox/tabbox.cpp` | content | 14 | 149 |
| 11 | `src/rules.cpp` | content | 14 | 40 |
| 12 | `src/wayland/seat.cpp` | content | 14 | 0 |
| 13 | `src/workspace.h` | content | 13 | 277 |
| 14 | `CMakeLists.txt` | content | 13 | 210 |
| 15 | `src/backends/drm/drm_pipeline.cpp` | content | 12 | 110 |
| 16 | `src/tabbox/tabboxhandler.cpp` | content | 11 | 85 |
| 17 | `src/backends/drm/drm_backend.cpp` | content | 11 | 50 |
| 18 | `src/backends/libinput/connection.cpp` | content | 10 | 211 |
| 19 | `src/pointer_input.cpp` | content | 9 | 113 |
| 20 | `src/wayland_server.h` | content | 9 | 72 |
| 21 | `autotests/CMakeLists.txt` | content | 9 | 57 |
| 22 | `src/events.cpp` | content | 8 | 134 |
| 23 | `src/xwayland/xwaylandlauncher.cpp` | content | 8 | 118 |
| 24 | `src/shadow.cpp` | content | 8 | 75 |
| 25 | `src/backends/drm/drm_egl_layer_surface.cpp` | content | 8 | 61 |
| 26 | `autotests/wayland/server/CMakeLists.txt` | content | 8 | 0 |
| 27 | `src/main_wayland.cpp` | content | 7 | 85 |
| 28 | `src/xkb.cpp` | content | 7 | 36 |
| 29 | `src/rules.h` | content | 7 | 21 |
| 30 | `src/options.h` | content | 7 | 14 |
| 31 | `src/backends/drm/drm_connector.cpp` | content | 6 | 61 |
| 32 | `src/backends/drm/drm_output.cpp` | content | 6 | 61 |
| 33 | `src/plugins/screencast/screencaststream.cpp` | content | 6 | 22 |
| 34 | `autotests/libinput/CMakeLists.txt` | content | 6 | 8 |
| 35 | `src/core/output.cpp` | content | 5 | 224 |
| 36 | `src/backends/drm/drm_egl_backend.cpp` | content | 5 | 67 |
| 37 | `src/scene/workspacescene.cpp` | content | 5 | 65 |
| 38 | `src/xdgshellwindow.h` | content | 5 | 48 |
| 39 | `src/backends/drm/drm_gpu.cpp` | content | 5 | 44 |
| 40 | `src/options.cpp` | content | 5 | 40 |
| 41 | `src/backends/drm/drm_connector.h` | content | 5 | 21 |
| 42 | `src/shadow.h` | content | 5 | 21 |
| 43 | `src/xwayland/xwayland.cpp` | content | 5 | 21 |
| 44 | `src/backends/drm/drm_pipeline.h` | content | 5 | 19 |
| 45 | `src/tabbox/tabbox.h` | content | 5 | 16 |
| 46 | `src/pointer_input.h` | content | 5 | 13 |
| 47 | `src/backends/drm/drm_backend.h` | content | 5 | 5 |
| 48 | `src/backends/drm/drm_gpu.h` | content | 5 | 5 |
| 49 | `src/core/output.h` | content | 4 | 90 |
| 50 | `src/backends/drm/drm_crtc.h` | content | 4 | 45 |
| 51 | `src/scene/windowitem.cpp` | content | 4 | 40 |
| 52 | `src/scripting/scripting.cpp` | content | 4 | 38 |
| 53 | `src/keyboard_input.cpp` | content | 4 | 24 |
| 54 | `src/plugins/krunner-integration/windowsrunnerinterface.cpp` | content | 4 | 24 |
| 55 | `src/netinfo.cpp` | content | 4 | 18 |
| 56 | `src/plugins/krunner-integration/dbusutils_p.h` | content | 4 | 17 |
| 57 | `src/scene/itemrenderer_opengl.cpp` | content | 4 | 14 |
| 58 | `autotests/integration/CMakeLists.txt` | content | 4 | 13 |
| 59 | `src/main.cpp` | content | 4 | 13 |
| 60 | `src/core/renderloop.cpp` | content | 4 | 12 |
| 61 | `autotests/drm/CMakeLists.txt` | content | 4 | 9 |
| 62 | `src/backends/drm/drm_output.h` | content | 4 | 7 |
| 63 | `src/placementtracker.cpp` | content | 4 | 4 |
| 64 | `src/wayland/seat_p.h` | content | 4 | 0 |
| 65 | `src/wayland/CMakeLists.txt` | content | 3 | 66 |
| 66 | `src/plugins/windowsystem/windowsystem.cpp` | content | 3 | 65 |
| 67 | `src/backends/fakeinput/fakeinputdevice.cpp` | content | 3 | 58 |
| 68 | `src/plugins/screencast/screencastutils.h` | content | 3 | 46 |
| 69 | `src/touch_input.cpp` | content | 3 | 43 |
| 70 | `src/internalwindow.cpp` | content | 3 | 42 |
| 71 | `src/plugins/windowsystem/windoweffects.cpp` | content | 3 | 40 |
| 72 | `src/waylandwindow.cpp` | content | 3 | 36 |
| 73 | `src/tabbox/switcheritem.h` | content | 3 | 30 |
| 74 | `src/plugins/kpackage/effect/effect.cpp` | content | 3 | 29 |
| 75 | `README.md` | content | 3 | 21 |
| 76 | `src/utils/xcbutils.h` | content | 3 | 21 |
| 77 | `src/plugins/kpackage/aurorae/aurorae.cpp` | content | 3 | 20 |
| 78 | `src/plugins/windowsystem/windowsystem.h` | content | 3 | 18 |
| 79 | `src/x11window.h` | content | 3 | 18 |
| 80 | `src/xwayland/xwaylandlauncher.h` | content | 3 | 18 |
| 81 | `src/core/outputbackend.cpp` | content | 3 | 14 |
| 82 | `src/kcms/decoration/decorationmodel.cpp` | content | 3 | 12 |
| 83 | `src/placement.cpp` | content | 3 | 11 |
| 84 | `src/xwayland/clipboard.cpp` | content | 3 | 9 |
| 85 | `src/wayland/tools/CMakeLists.txt` | content | 3 | 8 |
| 86 | `src/scene/decorationitem.cpp` | content | 3 | 5 |
| 87 | `src/core/outputbackend.h` | content | 3 | 4 |
| 88 | `src/core/rendertarget.h` | content | 3 | 3 |
| 89 | `src/plugins/qpa/backingstore.cpp` | content | 3 | 1 |
| 90 | `autotests/integration/transient_placement.cpp` | content | 3 | 0 |
| 91 | `src/plugins/magiclamp/magiclamp_config.h` | content | 3 | 0 |
| 92 | `src/plugins/screenshot/screenshotdbusinterface2.cpp` | content | 3 | 0 |
| 93 | `src/plugins/squash/package/contents/code/main.js` | content | 3 | 0 |
| 94 | `src/wayland/drmlease_v1.cpp` | content | 3 | 0 |
| 95 | `src/wayland/plasmawindowmanagement.cpp` | content | 3 | 0 |
| 96 | `src/wayland/seat.h` | content | 3 | 0 |
| 97 | `src/tabbox/switcheritem.cpp` | content | 2 | 132 |
| 98 | `src/utils/common.h` | content | 2 | 104 |
| 99 | `src/backends/drm/drm_crtc.cpp` | content | 2 | 97 |
| 100 | `src/dbusinterface.cpp` | content | 2 | 86 |
| 101 | `src/rulesettings.kcfg` | content | 2 | 44 |
| 102 | `src/layers.cpp` | content | 2 | 39 |
| 103 | `src/core/outputconfiguration.cpp` | content | 2 | 30 |
| 104 | `src/plugins/kpackage/scripts/scripts.cpp` | content | 2 | 26 |
| 105 | `src/plugins/windowsystem/windoweffects.h` | content | 2 | 26 |
| 106 | `src/tiles/quicktile.cpp` | content | 2 | 24 |
| 107 | `src/core/outputconfiguration.h` | content | 2 | 23 |
| 108 | `src/tiles/tile.cpp` | content | 2 | 23 |
| 109 | `src/activation.cpp` | content | 2 | 20 |
| 110 | `src/plugins/idletime/poller.h` | content | 2 | 17 |
| 111 | `src/plugins/screencast/screencastmanager.cpp` | content | 2 | 16 |
| 112 | `src/kcms/decoration/kwin-applywindowdecoration.cpp` | content | 2 | 14 |
| 113 | `src/utils/udev.cpp` | content | 2 | 14 |
| 114 | `tests/CMakeLists.txt` | content | 2 | 13 |
| 115 | `src/backends/libinput/events.cpp` | content | 2 | 12 |
| 116 | `src/helpers/wayland_wrapper/kwin_wrapper.cpp` | content | 2 | 12 |
| 117 | `src/plugins/kpackage/decoration/decoration.cpp` | content | 2 | 12 |
| 118 | `src/plugins/kpackage/windowswitcher/windowswitcher.cpp` | content | 2 | 12 |
| 119 | `src/backends/drm/drm_egl_backend.h` | content | 2 | 10 |
| 120 | `src/plugins/qpa/integration.h` | content | 2 | 10 |
| 121 | `src/tabbox/tabboxhandler.h` | content | 2 | 9 |
| 122 | `src/kcms/decoration/CMakeLists.txt` | content | 2 | 8 |
| 123 | `src/backends/drm/drm_plane.cpp` | content | 2 | 7 |
| 124 | `src/config-kwin.h.cmake` | content | 2 | 7 |
| 125 | `src/waylandwindow.h` | content | 2 | 7 |
| 126 | `src/backends/wayland/wayland_egl_backend.cpp` | content | 2 | 6 |
| 127 | `src/core/session.h` | content | 2 | 6 |
| 128 | `src/backends/drm/CMakeLists.txt` | content | 2 | 5 |
| 129 | `src/main_wayland.h` | content | 2 | 5 |
| 130 | `autotests/integration/quick_tiling_test.cpp` | content | 2 | 2 |
| 131 | `src/backends/wayland/wayland_backend.cpp` | content | 2 | 2 |
| 132 | `src/xwayland/primary.cpp` | content | 2 | 2 |
| 133 | `autotests/effect/CMakeLists.txt` | content | 2 | 0 |
| 134 | `src/decorations/decorationbridge.cpp` | content | 2 | 0 |
| 135 | `src/effect/effectloader.cpp` | content | 2 | 0 |
| 136 | `src/opengl/glplatform.cpp` | content | 2 | 0 |
| 137 | `src/plugins/blur/CMakeLists.txt` | content | 2 | 0 |
| 138 | `src/plugins/diminactive/CMakeLists.txt` | content | 2 | 0 |
| 139 | `src/plugins/glide/CMakeLists.txt` | content | 2 | 0 |
| 140 | `src/plugins/magiclamp/CMakeLists.txt` | content | 2 | 0 |
| 141 | `src/plugins/maximize/package/contents/code/main.js` | content | 2 | 0 |
| 142 | `src/plugins/mouseclick/CMakeLists.txt` | content | 2 | 0 |
| 143 | `src/plugins/mousemark/CMakeLists.txt` | content | 2 | 0 |
| 144 | `src/plugins/slide/CMakeLists.txt` | content | 2 | 0 |
| 145 | `src/plugins/slidingpopups/CMakeLists.txt` | content | 2 | 0 |
| 146 | `src/plugins/thumbnailaside/CMakeLists.txt` | content | 2 | 0 |
| 147 | `src/plugins/trackmouse/CMakeLists.txt` | content | 2 | 0 |
| 148 | `src/plugins/wobblywindows/CMakeLists.txt` | content | 2 | 0 |
| 149 | `src/popup_input_filter.cpp` | content | 2 | 0 |
| 150 | `src/wayland/pointer.cpp` | content | 2 | 0 |
| 151 | `src/scripting/scriptedeffect.cpp` | content | 1 | 97 |
| 152 | `src/plugins/qpa/integration.cpp` | content | 1 | 46 |
| 153 | `src/xwayland/selection.cpp` | content | 1 | 44 |
| 154 | `src/useractions.h` | content | 1 | 35 |
| 155 | `src/virtualdesktops.cpp` | content | 1 | 35 |
| 156 | `src/outline.cpp` | content | 1 | 32 |
| 157 | `src/input.h` | content | 1 | 22 |
| 158 | `src/backends/libinput/connection.h` | content | 1 | 21 |
| 159 | `src/core/rendertarget.cpp` | content | 1 | 19 |
| 160 | `src/kcms/common/CMakeLists.txt` | content | 1 | 19 |
| 161 | `src/scene/surfaceitem_wayland.cpp` | content | 1 | 14 |
| 162 | `src/kcms/screenedges/CMakeLists.txt` | content | 1 | 13 |
| 163 | `src/atoms.cpp` | content | 1 | 12 |
| 164 | `src/kcms/tabbox/CMakeLists.txt` | content | 1 | 12 |
| 165 | `src/scripting/scripting.h` | content | 1 | 12 |
| 166 | `src/xwayland/databridge.cpp` | content | 1 | 12 |
| 167 | `src/plugins/krunner-integration/CMakeLists.txt` | content | 1 | 11 |
| 168 | `src/xkb.h` | content | 1 | 11 |
| 169 | `src/xwayland/selection.h` | content | 1 | 11 |
| 170 | `src/atoms.h` | content | 1 | 10 |
| 171 | `src/backends/wayland/CMakeLists.txt` | content | 1 | 10 |
| 172 | `src/touch_input.h` | content | 1 | 10 |
| 173 | `src/xwayland/datasource.h` | content | 1 | 10 |
| 174 | `autotests/libinput/pointer_event_test.cpp` | content | 1 | 9 |
| 175 | `src/input_event.cpp` | content | 1 | 8 |
| 176 | `src/kcms/rules/CMakeLists.txt` | content | 1 | 8 |
| 177 | `src/kcms/scripts/CMakeLists.txt` | content | 1 | 8 |
| 178 | `src/plugins/qpa/CMakeLists.txt` | content | 1 | 8 |
| 179 | `src/scene/scene.h` | content | 1 | 8 |
| 180 | `src/scene/surfaceitem.h` | content | 1 | 8 |
| 181 | `src/backends/wayland/wayland_display.cpp` | content | 1 | 7 |
| 182 | `src/kcms/effects/CMakeLists.txt` | content | 1 | 7 |
| 183 | `src/tabbox/clientmodel.cpp` | content | 1 | 7 |
| 184 | `src/helpers/killer/CMakeLists.txt` | content | 1 | 6 |
| 185 | `src/kcms/decoration/declarative-plugin/CMakeLists.txt` | content | 1 | 6 |
| 186 | `src/kcms/options/CMakeLists.txt` | content | 1 | 6 |
| 187 | `src/outline.h` | content | 1 | 6 |
| 188 | `src/plugins/buttonrebinds/CMakeLists.txt` | content | 1 | 6 |
| 189 | `src/scene/workspacescene.h` | content | 1 | 6 |
| 190 | `src/xdgactivationv1.cpp` | content | 1 | 6 |
| 191 | `autotests/drm/mock_drm.h` | content | 1 | 5 |
| 192 | `src/core/inputdevice.cpp` | content | 1 | 5 |
| 193 | `src/globalshortcuts.cpp` | content | 1 | 5 |
| 194 | `src/internalwindow.h` | content | 1 | 5 |
| 195 | `src/kcms/desktop/CMakeLists.txt` | content | 1 | 5 |
| 196 | `src/kwin.kcfg` | content | 1 | 5 |
| 197 | `src/netinfo.h` | content | 1 | 5 |
| 198 | `src/org.kde.kwin.Compositing.xml` | content | 1 | 5 |
| 199 | `src/plugins/buttonrebinds/buttonrebindsfilter.cpp` | content | 1 | 5 |
| 200 | `src/plugins/idletime/CMakeLists.txt` | content | 1 | 5 |
| 201 | `src/plugins/windowsystem/plugin.h` | content | 1 | 5 |
| 202 | `src/plugins/windowsystem/windowshadow.h` | content | 1 | 5 |
| 203 | `src/qml/frames/plasma/frame_none.qml` | content | 1 | 5 |
| 204 | `src/qml/frames/plasma/frame_unstyled.qml` | content | 1 | 5 |
| 205 | `src/scripting/windowthumbnailitem.cpp` | content | 1 | 5 |
| 206 | `src/tabbox/clientmodel.h` | content | 1 | 5 |
| 207 | `src/tiles/quicktile.h` | content | 1 | 5 |
| 208 | `src/wayland/linuxdmabufv1clientbuffer.cpp` | content | 1 | 5 |
| 209 | `src/backends/drm/drm_object.cpp` | content | 1 | 4 |
| 210 | `src/debug_console.cpp` | content | 1 | 4 |
| 211 | `src/kcms/desktop/virtualdesktops.cpp` | content | 1 | 4 |
| 212 | `src/kcms/tabbox/layoutpreview.cpp` | content | 1 | 4 |
| 213 | `src/kcms/virtualkeyboard/kcmvirtualkeyboard.cpp` | content | 1 | 4 |
| 214 | `src/kcms/xwayland/kcmkwinxwayland.cpp` | content | 1 | 4 |
| 215 | `src/plugins/idletime/poller.cpp` | content | 1 | 4 |
| 216 | `src/plugins/krunner-integration/windowsrunnerinterface.h` | content | 1 | 4 |
| 217 | `src/plugins/qpa/window.cpp` | content | 1 | 4 |
| 218 | `src/plugins/screencast/CMakeLists.txt` | content | 1 | 4 |
| 219 | `src/plugins/screencast/screencaststream.h` | content | 1 | 4 |
| 220 | `src/plugins/windowsystem/CMakeLists.txt` | content | 1 | 4 |
| 221 | `src/scene/surfaceitem.cpp` | content | 1 | 4 |
| 222 | `autotests/integration/helper/CMakeLists.txt` | content | 1 | 3 |
| 223 | `data/CMakeLists.txt` | content | 1 | 3 |
| 224 | `src/kcms/virtualkeyboard/CMakeLists.txt` | content | 1 | 3 |
| 225 | `src/kcms/xwayland/CMakeLists.txt` | content | 1 | 3 |
| 226 | `src/main.h` | content | 1 | 3 |
| 227 | `src/qml/frames/plasma/frame_styled.qml` | content | 1 | 3 |
| 228 | `src/backends/drm/drm_buffer.cpp` | content | 1 | 2 |
| 229 | `src/backends/drm/drm_object.h` | content | 1 | 2 |
| 230 | `src/backends/drm/drm_virtual_output.cpp` | content | 1 | 2 |
| 231 | `src/backends/fakeinput/fakeinputdevice.h` | content | 1 | 2 |
| 232 | `src/core/inputdevice.h` | content | 1 | 2 |
| 233 | `src/kcms/screenedges/screenpreviewwidget.h` | content | 1 | 2 |
| 234 | `src/plugins/qpa/window.h` | content | 1 | 2 |
| 235 | `src/tiles/tile.h` | content | 1 | 2 |
| 236 | `src/wayland/display_p.h` | content | 1 | 2 |
| 237 | `src/wayland/shmclientbuffer.cpp` | content | 1 | 2 |
| 238 | `src/wayland/textinput.h` | content | 1 | 2 |
| 239 | `src/xdgactivationv1.h` | content | 1 | 2 |
| 240 | `.gitignore` | content | 1 | 1 |
| 241 | `autotests/integration/kwin_wayland_test.h` | content | 1 | 1 |
| 242 | `autotests/integration/test_helpers.cpp` | content | 1 | 1 |
| 243 | `src/backends/drm/drm_buffer.h` | content | 1 | 1 |
| 244 | `src/backends/drm/drm_egl_layer.cpp` | content | 1 | 1 |
| 245 | `src/backends/drm/drm_egl_layer_surface.h` | content | 1 | 1 |
| 246 | `src/backends/drm/drm_pipeline_legacy.cpp` | content | 1 | 1 |
| 247 | `src/backends/wayland/wayland_backend.h` | content | 1 | 1 |
| 248 | `src/client_machine.cpp` | content | 1 | 1 |
| 249 | `src/core/renderbackend.h` | content | 1 | 1 |
| 250 | `src/core/renderloop_p.h` | content | 1 | 1 |
| 251 | `src/core/session_logind.h` | content | 1 | 1 |
| 252 | `src/helpers/wayland_wrapper/CMakeLists.txt` | content | 1 | 1 |
| 253 | `src/helpers/wayland_wrapper/wl-socket.c` | content | 1 | 1 |
| 254 | `src/inputmethod.cpp` | content | 1 | 1 |
| 255 | `src/inputpanelv1window.cpp` | content | 1 | 1 |
| 256 | `src/layershellv1window.cpp` | content | 1 | 1 |
| 257 | `src/placementtracker.h` | content | 1 | 1 |
| 258 | `src/plugins/buttonrebinds/buttonrebindsfilter.h` | content | 1 | 1 |
| 259 | `src/rulebooksettings.cpp` | content | 1 | 1 |
| 260 | `src/scene/decorationitem.h` | content | 1 | 1 |
| 261 | `src/scene/item.h` | content | 1 | 1 |
| 262 | `src/scene/surfaceitem_wayland.h` | content | 1 | 1 |
| 263 | `src/screenedge.h` | content | 1 | 1 |
| 264 | `src/sm.h` | content | 1 | 1 |
| 265 | `src/utils/CMakeLists.txt` | content | 1 | 1 |
| 266 | `src/wayland/tools/qtwaylandscanner.cpp` | content | 1 | 1 |
| 267 | `src/xwayland/lib/CMakeLists.txt` | content | 1 | 1 |
| 268 | `autotests/integration/fakes/org.kde.kdecoration3/CMakeLists.txt` | content | 1 | 0 |
| 269 | `src/backends/x11/x11_windowed_backend.cpp` | content | 1 | 0 |
| 270 | `src/backends/x11/x11_windowed_backend.h` | content | 1 | 0 |
| 271 | `src/effect/offscreeneffect.cpp` | content | 1 | 0 |
| 272 | `src/kcms/decoration/utils.h` | content | 1 | 0 |
| 273 | `src/plugins/blur/blur_config.cpp` | content | 1 | 0 |
| 274 | `src/plugins/blur/blur_config.h` | content | 1 | 0 |
| 275 | `src/plugins/blur/shaders/downsample.frag` | content | 1 | 0 |
| 276 | `src/plugins/diminactive/diminactive_config.cpp` | content | 1 | 0 |
| 277 | `src/plugins/diminactive/diminactive_config.h` | content | 1 | 0 |
| 278 | `src/plugins/fadingpopups/package/contents/code/main.js` | content | 1 | 0 |
| 279 | `src/plugins/fallapart/CMakeLists.txt` | content | 1 | 0 |
| 280 | `src/plugins/highlightwindow/highlightwindow.cpp` | content | 1 | 0 |
| 281 | `src/plugins/highlightwindow/highlightwindow.h` | content | 1 | 0 |
| 282 | `src/plugins/kscreen/CMakeLists.txt` | content | 1 | 0 |
| 283 | `src/plugins/magiclamp/magiclamp_config.cpp` | content | 1 | 0 |
| 284 | `src/plugins/outputlocator/outputlocator.h` | content | 1 | 0 |
| 285 | `src/plugins/overview/kcm/CMakeLists.txt` | content | 1 | 0 |
| 286 | `src/plugins/qpa/backingstore.h` | content | 1 | 0 |
| 287 | `src/plugins/scale/package/contents/code/main.js` | content | 1 | 0 |
| 288 | `src/plugins/screencast/outputscreencastsource.cpp` | content | 1 | 0 |
| 289 | `src/plugins/screencast/outputscreencastsource.h` | content | 1 | 0 |
| 290 | `src/plugins/screencast/regionscreencastsource.cpp` | content | 1 | 0 |
| 291 | `src/plugins/screencast/regionscreencastsource.h` | content | 1 | 0 |
| 292 | `src/plugins/screencast/screencastsource.h` | content | 1 | 0 |
| 293 | `src/plugins/screencast/windowscreencastsource.cpp` | content | 1 | 0 |
| 294 | `src/plugins/screencast/windowscreencastsource.h` | content | 1 | 0 |
| 295 | `src/plugins/sheet/CMakeLists.txt` | content | 1 | 0 |
| 296 | `src/plugins/slide/slide_config.cpp` | content | 1 | 0 |
| 297 | `src/plugins/slide/slide_config.h` | content | 1 | 0 |
| 298 | `src/plugins/tileseditor/kcm/CMakeLists.txt` | content | 1 | 0 |
| 299 | `src/wayland/datadevice.h` | content | 1 | 0 |
| 300 | `src/wayland/outputdevice_v2.h` | content | 1 | 0 |
| 301 | `src/wayland/pointer.h` | content | 1 | 0 |
| 302 | `src/wayland/primaryselectiondevice_v1.h` | content | 1 | 0 |
| 303 | `src/wayland/subcompositor.cpp` | content | 1 | 0 |
| 304 | `autotests/integration/workspace_test.cpp` | add/add | 8 | 0 |
| 305 | `src/backends/drm/drm_blob.cpp` | add/add | 3 | 44 |
| 306 | `src/backends/drm/drm_blob.h` | add/add | 2 | 63 |
| 307 | `src/resources.qrc` | add/add | 1 | 8 |
| 308 | `src/backends/x11/standalone` | directory rename split |  | 0 |
| 309 | `src/libkwineffects/kwineffectsex.h` | file location |  | 112 |
| 310 | `src/libkwineffects/kwineffectsex.cpp` | file location |  | 36 |
| 311 | `src/effects.cpp` | modify/delete（KDE 已删除） |  | 599 |
| 312 | `src/composite.cpp` | modify/delete（KDE 已删除） |  | 426 |
| 313 | `src/effects/screenshot/screenshotdbusinterface1.cpp` | modify/delete（KDE 已删除） |  | 217 |
| 314 | `src/effects.h` | modify/delete（KDE 已删除） |  | 116 |
| 315 | `src/effects/screenshot/screenshot.cpp` | modify/delete（KDE 已删除） |  | 113 |
| 316 | `src/libkwineffects/kwineffects.cpp` | modify/delete（KDE 已删除） |  | 94 |
| 317 | `src/libkwineffects/kwineffects.h` | modify/delete（KDE 已删除） |  | 74 |
| 318 | `src/backends/x11/common/kwinxrenderutils.cpp` | modify/delete（KDE 已删除） |  | 72 |
| 319 | `src/effects/eyeonscreen/package/contents/code/main.js` | modify/delete（KDE 已删除） |  | 63 |
| 320 | `src/xwayland/selection_source.cpp` | modify/delete（KDE 已删除） |  | 54 |
| 321 | `src/composite.h` | modify/delete（KDE 已删除） |  | 47 |
| 322 | `src/unmanaged.cpp` | modify/delete（KDE 已删除） |  | 46 |
| 323 | `src/platformsupport/scenes/opengl/abstract_egl_backend.cpp` | modify/delete（KDE 已删除） |  | 40 |
| 324 | `src/wayland/outputmanagement_v2_interface.cpp` | modify/delete（KDE 已删除） |  | 40 |
| 325 | `src/backends/drm/drm_buffer_gbm.cpp` | modify/delete（KDE 已删除） |  | 35 |
| 326 | `src/main_x11.cpp` | modify/delete（KDE 已删除） |  | 29 |
| 327 | `src/platformsupport/scenes/opengl/egl_dmabuf.cpp` | modify/delete（KDE 已删除） |  | 29 |
| 328 | `src/wayland/autotests/client/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 29 |
| 329 | `src/effects/blur/blur.cpp` | modify/delete（KDE 已删除） |  | 26 |
| 330 | `src/libkwineffects/kwingltexture.cpp` | modify/delete（KDE 已删除） |  | 26 |
| 331 | `src/deleted.h` | modify/delete（KDE 已删除） |  | 25 |
| 332 | `autotests/tabbox/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 24 |
| 333 | `src/effects/screenedge/screenedgeeffect.cpp` | modify/delete（KDE 已删除） |  | 23 |
| 334 | `src/wayland/fakeinput_interface.cpp` | modify/delete（KDE 已删除） |  | 23 |
| 335 | `src/effects/screenshot/screenshotdbusinterface1.h` | modify/delete（KDE 已删除） |  | 22 |
| 336 | `src/effects/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 21 |
| 337 | `src/backends/drm/gbm_dmabuf.h` | modify/delete（KDE 已删除） |  | 20 |
| 338 | `src/effects/screenshot/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 20 |
| 339 | `src/plugins/colord-integration/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 20 |
| 340 | `src/backends/x11/common/kwinxrenderutils.h` | modify/delete（KDE 已删除） |  | 19 |
| 341 | `src/backends/x11/standalone/x11_standalone_backend.cpp` | modify/delete（KDE 已删除） |  | 17 |
| 342 | `src/plugins/kglobalaccel/kglobalaccel_plugin.cpp` | modify/delete（KDE 已删除） |  | 16 |
| 343 | `src/backends/x11/standalone/x11_standalone_glx_backend.cpp` | modify/delete（KDE 已删除） |  | 15 |
| 344 | `src/core/colorlut.cpp` | modify/delete（KDE 已删除） |  | 15 |
| 345 | `src/libkwineffects/kwinoffscreenquickview.cpp` | modify/delete（KDE 已删除） |  | 15 |
| 346 | `src/unmanaged.h` | modify/delete（KDE 已删除） |  | 15 |
| 347 | `src/effects/screenshot/screenshot.h` | modify/delete（KDE 已删除） |  | 14 |
| 348 | `src/kcms/compositing/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 14 |
| 349 | `src/libkwineffects/kwinglutils.cpp` | modify/delete（KDE 已删除） |  | 14 |
| 350 | `src/wayland/surface_interface.cpp` | modify/delete（KDE 已删除） |  | 14 |
| 351 | `src/wayland/tests/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 14 |
| 352 | `src/cursordelegate_opengl.cpp` | modify/delete（KDE 已删除） |  | 13 |
| 353 | `src/effects/zoom/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 13 |
| 354 | `src/platformsupport/scenes/opengl/openglbackend.h` | modify/delete（KDE 已删除） |  | 13 |
| 355 | `src/plugins/kglobalaccel/kglobalaccel_plugin.h` | modify/delete（KDE 已删除） |  | 13 |
| 356 | `src/tabbox/x11_filter.cpp` | modify/delete（KDE 已删除） |  | 13 |
| 357 | `src/effects/desktopgrid/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 12 |
| 358 | `src/libkwineffects/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 12 |
| 359 | `src/scene/workspacescene_opengl.cpp` | modify/delete（KDE 已删除） |  | 12 |
| 360 | `src/wayland/clientbuffer.cpp` | modify/delete（KDE 已删除） |  | 12 |
| 361 | `src/effects/magnifier/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 11 |
| 362 | `src/effects/screenedge/screenedgeeffect.h` | modify/delete（KDE 已删除） |  | 11 |
| 363 | `src/libkwineffects/kwinglobals.h` | modify/delete（KDE 已删除） |  | 11 |
| 364 | `src/modifier_only_shortcuts.cpp` | modify/delete（KDE 已删除） |  | 11 |
| 365 | `src/plugins/nightcolor/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 11 |
| 366 | `src/effects/invert/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 10 |
| 367 | `src/effects/showpaint/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 10 |
| 368 | `src/scripting/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 10 |
| 369 | `src/deleted.cpp` | modify/delete（KDE 已删除） |  | 9 |
| 370 | `src/effects/screenedge/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 9 |
| 371 | `src/platformsupport/scenes/opengl/basiceglsurfacetexture_wayland.cpp` | modify/delete（KDE 已删除） |  | 9 |
| 372 | `src/wayland/tests/fakeoutput.cpp` | modify/delete（KDE 已删除） |  | 9 |
| 373 | `src/wayland/datadevice_interface.cpp` | modify/delete（KDE 已删除） |  | 8 |
| 374 | `src/xwayland/selection_source.h` | modify/delete（KDE 已删除） |  | 8 |
| 375 | `src/plugins/kpackage/aurorae/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 7 |
| 376 | `src/plugins/kpackage/decoration/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 7 |
| 377 | `src/plugins/kpackage/effect/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 7 |
| 378 | `src/plugins/kpackage/scripts/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 7 |
| 379 | `src/plugins/kpackage/windowswitcher/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 7 |
| 380 | `src/utils/serviceutils.h` | modify/delete（KDE 已删除） |  | 7 |
| 381 | `src/wayland/outputdevice_v2_interface.cpp` | modify/delete（KDE 已删除） |  | 7 |
| 382 | `src/backends/drm/drm_dmabuf_feedback.cpp` | modify/delete（KDE 已删除） |  | 6 |
| 383 | `src/backends/x11/standalone/x11_standalone_effects_mouse_interception_filter.cpp` | modify/delete（KDE 已删除） |  | 6 |
| 384 | `src/effects/overview/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 6 |
| 385 | `src/effects/tileseditor/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 6 |
| 386 | `src/libkwineffects/kwinanimationeffect.cpp` | modify/delete（KDE 已删除） |  | 6 |
| 387 | `src/libkwineffects/kwinquickeffect.cpp` | modify/delete（KDE 已删除） |  | 6 |
| 388 | `src/plugins/kglobalaccel/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 6 |
| 389 | `src/backends/x11/standalone/x11_standalone_xinputintegration.cpp` | modify/delete（KDE 已删除） |  | 5 |
| 390 | `src/core/colorlut.h` | modify/delete（KDE 已删除） |  | 5 |
| 391 | `src/effects/outputlocator/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 5 |
| 392 | `src/utils/xcursortheme.cpp` | modify/delete（KDE 已删除） |  | 5 |
| 393 | `src/wayland/surface_interface.h` | modify/delete（KDE 已删除） |  | 5 |
| 394 | `src/backends/x11/standalone/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 4 |
| 395 | `src/effects/colorpicker/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 4 |
| 396 | `src/effects/private/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 4 |
| 397 | `src/effects/showfps/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 4 |
| 398 | `src/effects/startupfeedback/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 4 |
| 399 | `src/kcms/decoration/declarative-plugin/plugin.cpp` | modify/delete（KDE 已删除） |  | 4 |
| 400 | `src/tabbox/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 4 |
| 401 | `src/wayland/fakeinput_interface.h` | modify/delete（KDE 已删除） |  | 4 |
| 402 | `autotests/tabbox/mock_tabboxclient.h` | modify/delete（KDE 已删除） |  | 3 |
| 403 | `src/effects/blendchanges/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 3 |
| 404 | `src/effects/highlightwindow/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 3 |
| 405 | `src/effects/snaphelper/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 3 |
| 406 | `src/effects/touchpoints/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 3 |
| 407 | `src/libkwineffects/KWinEffectsConfig.cmake.in` | modify/delete（KDE 已删除） |  | 3 |
| 408 | `src/libkwineffects/kwinglutils.h` | modify/delete（KDE 已删除） |  | 3 |
| 409 | `src/plugins/kdecorations/aurorae/src/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 3 |
| 410 | `src/plugins/kdecorations/aurorae/src/aurorae.cpp` | modify/delete（KDE 已删除） |  | 3 |
| 411 | `src/scene/surfaceitem_x11.cpp` | modify/delete（KDE 已删除） |  | 3 |
| 412 | `src/scene/workspacescene_qpainter.cpp` | modify/delete（KDE 已删除） |  | 3 |
| 413 | `src/wayland/autotests/client/test_drag_drop.cpp` | modify/delete（KDE 已删除） |  | 3 |
| 414 | `autotests/integration/helper/copy.cpp` | modify/delete（KDE 已删除） |  | 2 |
| 415 | `autotests/integration/helper/paste.cpp` | modify/delete（KDE 已删除） |  | 2 |
| 416 | `autotests/libinput/input_event_test.cpp` | modify/delete（KDE 已删除） |  | 2 |
| 417 | `autotests/tabbox/mock_tabboxhandler.h` | modify/delete（KDE 已删除） |  | 2 |
| 418 | `src/backends/x11/windowed/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 2 |
| 419 | `src/core/renderlayer.h` | modify/delete（KDE 已删除） |  | 2 |
| 420 | `src/effects/backgroundcontrast/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 2 |
| 421 | `src/effects/screentransform/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 2 |
| 422 | `src/effects/slideback/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 2 |
| 423 | `src/platformsupport/scenes/opengl/egl_dmabuf.h` | modify/delete（KDE 已删除） |  | 2 |
| 424 | `src/plugins/kdecorations/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 2 |
| 425 | `src/plugins/kdecorations/aurorae/src/lib/auroraetheme.h` | modify/delete（KDE 已删除） |  | 2 |
| 426 | `src/plugins/kpackage/decoration/decoration.h` | modify/delete（KDE 已删除） |  | 2 |
| 427 | `src/plugins/kpackage/windowswitcher/windowswitcher.h` | modify/delete（KDE 已删除） |  | 2 |
| 428 | `src/scene/itemrenderer_qpainter.h` | modify/delete（KDE 已删除） |  | 2 |
| 429 | `src/backends/drm/drm_buffer_gbm.h` | modify/delete（KDE 已删除） |  | 1 |
| 430 | `src/backends/drm/drm_dumb_swapchain.cpp` | modify/delete（KDE 已删除） |  | 1 |
| 431 | `src/backends/x11/standalone/x11_standalone_backend.h` | modify/delete（KDE 已删除） |  | 1 |
| 432 | `src/backends/x11/standalone/x11_standalone_glx_backend.h` | modify/delete（KDE 已删除） |  | 1 |
| 433 | `src/effects/blur/blur.h` | modify/delete（KDE 已删除） |  | 1 |
| 434 | `src/effects/fullscreen/package/metadata.json` | modify/delete（KDE 已删除） |  | 1 |
| 435 | `src/effects/mousemark/mousemark.cpp` | modify/delete（KDE 已删除） |  | 1 |
| 436 | `src/effects/private/expolayout.h` | modify/delete（KDE 已删除） |  | 1 |
| 437 | `src/libkwineffects/kwinconfig.h.cmake` | modify/delete（KDE 已删除） |  | 1 |
| 438 | `src/platformsupport/scenes/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 1 |
| 439 | `src/scene/workspacescene_opengl.h` | modify/delete（KDE 已删除） |  | 1 |
| 440 | `src/scene/workspacescene_qpainter.h` | modify/delete（KDE 已删除） |  | 1 |
| 441 | `src/scripts/minimizeall/contents/code/main.js` | modify/delete（KDE 已删除） |  | 1 |
| 442 | `src/wayland/clientbuffer.h` | modify/delete（KDE 已删除） |  | 1 |
| 443 | `src/wayland/clientbuffer_p.h` | modify/delete（KDE 已删除） |  | 1 |
| 444 | `src/wayland/surface_interface_p.h` | modify/delete（KDE 已删除） |  | 1 |
| 445 | `src/effects/blur/blur.qrc` | modify/delete（KDE 已删除） |  | 0 |
| 446 | `src/effects/blur/shaders/downsample_core.frag` | modify/delete（KDE 已删除） |  | 0 |
| 447 | `src/effects/blur/shaders/noise.frag` | modify/delete（KDE 已删除） |  | 0 |
| 448 | `src/effects/blur/shaders/noise_core.frag` | modify/delete（KDE 已删除） |  | 0 |
| 449 | `src/effects/blur/shaders/upsample.frag` | modify/delete（KDE 已删除） |  | 0 |
| 450 | `src/effects/blur/shaders/vertex.vert` | modify/delete（KDE 已删除） |  | 0 |
| 451 | `src/effects/blur/shaders/vertex_core.vert` | modify/delete（KDE 已删除） |  | 0 |
| 452 | `src/platformsupport/scenes/opengl/openglbackend.cpp` | modify/delete（KDE 已删除） |  | 0 |
| 453 | `src/plugins/kdecorations/aurorae/src/aurorae.h` | modify/delete（KDE 已删除） |  | 0 |
| 454 | `src/plugins/kdecorations/aurorae/themes/plastik/code/CMakeLists.txt` | modify/delete（KDE 已删除） |  | 0 |
| 455 | `src/scene/surfaceitem_x11.h` | modify/delete（KDE 已删除） |  | 0 |
| 456 | `src/plugins/windowview/CMakeLists.txt` | modify/delete（deepin 已删除） |  | 0 |
| 457 | `src/plugins/windowview/Main.qml` | modify/delete（deepin 已删除） |  | 0 |
| 458 | `src/plugins/windowview/kcm/CMakeLists.txt` | modify/delete（deepin 已删除） |  | 0 |
| 459 | `src/plugins/windowview/kcm/windowvieweffectkcm.cpp` | modify/delete（deepin 已删除） |  | 0 |
| 460 | `src/plugins/windowview/kcm/windowvieweffectkcm.h` | modify/delete（deepin 已删除） |  | 0 |
| 461 | `src/plugins/windowview/kcm/windowvieweffectkcm.ui` | modify/delete（deepin 已删除） |  | 0 |
| 462 | `src/plugins/windowview/metadata.json` | modify/delete（deepin 已删除） |  | 0 |
| 463 | `src/plugins/windowview/windowviewconfig.kcfg` | modify/delete（deepin 已删除） |  | 0 |
| 464 | `src/plugins/windowview/windowviewconfig.kcfgc` | modify/delete（deepin 已删除） |  | 0 |
| 465 | `src/plugins/windowview/windowvieweffect.cpp` | modify/delete（deepin 已删除） |  | 0 |
| 466 | `src/plugins/windowview/windowvieweffect.h` | modify/delete（deepin 已删除） |  | 0 |
| 467 | `src/effects/windowview/CMakeLists.txt` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 468 | `src/effects/windowview/kcm/CMakeLists.txt` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 469 | `src/effects/windowview/kcm/windowvieweffectkcm.cpp` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 470 | `src/effects/windowview/kcm/windowvieweffectkcm.h` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 471 | `src/effects/windowview/kcm/windowvieweffectkcm.ui` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 472 | `src/effects/windowview/main.cpp` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 473 | `src/effects/windowview/metadata.json` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 474 | `src/effects/windowview/org.kde.KWin.Effect.WindowView1.xml` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 475 | `src/effects/windowview/qml/main.qml` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 476 | `src/effects/windowview/windowviewconfig.kcfg` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 477 | `src/effects/windowview/windowviewconfig.kcfgc` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 478 | `src/effects/windowview/windowvieweffect.cpp` | rename/delete（deepin 删除，KDE 移动） |  | 0 |
| 479 | `src/effects/windowview/windowvieweffect.h` | rename/delete（deepin 删除，KDE 移动） |  | 0 |

## 附录 B：在 KWin 6 上大概率可以丢弃的 deepin 提交

### B.1 从 KDE 搬来的修复（KWin 6 已包含）（16 个）

| # | commit | 日期 | 作者 | 标题 |
|---:|---|---|---|---|
| 1 | `b724380c842903f43cdf86d6ef2553153a461056` | 2023-09-15 | huangzeming | fix: Simplify backing store buffer handling |
| 2 | `d338f610cdc3630424a6a8199b67573493f49ef1` | 2023-11-21 | liuzheng | fix: screencasting: on memfd, skip the QImage step |
| 3 | `39b05d97d15577a9499fffce5235b19d5e559559` | 2023-11-24 | huangzeming | plugins/blur: Rewrite it |
| 4 | `3765ef1b5e81a7eb00de29021adafd3aab40b24a` | 2024-05-07 | zhangyaning | fix: update keyboard focus when grabbing popup closed. |
| 5 | `2bd16957fecd00aacae12c771d0136829947e7ac` | 2023-03-21 | Vlad Zahorodnii | feat: Refactor compositing teardown in X11Window and Unmanaged |
| 6 | `49ce09fee4f2b657b036fb301e3d44e2d4c67fe7` | 2024-05-23 | dingzhixuan | fix: xdg popup position incorrect |
| 7 | `8da3eae7cf70a92763cd3b549bc989158d6a4a67` | 2024-07-16 | yangtian | fix: stop setting XCURSOR_SIZE |
| 8 | `fc18056d5e1d86f0d07c3da98ebe57618f0909eb` | 2024-07-10 | dingzhixuan | fix: glmark2 will change size when plugin and plugout screen |
| 9 | `8b10d5c2afe4e916f69c47f7159c4d805ef4f2ec` | 2024-08-29 | dingzhixuan | fix: Improve Workspace::outputAt() |
| 10 | `d7193c643a4cba4edefeb78045c28af7ab3263a8` | 2025-02-18 | Lu YaNing | fix: Check GPU active state before handling change events |
| 11 | `57e48d350773b4edd47a4fd5243d043cc9f97fa4` | 2025-03-19 | rewine | fix: QQuickRenderControl can only sync/render when beginFrame() has been called |
| 12 | `a2e2fd539292255483cacc9e9b3b84f19fc35254` | 2025-03-21 | rewine | chore: do not call RHI QQuickRenderControl methods when using software rendering |
| 13 | `f22b24202e159a3f526402224fd2823f333471b2` | 2025-03-21 | rewine | fix: adapt offscreenquickview to Qt6 changes |
| 14 | `7a721b5603feb4646accbd1d24437d4213940445` | 2025-03-24 | Lu YaNing | fix: crash when hotplugging screens by updating window outputs |
| 15 | `3291806e1f4260c0ed6ae2268b7266df362c2a8f` | 2025-04-18 | Lu YaNing | fix: Window disappears after un-maximizing in mirror mode |
| 16 | `09da202b2900630a06b9f2efbe4de2ba1953d503` | 2025-04-21 | YaoBing Xiao | chroe: increase GPU reset timeout again |

### B.2 Qt 5.11 / UOS V20 降级适配（31 个）

| # | commit | 日期 | 作者 | 标题 |
|---:|---|---|---|---|
| 1 | `033587a4b790b424f790f15065e876f0d5a08f64` | 2023-05-04 | Chaojiang Luo | feat: download build tool deps |
| 2 | `1130419cb14089696adc7426dc1b50d5520d478a` | 2023-05-04 | Chaojiang Luo | feat: degrade Qt build deps |
| 3 | `f91264aaf8138f5d0ea4caea1a417a4044d87cd2` | 2023-05-04 | Chaojiang Luo | feat: degrade kde frameworks deps |
| 4 | `6f882023f6851584dd7592563980281d1b57a8dd` | 2023-05-04 | Chaojiang Luo | feat: degrade pipewire build deps |
| 5 | `114dad053323b1760d195a05924ebd0ca7191580` | 2023-05-05 | Chaojiang Luo | feat: replace libxcb-util-dev with libxcb-util0-dev |
| 6 | `fb5d941cccdbd7d11ee224c9f651ccdf22cd4608` | 2023-05-05 | Chaojiang Luo | feat: Add cmake module support |
| 7 | `234c88c13a1dfd7961e0fe0245c3000bebaaaa88` | 2023-05-05 | Chaojiang Luo | feat: Adapt to qt 5.11 interface and linked lib |
| 8 | `46d6a0395420b77d37bc6f74304e9bc482075b5c` | 2023-05-05 | Chaojiang Luo | feat: link X11 without X11 workspace |
| 9 | `b31a2f49d12cb6a9a239ccbea7f52f8a6cc66af8` | 2023-05-05 | Chaojiang Luo | feat: replace Qt::endl with \n |
| 10 | `29a1e7a9a4a7a3edff3495470fde737533319c62` | 2023-05-05 | Chaojiang Luo | feat: replace QRecursiveMutex with QMutex |
| 11 | `be02ee926ef40989b489f51ba474fc3527dabaf3` | 2023-05-05 | Chaojiang Luo | feat: Add missing head file memory |
| 12 | `1c4d10db3d9cc3db002c0b46281d09ad342d43e9` | 2023-05-05 | Chaojiang Luo | feat: adapt cxx std::span |
| 13 | `43f440e1bb6f4aacd401b4be5c616982655953ec` | 2023-05-05 | Chaojiang Luo | feat: replace Qt::SkipEmptyParts with QString::SkipEmptyParts |
| 14 | `e9a866f297156fcee360669561d055697b847424` | 2023-05-05 | Chaojiang Luo | feat: adapt QStringView |
| 15 | `be48e66bccd5429276efc183dcab2139e6775875` | 2023-05-05 | Chaojiang Luo | feat: replace qmlRegisterAnonymousType with qmlRegisterType |
| 16 | `eedbdbae13cc34e45252e95716435ee6bd2dce66` | 2023-05-05 | Chaojiang Luo | feat: replace convertTo with convertToFormat |
| 17 | `95e1e6bc04810281c44267cdb3e4d8af0ede33ff` | 2023-05-05 | Chaojiang Luo | feat: adapt QList<QByteArray> |
| 18 | `6fbebfda44d2ea0ba1c8915b96c7e3de3edeb7d8` | 2023-05-05 | Chaojiang Luo | feat: replace widget->screen with QGuiApplication::primaryScreen |
| 19 | `afac85bc4a5f32798d9763800104f94c064175f6` | 2023-05-05 | Chaojiang Luo | feat: redefine LEDs with typedef QFlags |
| 20 | `1b03f292f0fe2bedfac98e2c666f6c6d4f568fb3` | 2023-05-05 | Chaojiang Luo | feat: adapt QWheelEvent |
| 21 | `6b8c56d5861d55454b9222188190865eaba36795` | 2023-05-05 | Chaojiang Luo | feat: add missing head file |
| 22 | `629850c7fb883a51a3e9f2ddd7e729dbbb2a3753` | 2023-05-05 | Chaojiang Luo | feat: replace Q_NAMESPACE_EXPORT with Q_NAMESPACE |
| 23 | `211f58c0a3f44d403290b4c6529f374cf2d956df` | 2023-05-05 | Chaojiang Luo | feat: add default constructor |
| 24 | `560fc04705c54c8dfa6508f4465fd517fe3a7fbd` | 2023-05-05 | Chaojiang Luo | feat: fix cbegin and cend not found |
| 25 | `05d8aa214f9e55255b8d3de26fb5bfa2780dce7e` | 2023-05-05 | Chaojiang Luo | feat: replace loadRelaxed with load |
| 26 | `6cac9d823f6f3e2f2355332f11f4301e2859690e` | 2023-05-05 | Chaojiang Luo | feat: fix QHash init issue |
| 27 | `9d21b6e7ef49998da47e035aab61f093a9132067` | 2023-05-05 | Chaojiang Luo | feat: fix Q_DISABLE_COPY_MOVE not found |
| 28 | `75938e790af43155e544ad3201bc2e69c93a332e` | 2023-05-05 | Chaojiang Luo | feat: adapt AbstractSystemPoller_iid |
| 29 | `4a0ea22ba742f5fd80e0004f986ec8e6750d43cd` | 2023-05-06 | Chaojiang Luo | feat:replace createTextureFromNativeObject with createTextureFromId |
| 30 | `935c17b2215b56219918841a7e3c2ffc2580aa31` | 2023-05-06 | Chaojiang Luo | feat: adapt qpa QWindowSystemInterface interfaces |
| 31 | `b19e655cf1b451f08e01b694117975cd6e123b1e` | 2023-05-06 | Chaojiang Luo | feat: adapt screencast |

> B.3 中 `53128083`（com.deepin.wm DBus 变更）、`bc7bff3`（imageblur / appearance DBus 新接口）、`f64fe2d`（X11 授权）、`670f2a37`（最小化动画）、`a833ad3b`（分屏条显示）涉及 V25 的行为调整，迁移时需要保留它们的意图，只是代码不必照搬。

### B.3 Qt6 / KF6 / V25 构建适配（26 个）

| # | commit | 日期 | 作者 | 标题 |
|---:|---|---|---|---|
| 1 | `6760f97497cd5d28ce025a57a968b885ae4026bb` | 2025-03-13 | rewine | fix: support build on v25 |
| 2 | `f64fe2d113a29a0337b5680f3d29de504362ab52` | 2025-03-13 | rewine | fix(v25): avoid no authorization protocol specified in x11 |
| 3 | `bc7bff3baf0c05229d1b82ff796786f187798bef` | 2025-03-13 | rewine | fix: imageblur and appearance dbus use new api in v25 |
| 4 | `39f55f1405fedc2a144d3902eb9dd106cc49e1c8` | 2025-03-03 | rewine | chore: support build with qt6 |
| 5 | `670f2a37d3f42549bbff9b5a7ddb4727a2762665` | 2025-03-18 | rewine | fix: minimized animation error in v25 |
| 6 | `5fb4488e8fc9294124dbe30846d2cbbde69de1c3` | 2025-03-18 | rewine | fix: support build kscreenlocker on qt6 |
| 7 | `81547fc351c4a0a7e4e3294806e1a6ab62e85bce` | 2025-03-19 | rewine | chore: make it build on qt5 again |
| 8 | `3fb2ae8df88322314f4cd29a1b29b5f86d229c8f` | 2025-03-20 | rewine | fix: can't not find GraphicalEffects in qt6 |
| 9 | `9f3f9a48c7257da29ee8729e45666c5e5fc1c62d` | 2025-03-20 | rewine | fix: outline.qml not work with qt6 |
| 10 | `bb8942f8a03f01b558dffc1b34228a86b18cefe7` | 2025-03-21 | rewine | fix: AA_DisableHighDpiScaling not work at qt6 |
| 11 | `c5b634b4daaf71ee3b874a28322d67c6b4ede216` | 2025-03-21 | rewine | fix: can't show hover text in qt6 |
| 12 | `0f30a314d77b0b2dc883505e8dc1c7a6d3118f8e` | 2025-03-25 | rewine | fix: can't build translations in qt6 |
| 13 | `1b4c9b11e955e4de4968fda7389a4578506e8da9` | 2025-03-24 | rewine | chore(debian): deb support in qt6 |
| 14 | `0c964ad07085a351fcffa903c46f545efb674666` | 2025-03-28 | rewine | chore: make it build on qt5 again |
| 15 | `2bad6bacebab2702423d8e450c2ad36ff95e7dfe` | 2025-04-01 | rewine | chore: BUILD_ON_V25 should be an option |
| 16 | `a92dafc50aaaefe76c31263d158304733cc047c0` | 2025-04-08 | rewine | fix: dxcb may not loaded in qt6 |
| 17 | `a78e4bae4c61533618e38afde907743314510eea` | 2025-04-14 | rewine | fix: kpackage install namespace in kf6 |
| 18 | `e82293d6245af3041d5748fd4897fb78c5c090ae` | 2025-04-14 | rewine | chore(tabbox): Simplify loading switcher from kf6 KPackage |
| 19 | `e6e3492dea0fee2c47f563873bd76cbdc0918da1` | 2025-04-14 | rewine | fix: adapt tabbox qml to qt6 |
| 20 | `a833ad3b67d42f4502134fcc6e091fc936105a1e` | 2025-04-16 | rewine | fix(splitbar): can't show when use qt6 |
| 21 | `53128083c4e3435854bb4bd351af715a12b49912` | 2025-04-16 | rewine | fix: com.deepin.wm dbus has changed in v25 |
| 22 | `cecace22cdb33ea162cce7f5fa750c13d3042aac` | 2025-04-18 | rewine | fix(debain): kpackage has moved to kf6 |
| 23 | `f560645f87bcb625611264ba90911328478dc819` | 2025-05-16 | rewine | chore: Update README files for Deepin KWin Qt6/KF6 migration |
| 24 | `aaffb130b561402ba2b2f920ae3c89c9cc88dd1c` | 2025-05-06 | rewine | fix: build on qt 6.8 |
| 25 | `273923218486b31a3cae264308c6c2d63f996ef7` | 2025-05-06 | rewine | fix: disable unapply patch on v25 |
| 26 | `664808264f5a49c306834565d0e0c631a8feda27` | 2025-09-25 | Felix Yan | fix: build against Qt 6.9+ |

### B.4 KWin 6 中已有替代方案的模块（2 个）

| # | commit | 日期 | 作者 | 标题 |
|---:|---|---|---|---|
| 1 | `91e541428a1de0744b9cd5a1d0627052e4d5b871` | 2025-03-11 | rewine | fix(v25): build vendor kdecoration2 on qt6 |
| 2 | `a44913259444357ec8a0713841c5fc032a1e6f30` | 2023-11-08 | xinbo wang | feat: add XRender backend support. |

