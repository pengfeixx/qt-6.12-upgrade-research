# 调研三：Qt Widgets / Network / Multimedia / SVG / Wayland 接口变化（原始记录）

- 调研日期：2026-10-08
- 调研方式：独立调研代理（WebFetch/WebSearch 官方文档）
- 用途：支撑《阶段1-Qt6.8到6.12接口变化与包名变化分析.md》§4.6、§4.7、§4.8、§4.9
- 以下为代理返回的原始内容，未删改

---

# Qt 6.9 → 6.12 五模块接口变化调研（Deepin/UOS，Wayland+X11）

**数据来源说明**：用户提供的 per-module Changes 页面（`qtwidgets-changes-qt6.html`、`qtnetwork-changes-qt6.html`、`qtwaylandandclient-changes-qt6.html`）在当前 doc.qt.io 已 404；`qtmultimedia-/qtsvg-changes-qt6.html` 仍存在但只含 Qt5→6 迁移内容；`qt-dist-changes-6-x.html` 也已下线。6.9 起 Qt 官方将 per-module 变更并入各版本 What's New 页与 qtreleasenotes 仓库，本次以其为权威来源：
- WN69/610/611/612 = https://doc.qt.io/qt-6/whatsnew69.html / whatsnew610.html / whatsnew611.html / whatsnew612.html
- RN69/610/611/612 = https://github.com/qt/qtreleasenotes/blob/dev/qt/6.9.0/release-note.md（6.10.0/6.11.0/6.12.0 同理）
- BlogHC = https://www.qt.io/blog/high-contrast-mode-in-qt-6.10

## 一、Qt Widgets

- 行为｜6.9｜QWidget hide 事件｜已隐藏的 widget 再次隐藏时不再收到 hide 事件｜RN69
- 行为｜6.9｜QContextMenuEvent｜鼠标 press 被接受后仍发送上下文菜单事件（官方预告 Qt 7 将改为不发送，建议对未处理鼠标事件调用 ignore()）｜RN69/RN610
- 行为｜6.9｜GTK3 原生菜单（Linux）｜因 gtk_menu_popup() 弃用而移除，菜单改由 Qt 自绘｜RN69
- 行为｜6.9｜QColorDialogOptions｜不再自动把自定义颜色保存到全局 QSettings("QtProject")｜RN69
- 行为｜6.9｜QIcon::pixmap()｜不再按 DPR 缩放传给 QIconEngine::scaledPixmap() 的尺寸参数｜RN69
- 行为｜6.9｜Fusion 样式｜QPushButton 遵循 PM_ButtonMargin｜RN69
- 新增｜6.9｜QDockWidget::dockLocation、QStackedWidget::widgetAdded()、QComboBox::labelDrawingMode、QWidget::accessibleIdentifier；QHeaderView 大模型内存优化；QAbstractItemView::updateThreshold｜WN69
- 行为｜6.10｜高对比度主题自动应用｜Qt 自动检测系统高对比度设置（Windows 对比主题/macOS Increase Contrast/GNOME High Contrast）：Fusion 样式自动加强控件描边，Windows 11 样式改用 GetSysColor 取色；无需代码改动，但自研 QStyle 子类需自行适配｜WN610/BlogHC
- 行为｜6.10｜WA_ContentsMarginsRespectsSafeArea｜非顶层窗口不再默认设置该属性｜RN610
- 行为｜6.10｜QDockWidget 标签页分组｜分组标签栏改用各 dock 的窗口图标｜RN610
- 行为｜6.10｜GNOME 全屏快捷键｜由 Ctrl+F11 改为 F11｜RN610
- 重构｜6.10｜QAbstractItemDelegate::handleEditorEvent()｜item view 默认编辑器事件处理（Tab/Enter/Esc/焦点移出）移入该新虚函数，自定义 delegate 覆盖后应链回基类以保留标准行为｜WN610/RN610
- 新增｜6.10｜QLayout::horizontalSizeConstraint/verticalSizeConstraint；QScrollBar::createStandardContextMenu()；QListWidget/QTableWidget/QTreeWidget::setSupportedDragActions()｜WN610
- 行为｜6.11｜QTabBar｜tabsClosable 为真时中键点击即发出 tabCloseRequested()｜WN611/RN611
- 行为｜6.11｜QAction 图标文本｜连 Unicode 省略号"…"也一并剥离｜RN611
- 新增｜6.11｜QWizard::StretchBanner；QAbstractItemView::keyboardSearchFlags；QColumnView::previewColumnVisible；QStyleOptionMenuItemV2｜WN611
- 新增｜6.12｜QStyle::SP_TabScrollLeft/Right/Up/DownButton；QTreeView::itemDecorationRect()；QDialogButtonBox::standardButtonText()/standardButtonShortcut()；QStyleHints::toolTipWakeUpDelay｜WN612/RN612
- 行为｜6.12｜文本编辑 emoji 删除｜deletePreviousChar() 一次删除整个 emoji 序列（肤色/ZWJ/旗帜等）｜RN612
- 不兼容｜6.9-6.12｜常用类（QStyle/QDialog/QAbstractItemView 主体）无 ABI/API 删除，以上均为行为微调与新增｜综合

