# 10 排障案例库

> 管什么：一套通用的 Debug Flow、一张按症状查的索引、以及全部实战案例（每个都写成「现象 → 根因 → 判定手法 → 修复」）。
> 本文件与 `references/` 其余模块同属一个技能包，**没有上位总纲**；结论冲突时以一手证据为准。本文件给现场手法、错误原文、数值表与可复用的判定路径。

**用法**：先按 §10.1 走通用顺序；症状明确时直接跳到 §10.2 索引里的对应条目。

---

## 10.1 通用 Debug Flow

1. **先读完整日志**：看完整 `latest.log` / 崩溃前输出，grep 这几个关键串：`not a valid mod file`、`ParsingException`、`Invalid modId found`。
2. **确认模组是否真的出现在 Mod List**：不在列表 = 元数据解析失败，先修元数据，别去查代码。
3. **按元数据链依次核对**：TOML 换行 → `[[mods]]` 字段位置 → **modId 是否含连字符 / 大写 / 空格** → `pack.mcmeta` → Mixin remap → 图标与链接。字段全集见 [03-metadata.md](03-metadata.md)。
4. **重构建前清掉旧 jar**（指实例 `mods/` 里的旧同名 jar），避免"改了没生效"的假象干扰验证。
5. **Java 版本不匹配**时，先核对 [02-environment.md](02-environment.md) §2.1 的 Java 分档再重新构建。
6. **忽略崩溃后的清理噪音**：`CancellationException` / `IOException: Stream closed` 是崩溃后的收尾动作，**不是独立根因**；要往**更早**的日志里找 `Invalid modId` 这类真正的第一现场。
7. **构建成功 ≠ 游戏在跑新代码**。`build/libs` 里的 jar **不会**自动进游戏实例，必须手动拷进 `<版本>/mods/` 并删掉旧同名 jar。排查"改了还是崩"时，**第一步永远是比对 `build/libs/*.jar` 与 `mods/*.jar` 的时间 / 体积**，而不是重读代码。实测踩过：修复产物早就出来了，`mods` 里躺的还是几小时前的旧 jar，连崩三次还在查代码。
8. **"能进游戏但功能整体失效" → 先查 mixin 有没有被加载**（§10.10）。
9. **"语言 / 贴图没加载，但游戏能进" → 先查资源扫描计数**（§10.12）。
10. **"我这么配是合法的，但库 / 加载器说它非法，还直接把启动干掉" → 先去原版字节码确认那条约束是否真实存在**（§10.23）。
11. **"注入没生效，但启动日志一切正常" → 先查 `@At` 里的目标字符串**（§10.11）。
12. **改一处先分析根因**；不擅自改用户没要求的配置。

**两条贯穿全库的原则**（细节见 §10.29）：

- **会叫的失败好查，不叫的失败才难**：静默失效只能靠主动核对（资源计数、方法描述符、语言键集合）发现。
- **取证优先**：原版字节码、解包 jar、maven 元数据、实例 JSON 才是证据；文档、印象、日志摘要都可能骗人。

### 10.1.1 分诊表（先按失败形态分流）

报错到手时，先判断它属于哪一类，能省掉一半排查时间。

| 失败形态 | 典型表现 | 第一步查 | 第二步查 |
|---|---|---|---|
| **加载期硬失败** | 启动中断、模组不出现、`Invalid modId` | 元数据解析（§10.3 / §10.4） | 依赖声明（§10.8） |
| **加载期静默失效** | 游戏能进、功能整块不工作 | mixin 是否被加载（§10.10） | `@At` 目标串（§10.11） |
| **资源类静默失效** | 语言 / 贴图 / 图标不对，但不报错 | 资源扫描计数（§10.12） | 解包确认文件在包里 |
| **渲染期崩** | 进世界 / 渲染阶段崩 | 崩溃类名是不是第三方包（§10.21） | class file version 对照 |
| **构建期失败** | 任务报错、构建中断 | 报错原文（§10.30） | 工具链档位与插件互斥（§10.25） |
| **构建"成功"但产物不对** | 改了没效果、名字带两个版本号 | 产物 mtime 与内容（§10.14） | `build-logic` 是否重跑 |
| **工具"什么都没干"** | 无输出、无报错、退出码 0 | 系统盘余量（§10.26） | 管道退出码（§10.20） |
| **行为正确但显示反了** | 旋转 / 朝向 / 位置错 | 渲染管线里的变换（§10.28） | 模型空间正面对照 |

**分流原则**：先判"加载期 / 运行期 / 构建期 / 渲染期"，再判"会叫 / 不叫"。这四 × 二 个格子一填，能查的案例就只剩两三条。

---

## 10.2 症状索引（按症状查）

| 症状 | 案例 |
|---|---|
| 模组完全不加载 / 不是有效 mod 文件 | §10.3 元数据换行 |
| 模组不显示、`Invalid modId` | §10.4 modId 非法 |
| 模组列表图标跑到右侧大图、左边列表空 | §10.5 图标槽位 |
| 图标显示问号（只在部分模组列表 UI 里） | §10.6 图标尺寸 |
| 主页 / 反馈按钮点不动 | §10.7 字段位置 |
| Fabric 侧进服报晦涩错误 | §10.8 Fabric API 声明 |
| 进不了世界、报数据包错误 | §10.9 pack.mcmeta |
| Forge 启动告警 `ResourcePackInfo` / `Missing metadata` | §10.9 pack.mcmeta（Forge 例外） |
| 能进游戏但功能整块不工作 | §10.10 mixin 未加载 / §10.11 注入点失效 |
| 语言 / 贴图缺失但不报错 | §10.12 资源扫描 |
| 多语言失效（切成中文仍英文） | §10.13 literal 兜底 |
| 改代码没效果 / `UP-TO-DATE` / 产物名双版本号 | §10.14 旧产物与增量 |
| 构建卡死 / javadoc 拖垮整条构建 | §10.15 javadoc |
| `gradlew` 找不到主类 | §10.16 wrapper 主类 |
| 依赖报"无法访问 `class_XXXX`" | §10.17 `modImplementation` |
| ASM 版本冲突 | §10.18 ASM |
| 构建报"编码 GBK 的不可映射字符" | §10.19 编码 |
| `git push` 永久挂起 | [09-git-upload.md](09-git-upload.md) §9.3 |
| 上传 403 | [09-git-upload.md](09-git-upload.md) §9.5.2 |
| "任务完成 exit 0"但实际失败 | §10.20 管道退出码 |
| 第三方模组崩溃被当成自己的 bug | §10.21 误归因 |
| 给测试实例装模组包时踩的两个坑 | §10.22 取包 |
| 库的 fail-fast 校验把别的模组拖死 | §10.23 库的假设 |
| 多加载器 jar 在 Forge 上启动即崩（Fabric / Neo 正常） | §10.24 Mixin 兼容级 |
| Forge 高版本构建连环报错（FG6 → FG7） | §10.25 工具链互斥 |
| 构建 / 上传"无输出、无报错地失败" | §10.26 磁盘写满 |
| 进世界弹"此世界使用实验性设置" | §10.27 KnownPack |
| 弹射物模型朝向与飞行方向相反（俯仰却是对的） | §10.28 镜像空间里的 Y 轴旋转反号 |
| **报错原文查不到对应条目** | §10.30 错误原文速查 |
| **不知道用哪个工具取证** | §10.31 取证工具箱 |

