# 场景 02 · 客户端落后 / 卡死的排查 SOP

## 角色与目标

**角色**：值班运维 / 二线开发

**触发条件**：业务反馈"某客户端数据没更新 / 落后好几天 / 完全不动"。

**目标**：5 步定位问题边界，确定是 client / server / 网络 / 目标基础设施 哪一层的问题，给出可执行的下一步。

## 排查 SOP

```
[1] admin 看板状态 ──► [2] 服务端日志 ──► [3] 目标库实际数据
                                                    │
                                                    ▼
[5] 客户端日志       ◄── [4] MinIO 上传记录
```

### Step 1 · admin 看板状态

打开 admin Web → 找到该 client_id。看：

| 指标 | 健康 | 异常 |
|---|---|---|
| online 状态 | 绿点 | 红点 / 灰点 |
| 最后心跳时间 | < 1 分钟 | > 5 分钟 |
| catchup 倍率 | > 1.0 (在追上) | < 1.0 (在落后) |
| pacing version | 跟最新一致 | 落后于服务端 |
| sync enabled | true | false → 没人开 |

**关键判断**：

- **online + catchup > 1 + 数据没增长** → 客户端在追的不是最新数据（典型：在补历史回填），见 step 3
- **offline** → 客户端进程死了 / 网络断 / 心跳协议挂 → 跳 step 5
- **online + catchup < 1** → 客户端追速跟不上源端产生速度 → 见 [scenarios/03](./03-tuning-throughput.md)

### Step 2 · 服务端日志

```bash
ssh <sync-server> 'grep "client_id=<client_id>" /root/sync-server/data/server.log | tail -50'
```

关注三类信号：

**A. 收 block 频率**
```bash
grep "$(date +%Y/%m/%d)" /root/sync-server/data/server.log \
  | grep "receiving block.*client_id=<client_id>" | wc -l
```

- 几百以上：客户端在活跃推
- 个位数：客户端推得很少（可能源端没新数据 / 客户端 scanner 卡了 / 网络断断续续）

**B. ERROR / WARN**
```bash
grep "$(date +%Y/%m/%d)" /root/sync-server/data/server.log \
  | grep -E "ERROR|WARN" | grep "client_id=<client_id>"
```

可能的 ERROR 类型：
- `dial tcp ...: connection refused` → 服务端找不到目标 MySQL/MinIO，**基础设施问题**
- `received message larger than max` → block 超过 16 MB（v3.2 之前是 4 MB）
- `block validation failed ... ALREADY_PROCESSED` → 健康信号，无视

**C. 反复 receive 同一 block**
```bash
grep "client_id=<client_id>" /root/sync-server/data/server.log | tail -30 \
  | grep -oE "block_id=[a-z0-9-]+" | sort | uniq -c | sort -rn | head
```

如果某个 block_id 出现 > 5 次，可能：
- 客户端反复重试同一个块（参考下方"案例 1"复盘）
- 网络中间件在断连客户端

### Step 3 · 目标库实际数据

```bash
mysql -h <target> -u root -p smart_quality_v2_<client_id> -e \
  'SELECT MAX(createTime), COUNT(*) FROM detect;'
```

```bash
mysql -h <target> -u root -p smart_quality_v2_<client_id> -e \
  "SELECT DATE(createTime), COUNT(*) FROM file WHERE updateTime >= CURDATE() GROUP BY 1;"
```

**判断逻辑**：

| MAX(createTime) | 今天 updateTime 行数 | 含义 |
|---|---|---|
| 接近现在 (< 1h) | 大量 | 健康 |
| 几小时前 | 有写入 | 在追近期数据 |
| **几天前** | 有写入 | 在回填历史，不是卡死 |
| 几天前 | **0** | **客户端真没在推** → 跳 step 5 |
| 几月前 | 0 | scanner 应该卡了 |

> Tip：**MAX(createTime) 是 sender 推到的最新行，不是 sender 当前在传的位置**。如果 sender 在按时间顺序补历史，MAX 反而是最远没补到的边界。要看"当前在传哪天"，看 file 表里 updateTime ≈ 现在的行：
> ```sql
> SELECT DATE(createTime), COUNT(*) FROM file
> WHERE updateTime >= NOW() - INTERVAL 1 HOUR
> GROUP BY 1 ORDER BY 1;
> ```

### Step 4 · MinIO 上传记录

如果 file_block 有问题，看 MinIO 侧：

```python
import boto3
s3 = boto3.client("s3", endpoint_url="http://<minio>:29000", ...)
# 列最近上传
for page in s3.get_paginator("list_objects_v2").paginate(
    Bucket=f"smart-quality-<client_id>"):
    for o in page.get("Contents", []):
        # o["LastModified"] = MinIO 收到时间
        # o["Key"] 中含 source_date (路径里的 YYYY-MM-DD 段)
        ...
```

