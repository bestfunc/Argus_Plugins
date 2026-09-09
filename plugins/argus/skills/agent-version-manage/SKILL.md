---
name: agent-version-manage
display_name: Agent 版本管理
description: 管理 Argus Agent 的版本包与远程升级：上传安装包到版本库（本地 MCP 直传）、列版本、查更新记录、推送升级到指定 agent，以及配置 agent 上的三方软件 MCP 服务。查询类是 🟢 L1；配置三方软件是 🟡 L2；**push_agent_update 是 🔴 L3 —— 推错包会让 agent 起不来且只能到现场救**，推之前必须先确认目标机器证书落盘。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__get_agent_config,mcp__argus__run_safe_command,mcp__argus__list_agent_versions,mcp__argus__list_update_records,mcp__argus__push_agent_update,mcp__argus__list_host_software,mcp__argus__describe_host_software,mcp__argus__configure_host_software,mcp__argus__remove_host_software,mcp__plugin_argus_argus_files__upload_agent_version
---

# Agent 版本管理

管理 Argus Agent 的安装包与远程升级。

> ⚠️ **本 skill 里有一个能把机器推成砖的操作。** 动 `push_agent_update` 之前，
> 先读完下面「推错包 = 现场救砖」那一节，并按流程确认目标机器的证书已落盘。
> 其余工具都是安全的。

## 可用工具

| 工具 | 所在 MCP | 等级 | 说明 |
|------|---------|------|------|
| `list_agents` | `argus:` | 🟢 L1 | 定位目标设备（先做这步，别猜 agent_id） |
| `list_agent_versions` | `argus:` | 🟢 L1 | 列版本库里已上传的安装包，拿 `version_id` |
| `list_update_records` | `argus:` | 🟢 L1 | 查升级推送记录与最终状态 |
| `argus-files:upload_agent_version` | `argus-files:` | — | 把本机的安装包传进版本库（HTTP 直传，需管理员账号） |
| `push_agent_update` | `argus:` | 🔴 **L3** | **推送升级到指定 agent，每次都要邮箱审批** |
| `list_host_software` / `describe_host_software` | `argus:` | 🟢 L1 | 看某台现场上配了哪些三方软件 MCP 服务 |
| `configure_host_software` | `argus:` | 🟡 L2 | 新增/修改一条三方软件 MCP 配置（含目标地址与凭证） |
| `remove_host_software` | `argus:` | 🟡 L2 | 删除一条三方软件 MCP 配置 |

L2/L3 的邮箱验证码协议见 `/argus:mcp-authorization`。**铁律：验证码必须等用户给，不要自己编。**

---

## 推错包 = 现场救砖（读完再动 push）

`push_agent_update` 定成 L3 不是走形式，是有真实事故形态的：

**机理.** v1.31+ 的分发包**不嵌证书**，编译期塞的是占位符，证书靠 enrollment 落到磁盘。
如果目标机器还是 **legacy 模式**（证书本来嵌在 exe 里，磁盘上 `C:\ProgramData\Argus\certs`
是空的），装上新包后 Agent 启动时 `loadEmbeddedTLS` 会撞上那个占位符，
直接 `log.Fatalf` 退出 —— **进程根本起不来**。

**为什么救不回来.** 同一台机器的 **SYSTEM 服务实例**与**用户会话实例共用同一个 exe**。
换句话说，你没有"另一个还活着的实例"可以当救援通道：exe 一换，两个实例一起死。
Agent 死了，Argus 就再也够不着这台机器 —— 只能**到现场重装**。

产线机器"到现场"通常意味着停线、排班、跑一趟。所以：

- **推之前必须确认目标机器 `C:\ProgramData\Argus\certs` 里有证书**（下面有查法）。
- 没有证书就**先让它 enroll 或刷新证书让证书落盘**，确认落盘后再推。
- **先推一台验证，起来了再铺开**，别一次推一片。
- `push_agent_update` **不接受"全部"或通配**（`*` / `all` / `全部` 会被直接拒），
  必须逐台显式列 `agent_ids` —— 这个限制是故意的，别想办法绕。

