---
name: minecraft-mod-dev
description: Minecraft Java 版模组（Fabric / NeoForge / Forge，1.7.10 ~ 26.3 全版本线）开发技能。覆盖工程结构与元数据（fabric.mod.json / neoforge.mods.toml / pack.mcmeta / 图标槽位）、本地化、Mixin 与跨加载器 API 适配、Gradle/Java 档位与构建、多版本移植（L1/L2/L3、range 纪律）、测试部署与验收、Git 与 CurseForge/Modrinth 发布、排障案例库、Blockbench 美术与生物动画规范。触发词：新建模组、移植、加版本分支、构建失败、模组不加载、注入失效、多语言失效、发布、上传 CF/MR、mixin、pack.mcmeta、Gradle 档位、模型贴图、GeckoLib。
license: CC0-1.0
metadata:
  version: "2.0.0"
  standard: agentskills.io
---

# Minecraft 模组开发 · SKILL

> 本文件是 **Minecraft 模组开发的跨项目技术知识库**，与同目录的示例 `AGENTS.md` 配套使用。
> **可复用**：换模组、换版本线直接沿用；文中不绑定任何具体模组项目，项目专属值（modId、包名、配置路径、Project ID、语言键）只写进该项目自己的决策记录。
> **去个人化**：文中不含任何作者机器路径、代理端口、实例名、账号或项目 ID。所有环境相关处写成占位符，**按你的环境调整见 `README.md`**。
> **分层**：本文件是整合版、**结论以此为准**；细节、命令、字段全集与完整案例在同目录 `references/`，正文按需跳转。
> **优先级**：一手证据（`javap` / jar 解包 / maven 元数据 / 实例 JSON）> 本 SKILL > 文档与二手转述。凡带日期的实测结论都可能随平台升级过期，使用前先复核。

---

## §0 入口与全局纪律

### 0.1 适用范围与使用前提

- **适用**：Minecraft Java 版模组开发（Fabric / NeoForge / Forge，单版本线到多版本线）—— 建工程、写元数据、适配 API、构建、部署验收、发版、排障。
- **不含任何个人事实**：凡属"在你这台机器上才成立"的事实（JDK 装在哪、Gradle 缓存有哪些档、实例在哪个目录、代理端口是多少），**一律现场勘察**后再用，见 `references/02-environment.md` 的自查清单；**不要照抄本文里的示例路径**。
- **三层结构**：`SKILL.md`（本文件，整合版）→ `references/*.md`（分模块详解）→ 你项目自己的 `AGENTS.md`（只写该项目的硬约束）。
- **本文件与 `references/` 的分工**：本文件给**结论与判据**；`references/` 给**可执行细节**（完整字段表、命令、版本取值表、逐案例复盘）。

### 0.2 分层标签（全文通用，读时先看标签再决定信不信）

| 标签 | 含义 | 处理方式 |
|---|---|---|
| **铁律** | 违反会导致 ①构建失败 ②功能静默失效（不报错不崩）③已发布物必须返工 | 直接执行，不得商量 |
| **约定** | 推荐做法，可按项目调整 | 与用户当次指令冲突时以用户为准 |
| **实测** | 有 `javap` / jar 解包 / maven 元数据 / 实机验收等可复现证据 | 可直接采信；平台升级后可能过期 |
| **推断** | 尚未实测的推测 | **不许据此放宽版本声明或发布**，必须先探针 |
| **已作废** | 曾经采用、现已推翻或停用 | 保留以防被复述成事实 |

### 0.3 内容索引（章节 → 管什么 → 细节在哪）

| 章节 | 管什么 | 细节文件 |
|---|---|---|
| §0 | 使用方式、分层标签、场景路由、全局纪律、排障心法 | — |
| §1 | 发布身份、字段填写、`mod_id`/`mod_name` 分工、版本号询问、产物命名、前置依赖区间 | `references/01-publishing.md` |
| §2 | Java 分档、Gradle 三铁律、镜像、`JAVA_HOME`、离线优先、环境自检 | `references/02-environment.md` |
| §3 | MultiLoader 结构、元数据字段全集与模板、图标槽位、`pack.mcmeta`、`expand` 陷阱、KnownPack | `references/03-metadata.md` |
| §4 | 语言清单、键齐平、UTF-8 无 BOM、`§` 色码、可翻译组件与兜底、literal 事故 | `references/04-i18n.md` |
| §5 | Mixin 注册/remap/注入点校验、版本差异表、NeoForge 26.x 变更、事件桥接、可选前置 | `references/05-mixin-api.md` |
| §6 | Gradle 档位依据、FG6/FG7、MultiLoader 模板、依赖声明、编译编码、构建期故障 | `references/06-build.md` |
| §7 | L1/L2/L3 判据、触点差分、一手来源、分支方案、探针、range 纪律、老版本线、波次 | `references/07-porting.md` |
| §8 | 部署三查、实例判定、MD5 回读、验收清单 | `references/08-deploy-qa.md` |
| §9 | GitHub 分支/推送/代理/GCM/凭据红线、CurseForge、Modrinth、发布自检、Commit 规范 | `references/09-git-upload.md` |
| §10 | Debug Flow + 按症状索引的实战案例 + 排障心法 | `references/10-troubleshooting.md` |
| §11 | 易错结论对照（旧说法 → 现行口径）、待核实清单、元原则 | `references/11-verdicts.md` |
| §12 | Blockbench 模型/贴图硬规范、踩坑、多层染色贴图反推 | `references/12-art.md` |
| §13 | 生物分类、注册链、渲染对齐、AI/Goal、GeckoLib、货物生命周期 | `references/13-entities.md` |
| §14 | 标准工作流：建新模组 / 扩版本分支 / 发版本 / 上线排障 | `references/14-workflows.md` |
| §15 | 交付文档写什么、绝不写什么 | `references/15-docs.md` |

### 0.4 场景路由（我现在要干什么 → 读哪几块）

| 场景 | 阅读顺序 |
|---|---|
| 从零建新模组 | §1 → §3 → §4 → §6 → §14.1 |
| 接手别人的老工程 | §2（核实 Java 与缓存）→ §6 → §10 |
| 把一个模组扩到更多 MC 版本 | §7 → §3 → §6 → §8 |
| 要发一个版本 | §1（问版本号）→ §9 → §8 |
| 上线后出问题 | §10 → §5 → §7 |
| 换了电脑 / 环境变动 | §2 → §11.2 的待核实清单 |
| 判断某条旧结论还成不成立 | §11 → 对应章节 |
| Forge 高版本（1.21.9+ / 26.x）启动崩或构建崩 | §10 的 Forge 构建案例 → §6.1 → §5（Mixin / 事件注册） |
| 命令莫名空输出 / 无报错地失败 | §10（先查磁盘余量）→ §2.5 |
| 做模型 / 贴图 | §12 |
| 加生物 / 动画 | §13 |
| 写 README / API 文档 / 拆文档 | §15 |

### 0.5 全局使用纪律（铁律，不可商量）

1. **项目进度与决策只写该项目的决策记录**；本 SKILL 只放跨项目通用内容。项目专属值（modId、包名、配置路径、Project ID、语言键）**不当作通用默认值**。
2. **顺序铁律**：先探针后改码，先构建后部署，先本地验收后发布（冲突时以当前用户指令为准）。
3. **不猜**：找不到实例、查不到构建、拿不到 ID → **停下来报告**，不许靠目录名、关键词、记忆推断。
4. **管道退出码不可信**：`| tail`、`| grep | head` 给出的是末端命令的退出码；PowerShell 里 `| Select-Object -First N` 会**杀掉**正在运行的 Gradle。一律改为带外回读（`BUILD SUCCESSFUL`、`ls-remote` 比对哈希、上传后 File ID 回查）。
5. **凭据红线**：`curseforge_api_key` / `modrinthApiKey` 位于用户**全局** `gradle.properties`，**不读、不搜、不打印、不落盘、不转交外部命令**，只允许 Gradle 内部消费（`findProperty`）。
6. **改一处先分析根因**；**不擅自改用户没要求的配置**。
7. **修复必须重新构建 + 重新部署 + 重新验收**，不许只改文档。
8. **禁止随意回滚**：除非有直接证据证明旧实现是错的、或用户明确允许，不得改动已确认可用的实现；**严禁**回滚到曾经出过问题的写法上。

### 0.6 排障通用心法（复用性最高，先背下来）

1. **先取证再解释**：原版字节码（`javap`）/ jar 解包 / maven 元数据 / 实例 JSON 才是证据；文档、印象、日志摘要都可能骗人。
2. **失败信号分"会叫的"和"不叫的"**：不叫的（静默失效）只能靠主动核对发现 —— 资源扫描计数、注入点完整描述符、语言键集合差、元数据字段位置。
3. **两个目录分清**：`build/libs`（历史产物，**永不删除/移动/备份/记录**）与实例 `mods/`（只保留每个 modId 最新一版）。
4. **改动后必须看最终产物**：解包 jar、严格 JSON 解析、语言键求差集、元数据字段逐项核对。
5. **误判优先怀疑环境与第三方**：class file version、第三方同名/跨版本包、错放进实例的 jar。

---

## §1 发布身份与版本管理

> 细节（字段全集、命名正则、区间写法、自检清单）：`references/01-publishing.md`

### 1.1 发布身份（按项目确定，不要抄别人的）

| 项 | 怎么定 |
|---|---|
| `mod_author` | 你的署名；多作者逗号分隔 |
| `license` | `MIT` / `Apache-2.0` / `CC0-1.0` / `LGPL-3.0` 等；**必须与 jar 内 `LICENSE`、平台项目页三者一致** |
| 主页 URL | 能打开的仓库 / 文档站；**不要写 404 链接** |
| 反馈 URL | 该模组的 issue 页；仓库未建时先用你控制的备用反馈页 |
| `description` | 一句话简介，**单行**；英文在前、中文在后，用 ` | ` 分隔 |

- **禁止 404 按钮**：页面还没建好时不要猜 URL 顶上去。
- 描述里**不要再写一遍 issue 地址**（Forge 1.20.1 的模组列表会把 `issueTrackerURL` 渲染成"问题跟踪"入口，重复写会出现两次）。

### 1.2 字段落点（写错位置 = 按钮不显示）

| 字段 | 放哪 |
|---|---|
| `displayURL`（主页）、`iconFile`（26.2+）/ `logoFile`（≤26.1.2） | `[[mods]]` **子表内** |
| `issueTrackerURL`（反馈） | **文件顶层**（与 `modLoader` / `license` 同级） |
| Fabric `contact.homepage` / `contact.issues` / `icon` | `fabric.mod.json` 顶层 |

### 1.3 `mod_id` 与 `mod_name` 的分工（🚩 最高优先级）

- `mod_id`：**机器名**，小写蛇形，进包名、注册表、资源目录、maven 坐标。
- `mod_name`：**给人看的展示名**，允许空格与大写。
- `mod_name` **一处驱动四处门面**：`fabric.mod.json` 的 `name`、`neoforge.mods.toml` 的 `displayName`、CurseGradle 的 `displayName`、jar manifest 的 `Specification-Title`。
- **把 `mod_name` 写成 `mod_id` 原样值**会同时污染游戏内模组列表与 CurseForge 文件展示名。
- **发布前必做**：把 `displayName = "${mod_name} ${version} (<Loader>) for Minecraft ${minecraft_version}"` 展开一次，确认首词是展示名。

**modId 命名正则（违反 = `Invalid modId found`，整个 jar 不加载）**：