---

## 10.3 元数据换行 → 整个 jar 判为无效 mod 文件

- **现象**：`ParsingException: Invalid newline in basic string`；模组完全不加载；随后一堆 `Cowardly refusing to send event` + 第三方渲染模组崩溃。
- **根因**：`description` 基础字符串里出现**真实换行**（来自 `\n` 转义或 `gradle.properties` 的多行写法）。
- **判定手法**：先 grep `not a valid mod file`——**崩溃报告的先导往往是第三方渲染模组，根因在它之前的日志里**；看到先导就顺着往前翻，不要停在最后那个异常。
- **修复**：把反馈地址等附加信息并入同一行，去掉 `\n`。字段写法与 `expand` 陷阱详见 [03-metadata.md](03-metadata.md) §3.3.5、§3.1.2。
- **预防**：`gradle.properties` 的 `description` **必须单行**；经 `processResources.expand` 的 JSON / TOML 正文**禁写** `\n`、`\t`、`\u`。

## 10.4 modId 非法 → `Invalid modId found`

- **现象**：FML 抛 `InvalidModFileException`，整个模组不加载、启动失败；日志尾部连片 `CancellationException` / `IOException: Stream closed`。
- **根因**：`modId` 含连字符 / 大写 / 空格，或不匹配正则 `^(?=.{2,64}$)[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`。
- **判定手法**：**真正的根因在更早日志的 `Invalid modId found` 行**，后面的清理噪音不是独立根因。抓日志时按"第一条 `Invalid modId found`"定位，不要按"最后一条异常"定位。
- **修复与全链路同步**：见 [01-publishing.md](01-publishing.md) §1.5（modId 会同时驱动 fabric `id`、TOML `modId`、包名、资源目录、语言键前缀，改一处必须全改）。
- **附带坑**：改完 `gradle.properties` 后构建若显示 `UP-TO-DATE`，需 `--rerun-tasks`，否则 jar 里仍是旧 modId（见 §10.14）。

## 10.5 图标跑到右侧大图、左侧列表空

- **现象**：模组列表里图标出现在右侧横幅位置，左侧小图标位置空白。
- **根因**：26.2+ 把 `logoFile` 当废弃字段，1:1 方图被塞进右侧横幅槽位。
- **判定手法**：日志里搜 `uses the deprecated 'logoFile' property`——有这条警告基本就确诊。
- **修复**：改 `iconFile="icon.png"`。槽位规则与版本分界详见 [03-metadata.md](03-metadata.md) §3.1.3。

## 10.6 图标显示问号（只有强校验的列表 UI 出现）

- **现象**：其它模组列表里图标正常，**只有带硬校验的模组列表 UI 显示问号**。
- **根因**：图标超过 **256×256** 或非正方形，强校验的列表 UI（`Branding.ICON` 那一层）抛 `InvalidBrandingImageException`，**且不报错进日志**（静默失效）。
- **判定手法**：**读 PNG 的 IHDR**——偏移 16 / 20 起各 4 字节大端是宽、高，自己算实际像素；**别信看图软件的缩略图**（它可能已经缩放过）。
- **修复**：统一 **128×128** 正方形。

## 10.7 主页 / 反馈按钮点不动

- **现象**：模组信息页的 Homepage / Issues 按钮灰掉或点不动。
- **根因**：`displayURL` / `issueTrackerURL` / 图标字段写在 TOML **顶层**，不在 `[[mods]]` 子表内。
- **判定手法**：解包 jar 看 `mods.toml` / `neoforge.mods.toml` 的层级，确认这些键的缩进归属。
- **修复**：把 `displayURL` / `issueTrackerURL` / 图标字段挪进 `[[mods]]`。

## 10.8 Fabric API 缺失报晦涩错误

- **现象**：Fabric 侧启动或进服时报一条看不出因果的错误（类找不到 / 入口点失败），而不是"缺少前置"这种直白提示。
- **根因**：`implementation` 不打包 Fabric API，且 `fabric.mod.json` 缺 `depends` 声明 → 运行期才炸。
- **修复**：显式声明 `depends.fabric-api`。**注意措辞**：声明不会给"友好提示"，是加载器**直接拒绝启动**；它的价值是让失败发生在加载期而不是运行期。
- 依赖声明写法见 [06-build.md](06-build.md) §6.4.1。

## 10.9 pack.mcmeta：两种相反的表现

同一个文件在两条版本线上要求**正好相反**，这是最容易一刀切错的地方。

| 线 | 现象 | 处理 |
|---|---|---|
| NeoForge | 不带数据包内容却放了 `pack.mcmeta` → "数据包错误"、进不了世界 | **删掉** |
| Forge | 带 `assets/` 的模组缺 `pack.mcmeta` → 每次启动告警 `failed to load a valid ResourcePackInfo` / `Missing metadata in pack <modid>:...`（**模组其实加载成功**，表现为"Mod 加载警告"弹窗） | **每条 Forge 分支都要补最小 `pack.mcmeta`** |

最小内容只有 `pack.pack_format` + `description`，**不写 `data` 段**。`pack_format` 按目标版本取原版 `client.jar` 里的 `pack_version.resource`：

| MC 版本 | `pack_format` |
|---|---|
| 1.18.x | 8 |
| 1.19.x | 9 |
| 1.20 – 1.20.1 | 15 |
| 1.20.2 – 1.20.4 | 22 |
| 1.20.5 – 1.20.6 | 32 |
| 1.21.2 – 1.21.5 | 55 |
| 1.21.6 – 1.21.8 | 64 |

**放置位置也是铁律**：**只放 forge 模块**；放 `common` 会害 NeoForge 端报"数据包错误"（common 的产物会被两端各取一次）。详见 [03-metadata.md](03-metadata.md) §3.2.2。

## 10.10 能进游戏但功能整体失效 → 先查 mixin 是否被加载

