# 05 Mixin 与 API 适配

> 本文件管两件事：Mixin 的声明、注入点与兼容级；跨 MC 版本、跨加载器的 API 差异与桥接写法。
> 上游：`SKILL.md`（结论以它为准）；本文件给字段全集、判定命令、版本差异表与踩坑结论。
> 相邻模块：[03-metadata.md](03-metadata.md)（元数据与 mixin 声明落在哪个文件）、[06-build.md](06-build.md)（插件与档位）、[10-troubleshooting.md](10-troubleshooting.md)（症状级排查）、[11-verdicts.md](11-verdicts.md)（易错结论对照）。

标签约定：**铁律** = 违反会导致构建失败、功能静默失效或已发布物返工；**约定** = 推荐做法，可按项目调整；**实测** = 有一手证据（`javap`、jar 解包、maven 元数据、实机验收）；**推断** = 未实测，不许据此放宽版本声明或发布。

---

## 5.1 Mixin 声明：四条铁律与一处约定

### 5.1.1 配置必须两端各自声明（铁律）

加载器**不会**自动发现 jar 里的 `*.mixins.json`。任何一个加载器端漏了声明，该端**全部** mixin 静默失效——不报错、不进日志、功能凭空消失。

| 加载器 / 版本 | 声明位置 | 写法 |
|---|---|---|
| Fabric（全版本） | `fabric.mod.json` 顶层 | `"mixins": ["yourmod.mixins.json"]` |
| NeoForge 1.20.6+ | `META-INF/neoforge.mods.toml` | `[[mixins]]` + `config = "yourmod.mixins.json"` |
| Forge 1.20.6+ | `META-INF/mods.toml` | `[[mixins]]` + `config = "yourmod.mixins.json"` |
| Forge ≤1.20.4（含 1.20.1 FG6） | **jar manifest** 的 `MixinConfigs` 属性 | `MixinConfigs: yourmod.mixins.json`（多份用逗号分隔） |

```toml
# NeoForge / Forge 1.20.6+
[[mixins]]
config = "yourmod.mixins.json"
```

```json
// fabric.mod.json
"mixins": ["yourmod.mixins.json"]
```

```groovy
// 1.20.1（FG6）：manifest 属性，mods.toml 里没有 [[mixins]]
jar {
    manifest {
        attributes 'MixinConfigs': 'yourmod.mixins.json'
    }
}
```

要点：

- **FG6 时代（Forge ≤1.20.4）根本没有 `[[mixins]]` 字段**，照抄高版本的 TOML 写法不会生效，也不会报错。1.20.1 分支必须走 manifest。
- **MultiLoader 工程最容易漏**：common 里的 mixin 被 fabric / neoforge 两端共用，两端元数据**都要**写一份。只写一端时，另一端的症状是"功能在 A 端正常、在 B 端完全没反应"。
- **判定手法**：① 解包产物 `jar tf` 看两个元数据文件在不在；② 用 `unzip -p <jar> fabric.mod.json` / `unzip -p <jar> META-INF/neoforge.mods.toml` 逐字读；③ 启动日志搜 `Mixing <MixinClass> into <target>`，**一行都没有就是没声明**。
- **写法参考**：任取一个第三方 NeoForge 模组解包读它的 `neoforge.mods.toml`，比查文档快。字段位置、缩进、`[[mixins]]` 与 `[[mods]]` 的层级关系一目了然。

### 5.1.2 `refmap`：只在 SRG 时代需要（铁律）

| 版本线 | 运行时命名 | `mixins.json` 是否必须写 `refmap` | `remap` 取值 |
|---|---|---|---|
| Forge 1.18.2 / 1.19.2 / 1.20.1（FG6） | SRG | **必须**写 `"refmap": "yourmod.refmap.json"` | 保持默认 `true` |
| Forge 1.20.6+ | 官方映射（official） | 不需要（写了也基本无害，但注解处理器会找不到映射） | 对官方映射命名的方法用 `false` |
| Fabric 全线 | intermediary | Loom 自动生成并写入 | 默认 `true` |
| NeoForge 1.20.2+ | 官方映射 | 不需要 | 同 Forge 1.20.6+ 口径 |

