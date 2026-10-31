# 09 Git 与平台上传

> 管什么：每版本独立仓库的分支模型、两种推送策略与强推红线、代理与凭据、CurseForge / Modrinth 上传链路的全部字段与错误原文、Commit Message 规范。
> 本文件与 `references/` 其余模块同属一个技能包，**没有上位总纲**；结论冲突时以一手证据为准。本文件给命令、字段全集、判定手法与踩坑结论。

---

## 9.1 分支模型：一个版本目录 = 一个独立仓库

### 9.1.1 形态与首次建库

**约定**：多版本工程里，每个 MC 版本目录**独立成一个 git 仓库**，各自一个**无关历史的根提交**，彼此不合并、不与 `main` 合并。分支名 = 目录名 = jar 文件名里的 MC 版本 / 区间标签。

首次建库（`bash`）：

```bash
cd <版本目录>
git init -b <version>
git remote add origin <仓库 URL>
git add -A
git commit -m "chore: initial import"
git push -u origin HEAD:refs/heads/<version>
```

- 同一工程里**可能只有个别版本目录已经 `git init`**，其余目录还不是库。这类目录要现场 `git init -b <分支名>` + `git remote add origin <url>` 再推，**不要**假设"模板已经带了仓库"。
- 一个工程的多个版本目录**各自独立成库、各自一个根提交**，推完要**逐条** `git ls-remote` 回读比对本地 `HEAD`，别只验一条就报"全部成功"。
- 更新某版本：在该目录正常 `git add -A` + `git commit`，然后按 §9.1.3 的某一种策略推送。
- 取消本地连接只需删除该目录下的 `.git`，远端分支保留。
- push 走 HTTPS + 系统凭据管理器缓存；提交身份需先 `git config --global user.name/email`——**这是提交署名，与推送认证是两回事**，两者不要互相顶替。

### 9.1.2 首次推送前先补 `.gitignore`（MultiLoader 模板默认没有）

**铁律**：别假设"模板已带 `.gitignore`"。`jaredlll08/MultiLoader-Template` 生成的版本目录**通常没有** `.gitignore`，直接 `git add -A` 会把 `build/`、`build-logic/build/`、`.gradle/`、`.vscode/`、`fabric/build/`、`neoforge/build/`、`neoforge/runs/` 全部提交上去。

`.gitignore` 至少覆盖：

```text
.gradle/
build/
out/
classes/
run/
runs/
libs/
.idea/
*.iml
.vscode/
.eclipse/
.settings/
.classpath
.project
hs_err_*.log
*.hprof
.DS_Store
Thumbs.db
!gradle/wrapper/gradle-wrapper.jar
```

然后**必须**验证 buildlike = 0：

```bash
git add -A
git ls-files | grep -cE "(^|/)(build|\.gradle|run|runs|out|bin|\.idea|\.vscode)/"   # 必须输出 0
```

Windows 上 `core.autocrlf=true` 时索引存 LF、检出一律 CRLF，`git add` 会刷一屏 `LF will be replaced by CRLF` 警告——**正常**，不影响推送内容。

### 9.1.3 两种可选推送策略

各版本分支是**无关历史的根提交**，因此不存在"合并"这一步，只剩"推送"这一步。有两种策略，**按项目实际情况选一种并固定下来**，不要在同一条分支上来回切换。

| 策略 | 命令 | 适用条件 | 风险 |
|---|---|---|---|
| **A. 快进推送**（推荐默认） | `git push origin HEAD:refs/heads/<version>` | 远端分支只由本工作副本推进；本地领先远端且是快进关系 | 无覆盖；远端有他人提交时会被拒（`non-fast-forward`） |
| **B. 强推覆盖**（可选） | `git push --force origin HEAD:refs/heads/<version>` | 本地是**唯一事实来源**，且已确认远端没有需要保留的提交 | **不可逆覆盖**，见 §9.1.4 |

- 选 A 时若被拒（`! [rejected] ... (non-fast-forward)`）：先 `git fetch origin <version>`、`git log --oneline HEAD..origin/<version>` 看清远端多了什么，**不要**为了"推上去"就直接改 `--force`。
- 选 B 时**优先 `--force-with-lease`**（远端被别人更新过会拒绝，比裸 `--force` 安全）；裸 `--force` 只在已按 §9.1.4 比对过哈希、且远端确无待保留提交时使用。
- 无论选哪种，**推送完成后一律用 `ls-remote` 回读远端哈希**与 `git rev-parse HEAD` 比对，不能拿 push 的输出当成功判据（见 §9.3 末尾）。

### 9.1.4 强推红线

1. **只对版本分支强推**。`main` / `master` **禁止强推**（除非用户明确指名要求并承担后果）。版本分支才是强推对象，主干永远不该被覆盖。
2. **推前必须比对远端哈希**（§9.1.5）：`git ls-remote --heads origin <分支>` 拿远端哈希，与该分支**上次推送后记录的哈希**比对，**一致才可直接强推**；**不一致先停手问用户**，不得直接 `--force`。
3. **执行前确认本地确实是最新、最完整的那份**：先 `git status` 看有无未提交改动，先 `git log --oneline -5` 看基线；有未提交改动就先处理干净。
4. **用户未明确下达推送指令时不得推送**；"顺手推一下"一律禁止。
5. **远端原本不存在该分支时，`--force` 等价于新建、无覆盖风险**——但仍要先 `ls-remote` 确认"不存在"，不能凭印象。
6. 推送完成后回报**分支名 + 提交短哈希 + 提交信息**，便于对账；并把「分支 + 哈希 + 提交信息」记进项目台账，作为下次比对的基准。

```bash
# 快进推送（策略 A）
git -c http.proxy=http://127.0.0.1:<你的代理端口> -c https.proxy=http://127.0.0.1:<你的代理端口> \
    push origin HEAD:refs/heads/<version>

# 强推覆盖（策略 B，先按 §9.1.5 比对哈希）
git -c http.proxy=http://127.0.0.1:<你的代理端口> -c https.proxy=http://127.0.0.1:<你的代理端口> \
    push --force-with-lease origin HEAD:refs/heads/<version>
```

### 9.1.5 强推前必须比对远端哈希（实测事故）

- **现象**：强推输出 `+ 1ccf99f...4211dd6 (forced update)`——远端当时停在一个**不属于本工作副本**的提交上（另一份工作副本推的），被 `--force` 原地覆盖。
- **为什么没发现**：旧口径只要求"确认远端是否已有该分支"，**没有要求比对哈希**，于是"分支存在"就直接强推了。
- **取不回来**：本地没有该对象，`git fetch origin <sha>` 报 `couldn't find remote ref <sha>`（托管平台默认不允许按 SHA 拉游离对象）。
- **补救顺序**：① 找到推它的那份工作副本 ⇒ 先 `git push origin HEAD:refs/heads/backup-<sha>` 存成备份分支再处理；② 试提交页 `.../commit/<sha>`（未引用对象会短期保留）；③ 用 IDE 本地历史。
- **规矩**（**铁律**）：
  1. 推前 `git ls-remote --heads origin <分支>` 拿远端哈希；
  2. 与该分支**上次推送后记录的哈希**比对：一致才可直接强推；**不一致先停手问用户**，不要 `--force`；
  3. 每次推送后把「分支 + 哈希 + 提交信息」记进项目台账，作为下次基准。

### 9.1.6 403 权限拒绝：看报错里那个账号名

```text
remote: Permission to <owner>/<repo>.git denied to <account>.
fatal: ... The requested URL returned error: 403
```

这不是网络问题，是**凭据管理器缓存的账号与仓库属主不匹配**。排查：

```bash
printf "protocol=https\nhost=github.com\n\n" | git credential fill | grep '^username='
```

