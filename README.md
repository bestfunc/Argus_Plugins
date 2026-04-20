# Argus Plugins

[Argus](https://argus.bestfunc.com) 远程管理代理系统的 AI 助手 plugin 市场，支持 Claude Code / Qwen Code 等 CLI。

一条命令接入 14 个 AI skill + 两个 MCP connector（远程 + 本地），覆盖 Agent 盘点、健康检查、故障排查、批量操作、服务器巡检、远程终端、SQL、**大文件传输**、API 代理、隧道管理、远程桌面操控、远程浏览器等场景。MCP 认证走 OAuth，首次使用自动弹出 Argus 浏览器授权页，无需手动配 token。

**双 MCP 架构：**
- `argus` (远程 HTTP) — 38 个工具，走 `https://argus.bestfunc.com/api/mcp`
- `argus-files` (本地 stdio, 由 `@bestfunc/argus-file-mcp` 提供) — 3 个大文件工具，绕开 MCP base64 内联限制，AI context 开销与文件大小完全解耦

## 快速开始

### Claude Code（原生支持）

```bash
# 1. 添加 marketplace（在 Claude Code 会话里输入）
/plugin marketplace add bestfunc/Argus_Plugins

# 2. 安装 argus plugin（包含 14 个 skill + MCP connector）
/plugin install argus@argus-plugins

# 3. 查看 MCP 连接状态
/mcp
# 可以看到两个条目：
#   argus        → 需要认证（远程 MCP）
#   argus-files  → 本地进程，OAuth 在首次调用时触发

# 4. 触发 OAuth 授权（两个 MCP 分别 Authenticate 一次；第二次因同一 session 浏览器闪跳无感）
# 或直接让 AI 调一个 MCP 工具，会自动弹浏览器
# access_token 默认 30 天有效
```

> Node.js 环境：`argus-files` 通过 `npx` 启动，确保 Node ≥ 18 已安装（Claude Code / Qwen Code 通常已满足）。

### Qwen Code（自动转换格式）

```bash
# 1. 安装扩展（marketplace-url:plugin-name 格式）
qwen extensions install bestfunc/Argus_Plugins:argus

# 2. 重启 Qwen Code 让 MCP 配置生效
qwen

# 3. 查看 MCP 连接状态
/mcp
# argus 条目首次会显示 ✗ 已断开 / Needs authentication

# 4. 触发 OAuth 授权
/mcp auth argus
# 或直接让 AI 调一个 MCP 工具；浏览器打开 Argus 授权页，
# 登录 + 同意后自动完成，回到 /mcp 即可看到 ✓ Connected
```

Qwen Code 会自动把 Claude plugin 格式转成 Qwen extensions 格式并写入 `~/.qwen/extensions/argus/qwen-extension.json`。

### 其他支持 MCP 的客户端（Cursor / Zed / Cline 等）

本仓库 skill 是纯 Markdown，按客户端各自的规范复制到对应目录即可；MCP connector 单独按客户端 UI 手动配，URL 填 `https://argus.bestfunc.com/api/mcp`，留空 OAuth Client ID/Secret 走自动注册（OAuth 2.0 + PKCE + RFC 7591 DCR）。

---

**OAuth 授权流程**：首次使用浏览器会跳转到 Argus 授权同意页，登录 Argus 账号并同意授权后，access_token 默认 30 天有效，到期会自动静默刷新。随时可以在 Argus Console → 我 → 已授权应用 里撤销。

## 内置 skill（14 个）

按使用频次分层：

**基础定位 & 诊断**（最常用，先看这几个）

| Slash 命令 | 用途 |
|---|---|
| `/argus:agent-inventory` | 机器盘点 — 从"116 服务器"匹配到 agent_id |
| `/argus:agent-health-check` | 指标查看 — CPU/内存/磁盘/GPU/Top进程 |
| `/argus:troubleshoot-playbook` | 故障排查标准剧本（CPU/磁盘/内存/网络） |

**基础工具**

| Slash 命令 | 用途 |
|---|---|
| `/argus:terminal` | 远程终端命令（默认白名单 L1，任意命令 L3 审批） |
| `/argus:file-transfer` | 上传/下载文件到宿主机 |
| `/argus:sql-query` | SQL 查询（只读 L1 + 全量 L3） |
| `/argus:api-query` | HTTP API 代理（GET L1 + 全方法 L2） |
| `/argus:tunnel` | 端口映射/P2P/direct 三类隧道管理 |

**进阶与组合**

| Slash 命令 | 用途 |
|---|---|
| `/argus:bulk-ops` | 批量操作 — 一组 agent 并行执行同一操作 |
| `/argus:server-audit` | 服务器巡检报告（模板化输出） |
| `/argus:sql-data-export` | SQL 结果导出成 CSV/Excel/JSON |

**专项**

| Slash 命令 | 用途 |
|---|---|
| `/argus:computer-use` | 远程桌面操控（截图/鼠标/键盘，工业软件 GUI） |
| `/argus:remote-browser` | Chrome CDP 远程自动化（比 computer-use 优先） |

**协议说明**

| Slash 命令 | 用途 |
|---|---|
| `/argus:mcp-authorization` | MCP 三级授权协议（L1/L2/L3），所有 L2/L3 工具调用前参考 |

每个 skill 的详细说明见 [plugins/argus/skills/](plugins/argus/skills/) 下对应目录的 `SKILL.md`。

## MCP 工具

本 plugin 会在你的 Claude Code 会话里注册 `argus` MCP server（`https://argus.bestfunc.com/api/mcp`），暴露 38 个远程管理工具，覆盖 agent / command / computer_use / file / sql / proxy / tunnel 七个业务组。

工具权限分三级：

- 🟢 **L1 安全**（18 个）：直接调用
- 🟡 **L2 半可逆**（12 个）：首次调用需邮箱验证码，同 agent 15 分钟内免验证
- 🔴 **L3 不可逆**（8 个）：每次都要邮箱验证码

认证走 OAuth 2.0 + PKCE，access_token 默认 30 天有效，可在 Argus Console → 用户设置 → 已授权应用 随时撤销。

## 更新

```bash
# Claude Code
/plugin marketplace update argus-plugins

# Qwen Code
qwen extensions update argus
```

## 问题反馈

- 问题提交：<https://github.com/bestfunc/Argus_Plugins/issues>
- 商务合作：Great@bestfunc.com

## License

Apache-2.0
