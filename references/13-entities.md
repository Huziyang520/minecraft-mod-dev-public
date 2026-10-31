# 13 生物、动画与 GeckoLib

> 管什么：添加生物（原版组合体 / 原版换皮 / 全新生物）时的分类判定、注册链、渲染与判定箱对齐、AI 与 Goal、GeckoLib 5 资源路径、配置边界、货物生命周期、per-bone 渲染层、Mixin 注入器选择。
> 本文件与 `references/` 其余模块同属一个技能包，**没有上位总纲**；结论冲突时以一手证据为准。本文件给细节、推导、数值与踩坑。相关：[12-art.md](12-art.md)（模型与贴图规范）、[05-mixin-api.md](05-mixin-api.md) §5.1（注入点核定与 remap）、[03-metadata.md](03-metadata.md)（资源扫描与注册表声明）、[10-troubleshooting.md](10-troubleshooting.md)（渲染/黑屏类症状索引）。

标签：**铁律**（违反即返工）· **实测**（有一手证据）· **推断**（未实测，须先探针）。

## 13.1 生物分类铁律：先定性，再动手

**铁律**：添加任何生物前，必须先归入下表之一，再按对应栏位实现。**禁止跳过定性直接写类**。

| 类别 | 定义 | 正确做法 | 禁止 |
|---|---|---|---|
| **原版组合体** | 外观 100% 原版，无新模型；原版 A 驮/吊原版 B | Mixin 注入原版实体行为；实体与货物都是原版实例 | 注册新 `EntityType`；写新的实体基类；给它配 loot table / attribute / 渲染器 |
| **原版换皮 / 改行为** | 模型原版，行为不同 | 同上：Mixin + 原版实体 | 为"换个 AI"复制一份原版实体类 |
| **全新生物** | 有专属模型 / 动画 | 注册 `EntityType` + 属性 + 渲染器 + loot table，走完整注册链（§13.3） | 用"占位实体 + 硬编码贴图"混进正式版 |

**判定口诀**：杀掉这只生物后，玩家看到的每一块肉是否都能来自原版实体？能 → 原版组合体，不许注册新类型。

这条口诀的价值在于它把"外观"和"注册表"绑在一起：只要掉落物、贴图、模型都来自原版，它就不该出现在注册表里；一旦注册了新 `EntityType`，就自动获得"必须配齐属性 / 渲染器 / 战利品表 / 刷怪蛋"的全套义务（§13.3），而它的外观又根本不需要这些。

## 13.2 原版组合体的实现模式

以一个"幻翼 + 苦力怕"的组合体为样板（宿主 = 原版幻翼 `Phantom`，货物 = 原版苦力怕 `Creeper`），两种变体：

- **变体 A（驮载型）**：苦力怕被驮在幻翼背上。
- **变体 B（投掷型）**：苦力怕被吊在幻翼腹下，可被投出。

实现要点：

1. **宿主用原版工厂产出**：`EntityTypes.PHANTOM.create(level, ...)` 之类的原版 `EntityType` 工厂，**绝不 `new` 自定义类**。自定义子类会让原版 AI、原版渲染器、原版数据包引用全部失配。
2. **行为注入挂宿主原版类**：Mixin 目标写原版实体类（如 `PhantomMixin` 注入 `Phantom`），变体标记存 `@Unique` 字段，并随 `addAdditionalSaveData` / `readAdditionalSaveData` 持久化。26.3 的签名换成了 `ValueOutput` / `ValueInput`，字段读写要按该版本 API 改写（详见 [05-mixin-api.md](05-mixin-api.md) §5.4）。
3. **变体产生收敛到单一静态入口**：刷怪蛋、自然生成、命令、spawner 都调同一个方法，禁止各处自行 `create` + 打标。分散打标是"同一只生物带两个变体标记""标记漏打退回普通行为"的常见来源。
4. **货物是独立原版实体**：需要非标准挂点（吊在下方、负偏移）时**不用 passenger 机制**，由宿主 tick 每帧重新锚定（`snapTo`）。这样货物天然可被单独击杀、自带战利品、能被其它系统正常识别；passenger 机制会把货物锁进载具的乘坐槽，玩家打不到、AI 不跑、掉落物也不归它。
5. **AI 注入**：Goal 挂优先级 0 抢占原版私有 Goal；访问 `protected` 的 `goalSelector` / `targetSelector` 用 `@Accessor` mixin；写原版私有移动字段（如 `PhantomMoveControl#moveTargetPoint`）也用 `@Accessor` mixin。**禁止**用 `MoveControl#setWantedPosition` 碰运气——幻翼系会静默忽略，表现为"设置了目标点但生物不动"。