- **典型症状**：自定义内容（附魔 / 方块 / 物品 / 维度）没进注册表、创造栏找不到、数据驱动内容没生效、事件回调全部报"未注册"。游戏能进，日志看着正常。
- **判定手法**：
  1. 核对两端元数据里的 mixin 声明是否**成对**（fabric 侧 `fabric.mod.json` 的 `mixins` 与 forge 侧 `mods.toml` / manifest 的 `MixinConfigs`）。
  2. 在日志里搜 `Mixin Subsystem` 附近，看本模组的 `*.mixins.json` 是否出现在被加载列表里。
  3. 用 `javap -p` 检查目标类是否真的被改写（见 §10.11）。
- **根因**：加载器**不会**自动发现 jar 里的 `*.mixins.json`；MultiLoader 迁移后 `common` 的 mixin 两端共用，**必须两侧都声明**。漏一侧的表现就是"该侧全部静默失效"。
- **两端声明位置对照（最容易漏的地方）**：

  | 加载器 | 声明位置 | 备注 |
  |---|---|---|
  | Fabric | `fabric.mod.json` 的 `"mixins"` 数组 | 每一项是配置文件名，**不含 `.json` 后缀** |
  | Forge 1.20.1（FG6） | jar manifest 的 `MixinConfigs` 属性 | `mods.toml` 里**没有** `[[mixins]]` 段 |
  | Forge 1.20.2+ / NeoForge | `mods.toml` / `neoforge.mods.toml` 的 `[[mixins]]` 段 | 段名与字段随版本线有差异 |

  只要有一端没写，那一端的 mixin 就**不会加载**，而日志里通常**没有任何报错**——这是本案例最容易被误判成"代码写错了"的原因。
- **判定补充**：如果日志里 `Mixin Subsystem` 段落列出了配置名，但功能仍不生效，问题就已经从"配置没加载"转到 §10.11（目标串失配）。
- 两端声明模板见 [05-mixin-api.md](05-mixin-api.md) §5.1.1。

## 10.11 注入没生效但日志正常 → `@At` 目标串失配

- **现象**：`@Inject` / `@Redirect` 全部静默失效，日志里既没有 `Injection failure` 也没有异常。
- **根因**：`@At` 的目标串（`target` / `value`）是**字符串字面量**，与实际字节码对不上时**不报错**，只是回调永不执行。`require = 1` 能把它变成显式失败，但很多人不写。
- **判定手法**：`javap -p -c -s` 反汇编目标方法，逐字节核对目标串；扫描源码时**先排除注释里的死串**（注释里出现过的旧目标串极易被复制进注解）。
- **修复**：按真实字节码改目标串；两端方法名不同时注解里**同时列出**并加 `require = 1`。
- 完整机理、包搬迁导致的实测案例见 [05-mixin-api.md](05-mixin-api.md) §5.2.2。

## 10.12 语言 / 贴图没加载但能进游戏 → 查资源扫描计数

- **判定顺序**：
  1. 解包 jar，确认 `assets/<modid>/...` 是否真的在里面（先排除"没打进包"）。
  2. 对照另一加载器同一次启动的扫描数字。
  3. 检查扫描实现是否依赖**类加载器目录枚举**（这种写法在 jar 环境里天然失效）。
- **关键数字**：日志里"扫描了 N 个模组的 M 个命名空间，收集到 K 个资源文件"——**`M = 0` 或 `K = 0` 基本就是扫描器口径问题**，不是资源文件没打进 jar。
- 正确 API（FML 文件视图 `ModList...getContents().visitContent(...)`）见 [03-metadata.md](03-metadata.md) §3.2.1。
- **服务端资源同步类场景**（模组把语言 / 贴图从服务端下发给客户端）：换 jar 后**必须重启服务端**才会重扫；瘦客户端依赖服务端下发的内容，服务端跑旧 jar 的典型现象是"老模组有中文、新加的那批没有"。

## 10.13 多语言失效（切中文仍英文）

- **现象**：语言切到中文，界面仍是英文；或部分键中文、部分键英文。
- **根因**：用 `Component.literal(写死文案)` 顶替可翻译组件，或语言键前缀与 modId 不一致导致查不到。
- **判定手法**：解包 jar，把 `lang/zh_cn.json` 的键集合与代码里 `translatable(...)` 的键求差集；差集里的键就是"永远不会翻译"的。
- **修复**：玩家消息一律走 `Component.translatable(...)`；**禁止用 `literal` 顶替**。
- 完整复盘与取舍表见 [04-i18n.md](04-i18n.md) §4.3。

## 10.14 改了没效果 / `UP-TO-DATE` / 产物名双版本号

| 现象 | 根因 | 处理 |
|---|---|---|
| 改了 `gradle.properties` 但 jar 里还是旧值 | Gradle 增量，`processResources` 未重跑 | `--rerun-tasks` |
| 游戏里还是旧行为 | 实例 `mods/` 里是旧 jar | 比对时间 / 体积；部署新 jar 并删被替换的旧 jar |
| 产物名出现两次版本号（`...-1.2.0-1.2.0.jar`） | 只对 `Jar` 清 `archiveVersion`，漏了 `RemapJarTask` | 改为 `tasks.withType(AbstractArchiveTask).configureEach { archiveVersion = "" }` |
| 修完仍拿到双版本号产物 | 跑的是改动前的构建 | 改完 `build-logic` **必须重跑**，别信旧产物 |
| 关了 sources / javadoc 后构建失败 `Configuration with name 'sourcesElements' not found` | `configurations.named()` 对不存在的变体报错 | 改 `configurations.matching { it.name == variant }.configureEach { ... }` |
| 模块产物名非规范（`<展示名>-forge-1.21.1.jar`） | 模块内 `base { archivesName = ... }` 覆盖了约定 | 删掉该覆盖，交给约定插件 |

**判定要点**：产物名双版本号的根因永远是"有第二个任务也在拼 `archiveVersion`"，所以 `AbstractArchiveTask` 这一层必须统一处理，不能只处理 `Jar`；`sourcesElements` 报错则是"按名字取变体"的写法太硬。

## 10.15 javadoc 在 Java 25 下卡死整个构建

- **现象**：`withJavadocJar()` 默认开 doclint 严格模式；中文注释 + record / Mixin / 枚举常量缺 `@param` / `@return` 直接报 error 并 fail，把 `:<loader>:jar`、`publishToMavenLocal` 一起拖垮（表现为"构建卡死"，实际是 javadoc 任务失败后整条链路中断）。
- **根因**：doclint 把"注释里出现中文且缺标签"判成 error，而 `javadoc` 任务在 `jar` 之前。
- **修复**：`options.addStringOption('Xdoclint:none', '-quiet')`（只是把 error 降级为 warning）。
- **更彻底**：不产出 javadoc jar——发布到平台不需要它，构建时间还省一大截。

