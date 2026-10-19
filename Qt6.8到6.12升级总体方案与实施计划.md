# Qt 6.8 → 6.12 升级总体方案与实施计划

| 项目 | 内容 |
|---|---|
| 文档日期 | 2026-10-09 |
| 当前状态 | **Qt 6.12 净室打包完成：38 模块 / 467 个 deb / 约 1.9GB**（M0 里程碑达成，实录见 [Qt6.12净室打包记录](qt6.12-build/docs/Qt6.12净室打包记录.md)） |
| 基线 → 目标 | Qt 6.8.0（deepin 打包）→ Qt 6.12.0 LTS（2026-09-30 发布，5 年维护期） |
| 依据文档 | [阶段1-接口变化与包名变化分析](阶段1-Qt6.8到6.12接口变化与包名变化分析.md)、[阶段2-受影响应用与场景分析](阶段2-受影响应用与场景分析.md)、[中间过程数据](中间过程/README.md)（00～12 共 13 组原始数据） |

---

## 1. 执行摘要

- **接口层面可行性高**：Qt 6 系列次版本间保持源/二进制兼容，6.9–6.12 无公有 API 成批删除；不兼容点少且多为编译期可发现（预计以机械修复为主），真正工作量在行为回归测试与 QML 存量扫描。
- **包名层面零变化**：Debian 系命名规律（`qt6-*-dev`/`libqt6*6`/`qml6-module-*`）在 6.9–6.12 完全稳定，Qt 6.12 打包已顺利完成即为实证。
- **WebEngine 现实约束（已核实）**：不存在"WebEngine 6.12"——自 6.12 起独立成产品，首个版本 **6.140.0**（跟随 Chromium 版本号，含 qtpdf 与 qtwebview 后端）。截至 2026-10-09 正式版未发布（code.qt.io 仅有 `v6.140.0-rc`），qt6-webview 已打成"无 webengine 后端"形态。**独立线落地前，uos-ai、uos-service-support、deepin-deepinid-client 三个 deb 侧应用没有 Qt 6.12 路径**；好消息是 RC 源码已可得，可先行试打包（详见 [中间过程/13](中间过程/13-WebEngine独立发布核实-2026-10-09.md) 与 R3）。
- **真正的工程约束是私有 ABI 原子锁**：`libqt6core6` 以 `Provides: qt6-base-private-abi (= 6.8.0)` 与 **61 个组件**（整合口径，二进制 68 包 ∪ 源码 35 仓库）精确锁版。Qt 已打包 ≠ 可以发布：**这 61 个组件不同批重建改锁，Qt 6.12 无法进入正式仓库**。升级必然是"库→框架→桌面→应用"的原子切换。
- **影响面**：系统 deb 渠道 216 个受影响组件（含 20 个"私有 API ∩ QML"风暴眼）；玲珑渠道另有 14+ 应用独立链路（自带 Qt 6.8，节奏解耦，可作先行验证通道）。
- **建议总排期**：约 4.5 个月（2026-10 ～ 2027-02），5 个里程碑；估算总量约 **260～300 人日**（桌面侧为主，QA 与跨团队另计）。

---

## 2. 前期调研结论汇总

### 2.1 接口变化（详表见阶段1报告 §4，此处为决策要点）

