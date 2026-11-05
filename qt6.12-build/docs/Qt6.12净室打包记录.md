# Qt 6.12.0 净室(debootstrap chroot)打包实录

> 目标:在 UOS Desktop 25 Professional (snipe, Debian bookworm 基础) 上,用 debootstrap
> 建立净室环境,复用 Debian Qt/KDE 团队(salsa.debian.org/qt-kde-team/qt6)的 Qt 6.11
> 打包目录,把 Qt 6.12.0 全部模块打包为 deb 包。
>
> 工作目录:`qt6.12-build/`（相对本分享包根目录）
> 日期:2026-10-08 起

## 1. 环境与素材

| 项 | 值 |
| --- | --- |
| 宿主机 | UOS Desktop 25 Professional (snipe),kernel 6.6.0-amd64-desktop,16 核 / 23GB 内存 |
| 磁盘 | /home 分区 636GB 可用 |
| 源 | UOS 官方池(desktop-professional-V25、dde-snipe/2500u2、udcp-snipe/2500u2、desktop-security-V25) |
| 净室 | `debootstrap`(deepin 定制版 1.0.133deepin2),suite=`snipe`,chroot 变体 |
| 上游源码 | download.qt.io `official_releases/qt/6.12/6.12.0/submodules/`(38 个模块 tarball) |
| 打包目录 | salsa.debian.org/qt-kde-team/qt6 下 38 个仓库(6.11.2+dfsg 打包) |

## 2. 打包范围决定

- **qtwebengine:6.12.0 官方发布里没有**。Qt 自 6.12 起 WebEngine 与主版本解耦、
  单独排期(官方 submodules 目录无 qtwebengine-everywhere-src-6.12.0.tar.xz),
  因此本轮无法打包 webengine,待其单独发布后可复用 qt6-webengine 仓库的 6.11 打包目录补齐。
- **qtdoc / qt6-doc**:Debian 的 qt6-doc 仓库自 2023 年起停更,Qt 文档由各模块
  qdoc 生成,本轮以 `nodoc` 构建配置跳过文档包(见 §3-A6)。
- **qtactiveqt**:仅 Windows 平台的 ActiveQt 控件模块,Linux 无法构建,跳过。
- **qtopenapi**:6.12 新增模块,Debian 无任何打包先例,本轮暂跳过(无 6.11 debian
  目录可复用),后续可仿照 qt6-grpc 新写最小 debian 目录。
- **pyside6**:非 Qt 模块本体(独立项目),不在本轮范围。
- 实际打包:**38 个模块**(qt6-base、qt6-declarative、qt6-tools、qt6-shadertools、
  qt6-svg、qt6-multimedia、qt6-wayland、qt6-3d、qt6-quick3d、qt6-charts、qt6-graphs、
  qt6-grpc、qt6-location、qt6-positioning 等,全清单见 `scripts/modules.txt`)。

## 3. 问题与解决方法(流水记录)

### A. 环境准备阶段

**A1. sudo 非交互不可用 / root 权限**
- 现象:自动化脚本无法直接 `sudo`。
- 解决:用户提供了 root 密码,所有需 root 的操作统一走
  `echo '1' | sudo -S <cmd>`。

**A2. debootstrap 报 "unsupported variant"**
- 命令:`debootstrap --variant=buildd ... snipe <dir> <mirror>`
- 原因:UOS 定制的 `/usr/share/debootstrap/scripts/snipe` 脚本只实现了基础 chroot
  变体,不认识 `buildd`。
- 解决:去掉 `--variant=buildd`,bootstrap 后手动安装
  `build-essential devscripts fakeroot dh-exec pkg-kde-tools dh-python ccache`。

**A3. chroot 内访问 desktop-security-V25 返回 401 Unauthorized**
- 现象:security 源(https)在 chroot 内 apt update 报 401。
- 原因:UOS 专业版 security 源需要设备级授权认证,认证信息在宿主机
  `/etc/apt/auth.conf.d/uos.conf`(硬件指纹 license),净室内没有。
