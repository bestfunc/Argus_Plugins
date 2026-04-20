# Argus Plugins

[Argus](https://argus.bestfunc.com) 远程管理代理系统的 AI 助手 plugin 市场，支持 Claude Code / Qwen Code 等 CLI。

一条命令接入 14 个 AI skill + Argus MCP connector，覆盖 Agent 盘点、健康检查、故障排查、批量操作、服务器巡检、远程终端、SQL、文件传输、API 代理、隧道管理、远程桌面操控、远程浏览器等场景。MCP 认证走 OAuth，首次使用自动弹出 Argus 浏览器授权页，无需手动配 token。

## 快速开始

### Claude Code（原生支持）

```bash
# 1. 添加 marketplace
/plugin marketplace add bestfunc/Argus_Plugins

# 2. 安装 argus plugin（包含 14 个 skill + MCP connector）
/plugin install argus@argus-plugins
```

### Qwen Code（自动转换格式）

```bash
qwen extensions install bestfunc/Argus_Plugins:argus
```

格式 `marketplace-url:plugin-name` — 冒号前是本仓库，冒号后是 plugin 名（`argus`）。Qwen Code 会自动把 Claude plugin 格式转成 Qwen extensions 格式并落地。

### 其他支持 MCP 的客户端（Cursor / Zed / Cline 等）

本仓库 skill 是纯 Markdown，按客户端各自的规范复制到对应目录即可；MCP connector 单独按客户端 UI 手动配，URL 填 `https://argus.bestfunc.com/api/mcp`，留空 OAuth Client ID/Secret 走自动注册。

---

**首次调用 MCP 工具**时，CLI 会打开浏览器跳转到 Argus 授权同意页，登录并同意后即可使用（OAuth 2.0 + PKCE，30 天有效）。

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