---

## 标准流程

```
1. list_agents                        定位目标设备，记下 agent_id
2. argus-files:upload_agent_version   把包传进版本库（已在库里可跳过）
3. list_agent_versions                确认版本在库、拿 version_id
4. run_safe_command 查 certs 目录     ★ 确认目标机器证书已落盘
5. push_agent_update（先 1 台）        L3，邮箱审批
6. list_update_records + list_agents   确认那台真的起来了、版本变了
7. 重复 5-6 铺开剩下的机器
```

### 1. 定位目标设备

```
list_agents()
```

拿到 `agent_id` 与在线状态。模糊描述（"116 服务器"、"宝适 JAC 老线"）怎么匹配，
见 `/argus:agent-inventory`。

**注意 agent_id 后缀**：形如 `windows-abc123-用户名` 的是**用户会话实例**，
`windows-abc123` 才是 **SYSTEM 服务实例**。推送只会发给主实例，会话实例会被跳过
（见下面「常见跳过原因」），刷新证书也只有 SYSTEM 实例有效（见最后一节）。

### 2. 上传安装包到版本库（本地 MCP）

```
argus-files:upload_agent_version(
    file_path="/Users/x/code/Argus/agent/dist/argus-agent-v1.41.917-windows-amd64-lite.zip"
)
```

- **走 HTTP 直传，不占 AI context**：文件流式 multipart 直接打到
  `POST /api/admin/versions/upload`，全程常量内存，AI 只看到一段短 JSON。
  远程 MCP 那条 base64 内联的路会把十几 MB 的 exe 变成更大的字符串直接爆 context，
  所以安装包**一律走本地 MCP**。需要管理员账号。
- **参数可从文件名推断，但结果一定要核对**。命名约定：
  `argus-agent-<version>-<os>-<arch>[-<variant>].<zip|tar.gz|tgz|exe>`。
  推断结果在返回的 `field_sources`（逐字段标 `explicit` / `inferred`）和 `inference` 里回报。
  **推断不猜默认值** —— 任何一项定不下来会直接报错要求显式传 `version` / `os` / `arch`。
- **必须核对 sha256**。返回里同时给本地流式算出的 `local.sha256`、服务端落库的
  `server.sha256`，和布尔 `sha256_match`。
  **`sha256_match=false` 时 `ok=false`，绝对不要拿这条版本记录去推送** ——
  这是"上传的和落库的是同一个文件"的唯一证据。
- **同名冲突**：服务端按**文件名**（不是 version/os/arch）做唯一性检查，重名返回 409。
  出路三选一：沿用已有记录 / 先删旧记录再传 / 改个包名并存。

### 3. 确认版本在库、拿 version_id

```
list_agent_versions(os="windows", arch="amd64")
```

`os` / `arch` 可选，用来收窄。返回每条带
`id / version / os / arch / filename / file_size / sha256 / changelog / created_at`。
**`id` 就是推送要用的 `version_id`。**

推之前对着 `filename` 再看一眼：**别把 lite 包推给没有 NVIDIA GPU 的机器，
也别把 windows 包推给 linux**。包名里的 `os` / `arch` / `variant` 就是给你核对用的。

### 4. ★ 确认目标机器证书已落盘

这是整个流程里最容易被跳过、后果最重的一步。

```
run_safe_command(agent_id="windows-abc123", command="dir C:\\ProgramData\\Argus\\certs")
```

- 列出来有证书文件 → 可以推。
- 目录不存在 / 是空的 → **停，先别推**。让这台机器先 enroll 或刷新证书，
  确认证书落盘之后再回到这一步重查。
- 命令报错、机器离线、拿不准 → **也停**。看不清就不推，不要"应该有吧"。

`run_safe_command` 是 L1 白名单，`dir` 在白名单里，直接可用。
Linux/macOS 现场对应看 `/etc/argus/certs` 或 agent 配置里的证书路径
（`get_agent_config` 能看到配置）。

### 5. 推送升级（L3，每次邮箱审批）