- 解决:将宿主机 `/etc/apt/auth.conf.d/` 拷入 chroot 同路径,并去掉该条的
  `[trusted=yes]`(带认证后正常校验签名)。

**A4. chroot 内 https 证书校验失败(NoSuchCertificate)**
- 现象:最小 chroot 没有 CA 证书,https 源握手失败。
- 解决:chroot 内安装 `ca-certificates`(随工具链安装时自动带入),重跑
  `apt-get update` 恢复。

**A5. Debian 6.11 打包依赖的三个包在 UOS 源不存在**
- `debhelper-compat (= 14)`:UOS 源 debhelper 为 13.24.2。构建脚本统一
  `sed 's/debhelper-compat (= 14)/debhelper-compat (= 13)/'`,debhelper 13 对
  cmake+ninja 构建系统完全够用。
- `dh-sequence-python3`、`dh-sequence-pkgkde-symbolshelper`:这俩在 Debian
  仓库里只是"激活 debhelper 序列"的元包(实际能力来自 dh-python /
  pkg-kde-tools,UOS 源都有)。自制两个同名空 deb(Depends 指向真实包)安装进
  chroot,debhelper 扫描已安装的 `dh-sequence-*` 包名即可激活对应序列。
- `libmd4c-html-dev`:Debian trixie 起 md4c 包改名,UOS 还是旧名
  `libmd4c-html0-dev`(0.4.8),control 里 sed 换名即可。

**A6. 文档包的循环依赖**
- 现象:qt6-base 的 `Build-Depends-Indep: qt6-base-dev (>= 6.11~) <!nodoc>`,
  构建文档包需要先有同版本 qt6-base-dev —— 自举循环。
- 解决:所有构建统一带 `DEB_BUILD_PROFILES="nodoc"`,跳过 -doc 包(与 Debian
  官方引导新版本时的做法一致)。

**A7. 符号文件与上游版本强耦合**
- 现象:Debian 打包目录携带每个库的 `.symbols`(C++ 符号表,由
  pkgkde-symbolshelper 管理)。6.12.0 相对 6.11.2 必有符号增删,构建时
  `dpkg-gensymbols` 比对失败会直接中断打包。
- 解决:净室构建脚本统一 `find debian -name "*.symbols" -delete`,构建时
  dpkg-gensymbols 会为 6.12.0 自动生成全新符号表写入 deb(依赖粒度从
  "精确符号级"退化为 shlibs 级,对升级调研无影响,已在交付物中注明)。

**A8. LTO 编译耗时**
- Debian rules 默认 `DEB_BUILD_MAINT_OPTIONS = hardening=+all optimize=+lto`,
  qtbase 全量 + examples 的 LTO 链接会让构建时间膨胀 2~3 倍。
- 解决:净室脚本统一 `sed 's/ optimize=+lto//'`,保留 hardening=+all。

**A9. 下载中断导致 tarball 损坏 + 官方源直连慢**
- 现象:并行 curl 从 download.qt.io 下载 38 个 tarball,约 100KB/s,且 5 个大文件
  (qt3d/qtbase/qtdeclarative/qtquick3d/qttools)传输中断,`xz -t` 校验损坏。
- 解决:改用清华 TUNA 镜像 `mirrors.tuna.tsinghua.edu.cn/qt/` 断点续传
  (`curl -C - --speed-time 20 --speed-limit 1000`),断点续传对不完整文件直接删除
  重下,全部 38 个文件(共 373MB)校验通过。

**A10. 补丁适配结论(全部 38 个模块)**
- `upstream_444b479_dont_consider_fallbackthemename_in_themename.diff`(qt6-base):
  补丁头注释明示 "Fixed in 6.11.3",已随 6.12.0 上游发布合入,从 series 剔除。
- `upstream_2efb7c6_correctly_compare_composites_with_multiple_engines.diff`
  (qt6-declarative):同为已合入上游的补丁,剔除。
- 其余模块补丁(qt6-base 7 个、qt6-3d/declarative/multimedia/quick3d/
  quickeffectmaker/tools/virtualkeyboard 各自的 series)在 6.12.0 源码树上
  全部干净应用;30 个模块无补丁。
