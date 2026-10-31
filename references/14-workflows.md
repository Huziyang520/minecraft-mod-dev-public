# 14 标准工作流（跨模块串联）

> 本文件把散在各章的步骤串成**可直接照做的清单**：从零建模组、扩版本分支、发版本、线上排障。
> 上游：`SKILL.md`（结论以此为准）；本文件给逐步动作、判定点与退出条件。
> 交叉引用：[01-publishing.md](01-publishing.md) · [02-environment.md](02-environment.md) · [03-metadata.md](03-metadata.md) · [06-build.md](06-build.md) · [07-porting.md](07-porting.md) · [08-deploy-qa.md](08-deploy-qa.md) · [09-git-upload.md](09-git-upload.md) · [10-troubleshooting.md](10-troubleshooting.md)

---

## 14.1 通用前提（四条工作流都适用）

1. **先探针后改码**：任何"这个版本应该也能跑"的判断，必须先有 `javap` / maven 元数据 / 实机证据。
2. **先构建后部署**：产物在 `build/libs` 里才算构建完成。
3. **先本地验收后发布**：用户实机通过**并主动下令**才允许上传 / 推送。
4. **阶段必须分清**：已构建 / 已部署待验收 / 已验收 / 已上传 —— 汇报时逐项声明，不许混着说。

---

## 14.2 从零建新模组

### 步骤 1：定身份（先问，不要自己编）

- [ ] **逐模组问版本号**，给出预设（patch / minor / 进位 / 补丁进两位 / 用户自填）。
- [ ] **紧接着问 CurseForge `releaseType`**（`release` / `beta` / `alpha`）。
- [ ] 确定 `mod_id`（必须匹配 `^(?=.{2,64}$)[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`）。
- [ ] 确定 `mod_name`（人类展示名，允许空格与大写）。
- [ ] 确定 `license`、`mod_author`、主页 URL、反馈 URL。

> 细节：[01-publishing.md](01-publishing.md) §1.1–§1.4。

### 步骤 2：选模板与工具链档位

- [ ] 按 MC 版本从 `MultiLoader-Template` 拉对应分支；没有对应分支就复制同族最接近的目录作骨架。
- [ ] 读 `gradle/wrapper/gradle-wrapper.properties` 的 `distributionUrl`，**核对你机器上的 Gradle 缓存里有没有这一版**（[02-environment.md](02-environment.md) §2.3 铁律一）。
- [ ] 判断属于哪一档（现代档 / 遗留档），确认插件版本与 Gradle 档**互相支持**（[06-build.md](06-build.md) §6.1）。
- [ ] 低版本移植时**删掉 parchment**（版本不存在会导致下载失败）。
- [ ] 补 `.gitignore`（**别假设模板已带**）。

### 步骤 3：填 `gradle.properties`

```properties
mod_id=examplemod
mod_name=Example Mod
mod_author=YourName
license=MIT
version=1.0.0
description=One-line summary. Issues: https://github.com/<you>/<repo>/issues
```

- [ ] `description` **单行**，英文在前、中文在后用 ` | ` 分隔。
- [ ] `version` 只写模组版本，**不带 MC 前缀**。

### 步骤 4：写元数据

- [ ] `fabric.mod.json`：`id` / `name` / `version` / `description` / `authors` / `license` / `icon` / `contact` / `depends` / `entrypoints` / `mixins`。
- [ ] `neoforge.mods.toml`：`displayURL` / `iconFile`（26.2+）或 `logoFile`（≤26.1.2）在 **`[[mods]]` 内**；`issueTrackerURL` 在**文件顶层**；`[[mixins]]` 显式声明。
- [ ] `icon.png` 128×128（读 PNG IHDR 核实，不看缩略图）。
- [ ] 26.3 起**删掉** `modLoader` / `loaderVersion` 两行。

> 细节：[03-metadata.md](03-metadata.md) §3.1–§3.3。

### 步骤 5：资源与本地化

- [ ] `pack.mcmeta`：NeoForge / Fabric 线不含数据包内容就**不放**；Forge 线**必须补最小版**且只放 forge 模块。
- [ ] 语言文件：键与 `en_us` 对齐、UTF-8 无 BOM；业务模组建议 ≥13 种、库 / 前置 ≥ `en_us` + `zh_cn`。
- [ ] 玩家可见文本走可翻译组件，颜色由组件样式给，**不用 `§` 色码、不用 `literal` 顶替**。