```
^(?=.{2,64}$)[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$
```

- 长度 2~64；仅小写字母 + 数字 + 下划线；**禁止连字符 / 大写 / 空格**；可用 `.` 分段。
- 资源命名空间目录 `assets/<mod_id>/` 必须一致。
- 改 `mod_id` 要全链路同步：`gradle.properties` → 入口类常量 → 资源目录 → 语言 key / 注册 id 前缀 → 两端元数据。
- 改完若构建 `UP-TO-DATE`，需 `--rerun-tasks` 强制重跑 `processResources`。

### 1.4 版本号：构建前逐模组询问

- 构建新 jar 前**逐模组**问版本号，**不许自行决定递增方式**，并给出预设（patch / minor / 进位 / 补丁进两位 / 用户自填）。
- **`version` 只写模组版本**（`1.2.2`），不带 MC 前缀；MC 版本只体现在 jar 文件名与依赖区间。
- **必须逐模组问**：一个工程里的多个独立发布物依赖区间不同，不能一个号套所有仓库。
- 用户事后要求改号 → **重新构建 + 重新部署 + 重新验收**。
- **同一版本号绝不允许对应两份不同内容的 jar**。
- 问版本号时**紧接着问本次 CurseForge `releaseType`**（`release` / `beta` / `alpha`）。
- 版本号后缀必须带连字符：`1.2.0fix` ✗ → `1.2.0-fix` ✓。

### 1.5 产物命名与旧 jar 处置

- **jar 名** = `<mod_id>-<version>-<loader>-<mc 或区间>.jar`；`version` 段来自 `gradle.properties`，**不手工拼**。
- **maven 坐标 artifactId** 保持 `<mod_id>-<loader>-<mc>`、不含版本。
- **CF 展示名与 jar 名同源**；批量发布时逐条核对：首词是展示名、版本段与 jar 一致、**区间分支就写区间**。
- **默认不产出 `-sources.jar` / `-javadoc.jar`**（移除后模块脚本里的 `configurations.named('sourcesElements')` 会失败，见 §6.4）。

| 位置 | 旧版本 jar 规则 |
|---|---|
| `build/libs` | **永不删除 / 移动 / 备份 / 记录**，留原位 |
| 实例 `mods/` | 每次部署后**直接删除被替换的旧 jar**（同 modId 双版本会被 FML 拒载） |

### 1.6 前置依赖版本区间

1. **按"实际兼容的最早版本"写**，纯 semver 段比较（Fabric `">=1.2.2"`、NeoForge `"[1.2.2,)"`）。
2. **前置没有破坏性变化时不机械抬区间**。
3. **禁止版本格式混搭**：前置从 `26.2-1.2.1` 改成纯 `1.2.1` 后，下游若还写 `>=26.2-1.2.1`，段比较会倒挂（`26 > 1`），模组拒载。
4. 部署新前置前的固定动作：grep 下游元数据里的旧格式版本号，逐个核对无倒挂。

---

## §2 环境与工具链

> 细节（Java 分档依据、老版本线 Java 8 定夺、Gradle 三铁律、镜像、离线优先、自检清单）：`references/02-environment.md`

### 2.1 Java 分档（结论稳定，路径不稳定）

| 用途 | 构建 JDK |
|---|---|
| 1.18 ~ 1.20.4 | **17** |
| 1.20.5 ~ 1.21.11 | **21** |
| 26.1 ~ 26.3 | **25** |
| 1.7.10 / 1.12.2 / 1.16.5 | **8**（1.16.5 用 11 / 17 亦可） |

**三个层次必须分开判断**：① 游戏运行 JVM（启动器按实例 JSON 选）；② 跑 Gradle 的 JVM（`JAVA_HOME` / `org.gradle.java.home`）；③ 编译工具链与目标字节码（`java_version` / `options.release` / class major）。

- ⚠ **`java_version=17` 只是工具链目标**，不代表 Gradle 能跑在 17：Loom 1.13+ / MDG 2.0.123+ 的字节码是 Java 21 ⇒ **Gradle 必须跑在 JVM 21+**。
- **构建命令必须显式设 `JAVA_HOME`**（按目标 MC 线取档），不许依赖系统默认。
- **判定"真 JDK"**：有 `bin/javac`（Java 8 还要 `lib/tools.jar`）。**JRE 只能运行、不能编译**，把 `javac` 拷进 JRE 会得到 `No compiler is provided in this environment`。
- **路径过期时按降级策略找**：启动器运行时目录（`java-runtime-*`）→ `~/.gradle/jdks` → foojay resolver 按需下载。**禁止为"对齐版本号"自行另装 JDK**。

**常见坑（实测）**：
- 启动器运行时**常自带完整 `javac`**，且可能嵌套在 `windows-x64/<同名目录>` 下一层，浅层扫描会漏。
- 启动器按**路径**逐条解析 Java、不按 realpath 去重 ⇒ 符号链接会让列表出现两条同版本。
- 半截卸载残留（目录里只剩 `lib` 或 `conf`）会被工具链探测选中，然后以难查方式失败。

### 2.2 老版本线（1.7.10 / 1.12.2 / 1.16.5）的 Java 8

- **目标字节码不是瓶颈**：17 / 21 / 25 三档 JDK 上 `javac --release 8` 全部成功，产物 major 都是 **52**。
- **真正缺的是运行 Gradle 的 JVM**：FG 1.2 / 2.3 那代绑 Gradle 2.x / 4.x，跑不到 17+ 上。
- 两条出路：**装真 JDK 8**，或**现代 Gradle + `options.release = 8`**（后者成功率 1.16.5 最高、1.7.10 最难；但大概率仍需 JDK 8）。**先拿 1.16.5 试，禁止先装环境再找理由。**
- **「补全现有 JRE」不成立**：`javac` 依赖 `jdk.compiler` 模块 / `lib/tools.jar`，JRE 发布时就没有。
- **「原地覆盖 JRE 目录」不推荐**：虽然老实例仍能启动，但会产生"目录叫 jre、里面是 JDK"的名实不符、启动器显示名变化、注册表残留、回滚要重装 JRE。优先级：先试现代 Gradle → 构建端用 `org.gradle.java.home` 钉死 → 处理重复条目 → **原地覆盖排最后**。
- **系统目录属用户机器，任何改动前都要用户明确下令**；老实例按**路径**选 `java.exe`，改路径会直接影响能否启动。

### 2.3 Gradle 三铁律

**铁律一：先查缓存再决定是否下载**
- 动手前列一次本地 Gradle 缓存目录（Windows `%USERPROFILE%\.gradle\wrapper\dists`，Unix `~/.gradle/wrapper/dists`）。
- 缓存里已有同系列补丁版时，把 wrapper 指过去，**不要为对齐某个数字去新下一份**。
- 只有"缓存里确实没有"且"任务必需"两条都满足，才允许触发一次下载，并在 wrapper 注释里写明原因。
- 违反后果：占磁盘、拖慢构建、污染判断基准。

**铁律二：档位由**插件**决定，插件由 **MC 版本**决定 —— 不许一刀切**

| MC 线 | 构建 Java | Fabric 插件 | Forge / Neo 插件 | Gradle 档 |
|---|---|---|---|---|
| 26.1 / 26.2 / 26.3 | 25 | Loom 1.17.x | MDG 2.0.141~2.0.147 | **9.x** |
| 1.21.2-1.21.11 | 21 | Loom 1.17.21 | MDG 2.0.123 | **9.x** |
| 1.21.1 / 1.20.2-1.20.4 / 1.20.5-1.20.6 | 21 | Loom 1.8-SNAPSHOT | MDG 2.0.49-beta | 遗留档 **8.x** |
| 1.20-1.20.1 / 1.19.2 / 1.18.2 | 17 | Loom 1.9-SNAPSHOT | **ForgeGradle 6** | 遗留档 **8.x** |

- **Loom 依据**：`1.14` 起要求 Gradle 9.2，`1.16` 起要求 9.4；`1.8 / 1.9-SNAPSHOT` 只在 Gradle 8.x 可用（Gradle 8.9+ 移除了 `Problems.forNamespace`）。
- **Loom 硬门槛**：依赖 jar 内嵌 manifest 的 `Fabric-Loom-Version` 必须 ≤ 当前 Loom，否则 `ArtifactMetadata.validateLoomVersion` **直接中断构建**。换 MC 区间时先回读该属性。
- **FG6 依据**：其 6.0 分支自身用 Gradle 8.1.1 构建、直接依赖 `gradleApi()` ⇒ 绑定 Gradle 8.x API；FG 无 Gradle 9 适配迹象 ⇒ Forge ≤1.21.8 必须留在 8.x。**1.21.9+ / 26.x 改用 ForgeGradle 7**（§6.1）。
- **结论**：**至少需要两档**（9.x 与 8.x）。只留一档会让对应线直接构建失败。

**铁律三：Maven 走可达的镜像，且镜像放在官方源之前**

- 镜像地址会变，失效时先访问确认再替换；**不适用时删掉即可**，不影响正确性。
- **必须知道**：公共镜像通常**只覆盖** `mavenCentral` / `gradlePluginPortal` / `google`；**Fabric / NeoForged / Forge / Mojang 的源都没有镜像** ⇒ "配了镜像"≠"离线能构建"。
- 第三方插件自己的 Maven 源（CurseGradle、Cloth Config、Mod Menu、Jade）要单独加。
- **仓库顺序**：镜像 / 中央在前，加载器仓库在后，否则公共依赖（如 LWJGL）可能卡在 TLS 握手上。

**接手任务的强制动作：检查 → 核实 → 写注释**
1. **检查**：读 wrapper 的 `distributionUrl`，核对缓存里有没有这一版，判断属于哪一档。**批量同步时逐目录核对**。
2. **核实**：实际跑一次 `gradlew projects`（配置期）+ 目标模块 `jar`（构建期）。**禁止凭记忆**。
3. **写注释**：每个版本目录的 wrapper 顶部写清**档位与约束来源 / 镜像 / 是否复用缓存 / 上次实测日期与命令**。

### 2.4 `JAVA_HOME` 对照

| 分支线 | 构建 JDK |
|---|---|
| 26.1-26.2、26.3 | 25 |
| 1.20.2 起至 1.21.11 各区间 | 21 |
| 1.18.x、1.19.x、1.20-1.20.1 | 17 |
| 1.7.10、1.12.2 | 8（1.16.5 用 17 亦可） |

- **铁律：构建命令必须显式设 `JAVA_HOME`**。不设就会用系统默认 JVM：FG6 / 老线会以难查方式失败，javadoc 在 Java 25 下会卡死（§10）。
- **上传 / 脚本类任务可用的 JVM 档比"构建档"更宽**，但仍**按分支线就近取档**，不要一档通吃。特别地：**CurseForge 上传单独用 JDK 17**（§9.4）。

### 2.5 其它环境项（逐项勘察，不要沿用别人的值）

| 用途 | 怎么查 |
|---|---|
| Git 可执行文件 | `git --version`；路径含中文 / 空格时用全路径 |
| 本地代理 | `git config --get http.proxy` + 代理客户端的监听端口 |
| 测试实例根目录 | 先定位 `.minecraft`，再定位 `versions/`（§8.2） |
| 系统盘剩余空间 | 缓存 / 临时目录通常在系统盘；空间紧张时工具链会"空输出、无报错"地失败（§10） |
| MultiLoader 模板 | `https://github.com/jaredlll08/MultiLoader-Template`（按 MC 版本分分支） |
| 上传脚本 | 放**仓库之外**的私有目录，不入任何仓库（§9.4） |