输出谁，就是用谁在推。解决方式二选一：把该账号加成仓库 Collaborator（Write），或改用属主账号的凭据 / PAT。**不要**为此去动全局 `user.name/email`（那是提交署名，与推送认证是两回事）。

### 9.1.7 已知无害现象

- 推送成功后本地可能建不出 `refs/remotes/origin/<version>`（`git branch -vv` 显示 `[origin/<version>: gone]`）。只要 `git config branch.<version>.merge` = `refs/heads/<version>` 写对了，**无参数 `git push` 照样可用**（`git push --dry-run` 会回 `Everything up-to-date`），不必折腾。
- `git fetch` 会自动跟随并拉下远端可达的 tag（如 `v1.0.0`、`v1.1.0`），属正常行为。

---

## 9.2 网络：GitHub 直连不通，必须走本地代理

- 直连报 `Failed to connect to github.com:443 after 21000 ms`；受限网络 / 沙箱自带的代理可能只放行 `api.github.com`、`codeload.github.com`，**不放行 `github.com`**（`git ls-remote` 会得到 `CONNECT tunnel failed, response 502`）。
- 先探测端口是否在听（`powershell`；把 `<端口>` 换成你的客户端实际配置的端口）：

```powershell
foreach ($p in <端口1>,<端口2>) { if (Test-NetConnection 127.0.0.1 -Port $p -InformationLevel Quiet) { "OPEN $p" } }
```

- 推送命令（**先清掉环境注入的代理变量**，免得它们抢先生效）：

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
git -c http.proxy=http://127.0.0.1:<你的代理端口> -c https.proxy=http://127.0.0.1:<你的代理端口> \
    push origin HEAD:refs/heads/<version>
```

```powershell
Remove-Item Env:http_proxy,Env:https_proxy,Env:HTTP_PROXY,Env:HTTPS_PROXY -ErrorAction SilentlyContinue
git -c http.proxy=http://127.0.0.1:<你的代理端口> -c https.proxy=http://127.0.0.1:<你的代理端口> push origin HEAD:refs/heads/<version>
```

- **git 可执行文件用全路径调用**：git 装在含中文或空格的路径下时，PATH 解析在部分终端会乱码，此时显式写全路径（`<你的 git 路径>`）即可，不必改 PATH。
- 沙箱环境可能拦 `.git/objects` 写入（表现为 `git add` / `git commit` 失败）→ 该动作需要非沙箱执行；这不是仓库损坏。

---

## 9.3 GCM 在无 TTY 环境下静默挂住，push 永久卡死（实测）

- **现象**：`git push` 无任何输出、永不返回，看起来像在上传对象。
- **根因**：Git Credential Manager 拿不到交互能力时不报错，而是一直等——先起 `credential-helper-selector get`，再退化成向 `/dev/tty` 要用户名，于是报 `could not read Username for 'https://github.com': No such file or address`（或直接挂住）。Windows 凭据管理器里其实**通常已有可用的 `git:https://github.com` 凭据**，只是 GCM 不去用它。
- **判定手法**：
  1. `Get-CimInstance Win32_Process` 看进程树，长期停在 `credential-helper-selector get` 即为此坑；
  2. 枚举可见窗口确认没弹凭据对话框（真弹了就让人手工填，别乱杀进程）；
  3. `cmdkey /list` 只读**目标名**判断凭据是否存在（**严禁**读值）；Git Bash 会把 `/list` 当路径转换，需 `MSYS_NO_PATHCONV=1 cmdkey /list`；
  4. 网络是否正常要用 `ls-remote` 旁证，不能拿 push 的沉默当网络故障。
- **修复**：非交互推送加 `-c credential.interactive=false`（单用 `GCM_INTERACTIVE` / `GIT_TERMINAL_PROMPT=0` 挡不住挂起）：

```bash
GIT_TERMINAL_PROMPT=0 git -c credential.interactive=false \
    -c http.proxy=http://127.0.0.1:<你的代理端口> -c https.proxy=http://127.0.0.1:<你的代理端口> \
    push origin HEAD:refs/heads/<分支>
```

- **误导性退出码**：`git push ... | tail` 的退出码是 `tail` 的，**exit 0 不等于推送成功**。判定一律以 `ls-remote` 回读的远端哈希 == `git rev-parse HEAD` 为准：

```bash
git -c http.proxy=http://127.0.0.1:<你的代理端口> -c https.proxy=http://127.0.0.1:<你的代理端口> ls-remote --heads origin <分支>
```

---

## 9.4 凭据红线（CF 与 MR 通用）

- CF 键名 **`curseforge_api_key`**；MR 键名 **`modrinthApiKey`**；两者都在**用户全局** `gradle.properties`（`<你的 .gradle>/gradle.properties`）。
- **铁律**：AI **严禁读取、打印、检索该文件**（含个人隐私）；key 绝不写进任何仓库文件，也不得从 `gradle.properties` 解析出来后传给 `curl` / 任何外部命令。**只允许 Gradle 内部读取（`findProperty`）。**
- 文档里"把 key 读进变量交给 curl"这类旧示例**不作为读取授权**；优选已有的 Gradle init 脚本实现（脚本只检查键的**存在性**、不打印）。

---

## 9.5 CurseForge

### 9.5.1 两条链路

| 链路 | 状态 | 说明 |
|---|---|---|
| **CurseGradle 插件** | 配置可保留，**上传不可用** | `de.maxhenkel.cursegradle`（DSL 与 matthewprenger 版一致），**不在 Gradle 插件门户**；上传受 §9.5.2 阻塞 |
| **init 脚本 `cfUpload`** | **采用** | JDK `java.net.http.HttpClient` + **显式** HTTP CONNECT 代理 + legacy 端点 |

### 9.5.2 上传 403 的真相：Cloudflare 人机挑战，且 CurseGradle 内部 HttpClient 完全不读代理（实测）

**现象**：`gradlew :<module>:curseforge` 报 `[CurseForge] HTTP Error Code 403: Forbidden`。

**根因**：CurseGradle 1.5.2 的 `CurseUploadTask.uploadFile()` **自建** Apache HttpClient：`HttpClientBuilder.create()...build()`——它**既不调用 `useSystemProperties()`，也不读 `http(s).proxyHost` / 不读默认 `ProxySelector`**。于是上传请求**直连** `minecraft.curseforge.com`；直连 IP 被 Cloudflare 弹**托管人机挑战**（响应头 `cf-mitigated: challenge`、正文 `<title>Just a moment...</title>`）→ 403。**与 API key 无关。**

**判定证据表**（每条都能独立复现）：

| 实验 | 结果 | 结论 |
|---|---|---|
| `curl -x http://127.0.0.1:<你的代理端口>` POST `/api/projects/<id>/upload-file`（**用无效 token**） | `{"errorCode":403,"errorMessage":"Invalid API token provided."}` | 经代理能**真正打到 API**（不是人机页） |
| JDK `URL.openConnection()`（走默认 `ProxySelector`）POST 同一端点 | 同上（JSON） | Java 侧经代理也没问题 |
| JDK 请求 GET `/api/game/version-types`（带真实 key） | **HTTP 200** + 合法 JSON | 代理线路可用、key 有效 |
| 反射调用 **CurseGradle 自己 shade 的** `HttpClientBuilder.create().build()` GET `version-types` | **HTTP 403 + `<title>Just a moment...</title>`** | 铁证：该 HttpClient **没有**走代理 |
| 打印 key 存在性（root / 扩展 / 任务上） | 全部非空 | 排除"key 没读到" |
| `javap` `CurseGradlePlugin` 常量池 | `UPLOAD_URL = "%s/api/projects/%s/upload-file"`，`apiBaseUrl` 默认 `https://minecraft.curseforge.com` | **legacy 端点仍然有效**，不是"废弃端点" |

**两条已作废的旧推断（勿再重复）**：

