---
name: mcp-authorization
display_name: MCP授权
description: Argus MCP 工具的三级授权流程（v1.29+）：L1 直接调用 / L2 邮箱验证码授权 / L3 每次邮箱审批。所有涉及写入、破坏性或敏感读取的操作都要先看本 skill 理解协议。
user-invocable: false
---

# Argus MCP 授权协议（v1.29+）

Argus MCP 工具按风险等级分三档。调用前先判断工具属于哪档，按对应协议准备参数。

## 风险等级

| 等级 | 含义 | 示例 |
|------|------|------|
| 🟢 **L1 安全** | 只读或可逆；默认开放，直接调用 | `list_agents`, `get_agent_config`, `execute_select`, `run_safe_command`, `proxy_api_get`, `list_files`, `read_file`, `download_file`, `list_tunnels`, `tunnel_stats`, `toggle_tunnel`, `get_screen_info`, `get_ui_elements`, `screenshot`, `wait_stable`, `get_clipboard`, `set_clipboard`, `scroll` |
| 🟡 **L2 半可逆** | 可读敏感数据或配置可恢复；首次调用要邮箱授权，15 分钟快路径 | `upload_to_sandbox`, `proxy_api`, `create_tunnel`, `delete_tunnel`, `update_tunnel`, `set_agent_direct_listen` |
| 🔴 **L3 不可逆** | 可造成生产事故；**每次**都要邮箱审批，无快路径 | `run_command`, `execute_sql`, `upload_file`, `click`, `double_click`, `right_click`, `drag`, `key`, `type_text` |

> 工具等级和清单以 `tools/list` 返回的 description 为准（Server 在 description 里会明确标注）。

## 调用协议

### L1 —— 直接调用

```
execute_select(agent_id=..., preset=..., query="SELECT ...")
```

没有特殊字段。正常调用即可。

### L2 —— 首次授权 + 15 分钟快路径

**第一次对某个 agent 调用某个 L2 工具**：

1. 调用时必须传 `_approval_reason`，用人话讲清楚：你要做什么 / 为什么要做 / 预期影响
2. Server 会向**用户本人邮箱**发送 6 位验证码（15 分钟有效），并抛 `APPROVAL_PENDING` 错误
3. **等用户把邮箱收到的验证码告诉你**，不要继续假装调用，也不要自己编号码
4. 用户给出验证码后，用 `_approval_code` 参数**原样重试同一次调用**
5. 成功后，**同一 Agent 上的同一工具**在 15 分钟内无需再次验证码

```python
# 第 1 步：带 reason 发起（以 create_tunnel 为例）
create_tunnel(
    agent_id="windows-xxx",
    name="SSH-backend",
    target_host="192.168.1.10", target_port=22, listen_port=30022,
    _approval_reason="用户要 SSH 到 192.168.1.10 排查问题，建临时隧道"
)
# → 抛 APPROVAL_PENDING，告诉用户查邮箱

# 第 2 步：用户给验证码后
create_tunnel(
    agent_id="windows-xxx",
    name="SSH-backend",
    target_host="192.168.1.10", target_port=22, listen_port=30022,
    _approval_code="482913"
)
# → 成功

# 第 3 步：15 分钟内对同一 agent 上的 create/delete_tunnel 无需再授权
delete_tunnel(mapping_id="xxx")   # 直接通过
```

### L3 —— 每次审批

**每次调用都要重新授权**，没有任何快路径：

1. 每次调用都要传 `_approval_reason`，详细说明操作范围、预期影响、失败回滚方案
2. Server 发送 5 分钟有效的一次性验证码
3. 用户给验证码后用 `_approval_code` 重放
4. 重放成功后，下次再调还是要重新审批

```python
# 每次都要走完整流程
execute_sql(
    agent_id="linux-xxx",
    preset="main_db",
    query="UPDATE orders SET status='cancel' WHERE id=12345",
    _approval_reason=(
        "用户要求取消订单 12345。操作为单行 UPDATE，可通过下一条 UPDATE 恢复。"
        "影响范围：仅该订单；预期影响行数 1。"
    )
)
```

## 铁律

1. **绝不捏造验证码**：6 位数字必须从用户复制粘贴过来，你看不到用户邮箱
2. **绝不重用验证码**：一个验证码只能用一次，而且绑定到原参数（每次请求都要新生成）
3. **参数以 Server 存储为准**：重放时，除 `_approval_code` / `_approval_reason` 外的参数即使被你改了也不生效，Server 用第一次请求时写入数据库的原始参数执行
4. **优先用 L1 替代**：如果只是只读/查询，用 `execute_select` 代替 `execute_sql`、用 `run_safe_command` 代替 `run_command`、用 `proxy_api_get` 代替 `proxy_api`
5. **reason 要人话**：不要写 "调用 execute_sql"，要写 "用户让我统计上周订单量，跑一条 SELECT COUNT"——reason 会存到审计日志，将来查责时看的就是这段话
6. **用户说"撤销授权"**：引导用户到 Console `Settings → MCP → 我的 MCP 请求` 撤销 L2 快路径或查看历史请求

## 常见异常

| ErrCode | 含义 | 处理 |
|---------|------|------|
| `APPROVAL_PENDING` | Server 已发邮件，等用户给验证码 | 跟用户说"请查收邮箱，把 6 位验证码发给我" |
| `APPROVAL_INVALID` | 验证码错、过期、或已用过 | 叫用户重新发起或再收一遍 |
| `PERMISSION_DENIED` | 工作组没有该工具权限 | 别再试了，告诉用户需要管理员在 Console 里授权 |
| `CONSTRAINT_VIOLATED` | 参数违反工作组约束（路径黑名单 / SQL 关键字等） | 调整参数或换工具；告诉用户配置是什么限制的 |
| `INVALID_ARGUMENT` | L2 首次或 L3 没传 `_approval_reason` | 补上 reason 再调 |
| `AGENT_OFFLINE` | 目标 Agent 不在线 | 查 `list_agents` 确认状态 |

## 工作组级参数约束

即使你被授权调用一个工具，工作组管理员还可以限制具体参数：

- `read_file` / `download_file` / `upload_*`：`path_deny`（fnmatch 黑名单，如 `*.env` / `*.key`）
- `execute_sql`：`forbid_keywords`（禁用关键字，如 DROP/TRUNCATE/DELETE/ALTER）、`presets`（preset 白名单）
- `execute_select`：`presets`（preset 白名单）
- `proxy_api` / `proxy_api_get`：`url_allow`（URL regex 白名单）、`method_allow`（HTTP 方法白名单）

触发这些规则会返回 `CONSTRAINT_VIOLATED`，不会消耗验证码也不会创建审批。别反复撞，问用户要改参数还是要改配置。