```python
# 第 1 步：说清楚意图，触发验证码
push_agent_update(
    agent_ids=["windows-abc123"],          # 逐台显式列，先来一台
    version_id=42,
    _approval_reason="huawei-book 当前 v1.40.907，升到 v1.41.917 修 session manager 崩溃；"
                     "已确认 C:\\ProgramData\\Argus\\certs 下有证书，非 legacy 模式；"
                     "先升这一台验证，起来后再铺开其余 5 台；"
                     "回滚方案：版本库里 v1.40.907 记录仍在，可推回去"
)
# → APPROVAL_PENDING，用户查邮箱

# 第 2 步：用户给 6 位验证码后原样重试
push_agent_update(agent_ids=["windows-abc123"], version_id=42, _approval_code="482913")
```

`_approval_reason` 要写清 **哪台 / 从什么版本到什么版本 / 为什么 / 证书已确认 / 回滚方案**。
这封邮件是用户拦下一次误操作的最后一道关，写"升级 agent"等于没写。

**L3 没有快路径**：每台、每次都要重新审批。这是设计，不是 bug。

**读懂返回**：

| 字段 | 含义 |
|---|---|
| `version.version` / `filename` | 实际推的是哪个包 —— 推完对一眼，别推错版本还不知道 |
| `version.package_sha256` | 上传的**包文件**的 sha256（与 `list_agent_versions` 里的一致） |
| `version.binary_sha256` | 真正发给 agent 的**包内二进制**的 sha256。zip 包下这两个值**不同**，别混为一谈 |
| `pushed[]` | 成功下发的设备，带 `agent_label`（别名 · ID） |
| `skipped[]` | 被跳过的设备，带 `status` 和人话 `detail` |
| `success` | **由实际结果算出**：有任何一台被跳过就是 `false` |
| `note` | 提醒：指令已下发 ≠ 升级成功 |

**常见跳过原因**：

| `status` | 含义 | 怎么办 |
|---|---|---|
| `offline` | 设备离线，指令没发出去 | 等它上线再推这一台，别以为推过了 |
| `skipped_session_agent` | 这是用户会话实例，按规则只推主实例 | 正常行为，推对应的 SYSTEM 实例即可 |

### 6. 确认真的升上去了

**"指令已下发"只代表指令发出去了**，Agent 还要下载、校验、重启。过一会再查：

```
list_update_records(agent_id="windows-abc123", limit=10)
list_agents()   # 看这台的 version 字段变没变、是否重新上线
```

真正的验收信号是 **`list_agents` 里这台重新在线且 version 已变**，
不是更新记录里的 `sent`。

> ⚠️ **`list_update_records` 有一个会误导人的坑，务必知道**：
> 服务端只返回**最近 100 条**记录，而且这个窗口是**全局的**（所有 agent 混在一起按时间倒序切），
> `agent_id` 和 `limit` 是在这 100 条**之后**再过滤的。
> 所以某台的记录被别人的推送挤出窗口后，查它会返回**空数组** ——
> **"查不到记录"和"没推送过"长得一模一样**。
> 返回里的 `source_window` 和 `matched_before_limit` 就是提醒你这件事的。
> **不要据此得出"这台还没升级，再推一次"** —— 要判断升没升，看 `list_agents` 的 version。

---

## 配置 agent 上的三方软件 MCP 服务（L2）

让远端 AI 经 Argus 调用**宿主机上第三方软件自带的 MCP 接口**（SmartQuality v3 的 capmcp、
BestPLC 的 DiagMcp 等）之前，要先在对应 agent 上配一条服务。

```python
configure_host_software(
    agent_id="windows-abc123",
    name="diagmcp",                        # 服务名 slug，小写字母开头，仅小写字母/数字/连字符
    display_name="BestPLC 诊断",
    url="http://192.168.2.121:8332",       # ★ agent 会去连的地址
    auth_type="bearer",
    auth_secret="<软件自己的凭证明文>",
    max_concurrency=2,
    timeout_s=30,
    _approval_reason="给 192.168.2.121 上的 DiagMcp 配接入，供巡检读诊断数据"
)
```

