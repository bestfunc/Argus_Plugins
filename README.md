# Argus Plugins

[Argus](https://argus.bestfunc.com) 远程管理代理系统的 Claude Code plugin 市场。

一条命令接入 8 个通用 AI skill + Argus MCP connector，覆盖远程终端、SQL 查询、文件传输、API 代理、隧道管理、远程桌面操控、远程浏览器等场景。MCP 认证走 OAuth，首次使用自动弹出 Argus 浏览器授权页，无需手动配 token。

## 快速开始

```bash
# 1. 添加 marketplace
/plugin marketplace add bestfunc/Argus_Plugins

# 2. 安装 argus plugin（包含 14 个 skill + MCP connector）
/plugin install argus@argus-plugins
```

首次调用 MCP 工具时，Claude Code 会自动打开浏览器跳转到 Argus 授权同意页，登录并同意后即可使用。

## 内置 skill（8 个）

| Slash 命令 | 用途 |
|---|---|
| `/argus:file-transfer` | 上传/下载文件到宿主机 |
| `/argus:terminal` | 远程终端命令（默认白名单 L1，任意命令 L3 审批） |
| `/argus:sql-query` | SQL 查询（只读 L1 + 全量 L3） |
| `/argus:api-query` | HTTP API 代理（GET L1 + 全方法 L2） |
| `/argus:tunnel` | 端口映射隧道管理 |
| `/argus:computer-use` | 远程桌面操控（截图/鼠标/键盘） |
| `/argus:remote-browser` | Chrome CDP 远程控制 |
| `/argus:mcp-authorization` | MCP 三级授权协议说明（L1/L2/L3） |

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
/plugin marketplace update argus-plugins
```

## 问题反馈

- Argus Console：<https://argus.bestfunc.com>
- 问题提交：<https://github.com/bestfunc/Argus_Plugins/issues>
- 商务合作：support@bestfunc.com

## License

Apache-2.0
