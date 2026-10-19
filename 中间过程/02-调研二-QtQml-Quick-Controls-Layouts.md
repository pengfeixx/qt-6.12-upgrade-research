# 调研二：Qt Qml / Qt Quick / Quick Controls / Layouts 接口变化（原始记录）

- 调研日期：2026-10-08
- 调研方式：独立调研代理（WebFetch/WebSearch 官方文档 + qtdeclarative 源码核实弃用宏）
- 用途：支撑《阶段1-Qt6.8到6.12接口变化与包名变化分析.md》§4.4、§4.5
- 以下为代理返回的原始内容，未删改

---

## 调研结论：Qt 6.9–6.12 QML/Quick 系列接口与行为变化

**重要前置说明（来源核实结果）**：Qt 6 在 doc.qt.io 上**没有** Qt5 时代那种按次版本分节的 "Changes to Qt Qml" 页面。用户给的 `qtqml-changes-qt6.html`、`qtquick-changes-qt6.html`、`qtquickcontrols2-changes-qt6.html` 均 404；实际存在的 `qml-changes-qt6.html`、`quick-changes-qt6.html`、`qtquickcontrols-changes-qt6.html` 都是 Qt5→Qt6 迁移页，不含 6.9–6.12 条目。官方对次版本变化的权威披露渠道是：各版本 whatsnew 页（含逐模块条目）、qt.io 发布博客、CMake/类文档中的 "introduced in Qt 6.x" 标记，以及源码中的 `QT_DEPRECATED_VERSION_X_6_x` 弃用宏（本次已逐一核实）。以下 [WN-6x]=https://doc.qt.io/qt-6/whatsnew6x.html，[Blog-6x]=https://www.qt.io/blog/qt-6.x-released。

### 一、Qt Qml（语言/引擎/工具链）

- 【弃用｜6.9】`QJSValue::isVariant()` — 标记弃用，结果不可靠。来源：qtdeclarative 6.12 分支 src/qml/jsapi/qjsvalue.h（github.com/qt/qtdeclarative）
- 【行为｜6.9】`qmlformat` — 新增按最大行长自动断行，格式化输出变化。[WN-69]
- 【不兼容｜6.10】`TableModel`（QtQml.Models）— **移除**对复杂行结构的操作支持。[WN-610]
- 【不兼容｜6.10】`TableModel::setData()` — role 与 value 参数顺序对调（消除歧义），调用方需改。[WN-610]
- 【行为｜6.10】`qmllint` — 新增一批默认告警：Qt Creator/Design Studio 兼容性检查、JavaScript 告警、QML 枚举与 import 相关告警，现有工程告警数会上升。[WN-610]
- 【行为/语言｜6.11】QML 语言新增 `override`、`virtual` 关键字（配合已有 `final`），属性遮蔽（shadowing）检测收紧，见 "Property Shadowing and Override Semantics"。[WN-611]
- 【行为/语言｜6.11】`easingCurve` 成为命名值类型，`Easing` 命名空间改为单例。[WN-611]
- 【弃用｜6.11】`QQmlFile` — 整类（构造/析构及全部方法）弃用，改用 QQmlComponent。github.com/qt/qtdeclarative/blob/6.12/src/qml/qml/qqmlfile.h
- 【弃用｜6.11】`QQmlPropertyMap(QObject*)` 构造函数 — 弃用，改用 `create()` 或两参保护构造。…/src/qml/util/qqmlpropertymap.h
- 【行为｜6.11】`qmllint` — 上下文属性告警可用 `.contextProperties.ini` 配置文件消音（配合上条 6.10 告警收紧）。[WN-611]
- 【行为/重要｜6.12】**缺失 required properties 的组件：引擎现在直接报错**，不再静默构造不完整对象。这是 6.8→6.12 最需回归测试的引擎级行为变化。[WN-612]
- 【行为｜6.12】`string.arg()` — 支持 JS 风格多参数调用。[WN-612]
- 【行为｜6.12】qmltypes 生成 — 未注册类型记为 "opaque" 类型；工具链不再告警、qmlls 补全/跳转可用，且 **qmlsc/qmlcachegen 可将更多 binding 编译为 C++**。[WN-612]
- 【行为｜6.12】`qmllint`/`qmlls` — 对未知类型只告警一次（不再逐 binding 告警）；另新增 5 类告警：被遮蔽的类型成员、不兼容 file selector 变体（opt-in）、递归实例化类型、按文件名而非 qmldir 名引用类型、JS `new Array(n>1)`。[WN-612]
- 【工具｜6.12】`qmllint` 新增 `--only-explicit-categories` 选项（压制全部默认告警）。[WN-612]
- 【弃用｜6.12】`qmlDeprecatedRegisterTypesAndRevisions()` — 弃用（不再需要）。…/src/qmlintegration/qqmlintegration.h
- 【新增｜6.12】`QQmlEngine::setExternalSingletonInstance()`；QML 热重载内置于引擎（qmlpreview）；`Qt.escapeHtml()`。[WN-612、Blog-612]

### 二、Qt Quick

