# 06 构建与依赖管理

> 本文件管 Gradle 档位的选择依据、FG6/FG7 的 DSL 差异、MultiLoader 模板落地、依赖声明方式与编译编码配置。
> 上游：`SKILL.md`（结论以它为准）；本文件给完整依据、可复制片段与踩坑结论。
> 相邻模块：[02-environment.md](02-environment.md)（Java 分档、镜像、缓存检测）、[05-mixin-api.md](05-mixin-api.md)（Mixin 与注解处理器）、[07-porting.md](07-porting.md)（分支与版本区间）、[10-troubleshooting.md](10-troubleshooting.md)（构建故障的完整案例）。

标签约定：**铁律** = 违反会导致构建失败、功能静默失效或已发布物返工；**约定** = 推荐做法，可按项目调整；**实测** = 有一手证据；**推断** = 未实测，不许据此放宽版本声明。

---

## 6.1 为什么 Gradle 档位不能一刀切

Gradle 大版本会移除老插件依赖的 API，因此**不存在"一个 Gradle 版本通吃 1.18.2 → 26.3"**。

| 插件 | 版本约束 | 结论 |
|---|---|---|
| **Fabric Loom** | 官方发布说明：`1.14` 起要求 **Gradle 9.2**，`1.16` 起要求 **Gradle 9.4**；`1.8 / 1.9-SNAPSHOT` 只在 **Gradle 8.11** 可用 | Loom 版本与 Gradle 档位强绑定；Gradle 8.9+ 移除了 `Problems.forNamespace`，Loom ≤1.7 会报 `LoomProblemReporter` 实例化失败 |
| **ForgeGradle 6（FG6）** | `6.0` 分支**自身用 Gradle 8.1.1 构建、编译目标 Java 8、直接依赖 `gradleApi()`** | 绑定 Gradle 8.x API；FG6 已进入维护末期、**无 Gradle 9 适配迹象** ⇒ Forge ≤1.21.8 全线必须留在 8.x |
| **ForgeGradle 7（FG7）** | `[7.0.3,8)` / `[7.0.17,8)`，官方 MDK 的 wrapper 是 Gradle 9.5 / 9.7 | 1.21.11 起与 26.x 的官方解法，见 §6.2 |
| **ModDevGradle（MDG）** | 26.x 线用 `2.0.14x`；1.21.8 / 1.21.11 线用 `2.0.123`；三加载器分支用 `2.0.49-beta` | 随线走，不与 Loom 同档 |

**判定顺序（动手前必做）**：

1. 列出目标版本线要用的**全部插件**（Loom / MDG / FG / MixinExtras / ASM）。
2. 查每个插件的**最低 Gradle 要求**与**最高 Gradle 兼容性**。
3. 取交集；交集为空 ⇒ 该版本线必须拆工程（见 §6.2.4），**不是"降一个 Gradle 版本试试"**。
4. 逐个版本目录实测（`gradlew projects` 配置期 + `:模块:jar` 构建期）。

**禁止把"一套版本号"复制到所有版本目录。** 档位的具体清单与缓存核实方法见 [02-environment.md](02-environment.md)。

**想合并档位**：唯一路线是把旧线换到仍在维护的插件（Architectury Loom / NeoLoom 等，`org.relativitymc.neo-loom` 有使用先例），属**构建链改造**，必须逐版本实测；未实测前不许动。

---

## 6.2 Forge 侧的插件换代：FG6 → FG7（实测，来自官方 MDK）

### 6.2.1 版本约束对照

**结论（一手，来自 `forge-<mc>-<forge>-mdk.zip` 官方模板）**：

| MC 线 | Forge 插件 | 官方 MDK 的 wrapper | 备注 |
|---|---|---|---|
| ≤ 1.21.8 | `net.minecraftforge.gradle` `[6.0.24,6.2)`（**FG6**） | Gradle 8.x | **FG6 硬性拒绝 Gradle 9**：`Versions Gradle 9.0 and newer are not supported yet` |
| 1.21.11 | `net.minecraftforge.gradle` **`[7.0.3,8)`**（**FG7**） | **Gradle 9.5.0** | MDK 里 `mappings channel:'official'` + Java 21 |
| 26.1 / 26.2 / 26.3 | `net.minecraftforge.gradle` **`[7.0.17,8)`** | **Gradle 9.5.0 / 9.7.1** | MDK 里无显式 mappings 行；Java 25 |

