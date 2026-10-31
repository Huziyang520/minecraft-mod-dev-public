# 2 环境与工具链

> 管什么：Java 分档与"真 JDK"判定、老版本线的 Java 8 定夺、Gradle 三铁律与档位依据、Maven 镜像、`JAVA_HOME` 选择、环境残骸识别、开工前自检。
> 本文件与 `references/` 其余模块同属一个技能包，**没有上位总纲**；结论冲突时以一手证据为准。本文件给出判定手法、命令、依据表与踩坑结论。相关：[06-build.md](06-build.md) §6.1（Gradle 档位完整依据与插件换代）、[07-porting.md](07-porting.md)（按 MC 版本分线）、[10-troubleshooting.md](10-troubleshooting.md)（构建期"空输出"类故障）。

本文件不记录任何一台具体机器的路径与缓存清单。凡是"这台机器上在哪"的问题，都改成**怎么查出来**——路径会随启动器更新、JDK 升级、目录搬迁而失效，方法不会。

---

## 2.1 Java 分档：先按 MC 版本定档，再去找 JDK

### 分档表（按目标 MC 版本，不看当前机器装了什么）

| 目标 MC 线 | 构建 JDK | 依据 |
|---|---|---|
| 1.18.x / 1.19.x / 1.20–1.20.1 / 1.20.2–1.20.4 | **17** | 该代原版与加载器的 class 版本为 61；插件工具链目标 `java_version=17` |
| 1.20.5–1.20.6 / 1.21–1.21.1 / 1.21.2–1.21.5 / 1.21.6–1.21.8 / 1.21.9–1.21.11 | **21** | 该代要求 class 版本 65；Loom 1.13+ / MDG 2.0.x 自身字节码即 21 |
| 26.1.2 / 26.2 / 26.3 | **25** | 该代要求 class 版本 69；MDG 2.0.14x 与 Loom 1.15–1.17 的运行期要求 |
| 1.16.5 | 17（走现代 Gradle 路线时） | 见 §2.2 的路 B |
| 1.12.2 / 1.7.10 | **8**（真 JDK，非 JRE） | 老 ForgeGradle 绑定的 Gradle 2.x / 4.x 只能在 JVM 8 上运行 |

三条硬规则：

1. **档位由目标 MC 线决定，不由"当前机器装了哪一版"决定**。反过来的做法（先看有什么 JDK 再决定做哪个版本）会把版本声明放宽到未验证的范围。
2. **禁止为某条线另装一份 JDK**。缺档就报告，不要自行下载——多份同版本 JDK 会让"到底用的哪一份"变成不可判定问题。
3. **同一工程内不同模块可能要求不同 JVM**：`java_version` 是**工具链目标**（产出什么字节码），不等于"Gradle 跑在哪个 JVM"。两者必须分开判断，见 §2.1.4。

### 2.1.1 如何定位 JDK（而不是记住路径）

按优先级依次尝试，找到即停：

| 顺序 | 来源 | 怎么找 |
|---|---|---|
| ① | 工程显式指定 | 读 `gradle.properties` 的 `org.gradle.java.home`；有则直接用，不再猜 |
| ② | 环境变量 | `JAVA_HOME`；`echo $JAVA_HOME` / `$env:JAVA_HOME` |
| ③ | Gradle 工具链缓存 | 列 `~/.jdks/` 与 `~/.gradle/jdks/` 下的目录（Gradle 的 auto-provisioning 落在后者） |
| ④ | 启动器自带运行时 | 启动器运行目录下按 `java-runtime-*` 命名找（Mojang 官方启动器按 MC 版本分目录存放，常见后缀为 `delta` / `epsilon` 这类代号） |
| ⑤ | 系统安装目录 | 各平台的 JDK 安装根目录，按 `jdk-*` / `jre-*` 目录名找 |
| ⑥ | 兜底 | `settings.gradle` 里声明了 foojay resolver 时，让构建按需下载对应 JDK（仅在确实没有可用档时） |

```bash
# bash：列出所有候选 JDK 目录（不假设具体路径）
for d in "$JAVA_HOME" ~/.jdks/* ~/.gradle/jdks/*; do
  [ -d "$d" ] && printf '%s\n' "$d"
done
```

```powershell
# powershell：同上，并附带每个候选是否带编译器
Get-ChildItem "$env:USERPROFILE\.jdks","$env:USERPROFILE\.gradle\jdks" -Directory -ErrorAction SilentlyContinue |
  ForEach-Object { "{0}  javac={1}" -f $_.FullName, (Test-Path (Join-Path $_.FullName 'bin\javac.exe')) }
```

> 扫描时的两个已知陷阱：① **启动器运行时是嵌套目录**，浅层 `find -maxdepth 3` 扫不到真正的 `bin/`（实测嵌套可达 5 层）；② **半装壳目录**会命中但不可用（只剩 `lib/` 或只剩 `conf/`）。所以"扫到了"不等于"能用"，必须做 §2.1.2 的验证。

