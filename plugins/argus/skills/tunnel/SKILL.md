---
name: tunnel
display_name: 隧道
description: 通过 Argus 端口映射（隧道）功能，将 Agent 宿主机内网的 TCP 服务临时暴露到 Server 端口。list/stats/toggle 是 L1；create/delete/update 是 L2（邮箱授权 + 15 分钟快路径）。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__list_tunnels,mcp__argus__tunnel_stats,mcp__argus__toggle_tunnel,mcp__argus__create_tunnel,mcp__argus__update_tunnel,mcp__argus__delete_tunnel,mcp__argus__set_agent_direct_listen
---

# 隧道管理

通过 Argus 端口映射，将 Agent 宿主机或其内网设备的 TCP 服务映射到 Server 端口。

## 可用工具

| 工具 | 等级 | 说明 |
|------|------|------|
| `list_agents` | 🟢 L1 | 列出可用 Agent |
| `list_tunnels` | 🟢 L1 | 查看已有隧道 |
| `tunnel_stats` | 🟢 L1 | 查看连接数 / 流量 |
| `toggle_tunnel` | 🟢 L1 | 启用/停用已有隧道 |
| `create_tunnel` | 🟡 L2 | 创建新隧道（首次授权 + 15 分钟快路径） |
| `update_tunnel` | 🟡 L2 | 修改已有隧道（目标、端口等） |
| `delete_tunnel` | 🟡 L2 | 删除隧道 |
| `set_agent_direct_listen` | 🟡 L2 | 开启/关闭 Agent 的 direct 隧道监听 |

## 使用流程

### 1. 只读查看（L1）

```
list_tunnels(agent_id="xxx")          # 按 agent 过滤
tunnel_stats(tunnel_id="xxx")         # 流量统计
```

### 2. 创建隧道（L2）

```python
# 第 1 步：首次授权
create_tunnel(
    agent_id="windows-xxx",
    name="SSH-192.168.1.101",
    target_host="192.168.1.101",
    target_port=22,
    listen_port=30022,
    _approval_reason="用户要 SSH 到宿主机内网的 192.168.1.101，建隧道映射 22 端口"
)
# → APPROVAL_PENDING，用户查邮箱给验证码

# 第 2 步：用户给 6 位验证码后原样重试
create_tunnel(
    agent_id="windows-xxx",
    name="SSH-192.168.1.101",
    target_host="192.168.1.101",
    target_port=22,
    listen_port=30022,
    _approval_code="482913"
)

# 第 3 步：15 分钟内对同一 agent 的 create/delete/update_tunnel 调用无需再授权
delete_tunnel(tunnel_id="xxx")  # 直接通
```

### 3. 管理隧道（L2）

- `update_tunnel`: 改目标 host/port 或名称
- `delete_tunnel`: 删除
- `toggle_tunnel`: 启用/禁用（只是状态切换，是 L1）

### 4. Direct 监听（L2）

`set_agent_direct_listen(agent_id=..., enabled=true)` 让 Agent 开放 mTLS 直连端口，供"一端公网一端内网"场景的 direct 隧道拨号。

## 参数说明

- `agent_id`: 目标 Agent ID（隧道经由此 Agent 转发）
- `name`: 映射名称，建议包含用途和目标（如 "SSH-192.168.1.101"）
- `target_host`: Agent 侧目标地址（默认 127.0.0.1，可设为内网其他设备 IP）
- `target_port`: Agent 侧目标端口
- `listen_port`: Server 侧监听端口（外部通过此端口访问）
- `tunnel_type`: `port`（默认）/ `p2p` / `direct`

## 使用示例

- 内网 SSH: `target_host="192.168.1.101", target_port=22, listen_port=30022`
- 宿主机本地 Web: `target_host="127.0.0.1", target_port=8080, listen_port=38080`
- 内网数据库: `target_host="192.168.1.200", target_port=3306, listen_port=33306`

## 工作原理

```
外部用户 → Server:listen_port (TCP) → WebSocket 隧道 → Agent → target_host:target_port
```

- Server 在 `listen_port` 上监听 TCP 连接
- 通过 Agent 的 WebSocket 隧道将流量转发到内网目标
- 支持多个映射复用同一条 WebSocket 连接

## 授权协议（L2）

调用 `create_tunnel` / `update_tunnel` / `delete_tunnel` / `set_agent_direct_listen` 走首次授权 + 15 分钟快路径（见 `/MCP授权` skill）：

- **铁律**：验证码必须等用户给，不要自己编
- **铁律**：重放时 Server 用数据库里保存的参数执行，改 target_host/port 无效
- `_approval_reason` 要讲清楚**为什么要开这条隧道、暴露什么服务**

## 注意事项

- **仅支持 TCP**（SSH/HTTP/数据库等基于 TCP）
- **Agent 必须在线**才能使用隧道，Agent 离线时隧道自动断开
- **端口唯一性**：Server 侧 listen_port 不能重复
- **端口范围**：非管理员用户受工作组配置的端口范围限制
- **临时性**：隧道适合临时访问，长期使用建议配置 VPN 或专线
- **安全**：映射的端口会暴露在 Server 所在网络，注意防火墙
- **及时清理**：用完调 `delete_tunnel`，不要留着长期暴露