## 13.3 全新生物的注册链七项

**铁律**：全新生物必须一次配齐，缺一即视为未完成。

- [ ] **`EntityType.Builder`**：尺寸 / `eyeHeight` / `passengerAttachments` / `clientTrackingRange` / `notInPeaceful` 按原版同类镜像取值。
- [ ] **两端注册**：Fabric 用 `Registry.register`，NeoForge 用 `DeferredRegister`；common 层只认 `EntityRegistrar` 接口，不直接碰加载器 API。
- [ ] **`AttributeSupplier` 显式注册**：26.3 的自定义实体不在 vanilla `DefaultAttributes` 里，漏了首次生成即崩（`IllegalArgumentException: ... not registered`）。
- [ ] **客户端渲染器注册**：两端各自入口（Fabric `EntityRendererRegistry` / NeoForge `EntityRenderersEvent.RegisterRenderers`）。
- [ ] **刷怪蛋 + 语言键 + 战利品表**：26.3 刷怪蛋数据驱动，`new Item.Properties().spawnEgg(type)`；语言键与战利品表路径按 §13.3.1 的落点表。
- [ ] **生物名双语**：`zh_cn` / `en_us` 两条都要有，键与实体注册 id 一致。
- [ ] **自然生成权重**（生物群系 / 生成配置），改配置即生效（§13.7）。

原版组合体**恰好相反**：以上七项一律不许出现。刷怪蛋可以用自定义 `Item` 类生成"原版实体 + 变体标记"，但绝不引用不存在的 `EntityType`——引用不存在的类型会在注册表校验阶段直接抛错，或在客户端生成时拿到 null。

七项里最容易漏的是 `AttributeSupplier`：它不在编译期报错，只在**首次生成**时崩，且崩溃发生在服务端 tick，日志里看不到注册链的上下文。

### 13.3.1 注册链的落点与顺序

七项在工程里各有固定落点，顺序错了会得到"注册了但没生效"：

| 项 | 落点 | 顺序约束 |
|---|---|---|
| `EntityType` | common 的 `EntityRegistrar` 实现 | 必须在属性与渲染器之前 |
| `AttributeSupplier` | 各加载器入口的 `FabricDefaultAttributeRegistry#register` / `EntityAttributeCreationEvent` | 必须在 `EntityType` 注册之后、任何生成之前 |
| 渲染器 | 客户端入口（`EntityRendererRegistry` / `RegisterRenderers`） | 只在客户端；common 不引用 |
| 刷怪蛋 | 物品注册（`DeferredRegister.Items` / `Registry.register`） | 引用 `EntityType`，故在之后 |
| 战利品表 | 数据包 JSON（`data/<ns>/loot_table/entities/<id>.json`） | 与代码无关，缺了只是不掉落 |
| 语言键 | `assets/<ns>/lang/*.json` | 与代码无关，缺了显示 `entity.<ns>.<id>` |
| 生成权重 | 生物群系生成配置（数据包或代码） | 需要 `EntityType` 已注册 |

- **common 只认 `EntityRegistrar` 接口**：加载器 API 不进 common，否则另一端编译时会把另一端的类拖进来。
- **属性注册必须在两端都做**：Fabric 与 NeoForge 的入口不同，只做一端会出现"一端正常、另一端首刷崩"。
- **刷怪蛋图标**：26.3 数据驱动刷怪蛋会自动取实体渲染，不需要单独画图标；自定义 `Item` 生成变体时才要自己的贴图。
- **战利品表路径与注册 id 一致**：`<ns>:<path>` 对应 `data/<ns>/loot_table/entities/<path>.json`；路径写错不报错，只是掉落为空。

## 13.4 渲染与判定箱对齐

"模型看着悬空 / 穿地 / 巨大，碰撞箱在别处"是生物类模组最高频的问题。

### 13.4.1 三件套必须一致

**铁律**：模型在 Blockbench 里的离地高度、`EntityType#sized()` 的判定箱、渲染时的 `translate` / `scale`，三者必须一致。改其中任何一个，必须同步核对另外两个，并实机截图验收。

