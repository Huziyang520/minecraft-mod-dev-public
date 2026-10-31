# minecraft-mod-dev

**Minecraft Java 版模组开发文档集（面向 AI 编码 agent）** —— 一套可直接交给 Codex、Claude Code、Hermes、Cursor、GitHub Copilot 等 agent 阅读的模组开发知识库。

覆盖：多加载器（Fabric / NeoForge / Forge）工程结构、元数据、本地化、Mixin 与跨版本 API 适配、构建与依赖、多版本移植、测试部署验收、Git 与 CurseForge / Modrinth 发布、排障案例库、美术与生物动画规范。

- **许可证**：CC0-1.0（公有领域，随便用、随便改、不用署名）
- **无个人数据**：仓库内不含任何作者机器路径、代理端口、实例名、账号或项目 ID。**所有环境相关的地方都写成占位符**，需要你按自己的实际环境替换（见 [按你的环境调整](#按你的环境调整)）。

---

## 目录

- [这个仓库里有什么](#这个仓库里有什么)
- [怎么交给你的 Agent](#怎么交给你的-agent)
  - [Codex](#codex)
  - [Claude Code](#claude-code)
  - [Hermes Agent](#hermes-agent)
  - [Cursor / VS Code Copilot](#cursor--vs-code-copilot)
  - [只把 AGENTS.md 抄进模组仓库](#只把-agentsmd-抄进模组仓库)
- [第三方工具：CC Switch](#第三方工具cc-switch)
- [按你的环境调整](#按你的环境调整)
- [模块索引](#模块索引)
- [更新与贡献](#更新与贡献)
- [许可证](#许可证)

---

## 这个仓库里有什么

```
minecraft-mod-dev/
├── AGENTS.md                 # 可直接抄进你的模组仓库：项目硬约束清单
├── README.md                 # 本文件
├── LICENSE                   # CC0-1.0
└── references/               # 15 个主题模块（按需取用，不用全读）
    ├── 01-publishing.md      # 发布身份 / 版本号 / 产物命名 / 前置区间
    ├── 02-environment.md     # Java 分档 / Gradle 三铁律 / 镜像 / JAVA_HOME / 离线优先
    ├── 03-metadata.md        # 工程结构 / fabric.mod.json / mods.toml 字段全集 / 图标 / pack.mcmeta
    ├── 04-i18n.md            # 语言清单 / 键齐平 / 无 BOM / 可翻译组件 / 兜底
    ├── 05-mixin-api.md       # Mixin 注册与注入点校验 / 版本差异 / 事件桥接 / 可选前置
    ├── 06-build.md           # Gradle 档位依据 / FG6-FG7 / Loom-MDG / 依赖声明 / 构建故障
    ├── 07-porting.md         # L1-L2-L3 / 触点差分 / 探针 / range 纪律 / 老版本线
    ├── 08-deploy-qa.md       # 部署三查 / 实例判定 / MD5 回读 / 验收清单
    ├── 09-git-upload.md      # 分支管理 / 推送红线 / CurseForge / Modrinth / Commit 规范
    ├── 10-troubleshooting.md # Debug Flow / 症状索引 / 全部实战案例
    ├── 11-verdicts.md        # 易错结论对照 / 待核实清单 / 元原则
    ├── 12-art.md             # Blockbench 模型与贴图规范
    ├── 13-entities.md        # 生物分类 / 注册链 / 渲染对齐 / GeckoLib / 生命周期
    ├── 14-workflows.md       # 建新模组 / 扩版本 / 发版本 / 线上排障
    └── 15-docs.md            # 交付文档写什么、绝不写什么
```

**两份东西，两种用途**：

| 文件 | 是什么 | 怎么用 |
|---|---|---|
| `AGENTS.md` | **项目硬约束**（"必须 / 禁止"），带占位符 | 抄进你的模组仓库根目录，填掉占位符 |
| `references/` | **知识库**（命令、字段表、判据、案例），各模块平级、互相引用 | 按需打开对应的一篇；agent 也可以按 `AGENTS.md` §10 的索引跳转 |

**读法**：`references/` 每份文件开头都写明"管什么、什么时候读"，按场景直接打开那一篇即可，不必全读。模块之间的交叉引用都是相对链接，顺着链接就能跳到细节。

---

## 怎么交给你的 Agent

这套东西**不是**标准的 Agent Skill（那个标准要求根目录有 `SKILL.md`，本仓库有意不提供）。它走的是更通用的两条路：**`AGENTS.md` 作仓库指令** + **`references/` 作可按需检索的知识库**。

先把仓库放到本地：

```bash
git clone https://github.com/Huziyang520/minecraft-mod-dev-public.git minecraft-mod-dev
```

### Codex

Codex（CLI / IDE 扩展 / App）**原生读取仓库根目录的 `AGENTS.md`**，这是最省事的一条路。

```bash
# 在你的模组仓库根目录
cp <本仓库>/AGENTS.md ./AGENTS.md
cp -r <本仓库>/references ./references
```

然后提交进仓库。之后在这个仓库里跑 `codex`，它每次都会读到 `AGENTS.md`，并按 §10 的索引去 `references/` 里查细节。

> 想全局生效（对所有项目）：把精简后的约束放进 `~/.codex/AGENTS.md`。**不建议整份塞进去** —— 会长期占用上下文，把细节留在 `references/` 里按需读更划算。

### Claude Code

Claude Code 读 `CLAUDE.md`，也识别仓库根的 `AGENTS.md`。两种接法：

**方式 A —— 直接放 `AGENTS.md`**（推荐，与 Codex / Cursor / Copilot 共用同一份）：

```bash
cp <本仓库>/AGENTS.md ./AGENTS.md
cp -r <本仓库>/references ./references
```

**方式 B —— 用 `CLAUDE.md` 指过去**（你的仓库已经有 `CLAUDE.md` 时用这个，避免两份规则打架）：

```markdown
<!-- CLAUDE.md -->
本仓库的模组开发约束见 @AGENTS.md 。
遇到具体技术问题（元数据字段、Gradle 档位、Mixin 注入、排障案例等）时，
按 AGENTS.md §10 的索引到 references/ 下对应文件里查。
```

- `@AGENTS.md` 是 Claude Code 的文件引用语法，会把那份内容纳入上下文。
- 项目级 `.claude/` 目录下的配置同样生效，但**规则正文建议只维护一份**（`AGENTS.md`），其余位置用引用。

**方式 C —— 全局**：把约束精简后放进 `~/.claude/CLAUDE.md`，让所有项目都带上；细节仍留在 `references/`。

### Hermes Agent

Hermes 的**工作区指令**就是读仓库根目录的 `AGENTS.md`（它自己的技能系统另有一套，本仓库不依赖那套）。

```bash
cp <本仓库>/AGENTS.md ./AGENTS.md
cp -r <本仓库>/references ./references
```

- 首次在某个仓库里用时，若 Hermes 提示需要信任仓库，按提示执行 `hermes skills trust` 或直接确认即可。
- 想跨项目共享一份知识库，把 `references/` 放到一个固定目录，在 `AGENTS.md` 里用**相对路径或绝对路径**指向它即可（Hermes 没有强制的目录约定）。

> 细节：[Hermes 文档](https://hermes-agent.nousresearch.com/docs/)。

### Cursor / VS Code Copilot

两者都读仓库根目录的 `AGENTS.md`：

- **Cursor**：把 `AGENTS.md` 放进仓库根目录即可。也可以把 `AGENTS.md` 的 §1–§9 拆成几条 `.cursor/rules/*.mdc`，但**规则只维护一份**，别两处各写一套。
- **VS Code + GitHub Copilot**：同样读 `AGENTS.md`；`references/` 作为普通 Markdown 放在仓库里，Copilot 需要时会自己去读。

### 只把 AGENTS.md 抄进模组仓库

大多数时候你真正需要的只有这一步：

```bash
# 在你的模组仓库根目录
curl -fsSL https://raw.githubusercontent.com/Huziyang520/minecraft-mod-dev-public/main/AGENTS.md -o AGENTS.md
```

然后**把里面的 `<…>` 占位符换成你的真实值**（JDK 路径、代理端口、实例前缀、目标实例目录、脚本目录）。改完提交进仓库，团队里每个人的 agent 都会遵守同一套约束。

> `AGENTS.md` 是**项目硬约束**（"必须 / 禁止"），`references/` 是**通用知识**。不要把教程写进 `AGENTS.md`，也不要把某个项目的特有约定写进 `references/`。

---

## 第三方工具：CC Switch

[CC Switch](https://github.com/farion1231/cc-switch)（官网 [ccswitch.io](https://ccswitch.io)）是一个 Tauri 写的桌面管理器，把 **Claude Code / Codex / Gemini CLI / OpenCode / OpenClaw / Hermes / Pi** 等十几种 agent 的**供应商配置、MCP、技能、提示词**集中到一处管理。

对本仓库有用的三个能力：**提示词库批量分发 `AGENTS.md`**、**项目切换**、**切换供应商时不动你的配置文件**。

### 安装 CC Switch

到 [Releases 页](https://github.com/farion1231/cc-switch/releases/latest) 下载对应平台安装包（Windows `.msi` / macOS `.dmg` / Linux `.AppImage` 或 `.deb`）。首次启动会在 `~/.cc-switch/` 建库（SQLite）。

> 只有图形界面版本；无桌面环境（服务器 / SSH）用社区维护的 [CC Switch CLI](https://github.com/SaladDay/cc-switch-cli)（`brew install cc-switch-cli`），它默认共享同一个 `~/.cc-switch` 数据目录。

### 用提示词库分发 AGENTS.md

CC Switch 的 **Prompts** 面板是按工具分组的提示词库，带 Markdown 编辑器。把本仓库的 `AGENTS.md` 存成一条 prompt 后，可以一键写进某个项目的 `AGENTS.md` / `CLAUDE.md` / `GEMINI.md`：

1. Prompts 面板 → 新建 → 粘贴 `AGENTS.md` 内容 → 命名（例如 `minecraft-mod-dev 约束`）。
2. 选中目标工具（Claude Code / Codex / Gemini CLI …）与目标项目。
3. 启用后 CC Switch 会把内容写进该工具的提示词文件；**原文件内容会先被存回库**，不会丢。

> ⚠ 这里写的是"提示词文件"（`AGENTS.md` / `CLAUDE.md`），不是"技能"。本仓库没有 `SKILL.md`，所以 CC Switch 的 **Skills 面板不适用**（那套要求目录里有 `SKILL.md`）。`references/` 当普通文档放进仓库即可。

### 项目切换（多版本线场景）

如果你的模组仓库按 MC 版本分目录（`1.20.1/`、`1.21.1/`、`26.3/`…），可以把"每个目录的供应商 + MCP + 提示词"存成一个 **Project**，换目录时一键整套切过去，不用手动改配置。

### 切换供应商不会弄坏你的规则文件

CC Switch 的设计原则是**最小侵入**：切换供应商时只改写配置文件里的**连接字段**（endpoint、key、model、协议），以及该供应商自己的兼容开关。**插件、hooks、MCP、提示词文件、你手写的注释与格式全部保留**。首次改写前会把原文件备份到 `~/.cc-switch/backups/live-first-write/`。

所以：换模型 / 换中转站之后，仓库里的 `AGENTS.md` 与 `references/` **原样还在**，不用重做。

### 其它有用的能力

- **用量统计**：本地扫描各工具会话日志，按供应商 / 模型统计 token 与花费；不需要开本地路由也能用。
- **会话管理器**：把会话日志解析成结构化视图（文本 / 思考 / 工具调用 / 图片分开），可以导出 Markdown、复制恢复命令。排障时把某次崩溃排查的会话导出来复盘很有用。
- **MCP 统一面板**：一处管理所有 MCP server，按工具分发。

### 常见坑

| 现象 | 原因 / 处理 |
|---|---|
| 切了供应商但 CLI 还是老模型 | Codex / Gemini CLI 需要**重启终端或守护进程**；Claude Code 支持热切换不用重启 |
| 提示词写进去了但 agent 没读到 | 目标工具的提示词文件路径不对 —— 到"设置 → 应用配置"里改该工具的配置目录 |
| 配置被改回默认 | 模型是**供应商级字段**：你在工具里用 `/model` 选的模型，下次切换供应商时会被目标供应商保存的模型覆盖。要长期用某个模型，去 CC Switch 里编辑那个供应商 |
| 开了本地路由后配置里全是 `127.0.0.1` | 这是正常的：路由模式下工具请求先到 CC Switch 本地端口，再转发到真实供应商。退出 CC Switch 时会先写回直连配置 |
| 数据库版本报"太新" | CLI 版本落后于桌面版，升级 CLI 或等它跟进 |

> 细节以 [CC Switch README](https://github.com/farion1231/cc-switch) 与 [用户手册](https://github.com/farion1231/cc-switch/tree/main/docs/user-manual) 为准；版本迭代快，**以你装的这一版界面为准**。

---

## 按你的环境调整

**这一步不能跳过。** 本仓库为了能公开，把所有"只对某一台机器成立"的事实都换成了占位符。下列位置**必须按你的环境改**：

### 1. Java / JDK 路径（`references/02-environment.md`、`AGENTS.md` §2）

```bash
# 先找出你机器上到底有哪几档 JDK，且每个都是"真 JDK"（有 javac）
java -version
javac -version
```

- 常见来源：系统安装的 JDK、IDE 自带 JBR、启动器运行时（`<启动器>/runtime/java-runtime-*`）、`~/.gradle/jdks`（foojay 下载的）。
- **JRE 不能编译**：目录里没有 `bin/javac` 就是 JRE。把 `javac` 拷进 JRE 也没用。
- 分档结论是稳定的（1.18–1.20.4 → 17；1.20.5–1.21.11 → 21；26.x → 25；老版本线 → 8），**路径不稳定**。

### 2. Gradle 缓存与档位（`references/02-environment.md` §2.3）

```bash
# 你本地已经缓存了哪些 Gradle 发行版？
ls ~/.gradle/wrapper/dists
```

```powershell
Get-ChildItem "$env:USERPROFILE\.gradle\wrapper\dists" -Directory | Select-Object Name
```

- **先查缓存再决定是否下载**。缓存里已有同系列补丁版时，把 wrapper 指过去，不要为对齐数字多下一份。
- 档位由**插件**决定，不能一刀切：现代线（Loom 1.17 / MDG 2.0.14x）→ Gradle 9.x；1.20–1.21.1 与 Forge ≤1.21.8（FG6）→ Gradle 8.x。

### 3. 代理（`AGENTS.md` §2、`references/09-git-upload.md` §9.1）

```bash
git config --get http.proxy
```

把 `<你的代理端口>` 换成你实际在用的端口；没有代理就把相关条目整行删掉，**不要留占位符**。直连可用时不需要代理。

### 4. 测试实例（`AGENTS.md` §6、`references/08-deploy-qa.md`）

- 先定位你的 `.minecraft`（启动器里能看到），再看 `versions/` 下的实例目录名。
- 约定一个**只用于测试的目录前缀**（例如 `test-` 或你习惯的任何前缀），把它写进 `AGENTS.md` §6；`<禁部署标记>` 换成你用来标记"禁止写入"的名字。
- 加载器判定**用库坐标**，不要靠目录名：`net.neoforged.fancymodloader` ⇒ NeoForge；`net.fabricmc:fabric-loader` ⇒ Fabric；`net.minecraftforge:fmlloader` ⇒ Forge。

### 5. 发布脚本目录（`references/09-git-upload.md` §9.4）

上传脚本（CurseForge / Modrinth）**放在仓库之外的私有目录**，不要提交进模组仓库。把 `<脚本目录>` 换成你的实际路径。

### 6. 凭据

`curseforge_api_key` / `modrinthApiKey` 放在你**全局**的 `~/.gradle/gradle.properties` 里，**不要放进任何仓库**。本仓库的规则是：agent 不读、不搜、不打印、不落盘、不转交外部命令，只允许 Gradle `findProperty` 消费。

### 7. 验证改完没漏

```bash
grep -rn '<你的' . --include='*.md'
```

**占位符只在两处，含义固定**：

| 出现位置 | 占位符 | 什么时候要改 |
|---|---|---|
| `AGENTS.md`（抄进你的模组仓库时） | `<JDK 17>` / `<JDK 21>` / `<JDK 25>` / `<JDK 8>` / `<你的代理地址与端口>` / `<你的 git 路径>` / `<你的测试实例前缀>` / `<禁部署标记>` / `<实例目录>` / `<脚本目录>` / `<你的备用反馈页>` / `<9.6.1>` / `<8.11>` | **必须全改**，否则 agent 会照着占位符干活 |
| `references/`（只在你要照着执行某段命令时） | `<你的代理端口>`、`<你的 JDK 路径>`、`<脚本目录>`、`<你的反馈页>` 等 | 按需改；**不执行那段命令就不用动** |

判断方法：占位符里带"你的"三个字 = 要按环境填；写成 `<mod_id>` / `<version>` / `<Loader>` = 只是文档里的变量名，照抄时换成自己的值即可。

---

## 模块索引

| 我要做什么 | 先读 |
|---|---|
| 从零建一个模组 | [14-workflows.md](references/14-workflows.md) §14.2 → [01-publishing.md](references/01-publishing.md) → [03-metadata.md](references/03-metadata.md) |
| 接手别人的老工程 | [02-environment.md](references/02-environment.md) → [06-build.md](references/06-build.md) → [10-troubleshooting.md](references/10-troubleshooting.md) |
| 把模组扩到更多 MC 版本 | [07-porting.md](references/07-porting.md) → [05-mixin-api.md](references/05-mixin-api.md) → [08-deploy-qa.md](references/08-deploy-qa.md) |
| 发一个版本 | [01-publishing.md](references/01-publishing.md) → [09-git-upload.md](references/09-git-upload.md) → [08-deploy-qa.md](references/08-deploy-qa.md) |
| 上线后出问题 | [10-troubleshooting.md](references/10-troubleshooting.md) → [11-verdicts.md](references/11-verdicts.md) |
| 构建报错 / 卡死 | [06-build.md](references/06-build.md) → [10-troubleshooting.md](references/10-troubleshooting.md) |
| 元数据 / 图标 / 进不了世界 | [03-metadata.md](references/03-metadata.md) |
| Mixin 注入不生效 | [05-mixin-api.md](references/05-mixin-api.md) → [10-troubleshooting.md](references/10-troubleshooting.md) |
| 多语言失效 | [04-i18n.md](references/04-i18n.md) |
| 做模型 / 贴图 | [12-art.md](references/12-art.md) |
| 加生物 / 动画 | [13-entities.md](references/13-entities.md) |
| 写 README / API 文档 | [15-docs.md](references/15-docs.md) |
| 判断某条旧结论还成不成立 | [11-verdicts.md](references/11-verdicts.md) |

---

## 更新与贡献

```bash
git pull
```

- 各模块**平级**，没有上位总纲。改规则时**只改一个地方**：该规则属于哪个主题就改哪一份，并在需要的地方补相对链接。
- 新增实测结论请带**证据**（`javap` 输出、jar 解包结果、maven 元数据、错误原文），并标注证据强度（实测 / 推断）。
- **不要**提交任何个人环境信息（路径、端口、账号、项目 ID、实例名）。提交前自查：

  ```bash
  grep -rnE '([CD]:\\|/Users/|/home/)' . --include='*.md'
  ```

- 本仓库采用 CC0-1.0：贡献即视为放弃著作权，不要求署名。

---

## 许可证

[CC0 1.0 Universal](LICENSE) —— 公有领域奉献。你可以复制、修改、分发、商用，**无需署名**。

```
To the extent possible under law, the authors have waived all copyright and
related or neighboring rights to this work.
```
