---
name: agent-health-check
display_name: Agent 健康检查
description: 快速看一台或一组 Argus Agent 的运行状态：CPU/内存/磁盘/GPU/网络延迟 + top processes。支持单机深度检查、一组机器批量扫描、按阈值告警。基于 L1 工具，直接可用无需审批。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__list_agents_by_group,mcp__argus__list_agents_by_host_type,mcp__argus__get_agent_metrics,mcp__argus__get_agent_config,mcp__argus__run_safe_command
---

# Agent 健康检查

## 核心工具

| 工具 | 用途 |
|------|------|
| `get_agent_metrics` | 核心指标 — CPU / 内存 / 磁盘 / GPU / 网络延迟 / top processes |
| `run_safe_command` | 辅助诊断（`df -h` / `free -h` / `uptime` 等白名单命令） |

## `get_agent_metrics` 返回字段解读

```json
{
  "ts": "2026-04-20T16:50:00",        // 指标采集时间
  "cpu": 4.98,                         // 系统 CPU %
  "mem_percent": 26.23,                // 系统内存 %
  "mem_used": 35421908992,             // 已用字节
  "mem_total": 135034380288,           // 总字节
  "disk_read_bps": 166815150,          // 最近 10s 平均读字节/秒
  "disk_write_bps": 885881,            // 写字节/秒
  "disk_usage_percent": 50.63,         // 系统盘占用
  "net_rx_bps": 1800,                  // 网络接收字节/秒
  "net_tx_bps": 1961,
  "gpu_percent": 0.0,                  // 主 GPU 占用
  "gpu_mem_percent": 0.0,
  "gpu_name": "Tesla P40",
  "gpu_count": 4,                      // 多卡场景
  "gpu_cards": [ ... ],                // 每张卡独立指标
  "latency_ms": 8.1,                   // Agent 到 Server 的 RTT
  "top_processes": {
    "cpu": [{pid, name, value}, ...],  // CPU Top 5
    "gpu": [...],                       // GPU 显存 Top N
    "mem": [...],                       // 内存 Top 5
    "disk": [...]                       // 磁盘 IO Top 5
  }
}
```

## 单机检查（最常见）

```python
# 前置：用 agent-inventory skill 找到 agent_id
get_agent_metrics(agent_id="linux-116")
```

**判定参考**：

| 指标 | 正常 | 偏高 | 告警 |
|------|------|------|------|
| `cpu` | <30% | 30-70% | >70% |
| `mem_percent` | <60% | 60-85% | >85% |
| `disk_usage_percent` | <70% | 70-85% | >85% |
| `latency_ms` | <50ms | 50-200ms | >200ms 或 null（离线） |
| `gpu_percent` | 按业务判断 | — | — |

**告警时追加深度排查**：

```python
run_safe_command(agent_id=..., command="df -h")     # 磁盘细分
run_safe_command(agent_id=..., command="free -h")   # 内存细分
run_safe_command(agent_id=..., command="uptime")    # 负载 1/5/15 min
run_safe_command(agent_id=..., command="ps aux")    # Linux 进程
run_safe_command(agent_id=..., command="tasklist")  # Windows 进程
```

## 批量扫描（给一组机器打分）

用户常问"哪些机器 CPU 最忙？" → 套路：

```python
# 1. 用 agent-inventory skill 找一组 agent
agents = list_agents_by_group(workgroup="AI员工")
online = [a for a in agents if a.is_online]

# 2. 并行拉每台的 metrics（AI 在 tool call 层面可以并行）
for a in online:
    metrics[a.id] = get_agent_metrics(a.id)

# 3. 按 cpu 降序
sorted_by_cpu = sorted(metrics.items(), key=lambda x: x[1].cpu, reverse=True)

# 4. 列 markdown 表给用户
```

Claude Code / AI 客户端一般支持在一个响应里发多个 tool call 并行，**直接一口气并发调 5-10 个 agent 的 metrics**，比串行快 N 倍。

## 典型问答剧本

### "116 服务器磁盘满了吗？"

```
get_agent_metrics("linux-116") → disk_usage_percent 84%
run_safe_command("linux-116", "df -h") → 某个挂载点 96%
回答：系统盘 84% 还好，/mnt/disk2 已 96% 快满，top dir 是 ...
```

### "GPU 空闲的机器有哪些？"

```
list_agents_by_group("AI员工")
对每台 get_agent_metrics → 过滤 gpu_percent < 5 且 gpu_mem_percent < 10
```

### "昨天晚上 8 点某服务器卡了一下，现在还有问题吗？"

```
get_agent_metrics 拿的是**当前**快照，没历史
→ 只能判断当前状态
→ 如需历史，请用户去 Argus Console → Dashboard → 指标面板（有时间范围图表）
```

## 和其他 skill 的关系

- 发现**异常**后要排查具体进程/日志 → 转 `troubleshoot-playbook`
- 要批量跑同一条排查命令 → 转 `bulk-ops`
- 定期生成巡检报告 → 转 `server-audit`
- 不知道 agent_id 是啥 → 先 `agent-inventory`

## 注意事项

- `get_agent_metrics` 是 L1，**免验证码**，想并发扫 50 台机器完全 OK
- `top_processes.cpu` / `mem` 的 `value` 字段是百分比或字节（看 key）
- GPU 指标仅 NVIDIA 可用，Intel/AMD 会全 0（不是 bug）
- Agent 离线时 `get_agent_metrics` 返回 `AGENT_OFFLINE` 错误，不要当成健康状态
- 诊断 Windows 机器上 WSL 里的负载要用**用户会话 agent**（-hp/-bestfunc 后缀），系统服务 agent 看不到 WSL 进程
