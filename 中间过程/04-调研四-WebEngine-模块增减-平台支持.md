# 调研四：Qt WebEngine / PDF、模块增减、平台支持变化（原始记录）

- 调研日期：2026-10-08
- 调研方式：独立调研代理（WebFetch/WebSearch 官方文档与 wiki）
- 用途：支撑《阶段1-Qt6.8到6.12接口变化与包名变化分析.md》§4.10、§4.11、§5、§3
- 以下为代理返回的原始内容，未删改

---

# Qt 6.8 → 6.12 升级可行性调研报告

## A. Qt WebEngine 6.9–6.12 变化

### A1. Chromium 上游版本
| Qt 版本 | Chromium 基线 |
|---|---|
| 6.8.0–6.8.6（你的基线） | 122.0.6261.171（注意 6.8.7 LTS 已升 130.0.6723.192、6.8.8 升 134.0.6998.208） |
| 6.9.0–6.9.3 | 130.0.6723.192 |
| 6.10.0–6.10.3 | 134.0.6998.208 |
| 6.11.0–6.11.2 | 140.0.7339.264 |
| 6.12 | 不随 Qt 发布（见下） |

来源：https://wiki.qt.io/QtWebEngine/ChromiumVersions （基线版本，安全补丁持续回移植）

- 【重大】Qt 6.12 起 WebEngine 独立发布：首个独立版本 Qt WebEngine 6.140.0，与 Qt 6.12 同期（文档站已单独存在，基于 6.140 分支），复用 Chromium 140 基线并叠加补丁（至 151.0.7922.173）；后续按 Chromium 主版本递增（6.141…）。来源：https://wiki.qt.io/QtWebEngine/ChromiumVersions 、https://doc-snapshots.qt.io/qtwebengine/qtwebengine-index.html 、6.12 release note（https://code.qt.io/cgit/qt/qtreleasenotes.git/plain/qt/6.12.0/release-note.md）

### A2. 逐版本条目（whatsnew + qtreleasenotes）
- 6.9【不兼容级】权限 API 重写：新增 QWebEnginePermission，权限请求模型改变。来源：https://code.qt.io/cgit/qt/qtreleasenotes.git/plain/qt/6.9.0/release-note.md
- 6.9【行为】下载 API 异步化：downloadRequested 不再限于同步 accept/cancel，可延后处理。来源：同上
- 6.9【行为】内部渲染切换到 ANGLE（影响 Windows/D3D 路径）；另新增 PreferCSSMarginsForPrinting、JSTouchEventsEnabled 设置与 QWebEngineUrlRequestInfo::isDownload()。来源：https://doc.qt.io/qt-6/whatsnew69.html
- 6.10【新特性】Chromium extension 扩展支持、按 Profile 附加受信 CA 证书、Back/Forward 缓存设置。来源：https://doc.qt.io/qt-6/whatsnew610.html
- 6.11【移除】Pepper Plugin API（PPAPI）被移除，依赖 Pepper 插件的应用受影响。来源：https://doc.qt.io/qt-6/whatsnew611.html
- 6.11【新特性】WebExtension API 支持（JS 扩展 API 桥接到 C++，社区确认）。来源：https://github.com/qutebrowser/qutebrowser/issues/8616
- 6.11【行为】新增 OnlyPersistentCookies cookie 策略；可选 Rust 组件构建（-feature-webengine-rust-build，默认仍 C++）；支持 HTML inputmode。来源：https://doc.qt.io/qt-6/whatsnew611.html 及 6.11 release note
- 6.12/6.140【结构性】qtwebengine、qtpdf 及 qtwebview 的 webengine 后端移出 Qt 6.12 发布包（"does not ship with qtwebengine nor qtpdf"），改随独立 WebEngine 6.140 发布——Linux 打包/部署脚本需相应调整。来源：https://code.qt.io/cgit/qt/qtreleasenotes.git/plain/qt/6.12.0/release-note.md
- 说明：官方 per-module 页 https://doc.qt.io/qt-6/qtwebengine-changes-qt6.html 实际只含 Qt5→Qt6 迁移内容（模块重命名、defaultProfile 改离线模式等），6.9–6.12 无官方 Incompatible/Degraded 分类清单，以上为 whatsnew+release note 汇总；未发现 "Degraded Features" 类条目。