### 13.4.2 先算清整条渲染管线的位移，再动手

26.3 的 `LivingEntityRenderer#submit` 顺序（简化）：

```
poseStack.scale(state.scale, ...)
setupRotations(...)
poseStack.scale(-1, -1, 1)
this.scale(state, poseStack)          // 各实体渲染器在这里做自己的位移
poseStack.translate(0, -1.501, 0)     // ← 之后还有这一下
```

推导：`scale(-1, -1, 1)` 把 Y 轴取反，所以 `this.scale` 里写 `translate(0, +L, 0)` 实际是**向下**移动 `L`；随后 `translate(0, -1.501, 0)` 又把它抬起 `1.501`。净效果：模型整体抬高 `1.501 - L`。

以幻翼为例（原版数值，**实测**）：

| 量 | 值 |
|---|---|
| 原版 `this.scale` 里的 `L` | `1.3125` |
| 净抬高 `1.501 - L` | `0.1885` |
| 模型区间（相对脚底） | `+0.0635 ~ +0.3135` |
| 判定箱 `sized(0.9, 0.5)` | `0 ~ 0.5` |

结论：模型区间 `0.0635 ~ 0.3135` **正好落在 0.9 × 0.5 的判定箱（0 ~ 0.5）内——原版本来就是对齐的**。

**铁律**：任何"模型浮在判定箱上方"的结论，必须先按上式算出真实落点再下判断；**禁止看到源码里有个 1.3125 就认定它错**。

反面案例（**实测**）：把 `L` 从 `1.3125` 改成 `0.28`，净抬高变成 `1.501 - 0.28 = 1.221`，模型区间升到 `+1.10 ~ +1.35`，**高出判定箱 1.2 格**；后续所有排查都建立在"原版没对齐"这个错误前提上，越查越偏。这就是"先算再改"的反面教材：错的前提会自我强化。

### 13.4.3 偏移只走渲染器或脚本参数

**铁律**：禁止为了对齐手改 `*.geo.json` 的骨骼数据；偏移一律走渲染器 `translate` 或转换脚本参数（如 `-ModelYShiftPx`）。理由见 §13.4.5：模型资产是美术产物，改它会让美术与代码两边的数据源分叉。

### 13.4.4 挂载偏移写成"相对模型区间"的推导

**铁律**：挂载偏移必须写成推导过程，不许只给一个孤零零的数字。

例：模型区间 `+0.0635 ~ +0.3135` ⇒ 骑乘者脚底取 `0.32`（取区间上沿附近，留一点余量避免穿插）。写成代码常量时也保持相对表达：

```java
static final double MODEL_BAND_BOTTOM = 0.0635;
static final double MODEL_BAND_TOP    = 0.3135;
static final double RIDER_FOOT_Y      = MODEL_BAND_TOP + 0.0065;  // 0.32
```

模型区间一变，所有挂载点全部失配——所以挂载点必须是**区间的函数**，不是常量表里抄来的数。

### 13.4.5 定值验证与资产边界

**铁律**：改渲染偏移前先做定值验证：先原样跑一遍，按 F3+B 与日志数值确认"当前到底偏多少"，再改；**禁止"改完看不出效果就再改一版"**。

**铁律**：纹理 / 模型文件是美术资产，任何代码改动不得触碰 `*.bbmodel`、`*.geo.json`、`*.png`、`*.animation.json`（转换脚本产出的除外，且必须由脚本重新生成）。**验收时模型对不齐先怀疑代码，不怀疑资产。**

每个生物实机验收时开启 F3+B 看碰撞箱，截图存档。

### 13.4.6 判定箱相关字段的取值口径

`EntityType.Builder` 上与对齐直接相关的字段共四个，取值口径各不相同：

| 字段 | 含义 | 取值口径 |
|---|---|---|
| `sized(width, height)` | 碰撞箱宽 / 高，从脚底向上量 | 按**模型实际占位**取，不按视觉大小；宽取模型最宽处，高取模型最高处（含抬起后的净位移） |
| `eyeHeight` | 眼睛高度，用于视线检测与骑乘对齐 | 取模型头部中心的高度，**必须 ≤ 判定箱高度**，否则会出现"看不见却打得到"的判定错位 |
| `passengerAttachments` | 乘客挂点（相对实体原点） | 取**模型区间上沿**推导（§13.4.4），不要抄原版数值——模型区间不同 |
| `clientTrackingRange` | 客户端追踪距离（区块数） | 按生物可见性需求取；与判定箱无关，但调太小会让远处模型"突然出现" |

