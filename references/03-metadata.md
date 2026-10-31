# 3 工程结构与元数据

> 管什么：MultiLoader 工程结构、`fabric.mod.json` / `neoforge.mods.toml` / `mods.toml` 字段全集与落位、模组菜单图标与链接槽位、`pack.mcmeta` 各版本取值、资源枚举、产物命名、`expand` 陷阱、KnownPack。
> 上游：`SKILL.md`（结论以此为准）；本文件给出模板、字段表、版本分界与判定手法。相关：[01-publishing.md](01-publishing.md) §1.1（身份字段取值）、[04-i18n.md](04-i18n.md) §4.1（语言键与 BOM）、[06-build.md](06-build.md) §6.4（`processResources` 配置）、[11-verdicts.md](11-verdicts.md)（易错结论对照）。

本文件的示例用 `yourmod` 表示"你的 modId"，用 `<…>` 与 `${…}` 表示需要替换的占位。它们不代表任何真实项目，只是位置标记。

---

## 3.1 元数据规范

### 3.1.1 字段放置（铁律）

**违反 = 模组列表不显示主页 / 反馈 / 图标，或 jar 被判"不是有效 mod 文件"而完全不加载。**

| 字段 | 加载器 | 必须放在 | 违约症状 |
|---|---|---|---|
| `displayURL` | NeoForge / Forge | **`[[mods]]` 子表内** | 列表不显示主页按钮 |
| `issueTrackerURL` | NeoForge / Forge | **`[[mods]]` 子表内** | 列表不显示反馈按钮 |
| `iconFile`（26.2+）/ `logoFile`（26.1.2-） | NeoForge / Forge | **`[[mods]]` 子表内** | 图标缺失或跑错槽位 |
| `description` | NeoForge / Forge | `[[mods]]` 子表内 | 见下方"单行"规则 |
| `contact.homepage` / `contact.issues` / `icon` | Fabric | `fabric.mod.json` 顶层 | Mod Menu 链接区不显示 |

- **同一模组的 Fabric / NeoForge 两端身份必须一致**：作者、许可证、展示名、版本号、主页、Issue 六项逐字对齐。两端不一致时，用户在不同加载器上看到的是两个不同的模组页。
- **TOML 结构判定手法**：`[[mods]]` 一旦出现，其后的键都属于该子表。解包 jar 读 `META-INF/neoforge.mods.toml`（或 `META-INF/mods.toml`），确认目标键**缩进在 `[[mods]]` 之后**——只 grep 键名存在，无法区分顶层与子表。

### 3.1.2 模组菜单界面：图标 / 链接 / 配置入口

同一个模组会在多个"模组菜单"界面里露出门面，各界面读的字段并不相同，必须分别对齐。按「界面 → 版本范围 → 读什么字段 → 注意项」记：

| 模组菜单界面 | 版本范围 | 列表小图标 | 详情大图 / 横幅 | 主页 / 问题按钮 |
|---|---|---|---|---|
| **NeoForge 原生模组列表**（左侧列表 + 右侧详情） | **26.2 起**（26.1.2 及更早是旧版列表） | `iconFile` | `bannerFile`（缺才回退已废弃的 `logoFile`） | `displayURL` / `issueTrackerURL` |
| **Fabric Mod Menu** | 全版本 | `fabric.mod.json` 的 `icon` | 同左 | `contact.homepage` / `contact.issues`；**"链接"区只渲染 `custom.modmenu.links` 下的条目（键 = 显示文本、值 = URL），写在 `contact` 里的自造键（含非 ASCII 键）不会显示** |
| **Catalogue**（Fabric / NeoForge 通用列表） | 全版本 | `icon`（Fabric）/ NeoForge 侧 logo 字段 | 同左 | `contact.homepage` / `contact.issues` |
| **配置入口** | 全版本 | — | — | NeoForge 原生列表的"配置"按钮走 `IConfigScreenFactory`；Fabric 侧由 Mod Menu 读模组自己的配置工厂 |

#### `logoFile` 在 26.2 起已废弃 —— 写错就是"小图标跑进大图槽位"

- **两个槽位的语义（26.2+，游戏内实测确认）**：
  - **左侧列表项的小图标**：方形 1:1，读 **`iconFile`**；
  - **右侧详情页顶部的宽横幅**：横图，读 **`bannerFile`**；
  - `logoFile` 已废弃，只在上面两者都缺时回退。
- **版本分界（很重要）**：**26.1.2 及更早**原生列表**只有右侧那一处**图标位（写 `logoFile`，1:1 图只能显示在右边）；**26.2 起**才同时支持"左侧小图标 + 右侧横幅"两个槽位。跨版本维护同一份元数据时要按此分界分支处理。
- **只有 1:1 方图、没有宽横幅的模组，必须写 `iconFile="icon.png"`**：这样方图才会出现在**左侧列表**。若沿用旧的 `logoFile`，26.2+ 会把 128×128 方图拉伸进**右侧详情横幅**槽位（左侧列表反而没有图标），同时日志多一条加载警告：
  `Mod X uses the deprecated 'logoFile' property; change to 'bannerFile' and/or (for square icons) 'iconFile'`。
- **排查口诀**：症状 =「图标跑到右边的大图上、左边列表空着」→ 把 `logoFile` 改成 `iconFile`。
- 判定代码位置（NeoForge）：`client/gui/modlist/DefaultModDisplayInfo` 的 `banner()` / `icon()`；`internal/LogoFileWarningsHandler`。
- **版本分界补充**：`26.2` 与 `26.3` **都有**这套 `iconFile` / `bannerFile`（26.2.0.86 实测同样存在）；**26.1.2 及更早没有**，那些版本仍只有 `logoFile`。
- ⚠ **"文档查不到 ≠ 不存在"**：曾据"NeoForge 文档查不到 `iconFile` / `bannerFile`"推断字段不存在，被实机验收推翻。此类字段**以实机 / 反编译为准**。

