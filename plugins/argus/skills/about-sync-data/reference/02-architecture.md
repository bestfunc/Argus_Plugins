# 02 · 系统架构

## 定义

Sync-Data 采用 **客户端 / 服务端双进程 + gRPC 双向通信** 架构。客户端部署在源端机房（每个源一个客户端实例），服务端部署在中心机房（全局一个实例）。**没有消息队列中间件、没有分布式调度器**，整体是一个简单可控的二端结构。

## 端到端数据流

```
┌─────── 源端机房 ─────────┐                ┌─────── 中心机房 ────────┐
│                          │                │                          │
│  ① 业务 MySQL  ──────┐   │                │   sync-server (h2c)      │
│       (源)           │   │                │      ┌────────────────┐  │
│                      ▼   │                │      │ ② gRPC :50051  │  │
│  ② sync-client       ┌───┴────┐   gRPC    │      │ ③ Receiver    │  │
│      ├ Scanner       │ Sender ├───────────┼──►   │ ④ Validator    │  │
│      ├ BlockStore    │        │ SyncBlocks│      │ ⑤ Writer       │  │
│      ├ Sender        └────────┘ + h2c     │      └────────┬───────┘  │
│      ├ Heartbeat     ┌────────┐ Heartbeat │               │          │
│      ├ NetworkMon    │  Hb    ├◄──────────┼──►            ▼          │
│      ├ Checkpoint    └────────┘ pacing↓   │     ⑥ 目标 MySQL          │
│      └ ResourceLimit                       │     ⑦ 目标 MinIO          │
│                                            │                          │
│  ③ 源 MinIO (文件)   ←────  Scanner 拉    │     ⑧ Admin HTTP :8090   │
└──────────────────────────                  └──────────────────────────┘
```

数据走向（编号对应上图）：
1. **scanner** 按 `incremental_interval_sec` 周期 `SELECT * FROM <root_table> WHERE updateTime > <checkpoint> LIMIT <batch>` 拉源库行
2. **scanner** 对 file_block 表，递归查 `related[]` 关联表 + 从 **③ 源 MinIO** 异步下载关联文件到本地块目录
3. **blockstore** 把组装好的 block 写入 `data/blocks/YYYY-MM-DD/<blockID>/`，标记 ready
4. **sender** 按优先级（file_block P1 > 增量 simple P2 > 历史 simple P3）出队
5. **sender** 走 **SyncBlocks** 流式 RPC，分片上传到服务端（h2c · 16 MB 上限）
6. 服务端 **Receiver** 收齐后交 **Validator** 校验幂等（`processed_blocks.json` 命中则直接 ACK）
7. **Writer** 写目标 **MySQL**（INSERT/UPSERT，DDL 自动同步 schema） + 上传文件到目标 **MinIO**
8. 服务端 ACK → 客户端删除本地块、推进 checkpoint
9. **Heartbeat** unary RPC 双向带状态：客户端 → 进度统计；服务端 → 下发 pacing 策略

## 客户端模块（14 个 internal 包）

| 包 | 职责 | 关键参数 |
|---|---|---|
| `scanner` | 扫表主循环 · 按优先级生成 block | `incremental_interval_sec` `batch_size` `row_sleep_ms` |
| `block` | Block / TableData / FileRef 数据结构 + checksum | SHA256 |
| `blockstore` | 本地块存储 · 按日期分子目录 · rootIndex 索引 | `data/blocks/YYYY-MM-DD/` |
| `sender` | 块上传 · 优先级队列 · 限速 · 重试退避 | `retry_*` `network_limit_kbps` |
| `heartbeat` | 心跳 RPC · 拉服务端 pacing 策略 | `heartbeat_interval_sec=30` |
| `network` | NetworkMonitor · gRPC Heartbeat 探测代替 TCP dial | `probe_interval_sec=15` |
| `sqmode` | SmartQuality 专用 mode · sq_inbox 监视 · ChunkUpload | 8 MiB chunk |
| `modes` | mode 注册表（default / sqMode）· 服务端可下发切换 | v3.2 引入 |
| `policy` | PolicyStore · 合并本地 YAML + 服务端下发 | `pacing_version` |
| `checkpoint` | 每表最后同步行时间戳 · 定期刷盘 | `data/checkpoint.json` |
| `resource` | 内存监控 · `debug.SetMemoryLimit` · 触发限速 | `memory_limit_mb=200` |
| `schemacache` | 缓存源库表结构 + CREATE TABLE DDL | 启动加载 |
| `status` | HTTP 状态端口 (v3.2) | `/healthz` `/metrics` |
| `config` | YAML 加载 + 默认值 + 环境变量替换 | — |