`eyeHeight` 超高的典型症状：玩家瞄准头部打不中、生物"看不见玩家"却会追击。`sized` 与模型不一致的典型症状：模型穿地（判定箱比模型小）、或玩家站在半空（判定箱比模型大）。

判定手法：F3+B 打开碰撞箱后，从正侧方看"箱底是否贴地、箱顶是否包住模型最高点"；再从上方俯视看"箱宽是否覆盖翼展"。两个视角都过，才算对齐。

## 13.5 AI / Goal 规范

1. **目标选择与行为分离**：索敌 Goal 只写 `target`，`canUse` 永远 `return false`（样板名如 `PlayerTargetGoal`）；行为 Goal 读 `getTarget()`。把索敌逻辑写进行为 Goal 会让"抢到目标"和"做动作"绑死，无法独立调优先级。
2. **优先级**：需要抢占总控行为的 Goal 挂优先级 `0`，并正确声明 `Goal.Flag`（`MOVE` / `LOOK` / `TARGET`）。**不声明 flag 就等于不抢占**——flag 是原版 `GoalSelector` 做冲突检测的唯一依据，优先级数字只决定同 flag 内的先后。
3. **冷却计时放实体 tick 或 mixin tick，不放 Goal 字段**。Goal 会在失去目标时停止 tick，冷却随之冻结——"必须先杀掉吊着的苦力怕才会投掷"这类怪象多源于此：投掷 Goal 的冷却只在有目标时走，目标一丢冷却就停在半途。
4. **第一次 tick 兜底**：`finalizeSpawn` 会被 `/summon` 等路径跳过，一切"生成时初始化"逻辑必须放在首个服务端 tick 里做一次（样板：货物挂载、变体掷骰）。用 `@Unique` 的 `boolean initialized` 守卫，且该字段不参与存档（重进存档时首 tick 仍需再跑一次挂载）。
5. **飞行生物控制**：幻翼系只能通过 `moveTargetPoint`（accessor mixin）驱动；一般飞行生物用 `FlyingMoveControl` + `travelFlying`。混用会让移动控制静默失效（见 §13.2 第 5 条）。

## 13.6 GeckoLib 5 资源路径

1. **资源三件套固定路径**：

```
assets/<ns>/geckolib/models/*.geo.json
assets/<ns>/geckolib/animations/*.animation.json
assets/<ns>/textures/entity/*.png
```

26.3 GeckoLib 5 的模型 / 动画 id 是**裸名**：不带 `geckolib/` 前缀、不带 `.geo` 后缀。写全路径的旧写法在新版本里会找不到资源（静默回退成缺省模型）。

2. **`GeoModel` 子类把三个 id 固化为常量**，禁止每帧构造 `Identifier`。每帧 new 一个 `Identifier` 会持续产生短命对象，且在渲染热路径上被调用数千次；id 是常量，没有理由每帧拼。
3. **新动画由美术在 Blockbench 完成 → 转换脚本产出 → 代码只引用**；代码侧**禁止**临时改动画 json（改了就与美术源文件分叉，下次重导必丢）。
4. **模型高度对齐 hitbox 用转换脚本的 YShift 参数**，不用渲染器里塞魔法数（见 §13.4）。

### 13.6.1 骨骼 / UV 判定必须来自数据，禁止目测

**铁律**：确定"某个部位对应哪些骨骼"时，必须**解析 `.geo.json`**（脚本输出每个骨骼的 cube 的 `size` / `origin` / 每个面的 `uv` + `uv_size`），再下结论。

禁止的三件事：

- **凭截图猜**：翅膀 / 身体 / 尾巴在俯视角度极像，正交视角下更是无法区分。
- **只看 cube 的**第一个 UV 数字就断定贴图区域。实例：某模型把 `body` 误判为"苦力怕躯干"，实际它是 UV `v = 64` 的**幻翼身体**；只看了 `u` 值就下结论，方向完全错。
- **相信骨骼名**：`head` 可能是苦力怕头，也可能是幻翼头；某个模型里根本没有 `creeper_head` 这个骨骼。骨骼名是美术起的，不是语义标签。

判定贴图区域的三条硬依据：