### A3. 构建要求（WebEngine 6.140 官方值；Qt 6.8 时代页面未提供可对比数字）
- Python ≥ 3.8；Node.js 须 64 位 ≥ 20.0；CMake ≥ 3.19；Ninja；gperf/bison/flex/html5lib。来源：https://doc-snapshots.qt.io/qtwebengine/qtwebengine-platform-notes.html
- 编译器：需 C++20 能力（Qt 其余模块仅 C++17）；Linux 上 GCC ≥ 10 或 Clang ≥ 17，glibc ≥ 2.16、NSS ≥ 3.26；X11 需 libdrm/xrandr/xkbcommon 等全套 pkg-config 依赖；不支持静态构建。来源：同上
- 内存/磁盘：官方页未给数字；社区实测 QtWebEngine 6.x 源码+构建约 12 GB 磁盘、链接期建议 16–32 GB RAM（Gentoo 打包标记 ~24 GiB）。来源：https://www.linuxfromscratch.org/blfs/view/git/x/qtwebengine.html

## B. 其余本地已装模块 6.9–6.12 变化

- Qt Positioning：6.10 修复 QGeoPolygon::toString()、新增 holePath() 的 QDataStream 序列化；6.11 GeoClue2 DesktopId 改用 desktopFileName()（applicationName() 回退计划在下个大版本移除——升级预警）；6.12 行为变化：QGeoAreaMonitorPolling 的 start/stopUpdates 改经 QueuedConnection（更新在事件循环停止而非立即），SatelliteSystem 枚举新增 IRNSS、SBAS。来源：https://code.qt.io/cgit/qt/qtreleasenotes.git/plain/qt/6.10.0/release-note.md 及 6.11/6.12 同路径
- Qt WebChannel：6.9–6.12 各 release note 与 whatsnew 均无条目，接口稳定，预计零改动。来源：https://doc.qt.io/qt-6/qtwebchannel-changes-qt6.html（仅 Qt5 迁移内容）
- Qt SerialPort：6.9 新增 QT_SERIALPORT_SKIP_UDEV_LOOKUP 环境变量（可跳过 udev 枚举串口）、恢复 settingsRestoredOnClose 属性、Windows availablePorts() 改用 USB iManufacturer/iProduct 字段；6.10 新增写缓冲区大小限制、修复 Windows readyRead() 递归发射。来源：6.9/6.10 release note（同上路径）
- Qt HttpServer：6.10 新增 keep-alive 超时；6.11 路由可返回 QFuture<void>（responder 可跨线程）、可配置请求体大小限制；6.12 新增每 IP 活动连接数与总并发连接数限制（有默认值，长连接服务需关注）。来源：6.10–6.12 release note
- Qt StateMachine：6.9–6.12 无任何 release note 条目，接口稳定。来源：6.9–6.12 release note（无该模块节）
- Qt Remote Objects：6.9 暴露创建 QtRO 类型的私有 API（供 Python 等绑定从 .rep 生成动态类；6.10 note 中再次完善），公有 API 无不兼容变化。来源：6.9/6.10 release note
- Qt PDF：模块本身 6.9–6.11 无显著 API 条目；【重要】6.12 起 qtpdf 不再随 Qt 发布，改随 WebEngine 6.140 独立发布（源码本就同仓 qtwebengine）；另 6.9 起 qtbase 的 QPdfWriter 支持 PDF/X-4 输出与 output intent 定制（注意这是 Qt GUI 非 qtpdf）。来源：https://code.qt.io/cgit/qt/qtreleasenotes.git/plain/qt/6.12.0/release-note.md 、qt/6.9.0 同路径
- Qt SVG / SVG Widgets：6.9 新增 feBlend 滤镜原语；6.10 修复文本元素透明度渲染；6.11 潜在不兼容——QSvgGenerator 重新实现 QPaintDevice::initPainter()，子类必须回调基类实现；6.12 动画 SVG 可用于 QMovie/QML AnimatedImage、XSVG 相关代码移入 Qt GUI、修复 CVE-2026-6210/CVE-2026-8168。来源：6.9–6.12 release note、https://doc.qt.io/qt-6/whatsnew612.html
- Qt Tools：6.9 QDoc 要求 libclang ≥ 17（验证至 LLVM 20）、header 内文档注释为 TP；Designer 新增 qtVersion 属性且 macOS 默认 docked 模式；Qt Solutions 公共头改私有 _p.h（不兼容）；QHelpContentItem 设为 final 且不可拷贝（不兼容）。6.10 QDoc 新增 \notranslate、\qmlenum、源码链接生成，qt_add_translations 新增 QM/TS_OUTPUT_DIRECTORY。6.11 Linguist 新增 AI 翻译、标签、ltext2id/lcheck 工具，lupdate 支持 qTrId()，QFormBuilder 可保存枚举/标志属性。6.12 QDoc 兼容 LLVM/Clang 22、支持 Markdown 模板输出，新增 Objective-C 名字改编工具 objcnamemangler，Designer 支持 local-socket 客户端。来源：各版本 release note 及 whatsnew611/612
- 附（本地装有 5compat）：6.12 弃用 QBinaryJson::toRawData()。来源：6.12 release note