### 2.6 环境类自检清单

- [ ] 四个 Java 档位逐个存在且**是真 JDK**（有 `javac`）；缺失则按降级策略重定位并更新快照日期。
- [ ] `gradlew` 不可用时，缓存发行版二进制路径仍存在，且版本与各仓库 wrapper 一致。
- [ ] 测试实例清单与实际目录数一致；抽查 1–2 个按库坐标法复核加载器（§8.2）。
- [ ] 每个版本目录的 wrapper 注释四要素齐全且日期不早于上次构建。
- [ ] 新 MC 大版本时：Java 表、模板分支、实例清单三处都要动。

### 2.7 离线优先

```bash
export JAVA_HOME=...                                  # 按 §2.4 取档
./gradlew build --offline --console=plain > build.log 2>&1
grep -c "BUILD SUCCESSFUL" build.log                   # 判定只看这一行
```

- 依赖都在 `~/.gradle/caches/modules-2/files-2.1/` 时离线能成功；离线失败再补依赖，不要先去联网。
- 若某个 `maven.modrinth` 坐标永远解析不了，把缓存里的 jar 放仓库根 `libs/`（**进 .gitignore**），并在**每个会编译 common 源码的模块**里都声明 `compileOnly files(...)` —— 只写 `common` 不够。

---

## §3 工程结构与元数据

> 细节（字段全集、最小模板、图标槽位分界、`pack.mcmeta` 逐版本取值、KnownPack 机制、资源扫描 API）：`references/03-metadata.md`

### 3.1 元数据字段放置（铁律）

- **`displayURL` / `iconFile`（26.2+）/ `logoFile`（≤26.1.2）放 `[[mods]]` 子表内**；放顶层不生效。
- **`issueTrackerURL` 是文件级字段，必须放顶层**；放进 `[[mods]]` 不被读取。两处都写 = 同一地址出现两次。
- **`description` 必须单行**：含真实换行会让 TOML 解析失败 → **整个 jar 被判"不是有效 mod 文件" → 模组完全不加载**。
- **26.3 起 `neoforge.mods.toml` 不再写 `modLoader` / `loaderVersion`**（javafml 默认、版本不限）。

### 3.2 图标与链接槽位（最高频踩坑）

| 模组菜单 | 版本范围 | 列表小图标 | 详情大图 / 横幅 |
|---|---|---|---|
| NeoForge 原生列表 | **26.2 起**双槽位 | `iconFile` | `bannerFile`（缺才回退 `logoFile`） |
| NeoForge 原生列表 | 26.1.2 及更早 | `logoFile` | 同一处 |
| Fabric Mod Menu | 全版本 | `icon` | 同左 |
| Forge 原生列表 | 1.20–1.20.1 | `logoFile` | 无独立横幅槽位 |
| Catalogue | 全版本 | `icon` / logo 字段 | 同左 |

- **`logoFile` 在 26.2 起已废弃**：沿用旧字段时 128×128 方图会被拉伸进**右侧横幅**槽位，左侧列表反而没有图标。**排查口诀**：图标跑到右边大图、左边空 → 改成 `iconFile`。
- **图标统一 128×128 PNG**（正方形，边长 ≤256）：Catalogue 用 `SQUARE + LESS_THAN_OR_EQUAL 256` 硬校验，超限抛异常后**显示问号且不写日志**。读 PNG IHDR（偏移 16/20 起各 4 字节大端宽高），别信看图软件缩略图。
- **Forge 1.18.x 例外**：`logoFile` 不能含 `/`，必须写根文件名，否则点开详情页崩。
- ⚠ **"文档查不到 ≠ 不存在"**：`iconFile` / `bannerFile` 曾被据"官方文档没写"判为不存在，被实机验收推翻。**此类字段以实机 / 反编译为准。**

**Mod Menu 徽章与自定义链接**（Fabric 专属）：
- 前置库必须声明 `"custom": {"modmenu": {"badges": ["library"]}}`，否则列表里没有"前置"标记。
- 纯客户端模组的"客户端"徽章由 `"environment": "client"` **自动**产生。
- 自定义链接只认 `custom.modmenu.links`（键 = 语言键或纯文本，值 = 完整 URL）；`contact.*` **只能**喂固定的主页 / 问题两个按钮。
- ⚠ **NeoForge 原生列表没有徽章机制、没有自定义链接列表**，两端要各有入口就得维护两套。

### 3.3 `pack.mcmeta`（进不了世界的经典原因）

- **基础口径**：模组不含数据包内容时，**不要**在 common 资源里放 `pack.mcmeta` —— 加载器会把带该文件的包当 datapack 注册，`pack_format` 错误直接导致"数据包错误"。
- **Forge 线例外**：**凡是 Forge 端分支都要补最小可用 `pack.mcmeta`**（只写 `pack_format` + `description`，**不写 `data` 段**），且**只放 forge 模块**，否则每次启动都有资源包元数据告警。
- 取值来自该版本原版 `version.json` 的 `pack_version`，**按分支编译版本取，不靠记忆**；1.21.9+ 起官方 MDK 用 `min_format` / `max_format`，**照抄官方 MDK 最稳**。
- 明确需要 datapack 时，`pack_format` **必须用 datapack 格式而非资源格式**。
- 相关报错：`无法为文件 Mod File: ...jar 加载有效的 ResourcePackInfo`、`Missing metadata in pack mod:<modid>`。

### 3.4 `processResources.expand` 的转义陷阱（最重要的一条）

- **根因**：Gradle 的 `expand` 底层是 Groovy `SimpleTemplateEngine`，它把模板正文编译进**双引号 GString**；于是源码里两字符的 `\n` 被 Groovy 词法解析成**真实换行**。模板里对**属性值**做的 `'\n' → '\\n'` 只保护被替换进来的值，**保护不了正文里手写的转义**。
- **判定手法**：解包 jar，用**严格** JSON 解析器（`node -e 'JSON.parse(...)'`）过一遍全部 `.json`；宽松解析器可能放过，不能自证。
- **修复**：受影响的 JSON / TOML **正文一律不写 `\n`、`\t`、`\u`**；需要分隔就用 ` | ` 写在一行。`gradle.properties` 的 `description` 同样**必须单行**。

### 3.5 资源扫描（NeoForge 侧，铁律）

- **不要用类加载器枚举模组资源**：`ClassLoader.getResources("assets")` 依赖 FML 模块类加载器对"目录型资源"的枚举能力，生产环境里经常**恒为空**且不报错。
- **正确写法**：走 FML 文件视图 `ModList.get().forEachModFile(modFile -> { var contents = modFile.getContents(); contents.visitContent((path, resource) -> ...) })`。`ModList` 在模组构造期已可用，不需要延后。
- Fabric 侧对应：`FabricLoader.getInstance().getAllMods()` + `mod.findPath("assets/")`。
- **判定顺序**（语言 / 贴图缺失但不报错）：① 解包 jar 确认 `assets/<modid>/...` 在；② 对照另一加载器的扫描计数；③ 查扫描实现是否依赖类加载器目录枚举。日志里 `M = 0` 或 `K = 0` 基本就是扫描器口径问题。

### 3.6 动态注册表与 KnownPack

- 通过数据包往动态注册表（`Registries.ENCHANTMENT`、`VILLAGER_TRADE` 等）加条目时，提供该文件的包**必须声明 `KnownPack`**，否则注册表被判 Experimental，进世界时弹"此世界使用实验性设置"。
- 机制：条目生命周期 = `knownPackInfo()` 存在 → `stable`，否则 `experimental`。模组自己创建的包常在 `PackLocationInfo` 里写 `Optional.empty()`，**那一处空值就是弹窗源头**。
- **修法**：构造 `PackLocationInfo` 时传入 `Optional.of(new KnownPack(<modid>, "<pack-id>", "<固定版本串>"))`；**版本串用固定字符串**，不要塞模组自身版本（`(namespace,id,version)` 是身份三元组，随版本变会让老存档的包身份失配）。
- **排除项**：这**不是** `pack.mcmeta` 格式问题、不是依赖区间问题、也不是"原版行为改不了"。

---

## §4 本地化与文案

> 细节（语言代码清单、键齐平、兜底 API 的版本边界、事故复盘）：`references/04-i18n.md`

### 4.1 语言覆盖

- 路径 `assets/<mod_id>/lang/<code>.json`，全部 **UTF-8 无 BOM**，**键集合与 `en_us` 完全一致**。
- **建议基线**：业务模组 ≥ 13 种语言（`en_us` / `en_gb` / `zh_cn` / `zh_tw` / `zh_hk` / `lzh` / `ru_ru` / `de_de` / `ja_jp` / `fr_fr` / `es_es` / `pt_br` / `ko_kr`）；库 / 前置模组最低 `en_us` + `zh_cn`。
- 语言代码有效性按**该版本自己的资源索引**复核（老版本可能没有 `lzh` / `zh_hk` / `en_gb`）。
- **已有项目保留其已有覆盖，不降级**；额外语言以项目记录为准，不要凭通用清单反推删减。

### 4.2 四条配套规则（都是实测踩过的）

1. **键必须齐平**：某个语言文件少键时，游戏会把 **key 原样当文本**显示。增删键后逐个语言文件对照 `en_us` 补齐，**不要只补中英**。
2. **语言值里禁止写 `§` 颜色码**：原版字体渲染器解析，但第三方 UI 库（自绘文本引擎）**不解析** ⇒ 颜色静默丢失。颜色改由**组件样式**给（`withStyle(ChatFormatting.RED)`）。
3. **服务端发给客户端的提示必须带兜底**：`Component.translatable(key)` 在**接收端**解析，接收端没装模组就显示原始 key。能用 `translatableWithFallback(key, 兜底)` 就用（1.20.2+）；用不了时**退回 `translatable(key)`**。
4. **语言 JSON 必须无 BOM**：`[System.IO.File]::ReadAllBytes(f)[0..2]` 不得是 `EF BB BF`；写文件用 `UTF8Encoding($false)`。

### 4.3 事故复盘：一句 `literal` 会让多语言整套失效

- **现象**：客户端切成中文仍只显示英文；语言文件本身完全正常。
- **根因**：为了"别让没装模组的客户端看到原始 key"，把提示改成了 **`Component.literal(写死文案)`** —— 内容是服务端拼好的固定字符串，**完全绕过客户端语言表**。
- **取舍表**：

| 写法 | 装了模组的客户端 | 没装模组的客户端 | 结论 |
|---|---|---|---|
| `literal(写死文案)` | ❌ 永远那一份语言 | ✅ 可读 | **错**，牺牲整套多语言 |
| `translatable(key)` | ✅ 按自身语言显示 | ⚠ 显示原始 key | **可接受** |
| `translatableWithFallback(key, 兜底)` | ✅ 按自身语言显示 | ✅ 显示兜底 | **首选**（有该 API 时） |

- **判定规则**：能用 `translatableWithFallback` 就用；用不了**一律退回 `translatable(key)`**，**永远不要用 literal 顶替** —— "没装模组的客户端看到 key"是可接受的降级，"所有客户端都失去多语言"是不可接受的回归。

### 4.4 版本 API 边界（实测）

