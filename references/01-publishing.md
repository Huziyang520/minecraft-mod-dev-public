# 1 发布身份与产物规范

> 管什么：模组的对外身份（作者、许可证、链接、描述）、`mod_id` 与 `mod_name` 的分工、版本号决策、产物命名、前置依赖区间、发布链路顺序与自检。
> 上游：`SKILL.md`（结论以此为准）；本文件给出字段全集、占位模板、命令、判定手法与返工案例。相关：[03-metadata.md](03-metadata.md) §3.1（字段落位与最小模板）、[04-i18n.md](04-i18n.md) §4.6（描述与文案语言）、[09-git-upload.md](09-git-upload.md)（上传链路细节）。

本文件里的所有"值"都是**占位模板**，不是某个项目的既定值。换项目时按新项目重新确定；不要照抄，也不要为了"看起来统一"把别的项目的身份搬过来。占位符 `<…>` 与 `YourName` 必须在正式发布前全部替换掉——留着占位符发出去的后果是平台文件页显示 `YourName`，且该页面无法就地修正，只能发新版本。

---

## 1.1 身份字段：先定占位，再填真值

新建模组第一步是把下面这张表的每一格填掉。空格子会在发布链路里以奇怪的方式炸出来：平台文件页作者栏空白、模组列表没有"主页"按钮、jar 内 LICENSE 条目缺失、maven 坐标缺 group。

| 项 | 写在哪 | 占位 | 约束 |
|---|---|---|---|
| 作者名 / `mod_author` | `gradle.properties`、两端元数据 | `YourName` | 多作者用逗号分隔（`alice, bob`）；同一模组两端必须逐字一致 |
| 许可证 `license` | `gradle.properties`、两端元数据、`LICENSE` 文件 | `MIT` | **只是常见选择之一**，见 §1.2；三处必须同名同版本 |
| `LICENSE` 文件 | 仓库根 | `Copyright (c) <年份> YourName` + 全文 | 正文逐字照抄官方文本，不改措辞、不加译者注 |
| 主页 URL | `fabric.mod.json` `contact.homepage`、TOML `displayURL` | `https://github.com/<你的账号>/<你的仓库>` | 必须是能打开的页面，发布前实际访问一次 |
| 主反馈 / Issue | `contact.issues`、`issueTrackerURL` | `https://github.com/<你的账号>/<你的仓库>/issues` | 仓库未建时不许猜 URL |
| 备用反馈页 | 描述文本，或 Issue 字段 | `https://<你的反馈页>/` | 页面没建好前的兜底；主站建好后仍保留 |
| CurseForge 页 | 平台侧（发布后回填） | `https://www.curseforge.com/minecraft/mc-mods/<mod-id>` | slug 允许连字符，与 modId 是两码事 |
| 目标平台 | 工程结构 | MultiLoader（Fabric + NeoForge / Forge） | 决定元数据分支与 `pack.mcmeta` 去留 |
| 描述 | `gradle.properties` `description` | 中英双语单行 | 见 §1.3 |

### 字段 → 门面映射（一处填错，哪个界面会坏）

同一条身份事实往往被多个界面读取，改一处不改另一处就会出现"游戏里正常、平台上错"的割裂。填完身份表后按下表逐行核对一遍：

| 事实 | 读取方 | 坏掉的表现 |
|---|---|---|
| `mod_author` | `fabric.mod.json` `authors`、TOML `authors`、jar 的 `Specification-Vendor` | 模组列表作者栏空白；平台作者页对不上 |
| `license` | 两端元数据的 `license`、jar 内 LICENSE 条目 | 平台审核要求返工；jar 内许可证缺失 |
| `mod_name` | `fabric.mod.json` `name`、TOML `displayName`、上传插件 `displayName`、jar 的 `Specification-Title` | 游戏内列表显示小写机器名；平台文件名难看 |
| 主页 URL | `contact.homepage` / `displayURL` | 列表里没有"主页"按钮，或点了 404 |
| Issue URL | `contact.issues` / `issueTrackerURL` | "问题"按钮缺失或 404 |
| `version` | 元数据 `version`、jar 文件名、平台文件名 | 版本号对不上，无法判断跑的是哪一份 |
| `mod_id` | 元数据 `id` / `modId`、资源命名空间、maven 坐标 | `Invalid modId` 构建失败，或资源全丢 |