### 2.1.2 如何验证"这是真 JDK"（而不是 JRE / 残缺壳）

判定顺序（前两条任一失败即淘汰）：

| # | 检查 | 命令 | 通过判据 |
|---|---|---|---|
| ① | 有编译器 | `"<路径>/bin/javac" -version` | 打印 `javac <版本>`，不是"找不到命令" |
| ② | 版本符合目标档 | `"<路径>/bin/java" -version` | 主版本号 = 目标档（17 / 21 / 25 / 8） |
| ③ | 结构完整 | 列 `bin/`、`lib/` | `bin/` 下有 `javac`（Windows 为 `javac.exe`）；`lib/` 不是空的 |
| ④ | 编译器真能编译 | 写一个最小类，`javac` 编一次 | 产出 `.class`，无报错 |

```bash
# bash：一次性验证某个候选目录是不是可用 JDK
JDK="<你的 JDK 路径>"
"$JDK/bin/javac" -version && "$JDK/bin/java" -version
```

**JRE 与 JDK 的区别（决定"能不能编译"）**：

- javac 依赖 JDK 自带的 `jdk.compiler` 模块（Java 8 时代是 `lib/tools.jar`）。JRE 发布时**这两样都没有**。
- JRE 的特征：`bin/` 下**没有** `javac`，`lib/` 下**没有** `tools.jar`。
- 往 JRE 里拷一个 `javac` 只会得到 `No compiler is provided in this environment`——这是 JRE 的固有行为，不是配置问题。
- 结论：**JRE 能运行游戏，但不能编译**。需要编译老版本线时必须另有真 JDK 8。

**残缺壳的特征**：目录名看着像 JDK，但内容不全（只剩 `lib/`，或只剩 `conf/`）。这类目录被工具链探测选中时，失败方式极难定位（可能是"空输出、无报错"）。识别手法：比对同版本完整 JDK 的顶层目录结构，缺 `bin/` 或 `bin/` 里缺 `javac` 即为壳。

### 2.1.3 为什么"启动器里出现两个 Java 8"

- 现象：启动器的 Java 列表里同版本出现两条，其中一条路径看起来像快捷方式。
- 根因：Oracle JRE 安装器会创建一个指向"最新 JRE 8"的 **junction / 符号链接**（官方原话：*"The JRE installer creates a junction, which acts like a shortcut or symbolic link to the latest installed JRE 8 version"*）。第三方启动器**按路径逐条解析、不按 realpath 去重**，于是同一条物理 JRE 被列了两次。
- 判定：比对两条路径解析后的真实目标；若指向同一目录，即为链接造成的重复，不是重复安装。
- 修复方向：要清理的是**重复条目本身**（那条链接），不是"卸载一个 JRE"。删链接不影响原 JRE。

---

### 2.1.4 Java 版本的三个层次（必须分开判断）

| 层次 | 由谁决定 | 判断方法 |
|---|---|---|
| ① 游戏运行 JVM | 目标 MC / 加载器 / 第三方模组的字节码要求 | 读实例 JSON 的 `javaVersion.majorVersion`；或按 MC 版本分档表 |
| ② 运行 Gradle 的 JVM | Gradle 与插件自身的字节码要求 | 看插件的 class 版本（`javap -verbose` 读 major）；Loom 1.13 / MDG 2.0.123 是 Java 21 字节码 |
| ③ 编译工具链与目标字节码 | 决定编译 API 与输出 class 兼容性 | `java_version` / `options.release`；产出 class 的 major 版本 |

四条推论：

1. 系统 `PATH` 上的 `java -version` 与配置的 `JAVA_HOME` **可以并存、互不干扰**；在版本目录下运行 Gradle 会使用配置好或缓存的 JDK，不需要手改 `PATH`。
2. ⚠ **`java_version=17` 只是工具链目标，不代表 Gradle 能跑在 17**：Loom 1.13 / MDG 2.0.123 的字节码是 Java 21 ⇒ **Gradle 必须跑在 JVM 21+**。症状是构建期报 `Unsupported class file major version 65` 一类错误，而不是业务代码报错。
3. 清理过 JDK 工具链缓存后，构建会经 `settings.gradle` 里声明的 foojay resolver 按需重新下载正确 JDK（仅在必需时）。这是"缓存没了"的正常恢复路径，不是故障。
4. 26.x 构建需要 Java 25 运行时；Loom 1.8 及以下在 Java 17 / 21 上均可运行。

### 三层次错配的症状对照