| 能力 | 边界 |
|---|---|
| `Component.translatableWithFallback` | 1.19.2 不存在；1.20.2+ 可用 |
| `Component.literal` 静态工厂 | 1.18.2 不存在（用 `new TextComponent(String)`） |
| `Component.translatable(...)` 静态工厂 | 1.19.0 才引入；1.18.x 用 `new TranslatableComponent("key")` |
| `ServerPlayer#clientInformation()` | 1.20.2 才引入；1.20.1 及更早**没有**公开的客户端语言读取口，兜底固定 `en_us` |
| `sendSystemMessage(Component)` | 1.19 引入；1.18.x 服务端用 `sendMessage(Component, UUID)`、客户端 `displayClientMessage(Component, boolean)` |

### 4.5 本地化自检

- [ ] 业务模组语言数达标；库 / 前置 ≥ `en_us` + `zh_cn`；已有覆盖未被删减。
- [ ] 每个语言文件键集合与 `en_us` 求差集为 0。
- [ ] 全部语言 JSON：UTF-8、无 BOM、严格 JSON 可解析。
- [ ] 经 `expand` 展开的 JSON 正文没有 `\n` / `\t` / `\u` 转义。
- [ ] 玩家可见提示走可翻译组件，颜色由组件样式给，**无 `literal` 顶替**。
- [ ] **实机切换语言验证**：切到简体中文触发一次提示，确认显示中文而不是英文 / key。

---

## §5 Mixin 与 API 适配

> 细节（注入点校验手法、版本差异全表、事件桥接对照、可选前置模式、26.x 改名清单）：`references/05-mixin-api.md`

### 5.1 Mixin 注册（铁律）

- **加载器不会自动发现 jar 里的 `*.mixins.json`**，必须两侧都声明：
  - NeoForge / Forge：`META-INF/neoforge.mods.toml`（或 `mods.toml`）里的 `[[mixins]] config = "..."`
  - Fabric：`fabric.mod.json` 的 `"mixins": ["..."]`
- 漏一侧 = **另一侧全部 mixin 静默失效**，不报错、功能凭空消失。
- **1.20.1（FG6）例外**：Mixin 走 **jar manifest `MixinConfigs`**；`mixins.json` 必须写 `"refmap"`。

### 5.2 注入点必须对真实字节码核定

- **机理**：`@Inject` / `@Redirect` / `@ModifyArg` 的 `method` 只按**方法名**匹配；`@At` 里的 `target` 是**字符串字面量**，指向的类被换包或签名变了 → **匹配不到但不抛错**，回调永不执行。IDE 重命名与编译器都不会碰这个字符串。
- **判别手法（必做）**：对每个 `@At` 目标串，用 `javap -p -c -s <目标类>` 核对 **owner 类存在** + **描述符逐字一致**。
- **先确认这段 `@At` 不在注释块里** —— 历史遗留死代码里常见已失效的目标串。
- **防护**：跨大版本迁移后，把 `@At` / `@Slice` 目标串当作**必须逐条复核的清单**，与 `@Accessor` / `@Invoker` 的成员名同等级对待；**不要只改 import**。
- **`require = 1` 是把"目标不存在"从静默失效变成启动报错的唯一开关**。目标可能不存在时，`require = 0` 之外不允许"静默宽容"。
- **优先用不依赖参数描述符的注入器**：`@ModifyConstant` / `@ModifyArg` / `@ModifyReturnValue`。凡是把目标方法参数类型写进 handler 的（`@Inject` / `@Overwrite`）都可能因**编译器生成的桥接方法**匹配失败。

### 5.3 `remap` 与 `compatibilityLevel`

- **`remap`**：Forge **1.20.6+** 对 official 映射命名的方法 `@Inject` **必须 `remap = false`**（否则报 `Unable to locate obfuscation mapping`）；1.18.2 / 1.19.2 **保持 `remap = true`**（运行时仍用 SRG）。
- **`compatibilityLevel` 必须匹配加载器自带 Mixin**：Forge ≤1.20.6 自带 **0.8.5**（只认 `JAVA_17`，ASM 不支持 major 65）；Forge ≥1.21.1 自带 **0.8.7**（`JAVA_21` 合法）。
- **照抄 26.x 基线的 `JAVA_21` 到 Forge ≤1.20.6** ⇒ Forge 端**启动即崩**，而 Fabric 正常，极具迷惑性。
- **修法（分档）**：分支自带 0.8.5 时，forge 产物必须 **Java 17 字节码 + `JAVA_17`**。若分支是 21，**不能用 `options.release` / `targetCompatibility` 直接降**（会把解析属性 `org.gradle.jvm.version` 拉到 17，与 common 的 Java 21 变体冲突）；可行做法是编译后**改写 class 的 major 字节**，并同步改写 jar 内 `mixins.json`。
- **同源第二坑**：修完兼容级后若报 `Critical injection failure ... No refMap loaded`，是 jar 里**有 refmap 文件但 `mixins.json` 缺 `"refmap"` 键**（FG6 + SRG 必需）。

### 5.4 版本差异速查

| 事项 | 26.x | 1.21.x | 1.20.1 | 1.19.2 / 1.18.2 |
|---|---|---|---|---|
| 构建 Java | 25 | 21 | 17 | 17 |
| 入口判断服务端 | 构造函数注入 `Dist`（**禁止静态调 `FMLLoader.getDist()`**） | `FMLLoader.getDist()` | 同左 | 同左 |
| OP 权限判定 | `permissions().hasPermission(...)` | 1.21.9 起新 API；1.21.1 及更早 `hasPermissions(2)` | `hasPermissions(2)` | `hasPermissions(2)` |
| `Commands.performCommand` | `void` | 1.21.1 起 `void` | `int`（`ParseResults`） | 1.19.2 `int`；1.18.2 `(CommandSourceStack,String)I` |
| TOML 元数据文件 | `neoforge.mods.toml` | `neoforge.mods.toml`（1.20.6+） | `mods.toml` | `mods.toml` |

⚠ **权限体系重构的真实时点是 1.21.9，不是 1.21.2**。判据：用两侧版本实证，不要按"系列感"推测。

**26.x 改名清单（实测）**：

| 旧写法 | 26.x 写法 |
|---|---|
| `ResourceLocation` | `Identifier` |
| `GuiGraphics` | `GuiGraphicsExtractor` |
| `Screen#render(...)` | `Renderable#extractRenderState(...)` |
| `Screen#renderBackground(...)` | `Screen#extractBackground(...)` |
| `drawCenteredString` / `drawString` | `centeredText` / `text` |
| `FMLEnvironment.dist`（字段） | `FMLEnvironment.getDist()` |
| `Minecraft.setScreen` | `Gui#setScreen` + `setScreenAndShow`（后者多一次 `renderFrame`） |

- **取值手法**：`--offline` 构建失败后，反编译产物就在 `build/moddev/artifacts/minecraft-patched-*-sources.jar` 里，**解压后直接 grep 真实类名 / 签名**，不要拿旧版本的印象去猜。

### 5.5 事件桥接（Fabric API ↔ NeoForge / Forge）

| 能力 | Fabric API | NeoForge 等价 |
|---|---|---|
| 服务器启动 / 停止 | `ServerLifecycleEvents.SERVER_STARTED` / `SERVER_STOPPING` | `ServerStartedEvent` / `ServerStoppingEvent` |
| 服务端 tick | `ServerTickEvents.END_SERVER_TICK` | `ServerTickEvent.Post` |
| 指令注册 | `CommandRegistrationCallback` | `RegisterCommandsEvent` |
| 玩家加入 / 离开 | `ServerPlayerEvents.JOIN` / `LEAVE` | `PlayerEvent.PlayerLoggedInEvent` / `PlayerLoggedOutEvent` |
| 实体受伤后 / 死亡后 | `ServerLivingEntityEvents.AFTER_DAMAGE` / `AFTER_DEATH` | `LivingDamageEvent.Post` / `LivingDeathEvent` |
| 方块破坏后 | `PlayerBlockBreakEvents.AFTER` | `BlockDropsEvent` |
| 使用物品 / 方块 / 实体 | `UseItemCallback` / `UseBlockCallback` / `UseEntityCallback` | `PlayerInteractEvent.RightClickItem` / `RightClickBlock` / `EntityInteractSpecific` |

- **Forge 的 EventBus 换代**：EventBus **7**（Forge 1.21.8 起）**不再接受 `register(Class)` 注册"单监听方法"的类**（会崩 `Only a single listener found in class ...`）。改用事件类自带的 `BUS`：`ClientPlayerNetworkEvent.LoggingIn.BUS.addListener(X::onLogin)`，并去掉 `@SubscribeEvent`。**判据：查实例 JSON 里的 `eventbus` 版本（6.x 仍可用旧写法）。**

### 5.6 可选前置（反射门面 + 声明可选 + 不打包）

1. **探测门面**：`OptionalDepLink.isLoaded()` —— 内部 `Class.forName("<前置的标记类>")`，纯反射。
2. **禁止在关键类字节码里静态引用可选前置的类型**：任何在无该前置的环境下会被加载的类，其常量池里不能出现那些类型，否则 `NoClassDefFoundError`，**运行时守卫无效**。
3. **客户端注册**：Forge 端**不要**用 `@Mod.EventBusSubscriber` 注解，也**不要** `EVENT_BUS.register(Class)`（都会触发 ASM 扫描加载可选前置类）；改用 `addListener(方法引用)`。
4. **依赖声明**：Forge `mods.toml` 用 `mandatory = false`；Fabric **不写进 `depends`**，改用 `suggests`；编译期依赖留在 build.gradle，但**不要 `include` 打包**。

---

## §6 构建与依赖管理

> 细节（档位完整依据、FG6→FG7 换代、模板结构、依赖声明事故、编码配置、故障速查）：`references/06-build.md`

### 6.1 为什么档位不能一刀切

Gradle 大版本会移除老插件依赖的 API，**不存在"一个 Gradle 版本通吃所有 MC 线"**。

| MC 线 | Forge 插件 | 备注 |
|---|---|---|
| ≤ 1.21.8 | FG6 `[6.0.24,6.2)` | **FG6 硬性拒绝 Gradle 9** |
| 1.21.9-1.21.11 | FG7 `[7.0.3,8)` | 官方 MDK 用 Gradle 9.x |
| 26.1-26.3 | FG7 `[7.0.17,8)` | 官方 MDK 用 Gradle 9.x |

**FG7 的 DSL 与 FG6 差异**：没有 `fg` 插件、没有 `minecraft` 扩展的旧形态；`dependencies` 里直接 `implementation minecraft.dependency(...)`。

**FG7 三个坑**：
1. **不要挂 mixin 注解处理器**（official 映射下它会去找混淆映射，直接报错）；用 `compileOnly 'org.spongepowered:mixin:0.8.7'`。
2. **`processResources` 的 `expandProps` 里不能有 `group` 键**（Gradle 9 会把它当任务属性）。
3. **降字节码不能用 `options.release`**（会把解析属性拉低，与加载器构件的 JVM 声明冲突）；要降就**编译后改写 class major**。

**工程组织**：主工程 Gradle 大版本与 Forge 端冲突时，forge 端拆成**独立 Gradle 工程**（自带 wrapper），主工程 `settings.gradle` 注释掉 `include('forge')`。
- **判据（1 秒判断）**：看 `<分支>/forge/settings.gradle` **在不在** —— 在 ⇒ 独立工程，用 forge 目录自己的 wrapper，任务是 `jar`；不在 ⇒ 子模块，用分支根的 wrapper，任务是 `:forge:jar`。写错的表现是 `Task 'jar' not found in root project`，**别误判成插件 / 版本问题**。

### 6.2 MultiLoader 模板