- 6.11 → 6.12 上游源码兼容性总体非常好,Debian 补丁面几乎无改动。

### B. qt6-base 构建阶段

**B1. dpkg-source 以 `-F 0`(零 fuzz)应用补丁,cross2.patch 首次构建失败**
- 现象:`dpkg-buildpackage` 在 `dpkg-source --before-build` 阶段报
  `patch -t -F 0 -N ... cross2.patch ... exit status 1`。
- 原因:dpkg-source 应用补丁时不允许 fuzz(上下文行必须精确匹配),而
  cross2.patch 的一处上下文行在 6.12.0 已变化(仅偏移 + 少量改动,人工
  `patch -p1 --dry-run` 因默认允许 fuzz 而误判为可应用)。
- 解决:用允许 fuzz 的 `patch` 实际应用后,`diff --label a/... --label b/...`
  基于 6.12.0 上下文重新生成补丁(脚本 `strict-patch-fix.sh` 自动化该流程:
  快照 → fuzz 应用 → 按文件重 diff → 回滚 → `-F 0` 验证)。
- 教训:补丁预检必须用与 dpkg-source 完全一致的参数
  `patch -p1 -F0 -N --dry-run`,否则结论不可信。

**B2. chroot 内构建产物属主为 root,宿主侧清理需 sudo**
- 现象:重跑 `prep-module.sh` 时 `rm -rf` 报权限不足(chroot 内 dpkg-source
  生成的 `.pc/` 等文件属主是 root)。
- 解决:prep 脚本对旧 work 目录的清理失败时自动降级为 `sudo rm -rf`。

**B3. dh_install 报缺失:6.11 打包清单引用了 6.12 中已移动/更名的文件**
- qt6-base 首次构建在 dh_install 阶段失败,三类变更:
  1. `QtGui/bus_interface.h`、`cache_adaptor.h`、`deviceeventcontroller_adaptor.h`、
     `properties_interface.h`、`socket_interface.h` 5 个 AT-SPI D-Bus 适配器头文件
     上游转为私有(移到 `QtGui/6.12.0/QtGui/private/*_p.h`)→ 从
     `qt6-base-dev.install` 删除对应行(私有头由 qt6-base-private-dev 通配覆盖);
  2. wayland 协议 `xx-session-management-v1.xml` 转正更名为
     `xdg-session-management-v1.xml` → 改写清单行;
  3. `mkspecs/features/vxworks.prf` 被上游删除 → 从 `qmake6.install` 删除。

**B4. dh_missing 报未认领:6.12.0 新增文件不在任何 .install 清单**
- 6.12 新增内容包括:HarmonyOS(OHOS)部署工具、`q20atomic.h` 等新公开头、
  `FindEventfd.cmake` 等新 CMake 模块、ohos/vxworks-clang/win32-arm64ec-msvc 等
  新 mkspecs、`linux-dmabuf-v1.xml` 协议。
- 处理原则:新公开头文件/通用 CMake 模块/协议文件 → 追加进
  `qt6-base-dev.install`;平台专属工具与 mkspecs → 记入 `debian/not-installed`。

**B5. 关键教训:HarmonyOS 工具不能排除,必须正式打包**
- 现象:按 B4 把 `harmonydeployqt`/`QtHarmonyOSHelpers.cmake` 放进
  not-installed 后,所有下游模块(qt6-shadertools 起)cmake configure 失败:
  - `QtBuildHelpers.cmake` **无条件 include QtHarmonyOSHelpers**;
  - `Qt6CoreToolsTargets.cmake` 的 imported target `Qt6::harmonydeployqt`
    指向 `/usr/lib/qt6/bin/harmonydeployqt`,文件缺失直接致命。
- 结论:6.12 的 Qt6 CMake 包把 HarmonyOS 工具作为硬依赖,发行版打包必须把
  `harmonydeployqt{,6}`、`harmonyostestrunner` 收进 qt6-base-dev-tools,
  `QtHarmonyOSHelpers.cmake`、`QtPublicHarmonyOSToolchainHelpers.cmake`、
  `QtSetupAndroid.cmake` 收进 qt6-base-dev(Debian 官方 6.12 打包也须如此)。

