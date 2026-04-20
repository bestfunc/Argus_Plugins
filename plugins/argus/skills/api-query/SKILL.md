---
name: api-query
display_name: API查询
description: 通过 Argus Agent 代理访问宿主机上的 HTTP API 服务。默认用 proxy_api_get（L1 只读 GET）；POST/PUT/DELETE/PATCH 用 proxy_api（L2 邮箱授权）。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__proxy_api_get,mcp__argus__proxy_api,mcp__argus__get_agent_config
---

# API 查询

通过 Argus Agent 代理访问宿主机局域网内的 HTTP API 服务。

## 可用工具

| 工具 | 等级 | 说明 |
|------|------|------|
| `list_agents` | 🟢 L1 | 列出可用 Agent |
| `get_agent_config` | 🟢 L1 | 查看 Agent 是否启用了 API 代理 |
| `proxy_api_get` | 🟢 L1 | **默认用这个**。只支持 GET，直接可用 |
| `proxy_api` | 🟡 L2 | 全方法 HTTP（GET/POST/PUT/DELETE/PATCH），首次授权 + 15 分钟快路径 |

## 使用流程

### 1. 确定目标 Agent
- 如果用户未指定 Agent，先用 `list_agents` 列出可用主机
- 确认 Agent 启用了 api 功能（`get_agent_config`）

### 2. GET 请求（默认）

**优先用 `proxy_api_get`**：

```
proxy_api_get(
    agent_id="xxx",
    url="http://localhost:8080/health"
)
proxy_api_get(
    agent_id="xxx",
    url="http://192.168.1.100:3000/api/info",
    headers={"Authorization": "Bearer xxx"}
)
```

### 3. POST / PUT / DELETE / PATCH（L2）

需要写入或状态变更时用 `proxy_api`。🟡 **L2 工具**：

```python
# 第 1 步：首次授权
proxy_api(
    agent_id="xxx",
    url="http://localhost:8080/api/users",
    method="POST",
    body='{"name":"tom","role":"admin"}',
    headers={"Content-Type":"application/json"},
    _approval_reason="用户让我创建新管理员账号 tom，调 POST /api/users"
)
# → 抛 APPROVAL_PENDING，告诉用户查邮箱

# 第 2 步：用户给 6 位验证码后原样重试
proxy_api(
    agent_id="xxx",
    url="http://localhost:8080/api/users",
    method="POST",
    body='{"name":"tom","role":"admin"}',
    headers={"Content-Type":"application/json"},
    _approval_code="482913"
)

# 第 3 步：15 分钟内对同一 agent 的 proxy_api 调用无需再授权
proxy_api(agent_id="xxx", url=".../api/users/tom", method="DELETE")  # 直接通
```

## 参数说明

- `agent_id`: 目标 Agent ID
- `url`: 请求 URL（从**宿主机视角**，`localhost` 指 Agent 本机）
- `method`: HTTP 方法（仅 `proxy_api` 支持非 GET）
- `headers`: 请求头（可选）
- `body`: 请求体（可选，POST/PUT 传 JSON 字符串）
- `timeout`: 超时秒数（默认 30，最大 300）

## 使用示例

- 查询本地服务状态：`proxy_api_get(url="http://localhost:8080/health")`
- 调 REST API：`proxy_api_get(url="http://localhost:8080/api/users")`
- 访问局域网：`proxy_api_get(url="http://192.168.1.100:3000/api/info")`
- 提交数据（需授权）：`proxy_api(method="POST", ..., _approval_reason="...")`

## 授权协议（L2）

调用 `proxy_api` 走完整邮箱验证码流程（见 `/MCP授权` skill）：

- **铁律**：验证码必须等用户从邮箱里复制给你，不要自己编
- **铁律**：重放时 Server 用数据库里保存的 url / method / body 执行
- 同一 agent 上的 `proxy_api` 首次授权成功后，**15 分钟内无需再授权**（同工具 + 同 agent 维度）

## 和 tunnel skill 联动（重要）

`proxy_api_get` 是**从 Agent 发起 HTTP 请求**，只能访问 Agent 宿主机能看到的网络。

**场景**：Claude Code 本地想直接 curl 内网 API（不经过 Argus MCP）→ 用 tunnel skill 把内网端口映射到公网 Server 端口，再在 Claude Code 本机 curl。

### 两种方式对比

```
场景 A：AI 要分析 API 响应数据（常见）
  → 直接 proxy_api_get，数据进入 MCP 响应给 AI
  → 优点：零配置，authenticate 走 MCP Key
  → 缺点：响应 size 大（>100KB）会超 MCP 协议限制

场景 B：要给 Claude Code 本地脚本/Bash 用（调试外部工具）
  → 先 create_tunnel 把内网 80 端口映射到 argus.bestfunc.com:38080
  → Claude Code 本机 curl https://argus.bestfunc.com:38080/api/...
  → 优点：数据不进 MCP 响应，适合大 payload、二进制下载、WebSocket
  → 缺点：要建隧道、公网暴露、用完要删
```

### 典型联动

```
# 1. 建隧道把 Agent 内网的 API 服务暴露到公网
create_tunnel(
    agent_id=..., name="debug-api",
    target_host="192.168.1.100", target_port=3000,
    listen_port=38080, _approval_reason="调试 API"
)

# 2a. AI 访问（proxy_api_get）：从 Agent 出站，响应进 MCP
proxy_api_get(agent_id=..., url="http://192.168.1.100:3000/health")

# 2b. Claude Code 本机访问（Bash）：从 argus.bestfunc.com 中转
# Bash: curl -s https://argus.bestfunc.com:38080/health

# 3. 用完清理
delete_tunnel(mapping_id=...)
```

## 工作组级约束

管理员可能额外限制：

- `proxy_api.url_allow` / `proxy_api_get.url_allow`：URL 正则白名单（如 `^http://127\.0\.0\.1/`），不匹配返回 `CONSTRAINT_VIOLATED`
- `proxy_api.method_allow`：HTTP 方法白名单（如只允许 `GET,POST`）

## 注意事项

- **宿主机视角**: `localhost` 是 Agent 所在机器，不是 Claude Code 本机
- **超时**: 默认 30 秒，最大 300 秒
- **body**: 传字符串，JSON 要自己序列化（如 `'{"key":"value"}'`）
- **权限**: 受工作组限制，需要启用 api 功能
