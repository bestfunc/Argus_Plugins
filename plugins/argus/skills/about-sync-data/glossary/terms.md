# Glossary · 术语表

> 全部术语 + 缩写 + 跨文件引用入口。按字母 / 拼音排序。

## A

**ALREADY_PROCESSED**
服务端 validator 对幂等块的 ACK code。表示该 `block_id` 在 24h TTL 内已被处理过，本次直接 ACK 不重复落库。**不是错误**，是健康去重信号。日志中出现属于正常。详见 [reference/04-key-concepts.md · 幂等去重](../reference/04-key-concepts.md#5-幂等去重idempotency)。

**admin Web**
服务端的 HTTP 管理界面，默认监听 `:8090`。功能：每客户端 enable/disable、修改 pacing、查看 catchup 倍率。详见 [reference/02-architecture.md](../reference/02-architecture.md#服务端模块8-个-internal-包)。

## B

**Block / 块**
Sync-Data 同步的**原子单位**。要么完全落库，要么不落。两种类型：file_block（含文件 + 关联表）和 simple_block（纯行批）。详见 [reference/04-key-concepts.md · Block](../reference/04-key-concepts.md#1-block块)。

**block_id**
块的全局唯一标识。新格式 `fb-YYYYMMDD-xxxxxxxx` / `sb-YYYYMMDD-xxxxxxxx`（v3.1+），老格式 `fb-xxxxxxxx`（v3.0）。

**BlockReceiver**
服务端 gRPC service 实现，接收客户端推来的 block 数据。源码：`sync-server/internal/receiver/`。

**BlockStore**
客户端本地块队列。文件路径 `data/blocks/YYYY-MM-DD/<block_id>/`。详见 [reference/02-architecture.md · 持久化文件布局](../reference/02-architecture.md#持久化文件布局)。

**BlockWriter**
服务端块写入器，把 block 内的行写到目标 MySQL，文件写到目标 MinIO。源码：`sync-server/internal/writer/`。

## C

**catchup 倍率**
admin 看板展示的指标，表示客户端追上速度相对于源端产生速度的比例。> 1 表示在追上，< 1 表示在落后。

**checkpoint / 同步水位**
客户端 `data/checkpoint.json` 记录每张表"上次扫到 `updateTime` = X"。scanner 下次从 X 继续。详见 [reference/04-key-concepts.md · Checkpoint](../reference/04-key-concepts.md#3-checkpoint同步断点)。

**CheckpointSampler**
服务端 goroutine，每 60s 采样每个客户端目标库的 `MAX(updateTime)`，供 admin 进度面板使用。

**ChunkUpload**
v3.2 sqMode 引入的 RPC，做 MinIO multipart 中继。8 MiB 分片。

**client_id**
客户端的全局唯一标识。生产中**不在百科 / 公开文档中暴露真名**，统一用 `<client_id>` 占位。详见 [reference/01-product-overview.md · 不在百科范围里的事](../reference/01-product-overview.md#不在百科范围里的事)。

**ClientStore**
服务端 goroutine，跟踪每客户端的最后心跳时间，超过 90s 没心跳就标记 offline。

## D

**dead letter / 死信**
重试 ≥ `retry_max_consecutive_failures`（默认 10）次仍失败的 block，搁置到 `data/dead_letters/` 目录。详见 [reference/04-key-concepts.md · 死信](../reference/04-key-concepts.md#7-死信dead-letter)。

**default mode**
客户端默认运行模式（与 sqMode 对应）。scanner 扫表生成 block → sender 走 SyncBlocks RPC。详见 [reference/04-key-concepts.md · Mode](../reference/04-key-concepts.md#2-mode运行模式)。

**Debezium**
开源 MySQL CDC 工具。详见 [reference/06-competitive-landscape.md](../reference/06-competitive-landscape.md)。

## F

**fb-... / file_block**
文件块。优先级 1。含 1 条根行 + 关联表行（递归）+ 文件二进制。详见 [reference/04-key-concepts.md · Block](../reference/04-key-concepts.md#1-block块)。

**file_block_tables**
客户端 `config.yaml.tables.file_block_tables[]` 配置，定义哪些表用 file_block 模型。

**file_fields**
file_block 关联表行中的"文件字段映射"。包含 `field`（路径字段名）和 `bucket_field`（bucket 字段名）。scanner 用它们提取 MinIO 文件引用。

## G

**goroutine**
Go 的轻量级线程。Sync-Data 客户端有 7 个，服务端有 5 个。详见 [reference/02-architecture.md](../reference/02-architecture.md#客户端模块14-个-internal-包)。

**gRPC**
Sync-Data 客户端 / 服务端的通信协议。基于 HTTP/2 + Protocol Buffers。

## H

**h2c**
HTTP/2 over cleartext（无 TLS）。服务端默认监听方式。生产建议在 Nginx 上层加 TLS。

**Heartbeat**
unary RPC，客户端每 30s 调用一次。携带状态信息，服务端返回最新 pacing。详见 [reference/03-pacing-and-rate-control.md](../reference/03-pacing-and-rate-control.md#下发机制)。

## L

**legacy/**
v3.1+ blockstore 中存放老格式 block 的子目录。启动时迁移到此。

## M

**MaxRecvMsgSize / MaxSendMsgSize**
gRPC 服务端 / 客户端能收 / 发的单条消息上限。v3.x 设为 16 MB（commit `46e9dee`）。详见 [reference/02-architecture.md · 关键运行时常量](../reference/02-architecture.md#关键运行时常量)。

**MinIO**
S3 兼容的对象存储。Sync-Data 用作目标文件存储（必需）和源文件存储（可选）。

**mode**
客户端运行模式。v3.2 起支持 `default` 和 `sqMode`。可通过服务端心跳响应远程切换。详见 [reference/04-key-concepts.md · Mode](../reference/04-key-concepts.md#2-mode运行模式)。

**modes 注册表**
客户端 `sync-client/internal/modes/Registry`，管理已注册的 mode。

## N

**NetworkMonitor**
客户端 goroutine，每 15s 用 gRPC Heartbeat 探测服务端可达性。3 次连续失败时主动 invalidate 连接（commit `8df830d`），避免 sender 死锁。

**Nginx 反代**
生产部署的可选项，做 TLS 终结 + 路径分流（commit `738c828` 加的 `/grpc` 前缀支持）。详见 [reference/10-deployment-modes.md · Nginx 反代](../reference/10-deployment-modes.md#nginx-反代配置生产参考)。

## O

**offline / 在线状态**
服务端 ClientStore 维护的状态。90s 没收到客户端心跳就标记 offline。

**Otter**
阿里开源 binlog 同步工具。详见 [reference/06-competitive-landscape.md](../reference/06-competitive-landscape.md)。

## P

**pacing / 限速策略**
Sync-Data 的核心控制参数集合。三层覆盖：硬编码 → 本地 YAML → 服务端下发。详见 [reference/03-pacing-and-rate-control.md](../reference/03-pacing-and-rate-control.md)。

**pacing_version**
服务端下发策略的单调递增整数。客户端通过比较版本号决定是否 Apply 新策略。

**PolicyStore**
客户端 `sync-client/internal/policy/Store`，合并本地 YAML 配置和服务端下发的 pacing。

**Priority / 优先级**
sender 取块时的排序依据。P1 = file_block，P2 = simple_block 增量，P3 = simple_block 历史。

**Probe**
客户端 NetworkMonitor 用的 gRPC unary RPC，替代 TCP dial 做网络探测。v3 commit `e5cbb76` 引入。

**probetracker**
客户端 `sync-client/internal/network/probetracker`，跟踪 probe 连续失败计数（commit `27eb91c` 抽出）。

**processed_blocks.json**
服务端 validator 的去重表，24h TTL。

**pull-based pacing**
v3.3 设计方向：客户端主动 pull 策略，而非服务端推。详见 [reference/08-roadmap.md](../reference/08-roadmap.md#v33---pull-based-pacing--隔离基础设施)。

## R

**related[]**
file_block 模型下的关联表配置，递归展开。

**ResourceLimiter**
客户端 goroutine，监控内存使用，通过 `debug.SetMemoryLimit` 限制。

**root_table / 根表**
file_block_tables 配置中的主表，每条根表数据触发一个 file_block。

**rootIndex**
客户端 blockstore 的内存索引，按 "`table:root_id`" 映射到 block_id，防止重复出块。

## S

**sb-... / simple_block**
简单块。优先级 2（增量）/ 3（历史回填）。装多行（默认 30 行）。无关联表无文件。

**Scanner**
客户端 goroutine，主扫表循环。按 `incremental_interval_sec` 周期扫源库生成 block。

**schema ensured**
服务端 BlockWriter 写目标库前自动建表。日志：`schema ensured module=writer table=...`。

**schemacache**
客户端 `sync-client/internal/schemacache`，缓存源库 SHOW CREATE TABLE 结果。

**Sender**
客户端 goroutine，从 blockstore 出队，按优先级走 gRPC 上传。

**simple_tables**
客户端 `config.yaml.tables.simple_tables` 配置，定义哪些表用 simple_block 模型。

**simple_tables_exclude**
simple_tables 配置中的黑名单数组。一般用于排除大字段表 / 业务无关表。

**sqMode**
v3.2 引入的运行模式，专供 SmartQuality 客户端用 MinIO multipart 中继。详见 [reference/04-key-concepts.md · Mode](../reference/04-key-concepts.md#2-mode运行模式)。

**sync_state.json**
服务端的全局状态文件。`data/sync_state.json`。每客户端配置 + 统计 + pacing 都在这里。

**SyncBlocks**
客户端流式 RPC，分片上传 block 数据 + 文件。

**SyncService**
gRPC service 全名 `syncpb.SyncService`。proto 定义在两端的 `proto/sync.proto`。

## T

**Token / Token Store**
服务端鉴权。两种模式：(1) `secret_key` 做 HMAC 签名（推荐），(2) `tokens[]` 预置 client_id → token 映射。

**Token Bucket / 令牌桶**
sender 用的限速算法。`network_limit_kbps` 决定令牌生成速率。

**target_database**
服务端为每客户端独立分配的目标 MySQL 库名。模板 `smart_quality_v2_<client_id>`。

**target_minio**
服务端的目标 MinIO。每客户端独立 bucket `smart-quality-<client_id>`。

## U

**unary RPC**
gRPC 一来一回的简单 RPC（vs streaming）。Heartbeat / Probe / RegisterClient / ReportStatus 都是 unary。

**updateTime**
源库行的"最后修改时间"字段。scanner 用作增量水位。

## V

**Validator**
服务端块校验器。维护 `processed_blocks.json` 做幂等去重。详见 [reference/02-architecture.md](../reference/02-architecture.md#服务端模块8-个-internal-包)。

**v3.0 / v3.1 / v3.2 / v3.3**
版本演进。详见 [reference/08-roadmap.md](../reference/08-roadmap.md)。

## W

**WAL（Write-Ahead Log）**
*设想中* 的客户端 blockstore 升级方向，更强的崩溃恢复语义。**未承诺**。

## 缩写表

| 缩写 | 全称 |
|---|---|
| CDC | Change Data Capture |
| DDL | Data Definition Language（CREATE/ALTER/DROP） |
| ETA | Estimated Time of Arrival |
| h2c | HTTP/2 over cleartext |
| HMAC | Hash-based Message Authentication Code |
| RPC | Remote Procedure Call |
| TLS | Transport Layer Security |
| TTL | Time To Live |
| WAL | Write-Ahead Log |