| 风险等级 | 变更 | 版本 | 对本升级的含义 |
|---|---|---|---|
| 🔴 编译期 | moc 元对象布局（Q_REVISION≥13）、QChar/QString::arg 隐式转换收紧、头文件传递包含移除 | 6.9 | 全量重编时集中暴露，机械修复，工作量可提前用小样本量化 |
| 🔴 QML 引擎 | **缺失 required properties 的组件直接报错** | 6.12 | 所有 QML 应用需先跑 qmllint 全量扫描，这是最可能的"隐形阻断点" |
| 🔴 行为 | TableModel 移除复杂行支持、setData() 参数顺序对调 | 6.10 | 用到 TableModel 的应用（文管/控制中心类）需代码适配 |
| 🔴 合成器 | QWaylandShellSurface::setContentGeometry()、linux-dmabuf v4、wl_surface 不随 hide 销毁 | 6.9/6.12 | dde-shell、treeland 必改项 |
| 🔴 结构 | **qtwebengine/qtpdf 移出 Qt 6.12 发布包**，独立产品 WebEngine 6.140+ | 6.12 | 打包与维护策略需单独立项决策（见 §5.1-D） |
| 🟡 行为回归 | TextField 默认右键菜单（6.9）、高对比度自动应用（6.10）、Text 下划线渲染/MenuItem 快捷键显示（6.12）、QNAM 空闲连接 2 分钟关闭 + POST 不重发（6.10/6.11）、TimeoutError 新错误码（6.11）、XDG portal 优先打开 URL（6.12）、GStreamer≥1.20（6.9，本机 1.24 已满足） | 6.9–6.12 | 回归测试清单化，无需改代码 |
| 🟢 免费收益 | Canvas Painter/StyleKit/新图表、QML 热重载、qmlsc 编译范围扩大、Charts→Graphs 迁移提示 | 6.11/6.12 | 可作为升级的正向产出宣传 |

**工具链门槛（全部已满足）**：CMake ≥3.25（本机 3.31）、C++17 不变（GCC 12.3 ✓）、glibc ≥2.34（2.38 ✓）、GStreamer ≥1.20（1.24 ✓）；WebEngine 独立线另需 C++20、Node.js ≥20。

### 2.2 包名与打包变化（阶段1报告 §6）

- 二进制/源码包命名规律 6.9–6.12 **零变化**；无 t64 类改名事件（t64 决策仍建议随本次升级一并定案）。
- 新增源码包：`qt6-tasktree`、`qt6-canvaspainter`（Debian sid 已有先例）、`qt6-openapi`（需自打）、StyleKit 随 declarative 分发。
- WebEngine/PDF 拆分为独立产品线（6.140+），打包来源与安全更新节奏重新设计。
- 无现成 Debian/Ubuntu 6.12 打包可抄（Debian sid 停在 6.11.2），**我们的 Qt 6.12 打包已成为该路线的先行实践**。

### 2.3 影响的应用与场景（阶段2报告）

| 维度 | 数据 |
|---|---|
| 系统 deb 渠道 | 216 个受影响组件 = 两者(本机+linuxdeepin) 55 + 仅本机 134 + 仅源码 27 |
| 原子重建核心集 | **61 个私有 API 组件**（DDE 核心 12+ / 窗口管理栈 / DTK 全家 / deepin 应用 14+ / KF6·输入法·开发工具约 20） |
| 风暴眼（私有API∩QML） | **20 个**：kwin×3、treeland×2、dde-shell、dde-launchpad、dde-clipboard、dde-control-center、dde-file-manager、libdtk6declarative、uos-ai 等 |
| 高风险场景 6 类 | QML 引擎行为 / 私有 ABI 重建 / Widgets·样式视觉 / WebEngine 独立发布 / Wayland 合成器栈 / 网络与多媒体行为 |
| 玲珑渠道 | 18 应用官方清单（本机已装 14），runtime layer 自带 Qt 6.8.0/Chromium 122，与系统解耦 |
| 外部不可控项 | KF6/KDE 与第三方上游共 **65 个组件**（kwin 上游线、fcitx5、python3-pyqt6、qtcreator、poppler-qt6 等）跟随上游发版，deepin 只能打包跟随 |

---

## 3. 升级具体工作分解（WBS）与工作量

> 里程碑 P0 **已完成**：Qt 6.12 净室打包（详见下节 W-0）。以下为剩余工作。
> 工作量为**估算值**（假设：每包重建+冒烟 0.5～1 人日；有代码适配的包按要点估算；存量 QML 修复量以 P1 扫描结果校准）。

### W-0 已完成：Qt 6.12 净室打包实录要点（[完整记录](qt6.12-build/docs/Qt6.12净室打包记录.md)）

**方法与产物**：宿主 UOS V25 上 debootstrap 净室（snipe suite）+ 复用 Debian qt-kde-team 的 6.11 打包目录 + download.qt.io 官方 tarball（清华 TUNA 镜像断点续传）。**38/38 模块打包成功，467 个 deb（约 1.9GB）**；净室内已装 150 个 6.12.0-1 包、依赖闭环通过、`qmake6 --version` 报 6.12.0、**qmllint 等 QML 工具链可用（W-1.2 扫描可立即启动）**。