> 命名大小写：`mod_author` 是自由文本，仓库里的分支名 / 目录名可能是小写。新建内容以名片为准；**已有项目的字段不要顺手改大小写**，那会牵动已发布物的 `Specification-Vendor` 与平台作者页，属于无收益的返工。

---

## 1.2 许可证怎么选（建议，不是规定）

| 许可证 | 适用场景 | 注意 |
|---|---|---|
| `MIT` | 希望别人随便用、只保留版权声明 | 最常见，但**不是默认值**，也不是唯一正确答案 |
| `Apache-2.0` | 需要显式专利授权条款 | 与 MIT 一样宽松，条文更长，需附 `NOTICE` 处理 |
| `LGPL-3.0` | 库类项目：允许链接，修改部分需开源 | 下游分发时要满足条款，实务上易被误读 |
| `MPL-2.0` | 文件级 copyleft | 改动文件需开源，新增文件可闭源 |
| `All Rights Reserved` | 不允许再分发 / 修改 | 与模组生态的"可整合包分发"预期冲突，慎重 |

- **铁律**：`license` 字段、`LICENSE` 文件正文、平台填写的许可证三者必须同名同版本。出现过"元数据写 MIT、仓库里放的是另一份许可证"的情况，后果是平台审核要求返工，且已发布版本无法就地修正——只能发新版本。
- 换许可证不是改一个字符串：已发布版本的历史许可证不可撤回，只能在**下一个版本**生效，并在 changelog 里写明"自本版本起许可证变更"。老版本仍按老许可证分发。
- 库 / 前置与业务模组的许可证可以不同（例如库用 `LGPL-3.0`、业务模组用 `MIT`），但要在身份表里分别记录，不要假设"同一仓库就同一许可证"。

---

## 1.3 描述与链接字段的落位

### `gradle.properties`

```properties
mod_author=YourName
license=MIT
description=Short English description. 中文简介。Issues: https://github.com/<你的账号>/<你的仓库>/issues | https://<你的反馈页>/
```

- **必须单行**：MultiLoader 模板把同一个 `description` 同时展开进 `fabric.mod.json`（JSON 字符串）与 `neoforge.mods.toml`（`'''…'''` 多行基本串）。一处换行，两端一起坏。
- `gradle.properties` 里的 `\n` 会被 Gradle 解析成**真实换行**，插进 JSON 直接造成 JSON 非法。症状是**模组完全不加载**，日志先导往往是某个第三方渲染模组崩，真正的报错在它之前。要分隔就两端都用空格 / ` | ` 写在一行。
- 先英文后中文：英文在前是因为平台默认语言与搜索结果都按英文索引；中文独占的描述会被判为不合格文案。
- 描述末尾追加反馈地址；纯文本处用 ` | ` 分隔多个 URL。
- 描述里**不要**写 Minecraft 版本、加载器版本、构建日期这类会过期的信息——描述是长期门面，版本信息归平台字段。

### 元数据文件

| 加载器 | 字段 | 位置 | 违约症状 |
|---|---|---|---|
| NeoForge | `displayURL`、`issueTrackerURL` | **`[[mods]]` 子表内** | 放顶层不生效：模组列表不显示主页 / 反馈 |
| NeoForge | `iconFile`（26.2+）/ `logoFile`（26.1.2-） | `[[mods]]` 子表内 | 图标错槽位或缺失，见 [03-metadata.md](03-metadata.md) §3.1.2 |
| Fabric | `contact.homepage`、`contact.issues`、`icon` | 顶层 | Mod Menu 的链接区不显示 |

TOML 里"顶层 vs `[[mods]]` 子表"是最容易犯的结构错误：`[[mods]]` 一旦出现，后面的键就属于该子表，写在它之前的键才是顶层。判定手法是解包 jar 读 `META-INF/neoforge.mods.toml`，确认这些键的**缩进位置在 `[[mods]]` 之后**，而不是只 grep 键名存在。

---

## 1.4 按钮语义对应与 URL 校验

| 界面按钮 | Fabric 字段 | TOML 字段 |
|---|---|---|
| 主页 / Website | `contact.homepage` | `displayURL` |
| 问题 / Submit Bug | `contact.issues` | `issueTrackerURL` |

