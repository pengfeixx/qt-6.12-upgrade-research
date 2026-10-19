# 调研一：Qt Core / Qt GUI / Qt Core5Compat 接口变化与工具链要求（原始记录）

- 调研日期：2026-10-08
- 调研方式：独立调研代理（WebFetch/WebSearch 官方文档）
- 用途：支撑《阶段1-Qt6.8到6.12接口变化与包名变化分析.md》§3、§4.1、§4.2、§4.3
- 以下为代理返回的原始内容，未删改

---

# Qt 6.9–6.12 升级调研：Core / GUI / Core5Compat 接口变化与工具链要求

**来源说明**：官方页 `qtcore-changes-qt6.html`（含版本化快照）经验证只含 Qt5→Qt6 迁移内容，`changes-6.9.html` 等 URL 均为 404；Qt 官方按版本的权威变更清单是各版 `release-note.md`（whatsnew 页面即链接它）。以下条目源自 qtreleasenotes 各版本 release-note.md 的 qtbase/qt5compat 分节 + whatsnew 页 + supported-platforms 页。6.10 note 中重复收录的 6.9 条目已按内容去重。

## 一、Qt Core

**6.9**
| 类型 | API | 说明 |
|---|---|---|
| 不兼容 | moc/QMetaObject 元对象数据表 | 含 Q_REVISION 方法的类（revision≥13）数据表布局不兼容，直接读写原始数据的代码须适配 [RN69] |
| 不兼容 | QChar 构造函数 | 不再做隐式实参转换，只接受文档所列精确类型，需显式 cast [RN69] |
| 不兼容 | QString::arg() | 新增 QAnyStringView/QUtf8StringView 重载；C 字符串字面量/QByteArray 不再隐式转 QString [RN69] |
| 不兼容 | qtypeinfo.h / qstringfwd.h / QJsonDocument 头 | 移除多个传递包含（optional/tuple/variant、qglobal.h、QJsonValue），依赖传递包含的代码编译失败 [RN69] |
| 不兼容+弃用 | QT_ENABLE_QEXPLICITLYSHAREDDATAPOINTER_STATICCAST | 宏被弃用并移除 [RN69] |
| 弃用 | qsnprintf()/qvsnprintf() | 6.8 预警后正式弃用（因 asprintf 回退引入平台差异），改用 std::snprintf [RN69] |
| 弃用 | Qt::compareThreeWay() 指针重载 | 已弃用 [RN69] |
| 弃用 | QBasicTimer::start(chrono::milliseconds) | 弃用，新增 nanoseconds 重载 [RN69] |
| 行为 | QtCore 内嵌 MIME 数据库 | 由 freedesktop shared-mime-info 换成 Apache Tika（无图标/翻译描述，文件类型匹配结果可能不同）[RN69] |
| 行为 | QSet::remove()/removeIf() | 无实际删除时不再 detach [RN69] |
| 行为 | QSpan | 由共享容器转换而来的 QSpan 不再触发 detach [RN69] |
| 行为 | qEnvironmentVariable 等严格化 | q(v)nprintf 增加 printf 格式属性（新编译警告）；QT_SUPPORTS_INT128 依赖标准库支持（-ansi/-std=c++NN 下不再可用）；QT_ENABLE_STRICT_MODE_UP_TO 不再含 QT_NO_CAST_FROM_ASCII [RN69] |

**6.10**
| 类型 | API | 说明 |
|---|---|---|
| 不兼容 | QLockFile(QString) | 构造函数变为 explicit，`QLockFile f = str;` 失效 [RN610] |
| 行为 | QMetaObject QVector→QList 移植兼容 hack | 已弃用；信号槽参数类型以 "QVector<" 开头的匹配行为修正 [RN610] |
| 行为 | QDirListing::ExcludeSpecial | 更名为 ExcludeOther，旧名成过时别名 [RN610] |
| 行为 | QStringConverter::nameForEncoding() | 非法枚举值返回 nullptr（原为未定义行为）[RN610] |
| 行为 | QCoreApplication::installTranslator() | 空翻译器也返回 true 且会被注册查询，过早删除将 UB [RN610] |
| 行为 | QPointer null 判断 | 对象析构中途的 `!p`/`p==nullptr` 不再触发非法下转型（原为 UB）[RN610] |

