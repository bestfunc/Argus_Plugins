---
name: terminal
display_name: 终端
description: 通过 Argus 远程连接宿主机终端，执行命令、查看文件、排查问题。默认用 run_safe_command（L1 白名单），敏感或破坏性命令用 run_command（L3 每次邮箱审批）。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__run_safe_command,mcp__argus__run_command,mcp__argus__list_files,mcp__argus__read_file,mcp__argus__get_agent_metrics
---

# 远程终端

通过 Argus MCP Server 远程操作宿主机。

## 可用工具

| 工具 | 等级 | 说明 |
|------|------|------|
| `list_agents` | 🟢 L1 | 列出可用 Agent |
| `get_agent_metrics` | 🟢 L1 | 查看系统指标（CPU、内存、磁盘等） |
| `list_files` | 🟢 L1 | 列出目录内容 |
| `run_safe_command` | 🟢 L1 | **默认用这个**。白名单只读命令（uname, hostname, whoami, ls, dir, ps, top, netstat, df, du, cat, head, tail, grep, which, where 等） |
| `read_file` | 🟡 L2 | 读文件内容（首次授权 + 15 分钟快路径） |
| `run_command` | 🔴 L3 | 任意 shell 命令（每次邮箱审批，5 分钟验证码） |

## 使用流程

### 0. 排查节奏（先礼后兵）

碰到问题别上来就 `run_command`（L3 每次要邮箱验证码）。标准节奏：

```
①  list_agents / agent-inventory skill → 找对 agent_id
②  get_agent_metrics → 看大局指标（L1）
③  run_safe_command → 跑 ps / df / uptime / tasklist 等白名单只读命令摸情况
④  找到问题后，才考虑 run_command（L3）做"动作"
```

80% 诊断问题在 ③ 就看清了。**能 L1 解决就别升 L3**——用户被验证码打扰一次少一次就少一次，审批疲劳是真实的体验问题。具体剧本见 `troubleshoot-playbook` skill。

### 1. 确定目标 Agent
- 如果用户未指定，先用 `list_agents` 列出可用主机
- 根据用户描述匹配（如"116 服务器"匹配名称含 116 的）
- 记住选定的 agent_id，后续命令复用

### 2. 判断操作系统
首次连接用 `run_safe_command` 跑 `uname -s`：
- 返回 `Linux` → Linux 命令
- 返回 `Darwin` → macOS
- 失败或 Windows 特征 → Windows 命令

### 3. 只读/诊断（L1，默认路径）

**优先用 `run_safe_command`**，它是 L1 白名单，直接可用：

```
run_safe_command(agent_id="xxx", command="ps aux | head -20")
run_safe_command(agent_id="xxx", command="df -h")
run_safe_command(agent_id="xxx", command="tail -n 50 /var/log/app.log")
```

白名单外的命令会被 Agent 拒绝，此时才考虑 `run_command`。

### 4. 破坏性/写入/复杂命令（L3）

需要写入、删除、重启服务、执行脚本等操作时用 `run_command`。**这是 🔴 L3 工具，每次都要邮箱审批**：

```python
# 第 1 步：带 reason 发起
run_command(
    agent_id="xxx",
    command="systemctl restart nginx",
    _approval_reason=(
        "用户反馈 nginx 配置更新后没生效，让我重启 nginx 服务。"
        "影响：nginx 短暂中断（<2s）；回滚方案：再 systemctl restart 一次。"
    )
)
# → Server 抛 APPROVAL_PENDING，向用户邮箱发送 5 分钟验证码

# 第 2 步：用户把验证码告诉你后原样重试
run_command(
    agent_id="xxx",
    command="systemctl restart nginx",
    _approval_reason="（同上）",
    _approval_code="284713"
)
```

L3 每次都要新验证码，没有快路径。

### 5. 结果解读
- 分析命令输出，用简洁中文向用户说明结果
- 错误码非 0、磁盘满、内存不足等异常主动提示
- 建议下一步排查动作

## 访问内网设备

**不要**通过宿主机中转执行 `ssh`、`curl` 这类命令——交互式输入、密码传递、超时都会卡。

**正确做法**：用 `/隧道` skill 建端口映射，然后用 MCP 工具直接访问：