## 二、Qt Network

- 状态｜6.9｜QNetworkRequestFactory/QRestAccessManager/QRestReply｜结束"under development"，API 稳定｜RN69
- 行为｜6.9｜QHttpHeaders｜放宽 header 值校验，允许 UTF-8｜RN69
- 行为｜6.9｜QNetworkRequest::transferTimeout｜超范围值饱和到 INT_MIN/INT_MAX 而非报错｜RN69
- 新增｜6.9｜QHttp2Configuration::setMaxConcurrentStreams()｜WN69
- 行为｜6.10｜QNetworkAccessManager｜连接中断时不再错误重发非幂等请求（如 POST）｜RN610
- 行为｜6.10｜QSslCertificate::fromPath()｜不再接受空路径（原行为会搜索当前目录）；新增 fromFile()｜RN610
- 行为｜6.10｜data: URL 解析｜按 RFC 2397 逗号变为必需｜RN610
- 行为｜6.10/6.11｜QNetworkRequestFactory::createRequest()｜不再解码用户传入的 path，保持原样编码｜RN610/RN611
- 行为｜6.11｜QNetworkAccessManager 空闲连接｜默认启用 TCP keepalive 且不活跃连接 2 分钟后自动关闭；可用环境变量 QT_QNAM_TCP_KEEPIDLE / QT_QNAM_TCP_KEEPINTVL / QT_QNAM_TCP_KEEPCNT 调整——长连接应用需注意｜WN611/RN611
- 行为｜6.11｜QNetworkReply｜传输超时错误码由 OperationCanceledError 改为 TimeoutError｜RN611
- 行为｜6.11｜自定义 HTTP operation｜GET/PUT/POST/DELETE/HEAD 语义的自定义操作会被映射为标准方法，可能影响缓存与重定向处理｜RN611
- 新增｜6.11｜QNetworkRequest 每请求 TCP keepalive 参数；QAbstractSocket::KeepAlive*Option；QAuthenticator::clear()；QSsl 支持 ML-DSA 签名算法｜RN611
- 澄清｜—｜HTTP/2 默认｜HTTP/2 自 Qt 6.0 起即默认开启，6.9-6.12 未改变该默认（6.11 whatsnew 中 HTTP/2 并发一条属 Qt HTTP Server 模块）｜doc.qt.io/qt-6/network-changes-qt6.html
- 新增｜6.12｜QSslKeyingMaterial + QSslConfiguration::keyingMaterial()/setKeyingMaterial()（RFC 5705/8446 密钥导出）；QHostInfo::clearCache()；QHttpHeaderRange/Set 与 QHttpHeaders 的 QDataStream 运算符；OpenSSL 4 支持构建｜WN612/RN612
- 行为｜6.12｜QDesktopUnixServices（Linux）｜openUrl/openDocument 优先走 XDG desktop portal D-Bus 而非 xdg-open，行为可能有差异｜RN612
- 行为｜6.12｜QNetworkProxy｜libproxy 后端识别 socks5h 代理 scheme｜WN612

## 三、Qt Multimedia（Linux 重点）

- 要求｜6.9｜GStreamer 后端｜最低版本由 1.18 提升至 **1.20**（Deepin/UOS 软件源需确认）｜RN69
- 行为｜6.9｜QAudioDevice 相等比较｜仅依据 id() 与 mode()，不再比较完整设备信息｜RN69
- 行为｜6.9｜QT_MEDIA_BACKEND｜环境变量不区分大小写；录制/ capture 文件名已带扩展名时不再改动｜RN69
- 不兼容｜6.9｜QML VideoSink.videoFrameChanged｜信号参数移除 QVideoFrame（该信号未正式文档化）｜RN69
- 弃用｜6.9｜QMediaRecorder::encoderSettingsChanged()｜RN69
- 新增｜6.9｜Wayland 下经 PipeWire 的屏幕采集；VideoOutput::endOfStreamPolicy、mirrored；FFmpeg 升至 n7.1｜RN69/WN69
- 弃用｜6.10｜Windows 原生媒体后端｜自 6.10 起官方文档标记弃用（Linux 无关，但多平台代码需知）｜RN610
- 新增｜6.10｜原生 PipeWire 音频后端（新增可选项，未成为默认）；FFmpeg 后端 pitchCompensation 音调补偿；QPlaybackOptions；QAudioSink/QAudioSource 帧计数接口；QT_MULTIMEDIA 强制要求 FEATURE_thread｜WN610/RN610
- 行为｜6.10｜QAudioSource/QAudioSink｜以默认构造 QAudioFormat 创建时改用设备首选格式｜RN610
- 新增｜6.11｜QAudioSource/QAudioSink 回调式 start() 重载（实时音频处理）｜WN611
- 弃用｜6.12｜QMediaMetaData::ThumbnailImage｜后端改填 CoverArtImage 值，请迁移｜RN612
- 行为｜6.12｜QSoundEffect::setSource()｜URL 未变化也会发出 sourceChanged()｜RN612
- 新增｜6.12｜QMediaCaptureSession::setNativeVideoSource() + GStreamerVideoSource QML 元素（自定义 GStreamer 管线接入采集会话，对 Deepin 定制采集有用）；QScreenCapture/QWindowCapture::maximumFrameRate；nativePeriodFrameCount/setNativePeriodFrameCount；FFmpeg 升至 n8.1.2/9.0.1｜WN612/RN612
- 澄清｜—｜默认后端｜Linux 桌面默认仍为 FFmpeg（自 6.5 起），6.9-6.12 未发生默认后端切换；PipeWire 为 6.10 新增可选音频后端｜综合

