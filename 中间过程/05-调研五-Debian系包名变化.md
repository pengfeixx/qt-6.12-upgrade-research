# 调研五：Qt 6.8 → 6.12 Debian 系包名与打包变化（原始记录）

- 调研日期：2026-10-08
- 调研方式：独立调研代理（WebFetch/WebSearch Debian/Ubuntu 包索引、deepin 社区公开信息）
- 用途：支撑《阶段1-Qt6.8到6.12接口变化与包名变化分析.md》§6
- 以下为代理返回的原始内容，未删改

---

# Qt 6.8 → 6.12 Debian 系打包变化调研报告

## 一、Debian 官方打包现状

**各 suite 跟踪的 Qt6 版本（已确认）**
- bookworm (12, oldstable)：`6.4.2+dfsg-10`
- trixie (13, stable)：`6.8.2+dfsg-9+deb13u2`
- forky (14, testing) = sid (unstable)：**`6.11.2+dfsg-5`**（两 suite 同步）
- **Qt 6.12 尚未进入 sid**。tracker 显示 VCS 中有未上传的 `6.11.2+dfsg-6`，并提示上游已有 6.12.0。即当前 Debian 归档内最新的 Qt6 小版本为 **6.11.2，位于 sid 与 forky**
- 来源：https://tracker.debian.org/pkg/qt6-base 、https://packages.debian.org/sid/qt6-base-dev

**t64 时区/time_t 过渡（已确认）**
- `libqt6core6 → libqt6core6t64`（连带 libqt6gui6t64、libqt6dbus6t64 等）发生在 2024 年初的 64 位 time_t 过渡，当时 Debian sid 的 Qt6 仍是 **6.4.2** 系列（即 Qt 6.8 之前）；Ubuntu 24.04 首次随 6.4.2 发布 t64 名。bookworm 保留旧名，trixie 起全部 t64 后缀
- 升级含义：deepin 当前 `libqt6core6`（无 t64）是 bookworm 基线命名；对齐 Debian trixie+ 基线需接受 t64 后缀或自维护改名
- 来源：https://packages.debian.org/sid/libqt6core6t64 、https://launchpad.net/ubuntu/+source/qt6-base/+changelog 、https://bugs-devel.debian.org/1074633

**源码包清单变化（已确认，基于 packages.debian.org 源码包名搜索对比）**
- trixie(6.8.2) 共 37 个 `qt6-*` 源码包；sid(6.11.2) 相比 trixie **新增 4 个**：
  - `qt6-tasktree`（二进制 qt6-tasktree-dev `6.11.2-3`，forky+sid）
  - `qt6-canvaspainter`（qt6-canvaspainter-dev `6.11.2-4`，forky+sid）
  - `qt6-pim`（trixie 无 qt6-pim-dev，确认 trixie 后新增）
  - `qt6-ukui-platformtheme`（UKUI 平台主题，非 Qt 上游模块）
- `qt6-graphs`、`qt6-grpc`（含 protobuf）在 trixie 已存在，**非** 6.8 后新增
- **尚无** `qt6-openapi` 源码包（上游 6.12.0 已有 qtopenapi tarball）；**不存在** `qt6-labs-stylekit` —— StyleKit 不是独立 tarball/源码包，随 declarative/controls 分发
- **无移除**：trixie 的 37 个全部保留在 sid（qt6-datavis3d、qt6-quickeffectmaker 仍在）
- 来源：https://packages.debian.org/search?suite=sid&keywords=qt6-&searchon=sourcenames 、https://packages.debian.org/search?suite=trixie&keywords=qt6-&searchon=sourcenames 、https://packages.debian.org/sid/qt6-tasktree-dev 、https://packages.debian.org/sid/qt6-canvaspainter-dev

**二进制包命名规律（6.9–6.12）**
- 已确认现状（sid 6.11.2）：`qt6-<module>-dev`、`libqt6<module>6t64`、`qml6-module-*` 模式全部延续，未改名
- 推断：6.12 进入 sid 时命名不变，仅新增 qt6-openapi 等源码包

## 二、Ubuntu 打包现状

