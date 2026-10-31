# 11 易错结论对照

> 管什么：一份「旧说法 → 现行口径」的勘误表，加一份动手前必须复核的清单，再加解释"这些结论怎么来的"的元原则。
> 本文件与 `references/` 其余模块同属一个技能包，**没有上位总纲**；结论冲突时以一手证据为准。本文件只负责**纠正已被推翻的说法**、**标出还没核实的空白**。

**怎么用**：

1. 写代码 / 写文档 / 做判断前，先在本文件搜关键词——被推翻的说法最容易以"常识"的形式复活。
2. 每条对照都给**现行口径 + 依据类型**（`实测` / `推断` / `待核实`）。标 `推断` 的**不许据此放宽版本声明或发布**。
3. 看到 `已作废` 的行，**不要复述、不要"兼容性保留"**——它存在只是为了让人认出这个坑。

**证据强度标签**（全文通用）：

| 标签 | 含义 |
|---|---|
| **铁律** | 违反会导致构建失败 / 功能静默失效 / 已发布物返工 |
| **约定** | 推荐做法，可按项目调整 |
| **实测** | 有一手证据（`javap` / jar 解包 / maven 元数据 / 实机验收） |
| **推断** | 未实测，不许据此放宽版本声明或发布，必须先探针 |
| **已作废** | 曾采用、现已推翻，保留以防被复述成事实 |

---

## 11.1 元数据与资源

| 旧说法（已作废 / 已收窄） | 现行口径 | 依据 |
|---|---|---|
| 「`pack.mcmeta` 一律删掉」 | **按版本线拆分**：NeoForge 线**不带数据包内容就删**；Forge 线（1.18.x / 1.19.x / 1.20 – 1.20.1 等）**补最小 pack 元数据**（`pack_format` 8 / 9 / 15，不写 `data` 段）。位置也是铁律：**只放 forge 模块**，放 `common` 会让 NeoForge 端报"数据包错误" | 实测 |
| 「`pack.mcmeta` 只是个可选的说明文件」 | 它的存在与缺失在两条版本线上**表现正好相反**：NeoForge 多放一个就进不了世界；Forge 少放一个每次启动告警 `failed to load a valid ResourcePackInfo` / `Missing metadata in pack <modid>:...`（**模组其实加载成功**，只是弹"Mod 加载警告"） | 实测 |
| 「`pack_format` 取个大概的数就行」 | `pack_format` 必须取**目标版本原版 `client.jar` 里的 `pack_version.resource`**：1.18.x=8 / 1.19.x=9 / 1.20–1.20.1=15 / 1.20.2–1.20.4=22 / 1.20.5–1.20.6=32 / 1.21.2–1.21.5=55 / 1.21.6–1.21.8=64。填错不会有"格式错误"提示，而是资源加载行为异常 | 实测 |
| 「所有版本都验左侧小图标」 | 双槽位（`iconFile` / `bannerFile`）是 **26.2+** 的规则；更早版本按 `logoFile` 与加载器能力验收。26.2+ 把 `logoFile` 当废弃字段，1:1 方图会被塞进右侧横幅槽位 | 实测 |
| 「图标多大都行，看着清楚就好」 | 统一 **128×128 正方形**。超过 256×256 或非正方形会让强校验 UI（`Branding.ICON` 那一层）抛 `InvalidBrandingImageException`，**且不进日志** | 实测 |
| 「图标尺寸按文件属性看就行」 | 要看**真实像素**：读 PNG 的 IHDR（偏移 16 / 20 起各 4 字节大端 = 宽 / 高）。看图软件的缩略图可能已经缩放过，不可作为判据 | 实测 |
| 「NeoForge 资源扫描走类加载器目录枚举」 | 走 **FML 文件视图**：`ModList...getContents().visitContent(...)`。类加载器目录枚举在 jar 环境下天然失效（日志里"扫描了 N 个模组的 M 个命名空间，收集到 K 个资源文件"中 `M = 0` 或 `K = 0` 就是它的特征） | 实测，旧建议已作废 |
| 「资源没加载就是没打进 jar」 | 先解包确认 `assets/<modid>/...` 在不在；在包里但没生效，就是**扫描口径**问题，不是打包问题。这两者的修法完全不同 | 实测 |
| 「`displayURL` / `issueTrackerURL` 放文件顶层就行」 | 必须在 `[[mods]]` 子表内，否则按钮点不动 | 实测 |
| 「元数据文件名从 1.20.6 起才变」 | **未可靠裁决**：不同来源对 `mods.toml` / `neoforge.mods.toml` 的切换时点给出不同答案（1.20.6、更早、按加载器而异）。**列为待核实**（§11.7 第 3 项），不写死进模板 | 待核实 |
| 「`description` 里换行只是排版问题」 | `expand` 陷阱：经 `processResources.expand` 的 JSON / TOML 正文**禁写** `\n`、`\t`、`\u`；`gradle.properties` 的 `description` **必须单行**。写成多行会得到 `ParsingException: Invalid newline in basic string`，整个 jar 判为无效 mod 文件 | 实测 |
| 「`description` 写长一点没关系，多行更好读」 | 单行、先英后中。（Neo）Forge 的 `description` 建议含 Issue 链接，页面未建就用你控制的备用站；**禁止 404 按钮** | 约定 |
| 「动态注册表条目随便怎么塞都行」 | 26.x 用数据包条目时，**提供该条目的包必须声明 `KnownPack`**（`PackLocationInfo` 的第 4 个参数不能是 `Optional.empty()`），否则新建 / 进入世界每次都弹"实验性设置"。版本串用**固定字符串**，不要塞模组自身版本（`(namespace, id, version)` 是身份三元组，随版本变会让老存档记录的包身份失配） | 实测 |
| 「弹'实验性设置'是原版行为，改不了」 | 源头是模组自己写的那个 `Optional.empty()`，**能改**。用"代码 API 注册附魔"也绕不开——动态注册表只能在数据包加载阶段填充，"代码 API"最终仍是把 JSON 写进某个包 | 实测 |
| 「`modId` 只是个内部标识，改不改无所谓」 | `modId` 必须匹配 `^(?=.{2,64}$)[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`（禁连字符 / 大写 / 空格）；它同时驱动 fabric `id`、TOML `modId`、包名、资源目录、语言键前缀——**改一处必须全改** | 铁律 |

