---
name: file-transfer
display_name: 文件传输
description: 通过 Argus Agent 上传/下载文件到宿主机。list_files/read_file/download_file 是 L1（直接可用，受工作组 path_deny 约束）；upload_to_sandbox 是 L2（首次邮箱授权）；任意路径上传 upload_file 是 L3（每次审批）。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__list_files,mcp__argus__read_file,mcp__argus__download_file,mcp__argus__upload_to_sandbox,mcp__argus__upload_file,mcp__argus__run_safe_command,mcp__argus__run_command
---

# 文件传输

通过 Argus Agent 与远程宿主机之间传输文件。

## 可用工具

| 工具 | 等级 | 说明 |
|------|------|------|
| `list_agents` | 🟢 L1 | 列出可用 Agent |
| `list_files` | 🟢 L1 | 浏览远程目录 |
| `read_file` | 🟢 L1 | 读取文本文件内容（受工作组 `path_deny` 约束） |
| `download_file` | 🟢 L1 | 从 Agent 下载到本地（受工作组 `path_deny` 约束） |
| `upload_to_sandbox` | 🟡 L2 | **优先用这个**。上传到固定沙箱前缀（Linux `/tmp/argus-mcp/`；Windows `C:\argus-mcp\`）。首次授权 + 15 分钟快路径 |
| `upload_file` | 🔴 L3 | 上传到**任意路径**，每次邮箱审批（5 分钟验证码） |
| `run_safe_command` | 🟢 L1 | 辅助（创建目录、查看属性等只读） |
| `run_command` | 🔴 L3 | 解压、移动、链接等需要写入的辅助命令 |

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

### 3. 读取文件（L2）

```python
# 首次
read_file(
    agent_id="xxx",
    path="/etc/nginx/nginx.conf",
    _approval_reason="用户让我查 nginx 配置有没有开启 gzip"
)
# → APPROVAL_PENDING，用户查邮箱给验证码

# 用户给验证码后原样重试
read_file(
    agent_id="xxx",
    path="/etc/nginx/nginx.conf",
    _approval_code="284713"
)
# → 返回内容

# 15 分钟内读同一 agent 上的其他文件无需再验证码
read_file(agent_id="xxx", path="/etc/nginx/conf.d/default.conf")
```

### 4. 下载文件（L2）

```python
download_file(
    agent_id="xxx",
    remote_path="/var/log/app.log",
    local_path="/tmp/app.log",
    _approval_reason="用户让我把 agent 上的 app.log 取回来分析"
)
# → 收到 APPROVAL_PENDING 后同 read_file 流程
```

### 5. 上传到沙箱（L2，首选）

**只要不是必须指定任意路径，优先用 `upload_to_sandbox`**。它把文件固定写到 `/tmp/argus-mcp/`（Linux）或 `C:\argus-mcp\`（Windows），爆破面更小。

```python
upload_to_sandbox(
    agent_id="xxx",
    local_path="./script.sh",
    filename="script.sh",
    _approval_reason="用户让我上传一个诊断脚本去 agent 跑"
)
# → 返回最终路径（带沙箱前缀）
```

### 6. 上传到任意路径（L3，谨慎）

覆盖生产配置、推送二进制到 `C:\Program Files\...` 等场景才用 `upload_file`。**每次都要审批**：

```python
upload_file(
    agent_id="xxx",
    local_path="./new-config.yaml",
    remote_path="/etc/app/config.yaml",
    _approval_reason=(
        "用户确认覆盖 /etc/app/config.yaml。"
        "影响：app 重启时加载新配置；回滚：把原文件另存 .bak 再替换。"
    )
)
# → APPROVAL_PENDING，5 分钟验证码
```

## 路径格式

- **Windows**: 反斜杠 `C:\\Program Files\\Argus\\config.yaml`
- **Linux**: 正斜杠 `/home/bestfunc/app/config.yaml`
- **WSL**: 用 Linux 路径，通过用户会话 Agent（`-hp`/`-bestfunc` 后缀）访问

## 大文件上传（>1MB）

MCP 工具通过 base64 传输，不适合大文件。大文件用 Bash 直接调 Server REST API：

```bash
TOKEN="mak-你的MCP_Token"
AGENT_ID="windows-xxxx"
REMOTE_PATH="C:\\target\\file.exe"
LOCAL_FILE="/path/to/local/file.exe"

curl -X POST "https://argus.bestfunc.com/api/transfers/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -F "agent_id=$AGENT_ID" \
  -F "remote_path=$REMOTE_PATH" \
  -F "file=@$LOCAL_FILE"
```

或让 Agent 从公网 URL 自己拉：
```python
run_command(
    agent_id="xxx",
    command="curl -o C:\\target\\file.exe https://example.com/file.exe",
    _approval_reason="用户让 agent 从公网直接拉 file.exe"
)
```

## 授权协议

调用 `read_file` / `download_file` / `upload_to_sandbox`（L2）走首次授权 + 15 分钟快路径；
调用 `upload_file` / `run_command`（L3）每次都要审批。详见 `/MCP授权` skill。

- **铁律**：验证码必须等用户从邮箱里复制给你，不要自己编
- **铁律**：重放时 Server 用数据库里保存的 path / remote_path 执行，改路径无效
- `_approval_reason` 要说清楚**为什么要读/写这个文件**，会写入审计日志

## 工作组级约束

- `read_file.path_deny` / `download_file.path_deny` / `upload_file.path_deny` / `upload_to_sandbox.path_deny`：
  路径黑名单（fnmatch），如 `*.env` / `*.key` / `*.pem`
- 命中约束返回 `CONSTRAINT_VIOLATED`，不消耗验证码

## 注意事项

- `read_file` 有大小限制（默认 1MB，可远程改）；大文件用 `download_file`
- 文件通过 Server 中转（Agent ↔ MinIO ↔ Server），不走 P2P
- Windows 路径反斜杠要双写 `\\`
- 子 Agent（用户会话）能访问 WSL 和用户私有目录
- 优先 `upload_to_sandbox`，它是 L2 有 15 分钟快路径；`upload_file` 是 L3 每次都要审批