> 细节：[04-i18n.md](04-i18n.md) §4.1–§4.5。

### 步骤 6：构建配置

```groovy
tasks.withType(JavaCompile).configureEach { options.encoding = 'UTF-8' }
```

- [ ] `options.encoding = 'UTF-8'`（不设会在 Windows 上读坏中文注释）。
- [ ] 不需要 javadoc 就**移除 `withJavadocJar()`**（Java 25 下会卡死构建）。
- [ ] 移除后模块脚本改成 `configurations.matching { it.name == variant }.configureEach { ... }`。
- [ ] 删掉模块内 `base { archivesName = ... }` 覆盖，交给约定插件统一拼。
- [ ] 产物名 = `<mod_id>-<version>-<loader>-<mc 或区间>.jar`。

### 步骤 7：首次构建 + 部署

- [ ] **显式设 `JAVA_HOME`**（按 MC 线取档，[02-environment.md](02-environment.md) §2.4）。
- [ ] 先 `gradlew projects`（配置期），再 `:模块:jar`（构建期）；判定看 `BUILD SUCCESSFUL`。
- [ ] **首次构建前问目标测试实例**（放哪个 / 双端）；确认后后续构建直接自动部署。
- [ ] 部署三查：版本号取自 `gradle.properties`、产物 mtime 是本次构建、**从包内回读**确认改动在里面。
- [ ] 删实例 `mods/` 内被替换的旧 jar；`build/libs` 旧 jar 留原位。
- [ ] MD5 比对实例 jar 与 `build/libs` 产物；**部署成功并记录 MD5 之前不得结束输出**。

> 细节：[08-deploy-qa.md](08-deploy-qa.md) §8.3。

### 步骤 8：验收

- [ ] 按 [08-deploy-qa.md](08-deploy-qa.md) 的验收清单逐条过。
- [ ] 本地化：**实机切换语言**触发一次提示，确认显示对应语言而不是英文 / key。
- [ ] 混装：与常见第三方模组共存时启动不崩、它们自己的客户端指令不被误拦。

### 步骤 9：发布（用户主动下令后）

- [ ] 自检：发布字段、元数据、本地化、构建、Git 五份清单。
- [ ] 写 changelog（只版本小标题 + 本次真实改动）。
- [ ] CurseForge / Modrinth 上传（[09-git-upload.md](09-git-upload.md) §9.4–§9.5）。
- [ ] Git 推送（[09-git-upload.md](09-git-upload.md) §9.1–§9.2）。

---

## 14.3 扩一个 MC 版本分支

### 步骤 1：探针（不写代码，先钉事实）

```bash
javap -p -s <Commands.class>      | grep -A2 performCommand
javap -p -s <ServerPlayer.class>  | grep -A1 setGameMode
javap -p -s <ServerPlayer.class>  | grep -i permission
```

- [ ] 每个待覆盖 release 都跑一遍，记录 A / B / C 三处触点的**真实描述符**。
- [ ] 老版本线要用**它自己的映射名**重跑（类名完全不同）。
- [ ] 查兜底 API 的引入版本（如 `translatableWithFallback`）。
- [ ] 查 maven 元数据确认该 release 有对应加载器构建（L3）。

> 细节：[07-porting.md](07-porting.md) §7.3、§7.6。

### 步骤 2：判据

| 探针结果 | 动作 |
|---|---|
| 与相邻分支**全部一致** + L2 同档 + L3 可用 | **并入该分支**，放宽 `minecraft_version_range` |
| 任一触点不同 | **另开分支** |
| L2 跨档（Java 大版本变） | **必须另开分支**，即使 L1 零差异 |

- **合并的三份证据**：源码 `git diff -w` 零差异、触点同形态、L2 同档。**缺一即分叉。**

### 步骤 3：建目录