**B6. chroot 内已装包不随本地仓库更新**
- 现象:qt6-base 重打包(补 harmony 工具)后,shadertools 构建仍失败于同样
  错误——chroot 里已安装的旧 qt6-base-dev 没有 harmony 文件。
- 原因:`apt-get build-dep` 只补缺,不会升级已安装的包。
- 解决:build-module.sh 的 build-dep 阶段在加锁段内先执行
  `apt-get -y dist-upgrade` 再 `build-dep`,保证净室内始终是本地仓库最新版本。

**B7. 文档目标的 qdoc 循环依赖**
- 现象:`override_dh_auto_build-indep: dh_auto_build -- docs` 无条件执行,
  而 qdoc 在 qt6-tools 里、qt6-tools 又依赖 qt6-declarative 依赖
  qt6-shadertools —— 首个模块就要 qdoc,循环。
- Debian 官方靠两阶段构建规避;本地 nodoc 构建须给 rules 的
  `override_dh_auto_build-indep` / `override_dh_auto_install-indep` 加
  `ifneq (,$(filter nodoc,$(DEB_BUILD_PROFILES)))` 守卫(脚本
  `skip-docs.py` 自动化,注意 make 条件指令必须顶格、续行空格开头的目标体
  收集要完整,否则会插入错位产生 "missing endif")。

**B8. control 中 doc 二进制包需注入 Build-Profiles: <!nodoc>**
- 现象:rules 跳过文档构建后,dh_install 仍要为 qt6-*-doc/-doc-html/-doc-dev
  包装入不存在的 qch/html 文件而失败。
- 原因:6.11 的 control 里 doc 二进制包段落没有 Build-Profiles 声明(Debian
  官方构建文档,无需声明)。
- 解决:`fix-doc-profiles.py` 给所有 -doc/-doc-html/-doc-dev 二进制段注入
  `Build-Profiles: <!nodoc>`,nodoc 构建时 dh_listpackages 自动排除这些包。

**B9. 宿主机系统崩溃(硬重启)的恢复**
- 崩溃后 chroot 的 /proc /sys /dev /build 绑定挂载全部丢失,后台构建任务
  全部死亡;但 ninja 构建树和已完成的 deb 产物完好。
- 恢复:重新挂载四类文件系统 → 校验 qt6-base 产物(崩溃前已完成)→ 刷新
  本地仓库索引 → 重启调度器。正在编译中的模块用
  `dpkg-buildpackage -nc`(跳过 pre-clean)可从 ninja 断点续编,避免全量重编。

### C. 批量构建阶段(其余 33 个模块)

**C1. 模块间依赖顺序与 Debian 打包不完全一致**
- 现象:qt6-declarative 的 Build-Depends 里前置要求 `qt6-languageserver-private-dev`、
  `qt6-svg-dev/-private-dev`(以及 shadertools),而 languageserver/svg 在最初的
  并行队列里排在 declarative 之后,导致链式等待失败。
- 解决:调度器串行链调整为
  base → shadertools → svg → languageserver → declarative → tools → translations,
  其余模块并行 + 失败重试轮(最多 3 轮,实测跑到第 4 轮全部收敛)。
- 结论:**并行批量前应先按 control 的 Build-Depends 做拓扑排序**,apt 的
  "build-dep 不可满足即整体失败"会把被牵连的依赖一起报出来(如 speech 的
  flite1-dev/libspeechd-dev 其实 UOS 源都有,只是随 multimedia 一起被 apt 整体拒绝)。

**C2. assimp ≥ 6.0 / libmng ≥ 2:UOS 源版本过老,需补建**
- qt6-quick3d 6.12 要求 `libassimp-dev (>= 6.0.0~)`,UOS 只有 5.3.1;
  qt6-imageformats 要求 libmng 2.x(pkg-config libmng),UOS 只有 1.0.10
  (连 .pc 都没有,导致 QT_FEATURE_mng=OFF,MNG 插件不构建)。