## 10.16 `./gradlew` 报"找不到主类 GradleWrapperMain"

- **现象**：`Error: Could not find or load main class org.gradle.wrapper.GradleWrapperMain`，任何任务都跑不起来。
- **根因**：wrapper 的 jar 缺失（常见于 `.gitignore` 把 `gradle/wrapper/gradle-wrapper.jar` 也忽略了，或克隆时被过滤），部分 shell 环境也会触发。
- **修复**：直接用本地**已缓存的发行版二进制**，绕过 wrapper：

  ```bash
  ~/.gradle/wrapper/dists/gradle-<版本>-bin/<hash>/gradle-<版本>/bin/gradle --version
  ```

- **预防**：`.gitignore` 里保留 `!gradle/wrapper/gradle-wrapper.jar`（见 [09-git-upload.md](09-git-upload.md) §9.1.2）。

## 10.17 "无法访问 `class_XXXX`"

- **现象**：编译期报 `cannot access class_XXXX` / 找不到 intermediary 名。
- **根因**：Fabric 线必须用 **`modImplementation`** 引 fabric-loader / fabric-api，用 `implementation` 拿到的不是重映射后的产物，intermediary 名未重映射。
- **修复**：把依赖声明改成 `modImplementation`；不要用 `implementation` + 手工加映射这种绕法。
- 依赖声明对照见 [06-build.md](06-build.md) §6.4。

## 10.18 ASM 版本冲突

- **现象**：Mixin / 加载器在启动期抛 ASM 相关的 `IllegalArgumentException`（不支持的 class file version）或 `NoSuchMethodError`。
- **根因**：1.21.8 的 NeoForm 硬钉 `org.ow2.asm` **9.6**，而 sponge-mixin 0.17.x 会拉 **9.8**，两者混用。
- **修复**：`resolutionStrategy.force` 统一回 **9.6**：

  ```groovy
  configurations.all {
      resolutionStrategy.force 'org.ow2.asm:asm:9.6', 'org.ow2.asm:asm-tree:9.6', 'org.ow2.asm:asm-analysis:9.6', 'org.ow2.asm:asm-commons:9.6', 'org.ow2.asm:asm-util:9.6'
  }
  ```

- **判定手法**：`gradlew :<模块>:dependencies --configuration runtimeClasspath` 抓 ASM 的实际选中版本，别按声明版本猜。

## 10.19 构建报"编码 GBK 的不可映射字符"

- **现象**：`javac` 报 `编码GBK的不可映射字符`（或 `unmappable character`），编译失败。
- **根因**：`javac` 按**平台默认编码**（中文 Windows 上是 GBK）读 UTF-8 源码里的中文注释。
- **修复**：`compileJava.options.encoding = 'UTF-8'`（所有 `JavaCompile` 任务统一设）；**未设置前，Java 注释一律用英文**（这是规避而非解决）。
- **预防**：编码属于工程级约定，新分支从模板起就要带，别等报错再补。

## 10.20 管道退出码不可信（两次"成功"实为 FAILED）

- **现象**：命令报告"已完成"且 `exit 0`，实际整条构建 / 上传是 FAILED。
- **根因**：`git push ... | tail` 的退出码是 `tail` 的；`gradlew ... | grep | head` 同理——**管道的退出码来自最后一个命令**，与前面是否成功无关。
- **实测**：两次"报告已完成且 exit 0、实际 FAILED"，其中一次是 `-I` 的 init 脚本路径含空格被 shell 词分割 → **脚本根本没加载**，而管道掩盖了失败。
- **修复（铁律）**：一律改为**带外回读验证**：
  - 推送：`git ls-remote --heads origin <分支>` 的哈希 vs `git rev-parse HEAD`；
  - 上传：按返回的 File ID / version id 查回；
  - 构建：抓完整输出按 `BUILD SUCCESSFUL` 判定，不要 `| tail`；
  - 命令里**路径参数逐个加引号**。

## 10.21 第三方模组崩溃被误判成本模组的 bug

- **现象**：`UnsupportedClassVersionError: <第三方包>/utils/Match class file version 69.0 (Java 25) > 65.0 (Java 21)`，启动 / 渲染阶段崩。
- **根因**：该实例装的是**为更高 MC 版本 / Java 25 编译**的第三方 jar，却被放进 **Java 21** 的实例 ⇒ **装载错误**，不是本模组的问题。
- **判定手法（关键）**：
  1. **先看崩溃类名是不是第三方包名**——包名前缀不是自己的 `com.example.mod` 就直接怀疑装载；
  2. 在同一份日志里核对本模组的行是否正常：`Found valid mod file ... versions {...}`、mixin 注入成功、`initialized (enabled=true)`；
  3. class file version 的对应关系：`65.0` = Java 21、`69.0` = Java 25，报错里两个数字一并给出时，就是"用高版本编译、装进低版本运行时"。
- **处置**：把错版本的 jar 从该实例移出（若该模组在该 MC 版本没有构建，就不要装）。
- **教训**：给测试实例装语料时**必须按该实例的 MC 版本 + Java 运行时选构建**；把高版本编译的第三方模组塞进低版本实例会崩在渲染阶段，极易误判成自己的 bug。

## 10.22 测试实例取包的两个坑

- **跨版本单包**：一份 jar 自报支持十几个 release（例如声明 `1.16.5 ~ 1.21.8` 的单包），装进不匹配的版本照样崩，属**正常现象**，删掉即可，**别回头怀疑本工程**。判定：读 jar 元数据里的版本声明，与实际实例版本比对，而不是看文件名。
- **同名模组**：不同作者可能发布**完全同名但完全不同**的模组（历史上出现过两个同名项目、同名不同作者的配置库）。**解析后必须打印项目标题 + 产物文件名再下载**，只认标题和 slug，不认文件名。
- 实例侧台账与验收口径见 [08-deploy-qa.md](08-deploy-qa.md)。

## 10.23 库的 fail-fast 校验把别的模组拖死

- **现象**：服务端启动中止，报某库 `xxx has failed to load correctly`，看起来是库坏了。
- **根因**：某库的启动校验器自建了一条**原版并不存在**的规则——"一个附魔只能属于一张互斥标签"。而下游模组唯一可行的写法（要表达"我的附魔与原版 X 互斥、但可与原版 Y 共存"，原版标签颗粒度太粗时只能自建一张只含 X 的标签）必然让 X 同时出现在两张标签里，于是被判定为致命错误、中止整个启动。
- **判定手法（这是本案例的核心）**：反汇编原版相关方法。此处 `Enchantment#areCompatible(Holder, Holder)` 里对**双方**各取一次 `getfield Enchantment.exclusiveSet`（类型 `HolderSet`），说明某附魔的有效互斥集合 = **所有引用到它的标签的并集**，跨组重复完全合法 → 结论：**是校验器的假设错了，不是配置错了**。
- **修法**：把库里的 fail-fast 校验降级为 WARN（记日志但不计入错误数、不抛异常），真正的结构性错误仍保持 fail-fast。
- **别被日志摘要骗了**：这类问题会**冒充成库自身的加载失败**，而真正的错误行在它**上面几行**，内容往往是下游模组提供的 ID。只看 `ModLoadingException` 摘要会误判成"库坏了"。
- **结论**：**不要因为库写了 fail-fast 就认定自己的配置有问题**——约束本身可能才是 bug。这条与 §10.1 第 10 条同源。