- [ ] 每版本一个独立目录，**目录名 = 分支名 = jar 区间标签**。
- [ ] 复制**同族最接近**的目录作骨架。
- [ ] 26.x 升一个小版本通常只改 4 个文件：`gradle.properties`、根 `build.gradle`、`gradle-wrapper.properties`、`common/build.gradle`。
- [ ] 跨 MC 版本重建前 **clean**（`build/`、`.gradle/` 被 `.gitignore` 忽略，切分支不会清）。

### 步骤 4：落 6 处版本声明（漏改任何一处都会"能跑却装不上"或"装得上却崩"）

| 声明处 | 位置 | 放宽时注意 |
|---|---|---|
| `minecraft_version` | `gradle.properties` | 编译目标，仍是**单个**版本 |
| `minecraft_version_range` | 同上 | 上界对齐下一个 L2 / L1 硬墙；下界前移要逐个 release 有 L1 证据 |
| Fabric `depends.minecraft` | `fabric.mod.json` | 前缀约束语法，跨 minor 必须改写 |
| Fabric `depends.fabric-loader` | 同上 | 有硬下限 |
| Fabric `depends.fabric-api` | 同上 | 用到 API 类就是真前置 |
| NeoForge / Forge 元数据 + loader range | `mods.toml` / `neoforge.mods.toml` | 改名时点在 1.20.6 前后 |

### 步骤 5：按需适配（只做版本适配，不夹带改行为）

- [ ] Mixin `remap`：1.20.6+ Forge 用 `false`；1.18 / 1.19 用 `true`。
- [ ] Mixin `compatibilityLevel`：按**目标加载器自带 Mixin 版本**重估（≤1.20.6 Forge = `JAVA_17`）。
- [ ] `mixins.json` 是否需要 `"refmap"`（FG6 + SRG 必需）。
- [ ] EventBus 6 / 7（Forge 1.21.8 起必须用事件类自带 `BUS`）。
- [ ] 客户端消息发送 API、权限判定 API、GUI 渲染 API（26.x 改名清单）。
- [ ] 26.x 注册键：方块 / 物品要在构造**之前**带 `setId(ResourceKey)`。

> 铁律：**移植期间只允许因 API 形态被迫产生的差异**；需要改行为的，停下来问，不在移植里夹带。

### 步骤 6：构建

- [ ] 显式设 `JAVA_HOME`。
- [ ] `gradlew projects` → `:模块:jar`；判定看 `BUILD SUCCESSFUL`。
- [ ] 确认**没有**用 `options.release` / `targetCompatibility` 降字节码（要用就编译后改 class major）。
- [ ] 独立 forge 工程的判据：`<分支>/forge/settings.gradle` **在不在**（在 ⇒ 用 forge 目录自己的 wrapper，任务 `jar`；不在 ⇒ 用分支根 wrapper，任务 `:forge:jar`）。

### 步骤 7：部署 + 验收

- [ ] 有对应测试实例 → 实机验收。
- [ ] 无实例 → 只能编译验证，**声明范围必须诚实标注"未实机"**。
- [ ] 每个新覆盖的 release 至少各一次装载成功。

### 步骤 8：发布侧对齐

- [ ] CF `gameVersions` 按**分支设计区间全列**（如 `1.21.9-1.21.11` → 三个版本都勾）。
- [ ] `versionTypes` 按该 release 加载器只有 beta 时**如实标 beta**。

---

## 14.4 发一个版本

### 步骤 1：确认版本号与 releaseType

- [ ] 逐模组问版本号（给出预设）。
- [ ] 逐模组问 `releaseType`。
- [ ] 用户事后改号 → **重新构建 + 重新部署 + 重新验收**，不许只改文档。

### 步骤 2：五份自检

- [ ] **发布字段**（[01-publishing.md](01-publishing.md) §1.7）：图标 128×128、URL 可访问、`mod_name` 是展示名、前置区间无倒挂、同版本号无两份 jar。
- [ ] **元数据 / 工程**（[03-metadata.md](03-metadata.md) §3.6）：字段层级、`description` 单行、语言无 BOM、mixin 两端成对、`pack.mcmeta` 按线决定、产物名规范、严格 JSON 解析、语言键差集为 0。
- [ ] **本地化**（[04-i18n.md](04-i18n.md) §4.5）。
- [ ] **构建**（[06-build.md](06-build.md) §6.6）。
- [ ] **Git 与平台**（[09-git-upload.md](09-git-upload.md) §9.6）。