| 发行版 | Qt6 版本 | 状态 |
|---|---|---|
| 24.04 noble | 6.4.2（含 t64 改名） | 已确认 |
| 24.10 oracular | 6.6.2+dfsg-7 | 已确认 |
| 25.04 plucky | 6.8.2+dfsg-0ubuntu2 | 已确认 |
| 25.10 questing | 6.8.3+dfsg-0ubuntu2 | 已确认 |
| 26.04 resolute (LTS) | **6.10.2**（qt6-base-dev 6.10.2+dfsg-7） | 已确认 |

- 26.04 不会用 6.12：其 2026-04 发布早于 Qt 6.12.0（2026-09-30）（推断，基于日期事实）
- t64 之后（6.6→6.10）**未发现任何二进制包改名公告**（已确认，多渠道搜索无果）
- 来源：https://code.launchpad.net/ubuntu/+source/qt6-base/+changelog 、https://packages.ubuntu.com/resolute/qt6-base-dev 、https://launchpad.net/ubuntu/+source/qt6-base/6.8.2+dfsg-0ubuntu2 、https://pkgs.org/search/?q=qt6-base-dev

## 三、上游发布物命名

**源码 tarball（已确认）**
- 命名模式不变：`download.qt.io/official_releases/qt/6.12/6.12.0/submodules/` 下共 44 个模块，全部为 `<module>-everywhere-src-6.12.0.tar.xz`（含 qtbase-everywhere-src-6.12.0.tar.xz、qtcanvaspainter-、qttasktree-、qtopenapi-、qtgraphs-、qtgrpc-），另有合并包 `qt-everywhere-src-6.12.0.tar.xz`
- 来源：https://download.qt.io/official_releases/qt/6.12/6.12.0/submodules/ 、https://forum.qt.io（qt-everywhere-src-6.12.0 主题）

**在线安装器/MaintenanceTool 组件命名（已确认）**
- 不变：`qt.qt6.612.<arch>`（如 qt.qt6.612.gcc_64）、`qt.qt6.612.addons.<module>`；SDK 仓库路径 `online/qtsdkrepository/<os>/<arch>/qt6_612_*/`。6.12 CLI 文档仍按此命名
- 6.12.0 为 LTS（5 年维护），特性含 QML Hot Reload、StyleKit、Canvas Painter
- 来源：https://doc.qt.io/qt-6/get-and-install-qt-cli.html 、https://qtproject.mirror.liquidtelecom.com/online/qtsdkrepository/ 、https://www.phoronix.com（Qt 6.12 LTS 发布报道）

## 四、Deepin 侧

- **已确认 deepin 25 线在适配 Qt 6.9.2**（deepin 社区月报 2025-08："正在进行 Qt 6.9.2 的适配"）；deepin 25 beta DDE 移植指南含“Qt 6.9 说明”章节（DDE 尚未完整测试、QML import 需修正等已知问题）
- deepin-community GitHub 的 qt6-* 各仓库（qt6-location、qt6-wayland 等）公开提交仍停留在 6.8.x 系列；Qt 打包由 [sig-deepin-qt](https://github.com/deepin-community/sig-deepin-qt) SIG 统筹
- **未发现** deepin 打包或计划 Qt 6.10/6.11/6.12 的公开尝试
- 来源：https://www.deepin.org/zh/deepin-community-monthly-report-2025-8 、https://deepin-community.github.io/sig-dde-porting/posts/deepin-25-beta-dde-porting-guide/ 、https://github.com/deepin-community/qt6-location

## 五、对 6.8 → 6.12 升级调研的结论

1. **包名规律整体稳定**：6.9–6.12 期间 Debian/Ubuntu 的 `qt6-*-dev`、`libqt6*6t64`、`qml6-module-*` 命名零变化；唯一历史性命名事件是 t64（6.4 时代，早于 6.8）
2. **新增打包面**：6.12 相比 6.8 多出 tasktree、canvaspainter、openapi（上游 12 个 submodules 差异中 Debian 已打包 tasktree/canvaspainter，openapi 待打包）；无移除
3. **基线参考**：Debian 侧最完整的参考实现是 sid 的 6.11.2 打包（6.12 预计同构）；Ubuntu 26.04 是 6.10.2 的 LTS 参考实现；deepin 自己的下一目标是 6.9.2（6.8 → 6.12 跨度大于 deepin 现有路线）