## 10.24 Mixin 兼容级跨加载器崩（实测）

- **现象**：同一分支的 jar，Fabric / NeoForge 正常，**Forge 端启动即崩**：

  ```text
  MixinInitialisationError: Mixin config <modid>.mixins.json specifies compatibility level JAVA_21 which is not recognised
  ```

- **根因**：`common` 的 `mixins.json` 写死 `JAVA_21`（从新版本基线复制过来）。Fabric 自带新版 sponge-mixin 认识它；**Forge 线自带的 Mixin 版本更老**——1.20.4 / 1.20.6 = **0.8.5**（只认到 `JAVA_17`，且自带 ASM 不支持 major 65 的类文件），1.21.1 / 1.21.5+ 起才是 **0.8.7**。
- **判定手法**：从实例的 `<name>.json` 里读**库坐标**的 mixin 版本，**别猜**。

  | 加载器 / 版本线 | 自带 Mixin | 可用兼容级 | 是否需要 `"refmap"` 键 |
  |---|---|---|---|
  | Forge 1.20.4 / 1.20.6 | **0.8.5** | 只到 `JAVA_17` | 需要（SRG 运行期） |
  | Forge 1.21.1 / 1.21.5 / 1.21.8 | **0.8.7** | `JAVA_21` 合法 | 需要（SRG 运行期） |
  | Forge ≥ 1.21.9（FG7，official 映射） | 0.8.7 | `JAVA_21` 合法 | **不需要**（`reobf=false`） |
  | Fabric（新版 Loom） | 自带较新版 sponge-mixin | `JAVA_21` 合法 | 不需要（Loom 风格） |
  | NeoForge（自带 fancymodloader + sponge-mixin） | 随版本线，通常较新 | 按坐标核实 | 一般不需要 |
- **修复（按档位分）**：
  - 分支自带 Mixin **0.8.5**（Forge ≤ 1.20.6）：forge 模块产物必须 **Java 17 字节码 + `JAVA_17` 兼容级**。若分支 `java_version=17` 只需改写兼容级；若为 21，**不能用 `options.release` / `targetCompatibility` 直接降**——二者会把解析属性 `org.gradle.jvm.version` 拉到 17，与 `:common` / FML 发布的 Java 21 变体冲突（`only compatible with JVM runtime version 21 or newer`）。可行做法：保持 Java 21 编译，`compileJava.doLast` 把产物 `.class` 的 major 字节（第 7 字节）从 **65 改写为 61**（模组未用 Java 18+ 特性时安全），并同步改写 jar 内 `mixins.json`。
  - 分支自带 Mixin **0.8.7**（Forge ≥ 1.21.1）：`JAVA_21` 合法，**无需任何处理**。
- **预防**：复制新版本基线开新分支时，`mixins.json` 的兼容级要按**目标 Forge 自带的 Mixin 版本**重估，不能照抄。
- **同源第二坑（SRG 运行期）**：修完兼容级后若报 `Critical injection failure ... No refMap loaded`，是 jar 里**有 refmap 文件但 `mixins.json` 缺 `"refmap"` 键**（新版本基线是 Loom 风格、不需要该键；FG6 + SRG 必需）。修法：给 forge 产物的 `mixins.json` 注入 `"refmap": "<mod_id>.refmap.json"`。对照：能工作的 Forge-first 分支（1.18.x ~ 1.20.1）的 `mixins.json` 自带该键；`reobf=false`（官方映射运行期）的分支不需要 refmap。

## 10.25 Forge 高版本构建：从 FG6 的连环坑到 FG7（一整条链路）

给 1.21.9+ / 26.x 分支补 Forge 端时踩到的**顺序**与解法（**先看官方 MDK，不要先改缓存**）：

| # | 症状 | 根因 | 解法 |
|---|---|---|---|
| 1 | `Failed to apply plugin 'net.minecraftforge.gradle'` → `Found Gradle version Gradle 9.6.1. Versions Gradle 9.0 and newer are not supported yet` | 1.21.9+ / 26.x 的主工程被 Loom 1.15+ / MDG 2.0.14x 逼到 Gradle 9，而 **FG6 只支持 Gradle 8.x** | 换 **FG7**（官方 MDK：1.21.11 `[7.0.3,8)`、26.x `[7.0.17,8)`；wrapper Gradle 9.5 / 9.7）。或把 forge 端拆成独立 Gradle 8 工程（旧路线，仅在 FG7 不可用时） |
| 2 | `java.util.zip.ZipException: duplicate entry: mcp/client/Start.class`（FG6 + MC 1.21.11） | 1.21.11 起 FART src-fix 会把 inject 源码编进基础 jar，而 FG6 的 `findRaw` 合并 mci 与 compiled 时**不去重** | 用 FG7 即绕开；**不要**去改 `mcp_config` 缓存（删源文件 → `错误: 无源文件`；改内容 → 管线取不到源码，一样空） |
| 3 | FG7 下 `Unable to locate obfuscation mapping for @Inject target` | 挂了 mixin 注解处理器，在 official 映射环境里找混淆映射 | 去掉 `annotationProcessor`，改 `compileOnly 'org.spongepowered:mixin:0.8.7'`（official 运行期不需要 refmap） |
| 4 | FG7 下 `property 'group' doesn't have a configured value` | `processResources` 的 `expandProps` 里有 `group` 键，撞 Gradle 9 的任务属性 | 删掉该键（`mods.toml` 并未使用） |
| 5 | FG7 下 `only compatible with JVM runtime version 25` | 用 `options.release=21` 降级，把解析属性拉到 21 | 保持原 JVM 编译，**编译后改写 class major**（69→65）并把 `mixins.json` 兼容级改成 `JAVA_21` |
| 6 | `Could not find net.minecraftforge:forge:26.1.2-62.0.9-userdev.jar`（404） | Forge 的版本线是 `26.1=62.x / 26.1.1=63.x / 26.1.2=64.x / 26.2=65.x / 26.3=66.x`，拿 62.x 去配 26.1.2 当然没有 | 用该 MC 补丁版本自己的 Forge 构建（26.1.2 → `64.1.3`），loader 下界取线起点（`[62,)`） |
| 7 | `找不到符号: 方法 displayClientMessage(MutableComponent,boolean)`（26.x） | 26.x 的 `LocalPlayer` 换回 `sendSystemMessage(Component)`（1.21.6 – 1.21.8 才是 `displayClientMessage`） | 按目标版本选客户端发送 API，并**与同分支 fabric / neoforge 端保持一致** |
| 8 | 启动即崩：`IllegalArgumentException: Only a single listener found in class <Client 类>. You should directly call addListener() on the EventBus of LoggingIn` | **EventBus 7**（Forge **1.21.8** 起为 7.0.x，1.21.11 / 26.x 为 7.0.5 / 7.0.6）**不再接受 `register(Class)` 注册"单监听方法"的类**；`MinecraftForge.EVENT_BUS` 已换成 `EventBusMigrationHelper` 兼容层 | 用事件类自带的 `BUS`：`ClientPlayerNetworkEvent.LoggingIn.BUS.addListener(X::onLogin);`，并去掉 `@SubscribeEvent`。判据：查实例 `<name>.json` 里的 `eventbus` 版本（**6.x 仍可用 `register(Class)`**，7.x 必须改） |