**这一组的共同模式**：元数据类结论几乎都是"按版本线分叉"而不是"一刀切"。凡是看到某条规则被写成"一律 / 全部 / 都"，先怀疑它丢了版本分界。

---

## 11.2 Mixin / API / 版本差异

| 旧说法（已作废 / 已收窄） | 现行口径 | 依据 |
|---|---|---|
| 「兼容级写 `JAVA_21` 更现代，统一用」 | 兼容级**必须匹配加载器自带的 Mixin**：Forge ≤ 1.20.6 = **0.8.5**（只认 `JAVA_17`，且自带 ASM 不支持 major 65）；Forge ≥ 1.21.1 = **0.8.7**（`JAVA_21` 合法）。写错的表现是 `MixinInitialisationError: ... specifies compatibility level JAVA_21 which is not recognised` | 实测 |
| 「同一分支的 jar 两端都能跑，所以兼容级没问题」 | 兼容级是**逐加载器**判定的：同一份 `common` 的 `mixins.json`，Fabric 自带的新版 sponge-mixin 认识 `JAVA_21`，老版 Forge 的 0.8.5 不认识。表现就是"Fabric / NeoForge 正常，Forge 启动即崩" | 实测 |
| 「用 `options.release=21` / `targetCompatibility` 降级字节码就行」 | **不行**：二者会把解析属性 `org.gradle.jvm.version` 拉到 17，与 `common` / 加载器发布的 Java 21 变体冲突（`only compatible with JVM runtime version 21 or newer`）。可行做法是保持原 JVM 编译，**编译后改写 `.class` 的 major 字节**（第 7 字节，65→61 / 69→65）并同步改写 jar 内的 `mixins.json` | 实测 |
| 「`mixins.json` 的 `refmap` 键可有可无」 | **按运行期映射决定**：Loom 风格（官方映射）不需要；**FG6 + SRG 运行期必需**，缺了会报 `Critical injection failure ... No refMap loaded`（jar 里明明有 refmap 文件）。`reobf=false` 的分支不需要 | 实测 |
| 「`remap` 一律写 `false`」 | `false` 是 **1.20.6+** 的规则；1.20.1（FG6）保持默认 `true`，且 Mixin 走 **jar manifest `MixinConfigs`**（`mods.toml` 里没有 `[[mixins]]` 段） | 实测 |
| 「Mixin 配置写一端就够了，加载器会自己找」 | 加载器**不会**自动发现 jar 里的 `*.mixins.json`。**配置必须两端成对声明**，漏一侧的表现是"该侧全部静默失效"、日志无报错。Fabric 写在 `fabric.mod.json` 的 `"mixins"` 数组（不含 `.json` 后缀）；Forge 1.20.1 走 jar manifest `MixinConfigs`；1.20.2+ / NeoForge 走 TOML 的 `[[mixins]]` 段 | 铁律 |
| 「注解处理器装上更保险」 | 在 **official 映射**环境（FG7）里，mixin 注解处理器会去找混淆映射，报 `Unable to locate obfuscation mapping for @Inject target`。去掉 `annotationProcessor`，改 `compileOnly 'org.spongepowered:mixin:0.8.7'` | 实测 |
| 「`@At` 目标串写错会报错」 | **不会**。目标串是字符串字面量，失配时**静默失效**——回调永不执行、日志一切正常。必须靠 `javap -p -c -s` 核对真实字节码，并加 `require = 1` 让它变成显式失败 | 实测 |
| 「照抄另一条版本线的方法名最省事」 | **禁止**。注入点必须对**目标版本的真实字节码**核定（`javap -p -s` / `-c`）；两端方法名不同时注解里同时列出 + `require = 1` | 铁律 |
| 「照抄另一端的 patched 源码最快」 | **禁止**。patched 源码来自另一端的反编译结果，方法名 / 描述符可能已经不同；照抄会得到"看起来对、注入不上"的注解 | 铁律 |
| 「权限判定用 1.21.2 起的新 API」 | **1.21.9** 起用 `permissions().hasPermission(...)`（不是 1.21.2、不是 26.1） | 实测 |
| 「26.x 里 `FMLLoader.getDist()` 照旧静态调用」 | 26.x **禁止静态调** `FMLLoader.getDist()` | 实测 |
| 「客户端发消息一律 `displayClientMessage`」 | **按版本选**：1.21.6 – 1.21.8 才是 `displayClientMessage(MutableComponent, boolean)`；26.x 的 `LocalPlayer` 换回 `sendSystemMessage(Component)`。同分支的 fabric / neoforge 端必须保持一致 | 实测 |
| 「事件注册一律 `register(Class)`」 | **EventBus 7**（Forge **1.21.8** 起为 7.0.x，1.21.11 / 26.x 为 7.0.5 / 7.0.6）**不再接受**用 `register(Class)` 注册"单监听方法"的类，报 `IllegalArgumentException: Only a single listener found in class ...`。改用事件类自带的 `BUS`：`ClientPlayerNetworkEvent.LoggingIn.BUS.addListener(X::onLogin);` 并去掉 `@SubscribeEvent`。判据是实例 JSON 里的 `eventbus` 版本（**6.x 仍可用 `register(Class)`**） | 实测 |
| 「看见 `sponge-mixin` / fabric 字样就判成 Fabric」 | **加载器判定用库坐标**：`net.neoforged.fancymodloader` ⇒ NeoForge；`net.fabricmc:fabric-loader` ⇒ Fabric；`net.minecraftforge:fmlloader` ⇒ Forge。NeoForge **也带** `sponge-mixin`，字样不是判据 | 铁律 |
| 「Fabric API 用 `implementation` 就行」 | 必须用 **`modImplementation`** 引 fabric-loader / fabric-api，否则 intermediary 名未重映射，编译期报 `cannot access class_XXXX` | 实测 |
| 「缺前置会在运行期给个友好提示」 | **不会**：加载器**直接拒绝启动**。所以 `depends` 声明的价值是"让失败发生在加载期"，而不是"给提示" | 实测 |