| 错配 | 症状 | 判定手法 |
|---|---|---|
| Gradle 跑在低于插件要求的 JVM | `Unsupported class file major version 65`（或 69） | 报错在**配置期**，堆栈指向插件类 |
| 工具链目标低于依赖要求 | `class file has wrong version`、`NoSuchMethodError` | 报错在**编译期**，指向某个依赖的类 |
| 游戏 JVM 低于模组要求 | 启动即崩，日志有 `UnsupportedClassVersionError` | 报错在**运行期**，指向本模组或前置的类 |
| 三者都对但档位混用 | 构建成功、实机行为异常 | 比对产物 major 版本与实例 JSON 的 `majorVersion` |

---

## 2.2 老三条（1.7.10 / 1.12.2 / 1.16.5）的 Java 8 定夺

### 2.2.1 关键测量：目标字节码不是瓶颈

用同一份最小类在 17 / 21 / 25 三档上分别跑 `javac --release 8`：**全部成功，产出的 `.class` major 都是 52（Java 8 字节码）**，只有"source/target 8 已过期"的告警。⇒ 三档老分支**不需要**为了"产出 Java 8 字节码"而存在 JDK 8。

这条测量把问题范围缩小了：缺的不是"编译能力"，而是"能跑起老 Gradle 的 JVM"。

### 2.2.2 真正缺的只有一件事

**能跑起老 ForgeGradle 所绑定的老 Gradle 的 JVM**。官方口径：Gradle 9.x 自身要求 JVM 17~26；而 FG 1.2（1.7.10）/ 2.3（1.12.2）那代绑的是 Gradle 2.x / 4.x，跑不到 17+ 上。⇒ 两条出路，**都未实测**：

| 路 | 做法 | 代价 | 适用 |
|---|---|---|---|
| **A** | 补一个真 JDK 8（与保留现有 JRE 不冲突），老 FG + 老 Gradle 原样跑 | 一次下载 + 一份独立目录 | 1.7.10 / 1.12.2 |
| **B** | 换用支持较新 Gradle 的 ForgeGradle 方案（社区维护的老版本 FG 分支、FG 5/6 + Gradle 7/8），继续用现成的 17/21，编译目标写 `options.release = 8` | 需要逐版本实测插件兼容性 | 1.16.5 概率最高，1.12.2 次之，1.7.10 最难 |

- **先探哪条**：拿 1.16.5 当试验品试路 B，成了就省一次下载；不成再走路 A。**禁止先装环境再找理由**——先装再验证的顺序会让"到底需不需要"永久不可判定。
- 社区口径补充：1.12.2 已有 Gradle 8.1.1 + RetroFuturaGradle 的组合，**但 JDK 仍须 8**；1.7.10 有社区维护的 ForgeGradle-1.2 分支。若实测确认，则**路 B 也免不了要一份 JDK 8**，A/B 之争塌缩成"要不要装 JDK 8"。
- 判定手法：先在目标线跑一次 `gradlew --version`，看 Gradle 自身能否在现有 JVM 上启动。启动失败 → 必须换 JVM（路 A 或升级 Gradle）；启动成功但插件加载失败 → 走路 B。

### 2.2.3 "补全现有 JRE"不成立

javac 依赖打包进 JDK 的 `jdk.compiler` 模块（Java 8 时代是 `lib/tools.jar`），Oracle JRE 发布时**就没有**这两样；往 JRE 里拷 `javac.exe` 只会得到 `No compiler is provided in this environment`。补全 JRE 在技术上不成立，不要在"补哪几个文件"上花时间。

可以**零覆盖、零卸载、零重复下载**地达成目标：

- 现有 JRE **一个字节都不动**，继续负责"运行老客户端"；
- 另解压一份 **JDK 8 zip**（Adoptium / Corretto / Zulu 均提供 zip）到独立目录（放 `~/.jdks/` 下，Gradle 工具链会扫这里），只当编译器用；
- 不改 `PATH`、不跑安装器、不碰已有的 17/21/25 —— 那三档**已经全部就位，无需再下一份**。

### 2.2.4「把 JDK 8 原地覆盖到现有 JRE」的可行性判定

| 主张 | 结论 | 证据 |
|---|---|---|
| 覆盖后老游戏实例还能跑 | ✅ 能 | JDK 8 顶层就有 `bin/java.exe` / `javaw.exe` 且自带 `jre/`；启动器调用的正是 `<该目录>/bin/java.exe`。1.7.10 / 1.12.2 / 1.16.5 本就跑在 Java 8 上 |
| 覆盖后"只会有一个 java8" | ❌ 不成立 | 那条 junction 仍指向该目录，启动器对扫到的每条路径逐个解析、不按 realpath 去重 ⇒ 两条都进列表；**覆盖并不能把两条变成一条** |
| 覆盖没有副作用 | ❌ | ①注册表 `JavaSoft` 的 JRE 键与卸载项仍写旧版本号并指向该目录，Java 更新的"修复/卸载"会作用在非 Oracle 内容上；②`java -version` 的厂商与 build 变化 ⇒ 启动器列表里那一项**显示名会变**，容易被当成"原来的 8 不见了"；③目录名叫 `jre…` 里面却是 JDK ⇒ 又造一处名实不符；④回滚要重装 JRE |