1. **已作废**「legacy 端点 `minecraft.curseforge.com/api/*` 已废弃」——**错误**，该端点完全正常。
2. **已作废**「改用 CF 新的官方 API `api.curseforge.com/v1/*` 即可」——该域名在本网络环境**不可用**：直连与走代理都是 `403 / X-Cache: Error from cloudfront`、空正文（CloudFront 层拦截），与代理、密钥都无关；它的公共端点行为也不一致（`/v1/games` 403、根路径 `/` 200）。**不要往这条路上投时间。**

**代理方式的选择（关键）**：

- **不要用 SOCKS5（`socksProxyHost`）**：Java 的 SOCKS 实现会**在本地解析 DNS** 再把 IP 交给代理，代理客户端处于"规则模式"时按 IP 判定为直连 → 直连 Cloudflare → 握手被掐（`javax.net.ssl.SSLHandshakeException: Remote host terminated the handshake`）。
- **用 HTTP CONNECT 代理，并且"显式"指定**：只设 `https.proxyHost` 系统属性**也不够**——`sun.net.spi.DefaultProxySelector` 在**类初始化时**就把 `http(s).proxyHost` 缓存成快照，Gradle 自身往往在 init 脚本执行前已触发过它，于是缓存到的是"无代理"。必须**显式 `ProxySelector.setDefault(...)`** 或（更稳）在请求里显式 `.proxy(...)`。

### 9.5.3 可用方案：`cfUpload` 任务（显式代理 + legacy 端点）

参考实现：`<脚本目录>/curseforge-upload-init.gradle`（内含 `cfProxyCheck` / `cfProbe` / `cfProbeApache` 诊断任务与 `cfUpload` 正式任务）。

**做法要点**：

1. 用 **JDK `java.net.http.HttpClient`** + **显式** `.proxy(ProxySelector.of(new InetSocketAddress('127.0.0.1', <你的代理端口>)))`（HTTP CONNECT → 由代理端解析 DNS，绕开 SOCKS 坑；显式代理绕开"默认 ProxySelector 被缓存"的坑）。
2. **复用模块 `build.gradle` 里已有的 `curseforge { }` 配置**（Project ID / displayName / releaseType / gameVersionStrings / relations），换版本、换模块都不用改脚本——脚本只当"执行器"。
3. 先 `GET /api/game/versions` 把版本**字符串**解析成**数值 ID**（上传 metadata 的 `gameVersions` 是 `int[]`，与 CurseGradle 行为一致）。解析不到就**中止、不盲传**，并打印疑似候选。
4. multipart POST 到 `${apiBaseUrl}/api/projects/{projectId}/upload-file`，表单项 `metadata`（JSON）+ `file`（jar）。
5. **只认响应里的 File ID**：拿到 `{"id":<数字>}` 才算发布完成，并把 ID 记进项目台账。

**显式代理的最小写法**（Groovy，init 脚本内；这是绕开两个坑的关键，可直接抄）：

```groovy
import java.net.InetSocketAddress
import java.net.ProxySelector
import java.net.http.HttpClient

// 端口来源：Gradle 属性（-PproxyPort=...）优先，否则用占位值；按你的环境改
def proxyPort = (findProperty('proxyPort') ?: '<你的代理端口>') as int
def addr = new InetSocketAddress('127.0.0.1', proxyPort)

// 关键 1：显式设默认 ProxySelector（不能只靠 -Dhttps.proxyHost）
ProxySelector.setDefault(ProxySelector.of(addr))

// 关键 2：HttpClient 上再显式指定一次（双保险，绕开"默认 selector 已被缓存"）
def client = HttpClient.newBuilder()
    .proxy(ProxySelector.of(addr))
    .connectTimeout(java.time.Duration.ofSeconds(30))
    .build()
```

- **不要**用 `socksProxyHost`：Java 的 SOCKS 实现会本地解析 DNS，代理按 IP 判直连 → `SSLHandshakeException`。
- **不要**只设 `-Dhttps.proxyHost`：`sun.net.spi.DefaultProxySelector` 在**类初始化时**就把系统属性缓存成快照，Gradle 往往在 init 脚本执行前已触发过它。
- **诊断任务先跑**：`cfProxyCheck` 打印当前生效的 `ProxySelector` 与端口；`cfProbe` / `cfProbeApache` 分别用 JDK `HttpClient` 与 CurseGradle 自带的 Apache 客户端打同一个只读端点——**两者结果不同就说明"代理只对前者生效"**，这正是 §9.5.2 那条铁证的复现方式。

**命令（`powershell`）**：

```powershell
$env:JAVA_HOME='<你的 JDK 路径>'
Set-Location '<版本工程目录>'

# Fabric 端
.\gradlew.bat :<fabric 模块>:shadowJar cfUpload -PcfModule=<mod-id>-fabric `
  -Pcurse_release_type=release `
  -I '<脚本目录>/curseforge-upload-init.gradle' `
  --console=plain --no-daemon --offline

# NeoForge 端（只换 cfModule 与前置构建任务）
.\gradlew.bat :<neoforge 模块>:shadowJar cfUpload -PcfModule=<mod-id>-neoforge `
  -Pcurse_release_type=release `
  -I '<脚本目录>/curseforge-upload-init.gradle' `
  --console=plain --no-daemon --offline
```

> 前置：代理客户端已开启，HTTP 混合端口 `<你的代理端口>` 可连。
> **`cfUpload` 不带 `dependsOn` 构建任务**，因此命令里必须显式带上前置构建（如 `:<模块>:shadowJar`）——**但绑哪个任务不能想当然**，见 §9.5.5 的"产物判定"。否则可能传上旧 jar。
> 常见实现里 `cfProxyCheck` / `cfUpload` 由 init 脚本的 `gradle.rootProject` **只注册在根工程**，上传哪个端用 `-PcfModule=<mod-id>-fabric|<mod-id>-neoforge` 选，**不是** `:fabric:cfUpload` 这种子工程任务路径。命令中的路径若含空格，**必须逐个参数加引号**（真实事故：`-I "…含空格的目录/…"` 被词分割后脚本没加载，而管道退出码掩盖了失败）。

> **`-P` 的值里含点号时，PowerShell 下必须整体加引号**（实测）：`-PcfModule=<mod-id>-26.3` 不加引号会被拆成两段，Gradle 报 `Task '.3' not found in root project '...'`（它把 `.3` 当成了任务名）；写成 `"-PcfModule=<mod-id>-26.3"` 即正常。同一条命令连续对照验证过，不是偶发。

**多版本／多加载器批量上传的两个脚本字段**：

- 把「本次要声明的 MC 版本串」做成**每条记录自己的字段**（例如字段名 `mcv`），不要像早期那样把版本串硬编码在一处：跨区间发布时 `wanted = (cfg.mcv ?: ['<默认版本>']) + loaders + env`，每条记录按分支设计区间**全列**（如 `1.21.9-1.21.11` → `['1.21.9','1.21.10','1.21.11']`）。列出的每一串都会向 `/api/game/versions` 解析，**解析不到即中止、不盲传**。
- 需要**本次不写更新日志**时用 `-PcfChangelog=none` 让 `changelog` 传空字符串（默认仍取仓库根的 changelog 文件）——省得为了留空去改文件内容。
- 一次工程多个区间分批传时，每批只跑一条命令（每条 `cfUpload` 都是一次完整 Gradle 构建，两分半左右），拿到 `File ID` 立刻记进台账再跑下一条。

### 9.5.4 上传 metadata 的字段格式（与 CurseGradle 逐字对齐）

字段名取自 CurseGradle `CurseArtifact` 的 `@SerializedName`（`javap -v` 实测）：

```json
{
  "changelog": "",
  "changelogType": "markdown",
  "displayName": "Example Mod 1.2.0 (Fabric) for Minecraft <MC 版本>",
  "releaseType": "release",
  "gameVersions": [17045, 7499, 9638, 14454],
  "relations": { "projects": [ { "slug": "example-lib", "type": "requiredDependency" } ] }
}
```