- 两端四个值必须**相等**且都是**能打开的真实 URL**（发布前实际访问一次，不是凭记忆判断"应该能开"）。
- "对应值一致"不等于"主页与问题按钮必须同页"：主页可以是仓库首页或项目页，问题按钮必须是 issues 页或反馈站。两者指向同一页在语义上是错的——用户点"问题"却落到首页，等于没有反馈入口。
- **URL 字段只接受一个可点击地址时**：Issue 字段放主站，备用站写进 `description` 文本。Fabric 侧 `contact` 里可以加自定义键，但 **Mod Menu 的"链接"区只渲染 `custom.modmenu.links` 下的条目**（键 = 显示文本、值 = URL）；写在 `contact` 里的自造键（含非 ASCII 键）不会显示。要把备用地址做成可点按钮，必须走 `custom.modmenu.links`。
- **禁止 404 按钮**：仓库 / 平台页面尚未建立时不许猜 URL 顶上——Issue 字段直接放备用反馈页，等真实页面建好再改。已出现过"按钮点了 404"的发布事故，平台侧无法补救，只能发新版。

### 发布前 URL 检查（bash）

```bash
# 逐个打印元数据里的 URL，人工打开确认
grep -rhoE 'https?://[^"'\''< ]+' --include='*.json' --include='*.toml' . \
  | sort -u
```

判定口径：状态码 200 才算通过；301/302 要看最终落点是否为目标页；403 视为不可访问（平台审核也会失败）。

### 版本特例

| 版本 | 规则 |
|---|---|
| Forge 1.18.x | `logoFile` **不能带 `/`**，只能写根文件名（`icon.png`），否则点开模组详情崩 `IllegalArgumentException: Root resources can only be filenames, not paths` |
| Forge 1.19.2 | 建议同上，统一按根文件名写 |
| Forge 1.20.1+ | 可用 assets 路径 |
| NeoForge 26.2+ | `iconFile` + `bannerFile`；`logoFile` 已废弃，只在两者都缺时回退 |

---

## 1.5 `mod_id` 与 `mod_name` 的分工

这条是**最高优先级**，填错会同时污染游戏内模组列表与平台文件展示名，且发布后无法就地修正。

- `mod_id`：小写蛇形机器名（`yourmod`、`yourlib_yourmod`），进文件名、包名、注册表、命名空间目录。
- `mod_name`：**给人看的展示名**，首字母按品牌大小写、**允许空格**（`Your Lib`、`Your Mod`）。
- `mod_name` **一处驱动四处门面**：`fabric.mod.json` 的 `name`、`neoforge.mods.toml` 的 `displayName`、上传插件的 `displayName`、jar 内的 `Specification-Title`。
- 填成 `mod_id` 原样值（小写 / 连字符）会同时污染游戏内列表与平台文件页：曾出现把 `yourmod 1.3.0 (NeoForge) ...` 直接传上平台的事故，用户看到的是机器名而不是产品名。
- **每次发布前必做**：展开（或 `-dry` 跑一次）`displayName = "${mod_name} ${version} (<Loader>) for Minecraft ${minecraft_version}"`，确认**首词是展示名**；不符合就先改 `gradle.properties` 再构建，禁止"先传上去再说"。
- 旁证安全：`mod_name` 不参与 maven 坐标 / `archivesName`（都用 `mod_id`），唯一进文件名的地方是 jar 内 LICENSE 条目的 rename，允许空格。

### modId 命名正则（违反 = 构建失败 / `Invalid modId`）

```
^(?=.{2,64}$)[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$
```

| 要求 | 说明 |
|---|---|
| 长度 2~64 字符 | 过短 / 过长都不行；断言 `(?=.{2,64}$)` 先卡总长 |
| 仅小写字母 + 数字 + 下划线 | **禁止连字符 `-`**、大写、空格 |
| 必须以小写字母开头 | 不能数字 / 下划线开头 |
| 可用 `.` 分段 | 每段同样规则（每段也要小写字母开头） |

- 命名空间目录 `assets/<mod_id>/` 必须与之一致，`data/<mod_id>/` 同理。
- 平台项目 slug **允许带连字符**，但那是 slug 不是 modId，两码事；把 slug 写进 `mod_id` 是最常见的非法命名来源。
- 反例：`your-lib` 非法 → 改 `your_lib` 或 `your.lib`。