- 【行为｜6.9】`QQuickPaintedItem` — 恢复 FramebufferObject 渲染模式（仅 OpenGL 后端，Qt5 移植辅助）。[WN-69]
- 【行为｜6.9】场景图渲染器缓冲池优化 — 大几何场景内存峰值显著降低；海量 item 开销降低。属性能行为变化。[WN-69]
- 【新增｜6.9】`SafeArea` 附加属性 + `Qt::ExpandedClientAreaHint`/`NoTitleBarBackgroundHint` 窗口标志 + `QWindow::safeAreaMargins()`（影响 QQuickWindow 边到边布局策略）。[WN-69、Blog-69]
- 【新增/无破坏｜6.11】`Flickable` 定位/轻扫到子项 API；`qt_target_qml_from_svg` CMake 命令。[WN-611]
- 【行为/视觉｜6.12】`Text` — 现在正确渲染全部 `QTextCharFormat::UnderlineStyle`（虚线/波浪/拼写检查等），此前一律实线；富文本视觉会变。[WN-612]
- 【行为｜6.12】`Rectangle::color` — 赋 `undefined` 可恢复默认值 white。[WN-612]
- 【新增｜6.12】`Color` 单例、`AnimatedImage.loops/finishBehavior/finished()`、`ShapePath.strokeGradient`、VectorImage 的 `asynchronous/status` 等。[WN-612]
- 6.10：无破坏性变化（仅 FlexboxLayout，见 Layouts 节）。

### 三、Qt Quick Controls

- 【行为/重要｜6.9】`TextField`/`TextArea` — **默认自带 ContextMenu**（右键/菜单键弹出）；已有自定义菜单时不会重复弹出，但原依赖"无菜单"的 UI 行为改变。[WN-69、Blog-69]
- 【行为/样式｜6.10】**内置样式自动对齐系统高对比度设置**（桌面+移动），无需改代码即改变外观；同时无障碍（屏幕阅读器）呈现方式全面梳理。[Blog-610]
- 【样式｜6.10】FluentWinUI3 样式改进 Windows 11 对比度主题支持。[WN-610]
- 【行为｜6.11】无障碍客户端导航/角色变化通知改进（Controls+Widgets 共通）。[Blog-611]
- 【新增｜6.11】`DoubleSpinBox`；`DialogButtonBox.defaultButton/defaultStandardButton`。[WN-611]
- 【新增/样式｜6.12】**`QtQuick.Controls.Native` 新样式** — 编译期按平台选择最佳样式，改变样式选择策略。[WN-612]
- 【行为/视觉｜6.12】`MenuItem` — 自动显示关联 `Action` 的快捷键，现有菜单视觉变化。[WN-612]
- 【行为｜6.12】`SearchField` — 输入框为空时支持键盘导航。[WN-612]
- 【新增｜6.12】`ToolTip.policy`、`ComboBox.highlightOnHover`、`RangeSlider.crossingEnabled`、`Menu.separatorsCollapsible`。[WN-612]

### 四、Qt Quick Layouts

- 【新增(TP)｜6.10】`FlexboxLayout`（qtquick/layouts 模块，技术预览）— CSS Flexbox 风格布局。https://doc.qt.io/qt-6/qml-qtquick-layouts-flexboxlayout.html
- 6.9：无。6.11：官方明示 "No noteworthy changes"。6.12：无（whatsnew 无该模块条目）。

### 五、CMake / qt_add_qml_module

- 【新增｜6.12】`qt_add_qml_module()` 新增 `NO_GENERATE_QTCONF` 选项（抑制构建目录自动生成 qt.conf）。https://doc.qt.io/qt-6/qt-add-qml-module.html、[WN-612]
- 【新增｜6.12】每个含 QML 模块的可执行目标自动获得 `<exe>_qmlpreview` 目标（联动引擎热重载）。[WN-612]
- 6.9/6.10/6.11：qt_add_qml_module 文档中无 6.9–6.11 版本标注的参数变化 → 无。

### 六、QtQml 私有头文件

- 【不兼容(私有)｜6.12】Qt Qml Compiler 私有 API：`QQmlSA::FixSuggestion::hint()/setHint()` **移除**；`fixDescription()` **改名** `description()`。[WN-612]
- 【弃用(私有)｜6.11】`QQmlSslConfiguration::sslOptions()/setSslOptions()`（qqmlsslconfiguration_p.h）→ 改用 `sslOptionFlags`。github.com/qt/qtdeclarative
- 其余私有头各版本均无兼容承诺，直接内嵌私有头的代码（DDE/UOS 常见）需按版本重编核验。

### 升级风险要点（Deepin/UOS QML 重度使用场景）

1. **6.12 required properties 缺失即报错**（引擎级，最可能的升级阻断点）；2. **6.9 TextField/TextArea 默认右键菜单**；3. **6.10 TableModel 两处不兼容**；4. **6.10/6.11 qmllint 默认告警大幅收紧**（CI 需调整或用 `.contextProperties.ini`/`--only-explicit-categories`）；5. 6.10 内置样式对齐系统高对比度、6.12 Text 下划线/MenuItem 快捷键显示等视觉行为变化需 UI 回归；6. qmlsc 可编译范围在 6.12 扩大（opaque 类型），建议升级后重跑编译期检查获取收益。