**打包范围缺席项**（后续工作项来源）：qtwebengine（**独立产品 6.140.0，正式版在途、rc 已可得**，见 [中间过程/13](中间过程/13-WebEngine独立发布核实-2026-10-09.md)）、qtopenapi（Debian 无先例，建议仿 qt6-grpc 新写最小 debian 目录）、qtactiveqt（Windows 专属，永久排除）、qt6-*-doc 文档包（nodoc 构建，如需可做第二遍全量文档构建）。

**上游兼容性一手结论**：6.11 → 6.12 源码兼容性非常好——Debian 补丁面几乎无改动（38 模块中 30 个无补丁，其余除 2 个已上游合入的补丁剔除外全部干净应用），唯一失败的 cross2.patch 是零 fuzz 应用问题而非内容冲突。**这从实践上印证了阶段一"接口可行性高"的判断。**

**发布到正式源前的门槛**（打包记录遗留问题清单 + 依赖对比新增，对应 §6 基础软件团队；实测对比见 [中间过程/14](中间过程/14-依赖对比6.8vs6.12-2026-10-09.md)）：
1. **两个构建缺口修复（功能回退，必修）**：① qt6-multimedia 的 **GStreamer 后端未编入**（configure 时缺 gstreamer Play 组件，当前仅有 FFmpeg/PipeWire/VA 后端）；② qt6-imageformats 的 **MNG 插件未编入**（libmng 2.0.3 已补建但插件缺失）——排查净室环境后重编这两个模块即可；
2. **t64 全量排查**：Debian control 硬编码 t64 包名（首例 `libpipewire-0.3-0t64` → UOS `libpipewire-0.3-0`），467 个 deb 入库前必须全量审查；
3. 符号表策略：本轮删除 6.11 `.symbols` 由 dpkg-gensymbols 重生成（符号级→shlibs 级），正式入库前建议按 Debian 流程恢复 pkgkde-symbolshelper 增量维护；
4. LTO 已关闭（控时长），正式发布构建建议恢复 `optimize=+lto`；
5. 补建上游依赖 assimp 6.0.5、libmng 2.0.3 的版本策略对齐（加 `~uos1` 类后缀）；新增依赖实测很少且已满足：**liburing-dev/liburing2**（core，上游真实新增）、libpipewire-0.3-dev、libopenxr-dev、libjemalloc-dev、14 个 libxcb 扩展 -dev、clang-21（净室构建成功即证明 UOS 源可满足）；版本门槛提升项：cmake ≥3.25、assimp ≥6.0、libmng ≥2、GStreamer ≥1.20（逐模块精确 diff 见[中间过程/14](中间过程/14-依赖对比6.8vs6.12-2026-10-09.md) §4）；
6. 打包新知识已固化：6.12 的 HarmonyOS 部署工具是 Qt6 CMake 包的硬依赖必须收录；QML 插件 CMake 目标统一加 `plugin` 后缀；Lottie 模块 Generator→Helpers 更名重排包结构。

### W-1 决策与基线（P1 批次，约 10 人日）

| # | 工作项 | 说明 | 工作量 |
|---|---|---|---|
| 1.1 | Qt 6.12 包入库 testing/staging 仓 | 467 个 deb 进独立仓（不进正式源，被 `=6.8.0` 锁挡住也进不去）；**入库前先过 W-0 的发布门槛（t64 全量排查等 5 项）** | 2 人日 |
| 1.2 | **qmllint 基线扫描** | 用 6.12 工具链对 45 个 QML 组件存量 .qml 全量扫描，输出 required properties 等硬错误清单（**净室内 qmllint 6.12 已就绪，可立即启动**） | 3 人日 |
| 1.3 | **四大决策定案** | ① WebEngine 独立线：渠道已核实（[中间过程/13](中间过程/13-WebEngine独立发布核实-2026-10-09.md)），定"等正式 tag vs 用 6.140.0-rc 先行打包"与分批迁移策略；② kwin 走 deepin-kwin fork 还是 KDE 上游跟版；③ t64 后缀是否随升级定案（W-0 的 libpipewire 实例表明该问题无法回避）；④ 玲珑/系统双轨版本对齐策略 | 3 人日（会议+论证） |
| 1.4 | 保留 6.8 全套包快照 | 回退预案硬前提（场景 E：降级同样原子） | 1 人日 |
| 1.5 | 编译期错误小样本实证 | 取 DTK6 + dde-shell + treeland 在 6.12 下试编，量化 §2.1 编译期风险的真实修复成本 | 2 人日 |