- 上游 `https://github.com/jaredlll08/MultiLoader-Template`，**按 MC 版本分分支**；新版本移植要自行创建对应版本文件夹。
- 低版本移植时**删掉 parchment**（版本不存在会导致下载失败）。
- **`.gitignore` 别假设模板已带**（见 §9.1）。
- 模板结构演进：`buildSrc` → `build-logic` 发生在 26.x。
- **升一个小版本时通常只改 4 个文件**：`gradle.properties`、根 `build.gradle`、`gradle/wrapper/gradle-wrapper.properties`、`common/build.gradle`。

### 6.3 依赖声明（含真实事故）

- **Fabric API 是前置而非内嵌**：`implementation(...)` 不会被打进 jar（**只有 `include` 才内嵌**）；用到 API 类就**必须在 `fabric.mod.json` 的 `depends` 里显式声明**。⚠ **前置声明不给"友好提示"，是加载器直接拒绝启动** —— 别把机制写错成承诺。
- **Fabric 线必须用 `modImplementation`** 引 fabric-loader / fabric-api，否则报"无法访问 `class_XXXX`"。
- **加载器模块二次编译 common 源码时 classpath 可能缺 Mixin** ⇒ fabric 模块要显式 `compileOnly` Mixin。
- **Loom 插件 id 的版本分界**：`1.14` 之前只有旧 id `fabric-loom`，`1.15+` 才是 `net.fabricmc.fabric-loom`；`Loom <1.14` 不给 `mappings` 默认值，**必须显式写**。
- **ASM 版本冲突**：NeoForm 可能把 `org.ow2.asm` 硬钉某个版本，而 sponge-mixin 拉更高版本 ⇒ 用 `resolutionStrategy.force` 统一。

### 6.4 编译与编码配置

```groovy
tasks.withType(JavaCompile).configureEach { options.encoding = 'UTF-8' }
tasks.withType(Javadoc).configureEach {
    options.encoding = 'UTF-8'
    options.addStringOption('Xdoclint:none', '-quiet')
}
```

- 不设 `options.encoding` 时，Windows 默认 GBK 会把中文注释读坏，报"不可映射字符"。
- **`withJavadocJar()` 能在 Java 25 下卡死整个构建**：doclint 严格模式 + 中文注释 + record / 枚举缺 `@param` 直接 fail，把 `:neoforge:jar` 一起拖垮。不需要 javadoc 就**移除**。
- **移除 sources / javadoc 后的连带问题**：模块脚本里 `configurations.named('sourcesElements')` 找不到配置直接失败 ⇒ 改成 `configurations.matching { it.name == variant }.configureEach { ... }`。
- **模块内 `base { archivesName = ... }` 覆盖要删掉**，交给约定插件统一拼。

### 6.5 构建期常见故障（速查）

| 现象 | 处理 |
|---|---|
| `找不到主类 GradleWrapperMain` | 直接用缓存里的发行版二进制 `~/.gradle/wrapper/dists/gradle-<v>-bin/<hash>/gradle-<v>/bin/gradle` |
| 改了 `gradle.properties` 仍 `UP-TO-DATE` | 加 `--rerun-tasks` 强制重跑 `processResources` |
| 关了 sources / javadoc 后配置名报错 | `configurations.matching { ... }.configureEach { ... }` |
| **构建成功 ≠ 游戏在跑新代码** | 比对 `build/libs/*.jar` 与实例 `mods/*.jar` 的**时间 / 体积** |
| 路径含空格时脚本没加载 | **逐个参数加引号**；`-P` 值含点号时整体加引号 |
| `unsupported class file major version` | 先核对 §2.4 的 `JAVA_HOME` 对照 |

### 6.6 构建自检

- [ ] wrapper 档位与 §2.3 一致，注释四要素齐全。
- [ ] 本地缓存里确有该档（**列目录核实，不凭记忆**）。
- [ ] `gradlew projects` 与目标模块 `jar` 都真跑过，判定看 `BUILD SUCCESSFUL`。
- [ ] `JavaCompile` / `Javadoc` 的 encoding 已设；不需要 javadoc 时已移除 `withJavadocJar()`。
- [ ] Fabric 依赖用 `modImplementation`；用到 API 类时 `depends` 已声明 `fabric-api`。
- [ ] 产物名符合命名规则，且无重复版本号。

---

## §7 多版本移植策略

> 细节（触点差分法、一手来源清单、探针脚本、range 操作口径、老版本线触点、波次编排）：`references/07-porting.md`

### 7.1 L1 / L2 / L3 三层判据

| 层 | 卡什么 |
|---|---|
| **L1 源码触点** | 真正调用的 MC / 加载器 API 的签名与包路径 |
| **L2 工具链硬边界** | Java 大版本、映射 / 混淆体系、模板结构 |
| **L3 加载器可用性** | 该 release 有没有对应加载器的构建 |

> `L1 决定"能不能并"，L2 决定"该不该分"，L3 决定"分发时能挂哪几个端"。`
> **合并只在 L1 有实测零差异证据时才做；`minecraft_version_range` 永远不得宽于已实测区间。**

### 7.2 触点差分法（找耦合面）

1. 取全部历史分支，两两 `git diff -w -- common/src/main/java`（`-w` 过滤 CRLF / 缩进噪声）。
2. 把"变化的那几行"抽出来，就是**全部耦合面**。
3. **先量出耦合面，再谈分支数** —— 顺序反了就会不自觉"每个 MC 版本开一条分支"。

### 7.3 一手来源（可复现）

| 要查什么 | 来源 | 取哪个字段 |
|---|---|---|
| MC 发行清单、Java 要求 | Mojang `version_manifest_v2.json` + 逐版本 version JSON | `javaVersion.majorVersion`（**逐版本抓**） |
| NeoForge 可用区间 | `maven.neoforged.net/releases/net/neoforged/neoforge/maven-metadata.xml` | `<version>` 前缀 = MC minor |
| NeoForge 1.20.1 分叉（**必须同查**） | `.../net/neoforged/forge/maven-metadata.xml` | 只有 1.20.1 |
| Forge 可用区间 | `maven.minecraftforge.net/net/minecraftforge/forge/maven-metadata.xml` | 版本串 `-` 前段 |
| Fabric API 覆盖 | `maven.fabricmc.net/.../fabric-api/maven-metadata.xml` | **必须用锚定正则 `+1.20$`** |

- **二手转述（百科 / 论坛）只用来提假设，不用来做结论。**
- ⚠ **正则不加 `$` 会造出幻觉数据**：`+1.18.1` 的非锚定匹配会把 `+1.18.12`、`+1.18` 一并计入，从而"查到"根本不存在的构建。

### 7.4 range 的三类虚假承诺（重灾区）

1. **上界虚高**：声称覆盖到远高于实测能力的版本 —— 用户装了崩，比不装更糟。**唯一防线是 L2 / L3 实测，不是感觉。**
2. **下界虚高**：把已实测兼容的更低版本排除在外 —— 白丢用户，且**只会以静默方式发生**（列表里直接不显示，没人来报 bug）。
3. **范围宽于证据**：写的时候顺手，出事的时候没人记得当初没测。

> **下笔时把"我测过哪些 release"当成 range 的唯一依据。** 上界与下界同样危险。

### 7.5 合并的三份证据

合并只在三种证据下做：① 两分支 `git diff -w` 源码零差异；② 触点全部同形态；③ L2（Java / 映射 / 模板结构）同档。**三者缺一即分叉。**

### 7.6 落地前必须先跑的探针（未采样区间不许拍脑袋）

```bash
javap -p -s <解压后的 Commands.class>      | grep -A2 performCommand
javap -p -s <解压后的 ServerPlayer.class>  | grep -A1 setGameMode
javap -p -s <解压后的 ServerPlayer.class>  | grep -i permission
```

- **判定规则**：与相邻分支全部一致 → 并入并放宽 range；任一不同 → 另开分支。
- 老版本线要用**它自己的映射名**重跑（1.16.5 与 1.12.2 / 1.7.10 的类名完全不同）。
- **形态边界必须两侧实证，不能按"系列感"推测。**

### 7.7 一条分支的版本声明由 6 处构成（漏改任何一处都会"能跑却装不上"或"装得上却崩"）

| 声明处 | 位置 | 放宽时注意 |
|---|---|---|
| `minecraft_version` | `gradle.properties` | 编译目标，仍是**单个**版本；放宽靠 range |
| `minecraft_version_range` | 同上 | 上界对齐下一个 L2 / L1 硬墙；**下界前移要逐个 release 有 L1 证据** |
| Fabric `depends.minecraft` | `fabric.mod.json` | 语法是 `~26.3` 这类**前缀约束**，跨 minor 必须改写 |
| Fabric `depends.fabric-loader` | 同上 | 有硬下限（随 MC 版本抬高） |
| Fabric `depends.fabric-api` | 同上 | 用到 API 类就是**真前置**，必须显式声明 |
| NeoForge / Forge 元数据 | `mods.toml` 或 `neoforge.mods.toml` + loader range | **改名时点在 1.20.6 前后** |

### 7.8 老版本线（1.7.10 / 1.12.2 / 1.16.5）

- **各只适配一个 release**，不出 Fabric / NeoForge 端，但**留好钩子**：四项版本变量提为唯一改动点、range 先写单点、语言键保持同名、触点差异收敛到**一个平台适配类**。
- **钩子怎么留**：① 版本变量提为唯一改动点；② range 先写单点；③ 语言文件 / 配置读写保持同名键；④ 注入点未定前先按 §7.6 核一遍；⑤ 业务判定写 common，老版本线只实现桥。

### 7.9 执行波次（每波结束都要：产物 MD5 → 部署 → 实机验收 → 才允许推分支）

| 波 | 内容 | 退出条件 |
|---|---|---|
| W0 探针 | 触点探针 + 构建档位实测 | 每个 release 有 `javap` 记录；可疑区间的拆分与否定稿 |
| W1 老版本线 | 按成功率从高到低 | 每条：`build` 出 jar → 装进实例 → 行为与基线一致 → 无 `NoClassDefFoundError` |
| W2 最老的现代分支 | 下界放宽，6 处声明齐改 | 已有实例实机通过；新增覆盖的 release 至少各一次实机 |
| W3 相邻两条 | 一条放宽、一条收窄 + 下界前移 | 新旧下界各实机一次 |
| W4 新建区间（Java 跨界） | 新建分支 | 该区间每个 release 至少一次装载成功 |
| W5 中段区间 | 收窄 + 前移 / 新建 | **无实例的只能编译验证，声明范围必须诚实标注未实机** |
| W6 合并 + 全系列回填 | 零差异分支合并；回填基线增量 | 兜底 API 引入版本已确认；**未确认前禁止回填** |

---

## §8 测试部署与验收

> 细节（实例判定法、部署流程、MD5 回读、验收清单逐条）：`references/08-deploy-qa.md`

### 8.1 部署三条规则

1. **构建到 `build/libs` 后**，产物在 `<module>/build/libs/`；**没有对应测试版本时报告**（不要猜目标）。
2. **仅首次构建前**问用户确认目标实例（放哪个 / 双端）；确认后**后续构建直接自动拷贝到该实例，不再每次询问**。用户改目标时再问。
3. **旧版本 jar 处置**：`build/libs` 内**永不删除 / 移动 / 备份 / 记录**；实例 `mods/` 内被替换的**必须删**（同 modId 双版本会被 FML 拒载）。

### 8.2 实例判定（用库坐标，不用目录名）

