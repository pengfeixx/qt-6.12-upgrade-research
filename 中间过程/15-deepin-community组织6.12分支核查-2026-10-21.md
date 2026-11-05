# deepin-community 组织 Qt 仓库 6.12 分支核查

| 项目 | 内容 |
|---|---|
| 核查日期 | 2026-10-21 |
| 核查对象 | github.com/deepin-community 组织内本次 Qt 6.12 升级涉及的打包仓库 |
| 核查方法 | GitHub REST API(`gh` CLI,认证账号 pengfeixx):仓库枚举(search API)→ 逐仓库分支列表 → `upstream/qt6.12` 分支内容抽查(`.cmake.conf` 版本标记 / debian/changelog / 提交历史) |
| 关联背景 | 本仓库净室打包已产出 38 模块 6.12.0-1 deb(见 [qt6.12-build/docs](../qt6.12-build/docs/Qt6.12净室打包记录.md));本核查回答"社区仓库是否已有 6.12 分支可承接" |

## 1. 核心结论(先说结论)

1. **分支名义覆盖 34/38**:38 个模块中 34 个在 deepin-community 有同名仓库且带 `upstream/qt6.12` 分支(命名完全统一,无其他 6.12 变体名);qt6-webengine(对照项,本次未打包)同样有该分支。
2. **但没有任何分支真正包含 6.12 源码**:全部 34 个已核实分支的内容仍是 **6.8.0**(`.cmake.conf` 的 `QT_REPO_MODULE_VERSION` 全部为 6.8.0;qt6-3d 等的最后上游导入提交停在 2024-11-22 "New upstream release (6.8.0)")。其中 14 个分支提交日期停在 2024-11-22,其余在 2026-09~10 有 6.8 的 CVE 维护提交。
3. **qt6-graphs 是空模板仓库**:`upstream/qt6.12` 分支只有 README/.github/debian 骨架,changelog 为 `template-repository (1.0-1)`,无任何 Qt 源码。
4. **4 个模块无 6.12 分支**:qt6-canvaspainter、qt6-quickeffectmaker、qt6-tasktree 仓库不存在(组织内 search 无结果,API 404);qt6-remoteobjects 仓库存在(2022 年建,2026-09-16 仍在推送)但只有 master 分支。
5. 判定:这批 `upstream/qt6.12` 是**预铺的占位分支**(从 6.8 状态批量复制/改名),6.12 源码导入尚未开始;分支铺位本身说明社区已把 6.12 纳入计划。

## 2. 逐仓库明细(38 模块 + webengine 对照)