- `gameVersions` 是**数值 ID 数组**（不是字符串）；`Java <n>` 由 CurseGradle 的 `CurseArtifact.resolve()` 自动附加，属既有行为。
- `relations.projects[].type` 取值：`requiredDependency` / `embeddedLibrary` / `optionalDependency` / `tool` / `incompatible`。
- 没有前置依赖的加载器端，**`relations` 字段要整个省略**。

**三条实测踩坑（每一条都真实卡过一次）**：

1. **`metadata` 必须是普通表单字段**：`Content-Disposition: form-data; name="metadata"`，**绝对不能带 `filename=`**（带了会被服务端当文件上传处理）→ `{"errorCode":1001,"errorMessage":"Missing field metadata."}`。
2. **`relations` 只能"完全不写"或"至少 1 项"**：写成 `{"projects":[]}` 会被拒：`{"errorCode":1002,...,"Array item count 0 is less than minimum count of 1. Path 'relations.projects'..."}`——所以**一个加载器端若没有任何前置依赖，就不要写 `relations`**。
3. **legacy 端点偶发 HTTP 500**（`{"errorCode":500,...}`）属**服务端瞬时错误**，同一请求重跑即成功（实测首发 500、重跑 200）。**先重跑一次再排查**。

### 9.5.5 CurseGradle 配置细节（若保留配置块）

- `settings.gradle` 的 `pluginManagement.repositories` 加 `maven { url 'https://maven.maxhenkel.de/releases' }`；用前查 `https://maven.maxhenkel.de/releases/de/maxhenkel/cursegradle/maven-metadata.xml` 确认最新版（当时 1.5.2）；各模块 `id 'de.maxhenkel.cursegradle' version '<最新>'`，各自注册 `:<fabric>:curseforge` / `:<neoforge>:curseforge`。
- **产物判定（先判后写，铁律）**：`mainArtifact` 只吃 `File` 或 `AbstractArchiveTask`（不吃 `TaskProvider`），绑错任务名在配置期就炸。动手前先用 init 脚本或 `gradlew :<模块>:tasks` 确认任务清单：
  - **没有 `remapJar` → 绑 `jar`**：某些工程的 Loom 1.16.3 运行时 `loom.dontRemapOutputs()==true`（与 Gradle 属性 `fabric.loom.dontRemap` 相关），fabric 模块根本不注册 `remapJar`，此时 **fabric 与 neoforge 一样绑 `jar`**。
  - 有 `remapJar` 的老工程才绑它；若该任务在 `afterEvaluate` 才出现，把 `mainArtifact` 放进 `afterEvaluate { curseforge.project { ... } }`，依赖行仍可在块外用字符串任务名声明。
  - **不要一律 `shadowJar`**：先确定该工程真实可分发产物是 `jar` / `remapJar` / `shadowJar` 中的哪一个。
- **Gradle 9 + CurseGradle 两个语法坑**：① `project {}` 闭包是 `DELEGATE_FIRST`，裸写 `remapJar` / `jar` 会被当成 CurseProject 属性报 `No such property`；② Gradle 9 移除了 `tasks.xxx` 动态任务属性；唯一两端通用写法是 `tasks.named('xxx').get()`。
- **`environment` 分组必填**：只选 MC 版本 + 加载器会报 `Error Code 1021: ... environment group`，至少要 `addGameVersion 'Client'` 或 `'Server'`（该分组仅这两个版本，id 9638 / 9639）。纯客户端模组只写 `Client` 即可。
- **已上传文件元数据改错了的修正法**（不用删文件重传）：`POST https://minecraft.curseforge.com/api/projects/<projectId>/update-file`，multipart 字段 `metadata`（JSON：`{"fileId":<id>,"changelog":"...","changelogType":"markdown","releaseName":"<新展示名>","gameVersions":[...]}`）+ 头 `X-Api-Token`。token 仍只由 Gradle 持有。

### 9.5.6 changelog 文件规则（违反 = 把内部说明公开到 CF 页面）

**铁律**：文件内容会被 CurseGradle **原样**作为文件更新日志公开。因此**只允许**出现：版本小标题（`### <version> (YYYY-MM-DD)`）+ 本次真实改动条目（Markdown）。

**禁止**写入：`# <Mod> — Changelog` 之类 H1 大标题（CF 页面已有文件名上下文）、AI 维护规则 / 流程说明 / "只写真实改动"这类提示语——**流程规则只活在文档里，不进发布物**。

开发期可用 `## Unreleased` 段累积条目；**执行上传前必须把它改成 `### <version> (日期)`**。文件**不累积历史版本**（历史在 CF 页面与 git），下次发布整体替换为本版本内容。

### 9.5.7 其它 CF 规则

- **Project ID 定位方法**：打开作者的 CurseForge 成员页 `https://www.curseforge.com/members/<作者名>/projects` 找到项目，项目主页 URL 里的那串数字即 Project ID。**项目尚未发布（成员页找不到）则整个推送配置跳过，不猜 ID、不建空配置。**（ID 属项目细节，只记进项目台账。）
- `addGameVersion` 的字符串在执行时向 CurseForge API 实时校验；若新版 MC / 加载器字符串在 CF 尚未收录，任务会明确报错，届时再改字符串，**不要预先发明写法**。
- **执行时机（铁律）**：配置文件可以提前写好，但**只有用户实机测试通过并主动下达指令后才允许执行上传**，任何"顺手推一下"都禁止。
- 成功判据：`HTTP 200` 且正文为 `{"id":<File ID>}`；随后在 CF 项目页确认文件已出现在 Files 列表、展示名与 releaseType 正确，并把「目标 / 产物 / 展示名 / gameVersions / 前置 / File ID」记进项目台账。
- CF 页面直连也可能 403（Cloudflare 人机挑战，与 §9.5.2 同源），读不到 slug 时可改用搜索方式获取。

### 9.5.8 前置依赖必须写进上传 metadata 的 `relations`（实测）

**现象**：模组 jar 的元数据里明明声明了必需前置（Fabric `fabric.mod.json` 的 `depends`、NeoForge `neoforge.mods.toml` 的 `[[dependencies]] type="required"`），但 **CurseForge 项目的 "Relations / Dependencies" 区块是空的** → 启动器**不会显示、也不会自动下载**该前置。

**机制鉴定（实测，非推测）**：

- **CF 不解析 jar 元数据。** 判定方法：把一个 jar 内部声明的依赖项，与它所在 CF 项目依赖页上显示的项对照——
  - 若两者**不一致**（页面上显示的是别的项目，甚至列出 jar 里根本没声明的项目），即证明 CF **没有**读 jar；
  - 反过来说，页面上出现的每一项都只能来自"页面手工设置"或"上传 metadata 的 `relations`"这两个来源。
- 因此"因为文件里已经声明了前置，平台就把该项目从依赖搜索候选里过滤掉"这类说法**没有证据支持**：按该逻辑被隐藏的应当是 jar 里声明的那个项目，而不是页面上实际出现的另一个项目。
- 手工在页面上用搜索框补加**可能搜不到目标项目**（属页面搜索的候选 / 索引行为，会受项目新建时间、收录状态等影响）——**不要**因为它去删 jar 里的依赖声明。

**规矩**：