1. cube 的 UV 是否落在原版对应贴图的布局里（如原版苦力怕头 = `(8,8)` / `(0,8)` / `(24,8)` / `(16,8)` / `(8,0)` / `(16,0)`）。
2. `v` 值落在贴图的哪一半。以 128×128 画布为例：`v < 64` 为苦力怕区，`v >= 64` 为幻翼区。
3. cube 尺寸是否符合该部位的解剖特征（苦力怕头 8×8×8，躯干应为 8×12×4 一类）。

三条同时成立才下结论；只满足一条就下结论，是误判的主要来源。

## 13.7 配置规范

1. **所有可调行为参数进配置类**（速度 / 高度 / 冷却 / 偏移 / 开关 / 权重），**禁止**硬编码在 Goal 里；爆炸半径等"手感参数"也不例外。硬编码在 Goal 里的参数改一次要重编译，且分散在多处，无法一次性核对。

2. **几何推导值禁止进配置**。这是与第 1 条并列的分界线：

| 类别 | 归哪 | 例子 |
|---|---|---|
| 手感参数 | 配置文件 | 速度、冷却、半径、权重、开关 |
| 几何推导值 | 代码常量（相对模型区间的表达式） | 挂载高度、抬升量、锚点偏移 |

理由：几何值只在模型或渲染管线变化时该变，而配置文件不会跟着变——旧值会静默失配，且改模型的人根本不知道还有几个数字在等他改。几何常量必须写成相对表达式（如 `RIDER_CARRY_HEIGHT = MODEL_BAND_TOP`），并把推导过程写在注释里。

3. **配置改了要在下一次生成 / 下一次尝试时生效**（spawner 每次重建候选表），不要求热更已生成实体。热更已生成实体属于额外能力，不做也不算缺陷；但不重建候选表会让"改了配置没反应"。
4. **位置类配置**（模型抬升、挂载偏移）调整后必须实机截图对比验收。

## 13.8 原版组合体的"货物"生命周期

**背景**：货物 = 被驮 / 被吊的原版生物。这一节解决四类问题：重复挂载、残渣、幽灵生物、玩家"杀了一只另一只凭空消失"。

### 13.8.1 归属必须持久化

**铁律**：禁止只用一个字段记住货物。字段不存档，区块卸载 / 重进存档后宿主会"忘记"自己带着货，于是**再生一只**，旧的那只无 AI 生物就永久杵在原地——这就是"同时挂着两只苦力怕"的真实成因。

做法：货物生成时打上归属标签 `<modid>_cargo:<宿主 UUID>`。26.3 用 `Entity#addTag` / `entityTags()` / `removeTag`，标签随存档持久化。宿主每次取货先按标签**认领**（`getEntitiesOfClass` + 半径过滤），认领不到才新建。

用标签而不是字段的三个好处：跨区块卸载存活、可被其它系统（命令、调试工具）观测、不需要额外的存档 codec。

**判定手法**：宿主 tick 里打印"本次认领到的货物数"，正常情况下应为 0 或 1；出现 2 就是归属没持久化（旧货没被认领到，新货又被生成）。这个计数是唯一能在不重进存档的情况下发现重复挂载的证据。

### 13.8.2 同一时间只能有一只货物

认领优先、生成兜底（如 `adoptOrSpawn`）。释放时必须**摘掉标签**，否则散落出去的苦力怕会被宿主再次认领——表现为"投出去的苦力怕又回来了"。

### 13.8.3 补充策略按变体分别定义

必须在代码注释里写明（两种变体语义不同，不写清就会被"统一处理"掉）：

| 变体 | 货物丢失（被杀 / 丢失）后 | 投出 / 引爆后 |
|---|---|---|
| 驮载型 | **不补充**，宿主退回原版 AI | 随宿主自爆一起消失 |
| 投掷型 | **补充一只**，继续投掷 | 投出瞬间立刻补一只（始终看起来"满弹"） |

### 13.8.4 宿主死亡 = 释放，不是删除

**铁律**：宿主死亡（非自爆）时，货物必须 `release`：摘标签 + `setNoAi(false)` + `setNoGravity(false)`，让它以**原版 AI** 继续存在。

**禁止**在宿主死亡时直接 `discard()` 货物：玩家会觉得"我杀了一只，另一只凭空消失"。自爆是唯一例外（货物被点燃并作为爆炸源）。