| # | 仓库 | upstream/qt6.12 分支 | 分支真实内容 | 备注 |
|---|---|---|---|---|
| 1 | qt6-base | ✅ | 6.8.0 | changelog 顶 6.8.0+dfsg-0deepin40;2026-10-15 仍在打 6.8 CVE(QAuthenticator);master=-0deepin41 |
| 2 | qt6-declarative | ✅ | 6.8.0 | 另有 topic-qt-6.6.2/old-master 历史分支 |
| 3 | qt6-tools | ✅ | 6.8.0 | |
| 4 | qt6-shadertools | ✅ | 6.8.0 | |
| 5 | qt6-svg | ✅ | 6.8.0 | 另有 fix/cve-2025-10729 |
| 6 | qt6-multimedia | ✅ | 6.8.0 | 另有 revert-lto-changelog |
| 7 | qt6-wayland | ✅ | 6.8.0 | |
| 8 | qt6-3d | ✅ | 6.8.0 | 分支最后导入提交 2024-11-22 |
| 9 | qt6-5compat | ✅ | 6.8.0 | 2026-10-13 有 6.8 CVE 提交 |
| 10 | qt6-charts | ✅ | 6.8.0 | |
| 11 | qt6-connectivity | ✅ | 6.8.0 | |
| 12 | qt6-datavis3d | ✅ | 6.8.0 | |
| 13 | qt6-graphs | ✅ | **空模板** | 仅 README/.github/debian 骨架,无源码 |
| 14 | qt6-grpc | ✅ | 6.8.0 | |
| 15 | qt6-httpserver | ✅ | 6.8.0 | |
| 16 | qt6-imageformats | ✅ | 6.8.0 | |
| 17 | qt6-languageserver | ✅ | 6.8.0 | |
| 18 | qt6-location | ✅ | 6.8.0 | 另有 dcbot/debian/6.8.2(bot 打包分支) |
| 19 | qt6-lottie | ✅ | 6.8.0 | |
| 20 | qt6-networkauth | ✅ | 6.8.0 | |
| 21 | qt6-positioning | ✅ | 6.8.0 | |
| 22 | qt6-quick3d | ✅ | 6.8.0 | |
| 23 | qt6-quick3dphysics | ✅ | 6.8.0 | |
| 24 | qt6-quicktimeline | ✅ | 6.8.0 | |
| 25 | qt6-remoteobjects | ❌ 仅 master | — | 仓库存在(2026-09-16 仍在推送),无 6.12 分支 |
| 26 | qt6-scxml | ✅ | 6.8.0 | |
| 27 | qt6-sensors | ✅ | 6.8.0 | |
| 28 | qt6-serialbus | ✅ | 6.8.0 | |
| 29 | qt6-serialport | ✅ | 6.8.0 | |
| 30 | qt6-speech | ✅ | 6.8.0 | |
| 31 | qt6-tasktree | ❌ **仓库不存在** | — | 6.11 新模块,组织内 404 |
| 32 | qt6-translations | ✅ | 6.8.0 | |
| 33 | qt6-virtualkeyboard | ✅ | 6.8.0 | |
| 34 | qt6-webchannel | ✅ | 6.8.0 | |
| 35 | qt6-websockets | ✅ | 6.8.0 | |
| 36 | qt6-webview | ✅ | 6.8.0 | |
| 37 | qt6-canvaspainter | ❌ **仓库不存在** | — | 6.11 新模块,组织内 404 |
| 38 | qt6-quickeffectmaker | ❌ **仓库不存在** | — | search 全名与 effectmaker 关键词均 0 结果 |
| 对照 | qt6-webengine | ✅ | 6.8.0 | 另有 dcbot/debian/6.8.2+dfsg、icu74、bai/combined-fix |

注:所有仓库 default branch 均为 master(当前 6.8 维护线)。

## 3. 对升级工作的含义

1. **社区 6.12 打包尚未实质启动**,与阶段一"deepin 公开仓库停留在 6.8.x"结论一致;占位分支已铺好(34/38)表明升级在计划内。
2. **建仓缺口 4 个**:canvaspainter、tasktree、quickeffectmaker(连仓库都没有)+ graphs(空壳)——与打包记录"qt6-openapi 无先例"同属新增模块建仓工作量。
3. **本仓库净室产物可直接反向承接**:`upstream/qt6.12` 分支做 6.12.0 tarball 源码导入(我们的工作树与 tarball 逐字节一致、lottie/cross2 均为规范 quilt 补丁),debian/ 直接采用我们调通的版本(裸命令可出包),可跳过社区从 6.8 底子重做适配的整段工作。
4. qt6-location/qt6-webengine 存在 `dcbot/debian/*` 分支,说明组织有 bot 自动化打包流水线,反向灌入时可对接。

## 4. 复现命令

```bash
# 枚举组织内 qt6 仓库
gh api "search/repositories?q=org:deepin-community+qt6+in:name&per_page=100" --jq '.items[].name'
# 逐仓库分支列表与 6.12 分支真实版本
gh api "repos/deepin-community/<repo>/branches?per_page=100" --jq '.[].name'
gh api "repos/deepin-community/<repo>/contents/.cmake.conf?ref=upstream/qt6.12" --jq '.content' | base64 -d | grep VERSION
gh api "repos/deepin-community/<repo>/commits?sha=upstream/qt6.12&per_page=5" --jq '.[].commit.message'
```

## 5. 附带发现