**优先级（从最省事到最不划算）**：

1. **先试路 B（可能整个不需要 Java 8）**：1.16.5 用现代 Gradle + `options.release = 8` 试，三档现成 javac 已实测能产出 major 52。
2. **构建端歧义本来就不该靠"物理唯一"来消**：老三条各自 `gradle.properties` 里钉死 `org.gradle.java.home=<唯一那个 JDK 8 路径>`，扫描器找到几个都无所谓。
3. **若就是要列表干净**：该动的是**重复条目本身**——那条 junction 才是同一路径被列两次的来源；其次清理半装壳目录（只剩 `lib/` 或只剩 `conf/` 的那种）。**这些残骸才是"Java 列表里冒出看不懂条目"的真凶**，比覆盖安装目录对症得多。
4. **原地覆盖 JDK 8** 排最后：技术可行、能跑，但既得不到"唯一"，又多一处名实不符。若仍决定这么做，顺序必须是：先记录 `java -version` 全文并导出 `HKLM\…\JavaSoft` 相关键 → JDK 8 解压到临时目录 → 停掉 Java 更新计划任务 → 清空并拷入 → 回验 `<原路径>/bin/java -version` 与老实例启动 → 留一份可回滚的原件。

> 系统目录属使用者机器，**任何一条落地前都要明确下令**；本文件只记判定，不代自动执行。

### 2.2.5 老三条的运行环境约束（交叉引用）

1.7.10 / 1.12.2 / 1.16.5 的实例 JSON 都写 `javaVersion.majorVersion = 8`，且启动器按**路径**选中那份 JRE 的 `bin/java.exe`。⇒ **任何改动该路径的动作（换 JDK、删目录、改目录名）都会直接影响这三个实例能否启动**；改前必须先"改名保留回滚位"，改后必须开一次世界复验。

---

## 2.3 Gradle 三铁律

> 接手任何构建 / 移植任务，先按本节自查再动手。档位的完整依据（Loom 版本矩阵、FG6/FG7 换代）见 [06-build.md](06-build.md) §6.1。

### 2.3.1 铁律一：先查缓存，能复用就不下载

**动手前必做**：列一次已缓存发行版，确认目标版本**是否已在缓存里**。

```bash
# bash：列出已缓存的 Gradle 发行版（只看目录名，不看内容）
ls -1 ~/.gradle/wrapper/dists 2>/dev/null | sort -V
```

```powershell
# powershell：同上
Get-ChildItem "$env:USERPROFILE\.gradle\wrapper\dists" -Directory -ErrorAction SilentlyContinue |
  Select-Object -ExpandProperty Name | Sort-Object
```

- **能复用就复用**：若任务需要"最新档"，而缓存里是 `9.6.1` 而不是文档里写的 `9.6.0`，就把工程 wrapper 指到 **`9.6.1`**（同为 9.6 系列补丁版），**不要为了对齐某个数字去新下一份**。任何"改一格版本号"的动作前，先问一句：**这一版缓存里有吗？**
- **允许下载的唯一条件**：① 缓存里**确实没有**该档位；② 该档位是任务**必需**（例如工程插件只吃 8.x，而缓存里 8.x 只剩 8.11）。满足两条才允许触发一次下载，并在 `gradle-wrapper.properties` 注释里写明"为什么必须下这一版"。
- **禁止**：为了"保持最新""对齐文档""顺手试一下"而下载缓存里已有同类版本；也禁止缓存里已有可用版本时，仍按模板 / 文档里的旧数字去拉一份新的。
- **违反后果**：无谓占磁盘、拖慢构建、污染"到底该用哪版"的判断基准。**实测代价**：同一发行版曾出现 **5 份**不同来源副本、单份解压后 **685 个文件**、单目录 **744MB**；此后每次判断"该不该下载"都要先排除这堆副本，等于把一次小决定变成一次审计。
- **本条是快照类动作**：每次要判"该不该下载某档"，**必须重新列一次目录**。任何写死的"缓存清单"都会在几天内过期。

### 2.3.2 铁律二：档位由插件决定，插件由 MC 版本决定 —— 不许一刀切

Gradle 大版本会移除老插件依赖的 API，因此**不存在"一个 Gradle 版本通吃 1.18.2→26.3"**。

| MC 线 | 构建 Java | Fabric 插件 | Forge / Neo 插件 | Gradle 档 |
|---|---|---|---|---|
| **26.3 / 26.2 / 26.1.2** | 25 | Loom 1.15–1.17 | ModDevGradle 2.0.14x | **最新档 `9.6.1`** |
| 1.21.8 / 1.21.11 | 21 | Loom 1.13-SNAPSHOT | ModDevGradle 2.0.123 | **`8.14.3`**（受 Loom 1.13 约束） |
| **1.21.1 / 1.20.1 / 1.19.2 / 1.18.2** | 21 / 17 | Loom 1.8 / 1.9-SNAPSHOT | **ForgeGradle 6** | **遗留档 `8.11`** |