⇒ **"26.x 分支的 Loom/MDG 必须 Gradle 9.2+，而 FG6 只吃 Gradle 8.x"这个死结，官方解法就是 FG7。** 不要再试图把 Gradle 降到 8.x，也不要去改 FG6 的缓存或补丁。

判据：直接下官方 MDK（`forge-<mc>-<forge>-mdk.zip`），读它的 `gradle/wrapper/gradle-wrapper.properties` 与 `build.gradle`。**官方模板就是该线的事实标准**，比任何第三方教程可靠。

### 6.2.2 FG7 的 DSL 与 FG6 差异

照抄官方 MDK 的最小骨架：

```groovy
plugins { id 'net.minecraftforge.gradle' version '[7.0.17,8)' }   // 1.21.11 用 [7.0.3,8)

minecraft {
    mappings channel: 'official', version: '<mc>'   // 1.21.11 MDK 有；26.x MDK 省略
}

repositories {
    minecraft.mavenizer(it)
    maven fg.forgeMaven
    maven fg.minecraftLibsMaven
    mavenCentral()
}

dependencies {
    implementation minecraft.dependency("net.minecraftforge:forge:${mc}-${forgeVersion}")
}
```

| 项 | FG6 | FG7 |
|---|---|---|
| 插件版本 | `[6.0.24,6.2)` | `[7.0.3,8)` / `[7.0.17,8)` |
| Gradle | 8.x | 9.5+ |
| 依赖坐标写法 | `implementation "net.minecraftforge:forge:...:universal"` 等 | `implementation minecraft.dependency("net.minecraftforge:forge:${mc}-${forgeVersion}")` |
| 仓库声明 | `maven { url = 'https://maven.minecraftforge.net/' }` | `minecraft.mavenizer(it)` + `fg.forgeMaven` + `fg.minecraftLibsMaven` |
| `mappings` | 通常显式 `channel: 'official'` | 1.21.11 显式；26.x 省略 |
| 对 Gradle 9 | 直接拒绝 | 必须 |

### 6.2.3 FG7 三个坑（都实测踩过）

1. **不要挂 mixin 注解处理器**：`annotationProcessor 'org.spongepowered:mixin:...:processor'` 在 FG7 + official 映射下会去找混淆映射，直接报
   ```
   Unable to locate obfuscation mapping for @Inject target ...
   ```
   官方映射运行期也不需要 refmap；注解类用 `compileOnly 'org.spongepowered:mixin:0.8.7'` 提供即可。
2. **`processResources` 的 `expandProps` 里不能有 `group` 键**：Gradle 9 会把它当成任务自身的 `group` 属性并报
   ```
   property 'group' doesn't have a configured value
   ```
   `mods.toml` 也没用到 `${group}`，直接删。
3. **降字节码不能用 `options.release`**：它会把解析属性 `org.gradle.jvm.version` 拉到该值，与 Forge 构件声明的 JVM（26.3 = 25）冲突：
   ```
   only compatible with JVM runtime version 25
   ```
   要降就**编译后改写 class 的 major 字节**（见 [10-troubleshooting.md](10-troubleshooting.md) §10.25）。

### 6.2.4 工程组织：独立工程 vs 子模块

当主工程（Loom 1.13–1.17 + MDG）与 Forge 端所需 Gradle 大版本不一致时，两种形态都可用，**由 `forge/settings.gradle` 在不在决定**：

| 形态 | 判据 | 构建命令（在分支根目录执行） | 适用条件 |
|---|---|---|---|
| **子模块** | 根 `settings.gradle` 里 `include('forge')`；`forge/` **没有**自己的 wrapper | `./gradlew :forge:jar` | Forge 端与根工程共用同一个 Gradle 档位（典型：FG6 时代） |
| **独立工程** | `forge/` 自带 `settings.gradle` + `gradlew`；根工程里写成 `// include('forge')` | `cd forge` → `./gradlew jar` | Forge 端所需 Gradle 档位与主工程冲突（典型：FG7 + Gradle 9 vs Loom 8.x） |

**判据（动手前 1 秒就能判）**：看 `forge/settings.gradle` **在不在**——

- **在** ⇒ 独立工程：用 **forge 目录自己的** wrapper，任务是 `jar`；
- **不在** ⇒ 子模块：用**分支根**的 wrapper，任务是 `:forge:jar`。

写错的表现是：

```
Task 'jar' not found in root project ...
```