```json
// 1.20.1（FG6）的 mixins.json 头部
{
  "required": true,
  "minVersion": "0.8.5",
  "package": "com.example.yourmod.mixin",
  "compatibilityLevel": "JAVA_17",
  "refmap": "yourmod.refmap.json",
  "mixins": ["CommandsMixin"],
  "injectors": { "defaultRequire": 1 }
}
```

漏写 `refmap` 的表现：开发环境正常、**打包后注入点全部匹配不到**（运行时是 SRG 名，注解里写的是 Mojang 名），日志里能看到 `Mixing` 但回调不执行。

### 5.1.3 `remap` 的版本分界（铁律）

- **Forge 1.20.6+**：编译期用 SRG、运行时用官方映射。对官方映射命名的方法做 `@Inject` 时**不加 `remap = false` 会编译报** `Unable to locate obfuscation mapping`。
- **Forge 1.18.2 / 1.19.2**：运行时仍是 SRG，official 命名注入**保持** `remap = true`。
- **1.20.1（FG6）**：`remap` 保持默认 `true`；`false` 是 1.20.6+ 的规则，别提前用。
- 写法示例：

```java
@Inject(method = "performCommand", at = @At("HEAD"), remap = false, cancellable = true)
```

判据：先看目标版本运行时用的是 SRG 还是 official，再看注解处理器有没有挂。**不要按"版本越高越要 false"这种线性直觉判断。**

### 5.1.4 `required` / `defaultRequire`：把静默失效变成可听见的失败（约定）

- `required: true` + `"injectors": {"defaultRequire": 1}`：任一个 `@Inject` / `@WrapOperation` / `@ModifyArg` 找不到目标**立即崩启动**。
- 好处：注入点写错立刻暴露，不会带着"功能没生效"的产物发版。
- 代价：跨版本时一个失效注入点会挡住整条启动路径，需先把全部目标串核完再开。
- **两端方法名不同时的写法**：注解里**同时列出两个**方法名并配 `require = 1`，让两端各自匹配成功一个、另一端不匹配也不算失败。

```java
@Inject(method = {"render(Lnet/minecraft/client/DeltaTracker;Z)V", "render()V"},
        at = @At("HEAD"), require = 1, remap = false)
```

只写一个名字 + `require = 1` 时，另一端的启动会直接失败；不写 `require` 时，另一端静默不注入。两种都比"两端各写一个 mixin 类"更省维护。

### 5.1.5 `compatibilityLevel` 必须匹配加载器自带的 Mixin（铁律，实测）

Forge 自带 Mixin 版本与 `compatibilityLevel` 的合法值强绑定，写高了直接启动崩：

| 目标加载器 | 自带 Mixin | 允许的 `compatibilityLevel` | 典型症状 |
|---|---|---|---|
| Forge ≤1.20.6 | **0.8.5** | 只到 `JAVA_17` | 写 `JAVA_21` → 启动崩（ASM 不支持 major 65）；Fabric 端却正常，**极具迷惑性** |
| Forge ≥1.21.1 | **0.8.7** | `JAVA_21` 合法 | — |
| NeoForge 26.x | sponge-mixin（随 NeoForm 线走） | `JAVA_21` | 与 ASM 版本钉死有关，见 [06-build.md](06-build.md) §6.4.5 |
| Fabric | Loom 拉入的 sponge-mixin | 随 Loom 版本 | 与 Forge 端**不同步**（**推断**：两端构件版本独立解析；未逐版本实测，以解包结果为准） |

- 分支级修法：把 `compatibilityLevel` 提成构建变量，按加载器端分别注入，**不要两端共用一份 `mixins.json` 的头部**。
- 判据：解包加载器自带的 `sponge-mixin` / `mixin` 构件版本，或直接读实例 `<version>.json` 的库清单。
- **推断（未逐版本实测）**：同一工程里 Forge 端与 Fabric 端由各自插件解析 Mixin 构件，版本可以不同步，因此两端 `mixins.json` 的 `compatibilityLevel` 允许取不同值——这不属于"配置漂移"。判定前先分别解包两端自带构件，确认实际版本再定值。

---

## 5.2 注入点校验

### 5.2.1 失效机理：`@At` 目标串是字符串字面量

`@Inject` / `@Redirect` / `@ModifyArg` 的 `method = "xxx"` 只按**方法名**匹配，方法名还在就照常"应用"（日志甚至照打 `Mixing MixinXxx ... into ...`）。而 `@At` 里的 `target = "L<owner>;<name>(<desc>)V"` 是**字符串字面量**：目标类换包、方法签名变了，注入点**匹配不到但不会抛错**——那段回调**永不执行**。