`setNoGravity(false)` 不能漏：货物被吊着时通常关了重力，释放后不恢复会原地悬空。

### 13.8.5 无货物 = 无嵌合体行为

行为 Goal 的 `canUse` 必须包含 `cargo() != null`，这样货物没了以后宿主会**自动退回原版 AI**，而不是原地悬停。这是"退回原版"的统一机制：所有组合体行为 Goal 都靠这一个条件失活，不需要在每个 Goal 里各写一套清理。

## 13.9 per-bone 渲染层与原版特效复刻

### 13.9.1 分骨骼渲染层

只给模型的一部分加效果（例如"闪电只作用在苦力怕区域，翅膀 / 尾巴不动"），用 **`CustomBoneTextureGeoLayer<T, O, R>(renderer, boneName, texture)`**。GeckoLib 的 per-bone 钩子只画**该骨骼自身的 cube**，把骨骼名逐个传入即可精确圈定范围，不需要动模型资产。

常用覆写点：`getRenderType(R state, Identifier texture)`。

**"替换"而非"叠加"陷阱（实测）**：`CustomBoneTextureGeoLayer#preRender` 内部会调用 `BoneSnapshot.skipRender(true)` + `skipChildrenRender(true)`，即**把该骨骼在原贴图那一遍里隐藏掉**。用它做"覆盖层"会让这些部位变成**半透明幽灵**（只剩覆盖贴图在画，底下的原贴图消失）。

**铁律**：凡是要"在原贴图上**叠加**一层效果"（原版闪电苦力怕就是叠加：普通贴图 + energy swirl 两道），**必须覆写 `preRender` 为空方法**，只保留 `addPerBoneRender` + `getRenderType`；若确实要"换贴图"才保留默认行为。

判定方法：用 `javap -c -p <geckolib jar>` 看该层都调用了什么，不要靠猜。

**泛型边界坑（必记）**：`CustomBoneTextureGeoLayer` / `GeoRenderLayer` 的状态参数被约束为 GeckoLib 的 `GeoRenderState`，而 `GeoEntityRenderer` 允许使用原版 `LivingEntityRenderState`（编译期两者不相等）。解决办法：渲染层用**原始类型 + `@SuppressWarnings`** 桥接，并在层内通过 `GeoRenderState#getAnimatableAge()` 读动画年龄，**不要**强转成自己的状态类。强转会在运行时 `ClassCastException`，或在某些版本上静默读到 null。

**注意**：`SubmitNodeCollector#order(int)` 返回的是 `OrderedSubmitNodeCollector`，**不能**当 `SubmitNodeCollector` 传给父类方法（编译期就会报错）。GeckoLib 的渲染层本身在基础渲染之后执行，通常无需自己排序。

### 13.9.2 复刻原版特效 = 抄参数，不是"看起来像"

**铁律**：复刻原版特效必须逐项对齐原版实现里的每一个数值与资源。以原版闪电苦力怕（26.3）为例：

| 项 | 值 |
|---|---|
| 纹理 | `textures/entity/creeper/creeper_armor.png` |
| 渲染类型 | `RenderTypes.energySwirl(texture, u, v)` |
| uv 滚动 | `u = xOffset(t) % 1`，`v = t * 0.01 % 1`，其中 `xOffset(t) = t * 0.01`，`t = ageInTicks` |
| 颜色 | `-8355712` |
| 来源 | `CreeperPowerLayer` + `EnergySwirlLayer`（以本地反编译源码为准） |

`u` 与 `v` 的滚动速率都是 `0.01`/tick，`% 1` 取模保证贴图循环；`-8355712` 是 `0xFF808080` 的十进制写法（`0xFF000000 | 0x00808080`）。"看着像"的近似值会在不同光照下明显不同，尤其是这个偏灰的乘法色。

### 13.9.3 变体复用同一模型的代码组织

同一模型 + 动画的多个变体（普通 / 闪电）**禁止**复制渲染器与模型类：

- 渲染器构造参数化（`texture`、`charged` 开关）。
- `GeoModel` 只把贴图作为构造参数，动画 / 几何 id 仍是常量（§13.6 第 2 条）。
- 实体侧用继承 + 一个 `protected` 工厂方法（如 `newSkull(...)`）切换弹射物，AI 全部继承。

复制一份渲染器意味着以后每改一处要改两处，且必然漏一处。