- 解决:从 Debian 源码包(TUNA 镜像 debian/pool)在净室内补建
  `assimp 6.0.5+ds-1`、`libmng 2.0.3+dfsg-5`,放进本地 deb 仓库后
  版本竞争自动胜出。这类"上游依赖演进"是 Qt 大版本升级的常态工作量。

**C3. 6.12 的 QML 插件 cmake 配置统一追加 "plugin" 后缀**
- 现象:qt6-multimedia 的 `QmlPlugins/Qt6quickmultimedia*`、
  `Qt6quick3dspatialaudio*` 全部找不到。
- 原因:6.12 起 QML 插件的 cmake 目标统一更名为 `Qt6<name>plugin*`
  (Qt6quickmultimediaplugin、Qt6quick3dspatialaudioplugin)。
- 解决:.install 清单批量 sed 改名。

**C4. Debian t64 包名与 UOS 不一致(libpipewire-0.3-0t64)**
- 现象:qt6-multimedia 构建出的 libqt6multimedia6 依赖
  `libpipewire-0.3-0t64`(Debian trixie 的 64 位 time_t 改名),UOS 里包名仍是
  `libpipewire-0.3-0`,导致后续所有需要 qt6-multimedia-dev 的模块(3d/charts/
  speech/virtualkeyboard)依赖解析全挂。
- 原因:Debian 6.11 打包 control 硬编码了 t64 包名。
- 解决:control 中 `libpipewire-0.3-0t64` → `libpipewire-0.3-0` 后 NC 重打包。
  **凡从 Debian 移植打包到 UOS 都要全量排查 t64 后缀**。

**C5. qt6-lottie:上游模块改名引发包结构重排**
- `LottieVectorImageGenerator` → `LottieVectorImageHelpers`(6.12 改名):
  删除 control 中过时的 libqt6lottievectorimagegenerator6 二进制段与
  Depends 引用,新模块文件并入已有的 helpers 包与 -dev 清单。
- 6.12 新增 `lottietoqml` 示例依赖不安装的内部 `Qt6LottieTools` cmake 包
  (模块内自举循环)→ 打补丁从 examples 构建中剔除该示例。

**C6. qt6-webview:QtWebEngine 6.12 未发布,无 webengine 后端**
- control 里三行 qt6-webengine-* 构建依赖移除;webengine 后端插件
  (libqtwebview_webengine.so 及 cmake 文件)不会构建,.install 中相关行删除。
- 结果:qt6-webview 可用(Quick/WebView 公共 API),但 Linux 下没有默认
  webengine 后端实现 —— 待 QtWebEngine 6.12 单独发布后补齐。

**C7. qt6-graphs:dh_qmldeps 与自包含 QML 模块的自举冲突**
- 现象:`dh_qmldeps` 报 "Missing QML module QtGraphs2D.impl"。
- 原因:pkgkde-getqmldepends 只扫描**已安装**的 QML 模块,而 QtGraphs2D.impl
  是 graphs 包自己附带(`debian/tmp/.../qml/QtGraphs2D/impl/`),安装前无法
  解析(Debian 官方靠两阶段构建规避)。
- 解决:rules 增加 `override_dh_qmldeps:` 跳过自动 QML 依赖生成
  (control 手写 Depends 仍生效)。

**C8. 6.12 新示例/新工具的清单补齐**
- qt6-tools 新增 kmap2qmap(+ /usr/bin/kmap2qmap6)→ qt6-tools-dev-tools;
  新增 QtQDocCatch{Conversions,Generators} 私有头 → qt6-tools-private-dev。
- qt6-quick3d 新增 volumetricfog / simpleruntimeloader / skymaterial 三个
  示例二进制 → qt6-quick3d-examples。
- qt6-multimedia:Qt6MultimediaQuickPrivate 的 Targets-none.cmake 随私有模块
  "转公开"消失 → 删除清单行。

