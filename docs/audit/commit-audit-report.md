# deepin-kwin 提交来源审计报告

- 审计对象：`CharOfString/deepin-kwin`，分支 `master` / `claude/upbeat-lovelace-eahnx0`（两者 HEAD 相同）
- HEAD：`664808264f5a49c306834565d0e0c631a8feda27`（fix: build against Qt 6.9+）
- 对照基准：官方仓库 `https://github.com/linuxdeepin/deepin-kwin`，已拉取其全部分支
- 提交总数：**963**（无 merge commit，线性历史；根提交 `aa0a579` 'Init commit'，2023-08-21）
- 审计日期：2026-11-05
- 全量逐条分类清单：[`commit-audit-full.csv`](./commit-audit-full.csv)（963 行，每行一个完整 commit id）

> 说明：原 clone 是浅克隆（只有 50 个提交），审计前已执行 `git fetch --unshallow`，取回全部历史后再分析。

---

## 一、结论速览

| 维度 | 数量 | 说明 |
|---|---:|---|
| **本 fork 独有、官方仓库没有的提交** | **0** | `git rev-list HEAD ^upstream/master` = 0，反向也为 0；fork 的 master 与官方 master 指向**同一个 commit** |
| 被篡改 / 改写过的官方提交 | 0 | commit hash 完全一致，内容不可能不同 |
| 作者不是 deepin/UOS 员工的提交 | 2 | `6648082`（Arch Linux 维护者）、`2bd1695`（KDE 开发者） |
| 机器人 / 自动化提交 | 5 | 3 个 Transifex 翻译、1 个 deepin-admin-bot、1 个 gerritadmin 空提交 |
| 作者用个人邮箱（但属 deepin 开发者） | 3 | `305ad7b`（zccrs）、`17a3bbc` / `228890d`（justforlxz） |
| AI 协作（Co-authored-by: Copilot） | 3 | `305ad7b`、`33d0ff8`、`925dc91` |
| 引用 / 移植 KDE 上游代码 | 7 | 含整体导入 KWin 5.27.2 的 `fb932a1` |
| 从**未公开的内部分支** cherry-pick | 50 | 来源 commit 不存在于官方 GitHub 任何分支 |
| 作者日期早于仓库创建日（导入旧历史） | 72 | 2020-10 ～ 2023-08 期间写的补丁，重新提交进来 |
| 走 UnionTech 内部 Gerrit 流程（有 Change-Id） | 887 | 其余 76 个走 GitHub PR 流程 |
| 未带任何特殊标记的常规官方提交 | 823 | — |

**一句话结论：这个仓库里没有任何"非官方"提交。** 963 个提交全部原封不动地存在于 linuxdeepin/deepin-kwin 官方 master 中，本 fork 没有加入、删除或修改任何东西。

如果"非官方"指的是"官方仓库里，但不是 deepin/统信员工写的"，那么真正来自外部的只有 **2 个人工提交 + 3 个翻译机器人提交**，详见第三节。其余标记项（内部 cherry-pick、个人邮箱、AI 协作等）都属于官方开发流程内部的情况，列出来供参考。

---

## 二、fork 与官方仓库比对

```text
fork master      = 664808264f5a49c306834565d0e0c631a8feda27
upstream master  = 664808264f5a49c306834565d0e0c631a8feda27
git rev-list HEAD ^upstream/master | wc -l   -> 0   (fork 独有)
git rev-list upstream/master ^HEAD | wc -l   -> 0   (fork 落后)
```

fork 与官方完全同步，既没有领先也没有落后。

---

## 三、作者不属于 deepin / 统信的提交（逐条）

### 3.1 外部人员（真人）

