---
name: file-transfer
display_name: 文件传输
description: 通过 Argus Agent 上传/下载文件到宿主机。小文件 / 文本用远程 MCP（argus:* 前缀工具，受 path_deny 约束 + L2/L3 审批）；**大文件必须用本地 MCP（argus-files:* 前缀工具）**——走 HTTP 直连绕开 MCP base64 限制，AI context 与文件大小解耦。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__list_files,mcp__argus__read_file,mcp__argus__grep_file,mcp__argus__file_search,mcp__argus__archive_file,mcp__argus__unarchive_file,mcp__argus__download_file,mcp__argus__upload_to_sandbox,mcp__argus__upload_file,mcp__plugin_argus_argus_files__upload_file,mcp__plugin_argus_argus_files__upload_to_sandbox,mcp__plugin_argus_argus_files__download_file,mcp__argus__run_safe_command,mcp__argus__run_command
---

# 文件传输

通过 Argus Agent 与远程宿主机之间传输文件。**关键分流：文件 > 1 MB 用本地 MCP（`argus-files:*`），文本/小文件用远程 MCP（`argus:*`）。**

## 两种 MCP 通道

| 场景 | 用哪个 | 原因 |
|------|--------|------|
| 读远程文件内容到 AI 上下文（查 nginx.conf 等） | `argus:read_file` | 内容要进 AI context 分析 |
| 下载大文件到本地（> 1 MB） | `argus-files:download_file` | 走 HTTP 直流，不过 AI context |
| 上传本地大文件（> 1 MB） | `argus-files:upload_file` / `upload_to_sandbox` | 同上 |
| 上传短文本（< 100 KB） | `argus:upload_file` | L3 审批流程保护 |

**为什么分两条路：** 远程 MCP 的 upload_file / download_file 用 base64 内联，13 MB 的 exe 编码后 17.6 MB 会直接爆 AI context。本地 MCP 通过 `npx` 启动的 `@bestfunc-com/argus-file-mcp` 进程，读本地文件 → HTTP multipart 直传 Argus Server(MinIO) → 推送 Agent，AI 只看到 `{transfer_id, size, status}` 短 JSON。

## 可用工具

### 远程 MCP（`argus:*`）

| 工具 | 等级 | 说明 |
|------|------|------|
| `list_agents` | 🟢 L1 | 列出可用 Agent |
| `list_files` | 🟢 L1 | 浏览远程目录 |
| `read_file` | 🟢 L1 | 读取文本文件内容,**支持 offset/size 分片**(受工作组 `path_deny` 约束) |
| `grep_file` | 🟢 L1 | 在文件里行级正则搜索(流式扫描,适合 GB 级日志) |
| `file_search` | 🟢 L1 | 按文件名 glob 递归找文件(等价 `find -name`) |
| `archive_file` | 🟡 L2 | **打包** zip/tar.gz,可分卷(替代 `run_command` 跑 tar/zip) |
| `unarchive_file` | 🟡 L2 | **解压** zip/tar.gz,自动识别分卷(替代 `run_command` 跑 unzip/tar -x) |
| `download_file` | 🟢 L1 | 从 Agent 下载**并把内容返回 AI**(仅适合小文本) |
| `upload_to_sandbox` | 🟡 L2 | base64 上传到沙盒目录,**小文件用**。首次授权 + 15 分钟快路径 |
| `upload_file` | 🔴 L3 | base64 上传到任意路径,**小文件用**。每次邮箱审批 |
| `run_safe_command` | 🟢 L1 | 辅助(创建目录、查看属性等只读) |
| `run_command` | 🔴 L3 | 通用 shell 命令。**Windows 写 PowerShell 一定要设 `shell=\"powershell\"`**,server 自动处理转义 |

### 本地 MCP（`argus-files:*`）—— 大文件首选

| 工具 | 说明 |
|------|------|
| `upload_file(agent_id, local_path, remote_path)` | 本机 → Agent 任意路径，任意大小 |
| `upload_to_sandbox(agent_id, local_path, filename?)` | 本机 → Agent 沙盒目录 |
| `download_file(agent_id, remote_path, local_path)` | Agent → 本机指定路径 |

鉴权：OAuth access_token（装 plugin 时已授权），不走每文件 MCP 审批；权限边界等价 Web Console 用户通过 Transfers 页操作——工作组成员 + menu 权限控制。

## 使用流程

### 1. 确定目标 Agent

如果用户未指定，先用 `list_agents` 或 agent-inventory skill 找到对的 agent_id。注意**系统服务 vs 用户会话**的区别：