**6.11**
| 类型 | API | 说明 |
|---|---|---|
| 不兼容 | QTimeZone(Qt::TimeSpec) | 从未被支持的构造现于编译期拒绝（原先被误匹配到 int 重载）[RN611] |
| 不兼容 | qAddOverflow/qSubOverflow/qMulOverflow | 不再接受 char 作模板实参，需改用 qint8/quint8 [RN611] |
| 弃用 | QDir::Filter::Modified | 已弃用 [RN611] |
| 行为 | QRandomGenerator | x86 上不再直接使用硬件随机指令，改用 OS 提供的 CSPRNG [RN611] |
| 行为 | qHash(long double) | 不再哈希填充字节（修复非确定性结果）[RN611] |
| 行为 | QLocale C locale 短格式 | 短日期格式改为两位月份、短时间格式去掉秒 [RN611] |
| 行为 | qiterable.h | 不再包含 qmetacontainer.h（传递包含）[RN611] |

**6.12**
| 类型 | API | 说明 |
|---|---|---|
| 弃用 | Q_FLAGS / Q_ENUMS 宏 | 使用将产生弃用警告（Clang/MSVC），改用 Q_FLAG/Q_ENUM [RN612] |
| 弃用 | QTextStream::operator>>(char\*) | 安全原因弃用，改用 operator>>(QByteArray&) [RN612] |
| 行为 | QDom 默认 InvalidDataPolicy | AcceptInvalidChars → ReturnNullNode，非法 XML 字符返回空节点 [RN612] |
| 行为 | 日期/时间解析格式串 | 单引号内反斜杠不再转义单引号，解析与序列化规则统一 [RN612] |
| 行为 | QRandomGenerator | 回退到非安全 PRNG 时输出警告；新增 FEATURE_randomgenerator_disable_fallback [RN612] |
| 行为 | QStandardPaths（Apple 平台） | iOS/macOS 多处路径调整（Templates、Pictures 去重、PublicShare、/Library/Preferences 纳入）[RN612] |
| 行为 | QDesktopUnixServices（Linux/BSD） | 打开 URL/文档优先走 XDG desktop portal 而非 xdg-open [RN612] |
| 安全行为 | QByteArray::fromPercentEncoding / QXmlStreamReader | 修复未初始化内存泄漏；实体展开限制收紧、拒绝对 limit≤0 [RN612] |

## 二、Qt GUI

**6.9**
| 类型 | API | 说明 |
|---|---|---|
| 弃用 | Qt::MacWindowToolBarButtonHint | 自 Qt 5 起即空操作，正式弃用 [RN69] |
| 行为 | PNG 读取器 | 移除请求缩放读取时的解码期渐进降采样 [RN69] |
| 行为 | QIcon::pixmap() | 不再按 devicePixelRatio 缩放传给 QIconEngine::scaledPixmap() 的尺寸 [RN69] |
| 行为 | Windows 平台 | 移除 legacy 鼠标处理，`-nowmpointer` 失效（始终 Windows Pointer）[RN69] |
| 行为 | QRhi (D3D11/12) | 窗口更新请求改由专用 vblank 等待线程驱动 [RN69] |

**6.10**
| 类型 | API | 说明 |
|---|---|---|
| 不兼容/行为 | QPainterPath | 移除隐式共享（为缓存优化），换来完全可重入；依赖写时复制的代码需审查 [RN610] |
| 不兼容 | qmatrix4x4.h | 不再包含 qquaternion.h（传递包含）[RN610] |
| 不兼容 | QWindowSystemInterface::handleThemeChange | 移除可选 QWindow 参数（私有/平台头）[RN610] |
| 行为 | macOS Metal/Raster 窗口 | 内容改绘制到 NSView 根层 的子 CALayer（戳 NSView 的代码可能受影响）[RN610] |