**依据（已核实，非推测）**：

- **Fabric Loom**：官方发布说明 —— `1.14` 起要求 **Gradle 9.2**，`1.16` 起要求 **Gradle 9.4**；`1.8 / 1.9-SNAPSHOT` 只在 **Gradle 8.11** 可用（Gradle 8.9+ 移除了 `Problems.forNamespace`，Loom ≤1.7 会报 `LoomProblemReporter` 实例化失败）。
- **ForgeGradle 6**：其 `6.0` 分支**自身就用 Gradle 8.1.1 构建、编译目标 Java 8、直接依赖 `gradleApi()`** → 绑定 Gradle 8.x 的 API；FG 已进入维护末期、**无 Gradle 9 适配迹象** → Forge 全线（1.18.2 / 1.19.2 / 1.20.1 / 1.21.1）必须留在 8.x。
- **结论**：一个 MultiLoader 多版本工程**至少需要 3 档** —— `9.6.1`（26.x）+ `8.14.3`（1.21.8 / 1.21.11）+ `8.11`（Forge 全线 / 1.18.2–1.20.1 的 Fabric）。**只留一档会让对应线直接构建失败。**
- 想砍掉 `8.14.3`：必须先把 1.21.8 / 1.21.11 线的 Loom 从 1.13-SNAPSHOT 升到 1.17.x（**逐版本实测**），才能并入 `9.6.1`。**未实测前不许动。**

**档位归属的判定手法**（不要靠记忆）：

1. 读工程 `gradle/wrapper/gradle-wrapper.properties` 的 `distributionUrl` → 得到候选档；
2. 读 `gradle.properties` / 根 `build.gradle` 的插件版本（Loom / MDG / FG）；
3. 按上表反查该插件允许的 Gradle 区间；
4. 若两者不匹配，**改 wrapper 或改插件**，二选一，不要两头各退半步。

### 2.3.3 铁律三：Maven 默认走镜像，且镜像放在官方源之前

- 构建默认启用国内镜像（阿里云 / 腾讯云），无特殊原因不直连境外源；**镜像要放在官方源之前**：

```groovy
maven { name = 'AliyunPublic';  url = 'https://maven.aliyun.com/repository/public' }
maven { name = 'AliyunGradle';  url = 'https://maven.aliyun.com/repository/gradle-plugin' }
maven { name = 'TencentPublic'; url = 'https://mirrors.cloud.tencent.com/nexus/repository/maven-public/' }
```

- 镜像地址会变，失效时先访问确认再替换，**别照抄过期地址**。
- **必须知道的事实**：国内公共镜像**只覆盖** `mavenCentral` / `gradlePluginPortal` / `google`；**Fabric（`maven.fabricmc.net`）、NeoForged（`maven.neoforged.net`）、Forge（`maven.minecraftforge.net`）、Mojang（`libraries.minecraft.net` / `piston-meta.mojang.com`）都没有国内镜像** —— MC 本体 / 映射 / 加载器 / Fabric API 仍走官方源。所以"配了镜像"≠"离线能构建"，网络排查要分清这两类源。
- 第三方插件自己的 Maven 源（如 CurseGradle 的 `maven.maxhenkel.de`）单独加。
- `buildSrc` / `build-logic` 里的约定脚本：Maven 依赖也要加同一组镜像，否则"主工程能构建、约定插件拉不到"。

| 排查现象 | 更可能的根因 | 先查哪个源 |
|---|---|---|
| 镜像配了但仍超时 | 目标是 Fabric / NeoForge / Forge / Mojang 源 | 该官方源可达性 |
| 只有约定插件下载失败 | `build-logic` 未加镜像 | `buildSrc` / `build-logic` 的仓库块 |
| 换镜像后出现校验失败 | 镜像返回了不同的元数据或 302 | 该坐标在官方源的元数据 |
| 单机离线可用、换机不可用 | 依赖只落在本地缓存 | `~/.gradle/caches/modules-2` |

### 2.3.4 接手任务强制动作：检查 → 核实 → 写注释

| 步骤 | 做什么 | 判据 |
|---|---|---|
| **检查** | 读 `gradle-wrapper.properties` 的 `distributionUrl`；列缓存确认该版是否已有；按 §2.3.2 判断属于哪一档 | 三个结论齐全：档位 / 缓存有无 / 依据 |
| **核实** | **不许凭记忆下结论** —— 实际跑一次 `gradlew projects`（配置期）+ 目标模块 `jar`（构建期） | 配置期不报错；构建期输出含 `BUILD SUCCESSFUL` |
| **写注释** | **每个版本目录**的 `gradle-wrapper.properties` 顶部（或 `gradle.properties`）写清四件事 | 见下方模板 |