**判断**：
- `LastModified` 都是今天 → 客户端在传
- `LastModified` 几天前 → 客户端 stop 了，MinIO 没收新对象
- `Key` 路径里 `source_date` 一直停在某天 → 客户端 sender 卡在那天的 block 上

### Step 5 · 客户端日志（最关键）

SSH / RDP 到客户机，看 `<sync-client-dir>/logs/client.log` 或 stdout：

#### A. 看 sender / heartbeat

```bash
tail -200 client.log | grep -E "block sent|block failed|heartbeat|sender paused"
```

关键信号：
- `block sent and deleted` 频繁 → 在传
- `send failed, will retry ... attempt=N` 不断递增 → sender 在卡
- `sender paused` → 连续失败 ≥ 10 次进入暂停状态
- `heartbeat failed` → 网络断 / 服务端不可达

#### B. 看 scanner

```bash
tail -200 client.log | grep -E "scanner|scan completed|scanning"
```

- `scan completed table=<x> rows=N` 周期出现 → scanner 在工作
- 长时间无输出 → scanner 可能卡在某个表（或 lookup 异常表结构）

#### C. 看 NetworkMonitor

```bash
tail -200 client.log | grep -E "probe|network|offline"
```

- `probe failed` 频繁 → 网络层有问题，可能触发 invalidate 连接
- `network offline` → 客户端认为自己离线，sender 暂停

#### D. 死信目录

```bash
ls data/dead_letters/
```

有内容 → 有 block 已经超过重试上限被搁置，需要人工处理。

## 真实复盘案例

### 案例 1 · 大字段表把 client 卡死 14 天

**症状**：服务端反复收到客户端的同一个 `sb-20260516-*` block 的 receive log，但从无 `block written`。客户端 sender attempt 编号涨到 2900+。

**诊断**：
1. step 2 找到反复 receive 同一 block_id
2. step 3 目标库 14 天没新数据
3. step 5 客户端 log 看到 `recv ack: ResourceExhausted: received message larger than max (5296411 vs. 4194304)`

**根因**：服务端 gRPC `MaxRecvMsgSize` 是默认 4 MB；该 block 装的是大字段表（音频字段 payload）30 行 ≈ 5 MB，**永远过不去**。

**修复**：服务端代码加 `grpc.MaxRecvMsgSize(16*1024*1024)`（commit `46e9dee`），重启后下一次 retry 就成功。同时把大字段表加入 `simple_tables_exclude` 防止再生。

### 案例 2 · 基础设施宕机连带影响

**症状**：某天凌晨多个客户端同时 ERROR write。

**诊断**：
1. step 2 看到 `dial tcp <minio>: connect: no route to host` 集中在某 9 小时窗口
2. 该窗口里所有客户端的 ERROR 都指向 192.168.x.x:29000（MinIO + MySQL 在一台机器）
3. 该机器恢复后，所有客户端的 ERROR 在几分钟内归零

**根因**：目标基础设施宿主机宕机，多客户端都受影响 —— 不是 sync 本身的问题。

**修复**：恢复基础设施后客户端自动重试成功。

### 案例 3 · 客户端 scanner 停了（同步链路 `enabled` 但无新数据）

**症状**：客户端 online + heartbeat 正常 + 但目标库已 16+ 天无新数据；MinIO bucket 5-15 之后无任何上传。

**诊断**：
1. step 3 MAX(createTime) 停在 5-15
2. step 4 MinIO 5-15 之后 prefix 完全空（不是延迟问题）
3. step 5 客户端 log 显示 scanner 周期性输出 `scan completed table=detect rows=0`

**结论**：scanner 在跑，但源端就是没产数据 / 或者 scanner 的某个 checkpoint 异常没推进 —— 需要直接进客户机看 `data/checkpoint.json` 状态。

**修复**：超出本 SOP 范围，需进客户机用 mysql 客户端比对源端 `MAX(updateTime)` vs `checkpoint.json` 中记的水位。

## 输出格式（值班记录建议）

完成排查后填一个简单的事件记录：

```markdown
## <date> · <client_id> 故障排查

**症状**：<业务方描述>
**Step 1 admin 看板**：<online/offline,catchup 倍率>
**Step 2 服务端日志**：<receive 频率,关键 ERROR>
**Step 3 目标库**：<MAX createTime,今天写入行数>
**Step 4 MinIO**：<最新上传时间,source_date>
**Step 5 客户端日志**：<scanner/sender/probe 状态>

**根因**：<一句话>
**已采取的措施**：
- <动作 1>
- <动作 2>

**遗留 TODO**：<如需后续修代码 / 修配置>
```

## 速查

| 现象 | 直接跳哪一步 |
|---|---|
| 业务反馈"完全不动" | Step 1 看 online |
| "数据落后但能动" | Step 3 看目标库 |
| "某个 block 一直没过" | Step 2 grep block_id |
| "客户端日志看不出问题" | 翻 [scenarios/03](./03-tuning-throughput.md) 调速 |