**方法论教训**：遇到"插件与工具链互斥"时，**第一步应该是拉官方 MDK 看它用什么插件与 wrapper**（`https://maven.minecraftforge.net/net/minecraftforge/forge/<mc>-<forge>/forge-<mc>-<forge>-mdk.zip`），而不是继续在本地缓存与属性上打补丁——差一步就白烧一小时。

**顺序上的启示**：这 8 条里前 6 条都是"工具链版本不对"，只有 7、8 是代码适配。**先确定工具链能不能跑，再改代码**，顺序反过来会得到一堆假故障。

## 10.26 磁盘 / 临时目录写满：工具链会以"不报错"的方式失败

- **现象**：Gradle 命令**没有任何输出**、或只回一个空壳结果（看起来像工具坏了、参数写错了、任务不存在），反复重跑都一样；**日志里没有可归因的错误**。
- **根因**：**系统盘接近写满**。Gradle 缓存、wrapper 发行版目录、工具链临时目录、上传脚本的临时文件**全都落在系统盘**，与工程所在盘无关——所以"工程盘还有空间"并不代表能构建。
- **现场证据**：清理系统盘、余量恢复到约 **9.2GB** 后，**同一条命令原样重跑即成功**——失败与命令内容无关。
- **对策（通用口径）**：
  1. 任何"莫名空结果 / 无输出 / 无报错地失败"的构建或上传，**先看系统盘剩余空间**（`Get-PSDrive` 或 `df -h`），再去看代码、参数、插件版本；构建前建议预留 **≥10GB**。
  2. 空间紧张时优先清**可再生的缓存**（`Temp`、`_tmp` staging 等），**不要删 Gradle 缓存里的档位目录**——老版本分支与遗留档正在用（见 [02-environment.md](02-environment.md) §2.3.2）。
  3. 换盘执行（把工程 / 缓存挪到空间充足的盘）属**环境改造**，要先问用户，不要自行搬移。
- **别踩的陷阱**：不要因为"命令没报错、退出码 0"就认为成功（与 §10.20 同类）；也不要把它误判成"插件不兼容"去改构建脚本。

## 10.27 "此世界使用实验性设置"弹窗：真因是包没有声明 KnownPack

**现象**：装了模组后，**新建或进入世界**时每次都弹确认框（"此世界使用实验性设置，可能导致崩溃"），必须点确认才能进。逐模组排查可定位到具体模组，**关掉模组就正常** ⇒ 玩家会认为"是这个模组的 bug"。

**机制（`javap` 级证据，不是推断）**：

1. 世界的生命周期取自**动态注册表**（`Registries.ENCHANTMENT`、`Registries.VILLAGER_TRADE` 等）是否"非原版"：服务端日志可见 `non-stable registries: minecraft:enchantment(Experimental)`。
2. 每个注册表条目的生命周期，由**提供该文件的包**决定：

   ```text
   ResourceManagerRegistryLoadTask:
       RegistrationInfo(Resource.knownPackInfo(), known ? Lifecycle.stable() : Lifecycle.experimental())
   ```

   `Resource.knownPackInfo()` 就是该文件所在包的 `PackLocationInfo.knownPackInfo`。
3. `PackLocationInfo(id, title, source, Optional<KnownPack>)`——**模组自己创建的包（如运行时"内存数据包"）通常把第 4 个参数写成 `Optional.empty()`，这一处空值就是弹窗的源头。**
4. `KnownPack(namespace, id, version)`，判定只用「有没有」——`known ? stable : experimental`，**没有白名单校验**。

**修复（通用做法）**：

- **自己创建数据包的模组**（运行时生成包 / 自建 `RepositorySource`）：构造 `PackLocationInfo` 时传入

  ```java
  Optional.of(new KnownPack(<modid>, "<pack-id>", "<固定版本串>"))
  ```

  **版本串用固定字符串**（如 `"1"`），不要塞模组自身版本——`(namespace, id, version)` 是身份三元组，随版本变会让老存档记录的包身份失配。
- **数据包由 jar 内 `data/` 提供、包由加载器创建**的模组：自己无法改 `PackLocationInfo`。可行路径是把这些注册表条目**改由"自己创建已知包"的一方代发**（例如库模组把业务模组的 JSON 读进来，写进库自己的已知包），或改用加载器提供的注册表事件（NeoForge 有 `DataPackRegistryEvent`；**Fabric 侧是否有等价能力未经实测**，用前先查该版本 Fabric API 的实际事件表，不要照抄 NeoForge 的写法）。

**排除项（防止走弯路）**：

- 这**不是** `pack.mcmeta` 格式问题（§10.9）。两个模组都没有 `pack.mcmeta`，弹窗照旧。
- 这**不是** Fabric 版本谓词、依赖区间、标签格式问题：日志里没有任何 `Couldn't read tag list` / 解析失败。
- 这**不是**"原版行为、改不了"：源头是模组自己写的那个 `Optional.empty()`，**能改**。
- 用**代码 API 注册附魔**并不能绕开：动态注册表只能在数据包加载阶段填充，"代码 API"最终仍是把 JSON 写进某个包 ⇒ 仍受本条约束。

**判定手法**：看日志里的 `non-stable registries:` 一行（可由模组自己打印）；或在游戏内对比"关掉模组 vs 打开模组"。修复后该行应消失，弹窗同步消失。