```properties
# Gradle 9.6.1 = 最新档（复用已有缓存，未新下载）
# 约束来源：Fabric Loom 1.17.x / ModDevGradle 2.0.147（Loom 1.16+ 要求 Gradle 9.4+）
# 镜像：Aliyun public + gradle-plugin（见 settings.gradle）
# 实测：YYYY-MM-DD 通过 `gradlew :fabric:jar :neoforge:jar`
distributionUrl=https\://services.gradle.org/distributions/gradle-9.6.1-bin.zip
```

```properties
# Gradle 8.11 = 遗留档（复用已有缓存）
# 约束来源：ForgeGradle 6（仅支持 Gradle 8.x）+ Loom 1.8 / 1.9-SNAPSHOT
# 镜像：Aliyun public + gradle-plugin
# 实测：YYYY-MM-DD 通过 `gradlew :forge:jar`
```

**违反后果**：不核实就动版本 → 构建失败且难定位；不写注释 → 下次接手要全量重验；下错 / 多下发行版 → 磁盘与判断基准双污染。

---

### 2.3.5 缓存结构与"离线能不能构建"

判断"这台机器能不能离线构建"时，要看的是**缓存里有什么**，不是"配没配镜像"。

| 缓存位置 | 存什么 | 能否离线用 |
|---|---|---|
| `~/.gradle/wrapper/dists/gradle-<版本>-bin/<hash>/` | Gradle 发行版本体 | ✅ 有对应版本即可离线跑 Gradle 本体 |
| `~/.gradle/caches/modules-2/files-2.1/` | 依赖 jar 与 pom | ✅ 该坐标已下载过才可离线 |
| `~/.gradle/caches/modules-2/metadata-*/` | 元数据（版本列表、解析结果） | ⚠ 动态版本（`+` / `-SNAPSHOT`）仍会尝试联网刷新 |
| 工程 `build/` | 上次构建产物 | ❌ 换机即失效，不要当依赖来源 |
| 本地 `libs/` 目录 | 手动放置的 jar | ✅ 但属于"绕过依赖管理"，需显式声明 |

三条结论：

1. **动态版本与 SNAPSHOT 是离线构建的敌人**：`1.13-SNAPSHOT`、`2.0.+` 这类声明每次都要联网确认最新值，缓存再全也可能卡住。要么改成固定版本号，要么接受"必须联网一次"。
2. **"配了镜像"不等于"能离线"**：镜像只改变下载来源，不改变"没下载过就没有"这个事实。而且 Fabric / NeoForge / Forge / Mojang 四类源没有国内镜像（见 §2.3.3），这些依赖第一次解析必然要出网。
3. **离线失败的症状是"卡在解析依赖"而不是报错**：表现为长时间无输出后超时。判定手法是加 `--offline` 再跑一次——若立刻报"找不到依赖"，说明确实是缓存缺失；若仍卡住，问题在动态版本刷新。

```bash
# 判断某坐标是否已在本地缓存
find ~/.gradle/caches/modules-2/files-2.1 -maxdepth 2 -type d -name '<group>' 2>/dev/null
```

---

## 2.4 `JAVA_HOME` 对照表（按分支所属 MC 线选，别混用）

| 分支线 | 构建 JDK | 取值来源 |
|---|---|---|
| `26.1-26.2`、`26.3` | **25** | 启动器运行时里的 Java 25，或 `~/.jdks` 下的 JDK 25 |
| `1.20.5-1.20.6`、`1.21-1.21.1`、`1.21.2-1.21.5`、`1.21.6-1.21.8`、`1.21.9-1.21.11` | **21** | 启动器运行时里的 Java 21（自带编译器），或 `~/.jdks` 下的 JDK 21 |
| `1.18.x`、`1.19.x`、`1.20-1.20.1`、`1.20.2-1.20.4` | **17** | 系统安装的 JDK 17 |
| `1.16.5` | **17** | 走 §2.2 路 B |
| `1.7.10`、`1.12.2` | **8**（真 JDK） | 见 §2.2 路 A / B 结论 |

> ⚠ 上表最后一行**不采用**"覆盖原 JRE"的做法（判定见 §2.2.4，属**已作废**方案）。当前口径：不要覆盖 JRE；要么并排放 JDK 8，要么先走路 B 实测。

**实测补证**：多条 Forge 分支的**上传任务**（`cfUpload` 这类脚本型 Gradle 任务）在 Java 21 与 Java 25 下都能跑通。⇒ **上传 / 脚本类任务可用的 JVM 档比"构建档"更宽**，但仍坚持**按分支线就近取档**，不要一档通吃（同一工程在别的档仍可能因插件字节码要求而失败）。

**`JAVA_HOME` 设置的三个口径**：