- 系统服务 agent（`windows-xxx`）：SYSTEM 身份，**看不到**用户目录 `C:\Users\xxx\Desktop`、WSL 挂载
- 用户会话 agent（`windows-xxx-hp` / `-bestfunc`）：真实用户身份，**能**访问用户目录和 WSL

读 `/etc/hosts` 这种系统文件用系统服务；读用户桌面 / WSL 路径用用户会话。选错了会 "Permission denied" 或 "目录不存在"。

### 2. 浏览远程目录（L1）

```
list_files(agent_id="xxx", path="C:\\Users" 或 "/home")
```

### 3. 读取文件内容到 AI 上下文（L1）

**铁律:读之前先看大小**。先 `list_files` 拿到目标文件的 `size`,再决定怎么读。

#### 3.1 小文件(< 200KB):整文件读

```python
read_file(agent_id="xxx", path="/etc/nginx/nginx.conf")
# 返回 JSON: {content, encoding, size, offset, next_offset, truncated}
```

#### 3.2 中等文件(200KB ~ 1MB):分片读

```python
# 读首 100KB
r1 = read_file(agent_id="xxx", path="/var/log/app.log", size=100000)
# r1.truncated=true → 接着读下一片
r2 = read_file(agent_id="xxx", path="/var/log/app.log",
               offset=r1.next_offset, size=100000)
```

每片只占 ~100KB context,翻到 truncated=false 就读完了。

#### 3.3 大文件(> 1MB) — **不要硬读**

整读必爆 context,挑下面任一:

**A. 用 grep_file 搜关键内容(首选)**

调试日志的**铁律:一定带行范围**,否则全文搜会被早期内容把 max_results 吃完。

```python
# 1) 查最近报错 — tail_lines 最常用(等价 tail -n N | grep)
grep_file(
    agent_id="xxx",
    path="C:\\app\\logs\\service.log",
    pattern="ERROR|FATAL|panic",
    tail_lines=3000,              # 只扫最后 3000 行
    before=2, after=10,           # 命中前后各看 2/10 行
    max_results=50,
    ignore_case=True,
)
# 返回 matches + file_lines(全文总行数) + window_start/end(实际扫描的行范围)

# 2) 已知行号区间(如启动日志在 9700-10049 行) — start_line/end_line
grep_file(
    agent_id="xxx", path="/var/log/daemon.log",
    pattern="ERROR",
    start_line=9700, end_line=10049,
)

# 3) 排除噪声(grep -v) — invert=true
grep_file(
    agent_id="xxx", path="/var/log/app.log",
    pattern="DEBUG|TRACE",        # 排除所有 DEBUG/TRACE 行
    invert=True,
    tail_lines=1000,
)

# 4) 全文搜配置项 — 配置文件不大,不用范围
grep_file(
    agent_id="xxx", path="/etc/nginx/nginx.conf",
    pattern="^\\s*server_name\\s",
    ignore_case=True,
)
```

⚠️ **不带行范围 = 默认从头扫**,1GB 日志命中 max_results=100 后早期就 truncated 了,看不到最近的。

**B. 拿命中行号,用 read_file offset 精确读上下文**
```python
# grep_file 命中行 12345,要看附近 50 行原始格式
# 估算偏移:平均行长 200B,12345 × 200 ≈ 2.5MB
read_file(agent_id="xxx", path="/var/log/app.log",
          offset=2400000, size=20000)
```

**C. 下载到本地用 Read 工具**
```python
argus-files:download_file(agent_id="xxx",
                          remote_path="/var/log/big.log",
                          local_path="/tmp/big.log")
# 本地用 Read(file_path="/tmp/big.log", offset=..., limit=...) 翻页看
```

### 3.5 按文件名找(L1)

```python
# 找配置文件
file_search(agent_id="xxx", root="/etc", pattern="*.conf", max_depth=3)

# 找最近改动的日志
file_search(agent_id="xxx", root="C:\\ProgramData", pattern="*.log")
# 返回 hits:[{path, size, mod_time}] —— 按 mod_time 排序找最新即可

# only_files=false 想拿到目录列表
file_search(agent_id="xxx", root="/home", pattern="*", only_files=False, max_depth=1)
```

Glob 用 Go `filepath.Match`(类似 shell),**不支持 `**` 跨层匹配**,要跨层调大 `max_depth`。

### 4. 下载大文件到本地（本地 MCP）

```python
# 只想把文件拖回来存盘，不需要内容进 AI 上下文
argus-files:download_file(
    agent_id="xxx",
    remote_path="/var/log/app.log",
    local_path="/tmp/app.log"
)
# → 返回 {status: "completed", file_size: 12345678, local_path: "..."}
# 文件流直接写本地磁盘，AI context 只收到短 JSON
```