详细模块图：见 `docs/系统架构图.html` Tab② "系统模块"。

## 服务端模块（8 个 internal 包）

| 包 | 职责 | 关键参数 |
|---|---|---|
| `receiver` | gRPC service 实现 · 认证拦截器 · 块路由 | `MaxRecvMsgSize=16MB` |
| `writer` | BlockWriter · MySQL INSERT/UPSERT · MinIO put_object · DDL 自动同步 | — |
| `validator` | 块幂等去重 · TTL 清理 | `processed_blocks_ttl=24h` |
| `clientstore` | 在线状态 · 心跳超时检测 | `90s` 无心跳 → offline |
| `syncstate` | 全局状态 `sync_state.json` · 每客户端 pacing + 目标库配置 | `pacing_version` |
| `admin` | HTTP 管理面 (`:8090`) · CheckpointSampler 采样进度 | `checkpoint_sample_interval_sec=60` |
| `config / traceid / model` | 基础设施 | — |

## gRPC 接口清单

定义在 `sync-client/proto/sync.proto` 和 `sync-server/proto/sync.proto`（保持同步）。

| RPC | 类型 | 方向 | 用途 |
|---|---|---|---|
| `SyncBlocks` | client streaming | client → server | 分片上传 block 数据 + 文件 |
| `Heartbeat` | unary | client ↔ server | 心跳 · 状态汇报 · 拉 pacing 策略 |
| `Probe` | unary | client → server | 网络探测 · 比 TCP dial 更可靠 |
| `ChunkUpload` | client streaming | client → server | sqMode (v3.2) MinIO multipart 中继 |
| `RegisterClient` | unary | client → server | 首次上线注册 · 返回 token |
| `ReportStatus` | unary | client → server | 运行时状态汇报 (v3.2) |

## 持久化文件布局

### 客户端

```
data/
├── blocks/
│   ├── YYYY-MM-DD/<blockID>/    块目录 (meta.json + files)
│   └── legacy/                  老格式 block 迁移目录 (v3.1)
├── blockstore_state.json        rootIndex 索引快照
├── checkpoint.json              每表最后同步时间戳
├── client_id.dat                持久化 client_id (首次启动生成)
└── dead_letters/                失败超过上限的 block 归档
```

### 服务端

```
data/
├── sync_state.json              每客户端配置 + 统计 + pacing
├── processed_blocks.json        幂等去重表 · TTL 24h
├── checkpoint_samples.json      admin 进度面板的样本快照
├── temp/                        块接收临时目录
├── server.log                   stdout + stderr 统一 append
└── *.bak.YYYYMMDD_HHMMSS        关键 json 自动备份
```

## 关键运行时常量

| 常量 | 值 | 说明 |
|---|---|---|
| gRPC `MaxRecvMsgSize` | **16 MB** | v3.x 把默认 4 MB 调到 16 MB（commit `46e9dee`）· 防大字段表卡死 |
| gRPC `MaxSendMsgSize` | **16 MB** | 同上 |
| gRPC `MaxConcurrentStreams` | `max_connections=50` | 服务端单进程同时承载的 stream 上限 |
| KeepAlive `MinTime` | 10s | 客户端 keepalive ping 频率下限 |
| 客户端 offline 阈值 | 90s | 服务端 90s 没收到 heartbeat 标记 offline（commit `6012bce` 修过 ns/s 单位 bug） |
| sender 退避 | 30s × 1.5 ≤ 300s | 块上传失败时的退避（连续 10 次失败 → 暂停 sender） |
| `processed_blocks` TTL | 24h | 服务端去重表保留窗口 |
| sqMode chunk size | 8 MiB | MinIO multipart 最小 5 MB · 给 16 MB 上限留余量 |

## 通信链路

- **客户端 → 服务端**：直连 `host:50051` 或经 **Nginx 反代**（支持 `/grpc` 路径分流，commit `738c828`）。生产典型链路：`argus-sync.bestfunc.com:30020 → nginx → :50051`。
- **传输**：HTTP/2 over TCP（h2c · 无 TLS · `tls.enabled: false` 是默认）。TLS 可选（`server.tls.enabled: true` + `cert_file/key_file`）。
- **认证**：HMAC token 或预置 `tokens[]` 列表 · `secret_key` 做对称签名。

## 详细图谱

见 `docs/系统架构图.html` —— 包含 5 个视图：系统架构 / 系统模块 / 可配置项 / 块出块流程 / 技术全景。