## 四、Qt SVG

- 新增｜6.9｜feBlend 滤镜原语；QtSvg::Options 新增 3 个禁用动画的选项值；CSS 动画支持 fill/stroke/transform｜RN69/WN69
- 修复/新增｜6.10｜CSS 动画支持 fill-opacity/stroke-opacity/opacity；修复文本元素透明度渲染｜WN610/RN610
- 新增｜6.11｜CSS motion path 动画（offset-path 仅支持 path() 函数、offset-distance）；预定义 CSS timing function｜WN611
- 新增｜6.12｜QMovie 与 QML AnimatedImage 支持动画 SVG｜WN612/RN612
- 不兼容/弃用｜6.9-6.12｜无｜综合

## 五、Qt Wayland（client + compositor）

- 行为｜6.9｜wl_surface 生命周期｜窗口的 wl_surface 与窗口同生命周期，hide 时不再销毁——deepin 合成器若依赖 surface 销毁判断窗口关闭需注意｜WN69
- 新增(client)｜6.9｜xdg-toplevel-icon-v1（使 QWindow::setIcon() 生效）、xdg_system_bell_v1、appmenu(DBus 菜单栏)、wlr-data-control-unstable-v1；wayland.xml 升至 1.23｜WN69/RN69
- 不兼容(compositor)｜6.9｜QWaylandShellSurface::setWindowGeometry()｜不再适合传达 CSD 内容边界，自定义 shell 实现应改用新虚函数 QWaylandShellSurface::setContentGeometry()——deepin 这类自研 shell 集成必须适配｜RN69
- 变更｜6.10｜client 与 compositor 均无 noteworthy API 变化（仅协议元数据/构建信息更新）｜RN610
- 新增(client)｜6.11｜服务端按键重复（server-side key repeat）、经 zxdg_decoration_manager_v1 表达服务端装饰偏好、实验性 xx-session-management-v1、color-management-v1；另有 popup/滚动/剪贴板稳定性修复｜WN611/RN611
- 移除(qtbase/X11)｜6.11｜XCB 的 XRender 原生绘制引擎被移除（QT_XCB_NATIVE_PAINTING 不复存在）——X11 平台残留依赖需清理｜WN611
- 新增(client)｜6.12｜XdgSessionManager 协议（正式会话恢复，取代实验性 xx-session-management）；linux-dmabuf-v1 stable v5（qtbase 侧）；bradient CSD 主题显示最小化/最大化/关闭提示｜WN612/RN612
- 新增(compositor)｜6.12｜linux-dmabuf-v1 stable 协议 v4｜新 buffer integration 名为 "linux-dmabuf-v1"，旧 v3 实现更名 "linux-dmabuf-unstable-v1" 继续可用——deepin 合成器 buffer integration 配置建议迁移｜WN612/RN612
- 构建｜6.12｜CMake 策略 QTP0006｜qt_generate_wayland_protocol_{client,server}_sources() 在 NEW 下默认 PRIVATE_CODE｜WN612
- 弃用/drop 协议｜6.9-6.12｜未发现移除任何 wayland 扩展协议；client 侧实际移除的仅 GTK3 原生菜单（qtbase DBusMenu，6.9）与 XRender XCB 引擎（6.11）｜综合

## 六、升级风险速览（Deepin/UOS 相关 Top 项）

1. GStreamer 后端最低 1.20（6.9 起）；若软件源 GStreamer 过旧且依赖该后端需评估。
2. QNAM 空闲连接 2 分钟自动关闭（6.11）——长轮询/推送类应用重点回归。
3. 高对比度自动应用（6.10）——deepin 若有自研 QStyle 需自行适配 contrast 语义。
4. QWaylandShellSurface::setContentGeometry()（6.9）与 linux-dmabuf v4（6.12）——deepin 合成器适配点。
5. wl_surface 不再随 hide 销毁（6.9）——合成器生命周期假设需回归。
6. XRender XCB 引擎移除（6.11）——X11 兼容层清理。

**结论**：五个模块在 6.9-6.12 期间无大规模 API 删除，Widgets/Network/SVG 以行为微调与新增为主；Multimedia 与 Wayland 各有一个关键适配点（GStreamer 1.20、compositor setGeometry 拆分）。6.8 → 6.12 源码兼容性总体良好，主要工作量在行为回归测试而非代码改写。