### 改 modId 的全链路同步清单

1. `gradle.properties` 的 `mod_id`
2. Java 入口类里的 `MOD_ID` 常量
3. 资源目录 `assets/<mod_id>/...`（含 `lang/`、`textures/`、`models/`、`blockstates/`）
4. 语言 key 前缀、附魔 / 注册表 ID 前缀
5. `fabric.mod.json` 的 `id`、mixin 配置文件名
6. `neoforge.mods.toml` 的 `modId` 与 `[[dependencies.<modId>]]`
7. 数据包目录 `data/<mod_id>/`、`pack.mcmeta` 若在 forge 模块内的命名空间引用
8. 平台侧项目 slug（可选，不影响 modId，但两者长期不一致会造成困惑）

改完构建若显示 `UP-TO-DATE`，需 `--rerun-tasks` 强制重跑 `processResources`，否则 jar 里仍是旧 modId（实测踩过）。判定手法：`unzip -p <jar> fabric.mod.json | grep '"id"'` 看产物而不是看源码。

---

## 1.6 版本号：构建前逐模组询问

构建新 jar 之前必须**逐模组**（一个模组问一轮）确认版本号，不许自行决定递增方式。给预设而不是开放式提问，可以避免 `0.8.3fix` 这类不合规写法。

| 预设 | 含义 |
|---|---|
| `1.0.0 → 1.0.1` | 修订号 +1（patch）：修 bug、内部改动，行为与接口向后兼容 |
| `1.0.0 → 1.1.0` | 次版本 +1、修订号归零（minor）：新增功能，仍向后兼容 |
| `1.0.9 → 1.1.0` | 修订号已到 9 时进位到次版本：避免两位数修订号 |
| `1.0.9 → 1.0.10` | 修订号 +1 进入两位数：明确走补丁路线、不介意 `1.0.10` 写法 |
| 自定义 | 由需求方指定任意版本号 |

- **版本号后缀必须带连字符**：`0.8.3fix` ✗、`0.8.3-fix` ✓。后缀不带连字符时，整串会被当成一个不可比较的 tag，下游区间判定失效。
- **必须逐模组问**：同一个工程里 `lib` / `addon` / `main` 是三个独立发布物，依赖区间也不同，不能一个号套三个仓库。一个号套三个仓库的后果是"前置升了、下游没升"，区间倒挂（见 §1.8）。
- 改号后**必须重新构建 + 重新部署**；不许只改文档、只改 `gradle.properties` 就算完。只改属性的后果是 jar 文件名变了、jar 内容没变，之后所有"版本号对不上内容"的排查都会失效。
- **同一个版本号绝不允许对应两份不同内容的 jar**，否则无法判断实例里跑的是哪一份（此坑代价是连续多次崩溃且难以归因）。
- 询问版本号时**紧接着询问本次平台 `releaseType`**（`release` / `beta` / `alpha`，逐模组问）。"无额外设置"也要问，不许替需求方默认。
- 版本号只增不减：已发布版本号不可复用。要回滚只能发一个**更大**的版本号，不能用旧号重发不同内容。

版本格式：`<minecraft_version>-<mod_version>`（如 `26.2-1.2.1`）只用于**分支目录 / 区间标签**；`gradle.properties` 的 `version=` 只写 mod 版本（见 §1.7）。

---

## 1.7 产物命名与构建脚本要点

### jar 文件名

**规范**：`<mod_id>-<version>-<loader>-<minecraft_version>.jar`

```
yourmod-1.2.2-neoforge-26.2.jar
yourmod-1.2.0-fabric-1.21.6-1.21.8.jar      # 区间分支写区间
yourmod-1.2.0-forge-1.21.9-1.21.11.jar
```

| 规则 | 说明 |
|---|---|
| 段顺序 | modId → 版本 → 加载器 → MC（或区间），四段固定 |
| 加载器段 | 小写：`fabric` / `neoforge` / `forge` |
| MC 段 | 单版本写一个号；跨版本分支写区间 `A-B` |
| 版本段 | **只写 mod 版本**，不带 MC 前缀 |