| 库坐标 | 加载器 |
|---|---|
| `net.neoforged.fancymodloader` / `net.neoforged:*` | **NeoForge** |
| `net.fabricmc.loader` / `net.fabricmc:*`（非 sponge-mixin） | **Fabric** |
| `net.minecraftforge:*` / `cpw.mods.modlauncher` | **Forge** |

- ⚠ **判定加载器时不要 grep `"fabric"`**：NeoForge 的 `version.json` 里含 `net.fabricmc:sponge-mixin`，**"看见 fabric 字样"绝不能判成 Fabric**。
- **只认你约定的测试实例前缀**；整合包 / 正式游玩实例、以及目录名含你设定的禁部署标记的实例**一律禁碰禁写**。
- ⚠ **名字里带"测试"两个字的目录不一定是测试实例** —— 命名规则是**前缀**，不是"包含"。

### 8.3 部署流程

1. **部署前核对三件事**：① 目录后缀的加载器与 jar 一致；② jar 支持的 MC 版本与实例一致；③ 前置库已在实例中且版本满足 `depends` 区间。
2. **确认无游戏进程**；游戏运行中则中止并等用户关闭。
3. 报告清单：放什么文件、放到哪个 `mods/`、同名旧文件如何处理。
4. 删除实例 `mods/` 内被替换的旧 jar。
5. 部署后用 `Get-FileHash -Algorithm MD5`（或 `md5sum`）比对实例里的 jar 与 `build/libs` 产物。
6. 把"已部署到 X，待实机验收"写进项目决策记录（**只记结论，不建"修复记录"目录**）。
7. **部署成功并记录 MD5 之前不得结束输出。**

### 8.4 验收清单（每个 jar 都要过）

- [ ] 模组列表出现本模组，**小图标在左侧**、版本号正确、许可证正确；主页 / 反馈按钮可点。
- [ ] 进服 / 局域网：行为与基线语义一致；聊天提示在对应语言下正确。
- [ ] 权限侧：OP 不受限。
- [ ] **混装**：与常见第三方模组共存时启动不崩、它们自己的客户端指令与按键不被误拦。
- [ ] 卸载本模组后其余模组仍能进世界（证明没留残留状态）。
- [ ] 涉及资源的改动：能正常进入世界，**无"数据包错误"弹窗**。
- [ ] 配置文件改动能热重载（若模组支持）。
- [ ] 服务端类模组：换 jar 后**重启服务端**才重扫。

> **必须分清阶段**：构建成功 ≠ 部署成功 ≠ 实机验收。汇报时逐项声明："已构建""已部署，待实机验收""已验收""已上传"。

---

## §9 Git 与平台发布

> 细节（分支管理、推送红线、GCM 挂死、CurseForge / Modrinth 全链路、Commit 规范全文）：`references/09-git-upload.md`

### 9.1 GitHub：多版本分支管理

- **每版本独立目录 = 独立 git 仓库**，**分支名 = 目录名**（或区间标签）。
- **推送前先补 `.gitignore` 并验证 buildlike = 0**：`git ls-files` 里不得出现 `build/`、`.gradle/`、`run(s)/`、`out/`、`bin/`、`.idea/`、`.vscode/`。
- ⚠ **别假设模板已带 `.gitignore`** —— MultiLoader 模板生成的版本目录通常没有。

**两种推送策略（选一种并写进 `AGENTS.md`）**：

| 策略 | 适用 | 红线 |
|---|---|---|
| **快进推送**（默认推荐） | 多人协作、或分支可能被别处更新 | 被拒绝时先 `fetch` 看差异，**不要立刻强推** |
| **强推覆盖** | 各版本目录是**彼此的无关历史根提交**，本地永远是最新最完整的那份 | ① **推前必须 `ls-remote` 比对远端哈希**；② **禁止强推 `main` / `master`**；③ 用户未明确下达推送指令时不得推送 |

- **强推前比对哈希**：与该分支**上次推送后记录的哈希**比对，一致才可强推；不一致 → **停手问用户**（说明远端多了什么、是否先取回 / 存备份分支）。已发生过覆盖事故。
- **补救顺序**：① 找到推它的那份工作副本，先 `git push origin HEAD:refs/heads/backup-<sha>` 存成备份分支；② 试托管平台的提交页；③ IDE 本地历史。**先保住再说，别急着 force。**

### 9.2 推送卡死（GCM 无 TTY）

- **现象**：`git push` 永久挂起，既不成功也不报错。
- **判定**：进程树长期停在 `credential-helper-selector get`；确认没弹凭据对话框；`ls-remote` 旁证网络正常。
- **修复**：非交互推送加 **`-c credential.interactive=false`**（单用 `GIT_TERMINAL_PROMPT=0` 挡不住挂起）。
- **判定成功**：`git ls-remote --heads origin <branch>` 的远端哈希 == `git rev-parse HEAD`。**`git push | tail` 的退出码是 `tail` 的，exit 0 不等于推送成功。**

### 9.3 凭据红线（CurseForge 与 Modrinth 通用）

- 键名 `curseforge_api_key` / `modrinthApiKey`，放用户**全局** `gradle.properties`。
- **AI 严禁读取、打印、检索该文件**；key 绝不写进任何仓库文件，也不得解析出来后传给 curl / 任何外部命令。**只允许 Gradle 内部 `findProperty` 消费。**
- "把 key 读进变量交给 curl"**不作为读取授权**；优选 Gradle init 脚本（只检查键的**存在性**、不打印）。

### 9.4 CurseForge

- **两条链路**：CurseGradle 插件（配置可保留、**上传不可用**）与 **init 脚本 `cfUpload`**（`java.net.http.HttpClient` + **显式** HTTP CONNECT 代理 + legacy 端点）。
- **上传 403 的真相**：CurseGradle 自建 Apache HttpClient，**既不读系统属性也不读默认 `ProxySelector`** ⇒ 没走代理 ⇒ 撞 Cloudflare 人机挑战。
- ❌「legacy 端点已废弃」——**错**，该端点完全正常。
- ❌ **不要用 SOCKS5**：Java 的 SOCKS 实现在本地解析 DNS 再把 IP 交给代理 ⇒ TLS 握手失败。
- ✅ **用 HTTP CONNECT 代理并"显式"指定**：只设 `https.proxyHost` 系统属性不够，必须显式 `ProxySelector.setDefault(...)` 或在请求里显式 `.proxy(...)`。
- **只认响应里的 File ID**：拿到 `{"id":<数字>}` 才算发布完成。
- **前置依赖必须写进上传 metadata 的 `relations.projects`**（`slug` + `type`），**不要依赖网页搜索框手工补**。`relations` 只能"完全不写"或"至少 1 项"（空数组会被拒）。
- **`metadata` 必须是普通表单字段**（`name="metadata"`，**绝不能带 `filename=`**）；`gameVersions` 是**数值 ID 数组**。
- **changelog 文件会被原样公开**：只写版本小标题 + 本次真实改动，**禁止** H1 模组标题、流程说明、AI 维护规则。
- **执行时机（铁律）**：配置可提前写好，但**只有用户实机通过并主动下达指令后才允许上传**。

### 9.5 Modrinth

- **一个 version 只能一个加载器**（fabric / neoforge 分别发），与 CF"一个文件带多个 loader 标签"完全不同。
- **`data` 必须是 `application/json`**：PowerShell 的 `Invoke-RestMethod -Form` 发不出 MR 能解析的 `data`，用 `curl.exe -F "data=<payload.json;type=application/json"`（两个点缺一不可）。
- `dependencies` 空数组**合法**（与 CF 的 `relations` 相反）。
- `environment` 是**版本级字段**，请求体只接受英文枚举。
- `PATCH /version/{id}` **只改属性、换不了产物本体**；要换 jar 只能删掉版本重发 —— **删除是破坏性操作，必须先问用户**。
- **未建 MR 项目的模组不上传**，也**不作为 MR 前置依赖**。

### 9.6 发布自检

**GitHub**
- [ ] `.gitignore` 已覆盖构建产物；`git ls-files` 计数 buildlike = 0。
- [ ] 用户已**明确下达推送指令**。
- [ ] 推送命令带 `-c credential.interactive=false`；推送后 `ls-remote` 回读哈希并与本地 HEAD 比对。
- [ ] 强推前已比对远端哈希与基线。

**CurseForge**
- [ ] changelog 只有版本小标题 + 条目，无 H1、无流程文字。
- [ ] **未**读取用户全局 `gradle.properties`。
- [ ] 代理可用；命令里显式带了正确的**前置构建任务**，路径参数逐个加引号。
- [ ] 前置依赖已写进 `relations.projects`，上传后回读确认。
- [ ] 拿到 **File ID** 并已记录。

**Modrinth（仅当该项目在 MR 有项目时）**
- [ ] 已对比既有版本清单，确认本次要传的版本。
- [ ] fabric / neoforge **分别**建 version。
- [ ] 用 `curl.exe` 上传（`Invoke-RestMethod -Form` 必 400）。
- [ ] 拿到 **version id** 并已记录。

### 9.7 Commit 规范（摘要）

- **英文、Conventional Commits**：`<type>(<scope>): <description>`。
- **type**：`feat` / `fix` / `perf` / `refactor` / `build` / `ci` / `docs` / `style` / `test` / `chore` / `revert`。**按实际代码变化的性质判断**，不看文件所在目录。
- **scope**（可选）：按加载器（`common` / `fabric` / `neoforge` / `forge`）或按功能模块（`block` / `item` / `entity` / `recipe` / `registry` / `network` / `gui` / `render` / `config` / `command`）。无法提供有效信息时**宁可不用 scope**。
- **description**：小写开头、动词开头（`add` / `remove` / `update` / `fix` / `prevent` / `allow` / `replace` / `simplify` / `refactor` / `optimize`）、**不以句号结尾**、**不模糊**。
- **Body 只在 description 不足以说明时才写**，写具体做了什么，不写开发过程、不写 AI 式总结。
- **Breaking Change**：改 modId / 删已注册对象 / 改注册名 / 改 NBT 或数据组件结构 / 改网络格式 / 抬最低 MC 或 Loader 版本 / 删公开 API → `feat!:` + `BREAKING CHANGE:` 脚注。
- **禁止 AI 痕迹**：`Implement a comprehensive...` / `Improve and enhance...` / `Various improvements and fixes` / `Refactor the codebase to improve maintainability` / `Enhance overall performance and stability`。
- **禁止虚构**：Issue 编号、性能百分比、测试结果、兼容性结论。

---

## §10 排障案例库

> 完整案例（症状 → 根因 → 判定手法 → 修复）在 `references/10-troubleshooting.md`，含 40+ 条实战记录。

### 10.1 通用 Debug Flow（12 条）

1. 看完整 `latest.log` / 崩溃前输出，**grep `not a valid mod file`、`ParsingException`、`Invalid modId found`**。
2. 确认 mod 是否真的出现在 Mod List（不在 = 元数据解析失败）。
3. 依次检查 TOML 换行 / `[[mods]]` 字段位置 / **modId 是否含连字符** / `pack.mcmeta` / Mixin remap / 图标与链接。
4. 重构建前**清掉实例里的旧同名 jar**，避免缓存干扰验证。
5. Java 版本不匹配时，先核对 §2.1 再重新构建。
6. 忽略崩溃后的 `CancellationException` / `IOException: Stream closed` 噪音（是崩溃后的清理动作，非根因），**往更早日志找 `Invalid modId`**。
7. **构建成功 ≠ 游戏在跑新代码**。排查"改了还是崩"时，**第一步永远是比对 `build/libs/*.jar` 与 `mods/*.jar` 的时间 / 体积**。
8. **"能进游戏但功能整体失效" → 先查 mixin 有没有被加载**。
9. **"语言 / 贴图没加载，但游戏能进" → 先查资源扫描计数**。
10. **"我这么配是合法的，但加载器说它非法" → 先去原版字节码确认那条约束是否真实存在**。
11. **"注入没生效，但启动日志一切正常" → 先查 `@At` 里的目标字符串**。
12. 改一处先分析根因；不擅自改用户没要求的配置。