**这一组的共同模式**：Mixin / API 类结论的判据永远是**目标运行期的真实制品**（自带 Mixin 版本、映射方式、库坐标），不是"新版本写法更好"。

---

## 11.3 构建与工具链

| 旧说法（已作废 / 已收窄） | 现行口径 | 依据 |
|---|---|---|
| 「Gradle 固定三档，且全部已缓存」 | 档位由**插件约束**决定，不是拍数字；缓存状态**每次重列目录核实**（历史记录里同一档位一会儿"无缓存"、一会儿"在用"，就是没重列的后果）。正确顺序：① 列出本地已有发行版 → ② 由插件版本反推所需档位 → ③ 有则复用、无则**最多下载一次** | 实测 |
| 「为对齐版本数字，多下一份 Gradle 更保险」 | **禁止**。先查 `~/.gradle/wrapper/dists` 现有发行版，有就复用；**禁止为对齐数字多下一份**。缺则报告，不自行下载 | 铁律 |
| 「Gradle 档位一刀切，全工程统一最省心」 | **不许一刀切**：档位由**插件**决定。26.x 需要较新的档位（Loom 1.15+ / MDG 2.0.14x 把主工程逼到 Gradle 9 系）；1.21.2 – 1.21.11 与 Forge ≤ 1.21.8 各有自己的档位；Forge 全线（FG6）只能 Gradle 8.x。**三档都不可删** | 铁律 |
| 「Maven 仓库按官方源顺序写最标准」 | 默认**走可达的镜像且放官方源之前**。镜像放后面等于没放——官方源不通时会先卡满超时 | 约定 |
| 「`java_version=17` 说明 Gradle 也能跑在 17」 | **分开判断**：源码目标版本 ≠ Gradle JVM 版本。Loom 1.13 / MDG 2.0.123 这类插件要求 **Gradle JVM 21+**，与 `java_version` 无关 | 实测 |
| 「依赖系统 `JAVA_HOME` 就行，反正环境里配好了」 | 构建命令**必须显式设 `JAVA_HOME`**。多 JDK 并存时，环境变量随时可能指向错的那个，表现为"同样的工程昨天能构建今天不能" | 铁律 |
| 「FG6 能跑 Gradle 9」 | **不能**：`Found Gradle version Gradle 9.6.1. Versions Gradle 9.0 and newer are not supported yet`。1.21.9+ / 26.x 主工程被 Loom 1.15+ / MDG 2.0.14x 逼到 Gradle 9，此时换 **FG7**（官方 MDK：1.21.11 `[7.0.3,8)`、26.x `[7.0.17,8)`），或把 forge 端拆成独立 Gradle 8 工程 | 实测 |
| 「构建报错先去改 `mcp_config` 缓存」 | **先拉官方 MDK 看它用什么插件与 wrapper**。改缓存的下场：删源文件 → `错误: 无源文件`；改内容 → 管线取不到源码，一样空。`duplicate entry: mcp/client/Start.class` 这类问题用 FG7 直接绕开 | 实测 |
| 「Forge 版本号随便配一个 62.x 就行」 | Forge 版本线按 MC 补丁版本分：`26.1=62.x / 26.1.1=63.x / 26.1.2=64.x / 26.2=65.x / 26.3=66.x`。拿 62.x 配 26.1.2 会 404；loader 下界取**线起点**（如 `[62,)`） | 实测 |
| 「`expandProps` 里多塞几个键没关系」 | `group` 键会撞 Gradle 9 的任务属性，报 `property 'group' doesn't have a configured value`。删掉该键（元数据并未使用） | 实测 |
| 「`configurations.named('sourcesElements')` 是标准写法」 | 该变体不存在时直接报 `Configuration with name 'sourcesElements' not found`。改用 `configurations.matching { it.name == variant }.configureEach { ... }` | 实测 |
| 「`mainArtifact` 绑哪个任务都行，反正都是 jar」 | `mainArtifact` 只吃 `File` 或 `AbstractArchiveTask`（**不吃 `TaskProvider`**），绑错任务名在**配置期**就炸。而且"真实可分发产物"因工程而异：没有 `remapJar` 就绑 `jar`；有 `remapJar` 的老工程才绑它；`shadowJar` 不能跨项目硬套 | 实测 |
| 「Gradle 9 里 `tasks.xxx` 还能用」 | Gradle 9 **移除了** `tasks.xxx` 动态任务属性；`project {}` 闭包是 `DELEGATE_FIRST`，裸写任务名会被当成项目属性报 `No such property`。唯一两端通用写法是 `tasks.named('xxx').get()` | 实测 |
| 「产物名双版本号是模板问题」 | 根因是**有第二个任务也在拼 `archiveVersion`**：只对 `Jar` 清、漏了 `RemapJarTask` 就会得到 `...-1.2.0-1.2.0.jar`。统一在 `AbstractArchiveTask` 这一层处理 | 实测 |
| 「改了 `build-logic` 之后直接看产物就行」 | 改完 `build-logic` **必须重跑**，别信旧产物；改了 `gradle.properties` 但构建显示 `UP-TO-DATE` 时要 `--rerun-tasks`，否则 jar 里仍是旧值 | 实测 |
| 「关掉 sources / javadoc 就不会卡」 | 关掉之前，`withJavadocJar()` 的 doclint 严格模式会把"中文注释 + 缺 `@param` / `@return`"判成 error 并 fail，把 `:<loader>:jar`、`publishToMavenLocal` 一起拖垮。降级用 `options.addStringOption('Xdoclint:none', '-quiet')`；更彻底是不产出 javadoc jar | 实测 |
| 「`./gradlew` 报找不到主类就重装 Gradle」 | 先看 wrapper jar 是否缺失（`.gitignore` 里要保留 `!gradle/wrapper/gradle-wrapper.jar`）。应急：直接用已缓存的发行版二进制 `~/.gradle/wrapper/dists/gradle-<版本>-bin/<hash>/gradle-<版本>/bin/gradle` | 实测 |
| 「ASM 版本冲突没法控制」 | `1.21.8` 的 NeoForm 硬钉 `org.ow2.asm` **9.6**，而 sponge-mixin 0.17.x 拉 **9.8**。用 `resolutionStrategy.force` 统一回 9.6；判定靠 `gradlew :<模块>:dependencies --configuration runtimeClasspath` 看实际选中版本 | 实测 |
| 「源码里有中文注释很正常」 | `javac` 按**平台默认编码**（中文 Windows 上是 GBK）读 UTF-8 源码，报"编码 GBK 的不可映射字符"。所有 `JavaCompile` 任务统一 `options.encoding = 'UTF-8'`；**未设置前注释一律用英文** | 实测 |
| 「构建成功就说明产物是对的」 | 构建成功只说明任务跑完。产物对不对要**解包回读**：版本号、元数据字段、语言文件、主类是否都在。这是"部署三查"的第三查 | 铁律 |