## 10.28 弹射物模型朝向与飞行方向相反：镜像空间里的 Y 轴旋转反号

**症状**：自定义弹射物（`AbstractHurtingProjectile` / `ThrowableProjectile` 之类）**实体的旋转角算得完全正确**（按 `Projectile#shoot` 约定逐 tick 对齐真实速度），但**模型渲染出来是朝着反方向飞的**。特征很好认：**水平方向反了，俯仰角是对的**——头朝着尾后飞，但该抬头时照样抬头。

**根因（`javap` 级取证）**：新版实体模型渲染管线里，非生物模型提交前普遍带一句 `poseStack.scale(-1, -1, 1)`（原版 `WitherSkullRenderer#submit` 就是 `pushPose` → `scale(-1,-1,1)` → `submitModel` → `popPose`）。这个缩放**等价于绕 Z 轴转 180°**，它会让**之后在同一个 PoseStack 里做的 Y 轴旋转方向反号**：

```text
S = diag(-1,-1,1)          // S·R_y(θ) ≠ R_y(-θ)·S
```

而模型的"脸"一般在**模型空间 -Z**（MC 实体模型一律以 -Z 为正面）。把 `(0,0,-1)` 过一遍 `S·R_y(yRot)·R_x(xRot)`：竖直分量与俯仰一致，**水平分量 = 速度的反向**——正是上面那个症状。

**修法（通用）**：

- 优先**在渲染层加 Y 偏置**：`modelState.yRot = entity.getYRot(partialTicks) + 180.0F`。
- **只动 yaw，不要动 pitch**：绕 Y 轴转 180° 不影响 `R_x`，所以俯仰一行都不改（这是"俯仰本来就是对的"的必然要求）。
- **不要去改实体的 `setYRot` / `setXRot`**：实体旋转要保持原版 `shoot` 约定（`yRot = atan2(dz, dx) - 90`、`xRot = -atan2(dy, 水平分量)`），否则别的系统（`getViewVector`、追踪、同步、其它渲染器）会按错误约定读它。

**反例警告**：`ThrownItemRenderer` 写的是 `180.0F - yRot`（**减**），因为物品渲染还额外套了一层物品展示变换；**照抄那一行会得到"仍然反向，只是换了种错法"**。抄渲染器时先确认自己的模型正面朝向与管线是否与被抄对象一致。

**验证方法（不靠肉眼）**：把模型空间正面法线代进上面的矩阵式子，水平分量应与 `getDeltaMovement()` 的水平分量**同向**；再单独检查竖直分量是否等于 `deltaY`——这能立刻区分"整体反了"与"只有水平反了"。

---

## 10.29 排障通用心法（复用性最高）

1. **先取证再解释**：原版字节码 / jar 解包 / maven 元数据 / 实例 JSON 才是证据；文档、印象、日志摘要都可能骗人。
2. **失败信号分"会叫的"和"不叫的"**：不叫的（静默失效）只能靠主动核对（计数、描述符、键集合）发现。`@At` 目标串、`Optional.empty()`、图标尺寸、`pack.mcmeta` 都属"不叫的"。
3. **两个目录分清**：`build/libs`（历史产物，**不碰**）与实例 `mods/`（**只留最新一版**）。
4. **改动后必须看最终产物**：解包 jar、严格 JSON 解析、语言键求差、元数据字段核对——不看产物就等于没验证。
5. **误判优先怀疑环境与第三方**：class file version、第三方同名 / 跨版本包、错放进实例的 jar、磁盘空间、工具链档位。
6. **"工具坏了"永远排在最后**：先怀疑自己的调用方式（引号、路径、退出码），再怀疑工具。
7. **一条命令失败 ≠ 命令写错**：同一条命令原样重跑就成功的情况真实存在（§10.26），要能区分"参数错"与"环境不可用"。
8. **结论必须可复核**：每条踩坑结论都要能回答"用什么命令 / 看了哪个文件 / 哪个版本测的"；写不出这三项的就是推断，不能当铁律用（证据强度标注见 [11-verdicts.md](11-verdicts.md)）。

### 10.29.1 高频误判清单

这些是历史上真实发生过的误判方向，看到对应现象时先排除它，再往下查。

| 误判方向 | 实际是什么 | 正确入口 |
|---|---|---|
| "第三方渲染模组崩了，所以是它的 bug" | 它的崩溃是**先导**，根因在更早的日志 | §10.3 |
| "`CancellationException` 是根因" | 崩溃后的清理噪音 | §10.1 第 6 条 |
| "构建成功就说明游戏在跑新代码" | 实例 `mods/` 里还是旧 jar | §10.1 第 7 条 |
| "库写了 fail-fast，所以是我的配置非法" | 约束本身可能是错的 | §10.23 |
| "命令 exit 0，所以成功了" | 管道退出码是最后一个命令的 | §10.20 |
| "工具坏了 / 参数写错了" | 系统盘写满，工具链静默失败 | §10.26 |
| "我的模组在崩" | 错版本第三方包被装进实例 | §10.21 |
| "Mixin 注解写错了" | 配置根本没被加载（漏了一端声明） | §10.10 |
| "代码逻辑算错了" | 渲染管线里的镜像变换让结果反号 | §10.28 |
| "改一行就该生效" | 增量构建没重跑 / 产物名被覆盖 | §10.14 |

**用法**：误判清单不是"避免犯错"的清单，是**缩短搜索空间**的清单——命中任何一行就直接跳到正确入口，不再从零开始读代码。

---

## 10.30 错误原文速查

按报错原文定位，比按现象猜快。每条给出"它到底在说什么"与"下一步查哪个案例"。