IDE 的重命名、编译器的类型检查都碰不到这个字符串。因此：

- 注入点**必须对真实字节码核定**（`javap -p -s` / `-c`）。
- **禁止照抄其它版本线的方法名**：A 线的方法名在 B 线经常已经改了。
- **禁止照抄另一端的 patched 源码**：Fabric 端的 patched 源码用的是 intermediary 名，NeoForge 端是 official 名，抄过来必然对不上。

### 5.2.2 一手案例（实测）：跨大版本后三处钩子静默失效

- 现象：版本迁移后 Java 侧 import 与类型全部改对、编译通过、游戏能进；但 tooltip 一画就 NPE、渲染阶段回调再也不触发。
- 根因：`com.mojang.blaze3d.systems.GpuSurface` / `…systems.RenderPass` 整体搬到 `com.mojang.renderpearl.*`；`GameRenderer.render(DeltaTracker,boolean)` 改成 `render()`。三处 `@At` 目标串仍是老名字。
- 判定手法：对每个目标串跑 `javap`，核 owner 类是否存在 + 描述符是否逐字一致。`javap` 报"找不到类"或没有该描述符 → 该注入点必然不生效。
- 修复：逐条改写目标串，并把"`@At` / `@Slice` 目标串"提升为**跨大版本必查清单**，与 `@Accessor` / `@Invoker` 的成员名同级对待。

### 5.2.3 判定命令

```bash
# 1) 目标方法是否存在、描述符是什么（-s 打签名，-p 含私有成员）
javap -p -s path/to/net/minecraft/commands/Commands.class | grep -A2 performCommand

# 2) 目标方法内部有没有可注入的调用点（-c 反汇编字节码）
javap -p -c -s path/to/net/minecraft/server/level/ServerPlayer.class | grep -A40 "setGameMode"

# 3) 字段访问点：确认字段名 + 类型（@Accessor 用）
javap -p -s path/to/net/minecraft/world/entity/player/Player.class | grep -i "permission"
```

判定规则：

| 检查项 | 通过条件 | 不通过的含义 |
|---|---|---|
| owner 类存在 | `javap` 能列出该类 | 包路径已变，`@At` 必然失效 |
| 方法名存在 | 输出里有该名字 | `method =` 匹配不到，`require = 1` 时启动崩 |
| 描述符逐字一致 | `-s` 输出的 `descriptor:` 与注解里完全相同 | 重载选错 / 参数列表变了，注入到错的重载或不注入 |
| 返回类型一致 | `)V` / `)I` 与注解一致 | 1.21 起 `performCommand` 由 `)I` 变 `)V`，最常见 |

**重载必须用完整描述符对，不能只对方法名。** 同名方法有多个重载时，只写名字会命中第一个匹配项（或直接失败），且失败是静默的。

### 5.2.4 两类失败的可听性差异（重要）

| 失效对象 | 是否崩启动 | 表现形式 |
|---|---|---|
| `@At` 目标串 | **否** | 静默不执行，"没报错"证明不了注入生效 |
| `method =` 方法名（`require = 1`） | 是 | 启动即炸，报注入失败 |
| `@Accessor` / `@Invoker` 成员名 | 是 | 启动即炸（找不到字段 / 方法） |
| 注入处理器**参数列表**不匹配 | 是 | 启动即炸 |
| `compatibilityLevel` 过高 | 是 | 启动即炸（ASM 版本不支持） |
| 元数据漏声明 | **否** | 全部 mixin 静默失效 |

⇒ 排查顺序：先确认"会叫的失败"没有（启动能过说明声明、兼容级、`@Accessor` 都对），再逐条核"不会叫的失败"（`@At` 目标串）。

### 5.2.5 先排除注释块里的死串（约定）

历史遗留的注释代码里常留着已失效的 `@At` 目标串。全量扫描命中清单里**近半数**落在注释块内——为它们改代码是纯浪费。

判定手法：对每个命中的目标串，回到源文件看它所在行是否被 `//` 或 `/* */` 包住；只处理真实生效的注解。

### 5.2.6 `@Slice` / `ordinal` 的额外风险