（或反之）。**别误判成插件 / 版本问题**，更别因此去改 Gradle 档位。

**独立工程的做法（已验证可行）**：

- `forge/` 自带 `settings.gradle`、`build.gradle`、`gradlew` 与自己的 wrapper；
- 直接 `srcDirs = ['src/main/java', '../common/src/main/java']` 编译 common 源码；
- 主工程 `settings.gradle` 里**注释掉** `include('forge')`，避免两套插件互相污染；
- 独立工程里 `../gradle.properties` 仍是版本 / 元数据的唯一来源（用 `Properties` 读入后展开 `mods.toml`）；
- 好处：绕开 `:common` 的 `jvm.version` 属性冲突，且 Fabric/NeoForge 与 Forge 各自用最合适的 Gradle 版本。

> 同一项目里并存两种形态是**有意为之**：FG6 能吃 8.x，就并进根工程；1.21.9+ / 26.x 被 Loom/MDG 逼到 Gradle 9，与 FG6 冲突，才拆成独立工程。**同一条 MC 线上，Forge 与 Fabric/NeoForge 可以各用各的 Gradle 档**，不必为"统一"去降级或硬并。

---

## 6.3 MultiLoader 模板

| 项目 | 地址 |
|---|---|
| MultiLoader 多加载器模板（上游） | `https://github.com/jaredlll08/MultiLoader-Template` |

**使用要点**：

- **按 MC 版本选分支**：MultiLoader-Template 按 Minecraft 版本分分支（`1.21.8`、`1.21.1`、`1.20.1` 等），需拉取对应版本分支，**不要用 main 分支硬改**。
- 模板默认用 `officialMojangMappings()`（部分老分支）；低版本移植时**删掉 parchment**，避免 parchment 版本不存在导致下载失败。
- 新版本移植需自行创建对应版本文件夹（模板不会自动跟随新 MC 线）。
- **新版本模板落地的最小改动面**：以相邻版本目录为骨架复制，通常只需改 4 个文件——`gradle.properties`、根 `build.gradle`、`gradle/wrapper/gradle-wrapper.properties`、`common/build.gradle`。版本矩阵与元数据变更见 [03-metadata.md](03-metadata.md) §3.1。
- **`.gitignore` 别假设模板已带**（见 [09-git-upload.md](09-git-upload.md) §9.2）。
- 模板自身结构演进：`buildSrc` → `build-logic` 发生在 26.x；老分支照抄新结构会找不到插件。

**构建命令（PowerShell，模板目录与 JDK 路径按实际环境替换）**：

```powershell
$env:JAVA_HOME = "<JDK 25 安装目录>"
Set-Location "<版本目录>"
.\gradlew.bat :fabric:jar :neoforge:jar --console=plain --no-daemon
```

- `--console=plain` 保证日志可 grep、无 ANSI 干扰。
- `--no-daemon` 避免残留守护进程占住旧 JVM。
- **判定成功只看 `BUILD SUCCESSFUL`**，不看管道退出码（见 §6.6 第 5 条）。
- JAVA_HOME 必须显式设置，不要依赖系统变量；档位对照见 [02-environment.md](02-environment.md)。

---

## 6.4 依赖声明方式（含真实事故）

### 6.4.1 Fabric API 是前置而非内嵌

`fabric/build.gradle` 用 `implementation(...fabric-api...)`，Loom **不会**把它打进 jar（只有 `include` 才内嵌）。因此产物需要玩家自装 Fabric API——**必须在 `fabric.mod.json` 的 `depends` 里显式声明**（只要用到 API 类就是真前置），否则缺失时报的是运行期晦涩错误。

- 现象：产物一直需要玩家自装 Fabric API，但 `fabric.mod.json` 缺 `depends` 声明。
- 根因：`implementation` 不内嵌，而元数据没声明。
- 修复：补 `depends`。**补声明没有改变运行时行为**，只是让加载器提前给出明确提示。
- ⚠ **机制写错的技术描述本身就是虚假承诺**：曾把变更说明写成"补上 Fabric API 前置声明，避免未装 Fabric API 时进服提示钩子加载失败"——**前置声明不给友好提示，是加载器直接拒绝启动**。正确表述是"由加载器直接给出明确提示，而不是留到运行期报晦涩错误"。

### 6.4.2 必须用 `modImplementation`（铁律）