#### 图标文件本身

1. **统一 128×128 PNG**（正方形，边长 ≤256）。Catalogue 的 `Branding.ICON` 用 `SQUARE + LESS_THAN_OR_EQUAL 256` 硬校验——超限抛 `InvalidBrandingImageException` 后**显示问号缺省图，且不报错进日志**。Mod Menu、NeoForge 原生列表无此限制，所以症状是"别处正常、只有 Catalogue 问号"。128 是 Fabric 官方推荐尺寸，换新图永远不得超 256。
2. **图标接线（双端各一处）**：`fabric.mod.json` 加 `"icon": "icon.png"`；`neoforge.mods.toml` 在 **`[[mods]]` 子表内**加 `iconFile="icon.png"`（26.2+；26.1.2 及更早用 `logoFile`）。文件放对应加载器模块的 `src/main/resources/` 根。
3. **尺寸判定**：读 PNG IHDR——偏移 16/20 起各 4 字节大端宽高，**别只信看图软件缩略图**。
4. 改编尺寸时的安全顺序：**先生成临时文件 → 校验 IHDR → 再覆盖原件**。直接 `Move-Item -Force` 覆盖会先毁掉原 PNG，中途失败就失去原件。另：PowerShell 5.1 把**无 BOM 的 UTF-8 `.ps1` 按 ANSI 读**，脚本里的中文路径字面量会乱码；脚本要写成路径无关（`Join-Path (Get-Location) '...'`），由调用方设工作目录。

```powershell
# 读 PNG 真实尺寸（偏移 16/20 起各 4 字节大端）
$b = [System.IO.File]::ReadAllBytes('icon.png')
"$([BitConverter]::ToInt32($b,16)) x $([BitConverter]::ToInt32($b,20))"
```

#### Mod Menu 徽章

模组菜单列表里的**小标签**由 `fabric.mod.json` 的 `custom.modmenu` 控制，写错就是「该有标记却没有」：

- **前置库 / API 必须声明 `library` 徽章**，否则列表里不会出现「**前置**」标记：

```json
{
  "custom": {
    "modmenu": {
      "badges": ["library"]
    }
  }
}
```

- **纯客户端模组**的「**客户端**」徽章由 `"environment": "client"` **自动**产生，无需手写。
- 另一个可用徽章是 `deprecated`。
- ⚠ **NeoForge 原生模组列表没有徽章机制**（对应信息靠描述与依赖关系表达），不要在 `neoforge.mods.toml` 里找等价字段。
- ⚠ Mod Menu 的 `parent` 是「**模块归属**」语义：把该模组折叠进某个库的「模块」列表。只有当前置库确实把下游当模块时才写；**普通下游写了会从列表顶层消失**。

#### Cloth Config 配置界面约定

- **初始值取「当前配置值」，不是默认值**：控件初值用 `getter()` 的返回值。用默认值会导致界面永远显示默认值——表现为「改了不生效 / 再次打开又变回去」，而实际文件的旧值仍在生效（于是"显示是、功能却没开"）。
- **布局**：`builder.setGlobalized(true)` + `builder.setGlobalizedExpanded(true)` → **左侧分类树**（左分类、右条目）；不设置时多个分类会以**顶部标签按钮**呈现。
- **写值立即落盘**：配置项 setter 内「改内存 + `save()`」，不要只依赖 `setSavingRunnable`（它与各配置项 save 回调的先后顺序不保证，先跑 runnable 会把旧值写进文件）。
- **同一配置文件全局唯一实例**（按路径缓存），否则界面与业务代码各持一份内存副本，改完不生效。
- **文案走语言键**：`assets/<modid>/lang/` 提供 `en_us` + `zh_cn`（UTF-8 无 BOM）。
- **★ Cloth Config 的 modId 两端不同**：Fabric `cloth-config`、NeoForge `cloth_config`（NeoForge modId 不允许连字符）。判定必须两端都查，否则"装了也认不出来"；依赖用 `compileOnly(...) { transitive = false }`，仓库 `https://maven.shedaniel.me/`。

### 3.1.3 `description` 必须单行（违反 = 模组完全不加载）

- `description` 是**基础字符串**（双引号），**不能含真实换行**。
- MultiLoader 模板用 `'''…'''` 多行基本串展开时，源头 `gradle.properties` 也必须单行（见 [01-publishing.md](01-publishing.md) §1.3）。
- 症状链：展开后出现真实换行 → TOML 解析报 `ParsingException: Invalid newline in basic string` → **整个 jar 判为"不是有效 mod 文件"** → 模组完全不加载 → 连锁引发一堆 `Cowardly refusing to send event` 与第三方渲染模组崩溃。
- **诊断要点**：崩溃报告先导往往是某个第三方渲染模组（如 Sodium），但根因在它之前的日志里的 `not a valid mod file`。**先 grep `not a valid mod file` 找真正的元数据解析错误**，不要顺着崩溃报告查渲染。
- 修复：反馈地址并入同一行，去掉 `\n`。

### 3.1.4 元数据最小可运行模板

以下骨架可直接抄，只替换 `${…}` 占位与 URL；占位由 `processResources` 在构建时展开。

**Fabric（`fabric/src/main/resources/fabric.mod.json`）**：

```json
{
  "schemaVersion": 1,
  "id": "${mod_id}",
  "version": "${version}",
  "name": "${mod_name}",
  "description": "${description}",
  "authors": [ "${mod_author}" ],
  "license": "${license}",
  "icon": "icon.png",
  "contact": {
    "homepage": "https://github.com/<你的账号>/<你的仓库>",
    "issues": "https://github.com/<你的账号>/<你的仓库>/issues"
  },
  "environment": "*",
  "depends": {
    "fabricloader": ">=${fabric_loader_version}",
    "minecraft": "~${minecraft_version}",
    "java": ">=${java_version}"
  },
  "entrypoints": { "main": [ "<主类>" ] },
  "mixins": [ "<mod_id>.mixins.json" ]
}
```