## 13.10 Mixin 注入的"匹配失败"规范

1. **注入点优先使用不依赖参数描述符的注入器**：`@ModifyConstant`、`@ModifyArg`、`@ModifyReturnValue`（MixinExtras）等。凡是把目标方法的参数类型写进 handler 的（`@Inject` / `@Overwrite`），都可能因**编译器生成的桥接方法（bridge）** 而匹配失败。

   实例（**实测**）：`PhantomRenderer#scale` 同时存在 `scale(PhantomRenderState, PoseStack)` 与桥接 `scale(LivingEntityRenderState, PoseStack)`。`@Inject` / `@Overwrite` 按描述符匹配到了桥接方法，导致 `InvalidInjectionException`，客户端资源重载失败并**黑屏卡死**；而 `@ModifyConstant(method = "scale", constant = @Constant(floatValue = ...))` 无参数约束，一次命中。

   机理：泛型擦除后编译器为覆写方法生成 bridge，两者字节码同名不同描述符；按参数类型写的 handler 只会命中其中一个，且通常命中的是"看起来更通用"的桥接版——而常量只存在于真正实现的那个方法里。

2. **注入失败必须响亮**：`required: true` + `defaultRequire: 1`。禁止用 `require = 0` 之外的"静默宽容"掩盖问题；确需宽容时（例如目标可能不存在）必须在成功路径打一次性日志。

3. **每个关键渲染 / 行为补丁都要留一次性执行日志**（`AtomicBoolean` 守卫），否则"改了没生效"会伪装成"改对了没变化"。日志要带注入点标识，不要只打一句"loaded"。

4. **mixin 配置的 `client` 列表在多加载器工程里不可靠**：渲染类补丁放进 `client` 段可能完全不被应用（主 `mixins` 段的补丁正常）。**铁律**：渲染补丁放主 `mixins` 段，由 `IMixinConfigPlugin#shouldApplyMixin` 按 `MixinEnvironment.getDefaultEnvironment().getSide() == CLIENT` 判定，并把判定结果打进日志。

5. **排障必须直接读实机日志**，并在 mixin 插件 `preApply` 里 dump 目标类的**真实方法表 + 关键常量所在方法**。靠 dump 才能发现"运行时方法带常量、桥接方法不带"这一关键差别；**禁止用"看反编译源码猜运行时"代替 dump**。

6. **注入点必须对真实字节码核定**（`javap -p -s` / `-c`）：禁止照抄其它版本线的方法名，禁止照抄另一端的 patched 源码；两端方法名不同时注解里同时列出 + `require = 1`。

## 13.11 症状索引

| 症状 | 首要怀疑 | 判据 | 小节 |
|---|---|---|---|
| 模型悬空 / 穿地 / 巨大 | 三件套不一致 | 按 `1.501 - L` 算净抬高，与 `sized()` 对照 | §13.4.2 |
| 模型浮在判定箱上方 | 误判"原版没对齐" | 先算 `1.501 - L`，再下结论 | §13.4.2 |
| 骑乘者悬空 / 嵌进模型 | 挂载点写成常量而非区间函数 | 检查挂载点是否由 `MODEL_BAND_TOP` 推导 | §13.4.4 |
| 瞄准头部打不中 / 生物看不见玩家 | `eyeHeight` 超过判定箱高度 | 比对 `eyeHeight` 与 `sized().height` | §13.4.6 |
| 首刷即崩，日志无上下文 | `AttributeSupplier` 漏注册 | 搜崩溃栈里的 `DefaultAttributes` 报错 | §13.3 |
| 刷怪蛋点了没反应 | `EntityType` 未注册或刷怪蛋引用错类型 | 检查注册链与 `spawnEgg(type)` | §13.3.1 |
| 生物不动（设了目标点无效） | 用了 `setWantedPosition` 驱动幻翼 | 检查是否走 `moveTargetPoint` accessor | §13.2 |
| 索敌正常但不动手 | 行为 Goal 未声明 `Goal.Flag` | 检查 `setFlags(...)` | §13.5 |
| 冷却卡住不动 | 冷却计时放在 Goal 字段 | 检查计时器是否在实体 tick | §13.5 |
| `/summon` 出来的生物没有变体 / 没有货 | 初始化只在 `finalizeSpawn` 里 | 检查首 tick 兜底 | §13.5 |
| 同一宿主挂着两只货物 | 归属只存字段、未持久化 | tick 打印认领数，看是否为 2 | §13.8.1 |
| 投出去的货物又回来了 | 释放时未摘标签 | 检查 `removeTag` | §13.8.2 |
| 宿主死了货物凭空消失 | 走了 `discard()` 而非 `release` | 检查死亡分支 | §13.8.4 |
| 货物释放后原地悬空 | 漏了 `setNoGravity(false)` | 检查释放三步是否齐全 | §13.8.4 |
| 覆盖层部位变半透明幽灵 | 未覆写空的 `preRender` | `javap` 看该层调用链 | §13.9.1 |
| 特效颜色 / 滚动速度不对 | 抄了"看着像"的近似值 | 对照原版数值表 | §13.9.2 |
| 渲染补丁完全没生效 | 补丁放在 `client` 段 | 检查 `mixins.json` 段与 `shouldApplyMixin` | §13.10 |
| 客户端黑屏卡死 | 注入命中桥接方法抛 `InvalidInjectionException` | 读日志里的异常类名 | §13.10 |
| 骨骼 / 贴图区域判错 | 凭截图或骨骼名猜测 | 解析 `.geo.json` 的 `uv` + `size` | §13.6.1 |

