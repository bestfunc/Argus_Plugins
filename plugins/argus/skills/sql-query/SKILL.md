---
name: sql-query
display_name: SQL查询
description: 通过 Argus Agent 远程查询宿主机上的数据库。默认用 execute_select（L1 只读）；要跑 DML/DDL 用 execute_sql（L3 每次邮箱审批）。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__execute_select,mcp__argus__execute_sql,mcp__argus__get_agent_config
---

# SQL 查询

通过 Argus Agent 远程查询宿主机上的数据库。

## 可用工具

| 工具 | 等级 | 说明 |
|------|------|------|
| `list_agents` | 🟢 L1 | 列出可用主机 |
| `get_agent_config` | 🟢 L1 | 查看 Agent 配置的数据库预设 |
| `execute_select` | 🟢 L1 | **默认用这个**。只读查询，仅允许 SELECT / SHOW / DESCRIBE / EXPLAIN |
| `execute_sql` | 🔴 L3 | 任意 SQL（含 DML/DDL），**每次都要邮箱审批**（5 分钟验证码） |

## 使用流程

### 1. 确定目标 Agent
- 如果用户未指定 Agent，先用 `list_agents` 列出可用主机
- 让用户选择目标，或根据上下文自动匹配

### 2. 查看数据库预设
- 用 `get_agent_config` 查看该 Agent 配置了哪些数据库连接
- 记住 preset 名称，后续查询复用

### 3. 查询数据（默认）

**默认用 `execute_select`**，它是 L1 工具，直接可用：

```
execute_select(
    agent_id="xxx",
    preset="main_db",
    query="SELECT id, name FROM users WHERE status='active' LIMIT 50"
)
```

只允许 SELECT / SHOW / DESCRIBE / EXPLAIN。其他语句会直接被 Agent 拒绝。

### 4. 修改数据（仅在用户明确要求时）

要跑 INSERT / UPDATE / DELETE / ALTER / CREATE 等，必须用 `execute_sql`。这是 🔴 **L3 工具**：

```
# 第 1 步：首次调用要说明意图
execute_sql(
    agent_id="xxx",
    preset="main_db",
    query="UPDATE orders SET status='canceled' WHERE id=12345",
    _approval_reason=(
        "用户要求取消订单 12345。单行 UPDATE，范围限定在 id=12345；"
        "回滚方案：再跑一条 UPDATE 把 status 改回原值。"
    )
)
# → Server 抛 APPROVAL_PENDING，已向用户邮箱发送 5 分钟验证码

# 第 2 步：用户把 6 位验证码告诉你后，原样重试
execute_sql(
    agent_id="xxx",
    preset="main_db",
    query="UPDATE orders SET status='canceled' WHERE id=12345",
    _approval_reason="（同上）",
    _approval_code="284713"
)
```

**每次 L3 调用都要新验证码，没有快路径。**

## 参数说明

- `agent_id`: 目标 Agent ID
- `preset`: 数据库预设名称（从 get_agent_config 获取）
- `query`: SQL 语句
- `limit`: 结果行数上限（默认 100，可调大）
- `timeout`: 超时秒数（默认 30，最大 300）

## 查询技巧

- 先用 `SHOW TABLES` 或 `SHOW DATABASES` 了解数据库结构
- 用 `DESCRIBE 表名` 查看表结构
- 复杂查询先用 `EXPLAIN` 确认执行计划
- 大表查询务必加 `LIMIT`，避免超时
- 用 `COUNT(*)` 先确认数据量再拉取明细

## 授权协议（L3）

调用 `execute_sql` 走完整的邮箱审批流程（见 `/MCP授权` skill）：

- **铁律**：验证码不要自己编，必须等用户从邮箱里复制给你
- **铁律**：重放时 Server 用数据库里保存的原始 query 执行，改 query 无效
- `_approval_reason` 要写清楚**操作范围、预期影响、回滚方案**，它会写进审计日志

如果用户只是想查数据，不要直接上 `execute_sql`，先用 `execute_select` 试试。

## 工作组级约束

管理员可能额外限制（来自 `permissions.mcp_tool_constraints`）：

- `execute_sql.forbid_keywords`：禁用关键字（如 DROP / TRUNCATE），命中直接返回 `CONSTRAINT_VIOLATED`
- `execute_sql.presets` / `execute_select.presets`：preset 白名单，非白名单里的 preset 也会被拒

触发约束不会消耗验证码，改参数或换 preset 再试。

## 注意事项

- **预设制**: 数据库连接通过 preset 名称引用，DSN 配置在 Agent 端管理
- **权限**: 受 API Key 关联的工作组限制
- **超时**: 默认 30 秒，最大 300 秒
- **别滥用 execute_sql**：只要是读数据，一定优先 `execute_select`