**这一组的共同模式**：工具链类结论的判据是**插件与官方 MDK 的实际要求**，不是"我这儿能跑"。凡是"我这能跑"的结论，必须写成"在 `<插件版本>` + `<Gradle 档位>` 下能跑"。

---

## 11.4 发布与平台

| 旧说法（已作废 / 已收窄） | 现行口径 | 依据 |
|---|---|---|
| 「`version` 要带 MC 前缀」 | `version` 是**纯模组版本**；MC 版本 / 区间进 jar 文件名与依赖区间。产物名 `<mod_id>-<version>-<loader>-<MC 或区间>.jar` | 实测 |
| 「上传统一用 `shadowJar`」 | `shadowJar` 是**特定工程的示例**，不能跨项目硬套。先确定该工程真实可分发产物是 `jar` / `remapJar` / `shadowJar` 中的哪一个（没有 `remapJar` 就绑 `jar`） | 实测 |
| 「`cfUpload` 会自动先构建」 | **不会**：它不带 `dependsOn` 构建任务，命令里必须显式带上前置构建，否则可能传上旧 jar | 实测 |
| 「命令里的路径参数不用加引号」 | 路径含空格时**必须逐个参数加引号**。真实事故：`-I "…含空格的目录/…"` 被词分割后脚本根本没加载，而管道退出码掩盖了失败 | 铁律 |
| 「`-P` 的值直接写就行」 | 值里含**点号或冒号**时必须**整体加引号**：`-PcfModule=<mod-id>-26.3` 不加引号会被拆成 `-PcfModule=<mod-id>` 与 `.3`，Gradle 报 `Task '.3' not found in root project` | 实测 |
| 「CF 的 legacy 端点已废弃」 | **已作废的旧推断**：`minecraft.curseforge.com/api/*` 完全正常（`javap` 常量池里 `UPLOAD_URL = "%s/api/projects/%s/upload-file"`）。反而是"改用 `api.curseforge.com/v1/*`"这条路在部分网络下被 CloudFront 层拦（`403 / X-Cache: Error from cloudfront`、空正文），不要往那投时间 | 实测 |
| 「上传 403 是 key 错了」 | 403 的真相通常是**代理没生效**：CurseGradle 自建 HttpClient 不读 `http(s).proxyHost`、不读默认 `ProxySelector`，直连触发 Cloudflare 人机挑战（`cf-mitigated: challenge`、`<title>Just a moment...</title>`）。反向判据：`Invalid API token provided.` 说明线路是通的 | 实测 |
| 「用 SOCKS5 代理最省事」 | **不要**：Java 的 SOCKS 实现会本地解析 DNS，代理处于规则模式时按 IP 判直连 → `SSLHandshakeException: Remote host terminated the handshake`。用 **HTTP CONNECT**，且必须**显式**指定（`DefaultProxySelector` 会在类初始化时缓存快照，`https.proxyHost` 常常来不及生效） | 实测 |
| 「MR 必须用 `curl`，所以只能用 curl」 | 保留**协议要求**（`data` 部件必须是 `application/json`，`Invoke-RestMethod -Form` 发不出可解析的 `data`，报 `400 invalid_input`），但**优选已有 Gradle init 脚本**；凭据一律不由 AI 读取 | 实测 |
| 「MR 和 CF 一样，一个文件带多个加载器标签」 | MR **一个 version 只能一个加载器**：fabric / neoforge 必须分别发两条。前置依赖也不同——MR 的 `dependencies: []` **空数组合法**，CF 的 `relations` 空数组会被拒（`errorCode 1002`） | 实测 |
| 「`relations` 空数组只是没写而已」 | CF 的 `relations` 只能**完全不写**或**至少 1 项**；`{"projects":[]}` 报 `Array item count 0 is less than minimum count of 1. Path 'relations.projects'` | 实测 |
| 「jar 里声明了前置，平台就会自动显示」 | **CF 不解析 jar 元数据**。前置必须写进上传 metadata 的 `relations.projects`，jar 自身声明**照旧保留**（那是加载器的硬校验）——**两者并存，不是二选一** | 实测 |
| 「页面上搜不到前置项目，说明 jar 里不该声明它」 | 页面搜索的候选 / 索引行为会受项目新建时间、收录状态影响，**搜不到不代表不该声明**。不要去删 jar 里的依赖声明 | 实测 |
| 「`metadata` 加个 `filename=` 更规范」 | **绝对不能带** `filename=`：带了会被服务端当文件上传处理，报 `{"errorCode":1001,"errorMessage":"Missing field metadata."}`。必须是普通表单字段 `Content-Disposition: form-data; name="metadata"` | 实测 |
| 「上传偶发 500 要改代码」 | legacy 端点偶发 `{"errorCode":500,...}` 属**服务端瞬时错误**，同一请求重跑即成功。**先重跑一次再排查** | 实测 |
| 「`environment` 分组可以不填」 | 只选 MC 版本 + 加载器会报 `Error Code 1021: ... environment group`；至少要 `addGameVersion 'Client'` 或 `'Server'`（该分组仅这两个版本，id 9638 / 9639） | 实测 |
| 「MR 的 `environment` 用中文标签填」 | 请求体**只接受英文枚举**（`client_and_server` / `client_only` / `client_only_server_optional` / `singleplayer_only` / `server_only` / `server_only_client_optional` / `dedicated_server_only` / `client_or_server` / `client_or_server_prefers_both`），界面上的中文只是前端本地化 | 实测 |
| 「已上传版本的属性改不了」 | 能改属性、**改不了文件本体**：CF 用 `POST /api/projects/{id}/update-file`（multipart `metadata` + 头 `X-Api-Token`）；MR 用 `PATCH /v2/version/{id}`。换 jar 只能删版本重发，**删除是破坏性操作，必须先问用户** | 实测 |
| 「changelog 里写维护规则无所谓」 | 文件内容**原样公开**到 CF 页面。只允许版本小标题 + 本次真实改动；禁止 H1 模组标题、流程说明、AI 维护规则。开发期可留 `## Unreleased`，**上传前必须改成 `### <version> (日期)`** | 铁律 |
| 「changelog 要累积历史版本更完整」 | **不累积**：历史在平台页面与 git 里，下次发布整体替换为本版本内容 | 约定 |
| 「推送一律强制推送」 | 降级为**两种可选策略**（快进推送 / 强推覆盖），且强推只对版本分支、**禁止强推 `main` / `master`**、推前必须 `ls-remote` 比对哈希。详见 [09-git-upload.md](09-git-upload.md) §9.1.3–§9.1.5 | 铁律 |
| 「远端分支存在就直接强推」 | 只确认"分支存在"**不够**。必须拿远端哈希与**该分支上次推送后记录的哈希**比对：一致才可直接强推；**不一致先停手问用户**。曾经因为跳过这一步，把另一份工作副本的提交原地覆盖，且本地没有该对象、`git fetch origin <sha>` 也取不回来 | 铁律 |
| 「`--force` 和 `--force-with-lease` 一样」 | 优先 `--force-with-lease`：远端被别人更新过会**拒绝**，比裸 `--force` 安全。裸 `--force` 只在已比对过哈希、且远端确无待保留提交时使用 | 约定 |
| 「推送前不用管 `.gitignore`，模板自带」 | MultiLoader 模板生成的版本目录**通常没有** `.gitignore`，直接 `git add -A` 会把 `build/`、`.gradle/`、`run(s)/` 全提交上去。推送前先补 `.gitignore` 并验证 **buildlike = 0** | 铁律 |
| 「凭据文件读一下没关系，只是本地配置」 | **禁止**：`curseforge_api_key` / `modrinthApiKey` 在用户全局 `gradle.properties`——**不读、不搜、不打印、不落盘、不转交外部命令**，只允许 Gradle `findProperty` 消费 | 铁律 |
| 「上传前先把配置写好、顺手试一次也行」 | 配置可以提前写，但**只有用户实机测试通过并主动下达指令后才允许执行上传** | 铁律 |