- `@At` 带 `ordinal = N` 时，目标方法内部指令顺序一变（编译器优化、代码重构）就整体错位，且同样静默。
- 优先用 `@At("INVOKE")` + 完整 `target` 描述符定位，`ordinal` 只在无法区分时用，并在注释里写明"为什么必须是第 N 个"。
- 一个 mixin 里多个 `@At` 指向同一方法时，用 `@Slice(from = ..., to = ...)` 限定范围，避免跨方法误匹配。

---

## 5.3 版本差异速查表

### 5.3.1 现代线（26.1.2 / 1.21.1 / 1.20.1）

| 事项 | 26.1.2 | 1.21.1 | 1.20.1 | 备注 |
|---|---|---|---|---|
| 构建 Java | 21 / 25 | 21 | 17 | JDK 17 无法 build fabric-loom 1.15.x |
| 加载器 | NeoForge（26.x） | NeoForge + Forge | Forge（jar 亦兼容 NeoForge 1.20.1） | 26.1.2 无独立 Forge，用 NeoForge；1.20.1 的 NeoForge 官方已弃（2024），与 Forge 同源，依赖 modId 仍用 `forge` |
| 入口判断服务端 | 构造函数注入 `Dist`，用 `dist.isClient()` | `FMLLoader.getDist()` | `FMLLoader.getDist()` | 26.x **禁止静态调** `FMLLoader.getDist()` |
| OP 权限判断 | `player.permissions().hasPermission(Permissions.COMMANDS_GAMEMASTER)` | `player.hasPermissions(2)` | `player.hasPermissions(2)` | 真实重构时点见 §5.3.3 |
| `Commands.performCommand` 返回 | `void` | `int` | `int` | 注入点与签名都不同 |
| TOML 元数据文件 | `neoforge.mods.toml` | `neoforge.mods.toml` / `mods.toml` | `mods.toml` | 改名时点见 §5.3.4 |
| Mixin `@Inject` remap | — | 需 `remap = false` | 需 `remap = false`（FG6 实际保持默认 `true`，见 §5.1.3） | Forge 1.20.6+ 运行时用官方映射 |
| datapack `pack_format` | — | 48 | 15 | 取值表见 [03-metadata.md](03-metadata.md) §3.2.2 |

> 注：1.20.1 一列按 FG6 实际行为填（`remap` 默认 `true`、manifest 声明），与 1.20.6+ 的规则不同，不要合并成一行。

### 5.3.2 低版本（1.19.2 / 1.18.2）移植要点

向更早版本移植时，common 核心逻辑（白名单 / 配置 / 权限判断）大体不变，但下列 MC / Forge API 差异必须先适配：

| 事项 | 1.19.2 | 1.18.2 |
|---|---|---|
| 构建 Java / loom / FG | Java 17 · loom 1.8-SNAPSHOT · FG6 `[6.0.24,6.2)` | Java 17 · loom 1.8-SNAPSHOT · FG6 `[6.0.24,6.2)` |
| Forge | 43.4.0（`[43,)`） | 40.2.21（`[40,)`） |
| Fabric / fabric-loader | 0.76.0+1.19.2 / 0.14.24 | 0.51.0+1.18.2 / 0.13.3 |
| `pack_format`（资源 / 数据包） | 9 | 9 |
| 指令注入目标 | `performCommand(ParseResults,String)` | `performCommand(CommandSourceStack,String)` |
| 玩家收消息 | `sendSystemMessage(Component)` | 服务端 `sendMessage(Component, UUID)`、客户端 `displayClientMessage(Component, boolean)` |

- `Component.translatable(...)` 静态工厂是 **1.19.0** 才引入；1.18.x 必须 `new net.minecraft.network.chat.TranslatableComponent("key")`。
- `sendSystemMessage(Component)` 是 **1.19** 改名引入；1.18.x 服务端给玩家发消息用 `player.sendMessage(new TranslatableComponent(...), player.getUUID())`，客户端进世界通知用 `player.displayClientMessage(new TranslatableComponent(...), false)`（第二个 boolean = 是否 actionbar）。
- `Commands.performCommand` 重载不同：`ParseResults` 版本是 **1.19** 才加的；1.18.x 只能注入 `performCommand(Lnet/minecraft/commands/CommandSourceStack;Ljava/lang/String;)I`（返回 `int`），它拿不到已解析节点，需自己从命令字符串解析根指令（跳过前导 `/`、空格后取首个 token）。
- **Forge 客户端登录事件类名不同**：`ClientPlayerNetworkEvent.LoggingIn` 是 1.20 命名；1.18 / 1.19 用 `ClientPlayerNetworkEvent.LoggedInEvent`（`getPlayer()` 返回 `Player`）。
- **Mixin `remap`**：Forge 1.18.2 / 1.19.2 运行时仍用 SRG，official 命名注入保持 `remap = true`。
- **Mixin 版本**：这两条线 Forge 自带 0.8.5，`compatibilityLevel` 只能用 `JAVA_17`。