Fabric 线必须用 `modImplementation` 引 `fabric-loader` / `fabric-api`，否则报"无法访问 `class_XXXX`"——intermediary 名未重映射，`implementation` 拿到的类名是混淆名。

```groovy
dependencies {
    modImplementation "net.fabricmc:fabric-loader:${loader_version}"
    modImplementation "net.fabricmc.fabric-api:fabric-api:${fabric_api_version}"
}
```

### 6.4.3 加载器模块二次编译 common 源码时 classpath 缺 Mixin

fabric 模块要**显式 `compileOnly` Mixin**——该 MC 线的 fabric-loader 不再传递它。

```groovy
dependencies {
    compileOnly "org.spongepowered:mixin:0.8.5"
}
```

症状：common 里的 mixin 类在 fabric 模块二次编译时报 `cannot find symbol: @Inject`。

### 6.4.4 Loom 插件 id 与 mappings 的版本分界

- **Loom 1.14 之前只发布旧插件 id `fabric-loom`**，`settings.gradle` 的 `exclusiveContent` 必须显式放行 `fabric-loom` 组；**1.15+** 才是 `net.fabricmc.fabric-loom`。写错直接：
  ```
  Plugin ... was not found
  ```
- **Loom <1.14 不给 `mappings` 默认值** ⇒ 必须显式 `mappings loom.officialMojangMappings()`；Loom 1.14+ 才可省略。

```groovy
// settings.gradle（Loom < 1.14）
pluginManagement {
    repositories {
        exclusiveContent {
            forRepository { maven { url = 'https://maven.fabricmc.net/' } }
            filter { includeGroup 'fabric-loom' }   // 1.15+ 改成 'net.fabricmc.fabric-loom'
        }
    }
}
```

### 6.4.5 ASM 版本冲突

1.21.8 的 NeoForm 把 `org.ow2.asm` 硬钉 **9.6**，而 `sponge-mixin 0.17.x` 拉 **9.8** ⇒ 用 `resolutionStrategy.force` 统一回 9.6：

```groovy
configurations.all {
    resolutionStrategy.force 'org.ow2.asm:asm:9.6',
                               'org.ow2.asm:asm-tree:9.6',
                               'org.ow2.asm:asm-commons:9.6',
                               'org.ow2.asm:asm-util:9.6'
}
```

- 症状：启动报 `NoSuchMethodError` / `IllegalArgumentException: Unsupported class file major version`。
- 根因：同一 JVM 上出现两份 ASM，先加载的那份赢了。
- 判定手法：`gradlew dependencies --configuration runtimeClasspath | grep asm`，看是否有两条不同版本。
- 这类冲突与 Mixin `compatibilityLevel` 的报错**症状相似但根因不同**，别混：前者是库版本冲突，后者是注解配置写高（见 [05-mixin-api.md](05-mixin-api.md) §5.1.5）。

---

## 6.5 编译与编码配置

```groovy
tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
}
tasks.withType(Javadoc).configureEach {
    options.encoding = 'UTF-8'
    options.addStringOption('Xdoclint:none', '-quiet')
}
```

- 不设 `options.encoding` 时，Windows 默认 GBK 会把中文注释读坏，报"不可映射字符"。
- **`withJavadocJar()` 能在 Java 25 下卡死整个构建**：默认开 doclint 严格模式，中文注释 + record / Mixin / 枚举常量没逐项补 `@param` / `@return` 就直接报 error 并 fail，把 `:neoforge:jar`、`publishToMavenLocal` 一起拖垮。
  修法就是上面的 `-Xdoclint:none`（只是把 error 降级为 warning，**注释质量本身没变好**）。
  **不需要 javadoc 时，直接移除 `withJavadocJar()` 比加参数更省事。**
- **Gradle 必须跑在 JVM 21+**（Loom 1.13 / MDG 2.0.123 的字节码是 Java 21）；`java_version=17` 只是**工具链目标**，不是 Gradle 自己的 JVM。这两件事经常被混为一谈，症状是 `unsupported class file major version`。
- **三加载器分支不能把 Gradle 抬到 8.14.x 档**：FG6 绑定 Gradle 8.x API ⇒ 必须回到 `fabric-loom 1.8 / 1.9-SNAPSHOT + MDG 2.0.49-beta + FG [6.0.24,6.2) + Gradle 8.11` 这一组合，并逐版本实测。
- **`options.release` 慎用**：它同时影响解析属性 `org.gradle.jvm.version`（见 §6.2.3 第 3 条）。要精确控制字节码版本，用 `sourceCompatibility` / `targetCompatibility` 或构建后改写 major。