## C. 模块级增减与状态（6.8 → 6.12）

### C1. 新增模块
- Qt Canvas Painter（6.11 新增，TP；6.12 核心 API 转正，QtCanvas2D QML API 仍 TP）：硬件加速命令式 2D 绘制，适用 Qt Quick 与 QRhi；独立仓库（2025-11 邮件列表新建 repo）。来源：https://doc.qt.io/qt-6/qtcanvaspainter-index.html 、https://lists.qt-project.org/pipermail/development/2025-November/046738.html 、https://www.qt.io/blog/qt-6.12-released
- Qt TaskTree（6.11 新增，TP）：异步任务流的声明式组合/执行框架，独立仓库 qt/qttasktree。来源：https://github.com/qt/qttasktree 、https://code.qt.io/cgit/qt/qtreleasenotes.git/plain/qt/6.11.0/release-note.md
- Qt OpenAPI（6.11 新增，TP）：从 OpenAPI 规范生成 Qt HTTP 客户端（服务端生成），qtopenapi 仓库（6.12 release note 已有独立 qtopenapi 节）。来源：https://www.phoronix.com/news/Qt-6.11-Beta-1 、6.12 release note
- Qt Labs StyleKit（6.11 引入 labs QML 模块，TP；6.12 扩展至 Widgets 并持续 Labs）：单一样式源同时为 Widgets 与 Qt Quick 提供主题/样式。来源：6.11 release note（qtdeclarative 节）、https://www.qt.io/blog/qt-6.12-released
- Qt Qml Design Support（6.12 唯一正式新增模块）：QML 对象注册/引用类型集，面向未来 Quick 设计工具。来源：https://doc.qt.io/qt-6/whatsnew612.html

### C2. 移除/分离
- qtwebengine、qtpdf 移出 Qt 6.12 发布包（独立产品 WebEngine 6.140），qtwebview 的 webengine 后端随之转移。来源：https://code.qt.io/cgit/qt/qtreleasenotes.git/plain/qt/6.12.0/release-note.md
- 6.10 起 Qt Charts、Qt Data Visualization 被弃用（推荐迁移 Qt Graphs，模块仍随附）。来源：https://doc.qt.io/qt-6/whatsnew610.html
- 未发现 6.9–6.12 有模块 "moved to marketplace"（marketplace 迁移均属 2020 年 Qt5 时代旧事）。来源：搜索结论（https://www.qt.io/blogQt Marketplace 归档页）