### 5.3.3 触点的版本迁移（实测）

| 触点 | 形态迁移 |
|---|---|
| **A** `Commands#performCommand` | 1.18.2 `(CommandSourceStack,String)I` → 1.19.2 / 1.20.1 `(ParseResults,String)I` + `remap=true` → **1.21.1 起**同参数但 `)V` 且去掉 `remap` |
| **B** OP 等级判定 | 1.21.1 `player.hasPermissions(2)` → **1.21.9 起**：`ServerPlayer#hasPermissions(int)` **已被移除**，必须 `player.permissions().hasPermission(Permissions.COMMANDS_GAMEMASTER)` |
| **C** 聊天组件 | 1.18.2 用已删除的 `TranslatableComponent` → 1.19+ 用 `Component` |
| — | `ServerPlayer#setGameMode(Lnet/minecraft/world/level/GameType;)Z` **1.18.2 → 26.3 一次都没变过** |

> ⚠ **权限体系重构的真实时点是 1.21.9**，不是 1.21.2。早期指南与方案稿曾按"1.21.2 起就是新 API"推测，属**推断**，已被两侧实证推翻。判据：用 `1.21.2~1.21.5`（旧 API）与 `1.21.9~1.21.11`（新 API）两侧各取一个版本编译验证。**形态边界必须两侧实证，不能按"系列感"推测。**

### 5.3.4 元数据文件名改名时点（铁律）

NeoForge 文档在 **1.20.4 仍是 `mods.toml`**，到 **1.20.6 才见 `META-INF/neoforge.mods.toml`**。⇒ 覆盖 1.20.2~1.20.6 的分支要用**旧文件名**，不能照抄 26.x 写法。

| 版本区间 | TOML 文件名 | `[[mixins]]` 可用 |
|---|---|---|
| ≤ 1.20.4（Forge / FG6） | `META-INF/mods.toml` | 否（走 manifest `MixinConfigs`） |
| 1.20.6（Forge / NeoForge） | `META-INF/mods.toml` | 是 |
| ≥ 1.21.1（NeoForge） | `META-INF/neoforge.mods.toml` | 是 |
| ≥ 1.21.1（Forge） | `META-INF/mods.toml` | 是 |

文件名写错的症状：加载器**完全不读**该元数据，表现为"模组列表里没有本模组"或"依赖声明不生效"。字段全集见 [03-metadata.md](03-metadata.md) §3.1。

---

## 5.4 NeoForge 26.x 的 API 变更（编译 / 运行时实测）

| 旧用法（报错） | 新用法（可编译） |
|---|---|
| `FMLLoader.isProduction()`（非静态） | `FMLEnvironment.isProduction()` |
| `CommandSourceStack.hasPermission(int)`（**不存在**） | 权限统一走 NeoForge `PermissionAPI.getPermission(player, node)` |
| OP 判断 `player.getServer()`（私有） | `player.level().getServer().getPlayerList().isOp(...)` |
| `isOp(GameProfile)` | `isOp(NameAndId)` —— 参数类型已从 `GameProfile` 改为 `NameAndId`（`new NameAndId(profile)`） |
| 遍历 `ModFileInfo` / `IModFile` 扫资源（已移除） | 走 FML 文件视图 `ModList…getContents().visitContent(...)`（见 [03-metadata.md](03-metadata.md) §3.2.1）。曾记"用类加载器 `getResources("assets")`"，**已被实测推翻** |
| `LootTable.Builder`（NeoForge 无） | `LootTable.addPool(LootPool)`，`LootPool.Builder.build()` 无参返回 `LootPool`；桥接 `pool -> table.addPool(pool.build())` |
| 入口判断服务端 | 26.x **禁止静态调** `FMLLoader.getDist()`；用构造函数注入 `Dist`，`dist.isClient()` |
| 服务端数据包重载监听器 | `PreparableReloadListener` / `PreparationBarrier` 签名已变；无法匹配时可暂空实现（代价：对应缓存 `/reload` 不刷新，需重启），待 API 稳定后补全 |