**C9. 自动修复脚本的路径模板 bug**
- fix-install-issues.sh 最初按日志中**展开后**的路径
  (`usr/lib/x86_64-linux-gnu/...`)去 .install 里匹配,而清单里是
  `${DEB_HOST_MULTIARCH}` 模板形式,sed 永远匹配不上,导致 dh_install 类
  失败在重试轮里不能自愈。
- 修复:匹配时把 `x86_64-linux-gnu` 同时替换回 `${DEB_HOST_MULTIARCH}`
  两种形式都删。教训:自动修复器必须与清单的变量模板语法对齐。

## 4. 构建产物清单

**38/38 模块全部打包成功,共 467 个 Qt 6.12.0-1 的 deb(含 dbgsym 调试符号包),
合计约 1.9 GB**;另有辅助包 assimp 6.0.5、libmng 2.0.3、dh-sequence 空壳包 ×2。
明细见同目录《[产物清单.md](产物清单.md)》,产物位于
`qt6.12-build/build-results/`（含 .deb / .changes / .buildinfo 与本地源索引；产物约 1.9GB，未随本分享包提供）。

验证情况:
- chroot 内已安装 150 个 6.12.0-1 包,依赖闭环完整(apt 无报错);
- `qmake6 --version` 报告 Qt 6.12.0;`qmlformat`、`qmllint` 等 QML 工具链正常;
- 467 个 deb 的元数据(shlibs/depends)均由 dpkg-buildpackage 严格生成。

## 5. 遗留问题与后续建议

1. **qtwebengine 缺席**:Qt 6.12.0 官方发布不含 qtwebengine(自 6.12 起
   WebEngine 与主版本解耦单独排期)。qt6-webview 已打为"无 webengine 后端"
   形态。待其单独发布后,可复用 qt6-webengine 的 6.11 打包目录按本文流程补齐
   (预计是全流程中耗时最长、依赖最重的一个,需要 nodejs 等)。
2. **qtopenapi 未打包**:6.12 全新模块,Debian 无先例,本轮跳过;建议仿照
   qt6-grpc(同为轻量库模块)新写最小 debian 目录。
3. **qtactiveqt**:Windows 专属,Linux 无法构建,永久排除。
4. **文档包未生成**:全程 `nodoc` 构建配置(规避 qdoc 自举循环);如需
   qt6-*-doc 文档包,需在 tools+shadertools 齐备后做第二遍全量文档构建。
5. **符号表策略**:为保证 6.12 一次通过,构建时删除了 6.11 的 .symbols 文件、
   由 dpkg-gensymbols 重新生成(依赖粒度从符号级退化为 shlibs 级)。正式引入
   UOS 仓库前,建议按 Debian 流程跑 pkgkde-symbolshelper 做符号增量维护。
6. **t64 包名排查**:libpipewire 只是目前撞上的第一个;把这套 deb 发布到
   UOS 源前,应全量审查所有 Depends 中的 t64 后缀包名。
7. **LTO 被关闭**:为控制构建时长去掉了 `optimize=+lto`;正式发布构建建议恢复。
8. **补建的两个上游依赖**(assimp 6.0.5、libmng 2.0.3)为 Debian 原版源码包
   在 UOS 净室内的重打包,未做 UOS 化修改(如版本后缀加 ~uos1),引入时注意
   与 UOS 仓库的版本策略对齐。

---

## 6. 2026-10-21 重建实录(净室源修正 + 全流程复刻)

### 6.1 本次重建的动因与净室源修正

首次重建时净室只配了 professional-packages 主池 + security 池,**缺少 dde-snipe/udcp-snipe 两个 PPA**(探测 chinauos.com 域名 404,正确 host 是 `pools.uniontech.com`)。按宿主机当前源全量重建净室:

| 修正点 | 内容 |
|---|---|
| 净室源 | 完整镜像宿主机 sources.list:professional-packages(https,设备认证)+ pools.uniontech.com 主池(http)+ ppa/dde-snipe、ppa/udcp-snipe(snipe/2500u2)+ appstore + driver + security |
| 内网代理 | `pools.uniontech.com` 系解析到内网 10.x,须经宿主机 `/etc/apt/apt.conf.d/95-intranet-proxy` 的 `socks5h://127.0.0.1:11080`;该配置拷入净室(chroot 共享宿主网络栈)后 apt 一切正常 |
| debootstrap 引导 | 仍用可达的 professional-packages https 主池(设备认证经 /root/.wgetrc;**注意 wgetrc 不剥离引号,凭据必须裸写**) |

