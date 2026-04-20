---
name: agent-inventory
display_name: Agent 盘点
description: 定位 Argus 管理的宿主机（Agent）。从"116 服务器"、"宝适 JAC 老线"、"所有 Windows 产线套装"这类模糊描述，匹配到具体 agent_id 再执行后续操作。所有工具都是 L1 直接可用。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__list_groups,mcp__argus__list_host_types,mcp__argus__list_agents_by_group,mcp__argus__list_agents_by_host_type,mcp__argus__get_agent_config
---

# Agent 盘点

几乎所有 Argus MCP 任务的前置动作：**先定位目标 agent，再执行**。别让 AI 瞎猜 agent_id，先按下面这套流程找。

## 可用工具（全部 L1，直接调用）

| 工具 | 用途 |
|------|------|
| `list_agents` | 列全部 agent（含 id、别名、hostname、在线状态、版本、宿主机类型、工作组） |
| `list_groups` | 列所有工作组 |
| `list_host_types` | 列所有宿主机类型 |
| `list_agents_by_group` | 按工作组过滤 |
| `list_agents_by_host_type` | 按宿主机类型过滤 |
| `get_agent_config` | 查单个 agent 的 feature 开关 / SQL preset / Web preset |

## 机器名词汇表

每台 Agent 有**多个身份字段**，分别对应不同场景：

| 字段 | 例 | 用途 |
|------|----|------|
| `agent_id` | `windows-9e4d829bca59-bestfunc` | 系统内部主键，所有 MCP 工具都认它 |
| `display_name` / 别名 | `宝适-JAC 老线` | 用户给的业务名，用户日常口中的"XXX" |
| `hostname` | `PROJ-0013-SmartQuality-AI-宝适JAC` | 机器原始主机名 |
| `device_id` | UUID 硬件 ID | 系统迁移 / 换机识别用 |
| `os_type` / `version` | `windows` / `v1.29.602` | 配合筛选 |
| `host_type_id` | 1 / 2 / 3 | 宿主机类型（如 `产线套装` / `公司服务器` / `工程师套装`） |
| `workgroup_ids` | list | 所属工作组（如 `AI员工`） |

**关键理解**：同一台物理机可能对应**两个 agent_id**：
- 系统服务 Agent：`windows-<deviceid>`，以 SYSTEM 身份运行
- 用户会话 Agent：`windows-<deviceid>-<username>`（如 `-hp` / `-bestfunc`），以真实用户身份运行

用户会话能访问 WSL、用户目录；系统服务权限更高但看不到用户桌面。**根据任务类型选对 agent_id**。

## 模糊匹配套路

### 用户说名称（"116 服务器"、"宝适 JAC"）→ 找 id

```
1. list_agents()           # 拿全量列表
2. 在 display_name（"116服务器"）或 hostname 里做模糊匹配
3. 若多个候选 → 列出来让用户确认
```

### 用户说业务组（"AI员工组下所有在线机器"）

```
1. list_groups()                      # 确认有这个组
2. list_agents_by_group(workgroup="AI员工")
3. 过滤 is_online == true
```

### 用户说类型（"所有产线套装"、"公司 Linux 服务器"）

```
1. list_host_types()                  # 确认类型名
2. list_agents_by_host_type(host_type="产线套装")
3. 按 os_type 或 version 再过滤
```

### 需要知道某台机器上能跑啥（SQL 预设、Web 预设、feature 开关）

```
get_agent_config(agent_id="xxx")
```

返回：
- `features.cmd / sql / api / file / desktop / dependency / s3` — 功能开关
- `sql_presets` — 预配置的 SQL DSN 列表（execute_sql / execute_select 用）
- `s3_presets` — S3 连接列表
- `web_presets` — Web 浏览的 iframe URL

调 `execute_select` 前先 `get_agent_config` 看 `sql_presets`，避免猜错 preset 名。

## 常见场景

### "帮我查下所有在线的产线机器磁盘使用率"

```
1. list_agents_by_host_type("产线套装")
2. 过滤 is_online
3. 对每台跑 run_safe_command("df -h")（Linux）或 run_safe_command("dir C:\\")（Windows）
4. 结果汇总成表
```

### "找 '116 服务器'"

```
list_agents() → 在输出中找 display_name 含 "116"
→ 唯一命中 linux-116，即它
```

### "宝适-JAC 老线上的 SmartQuality 算法在不在跑"

```
1. list_agents() → 找 display_name="宝适-JAC 老线"
   → 通常有 windows-xxx 和 windows-xxx-bestfunc 两个 id
2. 算法服务在 WSL Docker 里 → 要用 -bestfunc 用户会话 agent
3. get_agent_config(windows-xxx-bestfunc) 确认 features.cmd=true
4. run_safe_command("wsl -- docker ps | grep smart_quality")
```

## 注意事项

- `list_agents` 结果会受工作组的 `allowed_host_type_ids` 限制，看不到就是权限问题，不是 bug
- **在线状态** 30 秒滞后：刚刚重启的 Agent 可能显示离线
- 宿主机类型 0 = **未分类**，`list_agents_by_host_type` 查不到它们
- 用户用中文名（`宝适-JAC`）时注意 display_name 字段里的**连字符**可能是 `-` 也可能是 `—` 甚至空格，宽容匹配