**这一组的共同模式**：平台类结论必须区分"协议要求"与"某次实现方式"。协议要求（字段类型、空数组合法性、一个 version 一个加载器）是硬的；实现方式（用哪个脚本、用 curl 还是 HttpClient）是可换的。

---

## 11.5 移植与版本声明

| 旧说法（已作废 / 已收窄） | 现行口径 | 依据 |
|---|---|---|
| 「一个 jar 声明支持的版本越多越好」 | 范围声明**以证据为界**，上界与下界同样危险。声明区间必须是**实际验过的版本串全列**，不是"理论上兼容" | 铁律 |
| 「声明区间就是把首尾两个版本写上」 | 区间内的**每一串**都要单独列（`1.21.9-1.21.11` → `['1.21.9','1.21.10','1.21.11']`），且每一串都要能被平台解析；解析不到即中止，不盲传 | 实测 |
| 「平台收录了新版本字符串，先按猜的写法写进配置」 | `addGameVersion` 的字符串在执行时向平台 API **实时校验**；若新版 MC / 加载器字符串尚未收录，任务会明确报错，届时再改字符串，**不要预先发明写法** | 实测 |
| 「移植时顺手统一下命名 / 文案更整齐」 | **移植只做版本适配**：允许不同的只有加载器 API、MC 版本 API、注册表 / 渲染写法。改 key / 文案 / 类名属**漂移**，不算适配；写不出适配理由的差异要改回去 | 铁律 |
| 「lang 文案里改个字不算改动」 | 移植不得擅自改文本：lang 文案、`description`、README / 文档正文、游戏内附魔描述这类文字，凡与本次版本适配无关，一律照抄基准线；**改一个字也算越界**。确需改动，先说明「原文 / 改法 / 理由」并取得同意 | 铁律 |
| 「切分支前不用先提交，反正会重来」 | 切分支前**先提交**；`build` 目录**不会**自动清。这两条在"每版本独立目录"的形态下仍然有效（不因为换了分支模型就失效） | 实测 |
| 「单工作树 + clean 流程最省事」 | 现行推荐是**每版本独立目录**；单工作树的 clean / 归档步骤**不并入默认流程**。理由：独立目录下每个版本的 `build/libs` 天然隔离，不需要靠清理脚本来保证"产物是本次的" | 约定 |
| 「语言数量越多越好 / 少几种没关系」 | 语言数量**按项目决定**，但**已发布的覆盖不许降级**。通用业务模组的最低要求与项目实际覆盖是两件事，不要用"最低要求"去砍已发布的内容 | 约定 |
| 「多语言缺键就用固定英文兜底」 | 旧 API 退 **`Component.translatable`**，**不得用固定英文 literal** 兜底。用 literal 的表现是"切成中文仍英文"，且这些键永远不会被翻译（判定：`lang` 键集合与代码里 `translatable(...)` 的键求差集） | 实测，旧方案已作废 |
| 「玩家消息用 `literal` 拼字符串最快」 | **禁止**用 `literal` 顶替可翻译组件；颜色**不用 `§` 色码**（第三方 UI 不解析），由组件样式给 | 铁律 |
| 「`mod_name` 只是个显示用的字符串」 | `mod_name` 是**展示名**，同时驱动 fabric `name`、TOML `displayName`、CF `displayName`、jar `Specification-Title`。**CF 展示名必须与 jar 名同源** | 铁律 |
| 「版本号后缀随便写，`0.8.3fix` 也行」 | 后缀**必须带连字符**（`0.8.3-fix`）；**同一版本号绝不允许对应两份不同内容的 jar**；改号必须重新构建 + 重新部署 | 铁律 |
| 「计划文档里写的口径就是现状」 | **计划是方案，不是现状**。与项目唯一事实来源冲突时，以事实来源为准；计划里出现的身份、配置、验收口径**不得当作现行标准执行** | 铁律 |
| 「每条版本线各派一个人独立开发最快」 | 那是把一份代码拆成四份维护：键名前缀、按钮文案、类名都会分叉，之后每次改动都要改四处且必然漏一处。正确做法是**基准分支先行 → 移植可并行 → 逐支审查**，移植只做版本适配并交付「与基准的差异清单」 | 铁律 |
| 「发现某支已经写成独立设计了，慢慢改回来就行」 | 直接 `git reset --hard HEAD` 回到干净状态再按基准重做。半成品的"差不多像"比重做更贵 | 约定 |