**NeoForge（`neoforge/src/main/resources/META-INF/neoforge.mods.toml`）**：

```toml
modLoader="javafml"
loaderVersion="${neoforge_loader_version_range}"
license="${license}"

[[mods]]
modId="${mod_id}"
version="${version}"
displayName="${mod_name}"
authors="${mod_author}"
# 26.2+ 用 iconFile（方形小图标）；26.1.2 及更早用 logoFile
iconFile="icon.png"
displayURL="https://github.com/<你的账号>/<你的仓库>"
issueTrackerURL="https://github.com/<你的账号>/<你的仓库>/issues"
description='''${description}'''

# Mixin 配置必须显式声明，否则 common 里的共享 mixin 全部静默跳过
[[mixins]]
config = "<mod_id>.mixins.json"

[[dependencies.${mod_id}]]
    modId="neoforge"
    type="required"
    # 26.3 起 neoforge.mods.toml 不再写 modLoader / loaderVersion，
    # neoforge_loader_version_range 属性也一并移除，这里改用 neoforge_version。
    versionRange="[${neoforge_version},)"
    ordering="NONE"
    side="BOTH"

[[dependencies.${mod_id}]]
    modId="minecraft"
    type="required"
    versionRange="${minecraft_version_range}"
    ordering="NONE"
    side="BOTH"
```

> `iconFile`（或 26.1.2- 的 `logoFile`）/ `displayURL` / `issueTrackerURL` 全部在 `[[mods]]` 子表内；顶层放这些字段不生效。
> 26.3 另需删掉 `modLoader` / `loaderVersion` 两行（javafml 默认、版本不限）。

**历史写法变体（旧分支用，勿套新分支）**：早期模板把 `logoFile` 写成 `logoFile="assets/<mod_id>/icon.png"`，且 `description` 里直接拼反馈地址——该写法在 Forge 1.18.x 会崩（见 §3.1.5），**新工程用上面的版本**。

### 3.1.5 Forge 1.18.x 的 `logoFile` 例外

该版本 `logoFile` **不能用含 `/` 的路径**，必须写根文件名（如 `icon.png`），否则点开模组详情页崩：

```
IllegalArgumentException: Root resources can only be filenames, not paths
```

1.19.2 建议同样处理；1.20.1 及以上才可用 assets 路径。

### 3.1.6 26.3 元数据变更

- `neoforge.mods.toml` **不再写 `modLoader` / `loaderVersion`**——NeoForge 只支持 javafml，这两个字段已降级为可选、默认不限版本。因此 26.3 模板**删除了 `neoforge_loader_version_range` 属性**（`gradle.properties` 与约定脚本的 `expandProps` 同步移除）。照抄旧模板保留 `loaderVersion="[4,)"` 虽仍能构建，但已无必要。
- 26.3 依赖组件版本（供排查用）：FML `12.0.0`、EventBus `8.0.5`、ASM `9.10.1`、AccessTransformers `11.0.2`。
- 26.2 → 26.3 版本矩阵差：

| 配置项 | 26.2 | 26.3 |
|---|---|---|
| `minecraft_version` | 26.2 | `26.3` |
| `java_version` / 构建 JDK | 25 | `25`（不变） |
| `minecraft_version_range` | `[26.2, 26.3)` | `[26.3, 26.4)` |
| `neo_form_version` | 26.2-1 | `26.3-1` |
| `neoforge_version` | 26.2.0.1-beta | `26.3.0.1-beta` |
| `fabric_loader_version` | 0.19.3 | `0.19.5` |
| `fabric_version`（Fabric API） | 0.152.1+26.2 | `0.160.5+26.3` |
| Fabric Loom | 1.16.3 | `1.17.21` |
| ModDevGradle | 2.0.141 | `2.0.147` |
| Gradle Wrapper | 9.5.0 | `9.6.1` |
| MixinExtras | 0.5.3 | `0.5.4` |
| sponge-mixin | 0.17.3+mixin.0.8.7 | 不变 |

- 26.3 模板落地做法：复制 26.2 目录为骨架，**只改 4 个文件**（`gradle.properties`、根 `build.gradle`、`gradle/wrapper/gradle-wrapper.properties`、`common/build.gradle`），其余原样继承。版本来源：NeoForge 官方 MDK-26.3（`NeoForgeMDKs/MDK-26.3-ModDevGradle`）+ Fabric 官方博客《Fabric for Minecraft 26.3》+ 各 Maven 元数据。
- **已知差异**：Fabric 官方示例模板 `fabric-example-mod` 在 26.3 发布初期**仍无 26.3 分支**（默认分支 26.2），Fabric 侧版本是按官方博客 + Maven 元数据推断；Fabric 博客建议 Gradle `9.6.0`，若本地已缓存 `9.6.1`（同系列补丁版，Loom 1.17 已适配 9.6 废弃警告），模板改用 9.6.1 以避免重复下载；Loom 取值存在歧义（博客写 Loom 1.17，Maven `<release>` 已到 1.18.1，实测 `1.17.21` 可正常加载并构建）。

### 3.1.7 附魔 / 特殊描述键的本地化

附魔描述写 `enchantment.<附魔ID>.desc` 键，放进对应命名空间的 `lang/`。只有名字键（`enchantment.<id>`）时，提示型模组（EnchantmentDescriptions、more-enchantment-info 等）在物品栏 / 附魔台 tooltip 里**显示不出说明文字**，且和"附魔没注册"完全不同——游戏照常能进，只是描述缺失。

