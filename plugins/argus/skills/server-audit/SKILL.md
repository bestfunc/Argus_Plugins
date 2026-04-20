---
name: server-audit
display_name: 服务器巡检
description: 对一组 Argus 管理的机器做标准化巡检，输出结构化 Markdown 报告。典型场景：每周/每月对"公司服务器"或"产线套装"做一遍健康体检。全程 L1，无需审批。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__list_agents_by_group,mcp__argus__list_agents_by_host_type,mcp__argus__get_agent_metrics,mcp__argus__get_agent_config,mcp__argus__run_safe_command
---

# 服务器巡检

和 `bulk-ops` 的区别：`bulk-ops` 是**通用批量框架**，这个 skill 是**固定模板的巡检报告**。

## 用户触发

用户说"巡检下所有公司服务器" / "每月体检报告" → 套这套固定流程。

## 巡检范围选择

```
1. list_host_types() 看可选类型
2. 问清用户要巡检哪一组：
   - "公司服务器"（常用，Linux 为主）
   - "产线套装"（Windows 产线机）
   - "工程师套装"（员工电脑）
   - 或按工作组：list_agents_by_group(...)
```

## 标准巡检项（全部 L1）

对每台 **在线** 机器采集：

### 基础信息（从 list_agents 返回就有）
- agent_id / display_name / hostname
- os_type / version / last_seen_at
- workgroups / host_type

### 指标（get_agent_metrics）
- CPU%
- 内存% + 总量 + 已用
- 磁盘主分区% + 读写 bps
- GPU%（如有）
- Agent-Server latency_ms
- Top 3 CPU / 内存 / 磁盘 IO 进程

### 系统状态（run_safe_command，Linux）
- `uptime` — 负载 1/5/15 min
- `df -h` — 所有分区使用率（不只是主盘）
- `free -h` — 内存细分（buffer/cache vs used）
- `ss -tln` 或 `netstat -tln` — 监听端口

### 系统状态（run_safe_command，Windows）
- `systeminfo` — 启动时间 / OS / 内存
- `tasklist` — 进程列表
- `ipconfig` — 网络
- `sc query` — 主要服务状态（若要看具体服务需白名单允许的 subcommand）

### Agent 配置（get_agent_config）
- features 开关是否符合预期
- sql_presets / s3_presets / web_presets 是否还有效

## 报告模板

```markdown
# Argus 服务器巡检报告

- 巡检范围: 公司服务器 (17 台，其中 15 台在线)
- 巡检时间: 2026-04-20 17:00
- 巡检人: Claude (Argus MCP)

## 概览

| 指标 | 健康 | 偏高/警告 | 告警 |
|------|------|----------|------|
| CPU  | 12   | 2        | 1    |
| 内存 | 13   | 2        | 0    |
| 磁盘 | 11   | 3        | 1    |
| 延迟 | 15   | 0        | 0    |

## 需要关注

### 🔴 告警

1. **linux-116** — 磁盘 /mnt/disk2 已 96%
   - 建议：清理 /mnt/disk2/old_data/2024-*

### 🟡 警告

1. **Aliyun-Server** — CPU 68% 连续走高
   - top: qdrant (pid 896383) 占 42%
2. **linux-176** — 内存 82%，主要是 java (18GB)

## 全量明细

| 机器 | 状态 | CPU | 内存 | 磁盘 | 延迟 | 版本 | 备注 |
|------|------|-----|------|------|------|------|------|
| linux-48 | ✅ | 5%  | 25% | 42% | 8ms  | v1.29.602 | — |
| linux-49 | ✅ | 12% | 38% | 51% | 10ms | v1.29.602 | — |
...

## 版本分布

- v1.30.x: 12 台（建议升级目标）
- v1.29.x: 3 台（待升级）
- 其他: 0

## 离线机器

| 机器 | 最后在线 | 离线时长 |
|------|---------|---------|
| windows-5eca...-bestf | 2026-03-15 11:22 | > 1 个月 |
| windows-d61c...-hp | 2026-04-18 09:31 | 2 天 |

建议确认这些机器是否已停用。
```

## 执行节奏

1. **筛列表**：`list_agents_by_host_type("公司服务器")` → 拿到全量
2. **过滤在线**：`is_online == True`
3. **并行采**：**一口气发** 15 个 `get_agent_metrics` tool call（别串行）
4. **并行辅助诊断**：同一轮再发 15 个 `run_safe_command("df -h")`
5. **Agent 配置**：并行 `get_agent_config`
6. **在 Claude Code 本地拼报告**：按模板输出 Markdown，或用 Bash 工具 `cat > report.md`

## 阈值参考

| 指标 | 健康 | 偏高/警告 | 告警 |
|------|------|----------|------|
| CPU | <30% | 30-70% | >70% 或持续 > 50% |
| 内存 | <60% | 60-85% | >85% |
| 磁盘（主盘） | <70% | 70-85% | >85% |
| 磁盘（非系统盘） | <80% | 80-90% | >90% |
| latency_ms | <50ms | 50-200ms | >200ms |
| 版本 | 最新 | 落后 1 个 minor | 落后 > 2 个 minor 或 > 60 天 |

根据业务调整（比如 GPU 机器 gpu_percent 90% 不一定是问题 —— 正常跑训练）。

## 定期化

Argus 没内置"定期巡检"调度（没实现 cron 任务）。外部调度：
- Claude Code 里用 `/loop 1d /argus:server-audit` 每日自动跑
- 或者在 CI / crontab 里用 Bash + argus MCP 脚本触发（需要 MCP API Key）

## 注意事项

- **离线机器**不是异常 — 可能是主动下线、调试、重装。报告里列出来让用户判断
- 产线机器 CPU 常期高（算法推理）属正常，用阈值判断会误报
- 巡检要**避开**业务高峰时段，别在产线忙的时候拖累它（虽然 get_agent_metrics 开销几乎为零）
- 大规模（50+ 台）巡检时，AI 对话可能遇到响应长度限制 — 可以分批（每批 15 台）或写 Bash 工具生成 CSV