1. **前置依赖一律写进 CF 上传 metadata 的 `relations.projects`**（`slug` + `type`），不要依赖网页搜索框手工补加。
2. `relations` 只能"**完全不写**"或"**至少 1 项**"；写成 `{"projects":[]}` 会被服务端拒（`errorCode 1002`）。脚本在未传参时应**整个省略**该字段。
3. 类型取值：`requiredDependency` / `optionalDependency` / `embeddedLibrary` / `tool` / `incompatible`；**脚本要对非法类型直接抛错**，别指望服务端给清晰提示。
4. **jar 自身的元数据依赖照旧保留**：那是加载器（Fabric / NeoForge）的硬校验、也是本地模组列表与启动器读的东西；CF 的 `relations` 只负责页面展示与启动器自动下载。**两者并存，不是二选一。**
5. 用法（init 脚本已支持；可把常用前置写成脚本默认值，避免每次忘记）：

   ```powershell
   # 显式声明（slug:type，逗号分隔多项）
   .\gradlew.bat -I <脚本目录>/curseforge-upload-init.gradle cfUpload "-PcfModule=<module>" `
     -PcfReleaseType=<release|beta|alpha> "-PcfRelations=<slug>:requiredDependency"
   ```

   > 值里含**点号或冒号**的参数必须**整体加引号**，否则会被 PowerShell 拆成多段（真实事故：`-PcfModule=<name>-26.3` 被拆成 `-PcfModule=<name>` 与 `.3`，Gradle 报 `Task '.3' not found`）。
6. 上传成功后**回读依赖页**核对（`/relations/dependencies`），确认前置确实出现在页面上；拿到 File ID 一并记进项目台账。
7. 补充：CurseGradle 的 `:<module>:curseforge` **上传不可用**（其自建 HttpClient 不读代理 → Cloudflare 403，见 §9.5.2），所以这类工程即使保留了 `curseforge { }` 配置块，实际发布也一律走 `cfUpload`。

---

## 9.6 Modrinth

**适用**：该模组在 Modrinth 上**已建项目**时才上传；没建项目的模组**不上传，也不要在 MR 上把它选作前置依赖**——MR 不允许依赖不存在的项目。Project ID 属项目细节 → 只记进项目台账。

**凭据与网络**：令牌放用户全局 `gradle.properties` 的 `modrinthApiKey`（红线同 §9.4）；网络走本地 HTTP 代理 `127.0.0.1:<你的代理端口>`。

### 9.6.1 接口要点（`https://api.modrinth.com/v2`，认证头 `Authorization: <PAT>`）

| 目的 | 请求 |
|---|---|
| 查自己的账号 / 项目（**免手工索要 ID**） | `GET /user` → `GET /user/{id}/projects` |
| 查某项目既有版本（判断哪些版本还没传） | `GET /project/{id}/version` |
| 新建版本（上传产物） | `POST /version`（multipart：`data` + `file`） |
| 改既有版本的属性（**换不了文件本体**） | `PATCH /version/{id}`，body 如 `{"version_type":"release"}` |

```json
{
  "project_id": "<项目 ID>",
  "version_number": "1.6.0",
  "name": "1.6.0",
  "changelog": "",
  "dependencies": [],
  "game_versions": ["<MC 版本>"],
  "loaders": ["fabric"],
  "version_type": "release",
  "featured": false,
  "status": "listed",
  "file_parts": ["file"],
  "primary_file": "file"
}
```

- **一个版本只能一个加载器**：fabric / neoforge 必须**分别发两个 version**（这点与 CF"一个文件带多个 loader 标签"完全不同）。
- `version_type`：`release` / `beta` / `alpha`，与 CF 的 `releaseType` 语义对齐。
- `changelog` 允许空字符串。
- 无前置依赖写 `"dependencies": []`——MR 与 CF 不同，**空数组合法**。

### 9.6.2 `environment`：版本级字段

MR 的"客户端 / 服务端"**不是** CF 那种 gameVersion 标签，而是 **version 对象上的 `environment` 字段**。**请求体只接受英文枚举**，界面上的中文只是前端本地化：

| 枚举值（请求体） | 前端中文标签 |
|---|---|
| `client_and_server` | 客户端和服务端（在客户端和服务器端均具备部分功能） |
| `client_only` | 仅客户端 |
| `client_only_server_optional` | 服务端可选（= 客户端必需） |
| `singleplayer_only` | 仅单人游戏 |
| `server_only` | 仅服务端 |
| `server_only_client_optional` | **客户端可选**（= 服务端必需） |
| `dedicated_server_only` | 仅专用服务器 |
| `client_or_server` | 两者可选，安装在任一端效果都相同 |
| `client_or_server_prefers_both` | **两者可选，两端均安装效果最佳** |

（响应里还会多出一个 `unknown`；请求体只有上表 9 个。）

**推荐口径**（与 CF 的 environment 语义对齐）：服务端模组 = `server_only_client_optional`（等价 CF `Server`）；双端库模组 = `client_or_server_prefers_both`（等价 CF `Client` + `Server`）；**已上传的版本不回溯修改**。

### 9.6.3 最大的坑：PowerShell 的 `Invoke-RestMethod -Form` 发不出 MR 能解析的 `data`

现象：`POST /version` 返回 `400 {"error":"invalid_input","description":"Error while parsing JSON: expected value at line 1 column 1"}`——服务端把 `data` 部件读成了空 / 非 JSON。

**改法：用 `curl.exe`**，把 JSON 写成临时文件，用 `<` 表示"字段值取自文件"，并显式声明部件类型：

```powershell
# 1) data JSON 落临时文件（UTF-8 无 BOM）
[System.IO.File]::WriteAllText($pf, $json, [System.Text.UTF8Encoding]::new($false))

# 2) 上传（$tok 只进变量、不打印）
& curl.exe -sS -X POST 'https://api.modrinth.com/v2/version' `
  -H ("Authorization: " + $tok) `
  --proxy http://127.0.0.1:<你的代理端口> `
  -F ("data=<" + $pf + ";type=application/json") `
  -F ("file=@" + $jar)

# 3) 清理临时文件
Remove-Item $pf -Force
```

- `-F "data=<文件;type=application/json"` 两个点缺一不可：`<` = 字段值取自文件（不是文件上传），`;type=` = 部件类型必须是 `application/json`。
- **成功判据**：返回体含 `"id"` 且 `"status":"listed"`；把 version id 记进项目台账。

**把已发布版本从 beta 补正为 release（不重传文件）**：

```powershell
Invoke-RestMethod -Method Patch -Uri ("https://api.modrinth.com/v2/version/" + $id) `
  -Headers @{ Authorization = $tok } -Proxy $proxy `
  -ContentType 'application/json' -Body '{"version_type":"release"}'
```

> `PATCH` **只改属性、换不了产物本体**；要换 jar 只能删掉版本重发——**删除是破坏性操作，必须先问用户**。

### 9.6.4 init 脚本三任务

自包含 init 脚本（`<脚本目录>/modrinth-upload-init.gradle`），**不引入第三方上传插件**（与 `cfUpload` 同思路：显式 HTTP CONNECT 代理 + JDK `HttpClient` + 手工拼 multipart，从而绕开"`data` 部件类型"的坑）：

| 任务 | 作用 |
|---|---|
| `mrProxyCheck` | 确认代理已生效（应打印 `MrProxySelector` + `127.0.0.1:<你的代理端口>`） |
| `mrVersions` | **只读**拉 `GET /project/{id}/version`，列出既有版本（判断本次要传哪些 / 要补传哪些） |
| `mrUpload` | 正式上传（`POST /version`，multipart：`data`(application/json) + `file`） |

```powershell
$env:JAVA_HOME='<你的 JDK 路径>'
Set-Location '<模组的对应版本目录>'
$s='<脚本目录>/modrinth-upload-init.gradle'

# 1) 看项目已有版本（对比本地 build/libs 决定传哪些）
.\gradlew.bat mrVersions -PmrProject=<项目ID> -I $s --console=plain --no-daemon
# 2) 上传：fabric / neoforge 各跑一次（一个 version 只能一个加载器）
.\gradlew.bat mrUpload -PmrModule=<表内键名> -Pmr_version_type=release -I $s --console=plain --no-daemon
```

