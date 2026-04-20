---
name: bulk-ops
display_name: 批量操作
description: 对一组 Argus Agent 并行执行同一操作（跑命令 / 取指标 / 查文件等）并汇总结果。核心套路：先 agent-inventory 筛出目标列表，再并行发 tool call。只读操作（L1）推荐，写操作（L3）逐台审批，不建议批量。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__list_agents_by_group,mcp__argus__list_agents_by_host_type,mcp__argus__run_safe_command,mcp__argus__get_agent_metrics,mcp__argus__list_files,mcp__argus__read_file,mcp__argus__execute_select
---

# 批量操作

## 核心能力

把"查 10 台机器的磁盘使用率"、"所有产线机器里有没有 BestPLC 进程在跑"这类**同一操作 × N 台机器**的任务一次性搞定。

关键：**MCP 客户端（Claude Code / Cursor / Zed 等）支持单轮消息里发多个 tool call 并行**，调 10 台 agent 的 `get_agent_metrics` 用 1 轮就能完成，比串行快一个数量级。

## 标准流程

```
Step 1: 用 agent-inventory skill 筛选目标 agent 列表
  list_agents / list_agents_by_group / list_agents_by_host_type
  + 过滤 is_online

Step 2: 对列表**并行**发起 tool call
  一次消息里同时调 N 个 run_safe_command / get_agent_metrics / ...
  不要一个个等结果

Step 3: 在 Claude Code 本地聚合、排序、筛选
  直接在聊天里画 markdown 表，或用 Bash 工具写脚本二次处理
```

## 推荐使用的 L1 工具（无审批）

- `run_safe_command` — 白名单只读命令（ps / df / uptime / netstat / tasklist / ...）
- `get_agent_metrics` — 硬件指标
- `get_agent_config` — feature / preset
- `list_files` / `read_file` — 查文件/读小文件
- `execute_select` — SQL 查询

## 典型场景

### "所有在线 Linux 公司服务器的磁盘使用率"

```
Step 1: agents = list_agents_by_host_type("公司服务器")
        online_linux = [a for a in agents if a.is_online and a.os_type=='linux']
Step 2: 并行调用：
        get_agent_metrics(a.id) for a in online_linux
Step 3: 汇总：
        | agent | disk_usage_percent | 状态 |
        |-------|--------------------|------|
        | linux-116 | 84%  | ⚠️ 偏高 |
        | linux-51  | 32%  | ✅ 正常 |
        ...
```

### "所有产线机器上 BestPLC.exe 进程的 CPU / 内存占用"

```
Step 1: agents = list_agents_by_host_type("产线套装")
        online = [a for a in agents if a.is_online]
Step 2: 并行：
        run_safe_command(a.id, "tasklist") for a in online
Step 3: 在输出里 grep "BestPLC.exe"，提取 PID + 内存列，汇总
```

注意：`run_safe_command` 禁止管道，**不能**直接 `tasklist | findstr BestPLC`，要在 Claude Code 本地把完整 tasklist 输出做文本搜索。

### "哪些产线机器的 argus-agent 版本不是最新的"

```
Step 1: list_agents_by_host_type("产线套装")
Step 2: 不用再发 MCP 调用，list_agents 返回就带 version
Step 3: 过滤 version != "v1.30.xxx"，列出来
```

### "批量检查各机器的 SmartQuality 服务状态"

```
Step 1: list_agents_by_group("AI员工")（业务在这个组）
Step 2: 并行 run_safe_command(a.id, "systemctl status smartquality")
        Windows 机器用 run_safe_command(a.id, "sc query smartquality")
        # sc 和 systemctl 都在白名单且限定了只读 subcommand
Step 3: 从输出 grep 关键字："active (running)" / "RUNNING" / ...
```

### "拉所有公司服务器的 /etc/hostname"

```
Step 1: list_agents_by_host_type("公司服务器")
Step 2: 并行 read_file(a.id, "/etc/hostname")
Step 3: 结果就是 hostname 列表
```

## 并行调用的正确姿势

**好**（一条消息里发多个 tool call，MCP 客户端并发处理）：

```
[tool_call: get_agent_metrics(linux-48)]
[tool_call: get_agent_metrics(linux-49)]
[tool_call: get_agent_metrics(linux-50)]
[tool_call: get_agent_metrics(linux-51)]
[tool_call: get_agent_metrics(linux-52)]
```

**差**（每次一个，N 轮对话）：

```
assistant: 我先查 linux-48
  → tool_call: get_agent_metrics(linux-48)
  [等结果]
assistant: 再查 linux-49
  → tool_call: get_agent_metrics(linux-49)
  ...
```

前者 10 台机器约 2-5 秒；后者 30-60 秒。

## 写操作（L2 / L3）慎用批量

**不要**用这个 skill 对 N 台机器批量发起 `run_command` / `upload_file` / `execute_sql`：

- 每一条 L2/L3 都要用户邮箱验证码，N 次 = N 封邮件 + N 次确认，用户会疯
- 误操作放大 N 倍，破坏力不可逆

若真的需要批量修改：
1. **先**用本 skill L1 工具确认目标列表对、影响范围清
2. 告诉用户具体要做什么、影响哪些机器、回滚方案
3. 让用户确认后，**每台机器单独**发 L3 调用，每次都确认
4. 或者让用户通过 Argus Console 的"批量推送更新" / "批量配置" 等 UI 操作（Console 里有批量能力 + 审计）

## 结果汇总建议

- 机器数 ≤ 15：直接画 markdown 表
- 机器数 > 15：用 Bash 工具写 CSV / 存到本地文件，告诉用户路径
- 关键字段按**语义排序**（CPU% 降序、磁盘使用率降序、离线时间升序）
- 异常行高亮（⚠️ / 🔴 emoji 标记）

## 关联 skill

- `agent-inventory` — 筛选目标
- `agent-health-check` — 单机深度指标
- `troubleshoot-playbook` — 发现异常后怎么排查
- `server-audit` — 定期巡检报告（更结构化的批量）
