# DTK 库在 Qt 6.12 净室中的打包记录

> 目标：从 github.com/linuxdeepin 拉取最新 DTK 相关库，在 Qt 6.12 净室
> (chroot2) 中用 `dpkg-buildpackage -b -us -uc` 打包；问题记录优先，不做
> 隐性修改。dtkwidget 经用户确认后做了源码适配（同时保证 Qt 6.8 可打包）。
> 日期：2026-11-02。工作目录：`qt6.12-build/dtk-src/`（克隆）、
> `qt6.12-build/build-results-dtk/`（产物+日志）。

## 1. 源码与环境

| 仓库 | commit | 日期 | 结果 |
| --- | --- | --- | --- |
| dtkcommon | f0523ee3 | 2026-10-28 | ✅ 6.7.52 |
| dtklog | b87fab6d | 2026-10-28 | ✅ 6.7.52 (Qt6 变体 libdtk6log) |
| dtkcore | df64f9ad | 2026-10-30 | ✅ 6.7.52 (libdtk6core) |
| dtkgui | fa3d6fd7 | 2026-10-28 | ✅ 6.7.52 (libdtk6gui) |
| dtkwidget | e82521d0 | 2026-10-28 | ✅ 6.7.52（修复后，见 §3） |
| dtkdeclarative | 0159003c | 2026-10-30 | ✅ 6.7.52（libdtk6declarative + chameleon QML 样式） |
| dtkdevice | 02cbbcd9 | 2026-03-16 | ✅ 1.0.4（Qt5 时代打包，原样构建） |
| dtkio | 3f0d6d5d | 2024-07-22 | ❌ 失败，见 §4 |

- 净室：`chroot2`（UOS snipe 基础 + aptly RC 仓库提供 Qt 6.12.0-1 /
  6.12.0+dfsg-0deepin0 全家 + 主池/dde-snipe PPA）。dtk-src、build-results-dtk
  分别绑定挂载到 `/build/dtk`、`/build/dtk-results`。
- 构建方式：`scripts/build-dtk.sh <repo>`（chroot 内 root 运行）：
  `apt-get build-dep .` → `DEB_BUILD_PROFILES="nodtk5 nodoc"
  dpkg-buildpackage -us -uc -b`。**nodtk5** 是 dtk 上游 control/rules 原生
  支持的构建 profile（只出 DTK6/Qt6 变体，不出 Qt5 变体），nodoc 跳过文档
  包；不属于打包改动。完整日志：`build-results-dtk/logs/<repo>.log`。
- 每个仓库成功后产物移入 `build-results-dtk/` 并在净室内
  `apt-get install ./*.deb`，供下游仓库 build-dep 取到 6.7.52。
- 共产出 **33 个 deb**（含 dbgsym），已全部装入净室。

## 2. 上游依赖环境差异（记录，未构成阻塞）

- `qt6-wayland-dev-tools` 候选仍是 6.8.0-0deepin15（RC 仓库无此包），
  与其余 qt6 6.12 组件混装；dtkgui 构建未受影响（qtwaylandscanner 可用）。
- `apt-get build-dep` 不识别 DEB_BUILD_PROFILES，把 `<!nodtk5>` 的 Qt5
  构建依赖（qtbase5-dev 等）也装进了净室；主池均有，无害。
- `treeland-protocols (>>0.5.8)`：主池 0.7.0 满足，无阻塞。
- dtkcommon 打包年代久远（debhelper compat 9），dh 全程告警
  "Compatibility levels before 10 are deprecated"、
  `${shlibs:Depends} used, but is not defined`；不影响产物。
- dtkgui 对 6.12 编译干净（`D_IGNORE_DEPRECATIONS` 已抑制废弃告警），
  成品 `ldd -r` 无未定义符号。

## 3. dtkwidget 对 Qt 6.12 的适配（源码修复，用户确认后实施）

### 3.1 故障现象

