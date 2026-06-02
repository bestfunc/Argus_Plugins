# 03 · 限速与流控

## 定义

**Pacing**（中文："节拍 / 限速策略"）是 Sync-Data 控制每个客户端**上行带宽、扫表节奏、并发度、重试退避**的统一参数集合。它的设计核心：**服务端可在运行时下发，客户端无需重启就能热生效**。

## 三层配置体系

按"运行时优先级"由低到高排列：

| 层 | 位置 | 何时生效 | 谁能改 |
|---|---|---|---|
| ① **默认值** | 代码硬编码 | 启动时 | 改代码 + 重新发布 |
| ② **本地 YAML** | `sync-client/config/config.yaml` 的 `scan` / `resources` / `network` / `heartbeat` / `retry` 节 | 客户端启动加载 | 修文件 + 重启客户端 |
| ③ **服务端下发 pacing** | `sync-server/data/sync_state.json` 的 `clients[<id>].pacing` 子结构 | 客户端**下次 Heartbeat 拉到新 `pacing_version`** 时热加载 | Admin Web 修改即可 |

**有效值优先级**：③ > ② > ①。即一旦服务端下发了某个字段，本地 YAML 那个字段就被覆盖。

## pacing 完整字段（v3.x）

来自 `sync-server/data/sync_state.json` 实际生产数据。每个客户端有自己一套。

### 网络与资源

| 字段 | 典型值 | 含义 |
|---|---|---|
| `network_limit_kbps` | 50 / 200 / 2000 | 上行带宽硬上限（kbps）· 越高客户端追积压越快 |
| `memory_limit_mb` | 200 | 客户端进程内存软限（触发 GC + 限速） |
| `source_max_open_conns` | 2 | 源库连接池最大连接 |
| `source_max_idle_conns` | 1 | 源库连接池空闲连接 |

### 扫表 / 出块

| 字段 | 典型值 | 含义 |
|---|---|---|
| `scan_incremental_interval_sec` | 120 | 增量扫描周期（秒） |
| `scan_batch_size` | 10 | 一批读多少行 |
| `scan_row_sleep_ms` | 50 | 行间延迟（避免压垮源库） |
| `scan_child_scan_interval_sec` | 1800 | 子表（关联表）兜底扫描周期 |
| `scan_child_scan_batch_size` | 5 | 子表批次大小 |
| `scan_historical_batch_size` | 5 | 历史数据批次大小 |

### 简单表

| 字段 | 典型值 | 含义 |
|---|---|---|
| `simple_tables_scan_interval_sec` | 600 | 简单表扫描周期（10 分钟） |
| `simple_tables_batch_size` | 30 | 一个 simple_block 装多少行 |
| `simple_tables_tables_per_round` | 5 | 一轮调度处理几张表 |
| `simple_tables_exclude` | `[migrations, detect, ...]` | 排除表黑名单（大字段表用这个剔除） |

### 心跳 / 重试

| 字段 | 典型值 | 含义 |
|---|---|---|
| `heartbeat_interval_sec` | 30 | 心跳周期 |
| `heartbeat_timeout_sec` | 5 | 单次心跳 RPC 超时 |
| `retry_interval_sec` | 30 | 初始退避间隔 |
| `retry_max_interval_sec` | 300 | 退避上限（5 分钟） |
| `retry_backoff_multiplier` | 1.5 | 指数退避倍数 |
| `retry_max_consecutive_failures` | 10 | 连续失败 N 次后 sender 暂停 |

## 下发机制

```
┌──────── Admin Web (8090) ────────┐
│  实施 / 运维 修改某个客户端 pacing │
└──────────────┬───────────────────┘
               │ 写入
               ▼
        sync_state.json
        (clients.<id>.pacing,
         pacing_version: N → N+1)
               │
               ▼
       服务端 SyncState 内存版本递增
               │
               │ 等待客户端下一次 Heartbeat
               ▼
       Heartbeat 响应携带 PolicyV_{N+1}
               │
               ▼
       客户端 PolicyStore.Apply()
               │
               ▼
       scanner / sender / heartbeat 通过 Provider 读到新值
       (无需重启, 1 个心跳周期内全模块生效)
```

**协议字段**：`pacing_version` 单调递增。客户端缓存当前版本，心跳响应里如果服务端版本更高就触发 `Apply`。

## 限速实施细节

### Sender 上行限速

- 实现方式：**Token bucket**（漏桶 / 令牌桶）
- 触发位置：sender 在 SyncBlocks RPC 分片发送前 acquire 令牌
- 单位：**kilobits / second**（注意是位不是字节，1 kbps = 125 B/s = 0.125 KB/s）
- 实测：`network_limit_kbps=50` ≈ **6.25 KB/s**，单个 5 MB 块要走 ~13 分钟

### Scanner 节奏限速

- `scan_row_sleep_ms` 在每行读取后 `time.Sleep`，避免一次性压垮源库
- `scan_batch_size` 控制单次查询的 `LIMIT`
- `simple_tables_tables_per_round` 控制一轮处理几张表，配合 `scan_interval_sec` 决定整体节拍

### 退避

退避算法（sender 失败重试）：
```
next_interval = min(
  retry_interval_sec × backoff_multiplier ^ attempt,
  retry_max_interval_sec
)
```

例如 `retry_interval_sec=30 backoff=1.5 max=300`：
- attempt 1 → 30s
- attempt 2 → 45s
- attempt 3 → 67.5s
- attempt 4 → 101s
- attempt 5 → 152s
- attempt 6 → 228s
- attempt 7+ → 300s（封顶）

连续失败 `retry_max_consecutive_failures=10` 次后 sender 进入 **paused** 状态，等待人工干预（重启客户端或等服务端恢复）。

## ★ 重要：v3.3 在做 pull-based pacing 改造

当前 v3.2 模式是**服务端推**（心跳响应携带），有几个**已知问题**：
- 修改 pacing 后要等下一次心跳才能生效（最长一个心跳周期）
- 客户端无法主动"按需问"策略
- 故障时无法快速通知客户端"暂停"

v3.3 的计划方向（来自 `docs/plans/2026-06-02-v3.3-pull-based-pacing.md`，措辞 = 设计中）：
- 客户端主动 pull 策略（带版本号，无变化时短返回）
- 服务端可发广播事件触发客户端立刻拉
- 引入"错误分类器"，自动调整 pacing（如 ResourceExhausted → 自动 batch_size / 2）

具体设计文档请直接看 `docs/plans/`。

## 调速实战参考

| 现象 | 建议调整 |
|---|---|
| 客户端追不上、积压在累积 | `network_limit_kbps` ↑（视上行带宽，可到 1000-2000） |
| 源库 CPU 过高 / 业务慢 | `scan_batch_size` ↓ + `scan_row_sleep_ms` ↑ |
| 单 block 体积大（接近 16 MB）频繁失败 | `simple_tables_batch_size` ↓（从 30 改到 10） |
| 大字段表（音频/图片字段）反复重试 | 加入 `simple_tables_exclude[]` 暂停同步 |
| 频繁 sender paused | 排查根因后再放开；不要简单调高 `retry_max_consecutive_failures` |

详细场景：见 [scenarios/03-tuning-throughput.md](../scenarios/03-tuning-throughput.md)。