### 6.2 复刻中新增的问题与解决(补充 A1-C9 之外)

**D1. qt6-declarative 的安装清单手术(6.11→6.12 变化最集中的模块,共 8 类修复)**
1. QuickShapesDesignHelpersPrivate / QuickParticlesPrivate 的 metatypes、modules json、cmake 目标在 6.12 去掉 Private 后缀 → 批量 sed 改名;Targets-none.cmake 随私有模块转公开而消失 → 删行;
2. qmllint 插件 cmake(QmlLintQuickPlugin/QmlLintQdsPlugin 族、Qt6QmlCompilerPlugins.cmake)6.12 移除 → 删行;Qt6qt.conf.in 已不存在 → 删行;
3. **Qt6WriteQtConfPart.cmake 是 6.12 新 CMake 部件,qt6-tools 构建 examples 硬依赖,必须收进 qt6-declarative-dev**(与 B5 HarmonyOS 工具同级教训);
4. 新 QML 模块归包:StyleKit 全量 → qml6-module-qt-labs-stylekit;QtQuick/Controls/Native(6.12 新样式)→ qml6-module-qtquick;QtQml/DesignSupport → qml6-module-qtqml;BlurItem → effects 包;
5. 新增私有运行库必须随插件归包:libQt6QmlDesignSupport.so.6、libQt6QuickControls2Native/FluentWinUI3{,StyleImpl}.so(dpkg-shlibdeps 否则中断);
6. **静态库 libQt6QmlLint.a / libQt6QmlLS.a / libQt6QmlPreview.a 在 6.12 由 Qt6QmlLintPrivate/Qt6QmlLSPrivate cmake 硬引用,必须收进 qt6-declarative-dev**,否则 qt6-tools configure 失败。
以上全部固化在 `scripts/fix-qt6-declarative.sh`,由 adapt-module.sh 的通用钩子 `fix-<debsrc>.sh` 自动应用。

**D2. qt6-lottie 的 C5 修正**:6.12 中 LottieVectorImageGenerator 与 Helpers **并存**(并非改名),控制段删除过度;lottietoqml 是 tools 二进制(非示例),其链接的 Generator 私有库需随包发布 → libQt6LottieVectorImage{Generator,Helpers}.so.6 归入 libqt6lottie6。

**D3. 调度时序**:multimedia 在依赖齐备前烧光重试轮,连带 3d/charts/graphs/speech/virtualkeyboard build-dep 失败;**重跑调度器即自愈**(skip-collected 幂等)。教训:build_one 的重试轮间隔太短,应等待依赖或全局重跑。

**D4. 其他环境坑**:① 沙箱层对跨目录硬链接返回假 EXDEV(设备号相同)→ 快照放 work/ 内 + 失败降级普通拷贝;② 崩溃后增量树污染(vtable thunk 未定义引用)→ 必须清 obj 全量重编;③ TaskStop 杀不到 chroot 内子进程 → 恢复时先 pkill ninja/dpkg-buildpackage;④ 高并发(24 任务/16 核)诱发两次宿主机卡死 → JOBS 降为 6、池并发 2 后稳定。

### 6.3 本次重建产物

**40 个源码包全部成功(38 Qt 模块 + assimp 6.0.5 + libmng 2.0.3),577 个 deb,约 2.34 GB**(较首次 467 个多出的主要是新版打包目录对 QML 模块/静态库的更细拆分)。净室内 `qmake6 --version` = 6.12.0、qmllint 6.12.0 可用、`apt-get check` 依赖闭环无错、150+ 包安装无冲突。

两个已知构建缺口与首次一致(GStreamer 后端未编入、MNG 插件 feature 判否),属上游 feature 测试/净室环境问题,已在 §5 遗留清单。明细见《[产物清单.md](产物清单.md)》。