### W-2 底层库与框架层（P2 批次，约 55 人日）

| # | 工作项（分应用） | 适配要点 | 工作量 |
|---|---|---|---|
| 2.1 | **DTK6 全家**（dtkcore/log/gui/widget/declarative/multimedia/systemsettings + dtkcommon） | 私有头同步、6.9 隐式转换/传递包含编译修复、QStyle 高对比度语义（6.10）、libdtk6declarative 受 QML 变化影响 | 15 人日 |
| 2.2 | **kwin×3 + libkwineffects** | 按决策走 fork/上游；私有头同步、QML 特效（required properties）、wl_surface 生命周期回归 | 10 人日 |
| 2.3 | **treeland + wallpaper-factory** | 私有 ABI + waylandclient 私有头、setContentGeometry、dmabuf v4、QuickControls2 行为、HTTP Server 并发默认值（6.12） | 15 人日 |
| 2.4 | dde-qt6integration + dde-qt6xcb-plugin + qt 平台主题插件 | QPA/平台私有 ABI 最深区域，重点验证 | 5 人日 |
| 2.5 | fcitx5-frontend-qt6（+fcitx-frontend-qt6） | 输入上下文插件私有 ABI | 2 人日 |
| 2.6 | 自研绑定库（gio-qt6、gsettings-qt6、udisks2-qt6、mpris、qapt、image-editor、util-dfm/dfm6 系、pdfium、rofd、dtkai） | 私有头重建 + 冒烟 | 8 人日 |

### W-3 桌面框架与核心应用（P3 批次，约 60 人日）

| # | 工作项（分应用） | 适配要点 | 工作量 |
|---|---|---|---|
| 3.1 | **dde-shell 全家**（libdde-shell、dock、tray-loader、clipboard、network-core、dde-services） | 风暴眼：compositor API + QML + 私有 ABI + wayland 私有头，全要害集合 | 15 人日 |
| 3.2 | dde-launchpad / dde-desktop / dde-session(-shell/-ui) / ddm | QML required properties、会话链路回归 | 8 人日 |
| 3.3 | **dde-control-center** + dcc 插件群（network/passkey/insider） | QML + 多媒体（相机/声音）+ 私有 ABI | 10 人日 |
| 3.4 | **dde-file-manager** + 服务插件 + dde-device-formatter | 私有 ABI + QML + MM 预览 + TableModel 排查 | 12 人日 |
| 3.5 | dde-grand-search、dde-application-manager、dde-polkit-agent、xdg-desktop-portal-dde、dde-calendar/dialog/app-services 等 | 私有头重建 + QML 修复 | 10 人日 |
| 3.6 | deepin-terminal（+qtermwidget）、deepin-screensaver、deepin-shortcut-viewer、deepin-im | 私有 ABI + QML | 5 人日 |

### W-4 应用层与专项（P4 批次，约 60 人日，与 P3 部分并行）

| # | 工作项（分应用） | 适配要点 | 工作量 |
|---|---|---|---|
| 4.1 | **网络行为回归组**：deepin-app-store、org.deepin.downloader、deepin-deb-installer、deepin-update-manager/update-ui/offline-update、dde-cooperation、uos-remote-assistance | QNAM 行为三连（POST 不重发/空闲关闭/TimeoutError）、portal 打开 URL | 10 人日 |
| 4.2 | **多媒体组**：deepin-screen-recorder、deepin-camera(玲珑)、deepin-face、deepin-music/movie-reborn、deepin-voice-note、dde-wirelesscasting | GStreamer 1.20 验证、PipeWire 采集、VideoSink、缩略图 | 10 人日 |
| 4.3 | **WebEngine 组**（专项）：uos-ai、uos-service-support、deepin-deepinid-client + 玲珑侧 deepin-mail/manual/reader/editor | WebEngine 6.140 独立线适配：权限 API（6.9 模型）、PPAPI 排雷、Chromium 122→140 行为差异 | 12 人日 |
| 4.4 | 其余私有 ABI 包纯重建（boot-maker、devicemanager、diskmanager、desktop-theme、picker 等 ~20 个） | 重建+改锁版本+冒烟，0.5～1 人日/包 | 15 人日 |
| 4.5 | 存量 QML 修复（依据 1.2 扫描清单） | required properties 等硬错误 | 预留 13 人日（以扫描结果校准） |