- 内网 SSH → `create_tunnel` 映射 22 端口，再用 `run_command`（L3）通过本地端口连接
- 内网 HTTP → `create_tunnel` 映射 80 端口，用 `proxy_api_get`（L1）/`proxy_api`（L2）访问
- 内网数据库 → `create_tunnel` 映射 3306 端口，用 `execute_select`（L1）/`execute_sql`（L3）查询

## Windows PowerShell 转义 — `run_command` 加 `shell="powershell"` 即可

**`run_command` 的 `shell` 参数**(仅 L3 `run_command`,不是 `run_safe_command`):

| `shell` 取值 | 行为 |
|--------------|------|
| `"powershell"` | server **自动** UTF-16 LE base64 编码 + 包成 `powershell -NoProfile -EncodedCommand <b64>` → Agent 透明跑 PowerShell。**AI 直接写自然脚本,绝不要自己 base64** |
| `"cmd"` 或不传 / `"auto"` | Windows agent 默认 `cmd /c <command>`(适合 dir / tasklist / sc 等经典 cmd) |
| `"sh"` / `"bash"` | Linux 上用,跟默认一样(目前 agent 端都走 sh -c) |

**什么时候必须 `shell="powershell"`**:
- 命令含 `$_` / `$env:` / `$p` 等变量
- 含管道 `|` + `Where-Object` / `Select-Object`
- 含花括号 `{}` 或引号嵌套
- 多行脚本
- 任何**只有 PowerShell 有**的 cmdlet(`Get-Process` / `Invoke-WebRequest` / `Test-Path` ...)

**为什么 server 帮你 base64?** Windows 上 agent 默认 `cmd /c`,PowerShell 脚本的 `$` / `"` / `|` / `<` / `>` 会被 cmd parser 吃掉。`-EncodedCommand` 让 cmd 看到的是一个无特殊字符的 base64 token,PowerShell 内部再解码 — 转义问题彻底绕开。

**例子**:
```python
# ✅ 一次写对,自然语法
run_command(
    agent_id="windows-xxx",
    command='Get-Process | Where-Object { $_.WorkingSet -gt 100MB } | Select Name,Id',
    shell="powershell",
    _approval_reason="排查 windows-xxx 内存占用,看哪些进程超 100MB"
)

# ❌ 不设 shell → cmd parser 吃掉 $_ 和管道,结果错乱
run_command(
    agent_id="windows-xxx",
    command='Get-Process | Where-Object { $_.WorkingSet -gt 100MB } | Select Name,Id',
    # 缺 shell="powershell" → 实际跑 `cmd /c Get-Process ...`,管道在 cmd 里语义不同
)
```

> **`run_safe_command` 没有 `shell` 参数** — 它的白名单只允许 `tasklist`/`ipconfig`/`sc`/`dir` 等经典 cmd 首词,加 shell=powershell 会让首词变成 `powershell` 过不了白名单。要跑 PowerShell 一律用 `run_command`(L3 审批)。

**别再自己手动 base64 / EncodedCommand 了** — server 替你做。

## Agent 身份说明

- **系统服务 Agent**（`windows-xxx`）：SYSTEM 身份，权限高，**无法**访问用户目录和 WSL
- **用户会话 Agent**（`windows-xxx-username`）：真实用户身份，可访问 WSL、用户目录、GUI
- 需要 WSL → 用户会话 Agent + `wsl <command>` 前缀
- 需要 root/管理员 → 系统服务 Agent

## 授权协议（L2/L3）

调用 `read_file` / `run_command` 走完整邮箱验证码流程（见 `/MCP授权` skill）：

- **铁律**：不要自己编 6 位验证码，必须等用户从邮箱里复制给你
- **铁律**：重放时 Server 用数据库里保存的原始参数，改 command / path 无效
- `_approval_reason` 要具体：**做什么、预期影响、失败回滚方案**

## 工作组级约束

管理员可能额外限制：

- `read_file.path_deny`：路径黑名单（fnmatch），如 `*.env` / `*.key`，命中返回 `CONSTRAINT_VIOLATED`

触发约束不会消耗验证码，换路径或问用户要改配置。

## 注意事项

- 每次只执行一条命令，等结果再决定下一步
- 长时间命令设合理 timeout（默认 30s，最大 300s）
- 输出过长用 `| head -50` 或 `| tail -50` 截取
- 不要在命令中包含密码（会写入审计日志）
- Windows 路径用 `\`
- 内网设备优先用隧道，不要命令行中转
