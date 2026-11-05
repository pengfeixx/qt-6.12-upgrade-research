# 中间过程文档索引

本目录保存 Qt 6.8 → 6.12 升级调研的全部中间产物，供后期查证溯源。
最终结论见上级目录《Qt6.8到6.12升级总体方案与实施计划.md》（总文档），及其支撑报告《阶段1-Qt6.8到6.12接口变化与包名变化分析.md》《阶段2-受影响应用与场景分析.md》。

## 阶段一（接口与包名调研）

| 文件 | 内容 | 支撑主报告章节 |
|---|---|---|
| 00-本地环境扫描-2026-10-08.md | 本机 Qt 6.8 安装基线：120 个 qt6 包完整清单、全部 Qt6 CMake 模块列表、工具链版本（CMake/GCC/glibc/GStreamer 等） | 阶段1 §1 环境基线、§3 工具链 |
| 01-调研一-QtCore-GUI-Core5Compat及工具链.md | Qt Core/GUI/Core5Compat 在 6.9–6.12 的不兼容、行为、弃用条目；CMake/C++/编译器/glibc 要求变化 | 阶段1 §3、§4.1–4.3 |
| 02-调研二-QtQml-Quick-Controls-Layouts.md | Qt Qml/Quick/Quick Controls/Layouts 接口与行为变化（含私有头文件变化、qmllint 收紧） | 阶段1 §4.4–4.5 |
| 03-调研三-Widgets-Network-Multimedia-SVG-Wayland.md | Widgets/Network/Multimedia/SVG/Wayland(client+compositor) 变化 | 阶段1 §4.6–4.9 |
| 04-调研四-WebEngine-模块增减-平台支持.md | WebEngine Chromium 版本与独立发布（6.140）、qtpdf 分离、其余小模块变化、模块增减与平台支持 | 阶段1 §4.10–4.11、§5 |
| 05-调研五-Debian系包名变化.md | Debian/Ubuntu/deepin 打包现状、t64、源码包增减、上游发布物命名 | 阶段1 §6 |

## 阶段二（受影响应用与场景，反向依赖查询）

| 文件 | 内容 | 支撑主报告章节 |
|---|---|---|
| 06-阶段二-本机Qt6反向依赖清单.tsv | 本机 295 个包（106 Qt 自身 + 189 消费者）× 所依赖 Qt6 运行库的 TSV 明细，可 grep/透视 | 阶段2 §2.1 |
| 07-阶段二-仓库级反向依赖统计.txt | 软件源索引（36,972 包）内 11 个关键 Qt6 库的 apt-cache rdepends 全量名单 | 阶段2 §2.2 |
| 08-阶段二-私有ABI依赖包仓库扫描.txt | 197 个 Depends 含 qt6-*-private-abi 的包及完整 Depends 原文（`qt6-base-private-abi (= 6.8.0)` 精确锁版本证据） | 阶段2 §3 |
| 09-阶段二-关键模块消费者分类清单.txt | WebEngine(9 应用)/Multimedia/Wayland 合成器/QML 消费者分类名单 | 阶段2 §4 |
| 10-阶段二补充-linuxdeepin组织仓库扫描/ | GitHub linuxdeepin 组织（265 仓库）源码级扫描：gh 代码搜索 API 原始返回、82 个 Qt6 仓库/35 个私有头仓库名单、27 个本机未装的新增应用、扫描汇总.md（含复现命令与踩坑记录） | 阶段2 §2.3 |
| 11-阶段二-整合统计/ | 本机(189) × linuxdeepin(82) 按包名映射去重的统一视图：整合应用清单.tsv（216 实体 × 10 列：分类/来源/私有API/QML/WebEngine/Multimedia/Wayland私有头/Network/Widgets）+ 统计结果.md（分布与风险名单） | 阶段2 §2.4 |
| 12-阶段二-玲珑应用分析/ | 玲珑渠道：工具链版本、已装应用与运行时 layer、Qt 版本盘点（runtime.dtk/webengine 均为 Qt 6.8.0、Chromium 122）、DDE 玲珑交付清单、26 个 linglong.yaml 仓库与示例 manifest | 阶段2 §6 |
| 13-WebEngine独立发布核实-2026-10-09.md | WebEngine "6.12" 发布状态核实：不存在 6.12 版；独立产品 6.140.0 截至 2026-10-09 仅 RC（code.qt.io tag 实测）；Rust 工具链预警；复现命令 | 总体方案 §1/R3/W-0、阶段1 §4.10 |
| 14-依赖对比6.8vs6.12-2026-10-09.md | 6.12 vs 6.8 依赖实测对比：真实新增仅 liburing（+PipeWire/VA）；GStreamer 后端与 MNG 插件两个构建缺口；构建依赖新增汇总（CMake 3.25/assimp 6/libmng 2 等） | 总体方案 W-0 |
| 15-deepin-community组织6.12分支核查-2026-10-21.md | GitHub deepin-community 组织 Qt 仓库 6.12 分支核查：34/38 有 `upstream/qt6.12` 占位分支但**内容全部仍为 6.8.0**（graphs 为空模板）；canvaspainter/tasktree/quickeffectmaker 仓库不存在、remoteobjects 无分支；净室产物可反向灌入承接 | 总体方案 §2.2/W-1.3、阶段1 §6.3 补充 |

## 一手实测资料（净室打包）

| 文件 | 内容 | 支撑 |
|---|---|---|
| ../qt6.12-build/docs/Qt6.12净室打包记录.md | **Qt 6.12.0 净室打包全程实录**：debootstrap 净室方法、38 模块打包范围决定（webengine/openapi/activeqt/doc 缺席原因）、A1–C9 共 20+ 个问题与解决（t64 陷阱、HarmonyOS 工具硬依赖、QML 插件 cmake 目标更名、assimp/libmng 补建等）、467 个 deb 产物验证、5 项发布前遗留门槛 | 总体方案 W-0、§1/§5、阶段1 §4.10 实测修正 |
| ../qt6.12-build/docs/产物清单.md | 467 个 deb 明细 | 同上 |

## 溯源说明

- 阶段一各调研文档为独立调研代理的原始返回内容，逐条标注了来源 URL（qt.io 官方博客、doc.qt.io whatsnew/release notes、wiki.qt.io、tracker.debian.org、launchpad 等）。
- 阶段二数据为本机/软件源实测命令输出，复现命令记录在阶段2报告附录（dpkg-query Depends 扫描、apt-cache rdepends、apt 索引原文扫描 private-abi 虚拟包）。
- 主报告中的风险评级与矩阵为汇总时的二次加工，原始条目可在对应中间文档中核对。
- 注意：Qt 官方已不再维护按次版本分节的 per-module "Changes in Qt 6.x" 网页（均 404 或仅含 Qt5→Qt6 迁移内容），6.9–6.12 变更的权威来源是 qtreleasenotes 仓库各版本 release-note.md + whatsnew 页面。
- 阶段二口径限制：私有 API 的使用面以二进制包运行时 Depends 为准；源码仓库中 Build-Depends qt6-*-private-dev 的包更广，需在阶段三查源码仓库补充。