- 描述内容要按**回调代码的实际行为**写（数值、持续时间、互斥对象），不要照抄类注释里的旧稿。
- 中英文两个语言文件都要补；键集合仍要与 `en_us` 齐平（见 [04-i18n.md](04-i18n.md) §4.2）。

---

## 3.2 资源与数据规范

### 3.2.1 NeoForge 侧枚举模组资源不要用类加载器（铁律）

**违反 = 语言 / 贴图 / 资源凭空缺失且不报错。**

- ❌ `ClassLoader.getResources("assets")` 依赖 FML 模块类加载器对"目录型资源"的枚举能力。实测生产环境里 3 个含 `assets/` 的 jar 只命中 1 个，扫描结果**恒为空**。
- ✅ 走 FML 的文件视图：

```java
ModList.get().forEachModFile(modFile -> {
    JarContents contents = modFile.getContents();          // IModFile → JarContents
    contents.visitContent((path, resource) -> {            // path 形如 assets/<ns>/xxx
        byte[] bytes = resource.readAllBytes();            // 直接取内容，别再拼路径读
    });
});
```

- `ModList` 在 `ModLoader.gatherAndInitializeMods` 里于 `constructMods()` **之前**创建，模组构造期（`@Mod` 构造器 / `onInitialize`）已可用，不需要延后到 `ServerStarted`。
- `net.neoforged.fml.ModFileInfo` 已移除；`IModFile`、`IModFileInfo`、`JarContents` 仍在（FML 11.x，位于 `loader-*.jar`）。
- Fabric 侧对应写法：`FabricLoader.getInstance().getAllMods()` + `mod.findPath("assets/")`。
- ⚠ 曾有相反建议（"用类加载器 `ClassLoader.getResources("assets")`，兼容 `file:` / `jar:` 两种协议"），**以实测结论为准**：不要用类加载器目录枚举。参见 [11-verdicts.md](11-verdicts.md)。

### 3.2.2 `pack.mcmeta` 规则（进不了世界的经典原因）

**基础口径（Fabric / NeoForge 线）**：若模组不含数据包内容，**不要**在 `common` 资源里放 `pack.mcmeta`。NeoForge 会把带该文件的包当作 datapack 注册，`pack_format` 错误直接导致"数据包错误"进不了世界。

**Forge 线例外（两次实测触发）**：

- 现象：Forge 启动弹"Mod 加载警告"并提示 `无法为文件 Mod File: ...jar 加载有效的 ResourcePackInfo`，日志另有 `Missing metadata in pack mod:<modid>`。
- 判定：**模组其实加载成功**（`initialized (enabled=true)`），告警来自 Forge 为"带 `assets/` 的模组"自动创建的 resource pack 缺 `pack.mcmeta`。
- 修复：**凡是 Forge 端分支都要补最小可用 `pack.mcmeta`**（只写 `pack.pack_format` + `description`，**不写 `data` 段**）：

| MC 版本 | `pack_format` | 来源 |
|---|---|---|
| 1.18.x | `8` | 原版 `client.jar` 内 `version.json` 的 `pack_version.resource` |
| 1.19.x | `9` | 同上 |
| 1.20–1.20.1 | `15` | 同上 |
| 1.20.2–1.20.4 | `22` | 同上 |
| 1.20.5–1.20.6 | `32` | 同上 |
| 1.21.2–1.21.5 | `55` | 同上 |
| 1.21.6–1.21.8 | `64` | 同上 |

- 取值**按分支编译版本取**，不靠记忆。判定手法：解包目标版本的 `client.jar`，读 `version.json` 的 `pack_version.resource`。
- 1.21.9+ 起原版改用 `resource_major` / `resource_minor`，官方 MDK 的写法是 `min_format` / `max_format`（**照抄官方 MDK 的值最稳**）：

| MC 版本 | 写法 |
|---|---|
| 1.21.11 | `min_format:[94,1]` / `max_format:94` |
| 26.2 | `min_format:[107,1]` / `max_format:107` |
| 26.3 | `min_format:121` / `max_format:121` |

- 实测旁证：1.21.11 / 26.x 实例里的存量模组仍在用传统数字且**任意值**都能正常加载（如 `pack_format: 6`、`31`、`46`、`64`）⇒ 该字段只要存在且格式合法即可。
- **放置位置（关键）**：新建的"Fabric + NeoForge + Forge"三端分支，`pack.mcmeta` **只放 forge 模块的 `src/main/resources/`**，不要放 common —— 否则 NeoForge 端会把整包当 datapack 注册，`pack_format` 不符时报"数据包错误"进不去世界（26.x 尤其明显）。
- ⇒ **口径修正**："不含数据包内容就不要放 `pack.mcmeta`"**只适用于 Fabric / NeoForge 线**；**Forge 线必须放在 forge 模块里**，否则每次启动都有告警。

**明确需要 datapack 时，`pack_format` 必须用 datapack 格式而非 resource pack 格式**：

| MC 版本 | datapack `pack_format` |
|---|---|
| 1.21.1 | `48` |
| 1.20.1 | `15` |

- 这两档数值与 resource pack 格式**不同**（1.21.1 的 resource pack 格式是 `34`，datapack 是 `48`），混用是"数据包加载失败"的常见原因。
- 26.x 用新版 `min_format` / `max_format`；26.1.2 需查对应 datapack 版本号。

| 症状 | 更可能的根因 | 先查什么 |
|---|---|---|
| 进世界弹"数据包错误" | `pack.mcmeta` 放在 common、`pack_format` 是 resource pack 格式 | `pack.mcmeta` 所在模块与数值来源 |
| 每次启动都有 Mod 加载警告 | Forge 线缺 `pack.mcmeta` | 该 jar 内是否含 `pack.mcmeta` |
| 弹"实验性设置"确认框 | 动态注册表缺 `KnownPack`（见 §3.2.7） | 日志里的 `non-stable registries` |