### 10.2 症状索引（高频）

| 症状 | 去哪查 |
|---|---|
| 模组完全不加载 / 不是有效 mod 文件 | §3.4 `expand` 陷阱；`Invalid modId found` → §1.3 |
| 图标跑到右侧大图、左边列表空 | §3.2（`logoFile` → `iconFile`） |
| 只有 Catalogue 显示问号 | §3.2（图标超 256 或非正方形） |
| 主页 / 反馈按钮点不动 | §1.2（字段放错层级） |
| Fabric 侧进服报晦涩错误 | §6.3（缺 `depends.fabric-api`） |
| 进不了世界、报数据包错误 | §3.3 `pack.mcmeta`；`KnownPack` → §3.6 |
| 能进游戏但功能整块不工作 | §5.1 mixin 未加载 / §5.2 注入点失效 |
| 语言 / 贴图缺失但不报错 | §3.5 资源扫描 |
| 多语言失效（切成中文仍英文） | §4.3 `literal` 事故 |
| 改代码没效果 / UP-TO-DATE | §6.5 |
| 构建卡死 / javadoc 拖垮 | §6.4 |
| `gradlew` 找不到主类 | §6.5 |
| 依赖报"无法访问 `class_XXXX`" | §6.3（`modImplementation`） |
| ASM 版本冲突 | §6.3（`resolutionStrategy.force`） |
| 构建报编码不可映射字符 | §6.4（`options.encoding`） |
| `git push` 永久挂起 | §9.2 |
| 上传 403 | §9.4 |
| "任务完成 exit 0"但实际失败 | §0.5（管道退出码） |
| 多加载器 jar 在 Forge 上启动即崩 | §5.3（`compatibilityLevel`） |
| 第三方模组崩溃被当成自己的 bug | 先看崩溃类名是不是第三方包名 |
| 命令无输出 / 无报错地失败 | 先查系统盘剩余空间 |
| Forge 高版本构建连环坑 | §6.1（FG6→FG7） |
| 界面之间来回切换时闪一下 | 退场动画**结束帧**仍在画自己的背景 |
| 附魔描述有的白字、有的灰字 | 描述行**没带样式**；原版 1.20.1 本就没有描述 |
| 界面元素"画上去看不见" | 先问三件事：**深度测试开着吗？谁在我之后画？有裁剪吗？** |
| 一进世界客户端就崩（26.x） | 缺 `META-INF/services` / 注册键未设 / 漏 `include` |
| 物品全裸键 / 洋红块 / 界面无贴图 | **模组资源整体没加载**（缺 `pack.mcmeta`） |
| 遮罩 / 背景"闪一下" | 背景被画了第二遍，或 `setScreenAndShow` 强制渲染一帧 |

### 10.3 排障心法（复用性最高）

见 §0.6。补充三条：

- **别被日志摘要骗**：`xxx has failed to load correctly` 的**真正错误行在它上面几行**，且内容往往是下游模组提供的 ID。
- **不要因为库写了 fail-fast 就认定自己的配置有问题** —— 约束本身可能才是 bug（反汇编原版确认后，把校验降级为 WARN）。
- **重跑一次再排查**：服务端偶发 5xx 属瞬时错误；"同一命令原样重跑就成功"往往不是代码问题。

---

## §11 结论纪律

> 完整对照表与待核实清单：`references/11-verdicts.md`

### 11.1 易错结论对照（旧说法 → 现行口径）

| 旧说法 | 现行口径 |
|---|---|
| "缺 JDK 8 ⇒ 老版本线一定建不了" | 目标字节码不是瓶颈（`--release 8` 三档全成功）；真缺口是**运行 Gradle 的 JVM** |
| "`net.neoforged:neoforge` 起点 1.20.2 ⇒ 1.20.1 没有 NeoForge 端" | **查了错的坐标**：该组织另有 `net.neoforged:forge`，专发 1.20.1 的 Forge 分叉 ⇒ 判断"某加载器在某版本不存在"必须反查全部相关坐标 |
| "`pack.mcmeta` 一律删掉" | 拆分：**NeoForge / Fabric 线**不带数据包内容就不要放；**Forge 线必须补**最小版，且只放 forge 模块 |
| "枚举模组资源用类加载器 `getResources("assets")`" | **实测推翻**：走 FML 文件视图 |
| "清掉旧 jar 再构建" | 目录边界分开：`build/libs` **原位保留永不删**；只清**实例 `mods/` 中被替换的版本** |
| "`version` 带 MC 前缀" | `version` 为**纯模组版本**；MC 版本只进 jar 文件名与依赖区间 |
| "上传任务统一绑 `shadowJar`" | 按项目真实可分发产物（`jar` / `remapJar` / `shadowJar`）判断，**不能跨项目硬套** |
| "Gradle 固定三档且全已缓存" | 档位由**插件约束**决定；缓存状态**每次重列目录核实** |
| "`java_version=17` ⇒ Gradle 能跑在 17" | **分开判断**：Loom 1.13+ / MDG 2.0.123+ 的字节码是 Java 21 ⇒ Gradle 必须跑在 **JVM 21+** |
| "1.20.x 的权限判定照抄 1.21.x 的 `allowsSelectors`" | **错**：1.20.x 只有 `hasPermission(int)`；`@Inject` 默认不校验方法存在 ⇒ 必须显式 `require = 1` |
| "权限 API 改动时点是 1.21.2 / 26.1" | **实测 1.21.9** |
| "所有版本都该验左侧小图标" | 双槽位是 **26.2+** 规则；更早版本按 `logoFile` 与加载器能力验收 |
| "文档查不到 `iconFile` / `bannerFile` ⇒ 字段不存在" | **文档缺席 ≠ 事实不存在**，以实机 / 反编译为准 |
| "1.20.2 之后只有 NeoForge，Forge 端可以砍" | 1.20.4 / 1.20.6 有**真 Forge** 构建；`neoforged:forge` 只做 1.20.1 |
| "命令没报错、退出码 0 ⇒ 成功" | 管道退出码不可信 ⇒ 一律带外回读 |
| "改一个功能需要就顺手改库的公开 API 签名" | **只新增、不改既有签名** |

### 11.2 待核实清单（动手前先查，别沿用旧快照）

| # | 待核实项 |
|---|---|
| 1 | 各 JDK 路径当前是否存在、是不是真 JDK |
| 2 | Gradle 各档位**当前缓存实况**（每次动手都要重列目录） |
| 3 | 目标版本线的元数据文件名（`mods.toml` / `neoforge.mods.toml`）边界 |
| 4 | 实例清单现状、各端在目标 release 上的验收缺口 |
| 5 | 老版本线构建路线（现代 Gradle + `release=8` 是否可行、要不要 JDK 8） |
| 6 | 老版本线的触点描述符（映射名与现行版本不同） |
| 7 | 未采样区间的触点是否与相邻分支一致 |
| 8 | 各平台 API 与规则是否变化（CF legacy 端点、MR `environment` 枚举、图标尺寸限制） |
| 9 | 元数据在本版本线的 `groupId` / loader range 具体值 |

### 11.3 元原则

1. **一手优先**：maven 元数据、Mojang version JSON、`javap`、解包 jar、实例 JSON > 文档 > 二手转述。
2. **文档缺席 ≠ 事实不存在**；"文档写了"也不等于现在成立。
3. **坐标要查全**：判断"某加载器在某版本不存在"必须把相关 artifactId 全部反查。
4. **退出码要带外验证**；工具的"成功"字样不是证据。
5. **范围声明以证据为界**：上界与下界同样危险。
6. **口径要标日期**：路径、缓存、实例、平台字段都会变，任何结论都应能回答"什么时候测的、用什么命令测的"。

---

## §12 Blockbench 美术制作

> 细节（踩坑表、镜像平铺判据、多层染色反推全过程）：`references/12-art.md`

### 12.1 硬规范（违反必出问题）

1. **像素密度 1:1**：1 模型单位 = 1 纹理像素。每个面的 UV 矩形**宽高必须等于该面的实际尺寸**，不要给不同尺寸的面套同一个 UV。
2. **UV 逐面指定**（`box_uv = false`）；改 UV 前 `Undo.initEdit({elements})`，改完 `Undo.finishEdit()`。
3. **禁止共面**：同一平面上出现两个**朝向相同**且面积重叠的面 → z-fighting 闪烁。**朝向相反**的两面共面不会闪（背面剔除），**同朝向一定闪**。
4. **纹理源区域必须是"纯料"**：不要混有透明像素 / 别的物件颜色的区域里取大矩形 UV。
5. **动画命名**必须 `animation.<modid>.<action>`，与代码里 `RawAnimation` 的字符串**逐字符一致**。
6. **骨架层级**（父子关系、pivot）一旦被动画引用就不要改。

### 12.2 消接缝的正确做法

- **用镜像平铺**：源块镜像再拼（周期 2×块宽），交界处像素连续。
- **源块必须严格纯净**：先按像素分类打印字符图，肉眼确认要取的矩形**全在同一类**。判据宁严勿宽。
- **镜像平铺的块必须整除目标区**。
- **禁止用纯色块填充**替代有细节的区域：可以"平"，但不能"死平"。
- ⚠ **一个"几乎纯"的源块，其杂色列会被平铺周期放大成规律条纹** —— 这就是"梳齿状竖纹"的真正根因。
- **间隙 ≥ 0.02 模型单位**，0.001 这类在几十格外仍会闪；**外扩量不要用统一值**（统一外扩会让平行的同类面重新共面）。
- **改完必做坐标复查**（遍历部件比较同轴面坐标，确认无相等项）；截图不是硬证据。

### 12.3 多层染色物品：从原版贴图反推分层模板

适用：物品贴图要拆成"外圈 / 内圈 / 厚度"等多层、由代码分别染色。

- **手画几何永远对不上**：正确做法是把原版贴图**逐像素拆成互不重叠的层掩膜**，不重画任何一个像素。
- **要求"不重不漏"**：各层像素数相加 = 原版不透明像素总数；再用合成图与原版逐像素比对。
- **每层只保留原版该像素自身的亮度**，按该层**最亮像素**归一化到 255：`v = 255 * lum / peak`。
- **归一化基准的选择就是全部**：按均值 → 大片像素被压到 255、沟槽全平；按众数 → 高光被裁剪；**按峰值 + 默认 tint 取峰值灰度**才能 1:1 还原。
- **换归一化基准必须重算默认 tint**；验证靠逐像素打印对照表，差值 > 1 就是判据错了。

---

## §13 生物与动画

> 细节（注册链七项、渲染管线偏移推导、货物生命周期、per-bone 渲染层、Mixin 注入器选择）：`references/13-entities.md`

### 13.1 生物分类铁律：先定性，再动手

