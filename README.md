# minecraft-mod-dev

**Minecraft Java 版模组开发技能包 / Agent Skill** —— 面向 Codex、Claude Code、Hermes、Cursor、GitHub Copilot 等支持 `SKILL.md` 或 `AGENTS.md` 的 AI 编码 agent。

覆盖：多加载器（Fabric / NeoForge / Forge）工程结构、元数据、本地化、Mixin 与跨版本 API 适配、构建与依赖、多版本移植、测试部署验收、Git 与 CurseForge / Modrinth 发布、排障案例库、美术与生物动画规范。

- **许可证**：CC0-1.0（公有领域，随便用、随便改、不用署名）
- **无个人数据**：仓库内不含任何作者机器路径、代理端口、实例名、账号或项目 ID。**所有环境相关的地方都写成占位符**，需要你按自己的实际环境替换（见 [按你的环境调整](#按你的环境调整)）。
- **权威文件**：`SKILL.md` 是整合版、**结论以此为准**；`references/` 是分模块详解（命令、字段全集、实测表格、案例）。

---

## 目录

- [这个仓库里有什么](#这个仓库里有什么)
- [安装到各个 Agent](#安装到各个-agent)
  - [Claude Code](#claude-code)
  - [Codex (OpenAI Codex CLI / IDE / App)](#codex)
  - [Hermes Agent](#hermes-agent)
  - [Cursor / VS Code Copilot / 其它 `AGENTS.md` 类 agent](#cursor--vs-code-copilot--其它-agentsmd-类-agent)
  - [只把 AGENTS.md 抄进你的模组仓库](#只把-agentsmd-抄进你的模组仓库)
- [第三方工具：CC Switch](#第三方工具cc-switch)
- [按你的环境调整](#按你的环境调整)
- [怎么用这个技能包](#怎么用这个技能包)
- [更新与贡献](#更新与贡献)
- [许可证](#许可证)

---

## 这个仓库里有什么

```
minecraft-mod-dev/
├── SKILL.md                  # 整合版技能入口（权威文件，先读这个）
├── AGENTS.md                 # 示例 AGENTS.md：抄进你的模组仓库当项目硬约束
├── README.md                 # 本文件
├── LICENSE                   # CC0-1.0
├── references/               # 分模块详解（按需加载，不用全读）
│   ├── 01-publishing.md      # 发布身份 / 版本号 / 产物命名 / 前置区间
│   ├── 02-environment.md     # Java 分档 / Gradle 三铁律 / 镜像 / JAVA_HOME / 离线优先
│   ├── 03-metadata.md        # 工程结构 / fabric.mod.json / mods.toml 字段全集 / 图标 / pack.mcmeta
│   ├── 04-i18n.md            # 语言清单 / 键齐平 / 无 BOM / 可翻译组件 / 兜底
│   ├── 05-mixin-api.md       # Mixin 注册与注入点校验 / 版本差异 / 事件桥接 / 可选前置
│   ├── 06-build.md           # Gradle 档位依据 / FG6-FG7 / Loom-MDG / 依赖声明 / 构建故障
│   ├── 07-porting.md         # L1-L2-L3 / 触点差分 / 探针 / range 纪律 / 老版本线
│   ├── 08-deploy-qa.md       # 部署三查 / 实例判定 / MD5 回读 / 验收清单
│   ├── 09-git-upload.md      # 分支管理 / 推送红线 / CurseForge / Modrinth / Commit 规范
│   ├── 10-troubleshooting.md # Debug Flow / 症状索引 / 全部实战案例
│   ├── 11-verdicts.md        # 易错结论对照 / 待核实清单 / 元原则
│   ├── 12-art.md             # Blockbench 模型与贴图规范
│   ├── 13-entities.md        # 生物分类 / 注册链 / 渲染对齐 / GeckoLib / 生命周期
│   ├── 14-workflows.md       # 建新模组 / 扩版本 / 发版本 / 线上排障
│   └── 15-docs.md            # 交付文档写什么、绝不写什么
└── push.ps1 / push.sh        # 一键推送脚本（本地用，不入库；见「更新与贡献」）
```

**读法**：agent 只加载 `SKILL.md` 的 frontmatter（`name` + `description`），判定相关时才加载正文；正文里按需跳转到 `references/` 的某一篇。这就是"渐进式披露"，所以 `references/` 拆得越细、单次消耗的上下文越少。

---

## 安装到各个 Agent

技能包本质就是**一个目录 + 一个 `SKILL.md`**（[Agent Skills 开放标准](https://agentskills.io/specification)）。各 agent 的差别只在"从哪个目录读"。

先把仓库放到本地：

```bash
git clone https://github.com/Huziyang520/minecraft-mod-dev-public.git minecraft-mod-dev
```

### Claude Code

**个人级（所有项目都能用）**：

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/minecraft-mod-dev
cp -r ./minecraft-mod-dev/* ~/.claude/skills/minecraft-mod-dev/
```

```powershell
# Windows PowerShell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills\minecraft-mod-dev"
Copy-Item -Recurse -Force ".\minecraft-mod-dev\*" "$env:USERPROFILE\.claude\skills\minecraft-mod-dev\"
```

**项目级（只在这个仓库生效，可提交给团队）**：

```bash
mkdir -p .claude/skills/minecraft-mod-dev
cp -r ./minecraft-mod-dev/* .claude/skills/minecraft-mod-dev/
```

**也可以直接软链接**，这样 `git pull` 后不用再复制：

```bash
# macOS / Linux
ln -s "$PWD/minecraft-mod-dev" ~/.claude/skills/minecraft-mod-dev
```

```powershell
# Windows（需要开发者模式或管理员）
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.claude\skills\minecraft-mod-dev" -Target "$PWD\minecraft-mod-dev"
```

**验证**：启动 `claude`，输入 `/skills` 应能看到 `minecraft-mod-dev`；或直接 `/minecraft-mod-dev` 手动调用。也可以问一句"帮我建个 Fabric 模组"，看它是否自动加载。

> 细节：[Claude Code Skills 文档](https://code.claude.com/docs/en/skills)。Claude Code 还支持 `~/.claude/skills/<name>` 作为**符号链接**指向别处，且会去重（多个位置指向同一目标只加载一次）。

### Codex

Codex（CLI / IDE 扩展 / App）读取 `AGENTS.md` 作为仓库级指令，并支持 Agent Skills 目录。

**方式 A —— 作为技能**（推荐，能享受渐进式加载）：把本仓库放进 Codex 的技能目录（通常是 `~/.codex/skills/<name>/`，以你所用版本的 `codex --help` / 官方文档为准）：

```bash
mkdir -p ~/.codex/skills/minecraft-mod-dev
cp -r ./minecraft-mod-dev/* ~/.codex/skills/minecraft-mod-dev/
```

**方式 B —— 作为仓库指令**：把本仓库的 `AGENTS.md` 复制到你**自己的模组仓库根目录**（见 [下一节](#只把-agentsmd-抄进你的模组仓库)）。Codex 每次在这个仓库里工作都会读它。

**方式 C —— 全局指令**：把 `SKILL.md` 的关键约束精简后放进 `~/.codex/AGENTS.md`（全局 `AGENTS.md` 对所有项目生效）。**不建议整份塞进去** —— 会长期占用上下文，把细节留在技能包里按需加载更划算。

> 细节：[Codex 文档](https://developers.openai.com/codex)。安装 CLI：`npm install -g @openai/codex`，或 `brew install --cask codex`，或见 [官方安装页](https://github.com/openai/codex)。

### Hermes Agent

Hermes 的技能目录是 `~/.hermes/skills/`（**唯一事实来源**），也支持**外部技能目录**和**项目级技能**。

**方式 A —— 放进主技能目录**：

```bash
mkdir -p ~/.hermes/skills/minecraft-mod-dev
cp -r ./minecraft-mod-dev/* ~/.hermes/skills/minecraft-mod-dev/
```

装好后它自动变成斜杠命令：`/minecraft-mod-dev 帮我加一个 1.21.1 的分支`。

**方式 B —— 外部目录（推荐，多个 agent 共享一份）**：把仓库放在 `~/.agents/skills/minecraft-mod-dev`，然后在 `~/.hermes/config.yaml` 里加：

```yaml
skills:
  external_dirs:
    - ~/.agents/skills
```

**方式 C —— 项目级**：在你的模组仓库里放 `<repo>/.hermes/skills/minecraft-mod-dev/` 或 `<repo>/.agents/skills/minecraft-mod-dev/`，首次使用时 Hermes 会提示 `run hermes skills trust`，信任该仓库后才加载（这是防"克隆来的仓库自带指令"的安全设计）。

**验证**：`/skills` 列出它；或 `hermes chat --toolsets skills -q "列出你的技能"`。

> 细节：[Hermes Skills 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)。Hermes 兼容 [agentskills.io](https://agentskills.io) 开放标准，并支持 `platforms` 字段限制操作系统。

### Cursor / VS Code Copilot / 其它 `AGENTS.md` 类 agent

这类工具通常读仓库根目录的 `AGENTS.md`（部分也读 `.agents/skills/` 或 `.cursor/rules/`）。

- **Cursor**：把 `AGENTS.md` 放进仓库根目录即可（Cursor 也识别 `.cursor/rules/*.mdc`，可把 §1–§9 拆成几条规则）。
- **VS Code + GitHub Copilot**：支持 `.agents/skills/<name>/SKILL.md`。按 Agent Skills 的默认约定：

  ```bash
  mkdir -p .agents/skills/minecraft-mod-dev
  cp -r ./minecraft-mod-dev/* .agents/skills/minecraft-mod-dev/
  ```

- **其它**：只要是"读 `AGENTS.md`"或"读 `SKILL.md`"的 agent，把对应文件放到位即可。`SKILL.md` 的 frontmatter 只用了标准字段（`name` / `description` / `license`），不依赖任何厂商扩展。

### 只把 AGENTS.md 抄进你的模组仓库

大多数时候你真正需要的只有这一步：

```bash
# 在你的模组仓库根目录
curl -fsSL https://raw.githubusercontent.com/Huziyang520/minecraft-mod-dev-public/main/AGENTS.md -o AGENTS.md
```

然后**把里面的 `<…>` 占位符换成你的真实值**（JDK 路径、代理端口、实例前缀、目标实例目录、脚本目录）。改完提交进仓库，团队里每个人的 agent 都会遵守同一套约束。

> `AGENTS.md` 是**项目硬约束**（"必须 / 禁止"），`SKILL.md` 是**通用知识**。不要把教程写进 `AGENTS.md`，也不要把某个项目的特有约定写进 `SKILL.md`。

---

## 第三方工具：CC Switch

[CC Switch](https://github.com/farion1231/cc-switch)（官网 [ccswitch.io](https://ccswitch.io)）是一个 Tauri 写的桌面管理器，把 **Claude Code / Codex / Gemini CLI / OpenCode / OpenClaw / Hermes / Pi** 等十几种 agent 的**供应商配置、MCP、Skills、提示词**集中到一处管理。

对本技能包有用的三个能力：**一键装技能**、**跨工具同步技能**、**切换供应商时不动你的技能与配置**。

### 安装 CC Switch

到 [Releases 页](https://github.com/farion1231/cc-switch/releases/latest) 下载对应平台安装包（Windows `.msi` / macOS `.dmg` / Linux `.AppImage` 或 `.deb`）。首次启动会在 `~/.cc-switch/` 建库（SQLite）。

> 只有图形界面版本；无桌面环境（服务器 / SSH）用社区维护的 [CC Switch CLI](https://github.com/SaladDay/cc-switch-cli)（`brew install cc-switch-cli`），它默认共享同一个 `~/.cc-switch` 数据目录。

### 用 CC Switch 装这个技能

三种方式，任选：

1. **从 GitHub 仓库安装**：Skills 面板 → 从 GitHub 仓库安装 → 填 `Huziyang520/minecraft-mod-dev-public`。它会拉 `SKILL.md` 和它引用的 `references/` 文件。
2. **从 ZIP 安装**：把本仓库打包成 zip（GitHub 的 `Code → Download ZIP`），Skills 面板选"从 ZIP 安装"。
3. **从 skills.sh 搜索**：Skills 面板支持搜索 [skills.sh](https://skills.sh) 索引；若本技能已被索引，可直接搜 `minecraft-mod-dev` 一键装。

### 装完要做的两件事

**① 选择同步方式**。CC Switch 的 Skills 面板支持三种落盘策略：

| 策略 | 行为 | 什么时候用 |
|---|---|---|
| **符号链接**（默认） | 在 `~/.cc-switch/skills/` 存一份，各工具目录建软链指过去 | 多个 agent 共用一份，改一处全部生效（推荐） |
| **复制** | 往每个工具目录各拷一份 | 目标工具不跟随软链（如某些沙箱 / 容器环境） |
| **只存中心目录** | 只放 `~/.agents/skills`，不往工具目录写 | 你已经在各工具的配置里声明了外部技能目录（如 Hermes 的 `external_dirs`） |

**② 勾选要同步到哪些工具**。Skills 面板可以按工具开关；"全部启用 / 全部停用"也支持。

> 软链失败时 CC Switch 会自动回退成复制，不会静默失败；但**回退后不再共享**，改动要重新同步。

### 切换供应商不会弄坏技能

CC Switch 的设计原则是**最小侵入**：切换供应商时只改写配置文件里的**连接字段**（endpoint、key、model、协议），以及该供应商自己的兼容开关。**插件、hooks、MCP、Skills、你手写的注释与格式全部保留**。首次改写前会把原文件备份到 `~/.cc-switch/backups/live-first-write/`。

对本技能包而言这意味着：**换模型 / 换中转站之后，`SKILL.md` 与 `references/` 还在原地**，不用重装。

### 与本技能包相关的其它 CC Switch 能力

- **提示词库（Prompts）**：可以把本仓库 `AGENTS.md` 的 §1–§9 存成一条 prompt，一键写进某个项目的 `AGENTS.md` / `CLAUDE.md`。
- **项目（Projects）**：把"某模组仓库的 供应商 + MCP + Skills + prompt 文件"存成一个项目，换项目时一键整套切过去 —— 适合同时维护多条 MC 版本线。
- **用量统计**：本地扫描各工具会话日志，按供应商 / 模型统计 token 与花费；不需要开本地路由也能用。
- **会话管理器**：把会话日志解析成结构化视图（文本 / 思考 / 工具调用 / 图片分开），可以导出 Markdown、复制恢复命令。排障时把某次崩溃排查的会话导出来复盘很有用。

### 常见坑

| 现象 | 原因 / 处理 |
|---|---|
| 切了供应商但 CLI 还是老模型 | Codex / Gemini CLI 需要**重启终端或守护进程**；Claude Code 支持热切换不用重启 |
| 技能装上了但某个工具看不到 | 该工具不支持软链且回退成了复制，但你没勾选那个工具；或该工具的技能目录不在 CC Switch 的默认探测列表里 —— 到"设置 → 应用配置"里改该工具的技能目录 |
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

上传脚本（CurseForge / Modrinth）**放在仓库之外的私有目录**，不要提交进模组仓库，更不要提交进本技能包。把 `<脚本目录>` 换成你的实际路径。

### 6. 凭据

`curseforge_api_key` / `modrinthApiKey` 放在你**全局**的 `~/.gradle/gradle.properties` 里，**不要放进任何仓库**。本技能包的规则是：agent 不读、不搜、不打印、不落盘、不转交外部命令，只允许 Gradle `findProperty` 消费。

### 7. 验证改完没漏

```bash
grep -rn '<你的' . --include='*.md'
```

**占位符只在两处，含义固定，不需要全部替换**：

| 出现位置 | 占位符 | 什么时候要改 |
|---|---|---|
| `AGENTS.md`（抄进你的模组仓库时） | `<JDK 17>` / `<JDK 21>` / `<JDK 25>` / `<JDK 8>` / `<你的代理地址与端口>` / `<你的 git 路径>` / `<你的测试实例前缀>` / `<禁部署标记>` / `<实例目录>` / `<脚本目录>` / `<你的备用反馈页>` / `<9.6.1>` / `<8.11>` | **必须全改**，否则 agent 会照着占位符干活 |
| `references/`（只在你要照着执行某段命令时） | `<你的代理端口>`、`<你的 JDK 路径>`、`<脚本目录>`、`<你的反馈页>` 等 | 按需改；**不执行那段命令就不用动** |
| `SKILL.md` | **无占位符**（只用 `<mod_id>` / `<version>` 这类**变量名**，不是要你填的值） | 不用改 |

判断方法：占位符里带"你的"三个字 = 要按环境填；写成 `<mod_id>` / `<version>` / `<Loader>` = 只是文档里的变量名，照抄时换成自己的值即可。

---

## 怎么用这个技能包

### 自动触发

`SKILL.md` 的 `description` 里写了触发词：新建模组、移植、加版本分支、构建失败、模组不加载、注入失效、多语言失效、发布、上传 CF/MR、mixin、pack.mcmeta、Gradle 档位。正常提问就会命中。

### 手动调用

- Claude Code / Hermes：`/minecraft-mod-dev`
- 其它 agent：直接说"用 minecraft-mod-dev 技能"或"按 SKILL.md 的规则做"

### 典型提问

```
按 SKILL.md 帮我从零建一个 Fabric + NeoForge 的模组骨架，MC 26.3
把这个 1.20.1 的模组扩到 1.21.1，先跑探针再决定要不要新分支
构建报 "not a valid mod file"，按 §10 排障
我要发版了，按 §9 走一遍自检
我的模组在 Forge 上启动就崩，Fabric 正常
```

### 分阶段汇报

技能包要求 agent 汇报时分清：**已构建 / 已部署待验收 / 已验收 / 已上传**。看到它混着说，就把 `AGENTS.md` §9 贴给它。

---

## 更新与贡献

```bash
git pull
```

### 一键推送脚本（`push.ps1` / `push.sh`）

仓库根目录带两个功能等价的一键推送脚本，**只在本地使用、已在 `.gitignore` 中排除，不会进仓库**。它做的事：

1. 检查 git 环境与远端配置（缺 `origin` 就自动加）；
2. **校验 `.gitignore` 生效**：暂存区里出现 `.local/`、`push.*`、`.vscode/`、`*.log` 等就直接失败；
3. 有改动就提交；
4. 推送前若远端分支已存在且本地有基线记录，**比对远端哈希**；
5. 推送后 **`ls-remote` 回读**，确认远端哈希 == 本地 HEAD 才算成功；
6. 把本次哈希写进 `.local/last-push-<branch>.txt`，供下次强推比对。

```powershell
# Windows PowerShell
.\push.ps1                                   # 提交并快进推送当前分支
.\push.ps1 -Message "feat: add porting module"
.\push.ps1 -Force                            # 强推（先比对远端哈希与基线，不一致就拒绝）
.\push.ps1 -DryRun                           # 只打印将要执行的命令
.\push.ps1 -Proxy "http://127.0.0.1:7890"    # 走代理
.\push.ps1 -NoProxy                          # 强制直连
```

```bash
# macOS / Linux / Git Bash
./push.sh                                    # 提交并快进推送当前分支
./push.sh -m "feat: add porting module"
./push.sh -f                                 # 强推（先比对哈希与基线）
./push.sh -d                                 # dry-run
./push.sh -p "http://127.0.0.1:7890"         # 走代理
./push.sh -n                                 # 强制直连
```

**为什么强推也要先比对哈希**：强推会不可逆地覆盖远端。脚本只在"远端哈希 == 上次推送后记录的基线"时才允许覆盖；不一致说明远端被别处更新过，脚本**直接拒绝并退出**，由你人工判断。确认远端是安全的，就手动把该哈希写进 `.local/last-push-<branch>.txt` 再重跑。

> 远端是新建的空仓库时，第一次推送属"新建分支"，没有覆盖风险，脚本会正常放行。
> 退出码：`0` 成功 / `1` 前置检查失败 / `2` 推送失败。

### 贡献规则

- 技能内容以 `SKILL.md` 为准，`references/` 是细节展开。改规则时**先改 `SKILL.md`，再同步 `references/`**，避免两处打架。
- 新增实测结论请带**证据**（`javap` 输出、jar 解包结果、maven 元数据、错误原文），并标注证据强度（实测 / 推断）。
- **不要**提交任何个人环境信息（路径、端口、账号、项目 ID、实例名）。提交前自查：

  ```bash
  grep -rnE '(C:\\|D:\\|/Users/|/home/)' . --include='*.md'
  ```

- 本仓库采用 CC0-1.0：贡献即视为放弃著作权，不要求署名。

---

## 许可证

[CC0 1.0 Universal](LICENSE) —— 公有领域奉献。你可以复制、修改、分发、商用，**无需署名**。

```
To the extent possible under law, the authors have waived all copyright and
related or neighboring rights to this work.
```