### 3.2.3 语言 JSON 必须 UTF-8 无 BOM

Minecraft 资源规范要求所有 JSON 无 BOM。

```powershell
# 判定：首三字节不得为 EF BB BF
$b = [System.IO.File]::ReadAllBytes($f)
if ($b[0] -eq 0xEF -and $b[1] -eq 0xBB -and $b[2] -eq 0xBF) { "HAS BOM" } else { "OK" }
```

PowerShell 5 的 `Set-Content -Encoding UTF8`、Windows 记事本默认都会写 BOM —— 写 JSON 一律用：

```powershell
[System.IO.File]::WriteAllText($f, $text, [System.Text.UTF8Encoding]::new($false))
```

- 虽然当前 Gson 会吞 BOM、部分语言合并库也会显式 strip BOM 兜底，但**带 BOM 的源文件仍判不合格，发现即转码重存**。
- 判定手法：`node -e 'JSON.parse(...)'` 严格解析一遍；宽松解析器可能放过，不能用来自证。
- 语言内容规范见 [04-i18n.md](04-i18n.md)。

### 3.2.4 服务端资源同步类问题的部署口径

若工程带"服务端扫描打包推送资源"的机制（服务端扫 `assets` 后打包、客户端进服拉取），部署口径如下：

- **换 jar 后必须重启服务端**才重扫；客户端还要重新进服拉新 sha1 的 resource pack。
- 两路口径（"客户端本地注入包" + "服务器下发包"）下，瘦客户端（没装下游 mod）完全依赖服务器包——服务器跑旧 jar 时现象就是"有的模组有中文、新加的模组没中文"。
- **排查顺序**：先确认三端 jar 同版本，再看代码。版本不一致时，任何代码检查都是浪费。

> 部署与三端一致性台账见 [08-deploy-qa.md](08-deploy-qa.md)；语言文件规范见 [04-i18n.md](04-i18n.md)；`pack.mcmeta` 相关报错见 [10-troubleshooting.md](10-troubleshooting.md)。

### 3.2.5 资源目录结构约定

| 路径 | 放什么 | 归属模块 |
|---|---|---|
| `common/src/main/resources/assets/<mod_id>/` | 语言、贴图、模型、方块状态 | common（两端共享） |
| `common/src/main/resources/data/<mod_id>/` | 数据包内容（recipe / loot / tags） | common |
| `fabric/src/main/resources/fabric.mod.json` | Fabric 元数据 | fabric |
| `fabric/src/main/resources/<mod_id>.mixins.json` | Fabric 侧 mixin 配置 | fabric |
| `neoforge/src/main/resources/META-INF/neoforge.mods.toml` | NeoForge 元数据 | neoforge |
| `forge/src/main/resources/META-INF/mods.toml` | Forge 元数据 | forge |
| `forge/src/main/resources/pack.mcmeta` | Forge 线的最小 resource pack 元数据 | **仅 forge** |
| `*/src/main/resources/icon.png` | 图标（每个加载器模块各一份） | 各自模块 |

- **资源不要放在根 `src/main/resources/`**：MultiLoader 的每个模块各自打包，根目录的资源不会被任一加载器读到。
- **同一份资源只放一处**：放 common 就两端都能读到；放两个加载器模块各一份时，改一处忘另一处是常见漂移源。

### 3.2.6 资源枚举的替代方案对照

| 方案 | 能否枚举目录 | 适用侧 | 结论 |
|---|---|---|---|
| `ClassLoader.getResources("assets")` | ❌ 生产环境不可靠 | 两端 | **禁用** |
| `ModList.get().forEachModFile` + `JarContents` | ✅ | NeoForge / Forge | 首选 |
| `FabricLoader.getInstance().getAllMods()` + `findPath` | ✅ | Fabric | 首选 |
| 硬编码资源清单（构建期生成） | ✅（清单本身要维护） | 两端 | 资源数量固定时可用 |
| 读 `jar` 的 `ZipEntry` 列表 | ✅ | 两端 | 兜底方案，需自己处理 jar 路径 |

判定手法：把 jar 放进一个含多个模组的实例，调用枚举代码并打印命中数量；只命中 1 个或 0 个即为类加载器方案的失败表现。

### 3.2.7 KnownPack：与 `pack.mcmeta` 同症状、不同根因

- **现象**：进世界弹"实验性设置"确认框；日志有 `non-stable registries` 相关行。
- **根因**：通过数据包往**动态注册表**里加条目时，包必须在 `PackLocationInfo` 里声明 `KnownPack`，否则该注册表被判 Experimental。
- **版本分界**：**26.x 必须声明**；**1.20.1 无此问题**（该版本动态注册表机制不同）。
- **判定顺序**：先确认没有 `pack.mcmeta` 格式问题（§3.2.2），再查 `non-stable registries` 日志行。两条要分开验，因为症状相同。
- 修复手法与案例见 [10-troubleshooting.md](10-troubleshooting.md) §10.10。

---

## 3.3 产物命名与构建脚本要点

### 3.3.1 双版本号（`archivesName` 与 `archiveVersion` 叠加）

`base.archivesName` 已含版本号时，Gradle 还会再追加一次 `-<version>`，产出形如 `yourmod-1.2.0-fabric-26.3-1.2.0.jar`。

- 早期方案：`tasks.withType(Jar).configureEach { archiveVersion = "" }`（并声称 loom 的 `remapJar` 继承 `Jar`）。
- **最终修正**：只对 `Jar` 清空**不够**——Loom 的 `remapJar` 是 `RemapJarTask`，会产出 `...-1.2.0-1.2.0.jar`。必须对 **`AbstractArchiveTask`** 清空：

```groovy
tasks.withType(AbstractArchiveTask).configureEach { archiveVersion = "" }
```