其他 26.x 相关：

- `net.neoforged.fml.ModFileInfo` 已移除（`IModFile` / `IModFileInfo` / `JarContents` 仍在）。
- `MixinExtras 0.5.x`（见 §5.8）。
- 静态工具类改为实例注入是 26.x 的整体方向：凡是"全局静态拿环境"的写法，都要换成"从构造函数 / 事件参数拿"。

---

## 5.5 事件总线：EventBus 6 → 7 换代（实测，1.21.8 起）

### 5.5.1 症状与错误原文

`MinecraftForge.EVENT_BUS.register(SomeClass.class)` 在 EventBus 7 下**只允许类里有多于一个监听方法**；单监听类直接崩：

```
Only a single listener found in class ... You should directly call addListener() on the EventBus of LoggingIn
```

### 5.5.2 正确写法

```java
// 用事件类自带的静态总线；同时去掉 @SubscribeEvent
ClientPlayerNetworkEvent.LoggingIn.BUS.addListener(MyClient::onLogin);
```

- 每个事件类在 EventBus 7 下都带自己的 `BUS` 字段，注册粒度从"类"变成"监听方法"。
- `@SubscribeEvent` 在这种写法下不再需要；保留它只会让人误以为还在走旧机制。
- NeoForge 侧对应写法是 `NeoForge.EVENT_BUS.addListener(EventClass.class, event -> ...)`，与 Forge 的 `BUS` 写法不同源，两端各写各的。

### 5.5.3 判据：查实例/版本 JSON 的 `eventbus` 版本

| `eventbus` 版本 | 覆盖版本 | 写法 |
|---|---|---|
| **6.x** | 1.20.4 = 6.2.33、1.20.6 = 6.2.27、1.21.1 = 6.2.27、1.21.5 = 6.2.32 | 旧写法（`register(Class)` + `@SubscribeEvent`）可用 |
| **7.x** | 1.21.8 = 7.0、1.21.11 = 7.0.5、26.x = 7.0.5–7.0.6 | **必须**改成 `BUS.addListener(...)` |

判定手法：读实例或版本目录下 `<version>.json` 的库清单，找 `net.minecraftforge:eventbus` 的版本号。**不要按 MC 版本号猜**——同一条线上 eventbus 会单独升版。

### 5.5.4 迁移检查表

- [ ] 全仓搜 `EVENT_BUS.register(`，逐个判断是否落在 7.x 线。
- [ ] 单监听类一律改成 `BUS.addListener`。
- [ ] 多监听类可保留 `register(Class)`，但要确认没有混入"只有 1 个方法"的类。
- [ ] 去掉 `@SubscribeEvent` 后，确认方法签名与事件类型仍匹配（`addListener` 是强类型的）。
- [ ] 客户端与服务端各查一遍：两侧 eventbus 版本可能不同。

---

## 5.6 跨加载器事件桥接（Fabric API ↔ NeoForge / Forge）

把依赖 Fabric API 的代码搬到 common + 双加载器的 MultiLoader 工程时，按下列等价事件桥接，无需在 common 依赖 Fabric 运行时。**每个桥接只需在两个 loader 模块各写一个"监听器适配器"，统一转成 common 内部事件或回调再派发。**