**要点：**

- **配置即授权，url 决定 agent 去连内网哪个地址。** 目标可以是 loopback，
  也可以是这台 agent 能到的**内网另一台机器**。
- **目标地址只由这份配置决定，调用方在运行时无法覆盖。**
  这是网关的安全红线：`svc_http` 请求的 payload 里**根本没有 host/port/url 字段**。
  否则 Argus 就成了一个带审计的内网 SSRF 跳板 —— 谁能调用，谁就能让 agent
  去打那个内网里的任意地址。
- **填错 url 是静默故障**：之后所有该服务的调用都会打到别的机器上，
  返回的是另一台机器的正常响应，调用方察觉不到。**落笔前跟用户核对地址。**
- **upsert 语义，没传的字段不动**：只想改超时就只传 `timeout_s`，不会把 url / 凭证冲掉。
- **凭证**：`auth_secret` 是**软件自己的凭证**（不是 Argus 的 key），Fernet 加密落库、
  对外只回掩码、审计里记 `<redacted>`、随请求下发给 agent 用完即弃**不在 agent 落盘**。
  传空串 = 显式清除，不传 = 不改。
- `enabled=false` 时该服务的调用一律按"未注册"处理。

看现有配置用 L1（注意这两个是**按服务名**查的，不是按 agent 查）：

```
list_host_software()                    # 列全部：哪个现场有什么软件、版本、在线状态
list_host_software(service="diagmcp")   # 只看某个服务
list_host_software(only_online=true)    # 只看在线现场（默认 false，离线也列出来便于排查）

describe_host_software(service="diagmcp", agents=["windows-abc123"])
# 看同一软件在各现场的版本与工具清单差异；agents 支持 agent_id / 别名 /
# group:<组> / host_type:<机型>，不填则全部。网关不做版本仲裁，差异摆出来由你判断。
```

删除用 `remove_host_software(agent_id=..., name=..., _approval_reason=...)`。
删除后该服务立即返回"未注册"，**凭证随配置一并删除，恢复要重新录入**。
影响面是"这台现场的这个软件整体不可用"，不影响其他现场的同名服务。

---

## 排查经验：只有系统级实例能刷新证书

**实测坑，会让你白忙一场：**

对着**用户级实例**（`agent_id` 带 `-用户名` 后缀那个）点"刷新证书"，
**永远不会生效** —— 它会**静默跳过**，而且：

- **返回是成功的**；
- **日志里也没有任何记录**。

也就是说，界面上看起来做完了，实际上什么都没发生。你会以为证书已经刷好了，
然后拿着这个错误前提去推包 —— 正好踩中本 skill 开头那个"起不来只能到现场"的坑。

**正确做法：刷新证书永远对 SYSTEM 服务实例做**，也就是不带用户名后缀的那个 `agent_id`。
刷完再用第 4 步的 `dir C:\ProgramData\Argus\certs` **实地确认证书落盘**，
不要相信"点了就是刷了"。

一句话：**看结果，不看动作。**

---

## 注意事项

- **推送不接受"全部"**：`agent_ids` 必须逐台显式列出，`*` / `all` / `全部` 会被拒。
- **先一台后一片**：第一台起来并确认 version 已变，再推剩下的。
- **离线机器不会排队**：`offline` 就是没发出去，等它上线再单独推。
- **别推错平台/变体**：包名里的 `os` / `arch` / `variant`（如 `lite`）就是给你核对的。
  有 NVIDIA GPU 的机器 lite 包够用；没有的需要含 FFmpeg 的完整版。
- **sha256 不匹配的版本记录不要用**：上传时 `sha256_match=false` 说明落库的不是你传的那个文件。
- **验收看 `list_agents` 的 version**，不看更新记录里的 `sent`。
- **手动部署兜底**：远程推送失败、或机器已经因为证书问题起不来的场景，
  走 `Agent手动部署` 那套（MCP 文件传输 + run_command），但那救不了"exe 换了起不来"
  —— 那种只能到现场。