- 实现：约定插件里 `archivesName = "${mod_id}-${version}-${project.name}-${minecraft_version}"`。
- 旧形式 `<mod_id>-<loader>-<MC>-<MC>-<version>` 已废止：同一份 MC 号出现两次，且版本在最后，人工核对时极易看错。
- **maven 坐标 artifactId 保持 `<mod_id>-<loader>-<minecraft_version>`、不含版本**，下游 `group:artifact:版本` 不受命名改动影响。改产物名不影响坐标，改坐标才是破坏性变更。
- `version=` 只写 mod 版本：`version=1.2.2`，不带 `26.2-` 前缀；MC 版本体现在 jar 文件名与依赖区间里。混写会让 semver 段比较首段变成 MC 号，直接引发倒挂。
- 跨版本分支（一个 jar 覆盖一段 MC）时末段用区间标签，实现方式是在约定插件里用分支级属性 `mc_range_label` 拼 `archivesName`，并对 `AbstractArchiveTask` 清空 `archiveVersion`（见 [03-metadata.md](03-metadata.md) §3.3）。
- **Forge 端照同一规则**，`version` 段来自该分支 `gradle.properties`，**不手工拼**。手工拼的后果是产物名与元数据 `version` 不一致，平台侧两个文件重名风险。
- **目录名 = 分支名 = 区间标签**：一个 MC 版本线一个独立目录，`26.3` / `26.1-26.2` / `1.21.9-1.21.11` 这类目录名同时是 git 分支名与 jar 末段取值，三者必须同步。

### 平台展示名必须与 jar 名同源

`displayName = "<mod_name> <version> (<Loader>) for Minecraft <MC 或区间>"`

批量发布时逐条核对三件事：

1. **首词是展示名**，不是小写 modId；
2. **版本段与 jar 名一致**（含后缀连字符）；
3. **区间分支就写区间**——写成单个 MC 版本会让用户搜不到其他版本。

### 默认不产出 `-sources.jar` / `-javadoc.jar`

- `build-logic` 移除 `withSourcesJar()` / `withJavadocJar()`，并删除 `multiloader-loader.gradle` 里的 `tasks.named('sourcesJar'){...}` 块。
- ⚠ 三个模块 `build.gradle` 里对 `'sourcesElements'` / `'javadocElements'` 的 `configurations.named(variant)` 引用必须改成**存在才配置**：

```groovy
configurations.matching { it.name == variant }.configureEach { ... }
```

否则构建直接失败 `Configuration with name 'sourcesElements' not found`（实测踩过，六个模块文件都要改）。根因是"变体配置已被移除、脚本仍无条件引用"；判定手法是看失败堆栈是否指向 `configurations.named` 那一行，而不是去查依赖。

- 不发 sources 的理由：sources/javadoc jar 会占用平台文件位、混淆用户下载选择，且对 MultiLoader 多模块工程没有实际价值。要发源码就发到仓库，不要发到模组平台。

### 旧版本 jar 处置（两个目录规则不同，别合并）

| 位置 | 规则 |
|---|---|
| `build/libs` 内的旧版本 jar | **永不删除、永不移动、永不备份、永不记录**，留原位（新旧共存是常态） |
| 目标实例 `mods/` 内的旧版本 jar | 每次部署新版本后**直接删除被替换的旧 jar**，仅保留每个 modId 最新一版 |

理由：同一 modId 双版本并存会被 FML 拒载（表现为启动即报重复模组或直接崩）；而 `build/libs` 里的产物是审计依据，删了就无法回溯"上次发的是哪一份"。删除实例内旧 jar 无数据风险（源码在仓库里，jar 可重建）。

---

## 1.8 前置依赖版本区间

1. **区间按"实际兼容的最早版本"写**，用纯 semver 段比较：

| 加载器 | 写法 | 位置 |
|---|---|---|
| Fabric | `"yourlib": ">=1.2.2"` | `fabric.mod.json` `depends` |
| NeoForge | `versionRange="[1.2.2,)"` | `neoforge.mods.toml` `[[dependencies.<modId>]]` |
| Forge | `versionRange="[1.2.2,)"` | `mods.toml` `[[dependencies.<modId>]]` |