### W-5 玲珑渠道（P4 并行，约 15 人日）

| # | 工作项 | 要点 | 工作量 |
|---|---|---|---|
| 5.1 | 玲珑工具链重建 | linglong-bin/builder/box/installer 在 68 包集中，原子切换必选 | 3 人日 |
| 5.2 | **runtime.dtk（Qt 6.12 内核）layer 制作与灰度** | 金丝雀通道：先于 deb 栈验证 | 4 人日 |
| 5.3 | runtime.webengine 层升级规划 | Chromium 122 安全压力，与 4.3 决策联动 | 3 人日 |
| 5.4 | 14 个玲珑应用重构建（源码修复与 deb 侧复用） | 双渠道一次适配 | 5 人日 |

### W-6 测试、集成与发布（P5–P6，QA 团队为主，约 50 人日，另见 §7 协作）

| # | 工作项 | 要点 |
|---|---|---|
| 6.1 | 场景化回归测试 | 按阶段2 §4 六类场景清单执行（QML/视觉/网络/多媒体/合成器/沙箱内行为） |
| 6.2 | 性能基线对比 | 6.8 vs 6.12（启动、渲染、内存；Quick 场景图优化应有正收益） |
| 6.3 | 第三方应用兼容性抽测 | `>=` 约束的存量 deb 应用直接跑 6.12 环境 |
| 6.4 | 原子切换演练 + 灰度发布 + 回退演练 | staging → 灰度 → GA；回退用 1.4 快照 |

**工作量汇总（桌面侧估算）**：W-1～W-5 约 200 人日 + W-6 QA 50 人日 ≈ **250 人日**；预留 10% 缓冲 → **约 260～300 人日**。若 W-1.5 实证发现编译修复量异常或 QML 扫描爆量，按 1.2/1.5 的实测数据修正。

---

## 4. 分应用计划排期（建议稿）

> 时间为建议排期，以各团队确认为准；关键路径 = DTK6 → kwin/treeland → dde-shell → 核心应用 → 集成。

### 里程碑

| 里程碑 | 时间 | 判据 |
|---|---|---|
| M0 ✅ | 已完成 | Qt 6.12 全模块打包成功 |
| M1 决策冻结 | 2026-10-30 | 四大决策定案；qmllint 扫描清单出报告；6.8 快照就位 |
| M2 桌面可登录 | 2026-11-27 | DTK6/kwin/treeland/dde-shell 在 6.12 staging 下桌面完整启动 |
| M3 核心应用可用 | 2026-12-31 | 控制中心/文管/启动器/终端/商店等核心应用功能可用 |
| M4 回归通过 | 2027-01-29 | 六类场景回归+性能基线+第三方抽测通过；61 包全部改锁完成 |
| M5 灰度/发布 | 2027-02-26 | 原子合入正式源，灰度→GA，回退预案演练通过 |

### 分应用排期表