- 判定手法：构建后 `ls build/libs`，看文件名里版本号出现几次；出现两次即为漏改。
- 发布物命名规范见 [01-publishing.md](01-publishing.md) §1.7。

### 3.3.2 跨版本分支的区间标签

每个版本一个独立目录，**目录名 = 分支名 = jar 区间标签**：

```
26.3 / 26.1-26.2 / 1.21.9-1.21.11 / 1.21.6-1.21.8 / 1.21.2-1.21.5
1.21-1.21.1 / 1.20.5-1.20.6 / 1.20.2-1.20.4 / 1.20-1.20.1 / 1.19.x / 1.18.x
1.16.5 / 1.12.2 / 1.7.10
```

- jar 命名 `yourmod-1.2.0-<loader>-<MC 区间>.jar`：实现方式 = 在约定插件里用分支级属性 `mc_range_label` 拼 `archivesName`，并对 `AbstractArchiveTask` 清空 `archiveVersion`。
- 三者不同步的后果：分支名与 jar 末段不一致，用户下载后无法判断适用版本。判定手法：`git branch --show-current` 与 jar 名末段并排比对。

### 3.3.3 关掉 sources / javadoc 的连带问题

约定插件不再产出 sources / javadoc 后，模块脚本里 `configurations.named('sourcesElements')` 找不到配置直接失败。改成：

```groovy
configurations.matching { it.name == variant }.configureEach { ... }
```

对不存在的变体静默跳过。**common / fabric / neoforge 三个模块脚本都要改**。

### 3.3.4 模块内 `archivesName` 覆盖要删掉

`forge` / `neoforge` 模块里原有的 `base { archivesName = "${mod_name}-forge-${minecraft_version}" }` 覆盖会产出非规范名（`Your Mod-forge-1.21.1.jar`）⇒ **必须删除该覆盖，交给约定插件统一拼**。

- 判定手法：`grep -rn 'archivesName' */build.gradle`，除约定插件外不该有第二处赋值。
- 后果：模块覆盖一旦生效，`mod_name` 里的空格会进文件名，且版本段丢失。

### 3.3.5 `processResources.expand` 会把 JSON 里的 `\n` 变成真实换行

- **现象**：`fabric.mod.json` 源文件写的是合法转义 `"description": "[EN] ...server.\n[CN] ..."`，构建产物里却变成字符串中间一个**真实换行** → 产物是非法 JSON，解析必然失败。
- **根因**：Gradle 的 `expand` 底层是 Groovy `SimpleTemplateEngine`，它把模板正文编译进一个**双引号 GString 字面量**；于是源码中的两字符 `\n` 被 Groovy 词法当作换行转义解析成真实 LF。约定脚本里的 `jsonExpandProps`（对**属性值**做 `'\n' → '\\n'`）只保护被替换进来的值，**保护不了模板正文里手写的 `\n`**。
- **判定手法**：把 jar 解开，用严格 JSON 解析器（`node -e 'JSON.parse(...)'`）过一遍全部 `.json`；Gson / 宽松解析器可能放过，不能用来自证。
- **修复**：`fabric.mod.json` 的 `description` 改**单行**，用 `" | "` 分隔双语，源码里不出现任何反斜杠转义。
- **推广结论**：凡经 `expand` 处理的 JSON（`fabric.mod.json` / `pack.mcmeta` / `*.mixins.json` / 语言文件）**正文一律不要写 `\n`、`\t`、`\u` 等转义**。TOML 侧同理，但 `neoforge.mods.toml` 的真实换行合法（用 `'''` 多行基本串）。

| 文件 | 能否含 `\n` | 说明 |
|---|---|---|
| `fabric.mod.json` | ❌ | 经 `expand`，`\n` 变真实换行 |
| `*.mixins.json` | ❌ | 同上；`refmap` 字段也经展开 |
| `pack.mcmeta` | ❌ | 同上 |
| `lang/*.json` | ❌ | 同上；文案一律单行 |
| `neoforge.mods.toml` | ✅（`'''` 内） | 多行基本串里的真实换行合法 |
| `mods.toml` | ✅（`'''` 内） | 同上 |

> `expand` / `processResources` 的完整配置见 [06-build.md](06-build.md)；跨版本打包格式差异见 [07-porting.md](07-porting.md)；该坑的真实案例见 [10-troubleshooting.md](10-troubleshooting.md)。

---

## 3.4 元数据类故障排查表（症状 → 根因 → 判定 → 修复）

元数据类故障的共同特征：**游戏能启动，但模组不加载 / 不显示 / 显示错**，且日志里往往没有直接的"这是元数据错误"字样。所以排查要按"先确认加载、再确认解析、最后确认字段"的顺序，不要从界面现象直接猜字段。

### 3.4.1 排查顺序（固定四步）

| 步骤 | 查什么 | 判据 |
|---|---|---|
| ① 是否加载 | 日志里该 modId 是否出现 `initialized (enabled=true)` | 没有 → 元数据解析失败，跳到 ②；有 → 问题在字段或界面 |
| ② 元数据是否被解析 | grep `not a valid mod file` / `Missing metadata` / `ParsingException` | 命中 → 按 §3.1.3 / §3.2.2 修 |
| ③ 字段是否落位 | 解包 jar 读元数据，看键是否在 `[[mods]]` 子表内 | 键在顶层 → 移进子表 |
| ④ 界面读的是哪个字段 | 按 §3.1.2 的界面对照表确认 | 字段与界面不匹配 → 改字段 |

### 3.4.2 症状对照表