| 能力 | Fabric API | NeoForge 等价（实测） |
|---|---|---|
| 服务器启动 | `ServerLifecycleEvents.SERVER_STARTED` | `ServerStartedEvent` |
| 服务器停止 | `ServerLifecycleEvents.SERVER_STOPPING` | `ServerStoppingEvent` |
| 服务端 tick | `ServerTickEvents.END_SERVER_TICK` | `ServerTickEvent.Post` |
| 指令注册 | `CommandRegistrationCallback` | `RegisterCommandsEvent` |
| 玩家加入 / 离开 | `ServerPlayerEvents.JOIN` / `LEAVE` | `PlayerEvent.PlayerLoggedInEvent` / `PlayerLoggedOutEvent` |
| 实体受伤后 | `ServerLivingEntityEvents.AFTER_DAMAGE` | `LivingDamageEvent.Post`（用 `getBlockedDamage()>0` 近似"完全格挡"） |
| 实体死亡后 | `ServerLivingEntityEvents.AFTER_DEATH` | `LivingDeathEvent` |
| 方块破坏后 | `PlayerBlockBreakEvents.AFTER` | `BlockDropsEvent`（`getLevel()` / `getPos()` / `getState()` / `getBreaker()`） |
| 使用物品 | `UseItemCallback` | `PlayerInteractEvent.RightClickItem` |
| 使用方块 | `UseBlockCallback` | `PlayerInteractEvent.RightClickBlock`（`getHitVec()`） |
| 使用实体 | `UseEntityCallback` | `PlayerInteractEvent.EntityInteractSpecific`（`getTarget()`） |
| 服务端数据包重载 | `SynchronousResourceReloadListener` | `AddServerReloadListenersEvent` + `PreparableReloadListener`（26.x 签名已变） |
| 自定义事件总线 | `Event` / `EventFactory`（内部事件容器） | 无需映射——common 自己定义即可（只在 Fabric 端借用容器） |

调用面收敛为三类：**事件生命周期**、**内部自定义事件**（借 Fabric 容器，等价于 common 自建）、**权限 / loader / 资源**。

后两者用 ServiceLoader 服务接口抽象，业务层 0 感知：

| 接口 | 职责 | 两端实现 |
|---|---|---|
| `IPlatformHelper` | 加载器判定、环境判定、平台特有调用 | fabric / neoforge 各一份 |
| `IPermissionService` | OP / 权限节点判定 | 按版本分支实现，见 §5.3.3 触点 B |
| `IModAssetScanner` | 扫描 jar 内资源 | 走 FML 文件视图 / Fabric 的资源 API |
| `IEventBridge` | 把平台事件转成 common 回调 | 两端各一个监听器适配器 |

结果：**业务代码零 `net.fabricmc` / `net.neoforged` 引用**，平台差异锁在一个薄接口后面。

---

## 5.7 可选前置库的通用模式（反射门面 + 声明可选 + 不打包）

场景：模组可以和一个第三方库联动（提供额外配置界面、网络通道、联动功能），但**不强制依赖**它；未安装时作为独立模组正常运行（配置驱动）。

四条做法：

1. **探测门面**：写一个极薄的 `OptionalDepLink`，只做存在性判断。

   ```java
   public final class OptionalDepLink {
       private static final boolean PRESENT = detect();
       private static boolean detect() {
           try {
               Class.forName("com.example.optlib.Constants", false, OptionalDepLink.class.getClassLoader());
               return true;
           } catch (ClassNotFoundException e) {
               return false;
           }
       }
       public static boolean isLoaded() { return PRESENT; }
   }
   ```

   用 `Class.forName(name, false, loader)`（**不初始化**），避免触发对方的静态初始化块。

2. **禁止在关键类字节码中静态引用可选前置的类型（铁律）**：任何在"无该前置"环境下会被加载的类，其常量池里不能出现 `com.example.optlib.*`。否则 `NoClassDefFoundError`，运行期守卫（`if (isLoaded())`）**无效**——类加载发生在方法调用之前。
   需要走反射的地方通常是：网络注册、`sendToServer` / `sendToAll` / `sendToPlayer`、打开对方提供的配置界面子类。

3. **客户端注册**：Forge 端不要用 `@Mod.EventBusSubscriber` 注解，也不要 `EVENT_BUS.register(Class)`（都会触发 ASM 扫描并加载该类）；改用 `EVENT_BUS.addListener(方法引用)`。EventBus 7 下这本来就是唯一合法写法（见 §5.5）。

4. **依赖声明**：
   - Forge `mods.toml`：可选前置写 `mandatory = false`。
   - Fabric `fabric.mod.json`：**不要**写进 `depends`，改用 `suggests`。
   - 编译期依赖保留在 `build.gradle`（`compileOnly` 或 `modCompileOnly`），但**不要 `include` 打包对方类**。

```toml
# mods.toml：可选前置
[[dependencies.yourmod]]
    modId = "optlib"
    mandatory = false
    versionRange = "[1.0,)"
    ordering = "AFTER"
    side = "BOTH"
```