### 步骤 3：changelog

- [ ] 只写版本小标题（`### <version> (<日期>)`）+ 本次真实改动条目。
- [ ] **禁止** H1 模组标题、流程说明、AI 维护规则 —— 文件内容会**原样公开**到平台页面。
- [ ] 开发期的 `## Unreleased` 段在上传前**改成** `### <version> (<日期>)`。
- [ ] 不累积历史版本（历史在平台页面与 git）。

### 步骤 4：上传（用户实机通过并主动下令后）

**CurseForge**
- [ ] 代理可用；显式 HTTP CONNECT 代理（不要 SOCKS5）。
- [ ] 命令里显式带上前置构建任务，路径参数逐个加引号；`-P` 值含点号时整体加引号。
- [ ] 前置依赖写进 `relations.projects`；无前置时**整个字段省略**。
- [ ] `metadata` 是普通表单字段（不带 `filename=`）。
- [ ] 拿到 **File ID** 才算发布完成，并记录。
- [ ] 回读依赖页确认前置确实出现在页面上。

**Modrinth**
- [ ] fabric / neoforge **分别**建 version（一个 version 只能一个加载器）。
- [ ] `curl.exe -F "data=<payload.json;type=application/json"`（`Invoke-RestMethod -Form` 必 400）。
- [ ] `dependencies` 空数组合法；`environment` 用英文枚举。
- [ ] 拿到 **version id** 并记录。
- [ ] **未建 MR 项目的模组不上传**，也不作为 MR 前置依赖。

### 步骤 5：Git

- [ ] 用户已明确下达推送指令。
- [ ] `.gitignore` 覆盖构建产物；`git ls-files` 计数 buildlike = 0。
- [ ] `git status` 干净、`git log --oneline -5` 基线正确。
- [ ] 强推前 `ls-remote` 比对远端哈希与基线；不一致**停手问用户**。
- [ ] 推送带 `-c credential.interactive=false`。
- [ ] 推送后 `ls-remote` 回读哈希，与本地 HEAD 比对。
- [ ] 回报：分支名 + 提交短哈希 + 提交信息。

---

## 14.5 上线后出问题

### 步骤 1：按 Debug Flow 走

见 [10-troubleshooting.md](10-troubleshooting.md) §10.1（12 条）。

### 步骤 2：落到症状索引

见 [10-troubleshooting.md](10-troubleshooting.md) §10.2。

### 步骤 3：结论写四要素

`现象 → 根因 → 判定手法 → 修复`。**只改根因那一处，不顺手改别的。**

### 步骤 4：修完必须三步重来

- [ ] **重新构建**（改了代码 / 元数据 / 资源都要重构建；`UP-TO-DATE` 时加 `--rerun-tasks`）。
- [ ] **重新部署**（删实例内被替换的旧 jar，MD5 比对）。
- [ ] **重新验收**（按 [08-deploy-qa.md](08-deploy-qa.md) 清单）。

### 步骤 5：回滚的判断

- 除非**有直接证据证明旧实现就是错的**、或用户明确允许，**不要回滚已确认可用的实现**。
- **严禁**回滚到曾经出过问题的写法上（用户实测报错、报"打不开 / 崩溃 / 乱码"的那一版）。
- 改动已验收的行为时，先说明「哪一处是用户验证过可用的」+「本次改动的证据是什么」；证据不足就保持原状、只改其它部分。

---

## 14.6 自检清单（四条工作流共用）

- [ ] 每一步的"退出条件"都满足了才进下一步，没有跳步。
- [ ] 所有判定都来自**带外回读**（`BUILD SUCCESSFUL` / `ls-remote` 哈希 / File ID / MD5），不是管道退出码。
- [ ] 部署前确认无游戏进程；部署后记录 MD5。
- [ ] 汇报里阶段清楚：已构建 / 已部署待验收 / 已验收 / 已上传。
- [ ] 结论与决策写进**项目决策记录**，没有另建"修复记录"目录、没有归档构建日志。
- [ ] 本次涉及的占位符（JDK 路径 / 代理端口 / 实例目录 / 脚本目录）都已按你的实际环境替换，仓库里**没有留下 `<你的…>`**。