**小文本（如 config 片段）** 直接 `argus:read_file` 就行；**日志/二进制/压缩包**用 `argus-files:download_file`。

### 5. 上传本地文件到沙盒（首选）

```python
# 本地 MCP，无大小限制
argus-files:upload_to_sandbox(
    agent_id="xxx",
    local_path="/Users/lugia/Downloads/diffgram-sync.exe",
    filename="diffgram-sync.exe"
)
# → 目标：C:\argus-mcp\diffgram-sync.exe（Windows Agent）
#        /tmp/argus-mcp/diffgram-sync.exe（Linux Agent）
```

### 6. 上传到任意路径（谨慎）

覆盖生产配置、推到 `C:\Program Files\...` 等场景。本地 MCP 版本，HTTP 直传：

```python
argus-files:upload_file(
    agent_id="xxx",
    local_path="/Users/lugia/workspace/new-config.yaml",
    remote_path="C:\\Program Files\\Argus\\config.yaml"
)
```

**或** 用远程 MCP 的 `argus:upload_file`（L3 每次审批）走 base64 路径——只在**本地没装 Node / 没装 argus-files MCP** 的极端情况用，且文件要 < 1 MB。

## 路径格式

- **Windows**: 反斜杠 `C:\\Program Files\\Argus\\config.yaml`
- **Linux**: 正斜杠 `/home/bestfunc/app/config.yaml`
- **WSL**: 用 Linux 路径，通过用户会话 Agent（`-hp`/`-bestfunc` 后缀）访问

## 大文件处理

**必须用本地 MCP（`argus-files:*`）**——远程 MCP 的 base64 编码会把 13 MB 的 exe 变成 17.6 MB 的字符串直接爆 AI context window。

本地 MCP 工作原理：
1. `@bestfunc-com/argus-file-mcp` 通过 npx 启动本地 Node 进程
2. 进程读本地文件 → HTTP multipart 直传 `https://argus.bestfunc.com/api/transfers/upload`
3. Server 入 MinIO → WebSocket 通知 Agent → Agent 从 MinIO 拉回来
4. AI 只看到 `{transfer_id, file_size, status}` 这种短 JSON

装 argus plugin 时自动带上 argus-files，`/mcp` 里会看到它作为独立条目。首次使用触发 OAuth loopback（浏览器 localhost 回调），同一 Argus 账号 session 会被复用，基本无感。

## 授权协议

### 远程 MCP 工具（`argus:*`）
- `read_file` / `download_file`（L1）：直接可用，只受工作组 `path_deny` 约束
- `upload_to_sandbox`（L2）：首次邮箱授权 + 15 分钟快路径
- `upload_file` / `run_command`（L3）：**每次**邮箱审批（5 分钟验证码）
- **铁律**：验证码必须等用户从邮箱里复制给你，不要自己编
- **铁律**：重放时 Server 用数据库里保存的 path / remote_path 执行，改路径无效

### 本地 MCP 工具（`argus-files:*`）
- OAuth access_token 鉴权（装 plugin 时已完成）
- 不走 MCP 每文件审批；权限边界等价 Web Console 用户通过 Transfers 页操作
- 审计仍写——Server 端 `/api/transfers/*` 接口有完整审计日志

## 工作组级约束

远程 MCP 的 `read_file.path_deny` / `upload_file.path_deny` 等 fnmatch 黑名单（如 `*.env` / `*.key`）**只对 `argus:*` 工具生效**；本地 MCP 走 `/api/transfers/*` 直连，不受这些约束（按设计）。敏感路径用工作组 menu 权限控制整个传输功能的开关。

## 注意事项

- `read_file` Agent 端硬上限 1MB/次,**大文件务必带 `size` 分片读**或改用 `grep_file` / 本地下载
- `grep_file` 用 Go RE2 正则语法,不是 PCRE(没有 lookaround、back-reference;`\d` `\w` 支持)
- `file_search` 单层 glob,不递归通配(`**` 不行);跨层用 `max_depth` 控制
- 远程命令行 `run_command`(L3) 可以跑 `tail/head/grep/find` 等系统命令,但**优先用专用工具**(grep_file/file_search),它们流式且跨平台一致,不需要审批
- 文件通过 Server 中转（Agent ↔ MinIO ↔ Server），不走 P2P
- Windows 路径反斜杠要双写 `\\`
- 子 Agent（用户会话）能访问 WSL 和用户私有目录