| 类别 | 正确做法 | 禁止 |
|---|---|---|
| **原版组合体**（原版 A 驮 / 吊原版 B，外观 100% 原版） | Mixin 注入原版实体行为；实体与货物都是原版实例 | 注册新 `EntityType`；写新的实体基类；配 loot table / attribute / 渲染器 |
| **原版换皮 / 改行为** | 同上：Mixin + 原版实体 | 为"换个 AI"复制一份原版实体类 |
| **全新生物** | 注册 `EntityType` + 属性 + 渲染器 + loot table，走完整注册链 | 用"占位实体 + 硬编码贴图"混进正式版 |

**判定口诀**：杀掉这只生物后，玩家看到的每一块掉落是否都能来自原版实体？能 → 原版组合体，**不许注册**。

### 13.2 注册链完整性（全新生物，缺一即未完成）

- [ ] `EntityType.Builder`（尺寸 / eyeHeight / passengerAttachments / clientTrackingRange / notInPeaceful 按原版同类镜像）
- [ ] 两端注册（Fabric `Registry.register` / NeoForge `DeferredRegister`），common 只认抽象接口
- [ ] `AttributeSupplier` 显式注册（自定义实体不在 vanilla `DefaultAttributes` 里，漏了首刷即崩）
- [ ] 客户端渲染器注册（两端各自入口）
- [ ] 刷怪蛋（26.x 数据驱动）、语言键、战利品表
- [ ] 生物名双语

原版组合体**恰好相反**：以上七项一律不许出现。

### 13.3 渲染与判定箱对齐（最高频翻车点）

- **三件套必须一致**：模型离地高度、`EntityType.sized()`、渲染时的 `translate/scale`。改任一个都要同步核对另两个，并**实机 F3+B 截图**验收。
- **改渲染偏移前，先把整条管线的位移算清楚**。现代版 `LivingEntityRenderer#submit` 的顺序是 `scale(state.scale)` → `setupRotations` → `scale(-1,-1,1)` → `this.scale(...)` → `translate(0, -1.501, 0)`。因为 `scale(-1,-1,1)` 把 y 取反，`this.scale` 里的 `translate(0, +L, 0)` 最终把模型抬高 `1.501 - L`。
- **【强制】任何"模型浮在判定箱上方"的结论，必须先按上式算出真实落点再下判断**；禁止看到源码里有个数字就认定它错。
- **禁止**为了对齐而手改 `geo.json` 的骨骼数据；偏移一律走渲染器 `translate` 或转换脚本参数。
- **挂载偏移必须写成"相对模型区间"的推导过程**，不许只给一个孤零零的数字。
- **改渲染偏移前先做定值验证**：先原样跑一遍、按 F3+B 与日志数值确认"当前到底偏多少"，再改。
- **纹理 / 模型文件是美术资产**：任何代码改动不得触碰 `*.bbmodel`、`*.geo.json`、`*.png`、`*.animation.json`。**验收时模型对不齐先怀疑代码，不怀疑资产。**

### 13.4 AI / Goal 规范

1. **目标选择与行为分离**：索敌 Goal 只写 target 永远 `return false`；行为 Goal 读 `getTarget()`。
2. **优先级**：抢占总控行为的 Goal 挂优先级 0，并正确声明 `Goal.Flag`（不声明就等于不抢占）。
3. **冷却计时放实体 tick 或 mixin tick，不放 Goal 字段**（Goal 会在失去目标时停止 tick，冷却会冻结）。
4. **第一次 tick 兜底**：`finalizeSpawn` 会被 `/summon` 等路径跳过，一切"生成时初始化"必须放首个服务端 tick 里做一次。
5. **几何推导值禁止进配置**：挂载高度、抬升、锚点偏移写成**相对模型区间的代码常量**，并把推导过程写在注释里（配置文件不会跟着模型变）。

### 13.5 GeckoLib

- 资源三件套固定路径：`geckolib/models/*.geo.json`、`geckolib/animations/*.animation.json`、纹理走 `textures/entity/`。
- `GeoModel` 子类把三个 id 固化为常量，**禁止每帧构造 `Identifier`**。
- 新动画由美术在 Blockbench 完成 → 转换脚本产出 → 代码只引用；**代码侧禁止临时改动画 json**。
- **分骨骼渲染层**：`CustomBoneTextureGeoLayer` 的 per-bone 钩子只画该骨骼自身的 cube。
- ⚠ **"替换"而非"叠加"陷阱**：该层的 `preRender` 内部会隐藏该骨骼在原贴图那一遍里的绘制 ⇒ 做"覆盖层"会让这些部位变成**半透明幽灵**。要**叠加**必须覆写 `preRender` 为空方法。判定方法：`javap -c -p <geckolib jar>` 看该层调用了什么，不要靠猜。
- **骨骼 / UV 判定必须来自数据**：解析 `.geo.json`（每个 cube 的 `size` / `origin` / 每面 `uv` + `uv_size`），禁止凭截图猜、禁止只看第一个 UV 数字、禁止相信骨骼名。

---

## §14 标准工作流

> 四条工作流的逐步清单：`references/14-workflows.md`

### 14.1 从零建新模组

1. **问版本号 + `releaseType`**（逐模组，§1.4）⇒ 填 `gradle.properties`（`mod_id` 合正则、`mod_name` 为展示名、`description` **单行**、作者与 license 按项目）。
2. **选模板分支**（按 MC 版本，§6.2），确认 Gradle 档位来自插件约束（§2.3）；**先查本地缓存再决定是否下载**。
3. **写元数据**：两端按 §3 模板；`icon.png` 128×128；字段放对层级；`[[mixins]]` 显式声明。
4. **资源**：`pack.mcmeta` 按加载器线决定；语言文件键与 `en_us` 对齐、UTF-8 无 BOM。
5. **构建**：`options.encoding = 'UTF-8'`；产物名按 §1.5。
6. **部署**：首次问目标实例（§8.3），之后自动；删实例内旧 jar、留 `build/libs` 旧 jar；MD5 比对。
7. **验收**：按 §8.4；本地化按 §4.5 实机切语言验证。
8. **发布**：§9（用户实机通过并主动下令后才上传 / 推送）。

### 14.2 扩一个 MC 版本分支

1. **探针先行**（§7.6）+ 兜底 API 引入版本。
2. **判据**：与相邻分支全部一致 → 并入并放宽 range；任一不同 → 另开分支（L1/L2/L3）。
3. **建目录**：每版本一个独立目录（目录名 = 分支名 = jar 区间标签），复制同族最接近的目录作骨架。
4. **落 6 处声明**（§7.7）。
5. **按需适配**：Mixin `remap` 与 `compatibilityLevel`、EventBus 6/7、客户端消息 API、权限 API。
6. **构建**：先 `gradlew projects` 再 `:模块:jar`；确认没有用 `options.release` 降字节码。
7. **部署 + 验收**：有实例先实机；无实例只能编译验证，**声明范围必须诚实标注未实机**。
8. **发布侧对齐**：CF `gameVersions` 按区间全列。

### 14.3 发一个版本

1. 逐模组问版本号 + `releaseType`。
2. 自检：§1、§3.6、§4.5、§6.6、§9.6。
3. 写 changelog（只版本小标题 + 条目）。
4. 用户实机通过并**主动下令**后：CF 走 `cfUpload`（显式代理 + legacy 端点 + `relations` + 只认 File ID）；MR 走 `mrUpload`（fabric / neoforge 分别一个 version）。
5. Git：确认用户指令 → buildlike = 0 → 基线检查 → 推送 → `ls-remote` 回读 → 回报分支 + 短哈希 + 提交信息。

### 14.4 上线后出问题

按 §10.1 Debug Flow → 对照症状索引 → 落到具体案例 → 修完**重新构建 + 重新部署 + 重新验收**，结论写进项目决策记录。

---

## §15 交付文档规范

> 完整规范（读者分析、禁写清单、命名与语言结构、交稿自检）：`references/15-docs.md`

### 15.1 只写"该读者需要的、关于本模组本身的事实"

- **玩家向**（README / 商店页 / 简介）：这是什么、怎么用、注意事项、相关链接、许可证。
- **开发者向**（API / 集成文档）：能力清单、公开接口、最小用法、契约（例如"公开 API 只增不改"）、使用中的坑。
- **判断标准**：逐句问"这句话对玩家 / 集成者有用吗？" —— 没用就不写。

### 15.2 绝不写进交付文档（AI 最常犯，出现即删）

1. **用户的规范要求 / 提示词信息本身**（"版本号不用写""只写本模组实际内容""按某规则维护"）。
2. **版本号与构建信息**（模组版本、构建命令与工具链、Loader / Java 版本）：会随每次发版过期。开发者向文档可以写**集成所需**的坐标与环境要求，但不写"本次版本号"。
3. **移植 / 回填备注**：跨版本 API 差异、"回填时注意……" —— 进**项目决策记录**。
4. **内部过程与决策**：Plan、踩坑过程、决策编号、日期流水、AI 协作规则。
5. **文档自身的元信息**："本文件由 AI 整理""后续会更新"。

### 15.3 命名与语言结构

- 拆分出的文档：**文件名用英文**、不编号、不排序。
- 对外文档与子文档一律**英文在前、中文在后**，不混排。
- README 保持短；长内容拆到子目录，并在 README 内用**相对链接**做好索引。

### 15.4 交稿自检

- 通读一遍，逐句问"读者是谁？这句对他有用吗？"
- 搜易犯词：`版本`、`1.x.x`、`移植`、`回填`、`本文件`、`AI`、`规范`、`提示词` —— 命中即复核是否应移到决策记录。

---

## 附 A：元数据地址规范（主页 / issue / 备用反馈）

同一个"反馈入口"在两种工程形态里落点不同，写错位置就会出现"按钮不显示""点进去是别人的项目"：

| 形态 | 主页 | issue 跟踪 | 备用反馈站 |
|---|---|---|---|
| 单模块 Forge 线 | `mods.toml` **`[[mods]]` 内**的 `displayURL` | `mods.toml` **文件顶层**的 `issueTrackerURL` | `gradle.properties` 的 `description` 尾部（**只放备用站**） |
| 多加载器线 | NeoForge：`neoforge.mods.toml` 的 `displayURL`；Fabric：`contact.homepage` | 同两文件的 `issueTrackerURL` / `contact.issues`（**同样在文件顶层**） | 同上（**只放备用站**），用 ` | ` 分隔、**不要换行** |

要点：

- **换行必须按目标语法选**：Forge 的 `mods.toml` 用**基本字符串** `description="${description}"`，所以 `gradle.properties` 里必须写 `\\n`——写 `\n` 会在生成物里注入**真实换行**，模组直接加载不了。构建后回读产物确认。
- **多加载器不要共用一句带换行的简介**：同一份 `${description}` 同时进 TOML 的多行字面量（真实换行合法）和 Fabric 的 JSON 字符串（真实换行非法）。想让两边都对，就**别换行**。
- **URL 必须指向该模组自己**：改之前先比对同一项目其它版本文件里已核对过的地址。
- **不要重复指向**：`displayURL` 只放主页、`issueTrackerURL` 只放 issue，别把同一地址塞进两个字段。
- **不要把 issues 地址写进 description**：Forge 1.20.1 的模组列表已把 `issueTrackerURL` 渲染成"问题跟踪"入口，description 里再写一遍会被**原样再显示一遍并带下划线**。改完回读产物确认。

## 附 B：说"commit"时的含义

用户说"commit"通常指**本地提交 + 推送到远端**两步，不是只 `git commit` —— 只做本地提交而没推送，等于没交。推送前后按推送纪律走：推前 `ls-remote` 比对远端哈希与上次记录，一致才推（快进或按约定强推），不一致先停下问用户；推后把「分支 + 哈希 + 提交信息」记进台账。