---

## 6.6 构建期常见故障（速查）

完整案例见 [10-troubleshooting.md](10-troubleshooting.md)。

1. **`./gradlew` 报"找不到主类 GradleWrapperMain"**：直接用已缓存的发行版二进制 `~/.gradle/wrapper/dists/gradle-<版本>-bin/<hash>/gradle-<版本>/bin/gradle`。缓存检测方法见 [02-environment.md](02-environment.md)。
2. **改了 `gradle.properties` 后构建仍 `UP-TO-DATE`**：jar 里还是旧值（例如旧 modId）⇒ 加 `--rerun-tasks` 强制重跑 `processResources`。
3. **关掉 sources / javadoc 后配置名报错**：改为
   ```groovy
   configurations.matching { it.name == variant }.configureEach { ... }
   ```
4. **构建成功 ≠ 游戏在跑新代码**：`build/libs` 里的 jar **不会**自动进实例，必须拷贝到 `<版本>/mods/` 并删掉旧同名 jar。排查"改了还是崩"时，**第一步永远是比对 `build/libs/*.jar` 与 `mods/*.jar` 的时间 / 体积**。
5. **命令里路径含空格**：`-I "<路径>"` 若写成未加引号的变量，会被 shell 词分割，init 脚本根本没加载，而 `| grep | head` 的退出码会掩盖失败。**管道退出码不可信**——`| tail`、`| grep | head`、`Select-Object -First N`（最后者还会杀掉 Gradle）一律改为抓完整输出按 `BUILD SUCCESSFUL` 判定。
6. **JAVA_HOME 选错档**：报 `unsupported class file major version` 时先核对 [02-environment.md](02-environment.md) 的 JAVA_HOME 对照表。
7. **`Task 'jar' not found in root project`**：工程形态判错（见 §6.2.4），不是插件问题。
8. **`Plugin ... was not found`**：Loom 插件 id 写错（见 §6.4.4）。
9. **`property 'group' doesn't have a configured value`**：`expandProps` 里混进了 `group` 键（见 §6.2.3 第 2 条）。
10. **`Unable to locate obfuscation mapping`**：挂了 mixin 注解处理器（见 §6.2.3 第 1 条）。
11. **`only compatible with JVM runtime version 25`**：用了 `options.release` 降字节码（见 §6.2.3 第 3 条）。

---

## 6.7 自检清单

- [ ] `gradle-wrapper.properties` 的档位与插件约束一致（逐个插件核对最低 / 最高 Gradle），且 wrapper 注释写了四要素：档位与约束来源 / 镜像 / 是否复用缓存 / 上次实测日期与命令。
- [ ] 已列目录核实在用的 Gradle 缓存档，不凭记忆；**没有为对齐数字多下一份**。
- [ ] 配置期 `gradlew projects` 与构建期 `:模块:jar` 都真跑过（两个阶段都会暴露不同问题）。
- [ ] Forge 端插件档位正确：≤1.21.8 用 FG6，1.21.11 用 `[7.0.3,8)`，26.x 用 `[7.0.17,8)`。
- [ ] `forge/settings.gradle` 在不在已确认，构建命令与任务名（`jar` / `:forge:jar`）对应。
- [ ] FG7 没有挂 mixin 注解处理器；`expandProps` 无 `group` 键；没用 `options.release` 降字节码。
- [ ] `JavaCompile` / `Javadoc` 的 encoding 已设；不需要 javadoc 时确认已移除 `withJavadocJar()`。
- [ ] Fabric 依赖用 `modImplementation`；用到 API 类时 `depends` 已声明 `fabric-api`。
- [ ] Loom 插件 id 与 `exclusiveContent` 的组名匹配（`fabric-loom` vs `net.fabricmc.fabric-loom`）。
- [ ] ASM 版本无冲突（`gradlew dependencies` 里同一构件只有一个版本）。
- [ ] 多加载器分支的 Gradle JVM ≥ 插件要求（通常 21+），且与 `java_version` 工具链目标区分开。
- [ ] 构建判定按 `BUILD SUCCESSFUL`，没有依赖管道退出码。
- [ ] 产物名符合 `<mod_id>-<version>-<loader>-<MC 或区间>.jar`，同一版本号不对应两份不同内容的 jar（命名口径见 [01-publishing.md](01-publishing.md)）。