1. **显式传参优于环境变量**：构建命令里直接指定（`./gradlew ... ` 前设 `JAVA_HOME`），避免系统变量被别的工具改动后产生"昨天能构建、今天不能"。
2. **一目录一档**：每个版本目录的 wrapper 注释里写清该档；跨目录批量操作时逐目录核对，不许假设"都一样"。
3. **不要改系统 `PATH`**：`PATH` 影响的是所有工具；改它会把"某条线构建失败"变成"所有工具都异常"。

```bash
# bash：按分支线选档（示意，路径用占位）
JAVA_HOME="<你的 JDK 21 路径>" ./gradlew :fabric:build :neoforge:build --console=plain
```

---

## 2.5 环境残骸识别（不是"删目录清单"，是"怎么认出来"）

残骸的共同特征：**名字像正式组件，内容不全，被工具链选中后失败方式难定位**。识别四类：

| # | 特征 | 症状 / 影响 | 处理 |
|---|---|---|---|
| ① | JDK 目录只剩 `conf/`（缺 `bin/`） | 被工具链探测选中会以难查方式失败 | 移开或删除；改用完整 JDK |
| ② | 工具链缓存里的 `*.zip.part` | 未完成的下载残留，占空间且可能被误当缓存 | 删除后重试下载 |
| ③ | JDK 目录只剩 `lib/`（缺 `bin/`） | 半截卸载残留，易与完整同版本 JDK 选错 | 移开；路径配置时别选到壳目录 |
| ④ | `latest/…` 形式的链接指向同版本 | 启动器列表里同版本出现两条 | 见 §2.1.3 |

> 处理需使用者在场；**建议移开或删掉前先确认**，不要在无人授权时动系统目录。

**识别手法（通用）**：对每个候选目录做"结构完整性 + 编译器存在 + 版本符合"三项检查（§2.1.2），三项全过才算可用。只凭目录名判断，必然踩到壳目录。

---

## 2.6 构建前必看：磁盘与进程

| 检查项 | 命令 | 判据 |
|---|---|---|
| 系统盘剩余空间 | `df -h` / `Get-PSDrive` | **≥10GB**；不足先清理可再生缓存 |
| Gradle 缓存位置 | `~/.gradle` | wrapper dists、工具链临时目录都在用户目录侧，不在工程目录 |
| 残留 Gradle / Java 进程 | 进程列表按 `java` / `gradle` 过滤 | 无残留；有则先结束再构建 |
| 目标实例是否在跑 | 进程列表按启动器 / `javaw` 过滤 | 部署前必须无游戏进程 |

- 空间不足时的失败方式很隐蔽：**"空输出 / 无报错"**，或被误判成代码问题。判定手法：先看磁盘余量，再读构建日志；**不要在一堆"莫名失败"里猜代码**。
- 清理只清**可再生缓存**（下载临时文件、旧日志），**不碰 `~/.gradle/wrapper/dists`**——那里是老版本线构建的唯一依赖来源。
- 空间检查已从"每次构建必查"降级为"构建失败且无明确报错时先查"，但**发布前仍要看一眼**。

---

## 2.7 可选联动 API 的依赖口径（`compileOnly`）

**铁律**：可选联动的第三方 API 一律 `compileOnly` / `modCompileOnly`：**不入 jar、不作前置、缺失时模组照常加载**。违反的后果是"用户没装该模组就崩"，而且崩在类加载阶段，日志指向本模组而非缺失的前置。

| 可选项 | 版本 | 仓库 | 声明位置 | 备注 |
|---|---|---|---|---|
| Cloth Config | `11.1.136` | `https://maven.shedaniel.me/` | common / forge `compileOnly`，fabric `modCompileOnly` | Forge 工件用官方映射发布，**无需 `fg.deobf`** |
| Mod Menu | `7.2.2` | `https://maven.terraformersmc.com/releases/` | fabric `modCompileOnly` | 仅提供模组列表的配置按钮 |
| **Jade (Waila)** | **`11.13.3`** | **`https://api.modrinth.com/maven`**（Modrinth Maven） | common / forge `compileOnly`、fabric `modCompileOnly` | 坐标 `maven.modrinth:jade:<版本>+forge` / `+fabric`；1.20.1 上两端同名版本都能解析；**API 用官方映射发布**，同样无需 `fg.deobf` |

### Jade 适配的关键事实（`javap` 逐条核对，非推测）

