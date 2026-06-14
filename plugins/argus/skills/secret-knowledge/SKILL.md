---
name: secret-knowledge
display_name: 运维秘籍速查
description: 海量运维/安全/网络/DevOps 速查知识库（vendored 自 GitHub the-book-of-secret-knowledge，22.8 万 star，已按分类拆成 11 个小文件）。当用户问某类命令、one-liner、CLI/Web 工具、排查技巧、加固/协议/证书速查时，先按分类选对应 reference 文件，再用 Grep 检索作答。配合 troubleshoot-playbook 当"命令/工具字典"用。
user-invocable: true
allowed-tools: Read, Grep, Glob
---

# 运维秘籍速查（The Book of Secret Knowledge）

运维/安全/网络/DevOps 知识合集，vendored 自 [trimstray/the-book-of-secret-knowledge](https://github.com/trimstray/the-book-of-secret-knowledge)（MIT，22.8 万 star）。内容已**按分类拆成 11 个小文件**放在 `reference/` 下，每个 2–42KB。

## ⚠️ 核心用法：先选分类文件，再 Grep，别整读

1. 按用户问题类型，从下表挑 **1–2 个** reference 文件
2. 对选中的文件 `Grep "<关键词>"` 命中行
3. `Read` 命中行附近 ±20 行，拿到完整条目（命令 + 注释 + 出处链接）
4. 给用户：命令/工具 + 来源链接，并按当前系统/权限裁剪

**不要一次读整个文件，更不要把多个文件全读进来** —— 用 Grep 精确命中。

## 分类索引（reference/ 下文件 → 何时查）

| 文件 | 内容 | 什么问题查它 |
|---|---|---|
| `oneliners-system.md` | 进程/文件/系统一行流：`ps` `lsof` `find` `strace` `top` `vmstat` `du` `tar` `dd` `kill` `chmod` `screen` `inotifywait` 等 | 看进程、找文件、查端口占用、系统资源、磁盘、打包 |
| `oneliners-network.md` | 网络一行流：`curl` `ssh` `tcpdump` `nmap` `netcat` `socat` `netstat` `dig` `host` `rsync` `hping3` `ngrep` 等 | 抓包、扫端口、测连通、传文件、DNS、隧道 |
| `oneliners-crypto.md` | 加密/证书一行流：`openssl` `gpg` `secure-delete` `dd` | 证书、密钥、加解密、安全擦除、TLS 排查 |
| `oneliners-text.md` | 文本/脚本一行流：`awk` `sed` `grep` `perl` `python` `git` | 日志分析、文本处理、批量替换、git 操作 |
| `shell-tricks.md` | Bash 技巧 + 实用 shell 函数 | shell 小技巧、重定向、快捷写法 |
| `tools-cli.md` | CLI 命令行工具大全（按用途索引） | "有没有做 XX 的命令行工具" |
| `tools-gui-web.md` | GUI 工具 + Web 在线工具（DNS/证书/抓包/编码/whois） | 桌面工具、在线小工具 |
| `systems-networks-containers.md` | 系统/服务 + 网络 + 容器/编排 的工具与资源 | 系统服务、容器、网络架构选型 |
| `hacking-pentest.md` | 安全/渗透测试工具与资源 | 渗透、漏洞、红队、安全测试 |
| `cheatsheets-news.md` | 速查表合集 + 每日知识/资讯源 | 找某主题 cheat sheet、学习资源 |
| `manuals-lists-blogs.md` | 手册/教程 + 精选 awesome 列表 + 博客/播客/视频 | 想深入某主题的手册/教程/优质博客 |

不确定选哪个时：先 `Grep "<关键词>" reference/*.md -l` 看命中哪些文件，再读最相关那个。

## 注意事项

- 这是**通用运维知识**，与 Argus 的 MCP 工具无关；排查 Argus agent（见 `troubleshoot-playbook`）时，可作为"命令/工具字典"配合用。
- 每个文件头部有 vendored 来源 + Snapshot 日期；原仓库会持续更新，需要最新可对照上游。
- 给用户高危命令（删除/重置/网络扫描）前，确认当前系统（Linux/Windows/WSL）、权限和授权范围，别无脑照搬。

## 来源与许可

Vendored from <https://github.com/trimstray/the-book-of-secret-knowledge>（**MIT License**）。原始版权归 trimstray 及贡献者所有，每个 reference 文件头部均保留来源与许可声明。