```json
// fabric.mod.json
"suggests": { "optlib": "*" }
```

> 通用结论：**可选前置一律走"反射门面 + 声明为可选 + 不打包"**，任何平台分支里都不让关键类常量池硬引用可选前置的类型。

判定手法（验收用）：在**未安装前置**的实例里启动一次，看是否 `NoClassDefFoundError`；再装上前置启动一次，看联动功能是否出现。两次都过才算落地。

---

## 5.8 MixinExtras 与注解处理器

| MC 线 | MixinExtras | 说明 |
|---|---|---|
| 26.2 | `0.5.3` | `@WrapOperation` / `@ModifyExpressionValue` 等注解由它提供 |
| 26.3 | `0.5.4` | 同上，随线升版 |

- MixinExtras 版本随 MC 线走，升版本时与 Loom / MDG 一起逐版本核对，不要跨线复用同一个版本号。
- **FG7 + official 映射下不要挂 mixin 注解处理器**（`annotationProcessor 'org.spongepowered:mixin:...:processor'`）：它会去找混淆映射，直接报 `Unable to locate obfuscation mapping for @Inject target ...`。注解类用 `compileOnly 'org.spongepowered:mixin:0.8.7'` 提供即可（细节见 [06-build.md](06-build.md) §6.2.3）。
- 需要 `@WrapOperation` 时，MixinExtras 必须真在 classpath 上；只写注解不引依赖 → 编译期 `cannot find symbol`，运行期 `ClassNotFoundException`。

---

## 5.9 编码与协作习惯

- **源码注释避免中文**：若 `javac` 未强制 UTF-8 而按平台默认（GBK）读取，UTF-8 源码里的中文注释会报 `编码 GBK 的不可映射字符`。更稳的做法是补上 `options.encoding = 'UTF-8'`（见 [06-build.md](06-build.md) §6.5）。
- **库代码注释用英文、业务代码可用中文**是常见分工；跨语言协作时统一英文更省事。
- **改动多个互相关联的 `@Inject` / 相同 API 时，注意同一次编辑在若干文件间保持 import 完整**：并行编辑同一文件出现过 import 丢失，症状是"另一个端编译报符号找不到"。
- **版本差异先查模板仓库对应分支**（MultiLoader-Template 按 MC 版本分分支）；元数据、License、README **全版本同步修改**。
- 不擅自改动未被要求的配置；改一处先分析根因再动手。

---

## 5.10 自检清单

- [ ] mixin 配置**两端成对声明**（Fabric `"mixins"` / NeoForge `[[mixins]]` / FG6 manifest `MixinConfigs`）。
- [ ] 1.20.1 分支的 `mixins.json` 写了 `refmap`，且 `remap` 保持默认 `true`。
- [ ] `required` / `defaultRequire` 与风险匹配；跨版本双方法名时用 `method = {a, b}` + `require = 1`。
- [ ] `compatibilityLevel` 与目标加载器自带 Mixin 版本匹配（Forge ≤1.20.6 = `JAVA_17`；≥1.21.1 = `JAVA_21`）。
- [ ] 每个 `@At` / `@Slice` 目标串已用 `javap -p -c -s` 核对 owner + 完整描述符；注释内的死串已排除。
- [ ] 重载方法用完整描述符定位，不用裸方法名；`ordinal` 有注释说明必要性。
- [ ] `remap` 取值与目标版本匹配（1.20.6+ Forge 用 `false`；1.18 / 1.19 / 1.20.1 用 `true`）。
- [ ] 权限判定按目标版本选（1.21.9+ 用 `permissions().hasPermission(...)`）。
- [ ] 玩家消息发送签名按版本选（1.18.x 双参 `sendMessage(Component, UUID)`）。
- [ ] EventBus 版本已核实（6.x 沿用旧写法，7.x 改 `BUS.addListener`）。
- [ ] 可选前置全部走反射门面，关键类字节码无 `com.example.optlib.*` 之类硬引用；未装前置的实例能正常启动。
- [ ] common 模块不直接 import 平台专属包；平台差异收敛在 `IPlatformHelper` 一类接口后。
- [ ] 26.x 没有静态调用 `FMLLoader.getDist()`，没有用 `ModFileInfo` 扫资源。
- [ ] 编译配置已设 `options.encoding = 'UTF-8'`，跨端注释不出现"编码不可映射"报错。