| 批次 | 时间窗 | 应用/组件（关键项加粗） | 适配动作 |
|---|---|---|---|
| P1 决策与基线 | 10-13～10-24 | Qt 6.12 包入库 staging；**DTK6/dde-shell/treeland 试编译样本**；全量 qmllint 扫描 | 决策、扫描、快照 |
| P2 底层库 | 10-27～11-21 | **DTK6 全家**（先行，W1 内启动）→ **kwin×3**、**treeland** → dde-qt6integration/xcb-plugin → fcitx5-frontend → gio-qt6/gsettings/udisks2/mpris/qapt/image-editor/util-dfm/rofd/dtkai | 代码适配+重建改锁+冒烟 |
| P3 桌面框架与核心应用 | 11-24～12-19 | **dde-shell 全家** → dde-launchpad/desktop/session/ddm → **dde-control-center**+插件 → **dde-file-manager** → grand-search/application-manager/polkit-agent/portal-dde/calendar/dialog → terminal/screensaver/shortcut-viewer/deepin-im | QML 修复+私有头重建+场景自测 |
| P4a 应用与专项 | 12-01～12-31（并行） | 商店/下载器/deb 安装器/更新器组（网络回归）；screen-recorder/face/voice-note/wirelesscasting（多媒体）；boot-maker/devicemanager/diskmanager 等 ~20 包纯重建 | 行为回归+重建 |
| P4b WebEngine 专项 | 12-01～12-31（并行） | **6.140.0 独立线打包（可先用 v6.140.0-rc 起步，正式 tag 后切换）**；uos-ai、uos-service-support、deepin-deepinid-client（deb 侧）；qt6-webview 补 webengine 后端 | 权限模型适配+PPAPI 排雷+Chromium 122→140 差异回归 |
| P4c 玲珑金丝雀 | 11-24～12-31（并行） | linglong 工具链重建 → runtime.dtk(Qt 6.12) layer 灰度 → 14 个玲珑应用重构建 | 金丝雀验证，结论反哺 P2/P3 |
| P5 系统回归 | 2027-01-04～01-29 | 全部 61+216 组件集成；六类场景回归；性能基线；第三方应用抽测 | 测试主导 |
| P6 发布 | 2027-02-01～02-26 | 原子合入、灰度、GA、回退演练 | 发布管理 |

---

## 5. 关键风险与预案

| # | 风险 | 概率/影响 | 预案 |
|---|---|---|---|
| R1 | QML required properties 硬错误存量超预期（1.2 扫描结果爆量） | 中/高 | P1 提前扫描；风暴眼 20 组件优先；玲珑金丝雀提前暴露 |
| R2 | 私有头 API 在 6.9–6.12 有未记录变更导致 DTK/kwin/treeland 适配超期 | 中/高 | W-1.5 试编译实证；关注 6.9 起 `find_package(FooPrivate)` 显式化（可临时 `QT_FIND_PRIVATE_MODULES=ON` 过渡） |
| R3 | WebEngine 独立线 6.140.0：**正式 tag 未出（截至 2026-10-09 仅有 rc）**，正式发布时点与下载通道待定；期间 3 个 deb 侧 WebEngine 应用无 6.12 路径 | 低/高（渠道已明确，只差时间） | 盯 code.qt.io `v6.140.0` tag；**可立即用 rc 源码复用 6.11 打包目录试打包**（官方确认可对 Qt 6.12 构建，构建时 Qt 版本=最低运行版本）；若正式版延期：GA 等待 / 三应用暂缓 / 临时玲珑化三选一；年内后续版本将要求 Rust 工具链，打包基础设施提前准备 |
| R4 | KF6/kwin 上游未及时发 6.12 兼容版 | 中/高 | P1 就与上游对齐版本计划；kwin 必要时走 deepin-kwin fork 自适配 |
| R5 | 原子切换期间依赖断裂（用户升级窗口故障） | 低/极高 | staging 全量演练 + 灰度 + 6.8 快照回退预案（场景 E） |
| R6 | 双轨共存期（系统 6.12 + 玲珑 6.8）行为不一致 | 高/中 | 明确共存窗口上限；跨沙箱交互（剪贴板/输入法/文件对话框）专项回归 |

---

## 6. 除桌面应用外，其他团队需要做的事项