- 半开区间 `[1.2.2,)` 表示"下限含、上限不设"。写成 `[1.2.2,1.3)` 会人为制造拒载，除非确实存在已知不兼容的上限。
- 上限只有在**已知破坏性版本**时才写；凭"可能会坏"写上限，会让下游在正常升级时被误拒。

2. **前置没有破坏性变化时，下游区间无需改动**——不要因为前置出了新版就机械抬区间。只有前置出现 API 移除 / 行为变更、且下游实际用到新行为时才抬高下限。机械抬区间的后果是老用户的前置版本被判不满足，模组突然拒载。

3. **禁止版本格式混搭**：下游区间与前置 `version=` 必须同格式。前置版本从 `26.2-x.y.z` 改成纯 `x.y.z` 后，下游若还写 `>=26.2-1.2.1`，会发生**段比较倒挂**：`26 > 1`，于是已装的 1.3.0 反而被判"不满足"，模组拒载。

4. **部署新前置版本前的固定动作**：grep 所有下游仓库元数据里的旧格式版本号，逐个核对区间比较结果无倒挂，再部署。

```bash
# 找出所有仍写 MC 前缀的旧格式版本约束
grep -rnE '(>=|versionRange=)[^"]*26\.[0-9]+-[0-9]' --include='*.json' --include='*.toml' .
```

**倒挂的判定手法**：把下游区间的**下限字符串**与前置**实际版本字符串**按 `.` 分段后逐段比大小。只要首段是 MC 版本号而另一侧是 mod 版本号，就一定倒挂；不需要等到实机拒载才发现。

| 前置实际版本 | 下游写的区间 | 比较结果 | 结论 |
|---|---|---|---|
| `1.3.0` | `>=1.2.2` | `1 >= 1`，次段 `3 > 2` | 满足 |
| `1.3.0` | `>=26.2-1.2.1` | 首段 `26 > 1` | **倒挂**，误判不满足 |
| `26.2-1.3.0` | `>=26.2-1.2.1` | 首段相等，次段 `2 = 2`，第三段 `1.3.0 > 1.2.1` | 满足（但格式已过时） |

---

## 1.9 多版本线并行的身份一致性

一个模组往往同时维护多条 MC 版本线（每条线一个目录 / 一个分支 / 一个独立仓库）。发布身份必须**跨线一致**，否则同一模组在不同版本上看起来像两个产品。

| 事实 | 跨线是否允许不同 | 说明 |
|---|---|---|
| `mod_id` | **禁止** | 同一模组所有版本线必须同名，否则平台上是两个项目、语言键与数据包目录也要分叉 |
| `mod_name` | **禁止** | 展示名是品牌，跨版本改名字等于换产品 |
| `license` | 允许（需显式记录） | 换许可证只能在新版本生效，不能悄悄改 |
| `mod_author` | 允许（需显式记录） | 作者变更属于身份变更，要写进 changelog |
| 主页 / Issue URL | **禁止** | 同一模组指向不同仓库会让用户找不到对应版本的反馈入口 |
| `description` | 允许微调 | 但结构必须一致：先英后中、单行、含 Issue 链接 |
| `version` | 必然不同 | 各线独立递增，不要求跨线对齐 |
| `mod_name` 里的版本后缀 | 允许 | 后缀是发布标识，不是身份 |

- **判定手法**：把所有版本线的 `gradle.properties` 并排取出，逐键比对 `mod_id` / `mod_name` / `license` / `mod_author` / `description` 五键。差异出现在前两键就是错误，出现在后三键必须有书面理由。
- 新增一条版本线时，**从基准线的 `gradle.properties` 复制**而不是重新手写。手写的后果是拼写漂移（`YourMod` vs `Yourmod`），而这种漂移在发布前几乎不会被发现。
- 库 / 前置与业务模组的版本号**互不联动**：库升到 1.3.0 不代表业务模组要升到 1.3.0，后者只在用到新 API 时才抬区间（见 §1.8）。

```bash
# 跨线身份比对：每条线一行，输出五键取值
for d in */; do
  [ -f "$d/gradle.properties" ] || continue
  echo "== $d"
  grep -E '^(mod_id|mod_name|license|mod_author)=' "$d/gradle.properties"
done
```

---

## 1.10 发布文案：changelog 与 releaseType

