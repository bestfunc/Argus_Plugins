# 01 · 产品总览

## 定义

**Sync-Data** 是一个面向多源、多客户端、跨网络场景的**结构化数据同步系统**。每个客户端从自己机房的源 MySQL（及关联的 MinIO 文件）增量扫描数据，通过 gRPC 推送到中心机房的服务端，由服务端落入**每客户端独立的目标 MySQL + MinIO bucket**。

一句话定位：**用块（block）作为同步原子单位的、关注文件 + 行联合一致性的、可中心化运维多个边缘客户端的结构化数据搬运管线**。

## 5 分钟搞清

```
源端机房 (per client)                  中心机房
┌──────────────────────┐               ┌─────────────────────────┐
│ MySQL (业务库)        │  扫描          │ sync-server (单进程)     │
│ MinIO  (文件)         │ ───────────►  │  ├ gRPC 50051           │
│                       │   gRPC/h2c    │  ├ Admin HTTP 8090      │
│ sync-client (单进程)   │ ◄─────────── │  └ 5 goroutine          │
│  └ 7 goroutine       │   心跳 / 策略 │                          │
└──────────────────────┘               │ 每客户端独立:             │
                                        │  · MySQL 目标库          │
                                        │  · MinIO bucket          │
                                        └─────────────────────────┘
```

工作流程：
1. **scanner** 按 `updateTime` 增量扫源库表 → 生成 block
2. **blockstore** 把 block 持久化到本地 `data/blocks/YYYY-MM-DD/`
3. **sender** 按优先级出队 → gRPC 流式推送
4. **server** 校验幂等 → 写目标 MySQL + 上传 MinIO
5. **heartbeat** 每 30s 互通进度 + 拉服务端下发的限速策略

## 目标场景

**典型客户**：连锁机构 / 多门店企业 / 边缘机房（每个门店一个客户端，中心机房做数据汇聚）。

**适用条件**：
- 源端是 **MySQL**（v3.x 暂不支持 PostgreSQL、Oracle、SQL Server 等异构源）
- 数据中有"**主表 + 关联表 + 文件**"的复合结构（如检测记录 + 检测通道 + 录音文件 .wav），需要保证三者**作为一个原子单位**同步
- 客户端可能**网络不稳定 / 长时间离线 / 带宽受限**，需要本地块队列兜底
- 中心运维要能**远程调速 / 远程开关 / 远程看进度**

**不适用场景**：
- 需要近实时（秒级）binlog 同步 → 用 Canal / Debezium
- 源是异构数据库 → 用 Otter / Flink CDC
- 数据无文件依赖、只是行复制 → 上面那些都更轻

## 当前规模（v3.x 状态）

| 维度 | 现状 |
|---|---|
| 已上线客户端数 | 10+（典型一个客户端对应一家门店 / 分支） |
| 服务端进程 | 1 个中心服务端 · 单 binary · 不分片 |
| 服务端单进程内存峰值 | ~180 MB（稳态运行 3+ 周） |
| 单 block 上限 | 16 MB（gRPC `MaxRecvMsgSize`） |
| 每客户端独立目标库 | `smart_quality_v2_<client_id>` |
| 每客户端独立 MinIO bucket | `smart-quality-<client_id>` |
| 客户端运行平台 | Windows（任务计划）+ Linux/macOS（systemd） |

## 版本节奏

事实来自 `git log` 与 `docs/plans/`、`docs/superpowers/plans/`：

| 版本 | 时间 | 关键变化 |
|---|---|---|
| **V2** | 2026-01 ~ 2026-02 | 早期实现（Redis Stream / 64-slot 路由 / React+Vue 双 UI 设计）。整体已在 commit `1dec55b` 删除 |
| **V3.0** | 2026-03 ~ 2026-05 | 完整重构：gRPC + Block + Scanner + BlockStore + Validator + Writer · 单 binary 单进程 |
| **V3.1** | 2026-05 | blockstore 按日期分子目录 · 老格式 `fb-xxxxxxxx` 迁到 `legacy/` |
| **V3.2** | 2026-05 中下旬 | sqMode（SmartQuality 客户端的 MinIO multipart 中继通道）· modes 注册表 · pacing_version 协议 |
| **V3.3**（在调研 + 实现中） | 2026-06 起 | pull-based pacing · knownTables 缓存 + flush API · 错误分类器 · 隔离基础设施 |

更详细的演进动机与设计：见 [08-roadmap.md](./08-roadmap.md)。

## 不在百科范围里的事

- **真实客户名称 / 门店名** — 百科一律用 `<client_id>` 占位
- **生产 IP / 域名 / 凭证 / Token** — 永不入文档
- **团队成员 / 内部决策辩论 / 营收数据** — 不写
- **`prompt迭代/` 目录** — 是设计草稿，不算事实
- **根目录的 README / architecture-docs / 模块解释.md** — V2 时代遗留，不再准确，**忽略**

## 一句话总结

> Sync-Data 是把"边缘门店 MySQL + 文件 + 业务关联表"原子地同步到中心机房的工程化方案，做了多客户端隔离、限速策略热下发、文件块幂等去重，专门服务于网络不稳的多门店场景。