**这一组的共同模式**：移植类结论的风险集中在"顺手改"和"想当然兼容"。凡是改动与本次版本适配无关，一律不动。

---

## 11.6 判定纪律（跨领域）

这几条不属于某个技术点，而是**判定方法本身**的勘误，违反任何一条都会得到错误结论。

| 旧说法（已作废 / 已收窄） | 现行口径 | 依据 |
|---|---|---|
| 「命令 exit 0 就是成功」 | **管道退出码是最后一个命令的**：`git push ... \| tail`、`gradlew ... \| grep \| head`、`Select-Object -First N`（最后者还会杀掉 Gradle）。一律带外回读：`ls-remote` 哈希、File ID、`BUILD SUCCESSFUL` | 铁律 |
| 「推送输出里有 `->` 就是推上去了」 | 以 `ls-remote --heads origin <分支>` 回读的远端哈希 == `git rev-parse HEAD` 为准。多分支场景要**逐条**验，别只验一条就报"全部成功" | 铁律 |
| 「日志最后一条异常就是根因」 | 崩溃后的 `CancellationException` / `IOException: Stream closed` 是清理噪音；**真正的第一现场在更早的日志里**（`Invalid modId found` / `not a valid mod file` / `ParsingException`） | 实测 |
| 「崩溃报告第一个模组就是罪魁」 | 崩溃报告的先导往往是第三方渲染模组，根因在它**之前**的日志里。判据是"第一条根因行"，不是"第一个出现的模组名" | 实测 |
| 「构建成功说明改动生效了」 | `build/libs` 里的 jar **不会**自动进实例。排查"改了还是崩"的第一步永远是比对 `build/libs/*.jar` 与 `mods/*.jar` 的时间 / 体积 | 铁律 |
| 「工具没输出就是工具坏了 / 参数错了」 | 先看**系统盘剩余空间**。Gradle 缓存、wrapper 发行版、工具链临时目录全在系统盘，写满时工具链会**以不报错的方式失败**（无输出、空壳结果、反复重跑都一样）；清理后同一条命令原样重跑即成功 | 实测 |
| 「库写了 fail-fast，所以是我的配置非法」 | 校验器自建的规则可能是**原版并不存在**的假设。判定手法：反汇编原版相关方法，确认那条约束是否真实。例：`Enchantment#areCompatible(Holder, Holder)` 对双方各取一次 `exclusiveSet`，说明有效互斥集合 = 所有引用到它的标签的**并集**，跨组重复合法 | 实测 |
| 「库报 `has failed to load correctly` 就是库坏了」 | 这类问题会**冒充成库自身的加载失败**，而真正的错误行在它**上面几行**，内容往往是下游模组提供的 ID。只看 `ModLoadingException` 摘要会误判 | 实测 |
| 「报错类名是我的包，所以是我的 bug」 | 先看**包名**：不是自己的包就怀疑装载（错版本 jar 装进实例、class file version 不匹配）。`65.0` = Java 21、`69.0` = Java 25 | 实测 |
| 「同名模组就是同一个项目」 | 不同作者可能发布**完全同名但完全不同**的模组。解析后必须打印**项目标题 + 产物文件名**再下载，只认标题与 slug，不认文件名 | 实测 |
| 「jar 声明支持十几个版本，装哪个实例都行」 | 跨版本单包（例如自报 `1.16.5 ~ 1.21.8`）装进不匹配的实例照样崩，属**正常现象**。判定：读 jar 元数据里的版本声明与实例版本比对，而不是看文件名 | 实测 |
| 「文档里写了就成立」 | **文档缺席 ≠ 事实不存在**；"文档写了"也不等于现在成立（快照会过期）。判据永远是一手证据 | 铁律 |
| 「某加载器在某版本不存在」 | **坐标要查全**：必须把相关 artifactId 全部反查（`neoforge` 与 `neoforged:forge` 这类同名不同坐标的教训），只查一个就断言"不存在"会错 | 实测 |
| 「按文件名 / 任务名生成结论」 | 不得根据文件名、任务标题、Issue 标题推断内容；必须先看真实变更（Diff、解包产物、字节码） | 铁律 |
| 「改一处先改再说，根因后面再找」 | 改一处先分析根因；不擅自改用户没要求的配置。没有根因的改动会带来"修好了但不知道为什么"的假象 | 铁律 |