- **changelog 不一定有**；有的话只写"版本小标题 + 本次真实改动"。禁止 H1 模组标题、流程说明、维护规则——文件内容会**原样公开**到平台页面，写进去的每一句都会被用户看到。
- 不写"修复了上一版的问题"这类过程叙述，写**改动后的行为**：`Fixed: 拦截提示在英文客户端显示为 key` 优于 `又修了一轮文案`。
- 不写内部术语：分支名、内部任务代号、`L1/L2` 这类分层标记对用户无意义。
- 平台 `releaseType` 逐模组确认；同一版本号在不同加载器上可以不同（例如 Fabric 侧 `beta`、NeoForge 侧 `release`），但要在发布记录里写明，否则下次无法解释为什么同版本标记不同。

| releaseType | 含义 | 何时用 |
|---|---|---|
| `release` | 稳定版，平台默认推荐 | 已通过实机验收、无已知阻断问题 |
| `beta` | 功能完整、待更多验证 | 新功能首版，或仅在部分版本线验证过 |
| `alpha` | 实验性 | 有已知限制、不建议普通用户使用 |

- （Neo）Forge 的 `description` **必须含 Issue 链接**；页面未建就用备用反馈页。缺失 Issue 链接时平台审核会要求补充，属可避免的返工。
- 描述与 changelog 的语言：**英文在前**，中文在后；不允许中文独占（面向玩家的公开文案统一规则见 [04-i18n.md](04-i18n.md) §4.6）。

---

## 1.11 发布链路顺序（阶段必须分清）

| 阶段 | 完成判据 | 谁来决定进入下一阶段 |
|---|---|---|
| 已构建 | 构建输出含 `BUILD SUCCESSFUL`，jar 落在 `build/libs` | 构建任务本身完成即可 |
| 已部署待验收 | jar 已拷入目标实例、旧 jar 已删、MD5 已记录 | 部署三查全部通过 |
| 已验收 | 实机进世界、目标功能按预期工作 | **需求方实机通过并主动确认** |
| 已上传 | 平台返回 File ID / version ID，且回读页面确认 | 需求方**主动下令**上传 |

- 顺序是**先构建后部署、先本地验收后版本发布**。跳过验收直接上传的后果是平台侧留下一个坏版本，撤回也只能标记为不可用，用户仍可能下载到。
- 汇报时必须写明当前处于哪个阶段。"已部署"不等于"已验收"，"已上传"不等于"用户已确认好用"——三个词不能混用。
- 上传前的最后一关是 §1.13 的自检清单；清单没跑完就上传，等于把返工成本转移到平台上。

---

## 1.12 常见返工清单（按发生频率排序）

| # | 现象 | 根因 | 判定手法 | 修复 |
|---|---|---|---|---|
| ① | 平台文件名是 `yourmod 1.2.0 (NeoForge)...` | `mod_name` 填成了 modId | 展开 displayName 看首词 | 改 `gradle.properties` 的 `mod_name`，重建重传 |
| ② | 模组完全不加载，崩溃报告指向渲染模组 | 描述里的 `\n` 被展开成真实换行，元数据解析失败 | grep 日志 `not a valid mod file` | 描述改单行，去掉转义 |
| ③ | 模组列表没有主页 / 反馈按钮 | `displayURL` / `issueTrackerURL` 写在 TOML 顶层 | 解包读 TOML 的键位置 | 移进 `[[mods]]` 子表 |
| ④ | 按钮点了 404 | 仓库 / 页面未建就填了猜的 URL | 发布前实际访问 | 改用备用反馈页，页面建好再换 |
| ⑤ | 图标跑到右侧大图、左侧列表空 | 26.2+ 仍用 `logoFile` | 看日志 `uses the deprecated 'logoFile'` | 改 `iconFile` |
| ⑥ | 改 modId 后 jar 里还是旧值 | `processResources` 命中 `UP-TO-DATE` | `unzip -p` 读产物 | `--rerun-tasks` 重跑 |
| ⑦ | 下游突然拒载 | 区间格式混搭导致倒挂 | 逐段比版本字符串 | 统一成纯 mod 版本格式 |
| ⑧ | 同一版本号两份不同 jar | 改号后只改了属性没重建 | 比对 jar 的 MD5 与 mtime | 重建 + 重新部署，版本号递增 |
| ⑨ | 构建失败 `Configuration with name 'sourcesElements' not found` | 移除 sources 变体后脚本仍无条件引用 | 看堆栈指向 `configurations.named` | 改 `configurations.matching {...}` |
| ⑩ | 平台审核要求补许可证 | 元数据 license 与仓库 LICENSE 不同名 | 三方比对 | 统一许可证，发新版本 |