- **插件入口**：Forge 用 `@snownee.jade.api.WailaPlugin("<modid>")` 注解的 `IWailaPlugin` 实现；Fabric 用 `fabric.mod.json` 的 **`"jade"` entrypoint**（Jade 的 `CommonProxy` 正是用这个名字取 entrypoint）。
- **客户端注册**：`IWailaClientRegistration#registerBlockComponent(IBlockComponentProvider, Class<? extends Block>)`；每个显示项用 `addConfig(ResourceLocation, boolean)` 注册**插件自己的开关**，默认值随注册写入，落在 `config/jade/` 的插件配置里。
- **插件配置界面的翻译键约定**：**`config.jade.plugin_<插件id>.<键名>`**（插件 id = 注解值 / Fabric 侧即模组命名空间）。
- **数据来源**：`BlockAccessor#getBlockEntity()` 就是**客户端已同步的方块实体**（倍率 / 红石模式 / 所有者都在其中），**不需要服务端数据包**，因此这个联动是纯客户端代码。
- **文档入口**：`https://jademc.readthedocs.io/en/latest/`（首页信息量很少，实际签名请以 jar 内 `snownee.jade.api` 包的 `javap` 结果为准）。

**判定手法（不要凭文档写实现）**：

```bash
# 核对 Jade API 的真实签名（解包到本地后）
javap -classpath <jade.jar> -p snownee.jade.api.IWailaClientRegistration | grep registerBlockComponent
```

### 该线的构建环境复核（可复现的固定动作）

- wrapper `8.11` 走已有缓存（`~/.gradle/wrapper/dists/gradle-8.11-bin`）；Gradle JVM 用启动器自带的 Java 21，工具链目标仍是 `java_version=17`。
- 构建命令（在工程根执行）：`./gradlew :forge:build :fabric:build --console=plain`。
- `forge` 是**根工程子模块**（没有自己的 `settings.gradle` / wrapper），任务名必须带冒号前缀 —— 判据是"看 `forge/settings.gradle` 在不在"，不在就是子模块。

---

## 2.8 环境变更后的复验顺序（改完必跑）

环境是**共享依赖**：改一处，影响所有版本线。所以任何环境变更都不能只验"改动的那条线"。

| 变更 | 必须复验 | 顺序理由 |
|---|---|---|
| 新增 / 替换某个 JDK | 用该档的所有版本线各跑一次配置期（`gradlew projects`） | 配置期就能暴露"插件字节码高于 JVM"这类错配，比跑完整构建快一个数量级 |
| 改 `~/.gradle/wrapper/dists` 内容 | 三档各跑一次目标模块 `jar` | 档位缺失表现为"另一条线突然拉不到发行版"，与业务代码无关 |
| 改镜像 / 仓库顺序 | 三条线各跑一次**冷缓存**构建（`--refresh-dependencies`） | 镜像顺序问题只在重新解析时暴露，热缓存会掩盖 |
| 改 `PATH` / 系统 `JAVA_HOME` | 所有线的构建命令显式传参复核 | 系统变量被改后，所有工具同时受影响，故障面最大 |
| 改实例 JSON 的 `javaVersion` | 该实例启动一次并进世界 | 启动器按路径选 JVM，改值不一定改路径，两者可能不一致 |
| 删任何目录 | 按 §2.1.2 重新验证剩余候选，并逐条线跑配置期 | 删掉的可能是某条线唯一可用的那一份 |

**判定手法（"改完到底有没有坏"）**：不看构建日志的结尾，看**配置期是否通过**。配置期通过、构建期通过、实机进世界通过，三段都过才算环境变更成功。只验一段的后果是把问题推迟到下一次发布。

**回滚位**：改动系统目录前，先把原目录**改名保留**（而不是删除），改完复验失败就改回来。改名保留的成本是一次重命名，删除的成本是重装。

---

## 2.9 环境类自检清单（开工前 / 环境变动后跑一遍）

- [ ] §2.1 分档表对应的四个 JDK 档（17 / 21 / 25 / 8）都能定位到**可用**路径；每个都过了 §2.1.2 的四项验证。
- [ ] 缓存里没有"同版本多份副本"（列 `~/.gradle/wrapper/dists`，同名不同 hash 的目录超过一个即为多份）。
- [ ] `gradlew` 不可用时，缓存发行版二进制路径仍存在（`~/.gradle/wrapper/dists/gradle-<版本>-bin/<hash>/gradle-<版本>/bin/gradle`）；Gradle 版本与各仓库 wrapper 一致。
- [ ] 每个版本目录的 `gradle-wrapper.properties` 注释四要素齐全，且日期不早于上次构建。
- [ ] 三档（`9.6.1` / `8.14.3` / `8.11`）**一个都没被删**；删任何一档前已确认该线有替代方案并实测。
- [ ] 镜像块在官方源之前；`build-logic` / `buildSrc` 也加了镜像。
- [ ] 每个候选 JDK 都验过 `javac -version`；JRE 与残缺壳已排除，未被写进 `org.gradle.java.home`。
- [ ] 系统盘剩余空间 ≥10GB（发布前必看）。
- [ ] 无残留 Gradle / Java 进程；部署前无游戏进程。
- [ ] 新 MC 大版本时：Java 分档表、模板分支、目标实例三处都要动。
- [ ] 改动过任何系统目录（JDK / 链接 / 注册表）后，老版本线实例各启动一次复验。