## 13.12 自检清单

**分类与注册**

- [ ] 生物定性（§13.1）正确；原版组合体未注册 `EntityType`
- [ ] 原版组合体的变体产生只走一个静态入口
- [ ] 全新生物七项注册齐备（`EntityType.Builder` / 两端注册 / `AttributeSupplier` / 渲染器 / 刷怪蛋 + 语言键 + 战利品表 / 双语名 / 生成权重）

**渲染对齐**

- [ ] 模型区间、`sized()`、渲染 `translate` 三者已按 `1.501 - L` 推导并核对
- [ ] 挂载偏移写成相对模型区间的表达式，推导过程在注释里
- [ ] 改偏移前做过定值验证（原样跑一遍 + F3+B + 日志数值）
- [ ] 代码未触碰 `*.bbmodel` / `*.geo.json` / `*.png` / `*.animation.json`
- [ ] 每个生物有 F3+B 对齐截图

**行为**

- [ ] 索敌 Goal 的 `canUse` 恒为 `false`，行为 Goal 读 `getTarget()`
- [ ] 抢占型 Goal 挂优先级 0 且声明了 `Goal.Flag`
- [ ] 冷却计时在实体 tick，不在 Goal 字段
- [ ] `finalizeSpawn` 之外的首 tick 兜底逻辑存在且幂等
- [ ] 组合体行为 Goal 的 `canUse` 含 `cargo() != null`

**GeckoLib**

- [ ] 资源路径与 26.3 裸名 id 规则一致
- [ ] `GeoModel` 的 id 是常量，无每帧 `Identifier`
- [ ] 骨骼 / UV 判定来自 `.geo.json` 解析，非目测
- [ ] 叠加型渲染层覆写了空的 `preRender`
- [ ] 复刻原版特效的每个数值都有出处

**配置与生命周期**

- [ ] 手感参数全在配置文件；几何推导值全在代码常量
- [ ] 配置变更在下一次生成时生效（候选表重建）
- [ ] 货物带归属标签 `<modid>_cargo:<UUID>`，认领优先、生成兜底
- [ ] 释放时摘标签 + `setNoAi(false)` + `setNoGravity(false)`
- [ ] 宿主非自爆死亡走 `release`，未 `discard()`

**Mixin**

- [ ] 注入器优先无描述符依赖（`@ModifyConstant` / `@ModifyArg` / `@ModifyReturnValue`）
- [ ] `required: true` + `defaultRequire: 1`
- [ ] 关键补丁有 `AtomicBoolean` 一次性日志
- [ ] 渲染补丁在主 `mixins` 段，由 `shouldApplyMixin` 判 side 并打日志
- [ ] 注入点已用 `javap` 核定，排障读实机日志 + `preApply` dump

**验收**

- [ ] `gradlew :fabric:jar :neoforge:jar` 双端零错误（IDE 报错不算数）
- [ ] 三条生成途径各验一次：自然生成（改配置权重）、刷怪蛋、`/summon`
- [ ] 杀掉货物 / 宿主各验一次（掉落、爆炸、释放行为符合设计）
- [ ] 存档重进：变体与货物状态不丢失、不翻倍
