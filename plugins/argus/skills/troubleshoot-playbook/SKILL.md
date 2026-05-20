---
name: troubleshoot-playbook
display_name: 故障排查手册
description: 标准故障排查剧本。用户抱怨"XX 机器有问题"时先套这个 skill 定位问题性质，再导向具体 skill（健康检查 / 终端 / SQL / 文件）。核心原则：梯度升级权限，能 L1 解决绝不升 L3。
user-invocable: true
allowed-tools: mcp__argus__list_agents,mcp__argus__get_agent_metrics,mcp__argus__get_agent_config,mcp__argus__run_safe_command,mcp__argus__read_file,mcp__argus__list_files,mcp__argus__run_command
---

# 故障排查手册

## 核心原则

1. **先诊断，后动手**：用 L1 工具摸清症状，不要上来就 `run_command` 要 L3 审批
2. **梯度升级**：L1（观察）→ L2（深度读敏感文件）→ L3（改东西/重启服务）
3. **让用户闭环**：每个 L3 动作前向用户说明清楚理由和回滚方案

## 接到问题时的入口

```
用户说"XXX 机器慢/连不上/有问题"
    ↓
Step 0: 定位 agent_id         → agent-inventory skill
Step 1: get_agent_metrics    → 看大局（CPU / 内存 / 磁盘 / 网络 / GPU）
Step 2: 按症状选剧本          → 见下方
Step 3: 动手修（必要时）      → run_command（L3 审批）
```

## 剧本 A：CPU 高

```
1. get_agent_metrics(agent_id)
   → top_processes.cpu 看占用前 5 的进程
2. Linux:
     run_safe_command("ps aux")       # 看完整进程列表（L1，无副作用）
     run_safe_command("uptime")       # 看负载 1/5/15 min
3. Windows:
     run_safe_command("tasklist")     # 进程列表
     run_safe_command("systeminfo")   # 系统整体信息
4. 定位恶性进程 → 询问用户是否要 kill
     run_command("kill -9 <pid>")     # L3，每次要审批 + reason
```

## 剧本 B：磁盘满

```
1. get_agent_metrics → disk_usage_percent
2. run_safe_command("df -h")          # Linux 细分
   run_safe_command("dir C:\\")       # Windows 系统盘
3. 定位大目录：
   Linux:
     # 注意 run_safe_command 禁止管道，du 单独跑
     run_safe_command("du -h /var")   # 逐级下钻
   Windows:
     run_command("dir /s C:\\...")    # 需要 L3（有写风险的目录 walk）
4. 清理：
     run_command("rm -rf /tmp/xxx")   # L3 审批
```

**特别场景：Docker 磁盘占用**
```
run_safe_command("docker ps -a")          # L1 看容器状态
run_command("docker system prune -f")     # L3 清理
```

## 剧本 C：内存泄漏 / 内存高

```
1. get_agent_metrics → top_processes.mem
2. Linux:
     run_safe_command("free -h")
     run_safe_command("ps aux --sort=-%mem")  # 管道禁止，需要拆
     # 先 ps aux，再在 Claude Code 本地排序，或用 L3 run_command 走管道
3. 若嫌疑进程是自家服务：
     get_agent_config(agent_id) 看 preset
     run_command("systemctl restart <service>")  # L3
```

## 剧本 D：机器连不上 / Agent 离线

```
1. list_agents() → 看 is_online 和 last_seen_at
2. 离线 < 1 分钟 → 大概率短暂抖动，等 30s 再试
3. 离线 > 5 分钟 → 真的挂了，需要：
   - 用 Argus Console → Agents 菜单看最后在线时间、推送历史
   - 若有其他 Agent 在同内网 → 让那台跑 `ping <目标 IP>`
     run_safe_command(agent_id="同内网另一台", command="ping 192.168.1.x")
4. 本机无法联网时要现场重启（无法远程修）
```

## 剧本 E：日志检查

```
1. 先 list_files 浏览日志目录：
     list_files(path="/var/log")        # Linux
     list_files(path="C:\\ProgramData\\...")   # Windows
2. 读最新日志：
     read_file(path="/var/log/syslog")   # L1，但大文件要 tail
     run_safe_command("tail -n 200 /var/log/syslog")   # 实际上 run_safe_command 白名单有 tail
3. 看应用日志时注意权限：系统日志用系统服务 agent，应用日志用用户会话 agent
```

## 剧本 F：数据库 / 应用异常

```
1. get_agent_config(agent_id) → 确认 sql_presets 里有哪些 DSN
2. execute_select("SELECT 1")  → 确认 DB 可达
3. 看应用 status：
     run_safe_command("systemctl status myapp")   # L1，subcommand 白名单里
4. 看应用日志 → read_file
5. 必要时重启 → run_command("systemctl restart myapp") L3
```

## 剧本 G：网络慢 / latency_ms 高

```
1. get_agent_metrics → latency_ms 看 Agent-Server 延迟
2. latency_ms > 200ms 通常是跨地域或运营商问题
3. 内网其他节点怎么样 → 用 create_tunnel 映射 ping 目标端口后 proxy_api_get 测
4. 如果是下载/上传慢 → 可能是 Server-MinIO 或 Agent-Server 隧道带宽问题
```

## 何时调用其他 skill

| 症状 | 先跑这个 skill |
|------|---------------|
| 不知道 agent_id | `agent-inventory` |
| 要看指标大局 | `agent-health-check` |
| 要跑具体命令 | `terminal` |
| 要查 DB | `sql-query` |
| 要传/取文件 | `file-transfer` |
| 要调 HTTP API | `api-query` |
| 需要进宿主机 GUI | `computer-use` |
| 需要开端口隧道 | `tunnel` |
| 需要自动化 Chrome | `remote-browser` |

## L3 动作检查清单

在调用 `run_command` / `upload_file` / `execute_sql` 等 L3 工具**之前**，先对用户说清楚：

1. **要做什么**：具体命令 / 文件 / SQL
2. **预期影响**：改什么、影响哪些服务、会不会断流
3. **回滚方案**：万一出问题怎么撤回
4. **让用户确认**再发起（此时还没调 MCP，用户反悔直接取消）
5. 用户同意后再调 → Server 返回 APPROVAL_PENDING → 用户查邮箱验证码 → 继续

**别**一次性提出一串 L3 动作（`重启 + 清理 + 再重启`），这会让用户审批疲劳。能合并的命令合并（一条 bash 包多步），或者分步做、每步都停顿让用户验证效果。

## 注意事项

- `run_safe_command` 禁止 `|` / `>` / `;` / `&`，这意味着经典的 `ps aux | grep foo` / `du -sh */` 不能直接用
  - 解决：抓原始输出到本地，由 AI 在 Claude Code 本地 bash 侧处理
  - 或者升级到 `run_command`（L3 审批）
- Windows 上 `powershell.exe` 和 `pwsh.exe` 不在 run_safe_command 白名单(因为能执行任意代码),要走 `run_command` + `shell="powershell"`(server 自动 base64 处理转义,**不要自己手动 -EncodedCommand**;详见 terminal skill)
- 排查产线机器要小心 —— 真的 `kill -9` 关键业务进程会导致产线停工，**必须**和用户三次确认