| 症状 | 根因 | 判定手法 | 修复 |
|---|---|---|---|
| 模组完全不加载，崩溃报告指向渲染模组 | 元数据解析失败（`description` 真实换行 / JSON 非法） | grep `not a valid mod file` | 描述改单行；严格 JSON 解析产物 |
| 列表里没有图标 | 字段缺失或放错层级 | 解包读元数据，确认 `iconFile` 在 `[[mods]]` 内 | 补字段 / 移层级 |
| 图标跑到右侧大图、左侧列表空 | 26.2+ 仍用 `logoFile` | 日志有 `uses the deprecated 'logoFile'` | 改 `iconFile` |
| 只有 Catalogue 显示问号，别处正常 | 图标边长 > 256 | 读 PNG IHDR 的宽高 | 缩到 128×128 |
| 没有主页 / 反馈按钮 | `displayURL` / `issueTrackerURL` 在顶层 | 解包读 TOML 的键位置 | 移进 `[[mods]]` 子表 |
| Mod Menu 链接区为空 | 用了 `contact` 里的自造键 | 检查 `custom.modmenu.links` 是否存在 | 改走 `custom.modmenu.links` |
| 前置模组没有"前置"徽章 | 缺 `library` badge | 检查 `custom.modmenu.badges` | 加 `["library"]` |
| 下游模组从列表顶层消失 | 误写 `parent`（模块归属语义） | 检查 `custom.modmenu.parent` | 删除该字段 |
| 每次启动都有 Mod 加载警告 | Forge 线缺 `pack.mcmeta` | 看 jar 内是否含 `pack.mcmeta` | 在 forge 模块补最小版 |
| 进世界弹"数据包错误" | `pack.mcmeta` 放在 common / `pack_format` 用错格式 | 检查所在模块与数值 | 移到 forge 模块；改用 datapack 格式 |
| 进世界弹"实验性设置"确认框 | 动态注册表缺 `KnownPack` | 日志有 `non-stable registries` | 声明 `KnownPack` |
| 语言键原样显示 | 语言文件缺键 | 与 `en_us` 求对称差 | 补齐键（见 [04-i18n.md](04-i18n.md)） |
| 改了 modId 但游戏里还是旧值 | `processResources` 命中 `UP-TO-DATE` | 解包读产物而非看源码 | `--rerun-tasks` 重跑 |
| 构建失败 `Configuration with name 'sourcesElements' not found` | 移除 sources 变体后脚本仍无条件引用 | 看堆栈指向 `configurations.named` | 改 `configurations.matching {...}` |

### 3.4.3 解包核对的固定命令

```bash
# 一次性核对产物里的元数据（不依赖源码）
unzip -p <jar> fabric.mod.json | python -m json.tool
unzip -p <jar> META-INF/neoforge.mods.toml
unzip -p <jar> META-INF/mods.toml          # Forge 1.20.1 及更早
unzip -l <jar> | grep -E 'pack.mcmeta|icon.png|\.mixins\.json'
```

- **判定原则：看产物，不看源码**。源码正确而产物错误，说明 `processResources` 没重跑或被别的插件改写。
- 严格 JSON 校验：`unzip -p <jar> fabric.mod.json | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{JSON.parse(s);console.log("OK")})'`。

### 3.4.4 两个"看起来一样、根因不同"的配对

| 配对 | 区分手法 |
|---|---|
| `pack.mcmeta` 格式错误 ↔ 缺 `KnownPack` | 前者报"数据包错误"，后者弹"实验性设置"；前者无 `non-stable registries` 行 |
| 图标字段错 ↔ 图标尺寸超限 | 前者影响所有列表界面，后者只影响 Catalogue（其它界面正常） |
| 元数据解析失败 ↔ 依赖缺失 | 前者 `initialized` 行缺失，后者有 `initialized` 但有 `Missing or unsupported mandatory dependencies` |

---

## 3.5 移植时如何保护既有身份（通用规则）

移植 / 扩建版本线时，最容易出事的是**身份被"顺手统一"**。以下规则不依赖任何具体项目，按它执行即可。

### 3.5.1 移植必须原样继承的东西

| 类别 | 具体项 | 为什么不能改 |
|---|---|---|
| 机器身份 | `mod_id`、包名根、maven `group` / `artifactId` | 改了就是另一个模组：下游依赖断、平台是两个项目、数据包目录要分叉 |
| 展示身份 | `mod_name`、`displayName`、jar 内 `Specification-Title` | 品牌名，跨版本改名等于换产品 |
| 注册身份 | 注册表 ID、附魔 / 物品 / 方块 ID、数据包路径 | 存档里的方块与物品会变成"未知方块"，玩家数据丢失 |
| 文案身份 | 语言键名、`description` 正文、游戏内附魔描述 | 其中相当一部分是作者自己写的措辞，**改一个字也算越界** |
| 配置身份 | 配置文件名、配置键名、默认值 | 玩家已有配置文件失配，表现为"设置被重置" |
| 协议身份 | 网络包 ID / 通道名 | 两端版本不匹配时静默不同步 |

### 3.5.2 允许因版本适配而不同的东西

只有以下几类允许不同，**且必须能写出适配理由**：

- 加载器 API（Fabric / NeoForge / Forge 的入口、事件、注册 API）
- Minecraft 版本 API（方法改名、类移动、注册表机制）
- 注册表 / 渲染写法（`RenderType`、`GuiGraphics`、数据生成器）
- 元数据字段名（`logoFile` → `iconFile`、`pack_format` 数值）
- Gradle / 插件版本（见 [02-environment.md](02-environment.md) §2.4.2）

写不出理由的差异 = 漂移，要改回基准线。

### 3.5.3 移植前必做的身份盘点

```bash
# 列出所有会被"顺手统一"的身份键，逐条记录基准值
grep -rnE '^(mod_id|mod_name|mod_author|license|group|version)=' --include='gradle.properties' .
```

盘点表（每个模组一行，写进内部决策记录，**不要写进仓库里的公开文档**）：