### C3. 技术预览转正
- Qt Graphs：6.8 已转正（早于你的升级窗口，无需动作）。来源：https://www.qt.io/blog/whats-new-in-qtgraphs-6.9
- Qt Canvas Painter 核心 API：6.12 转正（QtCanvas2D、QCanvasCustomBrush 仍 TP）。来源：https://www.qt.io/blog/qt-6.12-released
- QDoc documentationinheaders：6.9 TP → 6.10 稳定。来源：https://doc.qt.io/qt-6/whatsnew610.html
- Lottie（lottietoqml 工具 + VectorImage 后端）：6.11 不再 TP。来源：https://doc.qt.io/qt-6/whatsnew611.html
- CMake 部署 API（qt_finalize_project、qt_deploy_translations、QT_DEPLOY_SUPPORT 等）：6.12 离开 TP。来源：https://doc.qt.io/qt-6/whatsnew612.html
- FlexboxLayout（6.10 TP，基于 yoga 2.0.1）：至 6.12 未见转正公告，仍按 TP 对待。来源：https://doc.qt.io/qt-6/whatsnew610.html
- Qt Quick 3D XR：6.11 增强（multiview 渲染）、6.12 支持 2D AR 眼镜，未见转正公告。来源：https://www.phoronix.com/news/Qt-6.12-Beta-1
- Qt WebView C++ 绑定：6.11 以 TP 提供。来源：https://doc.qt.io/qt-6/whatsnew611.html

### C4. 平台支持变化（Linux 桌面重点）
- CMake 最低版本：6.9 起构建与使用均需 ≥ 3.22；6.12 起提高至 ≥ 3.25（6.10/6.11 无再提高的公告）。来源：https://doc.qt.io/qt-6/whatsnew69.html 、https://doc.qt.io/qt-6/whatsnew612.html
- glibc：6.10 起官方预编译二进制改在 RHEL 9 上构建，运行时最低 glibc 由 2.28 升至 2.34（6.12 页面确认 "linked against glibc 2.34"）；Deepin/UOS 需确认系统 glibc ≥ 2.34，否则只能源码编译。来源：https://doc.qt.io/qt-6/whatsnew610.html 、https://doc.qt.io/qt-6/supported-platforms.html
- 编译器：6.12 支持矩阵列 Debian 11（GCC 10）至 openSUSE 16（GCC 15）；WebEngine 额外要求 GCC ≥ 10 / Clang ≥ 17 且 C++20。来源：https://doc.qt.io/qt-6/supported-platforms.html 、https://doc-snapshots.qt.io/qtwebengine/qtwebengine-platform-notes.html
- X11：6.11 移除 XCB 平台上遗留的 X11/XRender 原生绘制引擎（使用原生绘制 API 的代码为不兼容项）。来源：https://doc.qt.io/qt-6/whatsnew611.html
- Wayland：6.9 新增 xdg-toplevel-icon-v1、wl_surface 与窗口同生命周期；6.11 新增服务端按键重复、装饰偏好、实验性会话管理、AT-SPI Collection 接口（改善 Orca 支持）。来源：whatsnew69/611
- 其他：6.12 打开 URL 优先走 XDG portal；Qt 6.12 是最后支持 Windows 10 的版本（LTS 五年维护）。来源：https://www.qt.io/blog/qt-6.12-released 、supported-platforms 页

主要来源索引：wiki.qt.io/QtWebEngine/ChromiumVersions；doc.qt.io/qt-6/whatsnew69|610|611|612.html；code.qt.io/cgit/qt/qtreleasenotes.git（qt/6.9.0–6.12.0/release-note.md）；doc-snapshots.qt.io/qtwebengine/（WebEngine 6.140 独立文档）；doc.qt.io/qt-6/supported-platforms.html；qt.io 官方博客（qt-6.12-released 等）。