---

## 11.7 待核实清单（动手前先查这些）

这些项**没有可靠结论**或**会随时间失效**，每次动手前重新核一遍；核完把结论与"什么时候、用什么命令核的"一起记进项目台账。

| # | 待核实项 | 为什么重要 | 怎么核 |
|---|---|---|---|
| 1 | 各 JDK 是否仍在原路径、是否真能编译（JRE 不能编译） | 所有构建与老版本启动的前提 | 逐个跑 `java -version` / `javac -version`；确认是 JDK 不是 JRE |
| 2 | Gradle 各档位**当前缓存实况** | 决定"复用"还是"允许下载一次" | 列 `~/.gradle/wrapper/dists` 目录，按实际目录名判定 |
| 3 | 元数据文件名（`mods.toml` / `neoforge.mods.toml`）的**逐版本边界** | 决定每条分支的元数据写法 | 解包对应加载器版本自己的 jar 看它读哪个名字；或拉官方 MDK 对照 |
| 4 | 测试实例清单现状（新增 / 删除 / 加载器类型） | 部署与验收是否可行 | 逐个读实例 JSON 的**库坐标**，不按目录名猜加载器 |
| 5 | 老版本分支的构建路线（是否需要真 JDK 8、备选路线是否可行） | 最老那几条分支能否开工 | 拉该版本官方 MDK 看要求的 Java 与 Gradle |
| 6 | 老版本分支的注入点 / 事件描述符 | 能否用 Mixin、还是只能走事件 | `javap -p -s` 对真实字节码采样 |
| 7 | 未采样区间的版本分组是否与官方分组一致 | range 写法与分支数量 | 查官方 version JSON / 加载器元数据 |
| 8 | 环境残骸是否仍在（半下载的包、重复 JDK、junction） | 工具链探测与启动器列表干净度 | 列目录 + 查软链接目标；**不要凭记忆删** |
| 9 | 各平台 API 与规则是否变化（CF legacy 端点、MR `environment` 枚举、图标限制、加载器内容 API、CurseGradle 版本） | 发布链路与元数据规则 | 拉 maven 元数据 + 实际调一次只读接口 |
| 10 | `neoforge.mods.toml` 在 1.20.x 的 groupId / loader range 具体值 | 一条分支覆盖两个 release 时的 range 写法 | 解包该版本加载器制品，读它声明的坐标与范围 |
| 11 | 未实机复验的分支（有产物但没在真机跑过） | 决定发布是否维持"区间全列"的声明口径 | 逐分支实机启动 + 功能验收，缺证据就不声明该区间 |

**纪律**：清单里的每一项，核完都要写"**结论 + 时间 + 命令**"。只写结论不写后两项，下次还得重来。

**"待核实"不是"可以猜"**：标为待核实的项，处理方式只有两种——① 花时间核实；② 明确按保守口径处理（不声明、不发布、不据此改配置）。**没有第三种"按经验填一个"**。

### 11.7.1 开工前的复核动作（按任务类型取用）

不需要每轮把 §11.7 全核一遍，按任务类型挑相关项：