- 脚本内是一张表，**换版本时改表里的 `jar` 与 `version`（必须手工同步两处，与 CF 脚本一样）**。
- 安全：脚本只检查 `modrinthApiKey` 的**存在性**、不打印；上传成功会打印 `★ version id`，记进项目台账。
- 未过审 / 未建 MR 项目的模组：**不写条目、不上传、也不能被选作前置依赖**；过审后照抄条目补行。

### 9.6.5 与 CurseForge 的差异速查

| 项 | CurseForge | Modrinth |
|---|---|---|
| 令牌键名 | `curseforge_api_key` | `modrinthApiKey` |
| 认证头 | `X-Api-Token` | `Authorization` |
| 加载器区分 | 同一文件带 loader 标签 | **每个加载器单独一个 version** |
| 前置依赖 | `relations.projects`（为空则整个字段省略） | `dependencies` 数组（**空数组合法**） |
| 上传部件 | `metadata`（普通字段）+ `file` | `data`（**必须 application/json**）+ `file` |
| 改已发布属性 | `POST /api/projects/{id}/update-file` | `PATCH /v2/version/{id}` |
| 查"漏传了哪些版本" | CF 项目页 Files 列表 | `GET /project/{id}/version`（对比本地 `build/libs`） |
| 客户端 / 服务端口径 | `Client` / `Server` 作为 gameVersions 标签 | 版本级 `environment` 枚举 |

---

## 9.7 Commit Message 编写规范

> **铁律**：提交信息一律英文、Conventional Commits 格式、只描述实际发生的修改。本节规则完整保留，可直接当检查表用。

### 9.7.1 基本原则

1. **必须使用英文**：`type`、`scope`、描述、Body、Footer 均用英文；**不得混用中文**。代码中的类名、方法名、注册名、Mod ID 等专有名称保持原样。
2. **遵循 Conventional Commits 格式**：

   ```text
   <type>(<scope>): <description>
   ```

   `scope` 可以省略：

   ```text
   <type>: <description>
   ```

3. **语言规范、自然、简洁**：用符合软件工程习惯的英文；优先简短准确的动词；避免冗长技术说明；不用营销式、宣传式或夸张表达；不为了显得专业而堆砌术语。
4. **不得出现 AI 生成痕迹**：必须像开发者实际提交时写的内容，而不是 changelog 或技术报告。禁止：

   ```text
   Implement a comprehensive...
   Improve and enhance...
   Various improvements and fixes
   Update several components...
   Refactor the codebase to improve maintainability
   Make some necessary changes
   Address various issues
   Enhance overall performance and stability
   ```

5. **只描述实际发生的修改**：不得虚构、不得夸大范围、不得根据文件名或任务目标猜测没有发生的改动；无法确定具体修改内容时，先分析代码变更再写。
6. **描述"这次提交做了什么"**，不描述开发过程、思考过程或空泛效果。

   推荐：

   ```text
   fix(recipe): prevent duplicate recipe registration
   ```

   不推荐：

   ```text
   fix(recipe): improve recipe registration logic and make it more reliable
   ```

### 9.7.2 Type 规范

| Type | 使用场景 | 示例 |
|---|---|---|
| `feat` | 新增功能、方块、物品、实体、GUI、游戏机制等 | `feat: add copper tools` |
| `fix` | 修复 Bug、崩溃、错误行为等 | `fix: prevent crash during chunk unload` |
| `perf` | 性能优化 | `perf: reduce multiblock tick overhead` |
| `refactor` | 重构代码，不改变实际功能 | `refactor(registry): simplify item registration` |
| `build` | Gradle、Loader、依赖、构建配置等 | `build: update NeoForge version` |
| `ci` | CI/CD、GitHub Actions 等 | `ci: add CurseForge publishing workflow` |
| `docs` | README、文档、注释等 | `docs: update supported Minecraft versions` |
| `style` | 格式调整，不改变代码逻辑 | `style: format registry classes` |
| `test` | 新增、修改或修复测试 | `test: add recipe loading tests` |
| `chore` | 其他维护性修改 | `chore: update mod metadata` |
| `revert` | 回滚之前的提交 | `revert: revert multiblock optimization` |

**Type 选择原则**：按**实际代码变化的性质**判断，不按文件所在目录判断。

- 改 `build.gradle` 添加新功能依赖 → `build`
- 改机器逻辑以修复崩溃 → `fix`
- 改机器逻辑以降低 Tick 开销 → `perf`
- 改注册代码但功能不变 → `refactor`
- 新增机器 → `feat`

### 9.7.3 Scope 规范

`scope` 描述此次修改影响的具体模块；可选，但多加载器项目推荐使用。

**按 Loader / Platform**：`common` / `fabric` / `neoforge` / `forge` / `quilt`

```text
feat(common): add item registration helper
fix(fabric): fix key binding registration
fix(neoforge): fix event bus registration
```

**按功能模块**：`block` / `item` / `entity` / `recipe` / `registry` / `world` / `network` / `client` / `server` / `gui` / `render` / `data` / `config` / `command`

```text
feat(block): add copper machine
fix(recipe): prevent duplicate recipe registration
refactor(registry): simplify entity registration
fix(network): fix server packet handling
```

**选择原则**：简短、稳定、能明确指出修改位置、与项目现有模块划分一致。不要为了每个 Commit 都加 Scope 而强行创造 Scope。

```text
fix(code): fix the code      # 不推荐
fix: fix machine crash       # 没有有效 scope 时，这样更好
```

### 9.7.4 Description 规范

1. **必须英文**。`feat: add copper tools` ✔；`feat: 添加铜制工具` ✘。
2. **用简洁动词**：`add` / `remove` / `update` / `rename` / `move` / `fix` / `prevent` / `allow` / `support` / `replace` / `simplify` / `refactor` / `optimize`。

   ```text
   feat: add copper tools
   fix: prevent duplicate recipe registration
   refactor: simplify block registration
   perf: reduce entity lookup overhead
   ```

3. **不以句号结尾**：`fix: prevent crash during chunk unload` ✔；末尾加 `.` ✘。
4. **小写开头**（专有名词、类名、Mod ID 按实际名称保留大小写）：

   ```text
   feat: add copper tools          # 推荐
   feat: Add Copper Tools          # 不推荐
   build: update NeoForge to 21.1.77
   ```

5. **不用模糊描述**：`fix: fix some issues` / `chore: make some changes` / `refactor: improve code` 都属不合格，应写成 `fix: prevent crash when loading invalid recipes` 这类具体描述。

### 9.7.5 长度与 Body

- Description 尽量控制在一行内。反例：`fix(recipe): fix the recipe registration system to prevent recipes from being registered multiple times during reload`——复杂内容应放 Body，而不是塞进 Description。
- **只有在 Description 无法充分说明时才加 Body**：

```text
feat(machine): add automatic smelting

- Add fuel handling
- Add processing progress
- Add redstone control
```

Body 应当：只描述实际修改、用简洁英文、说明重要实现细节、不重复 Description、不写开发过程、不写 AI 式总结。

不要写：

```text
This change introduces a comprehensive and robust automatic smelting
system designed to improve the overall gameplay experience.
```

应直接写：

```text
- Add fuel handling
- Process one item per cycle
- Add redstone control
```

### 9.7.6 Breaking Change 规范

以下情况可能影响存档、世界数据或现有 Mod 环境，应标为 Breaking Change：

- 修改 Mod ID / Namespace
- 删除已注册的方块、物品或实体
- 修改已有注册对象的 Registry Name
- 修改 NBT / DataComponent 结构导致旧数据无法读取
- 修改网络数据格式导致客户端与服务端不兼容
- 提高最低 Minecraft 版本
- 提高最低 Loader 版本
- 删除公开 API
- 修改可能导致旧存档无法正常加载的核心数据结构

用 `!`：

```text
feat!: rename mod id
```

或用 Footer：

```text
feat: rename mod id

BREAKING CHANGE: old world data using the previous mod id is no longer compatible
```