---

## 1.13 发布记录与不可逆操作

平台上的发布是**不可逆**的：文件可以标记为不可用，但不能删除、不能换内容、不能改文件名。因此发布前要把"能撤回"与"不能撤回"分清，把全部成本压在前置校验上。

| 操作 | 可逆性 | 处理方式 |
|---|---|---|
| 改 `gradle.properties` 的版本号 | 可逆 | 重建即可，但必须递增，不许回退到旧号 |
| 覆盖 `build/libs` 里的同名 jar | **不可逆** | 靠"版本号只增不减"避免；旧 jar 永不删除 |
| 删除实例 `mods/` 内旧 jar | 可逆 | 可从 `build/libs` 重建，但要知道对应版本号 |
| 平台上传文件 | **不可逆** | 只能标记不可用；补救方式是发新版本 |
| 修改项目 slug | 半可逆 | 旧链接会 404，属"禁止 404 按钮"同类事故 |
| 修改已发布版本的 changelog | 可逆 | 但已下载用户看不到更新，等于没改 |

**发布记录要写四件事**（写进该项目的内部决策记录，不要写进仓库里的公开文档）：

1. 版本号 + `releaseType` + 目标平台；
2. jar 的 MD5 与 mtime（用于判断"实例里跑的是不是这一份"）；
3. 部署到的目标实例标识；
4. 验收结论（谁、在哪条版本线上、验了什么）。

判定"实例里跑的是哪一份"的手法：比对实例 `mods/` 内 jar 的 MD5 与 `build/libs` 产物的 MD5。不一致说明部署没成功或部署的是旧包，此时不要从代码找原因。

**同一版本号两份不同内容 jar 的检测**：按版本号分组，比对组内每个 jar 的 MD5；组内出现两个不同 MD5 即为违规。这类问题在实例侧的表现是"改了没生效 / 时好时坏"，而代码本身完全正常。

```bash
# 列出各 jar 的 MD5 与文件名，按版本号人工分组比对
for f in build/libs/*.jar; do
  printf '%s  %s\n' "$(md5sum "$f" | cut -d' ' -f1)" "$(basename "$f")"
done | sort
```

---

## 1.14 发布自检清单

- [ ] 身份表九项全部填实，无 `YourName` / `<你的仓库>` 占位残留。
- [ ] `license` 字段、`LICENSE` 文件、平台许可证三者同名同版本。
- [ ] `icon.png` 为 128×128（读 PNG IHDR，偏移 16/20 起各 4 字节大端宽高，别信看图软件缩略图）。
- [ ] 两端元数据字段齐全（用 [03-metadata.md](03-metadata.md) §3.1.3 模板逐项对）。
- [ ] 所有 URL 按钮真实可访问、无 404；反馈地址含备用页。
- [ ] 每个待发布模组的 `mod_name` 是人类展示名；展开后的平台 displayName 首词不是小写 mod_id。
- [ ] 版本号已**逐模组**问过；平台 `releaseType` 已**逐模组**问过。
- [ ] 版本号后缀带连字符；同一版本号没有对应两份不同内容的 jar。
- [ ] jar 文件名符合 `<mod_id>-<version>-<loader>-<MC 或区间>.jar`；区间分支写的是区间。
- [ ] `version=` 只含 mod 版本，无 MC 前缀；产物名与元数据 `version` 一致。
- [ ] 下游模组前置区间与前置实际 `version=` 同格式、无倒挂。
- [ ] `description` 单行、无 `\n` 转义；先英后中；含 Issue 链接。
- [ ] changelog 无 H1 标题、无流程说明、无维护规则、无内部术语。
- [ ] `build/libs` 旧 jar 未被移动或删除；目标实例 `mods/` 内无同 modId 双版本。
- [ ] 构建新 jar 前已确认目标测试实例（仅首次需问，之后沿用；换目标再问）。
- [ ] 汇报阶段用词准确：已构建 / 已部署待验收 / 已验收 / 已上传。