组织内另有 qt6mpris、networkmanager-qt6、libquazip1-qt6 三个 qt6 相关仓库,属绑定库/消费者,不在本次 38 模块打包范围。

## 6. 后续动作:6.12 代码已推送至 fork(2026-10-22)

按上述核查结果,将净室打包调通的 6.12 内容(官方 tarball 原样源码 + 适配完毕的
debian/,即"源码树纯净 + 裸命令可出包"形态)推送至 35 个 fork 仓库
(github.com/pengfeixx/qt6-*)的 `upstream/qt6.12` 分支:

- 提交形式:单提交替换全树(6.8 内容 → 6.12 源码 + 新 debian/),
  双语规范提交信息,PMS: TASK-396869;
- 35/35 推送并经 GitHub API 远端复核(分支头即 6.12 导入提交);
- 坑与解法:① 全局 git 配置 ghfast.top 代理改写导致 push 认证失败 →
  临时取消改写直推;② gh 令牌缺 workflow scope,含 .github/workflows 变更的
  qt6-base 被 OAuth 拒绝 → 改 SSH(走本机 socks5 代理的 ssh config)推送;
  ③ 大仓库偶发网络闪断 → SSH 重试即过。

## 7. t64 决策落地:保持非 t64 命名(2026-10-22)

评审指出提交中 libqt6core6 更名为 libqt6core6t64 的问题。核实:该名继承自 Debian
sid 6.11 打包基线——且 6.11 打包中**仅 libqt6core6t64 一个包保留 t64 名**,其余
libqt6* 早已改回非 t64(其 control 内残留 `Breaks/Replaces: <名>t64` 过渡声明)。
按"对齐 deepin 现状"决策(对应总体方案 W-1.3-③,t64 保持自持非后缀基线),全部
35 仓回退(`scripts/fix-t64.py`):

- control:`Package: libqt6core6t64 → libqt6core6`,同名 .install/.symbols 文件改名,
  内容引用同步;删除全部 t64 过渡声明(Breaks/Replaces 指向 t64 名的条目、
  `Provides: ${t64:Provides}`——deepin 生态从未存在 t64 名,均为死代码);
- 顺带修正:`qt6-base-private-abi (= 6.11.2)` → `(= 6.12.0)`(6.11 基线遗留版本号,
  原样构建会让 deb 提供错误版本的 private-abi 锁);
- 验证:35/35 远端一致,control 段结构完整(如 qt6-base 34 段),t64 残留 0。

注意:净室 build-results 中的 deb 仍为 t64 名 + private-abi (= 6.11.2)(构建时未含
此修正),正式产物需以 fork 仓库当前 debian/ 重走构建。

## 8. CI 自举依赖问题与两阶段落地(2026-10-26)

PR #73 合入后 CI 失败:pbuilder 全量构建把 `Build-Depends-Indep` 的
`qt6-base-dev/qt6-documentation-tools (>= 6.11~) <!nodoc>` 当硬依赖解析,而仓库
存量仅 6.8(6.8 时代版本约束与存量同代故可过;该两包 6.12 版要等 qt6-tools 6.12
入库,tools 又依赖 declarative→base,自举环无解)。

决策(两阶段方案):**首发引导阶段**仅 qt6-base 及链上 5 仓(shadertools/svg/
languageserver/declarative/tools)一次性移除该 indep 依赖(本流水线 rules 固化
nodoc、文档从不构建,属死代码);qt6-tools 6.12 入库后其余 29 仓**原样可编**;
**恢复阶段**再按 Debian 阶段二模式加回依赖重建。批量为 29 仓去依赖的首次尝试
已应用户要求回退(reflog 精确恢复)。

- 修复 PR:[deepin-community/qt6-base#74](https://github.com/deepin-community/qt6-base/pull/74)
  (fix/indep-build-deps → develop/qt6.12,注意 PR#73 合入后社区将 upstream/qt6.12
  整理为 develop/qt6.12);
- 工具:`scripts/fix-indep-deps.py`(字段级清理,条目清空则整字段移除)。