需要说明迁移方式时明确写出：

```text
feat!: rename mod id

BREAKING CHANGE: old world data using the previous mod id is no longer compatible
Migration is required before loading existing worlds.
```

### 9.7.7 Issue / PR 关联

```text
Closes #42
Fixes #87
```

```text
fix(client): prevent item model flickering

Adjust the render layer ordering for translucent models.

Fixes #87
```

只有确实关联 Issue 时才添加，**不要虚构 Issue 编号**。

### 9.7.8 常见示例

| 类别 | 示例 |
|---|---|
| 新功能 | `feat: add copper tools` / `feat(block): add copper machine` / `feat(gui): add machine configuration screen` |
| Bug 修复 | `fix: prevent crash during chunk unload` / `fix(recipe): prevent duplicate recipe registration` / `fix(client): fix item model flickering` |
| 性能优化 | `perf(entity): reduce entity lookup overhead` / `perf(world): reduce block entity tick cost` |
| 重构 | `refactor(registry): simplify item registration` / `refactor(network): reorganize packet handlers` |
| 构建 | `build: update NeoForge to 21.1.77` / `build: update Gradle configuration` |
| CI | `ci: add CurseForge publishing workflow` |
| 文档 | `docs: update supported Minecraft versions` |
| 格式 | `style: format registry classes` |
| 测试 | `test: add recipe loading tests` |
| 杂项 | `chore: update mod metadata` |
| 回滚 | `revert: revert multiblock optimization` |

### 9.7.9 AI 生成 Commit Message 的额外要求

1. **先分析 Diff，再生成 Commit**：不得仅根据任务名称、Issue 标题或文件名生成。应先判断：改了什么、为什么改、是否新增功能、是否修复 Bug、是否重构、是否影响兼容性、是否属 Loader 特定代码、是否存在 Breaking Change。
2. **不要总结得比 Diff 更多**：只改了一个注册方法就写 `refactor(registry): simplify item registration`，不要写成 `refactor: overhaul the entire registration system`。
3. **不要主动添加不存在的信息**：Issue 编号、性能提升百分比、测试结果、用户影响、兼容性结论、性能数据、修复原因——除非能从实际修改或用户提供的信息中确认。
4. **不要生成 AI 风格总结**，禁止这些模板化表达：

   ```text
   improve overall stability
   enhance code quality
   improve maintainability
   improve overall performance
   ensure proper functionality
   implement comprehensive support
   provide better compatibility
   address various issues
   make necessary changes
   update related components
   ```

   除非后面跟了明确、具体的技术对象。

5. **优先最短且准确的表达**：实际修改是"修复客户端启动时因缺少纹理导致的崩溃"，就写 `fix(client): prevent crash from missing texture`，不要写成 `fix(client): improve texture handling to ensure better stability and prevent potential crashes during client initialization`。
6. **不要为了完整强行加 Body**：简单修改只需一行，不要附三行重复内容。

### 9.7.10 生成顺序与最终格式

1. 分析实际代码 Diff → 2. 判断修改性质 → 3. 选 `type` → 4. 判断是否需要 `scope` → 5. 用简洁自然的英文描述实际修改 → 6. 判断是否 Breaking Change → 7. 必要时加 Body → 8. 有明确 Issue 才加 Footer → 9. 检查是否存在中文 → 10. 检查模板化 / 夸张 / AI 风格表达 → 11. 检查是否描述了 Diff 中不存在的内容 → 12. 输出。

```text
<type>(<scope>): <description>
```

复杂提交：

```text
<type>(<scope>): <description>

<body>

<footer>
```

**最终要求**：准确、简洁、自然、专业、英文、符合 Conventional Commits，并且看起来像真实开发者编写的提交，而不是 AI 生成的总结。

---

## 9.8 错误原文速查（Git / 平台）

按报错原文查，比按现象猜快得多。每一行都给了"它到底在说什么"和"下一步查哪里"。

| 错误原文 | 真正含义 | 定位 / 处理 |
|---|---|---|
| `Failed to connect to github.com:443 after 21000 ms` | 直连被墙 / 代理未生效 | §9.2：先探测端口，再显式带 `-c http.proxy=` |
| `CONNECT tunnel failed, response 502` | 上游代理不放行 `github.com` | 换放行该域名的代理；`api.github.com` / `codeload.github.com` 放行不代表 `github.com` 放行 |
| `could not read Username for 'https://github.com': No such file or address` | GCM 在无 TTY 下退化成要 tty 输入 | §9.3：加 `-c credential.interactive=false` |
| `remote: Permission to <owner>/<repo>.git denied to <account>.` + `403` | 凭据缓存的账号不是仓库属主 | §9.1.6：`git credential fill` 看 username，别动 `user.name` |
| `! [rejected] ... (non-fast-forward)` | 远端有你没有的提交 | 先 `fetch` + `log HEAD..origin/<分支>` 看清差异，别直接改 `--force` |
| `+ <sha>...<sha> (forced update)` | 强推已生效 | 立刻按 §9.1.5 核对这是不是预期覆盖；不是就找备份 |
| `couldn't find remote ref <sha>` | 托管平台不允许按 SHA 拉游离对象 | §9.1.5：只能从原工作副本 `push` 成 backup 分支 |
| `[CurseForge] HTTP Error Code 403: Forbidden` | CurseGradle 自建 HttpClient 没走代理 → Cloudflare 人机挑战 | §9.5.2：改用 `cfUpload`（显式代理 + legacy 端点） |
| `{"errorCode":1001,"errorMessage":"Missing field metadata."}` | `metadata` 部件带了 `filename=`，被当文件上传 | §9.5.4 第 1 条 |
| `{"errorCode":1002,...,"Array item count 0 is less than minimum count of 1. Path 'relations.projects'...}` | 写了 `{"projects":[]}` | §9.5.4 第 2 条：无前置就整个省略 `relations` |
| `{"errorCode":500,...}`（legacy 上传） | 服务端瞬时错误 | 原样重跑一次再排查 |
| `Error Code 1021: ... environment group` | 没选 `Client` / `Server` 环境分组 | `addGameVersion 'Client'` 或 `'Server'`（id 9638 / 9639） |
| `No such property: remapJar for class: ...CurseProject` | `project {}` 闭包 `DELEGATE_FIRST`，裸写任务名被当属性 | §9.5.5：`tasks.named('xxx').get()` |
| `Task '.3' not found in root project` | `-P` 值含点号未整体加引号，被 shell 拆段 | 写成 `"-PcfModule=<mod-id>-26.3"` |
| `400 {"error":"invalid_input","description":"Error while parsing JSON: expected value at line 1 column 1"}` | MR 的 `data` 部件为空 / 非 JSON | §9.6.3：`curl.exe` + `-F "data=<file;type=application/json"` |
| `Invalid API token provided.` | token 无效（**说明线路是通的**） | 反向判据：拿到这条说明代理没问题，问题在 key |

**判读原则**：把"线路问题"和"凭据 / 格式问题"分开看。`Invalid API token provided.` 与 `Just a moment...` 是两类完全不同的失败——前者证明请求已打到 API，后者证明根本没打到。

---

## 9.9 上传前产物核对（比命令更容易出错的一步）

上传链路的失败里，相当一部分不是命令写错，而是**传错了文件**。执行上传前逐项过：

