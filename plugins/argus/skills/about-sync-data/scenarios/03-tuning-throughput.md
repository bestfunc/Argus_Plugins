# 场景 03 · 客户端追不上时调速

## 角色与目标

**角色**：实施运维 / 二线开发

**触发条件**：客户端在线、心跳正常、有在推 block，但**追积压速度太慢**，业务方关心"什么时候能赶上"。

**目标**：通过 pacing 调整旋钮，提升单客户端追积压速度，而不导致：
- 源 MySQL 压力过大
- block 体积超 16 MB 上限
- 服务端 / 目标库被打爆

## 调速前先量化现状

打开 admin Web 找到该客户端，记录：

| 指标 | 当前值 | 备注 |
|---|---|---|
| `network_limit_kbps` | ? | 关键限速参数 |
| `scan_batch_size` | ? | 每次扫多少行 |
| `simple_tables_batch_size` | ? | 一个 simple_block 装多少行 |
| 目标库 MAX(createTime) | ? | 当前同步到哪天 |
| 距今差距 | ? 天 ? 小时 | = NOW - MAX |
| 每天源端产出 | ? MB | 看历史 total_bytes_sent / 历史天数 |
| 每天客户端实际追上 | ? MB | 看今天 updateTime 行数对应的字节量 |

只有量化后才能判断"该不该调"和"调多大"。

## 调速决策树

```
积压速度 = 源端日产出 - 客户端日追上

正值 (越拉越远) ──► 必须调速,优先 kbps
0    (稳定但没追) ──► 调速能追上,看 kbps
负值 (在追上)    ──► 估算 ETA,等就行 OR 加速追完
```

## 主要旋钮（按效果排序）

### ① `network_limit_kbps` — 网络上行带宽

**最直接的旋钮**。直接决定客户端每秒能发多少。

| 当前值 | 上行实际值 | 单 5 MB block 耗时 |
|---|---|---|
| 50 (默认) | ~6.25 KB/s | ~13 分钟 |
| 200 | 25 KB/s | ~3 分钟 |
| 500 | 62.5 KB/s | ~80 秒 |
| 1000 | 125 KB/s | ~40 秒 |
| 2000 | 250 KB/s | ~20 秒 |
| 5000 | 625 KB/s | ~8 秒 |

**建议**：
- 网络好的客户端（专线 / 千兆内网）：调到 1000-2000
- 家庭宽带客户：50-200，避免占满影响业务
- **同机多 client_id**：所有 client 加起来不要超过出口带宽 60%

**风险**：调过高客户端 sender 占满上行，可能影响本地业务的网络使用。

### ② `simple_tables_batch_size` — 单 block 行数

**第二重要的旋钮**。每个 simple_block 装多少行。

| 当前值 | 单 block 体积（小表） | 单 block 体积（大字段表） |
|---|---|---|
| 30 (默认) | KB 级 | 可达 5 MB（如音频表） |
| 10 | 几百字节 | ~1.5 MB |
| 5 | 几百字节 | ~800 KB |

**建议**：
- 默认 30 适用绝大多数表
- 含大字段（音频 / 图像 / JSON）的表：**优先剔除（exclude）**而不是 batch_size 调小
- 如果某些表既不能剔除又超 16 MB → 调到 10 或 5

### ③ `scan_batch_size` 和 `scan_row_sleep_ms` — 源库扫表节奏

scanner 端的节奏旋钮，**影响源 MySQL 压力**：

| 字段 | 默认 | 调高效果 | 调低效果 |
|---|---|---|---|
| `scan_batch_size` | 10 | 单次 SELECT 拉更多行 | 单次拉得少，扫得密 |
| `scan_row_sleep_ms` | 50 | 行间停顿更久，对源库更友好 | 扫得更急，源库压力大 |

**建议**：
- 默认即可
- 源库 CPU 紧张：`row_sleep_ms` 调到 100-200
- 客户端在补历史想加速：`scan_batch_size` 调到 50-100 + `row_sleep_ms` 调到 20

### ④ `scan_incremental_interval_sec` — 扫描周期

scanner 多久跑一次。

