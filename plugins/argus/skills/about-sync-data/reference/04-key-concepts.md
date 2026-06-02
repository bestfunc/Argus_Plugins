# 04 · 关键概念

## 定义

本文档把 Sync-Data 里**所有需要先理解才能读懂代码 / 日志 / 配置**的核心概念集中讲清楚。术语首次出现处都给定义；如需查更短的解释，直接查 [glossary/terms.md](../glossary/terms.md)。

## 概念地图

```
                       ┌─── client_id ───┐
                       │  (持久标识)      │
                       └────────┬────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
         ┌────▼─────┐      ┌────▼─────┐      ┌────▼─────┐
         │ 目标库    │      │ MinIO    │      │ pacing   │
         │ (per id) │      │ bucket   │      │ (per id) │
         └──────────┘      └──────────┘      └──────────┘
                                ▲
                                │ 装载
                                │
                       ┌────────┴────────┐
                       │     block       │
                       │ (同步原子单位)   │
                       └────────┬────────┘
                                │
                  ┌─────────────┴──────────────┐
                  │                            │
         ┌────────▼────────┐         ┌─────────▼────────┐
         │  file_block     │         │  simple_block    │
         │  fb-YYYYMMDD-x  │         │  sb-YYYYMMDD-x   │
         │  P=1 (高优)     │         │  P=2 增量 / 3 历史│
         └─────────────────┘         └──────────────────┘
                  │
                  │ 含
                  ▼
         ┌───────────────────────────────┐
         │ ① 根行 (1 行)                │
         │ ② 关联表行 (递归)             │
         │ ③ 文件 (MinIO 路径 + 本地副本)│
         └───────────────────────────────┘
```

## 1. Block（块）

> **同步的原子单位**。一个 block 装的内容要么全部成功落库，要么完全不落 —— 服务端通过 `block_id` 做幂等去重。

两种类型：

### file_block（文件块）

- ID 格式：`fb-YYYYMMDD-xxxxxxxx`（v3.1+）或老格式 `fb-xxxxxxxx`
- **优先级 1**（最高）
- 来源：定义在 `tables.file_block_tables[]` 的根表（典型如 `detect`）
- 装载内容：
  - 1 条根行
  - 关联表 (`related[]`) 的相关行，**递归**展开
  - 关联行中文件字段（`file_fields[]`）引用的 **MinIO 文件二进制**（异步从源 MinIO 下载）
- 用途：保证"主表 + 子表 + 文件"作为一个事务级单位同步

### simple_block（简单块）

- ID 格式：`sb-YYYYMMDD-xxxxxxxx`
- **优先级 2**（增量）或 **3**（历史回填）
- 来源：定义在 `tables.simple_tables`（不能在 file_block_tables 里的所有表）
- 装载内容：**最多 `simple_tables_batch_size`（默认 30）行**，**无关联表、无文件**
- 用途：纯行批量复制，适用元数据表 / 日志表 / 配置表

### Block ID 演进

| 版本 | 格式 | 说明 |
|---|---|---|
| V3.0 | `fb-xxxxxxxx` / `sb-xxxxxxxx` | 老格式，扁平 |
| V3.1+ | `fb-YYYYMMDD-xxxxxxxx` / `sb-YYYYMMDD-xxxxxxxx` | 带日期前缀 · 按日子目录持久化 |

V3.1+ 启动时自动把老格式块迁到 `data/blocks/legacy/`，新生成的块用新格式。

### Block 内容计算

`Block.ComputeChecksum()` 算 **SHA256**，**仅覆盖数据结构（meta + rows + file 引用）**，不含文件二进制和辅助 DDL。这样：
- 服务端可以独立校验数据完整性
- 文件下载失败不影响 block 本身的 checksum 稳定

## 2. Mode（运行模式）

> 客户端如何"产生 block"和"上传文件"的整体策略。**服务端可远程切换**。

v3.x 已有：

| Mode | 用途 | 关键差异 |
|---|---|---|
| `default` | 通用同步 | scanner 扫表生成 block · sender 走 SyncBlocks RPC · 文件直接打包进 block |
| `sqMode` (v3.2+) | SmartQuality 客户端专用 | 监视 `sq_inbox` 目录 · 文件走 ChunkUpload (MinIO multipart 中继) · 8 MiB 分片 |

注册位置：`sync-client/internal/modes/Registry.Register(name, impl)`。

服务端通过 Heartbeat 响应可下发 `mode` 字段，客户端检测到变化时切换。如果客户端没有该 mode 注册，会**保持当前 mode 不变**（"no mode to start" 兜底）。