| commit | 日期 | 作者 | 提交者（合入人） | 标题 | 说明 |
|---|---|---|---|---|---|
| `664808264f5a49c306834565d0e0c631a8feda27` | 2025-09-25 | Felix Yan `<felixonmars@archlinux.org>` | rewine `<luhongxu@deepin.org>` | fix: build against Qt 6.9+ | Arch Linux 打包者提交的构建修复（Qt 6.9 中 `QGenericUnixServices` 改名为 `QDesktopUnixServices`），2 文件 +11/-1，由 deepin 维护者合入 |
| `2bd16957fecd00aacae12c771d0136829947e7ac` | 2023-03-21 | Vlad Zahorodnii `<vlad.zahorodnii@kde.org>` | zhangyu `<zhangyud@uniontech.com>` | feat: Refactor compositing teardown in X11Window and Unmanaged | KDE KWin 上游补丁被统信员工移植进来，保留了原作者署名，附带统信 PMS bug 链接 254771，8 文件 +30/-16 |

### 3.2 机器人 / 自动化

| commit | 日期 | 作者 | 提交者 | 标题 | 说明 |
|---|---|---|---|---|---|
| `aa0a579806de69e35ccebcacc2f3d5d729f91137` | 2023-08-21 | gerritadmin `<gerritadmin@uniontech.com>` | gerritadmin | 'Init commit' | 统信 Gerrit 建仓空提交，只有一个空 `.gitignore` |
| `48eaf3c1f7647a97c37d781679328ef434e5d51f` | 2025-06-24 | transifex-integration[bot] | GitHub (web) | i18n：Updates for project kwin (#367) | Transifex 翻译同步，经 GitHub PR 合入 |
| `14632fab12b074903a97441e106908ece9924d2f` | 2025-06-26 | transifex-integration[bot] | GitHub (web) | Updates for project deepin kwin (#369) | 同上 |
| `af560bd312ace3504a8ee4b8cc3bf6dddec80071` | 2025-07-11 | transifex-integration[bot] | GitHub (web) | Updates for project TreeLand (#371) | 同上 |
| `f1b81267f6a47534876e7fb318f01bf4d5ba10f3` | 2025-04-07 | deepin-admin-bot `<deepin-admin-bot@deepin.org>` | deepin-admin-bot | chore: Sync by linuxdeepin/.github@0ff70d7 | 从 deepin 组织统一模板同步 7 个 `.github/workflows/*.yml`（备份到 GitLab、自动打 tag、CLA 检查、commitlint、license 检查、cppcheck 等） |

另有 `f560645`（作者 rewine，提交者 deepin-bot[bot]，更新 README）和 5 个经 GitHub 网页合并的提交（`48eaf3c` `89db151` `14632fa` `af560bd` `e2123e3`，GPG 状态 `E` = GitHub web-flow 签名，本地无 GitHub 公钥所以无法校验，不代表签名有问题）。

### 3.3 作者用个人邮箱（实为 deepin 开发者，属官方）

| commit | 日期 | 作者 | 提交者 | 标题 | 判断依据 |
|---|---|---|---|---|---|
| `305ad7b6ebc614c9f571c09bd2c6462a995f7f72` | 2025-06-04 | zccrs `<zccrs@live.com>` | zccrs | chore: Guard against a null blob pointer before calling blob()->blobId() | zccrs 是本仓库第二大提交者（以提交者身份合入 83 个提交），deepin 核心维护者；此提交带 `Co-authored-by: Copilot` |
| `17a3bbc1029029b8606b395703b305d82c7f2c41` | 2025-01-09 | justforlxz `<justforlxz@gmail.com>` | justforlxz | fix: missing window alpha check | 带统信 Gerrit Change-Id 和 PMS bug 296761；该邮箱以"爱折腾的小竹同学"身份另合入 30 个提交 |
| `228890d5dbda8c65c84c7e20ec7983d072155612` | 2025-01-10 | justforlxz `<justforlxz@gmail.com>` | guoyao `<guoyao@uniontech.com>` | fix: missing libgbm.so.1 check | 带统信 Gerrit Change-Id |
| `2e1db0dfac93ffa23acbc3d157709afc29ef2485` | 2023-12-05 | yangtian `<yangtian@unitontech.com>` | yangtian `<yangtian@uniontech.com>` | fix: collapse caused by surfaceitem setScissorAlpha | 作者邮箱是 `unitontech` 拼写错误，提交者是同一人的正确统信邮箱 |

---

## 四、AI 协作提交（Co-authored-by: Copilot）

| commit | 日期 | 作者 | 标题 | 改动 |
|---|---|---|---|---|
| `305ad7b6ebc614c9f571c09bd2c6462a995f7f72` | 2025-06-04 | zccrs | chore: Guard against a null blob pointer before calling blob()->blobId() | 1 文件 +5/-1 |
| `33d0ff8663662450add5acb60b6c3b8f82e0ff4b` | 2025-06-04 | rewine | chore: the variable name tileOutputPositon is misspelled | 变量名拼写修正 |
| `925dc91a036351ec177f87d63a9587b4ce6c1c74` | 2025-06-04 | rewine | fix: the typo: change "witch" to "which" in the comment. | 注释拼写修正 |

都是很小的改动，由 deepin 维护者提交。

---

## 五、KDE 上游代码相关提交

deepin-kwin 本身就是 KDE KWin 的派生项目，以下是明确引用或移植 KDE 代码的提交：

| commit | 日期 | 作者 | 标题 | 上游来源 |
|---|---|---|---|---|
| `fb932a112a3fe280ec23f43c6d88d24af211e6cd` | 2023-08-18 | Chaojiang Luo | init kwin from upstream v5.27.2 | 整体导入 invent.kde.org/plasma/kwin v5.27.2，**3339 个文件 / 1,300,272 行**，仓库绝大部分代码来自这里 |
| `2bd16957fecd00aacae12c771d0136829947e7ac` | 2023-03-21 | Vlad Zahorodnii（KDE） | feat: Refactor compositing teardown in X11Window and Unmanaged | KDE 原作者补丁 |
| `18d63bad13aa244eb9944c1ba69c993c0fa28b1e` | 2023-11-21 | liuzheng | fix: Fix the black screen issue with gles screencast | 参考 KDE commit 39ee72f3 |
| `2d3e05ce3d743b5a8b20e0872992fbdc2cba2de8` | 2024-01-25 | Lu YaNing | feat: protocol: add wl_pointer.warp event | 参考 wayland MR 340、KDE MR 3592 / 4480 |
| `4992d94a9477313aaad07a6ed6205ade8279f06a` | 2024-02-26 | Lu YaNing | fix: Revert "Xwayland:Update pointer state before ending the drag" | **回退** KDE MR 3592 的修复（会导致跨进程拖拽崩溃） |
| `09da202b2900630a06b9f2efbe4de2ba1953d503` | 2025-04-21 | YaoBing Xiao | chroe: increase GPU reset timeout again | 移植 KDE MR 4702 |
| `664808264f5a49c306834565d0e0c631a8feda27` | 2025-09-25 | Felix Yan | fix: build against Qt 6.9+ | 引用 Qt 上游变更 609639 |

另有 `e51d513`、`fc18056`、`0d8971b`、`12a8cd1` 等提交在说明里提到 "upstream"，但内容是 deepin 自己对 KWin 行为的修改（例如移除上游的 Meta 快捷键），不是移植代码。

---

## 六、从未公开内部分支 cherry-pick 的提交（50 个）

这些提交带有 `(cherry picked from commit xxx)`，但来源 commit **在官方 GitHub 仓库的所有分支（已全部拉取）中都不存在**，说明它们来自统信内部 Gerrit 的私有分支。它们仍是官方员工按官方流程合入的，只是原始出处不公开。

| # | commit | 作者日期 | 作者 | 标题 | 来源 commit（内部） |
|---:|---|---|---|---|---|
| 1 | `09da202b2` | 2025-04-21 | YaoBing Xiao | chroe: increase GPU reset timeout again | `97873f376e08` |
| 2 | `1f0a7b3e9` | 2025-03-12 | guoyao | refactor(main_wayland): not need signal handler | `abc4a4ed9560` |
| 3 | `76c96ce70` | 2025-03-11 | pengwenhao | fix: primary selection device resource crash when empty | `d9d66ba30a54` |
| 4 | `1f2d47a6f` | 2025-03-04 | YaoBing Xiao | feat: Add all connector status output | `8b951030075a` |
| 5 | `0195c83da` | 2025-03-04 | YaoBing Xiao | fix: screen frozen | `4494654cd5cb` |
| 6 | `05bb426bc` | 2025-03-03 | pengwenhao | fix: drm_restorer dead loop crash | `45bd2d3cb51f` |
| 7 | `c5492d914` | 2025-02-27 | zorowk | fix: remote access manager release crash | `1a3f8586dd44` |
| 8 | `b46adb10c` | 2025-02-26 | Lu YaNing | fix:  Crashes when rapidly plugging/unplugging external displays | `5ccd73d417a6` |
| 9 | `0c5e74293` | 2025-02-24 | Lu YaNing | fix: Add null pointer check for atoms in WindowRadius::updateWindowRadius | `d15a7d67b87b` |
| 10 | `a500d441a` | 2025-02-17 | pengwenhao | fix: retmoe access graphics stress test causes memory leak | `9928069f450e` |
| 11 | `768e8c56b` | 2025-02-08 | YaoBing Xiao | fix: delete udev timer | `8fe4b46dbfde` |
| 12 | `d06629348` | 2025-01-07 | YaoBing Xiao | fix: high cpu usage when adjusting brightness or color temperature | `147e320de1a4` |
| 13 | `81db552bb` | 2024-12-30 | YaoBing Xiao | fix: some X11 app in Alt+TAB icon display errors | `457ef6e48e4b` |
| 14 | `09667b087` | 2024-12-27 | YaoBing Xiao | fix: abnormal maximization window | `57a10b7ff379` |
| 15 | `f87521e2f` | 2024-12-23 | YaoBing Xiao | fix: multiple output initialization position are [0, 0] | `0db8f1cfc2c8` |
| 16 | `33234e528` | 2024-12-18 | huangzhenyan | fix: enable outputs appropriately | `0069d626953d` |
| 17 | `e5be7f72a` | 2024-12-05 | huangzhenyan | fix: do not send rgb(0,0,0) to CTM | `fc4ff5d9a716` |
| 18 | `3a325063c` | 2024-12-05 | yangtian | fix: dont raise the minimized splitscreen window | `1cefb0f72113` |
| 19 | `40c7e9ba4` | 2024-12-04 | huangzhenyan | fix: avoid deepin-screen-recorder crash | `8e4310daf312` |
| 20 | `f3fb544f5` | 2024-11-28 | huangzhenyan | fix: eyeprotection effect blink when dpms turns on | `b038ea326c3b` |
| 21 | `c34889bec` | 2024-12-03 | dingzhixuan | fix: [drm] turn of outputs when prepare for sleep | `dee9e59631bd` |
| 22 | `bbdc01988` | 2024-12-02 | huangzhenyan | fix: remote_access get invalid fd of a deleted remote_buffer | `82718be21a21` |
| 23 | `277cc1e3a` | 2024-12-02 | yangtian | fix: add effect log when changing status | `01457b1ee4ca` |
| 24 | `dadba30c8` | 2024-12-02 | huangzeming | fix: log the information of grab client | `c7b1a7b8f1cb` |
| 25 | `6404957cb` | 2024-11-19 | huangzeming | fix: rewrite maximizeex | `aefd42f2a38f` |
| 26 | `cd20c8ae1` | 2024-11-16 | huangzhenyan | fix: eyeprotection and colormode flick | `2d8826df38db` |
| 27 | `c25e98423` | 2024-11-13 | huangzeming | fix: network tip on lock screen | `68240fd03a40` |
| 28 | `8b1e31e08` | 2024-11-08 | huangzhenyan | fix: screen frozen after switch session | `15f11d455553` |
| 29 | `302d02da9` | 2024-10-28 | huangzhenyan | feat: add colormode switching with refactoring | `12174702b11e` |
| 30 | `77e6a3d3d` | 2024-10-17 | huangzhenyan | refactor: refactor drm blob: mode, gamma, ctm | `8e4107704c4c` |
| 31 | `7a721b560` | 2025-03-24 | Lu YaNing | fix: crash when hotplugging screens by updating window outputs | `e81356ba6ba5` |
| 32 | `e4ea04b32` | 2025-03-11 | Yixue Wang | fix: client not raised without xinput | `f2cce2b07fc7` |
| 33 | `caedab3f3` | 2025-02-11 | YaoBing Xiao | feat: Enable Debugging for GL Framebuffer | `1fb3bde40f68` |
| 34 | `df8ad4bcd` | 2025-02-05 | YaoBing Xiao | fix: asan heap-use-after-free | `d316968f89b8` |
| 35 | `2a7c524fb` | 2025-02-14 | YaoBing Xiao | fix: asan heap-use-after-free | `51e671e9dd16` |
| 36 | `4ccaaff92` | 2025-02-18 | pengwenhao | fix: root window _NET_KDE_COMPOSITE_TOGGLING was not set success | `31bc53d72308` |
| 37 | `ec4203b68` | 2025-01-04 | YaoBing Xiao | fix: Lag during composition after mode modification | `2cbf47df48fa` |
| 38 | `25ef3ba8b` | 2024-11-23 | huangzhenyan | fix: mode id confilt | `a49607f9d918` |
| 39 | `e4751bc1f` | 2024-08-22 | dingzhixuan | fix: add interface to perferred cursor show output | `209f5e1ae06f` |
| 40 | `dd8ab6cf8` | 2024-08-07 | dingzhixuan | fix: remove virtual output | `63c81f1754e1` |
| 41 | `fd57f88d2` | 2024-08-06 | dingzhixuan | fix: soft cursor will blink on pgx | `1200ee6c61a6` |
| 42 | `d38121a7b` | 2024-07-24 | dingzhixuan | fix: add edge software cursor item in pgx | `6e2c37fa64b0` |
| 43 | `1c4b9cfca` | 2024-06-07 | dingzhixuan | fix: pgx warmboot or coldboot crash | `d11458207d4e` |
| 44 | `7aa514032` | 2024-05-30 | dingzhixuan | fix: remote access buffer tearing | `e94752a7353c` |
| 45 | `c10c75384` | 2024-05-15 | zhijie geng | fix: subsurface window's position can't update immediately | `074a53b7015e` |
| 46 | `49ce09fee` | 2024-05-23 | dingzhixuan | fix: xdg popup position incorrect | `b9ea5f53a1ff` |
| 47 | `3765ef1b5` | 2024-05-07 | zhangyaning | fix: update keyboard focus when grabbing popup closed. | `722989f907b7` |
| 48 | `1c593d20a` | 2024-05-23 | chenjiafeng | feat: add function for select display output devices by outputlayoutrc file | `4960da918202` |
| 49 | `015b5b473` | 2024-05-09 | chenjiafeng | fix: kwin_wayland添加no-vt-switch参数 | `5735399778a3` |
| 50 | `bae184419` | 2024-01-16 | yangtian | fix: tooltip shadow | `28c214871a18` |

---

## 七、作者日期早于仓库创建日的提交（72 个）

仓库根提交是 2023-08-21，但下面这些提交的**作者日期**更早（最早 2020-10-19）。它们的提交日期都是 2023-08-21 之后，说明是把旧仓库 / 内部分支里写好的补丁（deepin 定制 Wayland 协议、为适配老版本 Qt 5.11 和 Debian 依赖做的降级等）重新应用到新仓库。作者都是统信员工（另有 1 个是 KDE 的 `2bd1695`），**不是外部代码**。

| # | commit | 作者日期 | 作者 | 标题 |
|---:|---|---|---|---|
| 1 | `8516e11aa` | 2020-10-19 | luochaojiang | feat: kwin:Reset position set flag if the window has been moved by seat |
| 2 | `b46ea1925` | 2022-04-01 | Chaojiang Luo | feat: Add ddeshell server implementation |
| 3 | `2ef71691d` | 2022-04-02 | ut004146 | feat: add clinet-management protocol |
| 4 | `52198fb91` | 2022-04-06 | ut004146 | fix: add send window id in wayland server |
| 5 | `16d550401` | 2022-04-06 | Chaojiang Luo | feat: Add new plasma shell role server implementation |
| 6 | `6e066f372` | 2022-04-07 | ut004146 | feat: add fakeinput axis event for capture |
| 7 | `7c4d50a97` | 2022-04-07 | Chaojiang Luo | feat: Add brightness wayland server implementation |
| 8 | `7fc7e1f80` | 2022-04-08 | Chaojiang Luo | feat: add ddeseat protocol server implementation |
| 9 | `7ef5d3e1a` | 2022-04-08 | ut004146 | fix: add strut protocol for supporting fashion mode |
| 10 | `afb0b4409` | 2022-05-30 | Chaojiang Luo | feat: Add screenshot extend interface |
| 11 | `6229ee193` | 2022-10-09 | Chaojiang Luo | feat: Add dde security protection |
| 12 | `1f9588f0d` | 2022-10-24 | Chaojiang Luo | feat: init dde security protection |
| 13 | `2bd16957f` | 2023-03-21 | Vlad Zahorodnii | feat: Refactor compositing teardown in X11Window and Unmanaged |
| 14 | `4ffc14db3` | 2023-05-04 | Chaojiang Luo | feat: change EGL build deps |
| 15 | `6f882023f` | 2023-05-04 | Chaojiang Luo | feat: degrade pipewire build deps |
| 16 | `f91264aaf` | 2023-05-04 | Chaojiang Luo | feat: degrade kde frameworks deps |
| 17 | `1130419cb` | 2023-05-04 | Chaojiang Luo | feat: degrade Qt build deps |
| 18 | `033587a4b` | 2023-05-04 | Chaojiang Luo | feat: download build tool deps |
| 19 | `c4252a416` | 2023-05-05 | Chaojiang Luo | feat: remove throwError |
| 20 | `75938e790` | 2023-05-05 | Chaojiang Luo | feat: adapt AbstractSystemPoller_iid |
| 21 | `9d21b6e7e` | 2023-05-05 | Chaojiang Luo | feat: fix Q_DISABLE_COPY_MOVE not found |
| 22 | `d0675279b` | 2023-05-05 | Chaojiang Luo | feat: fix QImage format not supported |
| 23 | `6cac9d823` | 2023-05-05 | Chaojiang Luo | feat: fix QHash init issue |
| 24 | `05d8aa214` | 2023-05-05 | Chaojiang Luo | feat: replace loadRelaxed with load |
| 25 | `ca35f5dc7` | 2023-05-05 | Chaojiang Luo | feat: fix vector and pair traverse |
| 26 | `b68a1353f` | 2023-05-05 | Chaojiang Luo | feat: fix QSet init error |
| 27 | `c609e2573` | 2023-05-05 | Chaojiang Luo | feat: change Qt::hex and Qt::dec to std |
| 28 | `560fc0470` | 2023-05-05 | Chaojiang Luo | feat: fix cbegin and cend not found |
| 29 | `211f58c0a` | 2023-05-05 | Chaojiang Luo | feat: add default constructor |
| 30 | `1a483bd23` | 2023-05-05 | Chaojiang Luo | feat: fix QString has no interface isUpper |
| 31 | `629850c7f` | 2023-05-05 | Chaojiang Luo | feat: replace Q_NAMESPACE_EXPORT with Q_NAMESPACE |
| 32 | `6b8c56d58` | 2023-05-05 | Chaojiang Luo | feat: add missing head file |
| 33 | `d4a544d9b` | 2023-05-05 | Chaojiang Luo | feat:remove consteval |
| 34 | `ed0109018` | 2023-05-05 | Chaojiang Luo | feat: fix std::map has not contains |
| 35 | `1b03f292f` | 2023-05-05 | Chaojiang Luo | feat: adapt QWheelEvent |
| 36 | `afac85bc4` | 2023-05-05 | Chaojiang Luo | feat: redefine LEDs with typedef QFlags |
| 37 | `6fbebfda4` | 2023-05-05 | Chaojiang Luo | feat: replace widget->screen with QGuiApplication::primaryScreen |
| 38 | `95e1e6bc0` | 2023-05-05 | Chaojiang Luo | feat: adapt QList<QByteArray> |
| 39 | `eedbdbae1` | 2023-05-05 | Chaojiang Luo | feat: replace convertTo with convertToFormat |
| 40 | `be48e66bc` | 2023-05-05 | Chaojiang Luo | feat: replace qmlRegisterAnonymousType with qmlRegisterType |
| 41 | `e9a866f29` | 2023-05-05 | Chaojiang Luo | feat: adapt QStringView |
| 42 | `43f440e1b` | 2023-05-05 | Chaojiang Luo | feat: replace Qt::SkipEmptyParts with QString::SkipEmptyParts |
| 43 | `1c4d10db3` | 2023-05-05 | Chaojiang Luo | feat: adapt cxx std::span |
| 44 | `be02ee926` | 2023-05-05 | Chaojiang Luo | feat: Add missing head file memory |
| 45 | `29a1e7a9a` | 2023-05-05 | Chaojiang Luo | feat: replace QRecursiveMutex with QMutex |
| 46 | `b31a2f49d` | 2023-05-05 | Chaojiang Luo | feat: replace Qt::endl with \n |
| 47 | `46d6a0395` | 2023-05-05 | Chaojiang Luo | feat: link X11 without X11 workspace |
| 48 | `234c88c13` | 2023-05-05 | Chaojiang Luo | feat: Adapt to qt 5.11 interface and linked lib |
| 49 | `fb5d941cc` | 2023-05-05 | Chaojiang Luo | feat: Add cmake module support |
| 50 | `114dad053` | 2023-05-05 | Chaojiang Luo | feat: replace libxcb-util-dev with libxcb-util0-dev |
| 51 | `80693f55e` | 2023-05-06 | Chaojiang Luo | workaround: delete QProcess::splitCommand |
| 52 | `2881a8c96` | 2023-05-06 | Chaojiang Luo | workaround: delete qmlRegisterSingletonInstance |
| 53 | `34a3370f5` | 2023-05-06 | Chaojiang Luo | workaround: delete qml setInitialProperties and createWithInitialProperties |
| 54 | `93d571191` | 2023-05-06 | Chaojiang Luo | workaround: fix ECM_GLOBAL_FIND_VERSION error |
| 55 | `6c3d5818c` | 2023-05-06 | Chaojiang Luo | feat: delete Qt::AA_DisableSessionManager on 5.11 |
| 56 | `b19e655cf` | 2023-05-06 | Chaojiang Luo | feat: adapt screencast |
| 57 | `935c17b22` | 2023-05-06 | Chaojiang Luo | feat: adapt qpa QWindowSystemInterface interfaces |
| 58 | `4a0ea22ba` | 2023-05-06 | Chaojiang Luo | feat:replace createTextureFromNativeObject with createTextureFromId |
| 59 | `f4a853e36` | 2023-05-12 | Chaojiang Luo | workaround: Add QScopeGuard define in kwin |
| 60 | `fc87aef75` | 2023-05-18 | Chaojiang Luo | feat: Add env to enable interfaces blacklist |
| 61 | `5d4b4383c` | 2023-05-19 | Chaojiang Luo | feat: support csd client use splitmenu |
| 62 | `b91d40426` | 2023-05-22 | Chaojiang Luo | feat: add decoration deepin-chameleon |
| 63 | `53d1a9fba` | 2023-05-24 | Chaojiang Luo | feat: Add compatibility for windowId |
| 64 | `63258401c` | 2023-05-25 | Chaojiang Luo | feat: Add effects scissorwindow |
| 65 | `2da503d5c` | 2023-06-13 | Chaojiang Luo | workaround: fix dock and lunch position issue |
| 66 | `3923eef9c` | 2023-07-11 | diguoliang | fix: the issue of slow dock startup |
| 67 | `b423fcefe` | 2023-07-13 | Chaojiang Luo | fix: Add scissor shader to render |
| 68 | `b032979ba` | 2023-07-14 | Chaojiang Luo | workaround: set default cornerRadius for all window |
| 69 | `05ae7fe6a` | 2023-07-14 | diguoliang | fix: the problem of minimizing DTK windows not working |
| 70 | `7ef13bb7d` | 2023-07-14 | Chaojiang Luo | fix: dispatch focus to onscreendisplay |
| 71 | `d3704c6d2` | 2023-08-16 | Chaojiang Luo | feat: Add ctm feature |
| 72 | `fb932a112` | 2023-08-18 | Chaojiang Luo | init kwin from upstream v5.27.2 |

---

## 八、其他检查项（未发现问题）

- **签名**：958 个无 GPG 签名（N），5 个为 GitHub web-flow 签名（E，本地无公钥无法校验）；没有签名校验失败（B）的提交。
- **二进制文件**：初始导入之后新增的二进制文件只有 `b5e7de7`（feat: add multitaskview）和 `24742b1` 中的 PNG 图标与 `.qm` 翻译文件，与提交说明相符。
- **构建脚本**：检查了导入后所有对 CMake / debian / shell / yml 的修改，没有新增 `curl` / `wget` / `git clone` / `FetchContent` / `ExternalProject` 这类构建期联网下载。`033587a` 的标题 "download build tool deps" 实为 "downgrade"（降低 cmake / breeze / debhelper 的最低版本要求），属拼写错误。
- **CI 工作流**：`.github/workflows/` 只由 `f1b8126`（deepin-admin-bot 从 linuxdeepin/.github 统一同步）引入，内容是 deepin 组织的标准工作流。

---

## 九、作者统计（git shortlog）

| 提交数 | 作者 |
|---:|---|
| 165 | huangzeming `<huangzeming@uniontech.com>` |
| 118 | Chaojiang Luo `<luochaojiang@uniontech.com>` |
| 107 + 36 | zhangyu / zhang yu `<zhangyud@uniontech.com>` |
| 69 | dingzhixuan `<dingzhixuan@uniontech.com>` |
| 68 + 2 | rewine `<luhongxu@deepin.org>` / `<luhongxu@uniontech.com>` |
| 63 + 1 | yangtian `<yangtian@uniontech.com>`（含 1 个 unitontech 拼写错误） |
| 60 | xinbo wang `<wangxinbo@uniontech.com>` |
| 44 | zhijie geng `<gengzhijie@uniontech.com>` |
| 40 + 5 | zhangyaning / zhang yaning `<zhangyaning@uniontech.com>` |
| 39 | huangzhenyan `<huangzhenyan@uniontech.com>` |
| 其余 | 均为 `@uniontech.com` / `@deepin.org`，除第三节列出的 9 个提交外 |

作者邮箱域名分布：uniontech.com 885，deepin.org 69，github noreply（Transifex 机器人）3，gmail.com 2，unitontech.com（拼写错误）1，live.com 1，kde.org 1，archlinux.org 1。

---

## 十、复现方法

```bash
git fetch --unshallow origin master
git remote add upstream https://github.com/linuxdeepin/deepin-kwin
git fetch upstream '+refs/heads/*:refs/remotes/upstream/*'
git rev-list HEAD ^upstream/master | wc -l        # fork 独有提交数
git log --format='%h|%an <%ae>|%cn <%ce>|%s' | grep -vE '@(uniontech\.com|deepin\.org)>\|'
git log --grep='cherry picked from' --format='%h %s'
git log -i --grep='co-authored-by: copilot' --format='%h %s'
```