| 核对项 | 怎么查 | 不合格的后果 |
|---|---|---|
| 版本号与 jar 名同源 | 读 `gradle.properties` 的版本，对比 `build/libs` 里的文件名 | 平台展示名与实际内容不符，无法回溯 |
| jar 是**本次构建**产物 | 看 mtime，必须晚于本次构建开始时间 | 传上旧 jar，玩家拿到旧行为 |
| 改动真的在包里 | 解包 jar 回读关键文件（`fabric.mod.json` / `mods.toml` / 语言文件 / 主类） | "构建成功"但内容没更新 |
| 同一版本号只有一份 jar | 搜 `build/libs` 有无同名不同内容的产物 | 同版本号对应两份不同内容，无法对账 |
| 元数据里的展示名 | 与 jar 文件名同源 | CF 展示名与产物名对不上 |
| 前置依赖 | `relations` 里已声明，且上传后回读依赖页 | 启动器不显示、不自动下载前置 |
| 目标加载器 | `-PcfModule` 选的是要传的那一端 | 把 Fabric 包传到 NeoForge 标签下 |
| 代理可用 | 先跑 `cfProxyCheck` / `mrProxyCheck` | 403 或挂起，浪费一整轮构建 |

**铁律**：`cfUpload` / `mrUpload` **不带 `dependsOn` 构建任务**，所以"前置构建"必须显式写在同一条命令里；而且**绑哪个任务不能想当然**（`jar` / `remapJar` / `shadowJar` 因工程而异，见 §9.5.5）。最省事的做法是先跑一次 `gradlew :<模块>:tasks` 看清任务清单，再拼命令。

---

## 9.10 多区间 / 多加载器批量发布编排

一次要发多个 MC 区间、多个加载器时，**编排方式比单条命令更容易出错**。可行口径：

1. **每条记录自带版本串**：把"本次要声明的 MC 版本串"做成配置记录里的字段（如 `mcv`），不要在多处硬编码；跨区间发布时按 `wanted = (cfg.mcv ?: [<默认版本>]) + loaders + env` 组装。每条记录按该分支设计的区间**全列**（`1.21.9-1.21.11` → `['1.21.9','1.21.10','1.21.11']`），列出的每一串都会向 `/api/game/versions` 解析，**解析不到即中止、不盲传**。
2. **一次只跑一条命令**：每条 `cfUpload` 都是一次完整 Gradle 构建（两分半左右）。拿到 `File ID` 立刻记进台账，再跑下一条；不要并行发同一工程的多次上传（会抢同一个 `build` 目录与同一个 daemon）。
3. **加载器分别发**：CF 一个文件可带多个 loader 标签，但 MR **一个 version 只能一个加载器**，fabric / neoforge 必须各自一条。
4. **changelog 留空用参数**，不要为了留空去改文件内容：`-PcfChangelog=none` 传空字符串（默认仍取仓库根的 changelog 文件）。
5. **发布完成后逐条回读**：CF 按 File ID 在 Files 列表确认，MR 按 version id 确认 `status: "listed"`；**不要只验一条就报"全部成功"**（与 §9.1.1 的 `ls-remote` 逐条回读同一条纪律）。

**台账最小字段**（每条发布记录一行，缺字段就无法回溯）：

| 字段 | 示例形态 |
|---|---|
| 目标 | `<MC 版本或区间>` + `<加载器>` |
| 产物 | jar 文件名（含版本号） |
| 展示名 | 平台上的 `displayName` |
| `gameVersions` | 数值 ID 数组（或声明用的字符串串） |
| 前置 | `relations` 里声明的 slug + type，无则写"无" |
| 平台 ID | CF File ID / MR version id |
| 分支 + 哈希 | 对应的 git 分支与推送后回读的提交哈希 |

**发布后的核对顺序**（三步都要做，不能只看上传命令的返回）：

1. **命令层**：确认返回 `HTTP 200` 且正文含 `{"id":<数字>}`；没有 ID 就当没传。
2. **页面层**：CF 项目页 Files 列表能看到新文件、展示名正确、`releaseType` 正确；有前置的去 `/relations/dependencies` 回读确认前置已出现在页面上。MR 侧确认 `status: "listed"`、`loaders` 只含一个加载器。
3. **台账层**：把「目标 / 产物 / 展示名 / gameVersions / 前置 / File ID / version id / 分支 + 哈希」写下来。**写完这一步才算发布结束**。

**失败的常见分层**（按这个顺序排查最快，别一上来就改脚本）：

| 层 | 表现 | 先查什么 |
|---|---|---|
| 代理层 | 403、`Just a moment...`、挂起、连接超时 | `cfProxyCheck` / `mrProxyCheck`；代理是否真的放行该域名 |
| 凭据层 | `Invalid API token provided.`（说明线路通了） | key 是否存在（只查存在性，**不读值**） |
| 格式层 | `errorCode 1001` / `1002` / `400 invalid_input` | `metadata` 是否带 `filename=`、`relations` 是否空数组、`data` 部件类型是否 `application/json` |
| 数据层 | `errorCode 1021`（environment group）、版本串解析不到 | 是否选了 `Client` / `Server`；MC 版本串是否已被平台收录 |
| 产物层 | 传上了旧 jar、加载器标签挂错 | 前置构建任务、`-PcfModule` / `-PmrModule` 的取值 |
| 服务端层 | `errorCode 500` | 原样重跑一次，多半自己好了 |

---

## 9.11 自检清单

**Git / 分支**

- [ ] 每个版本目录独立成库，分支名 = 目录名；`.git` 只在本版本目录内。
- [ ] `.gitignore` 已覆盖 `build/`、`.gradle/`、`run(s)/`、`out/`、`.idea/`、`.vscode/` 等；`git ls-files` 计数 **buildlike = 0**。
- [ ] 推送策略（快进 / 强推）已选定并固定；**没有对 `main` / `master` 强推**。
- [ ] 强推前已 `git ls-remote --heads origin <分支>` 拿哈希，并与上次记录的哈希比对一致。
- [ ] 本地 `git status` 干净、`git log --oneline -5` 基线正确。
- [ ] 用户已**明确下达推送指令**。
- [ ] 推送命令带 `-c credential.interactive=false`；路径 / 含点号的值逐个加引号。
- [ ] 推送后用 `ls-remote` 回读哈希并与 `git rev-parse HEAD` 比对；回报分支名 + 短哈希 + 提交信息。
- [ ] 「分支 + 哈希 + 提交信息」已记进项目台账。

**凭据**

- [ ] **未**读取、打印、检索用户全局 `gradle.properties`；key 只经 Gradle `findProperty` 消费。

**CurseForge**

- [ ] changelog 文件只有版本小标题 + 条目，无 H1 模组名标题、无流程文字；`Unreleased` 已改成本次版本号与日期。
- [ ] 成员页能找到项目、Project ID 正确；未发布项目**未**建空配置。
- [ ] 代理已开启、`127.0.0.1:<你的代理端口>` 可连；`cfProxyCheck` 通过。
- [ ] 命令里显式带了正确的**前置构建任务**（不是想当然的 `shadowJar`），路径参数逐个加引号。
- [ ] **前置依赖已写进上传 metadata 的 `relations.projects`**（不是靠页面搜索框手工补），上传后在 `/relations/dependencies` 回读确认。
- [ ] 拿到 **File ID** 并已记录。

**Modrinth（仅当该模组在 MR 有项目时）**

- [ ] 已用 `GET /project/{id}/version` 对比本地 `build/libs`，确认本次要传的版本与**补传的历史版本**清单。
- [ ] fabric / neoforge **分别**建 version；`environment` 枚举取值正确。
- [ ] 用 `curl.exe` + `-F "data=<payload.json;type=application/json"` 上传（`Invoke-RestMethod -Form` 必 400）。
- [ ] 没建 MR 项目的模组**不上传**、也**不作为 MR 前置依赖**。
- [ ] 拿到 **version id** 并已记录；`version_type` 与用户要求一致。

**Commit**

- [ ] 全英文、Conventional Commits、`type` 选对、`scope` 有效或省略。
- [ ] 描述小写开头、无句号、具体到实际修改；没有模板化 / 夸张 / AI 风格表达。
- [ ] 没有虚构 Issue 编号、性能数字、测试结论。
- [ ] Breaking Change 已用 `!` 或 `BREAKING CHANGE:` 标注。