| 键 | 基准值 | 各版本线是否一致 | 不一致的理由 |
|---|---|---|---|
| `mod_id` | | | 不允许不一致 |
| `mod_name` | | | 不允许不一致 |
| 包名根 | | | 不允许不一致 |
| 配置文件名 | | | 不允许不一致 |
| 语言键前缀 | | | 不允许不一致 |

### 3.5.4 移植后的身份复核

| 检查 | 命令 / 手法 | 通过判据 |
|---|---|---|
| 元数据 `modId` 一致 | 解包各版本 jar 读元数据 | 与基准逐字相同 |
| 包名根一致 | `unzip -l` 看 class 路径前缀 | 与基准逐字相同 |
| 语言键集合一致 | 与基准的 `en_us` 求对称差 | 差集为空（新增键需显式记录） |
| 配置键一致 | 比对默认配置文件 | 键名与默认值均相同 |
| 展示名一致 | 展开 displayName | 首词与基准相同 |

> 完整的多版本移植策略（L1/L2/L3 判据、触点差分、探针口径）见 [07-porting.md](07-porting.md)；仓库与分支命名见 [09-git-upload.md](09-git-upload.md)。

---

### 3.5.5 移植时最常见的三类身份破坏（按发生频率）

| # | 破坏方式 | 症状 | 判定手法 | 修复 |
|---|---|---|---|---|
| ① | 把 `mod_name` "统一"成小写 | 平台文件名变成机器名；游戏内列表首字母小写 | 展开 displayName 看首词 | 恢复基准展示名，重建重传 |
| ② | 改了包名根 | 下游依赖断；语言键前缀与数据包路径分叉 | 比对 jar 内 class 路径前缀 | 改回基准包名 |
| ③ | 顺手改了语言文案措辞 | 玩家看到的文字与其它版本线不一致 | 与基准 `en_us` 逐键求差 | 恢复原文；确需改动先说明原文 / 改法 / 理由 |
| ④ | 配置键改名 | 玩家已有配置失配，表现为"设置被重置" | 比对默认配置文件的键名 | 加一次性幂等迁移，或改回原名 |
| ⑤ | 注册 ID 前缀改了 | 存档里的方块 / 物品变"未知" | 比对注册表导出 | 改回基准 ID；已发布的无法补救 |

- ③ 的特殊性：这类文字**相当一部分是作者自己写的措辞**，与本次版本适配无关的一律照抄基准线，**改一个字也算越界**。确需改动，先说明「原文 / 改法 / 理由」并取得同意，再动。
- ⑤ 的不可逆性：注册 ID 一旦进入玩家存档，改 ID 等于让存档里的对象消失。这条没有任何"迁移"手段能补救，只能靠移植时不改。

### 3.5.6 元数据的"跨版本分支"处理原则

同一条元数据文件要覆盖多个 MC 版本时，按**字段的版本分界**分支，而不是按"整份文件复制一份"：

| 字段 | 分界 | 处理 |
|---|---|---|
| `iconFile` / `logoFile` | 26.2 | 26.2+ 写 `iconFile`；26.1.2- 写 `logoFile` |
| `bannerFile` | 26.2 | 26.2+ 可选；更早版本无此字段 |
| `modLoader` / `loaderVersion` | 26.3 | 26.3 起删除；更早保留 |
| `pack.mcmeta` 数字格式 | 1.21.9 | 更早写 `pack_format`；1.21.9+ 写 `min_format` / `max_format` |
| `pack.mcmeta` 存在性 | — | Fabric / NeoForge 线不放；Forge 线必放 |
| `[[mixins]]` | 1.20.1（FG6） | FG6 走 jar manifest `MixinConfigs`，TOML 无 `[[mixins]]` |

- **不要为每个版本复制整份元数据**：复制会导致"改一处漏三处"。用 `processResources.expand` 的属性做条件化，或按模块分文件。
- 判定手法：`diff` 各版本线的元数据文件，差异应只落在上表列出的字段上；出现其它差异即为漂移。

---

## 3.6 元数据 / 工程自检清单（每次构建与发布前抽查）

- [ ] 每个待发布模组：`icon.png` 为 128×128（读 IHDR），fabric / neoforge 两端元数据字段齐全（用 §3.1.4 模板逐项对）。
- [ ] 按目标版本选对图标字段：26.2+ 用 `iconFile`；26.1.2- 用 `logoFile`；Forge 1.18.x 的 `logoFile` 不带 `/`。
- [ ] `displayURL` / `issueTrackerURL` 在 `[[mods]]` 子表内（解包读 TOML 确认位置，不只 grep 键名）。
- [ ] `description` 单行、无转义；经 `expand` 的 JSON 正文无 `\n` / `\t` / `\u`。
- [ ] 语言 JSON 全部 UTF-8 无 BOM（读首三字节，不得为 `EF BB BF`）。
- [ ] mixin 配置两端成对；新 mixin 的注入点已对过字节码（见 [05-mixin-api.md](05-mixin-api.md)）。
- [ ] `pack.mcmeta` 按加载器线决定去留（Fabric / NeoForge 线删、Forge 线补最小版）。
- [ ] 26.x 数据包条目已声明 `KnownPack`（否则弹"实验性设置"）。
- [ ] 模块内无多余的 `archivesName` 覆盖；`AbstractArchiveTask` 的 `archiveVersion` 已清空。
- [ ] jar 文件名为 `<mod_id>-<version>-<loader>-<MC 或区间>.jar`，无重复版本号。
- [ ] 解包 jar 后用严格 JSON 解析器验证全部 JSON，并逐语言文件与 `en_us` 求键集合差。
- [ ] 资源枚举走 FML 文件视图 / Fabric `getAllMods`，无 `ClassLoader.getResources("assets")` 残留。
- [ ] 移植分支的身份盘点表已填，且 `mod_id` / `mod_name` / 包名根与基准逐字一致。