## 3. Checkpoint（同步断点）

> 客户端记录"上次同步到哪了"的位置标记。

存储位置：客户端 `data/checkpoint.json`，按表分项。

数据结构（简化）：
```json
{
  "tables": {
    "detect": {
      "last_updateTime": "2026-05-15 18:58:27.832",
      "last_id": "fffadb8d..."
    }
  }
}
```

下次扫描时 scanner 用 `WHERE updateTime > <last_updateTime>` 接着扫。这种"水位标记"策略要求源库的 `updateTime` 字段是 **monotonic**（修改时一定更新）。

服务端侧的同名概念是 **CheckpointSampler**：每 60 秒采样每个客户端目标库的 `MAX(updateTime)`，给 admin 进度面板展示。

## 4. Pacing（限速策略）

详见 [03-pacing-and-rate-control.md](./03-pacing-and-rate-control.md)。简而言之：

- **三层配置**：硬编码默认 / 本地 YAML / 服务端 sync_state.json（后者覆盖前者）
- **热加载**：通过心跳响应的 `pacing_version` 单调递增触发
- **覆盖面**：网络限速 + 扫表节奏 + 心跳 + 重试 + 简单表黑名单

## 5. 幂等去重（idempotency）

> 同一个 `block_id` 被服务端处理过 → 后续重复推送直接 ACK 不重复落库。

实现：服务端 `validator` 维护 `data/processed_blocks.json` 表 + 24h TTL 滚动清理。

日志特征：
```
WARN block validation failed module=receiver block_id=sb-... err="block sb-... already processed" code=ALREADY_PROCESSED
```

`ALREADY_PROCESSED` 不是错误，**是健康的去重信号**。

## 6. 关联表（related）

> file_block 模型下，与根表一起构成"业务一致性闭包"的下游表。

配置位置：`sync-client/config/config.yaml` 的 `tables.file_block_tables[].related[]`。

典型例子（来自配置）：
```yaml
file_block_tables:
  - root_table: detect
    related:
      - table: detect_channel
        fk_field: detectID
        fk_ref: id
        file_fields:
          - field: audioPath
            bucket_field: audioBucket
      - table: file
        fk_field: detectID
        fk_ref: id
```

scanner 拉到 1 条 `detect` 行后，会按 `fk_field IN (...)` 拉 `detect_channel` 和 `file` 表里相关的所有行，再扫文件字段（`file_fields`）拉文件二进制。

## 7. 死信（dead letter）

> 重试次数超过 `retry_max_consecutive_failures`（默认 10）后被搁置的 block。

存储：客户端 `data/dead_letters/<blockID>/`。

清理策略：
- 数量上限：`max_dead_letters=100`
- 年龄上限：`dead_letter_max_age_hours=168`（7 天）

人工干预（典型）：检查问题、调整配置后把目录手动移回 `data/blocks/<date>/` 重试。

## 8. Pacing version

> 服务端下发策略的单调递增整数。

机制：
- 服务端 admin 修改 pacing → `pacing_version + 1`
- 客户端心跳响应里读到比本地大的 `pacing_version` → 调 `PolicyStore.Apply(new_policy)`
- 各模块（scanner / sender / heartbeat）通过 Provider 拿到新参数

服务端日志：
```
INFO pacing policy pushed module=receiver client_id=<id> from_version=0 to_version=2
```

## 9. 优先级（priority）

> sender 取块时的排序依据。

| Priority | 类型 | 来源 |
|---|---|---|
| 1 | file_block | 扫到一条根表数据立刻打包 |
| 2 | simple_block (增量) | 简单表当天的新行 |
| 3 | simple_block (历史回填) | 简单表的老数据补齐 |

sender 总是先把 P1 清空再处理 P2/P3。注意：**这不是绝对的**，P1 一直产但 P2 也会被处理，避免饥饿（具体调度策略见 `sync-client/internal/sender`）。

## 10. Schema ensured（DDL 自动同步）

> 服务端写目标库前，自动检查表是否存在，不存在则用源库的 CREATE TABLE DDL 创建。

日志特征：
```
INFO schema ensured module=writer table=smart_quality_v2_<id>.audio_ai_req_log
```

来源：客户端的 `schemacache` 缓存源库的 `SHOW CREATE TABLE` 输出，随 block 一起传到服务端，服务端按需在目标库 CREATE。

v3.3 正在做 `knownTables 缓存加 30 分钟 TTL + flush API`（commit `694c4c2`），优化大量小表场景下的重复检查开销。