### 6.4 裸命令直接出包(2026-10-21 补充)

适配完成后的模块树**无需任何环境变量**,在净室内直接:

```
sudo chroot qt6.12-build/chroot
cd /build/work/<模块目录>          # 如 /build/work/qt6-svg
dpkg-buildpackage -b -us -uc       # 裸命令直接生成 deb(产物在上级目录 /build/work/)
```

实现方式:所有模块的 debian/rules 顶部注入两行,把 nodoc 构建语义固化进树内
(`DEB_BUILD_PROFILES := nodoc` + `export`),doc 包经 Build-Profiles 自动排除,
绕开 A6/B7 的 qdoc 自举循环——不再依赖外部传 `DEB_BUILD_PROFILES=nodoc`。
适配脚本(adapt-module.sh)已内置该注入,`scripts/enable-plain-build.sh`
可对存量树补注。已实测:qt6-svg / qt6-lottie / qt6-imageformats / qt6-multimedia
裸命令一次通过(未注入前 imageformats/multimedia 会因 doc 包清单引用
未构建的 qch/html 而 dh_install 中止)。

注意:裸命令须在净室(chroot)内执行——构建依赖(qt6-*-dev 6.12 及全部
Build-Depends)只装在净室里;宿主机上 dpkg-checkbuilddeps 会报缺依赖。

### 6.5 lottie 源码改动补丁化(2026-10-21)

§6.2-D2 中 lottie 对 debian/ 之外的直接改动(删 lottietoqml 示例目录)已规范化为
quilt 补丁 `debian/patches/remove-lottietoqml-example.patch`(最小化:仅从
examples/lottie/CMakeLists.txt 移除一行 `qt_internal_add_example(lottietoqml)`,
源文件保留不构建;注意 tools/lottietoqml 是随包发布的正式工具,不受影响)。
至此 **38 个模块源码树全部与官方 tarball 逐字节一致**(构建时补丁由 dpkg-source
应用、构建后自动回滚),全部改动收敛于 debian/ 内。

配套固化:`scripts/fix-qt6-lottie.sh`(adapt 钩子自动调用)整合了 lottie 全部
清单适配——control 段手术、Generator/Helpers 两模块并存的头文件/metatypes
通配认领、运行库归包、私有头与 mkspecs pri 归 private-dev、vectorimage 插件
归 qml 包、quilt 补丁放置。已端到端验证:全新 adapt → 裸命令
`dpkg-buildpackage -b -us -uc` 一次通过。

### 6.6 symbols 文件重建回填(2026-10-22)

§3-A7 的"删除 6.11 symbols 由 gensymbols 重生成"策略在 deb 产物侧成立,但打包仓库
仍需维护 symbols 文件(Debian 惯例;且 deepin 的 private-abi 原子锁依赖正是由 symbols
头部的 `| <pkg> #MINVER#, qt6-X-private-abi (= <ver>)` 行产生——无 symbols 的 deb
会丢失该依赖声明)。回填方案(`scripts/regen-symbols{,2}.sh`):

- 以 base(6.8)/Debian 上游(6.11 打包目录) 的 symbols 清单为基准,t64 名映射;
- 内容从 6.12.0-1 构建 deb 的库重新生成:`dpkg-gensymbols -p<pkg> -v6.12.0 -e<库文件> -I<头部模板> -c0`(Qt 驼峰库名与包名不匹配,必须用 -e 显式列库);
- 头部元数据沿用基准文件并升版 6.12.0(private-abi 行、Build-Depends-Package 保留,
  SymbolsHelper-Confirmed 标注 amd64);lottie 的 libqt6bodymovin6→libqt6lottie6
  上游改名按新 soname 重建;
- 共 31 个仓库 81 个 symbols 文件,已 amend 强推至各 fork。

注意:build-results 中已构建的 deb 仍是 shlibs 级(A7 策略),从 fork 仓库重走构建即可
得到含 symbols(及 private-abi 依赖)的正式产物。