| 团队 | 事项 | 关联批次 |
|---|---|---|
| **基础软件/软件源团队** | ① **467 个 Qt 6.12 deb 的入库门槛处理（W-0 遗留 5 项）**：t64 全量排查（首例 libpipewire 已撞上）、.symbols 符号表恢复、LTO 恢复、assimp/libmng 版本策略、qt6-webview 后端补齐；② 新源码包打包：qt6-openapi（建议仿 qt6-grpc 写最小 debian 目录）、qt6-tasktree/canvaspainter 版本跟进；③ **WebEngine/PDF 独立产品线（6.140.0）打包**：rc 已可先行，正式 tag 后切换；**年内后续版本要求 Rust 工具链与更高编译器，构建基础设施提前准备**；④ t64 命名决策落地（若定案，涉及全量包依赖重写）；⑤ staging/灰度仓基础设施与 6.8 快照仓；⑥ 确认 GStreamer≥1.20、Node.js≥20（WebEngine 构建）在目标发行版可用 | P1–P4 |
| **CI/构建基础设施团队** | ① 68 包原子重建的流水线编排（依赖序自动排队）；② WebEngine 构建资源（单次 ~12GB 源码、链接期 16–32GB 内存机器）；③ **qmllint 纳入 CI 门禁**（6.12 规则 + `--only-explicit-categories` 过渡策略）；④ 构建环境 CMake≥3.25、编译器基线核查 | P1–P5 |
| **QA/测试团队** | ① 依据阶段2 §4 六类场景制定回归用例集（QML/视觉/网络/多媒体/合成器/沙箱）；② 性能基线对比（6.8 vs 6.12）；③ 第三方存量应用兼容性抽测（`>=` 约束直接换库运行）；④ 升级/回退路径演练 | P5–P6 |
| **安全团队** | ① WebEngine Chromium 122→140 行为与安全评审、独立线漏洞响应 SLA 评估；② Qt 6.12 的 SBOM/CRA 合规材料引入（上游已提供）；③ 6.9–6.12 安全修复清单纳入发布说明 | P4–P6 |
| **输入法团队（fcitx）** | fcitx5-frontend-qt6 私有 ABI 适配与全输入法回归（含玲珑沙箱内输入） | P2 |
| **应用商店/下载服务团队** | ① 商店本体网络行为回归（POST 不重发/空闲连接关闭对大文件下载的影响）；② 商店内应用（玲珑 runtime 层）的 Qt 版本灰度配合；③ 第三方开发者公告：SDK、兼容性声明、升级指引 | P4–P6 |
| **玲珑（linglong）团队** | ① linglong 工具链（bin/builder/box/installer）随 68 包原子重建；② runtime.dtk Qt 6.12 layer 制作与灰度（金丝雀）；③ runtime.webengine 升级规划（Chromium 122 安全压力）；④ 14+ 玲珑应用重构建排期 | P2–P4 |
| **AI/UOS 服务团队** | uos-ai（WebEngine+QML+MM 三重命中）、uos-service-support、deepin-ai-daemon、dtkai 的适配与验证 | P3–P4 |
| **生态合作/开发者关系团队** | ① 面向第三方应用开发者的迁移公告与技术支持（CMake 3.25、qmllint 收紧、弃用清单）；② OEM/整机伙伴的升级窗口同步；③ python3-pyqt6、qtcreator 等开发工具链的跟版计划确认 | P4–P6 |
| **发布/版本管理团队** | ① 原子切换的变更集编排与窗口选择；② 升级路径与回退预案演练；③ 发布说明（新特性 + 行为变化 + 已知问题） | P6 |
| **文档/培训团队** | 内部适配手册（基于阶段1 §4 逐模块条目）与外部开发者文档更新 | P3–P5 |

---

## 7. 结语

Qt 6.12 净室打包成功后，升级剩余工作的本质是：**以 61 个私有 API 组件为原子集的组织化重建与回归**，叠加 WebEngine 独立化、玲珑双轨两个专项决策。打包实践已给出两个重要的一手信号：其一，6.11→6.12 的 Debian 打包面几乎零改动，印证接口兼容性判断；其二，WebEngine 缺席与 t64 包名陷阱是真实存在的硬问题，均已列入 P1 决策与发布门槛。技术不确定性已在阶段一/二收敛为可枚举的清单；主要不确定性来自排期协调（65 个第三方上游组件的跟版节奏）、存量 QML 修复量（P1 扫描即可量化）与 WebEngine 独立线的可得性。建议按 §4 排期启动，两周内完成 M1 决策冻结。

> 附：全部支撑数据与调研原始记录见 [中间过程/README.md](中间过程/README.md)（阶段一调研 5 份 + 阶段二数据 7 组 + 净室打包实录，共 14 份可查证文档）。