| 本轮任务 | 必须复核的项 | 可以跳过 |
|---|---|---|
| **建新模组 / 新建分支** | 1（JDK 可用）、2（Gradle 缓存）、3（元数据文件名边界）、7（版本分组） | 6、10（还没有注入点与 range 需求） |
| **扩版本 / 移植** | 3、4（目标实例）、6（注入点描述符）、7（range 分组） | 1、2（沿用基准线的环境） |
| **发版本 / 上传** | 9（平台 API 与规则）、11（未复验分支）、4（部署目标） | 5、8 |
| **线上排障** | 4（实例加载器）、1（运行时 JDK）、8（环境残骸） | 3、7、10 |
| **改元数据 / 图标 / 资源** | 3、9（平台字段与图标限制） | 5、6 |
| **改 Mixin / 事件** | 6（描述符）、3（兼容级随版本线） | 2、7 |

**复核的产出不是"知道了"，而是"写下来了"**：结论 + 时间 + 命令三项缺一不可，否则下一个人（或下一次会话）还得重核一遍。

---

## 11.8 元原则（解释"这些结论为什么成立"）

1. **一手优先**：maven 元数据、官方 version JSON、`javap`、解包 jar、实例 JSON > 文档 > 二手转述。任何二手结论都可以被一手证据推翻，反之不行。
2. **文档缺席 ≠ 事实不存在；"文档写了" ≠ 现在成立**。快照会过期，引用快照必须带日期与命令。
3. **坐标要查全**：判断"某制品在某版本不存在"，必须把相关 artifactId **全部**反查。漏一个坐标就会得到"不存在"的错误结论。
4. **退出码要带外验证**；工具的"成功"字样不是证据。能带外回读的（哈希、File ID、`BUILD SUCCESSFUL`）一律带外。
5. **范围声明以证据为界**：上界与下界同样危险。没验过的版本不写进声明，验过的版本不遗漏。
6. **口径要标来源**：本地路径、缓存、实例、平台字段都会变。任何结论都应能回答"什么时候测的、用什么命令测的、在哪个版本上测的"。
7. **静默失效比报错危险**：会叫的失败会自己找上门，不叫的（`@At` 目标串、`Optional.empty()`、图标尺寸、漏声明的 mixin 配置）只能靠主动核对发现。核对手段是**计数、描述符、键集合**这三类可量化检查。
8. **分层结论**：同一条规则在 A 版本线成立、在 B 版本线不成立时，写成两条带版本边界的结论，**不要合并成一条"一般情况"**。
9. **推翻要留痕**：结论被推翻时，把旧说法标 `已作废` 而不是删掉——否则它会以"常识"的形式在下一次对话里复活。
10. **区分"协议要求"与"实现方式"**：平台字段类型、空数组是否合法、一个 version 几个加载器属协议；用哪个脚本、用 curl 还是 HttpClient 属实现。协议不能改，实现可以换——把两者混为一谈会导致"因为某个示例这么写，所以必须这么写"的假铁律。
11. **区分"插件要求"与"我能跑"**：任何工具链结论都要带上插件版本与档位；"我这能跑"不是证据，只是待核实项。
12. **改动要能说清四要素**：现象 → 根因 → 判定手法 → 修复。缺"判定手法"的结论无法复用，缺"根因"的修复会复发。

### 11.8.1 旧说法复活的四条常见路径

勘误表存在的意义是防复活。以下是历史上最常见的四种复活方式，看到就当场拦下：

| 复活路径 | 表现 | 拦截方式 |
|---|---|---|
| **从旧文档复制** | 新文档里出现一段与勘误表 `已作废` 行措辞几乎一致的话 | 写文档前搜勘误表关键词；引用旧文档必须过一遍本文件 |
| **从旧代码继承** | 老分支里的写法被"顺手照抄"到新分支（兼容级、`remap`、`logoFile`、`pack.mcmeta`） | 移植前先列出该分支的版本边界；照抄的每一行都要能说出"为什么这条线也成立" |
| **从示例硬套** | 某个工程的示例（`shadowJar`、`curl`、单工作树 clean）被当成通用规则 | 先问"这是协议要求还是某次实现方式"（§11.8 第 10 条） |
| **从印象推断** | "新版本应该更现代，所以用新写法" | 判据换成一手制品：自带 Mixin 版本、映射方式、库坐标 |

---

## 11.9 自检清单

- [ ] 写下的每条结论都标了证据强度（`实测` / `推断` / `待核实` / `铁律` / `约定`）。
- [ ] 没有把 `推断` 当成依据去放宽版本声明或执行发布。
- [ ] 涉及版本差异的结论都带了**版本边界**，没有写成"一律"。
- [ ] 复述过的旧说法在本文件里查过，确认它不是 `已作废` 行。
- [ ] 区分过"协议要求"与"实现方式"，没有把某次示例写成通用规则。
- [ ] 工具链结论都带了插件版本与档位，不是"我这能跑"。
- [ ] §11.7 待核实清单里与本轮任务相关的项都已重核，并记下"结论 + 时间 + 命令"。
- [ ] 判定成功用的是带外回读，不是退出码或日志里的"成功"字样。
- [ ] 结论能回答"什么时候测的、用什么命令测的、在哪个版本上测的"。
- [ ] 结论写清了「现象 → 根因 → 判定手法 → 修复」四要素。
- [ ] 新发现的勘误已追加到对应小节，旧说法标 `已作废` 而不是删除。
- [ ] 本轮任务对应的 §11.7 复核项已按 §11.7.1 挑好，没核的项已明确按保守口径处理。