| 默认 | 调短 | 调长 |
|---|---|---|
| 120s | 60s → 数据更新更及时 | 300s → 更省源库资源 |

实际"新数据生效延迟"主要由扫描周期 + 发送队列长度共同决定。

### ⑤ `simple_tables_exclude` — 排除大表 / 不要的表

> 这不是限速，而是**直接去掉负载**。最高效的"治标"手段。

候选表：
- `migrations`（DB 迁移记录，业务无关）
- `audio_ai_req_log`（大字段，已知会让 block 接近上限）
- `*_log` / `*_history` / `*_archive`（一般不需要实时同步）
- 任何业务方确认"目标库不需要"的表

```yaml
simple_tables:
  exclude:
    - migrations
    - audio_ai_req_log
    - score_analysis_clean_log
    - feature_export
```

修改后下次心跳生效（通过 admin Web 改 pacing）。

## 三个典型调速场景

### 场景 A · 客户端落后多天，要追上

**前提**：网络带宽富余，源 MySQL 不忙。

**配方**：
```yaml
pacing:
  network_limit_kbps: 2000          # 大幅调高（默认 50）
  scan_batch_size: 50               # 加快扫描
  scan_row_sleep_ms: 20             # 减少行间停顿
  simple_tables_batch_size: 30      # 保持默认
  scan_incremental_interval_sec: 60 # 缩短到 1 分钟
```

**预估**：每天能多追 100 MB-1 GB（取决于上行带宽实际值）。

### 场景 B · 客户端追积压中频繁卡 block

**前提**：网络 ok，但某些 block 反复重试。

**诊断**：先按 [scenarios/02](./02-incident-debug-workflow.md) 确认根因。如果是大字段表：

**配方**：
```yaml
pacing:
  simple_tables_exclude:
    - <大字段表>                     # 关键
  simple_tables_batch_size: 10      # 兜底
```

修改后客户端心跳生效。

### 场景 C · 源 MySQL 业务白天忙、晚上闲

**前提**：客户端按业务峰谷调速。

**配方**（不能动态切，但可以折中）：
```yaml
pacing:
  network_limit_kbps: 200            # 平均值
  scan_batch_size: 20                # 中速
  scan_row_sleep_ms: 80              # 稍慢
```

也可以人工每天晚上 11 点调高、早上 7 点调低，但成本高，一般不做。

## 实际调速步骤（在 admin Web）

1. 找到该 client_id 卡片，点 "编辑 pacing"
2. 改字段后保存 → 服务端 `sync_state.json` 更新 + `pacing_version + 1`
3. 等下一次心跳（≤ 30s），客户端会拉到新策略
4. 客户端日志出现：
   ```
   INFO pacing policy applied module=heartbeat from_version=N to_version=N+1
   ```
5. 观察 1-2 小时 catchup 倍率变化

## 失败模式

**调速失败**的几种现象 + 处理：

| 现象 | 原因 | 处理 |
|---|---|---|
| 调高 kbps 后 admin 看板没变化 | 没等到下次心跳 | 等 30s |
| 调高 kbps 后客户端没生效 | pacing_version 没更新 | 看客户端日志是否报 `pacing policy applied` |
| 调高 kbps 后 sender ERROR 增多 | block 实际很大，4 MB 限制其实是源 MinIO 拉的 timeout | 看 [reference/03](../reference/03-pacing-and-rate-control.md) 排查 |
| 调高 batch_size 后源库 CPU 飙升 | 太激进 | 回滚 + 反向调 `row_sleep_ms` |
| 改了 exclude 后客户端报错某表找不到 | 该表已被排除但 file_block 还引用 | 检查 file_block_tables.related 是否依赖该表 |

## 速查

| 想干嘛 | 改哪个字段 |
|---|---|
| 让追上更快 | `network_limit_kbps` ↑ |
| 防止 block 超 16 MB | `simple_tables_batch_size` ↓ |
| 不让某表同步 | `simple_tables_exclude` 加 |
| 减少源库压力 | `scan_row_sleep_ms` ↑ + `scan_batch_size` ↓ |
| 加快新数据生效 | `scan_incremental_interval_sec` ↓ |