| 错误原文 | 真正含义 | 去哪 |
|---|---|---|
| `ParsingException: Invalid newline in basic string` | `description` 里有真实换行 | §10.3 |
| `not a valid mod file` | 元数据解析失败的总入口，**先找它而不是找最后一个异常** | §10.3 / §10.4 |
| `InvalidModFileException` / `Invalid modId found` | modId 不匹配正则 | §10.4 |
| `CancellationException` / `IOException: Stream closed` | 崩溃后的清理噪音，**不是根因** | §10.1 第 6 条 |
| `uses the deprecated 'logoFile' property` | 26.2+ 该字段已废弃，图标会跑错槽位 | §10.5 |
| `InvalidBrandingImageException`（且不进日志） | 图标超 256×256 或非正方形 | §10.6 |
| `failed to load a valid ResourcePackInfo` / `Missing metadata in pack <modid>:...` | Forge 线缺 `pack.mcmeta`（模组其实加载成功） | §10.9 |
| `MixinInitialisationError: ... specifies compatibility level JAVA_21 which is not recognised` | 目标加载器自带的 Mixin 太老 | §10.24 |
| `Critical injection failure ... No refMap loaded` | jar 里有 refmap 但 `mixins.json` 缺 `"refmap"` 键 | §10.24 末 |
| `Injection failure` 完全没有（静默） | `@At` 目标串失配，或 mixin 配置没被加载 | §10.10 / §10.11 |
| `Configuration with name 'sourcesElements' not found` | `configurations.named()` 对不存在的变体报错 | §10.14 |
| `Could not find or load main class org.gradle.wrapper.GradleWrapperMain` | wrapper jar 缺失 | §10.16 |
| `cannot access class_XXXX` | 用了 `implementation` 而非 `modImplementation` | §10.17 |
| `UnsupportedClassVersionError: ... class file version 69.0 (Java 25) > 65.0 (Java 21)` | 高版本编译的第三方包装进了低版本运行时 | §10.21 |
| `编码GBK的不可映射字符` / `unmappable character` | `javac` 按平台默认编码读 UTF-8 源码 | §10.19 |
| `Failed to apply plugin 'net.minecraftforge.gradle'` + `Gradle 9.0 and newer are not supported yet` | FG6 只支持 Gradle 8.x | §10.25 第 1 条 |
| `duplicate entry: mcp/client/Start.class` | FG6 的 `findRaw` 合并时不去重 | §10.25 第 2 条 |
| `Unable to locate obfuscation mapping for @Inject target` | 混入注解处理器，在 official 映射环境里找混淆名 | §10.25 第 3 条 |
| `property 'group' doesn't have a configured value` | `expandProps` 里的 `group` 撞 Gradle 9 任务属性 | §10.25 第 4 条 |
| `Only a single listener found in class ...` | EventBus 7 不再接受 `register(Class)` 的单监听方法类 | §10.25 第 8 条 |
| `Could not find net.minecraftforge:forge:<mc>-<forge>-userdev.jar`（404） | MC 补丁版本与 Forge 版本线错配 | §10.25 第 6 条 |
| `找不到符号: 方法 displayClientMessage(MutableComponent,boolean)` | 该版本已换回 `sendSystemMessage(Component)` | §10.25 第 7 条 |
| `non-stable registries: ...(Experimental)` | 提供该条目的包没声明 `KnownPack` | §10.27 |

**判读要点**：`UnsupportedClassVersionError` 与 `NoClassDefFoundError` 这类"类层面的错"几乎总是指向**装载 / 编译目标**，而不是逻辑；`Injection failure` 之类的**缺失**（该报没报）才是 Mixin 类问题的特征。

---

## 10.31 取证工具箱（判定手法一览）

排障的胜负多半取决于"用什么工具看"。下表是各手法能回答什么问题，全部为一手证据。

| 手法 | 命令 / 操作 | 能回答什么 | 对应案例 |
|---|---|---|---|
| **反汇编目标方法** | `javap -p -c -s <类>` | 注入点真实存在吗、`@At` 目标串对不对、方法描述符是什么、原版约束是否真实存在 | §10.11 / §10.23 |
| **看方法签名** | `javap -p -s` | 参数 / 返回类型（描述符），两端方法名不同的核对 | §10.11 |
| **解包 jar** | 解压到临时目录 | 资源真的打进包了吗、`mixins.json` 里有什么键、`pack.mcmeta` 在不在、refmap 文件在不在 | §10.9 / §10.12 / §10.24 |
| **读 PNG 头** | 读 IHDR：偏移 16 / 20 起各 4 字节大端 = 宽 / 高 | 图标真实像素，绕开看图软件的缩略图 | §10.6 |
| **严格 JSON 解析** | 用解析器而不是正则 | 语言文件 / 元数据是否合法、键集合差集 | §10.13 |
| **语言键求差** | `lang/zh_cn.json` 的键 vs 代码里 `translatable(...)` 的键 | 哪些键永远不会被翻译 | §10.13 |
| **读实例 `<name>.json` 的库坐标** | 找 `sponge-mixin` / `eventbus` / fancymodloader 等坐标与版本 | Mixin 兼容级能不能用、EventBus 是 6.x 还是 7.x、加载器是哪一种 | §10.24 / §10.25 第 8 条 |
| **查 maven 元数据** | 拉 `maven-metadata.xml` | 某版本到底存在不存在、最新版是多少 | §10.25 第 6 条 |
| **拉官方 MDK** | 下载对应 `-mdk.zip` 看插件与 wrapper | 这条版本线的"标准答案"是什么 | §10.25 方法论 |
| **比对产物时间 / 体积** | `build/libs/*.jar` vs 实例 `mods/*.jar` | 游戏跑的是不是本次产物 | §10.1 第 7 条 |
| **看系统盘余量** | `Get-PSDrive` / `df -h` | "无输出无报错地失败"是不是空间问题 | §10.26 |
| **带外回读** | `ls-remote` 哈希 / File ID / `BUILD SUCCESSFUL` | 命令到底成没成 | §10.20 |

**手法选择原则**：

- 问"某条约束真的存在吗" → 只能反汇编，文档与印象都不算证据。
- 问"某文件在不在包里" → 只能解包，构建日志不算。
- 问"某数字对不对" → 只能实测（PNG 头、库坐标、maven 元数据），不能按印象填。
- 问"命令成没成" → 只能带外回读，退出码不算。

---

## 10.32 自检清单

- [ ] 读日志时抓的是**第一条**根因行（`not a valid mod file` / `Invalid modId found` / `ParsingException`），不是最后一条异常。
- [ ] 模组确实出现在 Mod List；不在就先修元数据。
- [ ] 比对过 `build/libs/*.jar` 与实例 `mods/*.jar` 的时间与体积，确认游戏跑的是本次产物。
- [ ] 两端 Mixin 声明**成对**；日志 `Mixin Subsystem` 附近能看到本模组的 `*.mixins.json`。
- [ ] 注入失效时用 `javap -p -c -s` 核对过 `@At` 目标串，且排除了注释里的死串。
- [ ] 资源类问题核对过扫描计数（`M` / `K` 不为 0），并解包确认 `assets/<modid>/...` 存在。
- [ ] 图标是 128×128 正方形，且用 PNG IHDR 实测过像素（不是看图软件缩略图）。
- [ ] `pack.mcmeta` 按版本线处理：NeoForge 无数据包内容就删，Forge 补最小元数据且只放 forge 模块。
- [ ] 判定成功用的是带外回读（`BUILD SUCCESSFUL`、`ls-remote` 哈希、File ID），不是管道退出码。
- [ ] "无输出、无报错地失败"时先看过系统盘剩余空间。
- [ ] 结论按「现象 → 根因 → 判定手法 → 修复」写清，并标注证据强度（实测 / 推断）。