**6.11**
| 类型 | API | 说明 |
|---|---|---|
| 弃用 | QColormap | 类整体弃用 [RN611] |
| 移除 | XCB 原生 X11 绘制引擎 (XRender) | `-xcb-native-painting`/`QT_XCB_NATIVE_PAINTING` 实验特性移除 [RN611] |
| 行为 | macOS Tahoe | 窗口模态消息框不再使用原生 NSAlert（按钮无响应 bug）[RN611] |

**6.12**
| 类型 | API | 说明 |
|---|---|---|
| 不兼容 | QAccessible\*Interface | 禁止拷贝 [RN612] |
| 行为 | QScreen::grabWindow (macOS) | 改用 ScreenCaptureKit（弃用的 CGDisplayCreateImageForRect），首次截屏仍弹权限对话框 [RN612] |
| 不兼容 | QCocoaNativeInterface::setEmbeddedInForeignView | 不再支持/需要 [RN612] |

## 三、Qt Core5Compat

- **6.9**：无 API 变化（仅行为修复：QTextCodec 恢复与 Qt5 一致的 UTF BOM 写出；QStringRef→QStringView 转换保留 null 性）[RN69]
- **6.10**：无（条目均为 6.9 修复的重复收录与构建元数据）[RN610]
- **6.11**：无（仅 QLinkedList 共享 range-erase 别名 bug 修复）[RN611]
- **6.12**：**弃用** QBinaryJson::toRawData()（立即弃用；返回内存需手动释放，与 Qt5 所有权模型冲突，改用 toBinaryData()）[RN612]

## 四、构建工具链要求变化（6.8 → 6.12）

- **CMake 最低版本**：6.8 基线 3.16+；**6.9 起**构建 Qt 与使用 Qt 的 CMake 项目均要求 **≥3.22** [WN69]；**6.12 起**构建和使用均提升至 **≥3.25** [WN612 / RN612 commit db9a9f50]。6.10/6.11 无提升。
- **C++ 标准**：无变化，Qt 6 全系列（含 6.12）仍为 **C++17**；6.9–6.12 的 whatsnew/release note 中均无标准提升条目（Qt 内部以 q20/q23 前缀垫片渐进采用新特性）。
- **编译器最低要求**（supported-platforms 矩阵 6.9 → 6.12）：
  - Linux：测试基线 RHEL 8.x/GCC 10(toolset) → **RHEL 9.6/GCC 11**（Debian 11/GCC 10 仍在列）；官方预编译二进制 **6.10 起改用 RHEL9 构建，glibc 最低 2.28 → 2.34** [WN610]
  - Windows：MSVC 2022 不变；MinGW-w64 13.1 → **15.1**；ARM64EC 实验支持需 MSVC 2026；**6.12 是支持 Windows 10 的最后版本** [SP612]
  - macOS：Xcode 15 → **16**（macOS 15 SDK）；运行最低 macOS 12 → **14.4**（含 macOS 26）[SP612]
- **其他构建变化**：6.9 起使用私有模块需显式 `find_package(Qt6 COMPONENTS FooPrivate)`（可用 QT_FIND_PRIVATE_MODULES=ON 暂时恢复）；6.10 移除 qt6_generate_deploy_script 的 FILENAME_VARIABLE（6.6 已弃用）；6.12 新增 static_compiler_runtime 特性（MSVC 静态运行时）。

## 来源 URL

- RN6x: `https://github.com/qt/qtreleasenotes/blob/dev/qt/6.9.0/release-note.md`（6.10.0/6.11.0/6.12.0 同理）
- WN6x: `https://doc.qt.io/qt-6/whatsnew69.html` / `whatsnew610.html` / `whatsnew611.html` / `whatsnew612.html`
- SP: `https://doc.qt.io/qt-6/supported-platforms.html`（6.9 对照：`https://doc.qt.io/qt-6.9/supported-platforms.html`）

**升级要点结论**：源代码层面最大风险点是 6.9 的 moc 元对象布局（Q_REVISION）与 QChar/QString::arg 隐式转换收紧、6.10 的 QPainterPath 去隐式共享、6.12 的 Q_FLAGS/Q_ENUMS 与 QTextStream op>>(char\*) 弃用；构建侧需先把 CMake 升到 3.25+，Linux 部署环境需 glibc ≥2.34（若用官方预编译包）。