master（e82521d0）在净室 Qt 6.12 下 `dpkg-buildpackage -b -us -uc` 于
**链接 examples/collections 可执行文件**时失败：

```
/usr/bin/ld: ../../src/libdtk6widget.so.6.7.52: undefined reference to
  `QExplicitlySharedDataPointer<QFontPrivate>::~QExplicitlySharedDataPointer()'
```

库本体 libdtk6widget.so 编译通过（共享库允许未定义符号），下游任何可执行
文件链接它都会失败。同一代码在 Qt 6.8（UOS 池 6.7.47/6.7.51）构建正常。

### 3.2 根因（有最小复现与符号级证据）

- 引用点唯一：`src/widgets/dstyleoption.cpp:385`
  `const auto fontD = D_PRIVATE_MEMBER(font, QFont_d_tag{});` ——
  **按值拷贝** `QExplicitlySharedDataPointer<QFontPrivate>`（QFont::d 私有
  成员，经 `D_DECLARE_PRIVATE_MEMBER` 显式实例化技巧访问，
  dstyleoption.cpp:17 + `<private/qfont_p.h>`）。
- 目标文件 `dstyleoption.cpp.o` 中该析构 D1 符号只有 `U`（引用）没有
  `W`（内联定义发射）；拷贝构造 C1/C2 却正常发射为 W。QtCore/QtGui 均未
  导出该模板符号（nm -DC 确认），Qt 6.12 头文件中该析构为类内内联、亦无
  extern template 抑制 —— 是 Qt 6.12 对 QExplicitlySharedDataPointer 的
  重构（改为继承 QSharedDataPointerBase 的 CRTP 结构）与该"按值拷贝私有
  成员"用法组合下，GCC 不再在本编译单元发射该析构实例（6.8 无此问题）。
- 最小复现（`dtk-src/_repro/`，净室 6.12 头文件直接编译）：
  - 按值拷贝 + 宏 → D1 为 U（复现故障）；
  - **改为按引用绑定（`const auto &`）→ 符号完全消失**；
  - 自有类型按值拷贝（无宏/无私有类型）→ 析构正常 W 发射。

### 3.3 修复

`/home/uos/work/dtk/dtkwidget/src/widgets/dstyleoption.cpp:385`
（工作树修改，未提交）：

```diff
-        const auto fontD = D_PRIVATE_MEMBER(font, QFont_d_tag{});
+        const auto &fontD = D_PRIVATE_MEMBER(font, QFont_d_tag{});
```

引用绑定不产生拷贝/析构的 odr-use，语义与原按值拷贝完全一致（后续仅读
`fontD->dpi`），且免一次原子引用计数加减；无版本条件、6.8/6.12 通用。
全仓库扫描确认其余 D_PRIVATE_MEMBER 用法（dblureffectwidget.cpp:1111、
dtabbar.cpp:2388-2392）均为指针/引用/自有类型，不受影响。

### 3.4 双环境验证

| 环境 | Qt | 命令 | 结果 |
| --- | --- | --- | --- |
| 净室 chroot2 | Qt 6.12.0-1 / 6.12.0+dfsg-0deepin0 | `DEB_BUILD_PROFILES="nodtk5 nodoc" dpkg-buildpackage -us -uc -b` | ✅ exit 0，全部二进制包产出（含 dtk6widget-examples），成品 `ldd -r` 无未定义符号 |
| 宿主机 UOS 25 | Qt 6.8.0+dfsg-0deepin36 | 同上（用户工作树 `/home/uos/work/dtk/dtkwidget` 原地构建） | ✅ exit 0，同套二进制包产出；宿主机构建日志 `/tmp/dtkwidget-68-build.log`，产物在 `/home/uos/work/dtk/` |

修复后的 libdtk6widget 6.7.52 已装入净室，dtkdeclarative 随之基于
6.7.52 widget 重新构建（此前临时用池内 6.7.51 widget 构建过一版）。

### 3.5 上游提示

dtkgui 的 `src/util/dfontmanager.cpp:15` 声明了同一访问器
（`D_DECLARE_PRIVATE_MEMBER(QFont_d_tag, ...)`），当前仅指针级使用故未
触发；若将来出现按值拷贝同样会在 6.12 下断链，建议上游同步改引用绑定。

## 4. dtkio 失败记录（未修改，两个独立问题）

1. **control 构建依赖缺失**：CMake 需 `Qt6Core5Compat`，但
   debian/control 未声明 `qt6-5compat-dev`。净室内手工安装
   qt6-5compat-dev 6.12 后进入下一错误（此为打包缺陷，已记录）。
2. **Qt5/Qt6 混合解析冲突（结构性）**：
   ```
   CMake Error: The INTERFACE_QT_MAJOR_VERSION property of "Qt5::Core"
   does not agree with the value of QT_MAJOR_VERSION already determined
   for "dtkburn". （dtksmount 同）
   ```
   dtkio 1.0.1（2024-07 后未更新）是 Qt5 时代代码（硬依赖 Qt5::Core、
   libudisks2-qt5、libqt5xdg），而其 dtk cmake 链路在 Qt6 优先的环境把
   QT_MAJOR_VERSION 判为 6。需上游改造方可构建，本轮按约定止于记录。

## 5. 推仓后状态（2026-11-03 复核）

用户侧完成打包推仓后，RC 源（aptly snipe-2500u2 release-candidate）已可
检索到 dtk 全家，**但经核实为 Qt 6.8 环境构建的产物线，与 Qt 6.12 栈互斥**：

- 源内版本：libdtk6core **6.7.52.1-1**，其余（common/log/gui/widget/
  declarative/chameleon）为 6.7.52；libdtkdevice 未推。
- 关键证据：源内 libdtk6core 6.7.52.1-1 与 libdtk6widget/libdtk6gui 6.7.52
  的 Depends 均含 `qt6-base-private-abi (= 6.8.0)`，且 Qt 依赖钉在
  `6.8.0+dfsg-0deepin42/36`；该 anchor 包在本净室所有已配源中均不存在，
  `apt-get install libdtk6core` 报 "Depends: qt6-base-private-abi (= 6.8.0)
  无法满足"。对照：本净室 6.12 构建的 dtk 包 Depends 为
  `libqt6*6 (>= 6.12.0)`、无 private-abi 钉死。
- 净室处置：dist-upgrade 曾尝试以源内同版本 6.7.52（6.8 构建）覆盖个别
  附属包（widget-bin/dbgsym），已用本地 deb 重装还原，并对 dtk 主包
  `apt-mark hold`（18 个包），防止后续批量升级再次混入 6.8 构建；
  `apt-get check` 一致性通过。
- 结论与建议：若目标是 Qt 6.12 全栈进 RC 源，需把本净室 6.12 构建的
  33 个 deb（`build-results-dtk/`）推仓，或让内部流水线在 Qt 6.12 环境
  重建 dtkcore（源内 6.7.52.1-1 亦为 6.8 构建）。另注意 6.12 线的
  dtkcore 若沿用公司版本规范，建议以 6.7.52.1 重打（避免与源内 6.8 版
  同版本号冲突）。

## 6. 交付物

- deb（33 个）：`qt6.12-build/build-results-dtk/`
- 各仓库完整构建日志：`qt6.12-build/build-results-dtk/logs/*.log`
- 构建驱动脚本：`qt6.12-build/scripts/build-dtk.sh`
- 最小复现：`qt6.12-build/dtk-src/_repro/`
- dtkwidget 修复补丁：`/home/uos/work/dtk/dtkwidget`（git 工作树，
  仅 `src/widgets/dstyleoption.cpp` 一处改动）
- 6.8 回归产物：`/home/uos/work/dtk/*.deb`、`/tmp/dtkwidget-68-build.log`
