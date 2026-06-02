# FAQ 02 · 开发 / 运维内部问题

> 给团队内部用。覆盖加表 / 调参 / 报错对照 / 代码导航。

## 1. 加一张新表的同步

**Q：新需求要把某个表加进同步，怎么改？**

判断该表的类型：

| 表特征 | 该用什么模型 | 配置位置 |
|---|---|---|
| 主表 · 行数中等 · 有关联表 · 关联表有文件字段 | **file_block** | `tables.file_block_tables[]` |
| 业务无关联 · 纯行 · 无文件 | **simple_block** | `tables.simple_tables.include[]` |

加 file_block 示例（`sync-client/config/config.yaml`）：
```yaml
tables:
  file_block_tables:
    - root_table: <new_root_table>
      root_database: <db>
      time_field: updateTime
      id_field: id
      related:
        - table: <related_table>
          database: <db>
          fk_field: <fk_to_root>
          fk_ref: id
          file_fields:                # 如有文件字段
            - field: filePath
              bucket_field: fileBucket
```

加 simple_block 更简单：
```yaml
tables:
  simple_tables:
    include:
      - <new_table>
```

**改完要做什么**：
1. 重启客户端（本地 YAML 改动**不能热加载**）
2. 等 scanner 下次扫描周期触发
3. 服务端 Writer 首次会自动 `schema ensured` 建表

**Q：能不能让某些字段不同步？**
A：当前**不支持列级过滤**。scanner 是 `SELECT *`。如果业务确实需要，可以在 sync 完后用目标库的 view 屏蔽列。

## 2. 修改 pacing 不用重启

**Q：怎么不重启客户端就改限速？**
A：从 admin Web 改 `sync_state.json` 里的 pacing → 服务端 `pacing_version + 1` → 客户端下次心跳（≤ 30s）自动拉到新策略生效。客户端日志：
```
INFO pacing policy applied module=heartbeat from_version=N to_version=N+1
```

详见 [reference/03-pacing-and-rate-control.md · 下发机制](../reference/03-pacing-and-rate-control.md#下发机制)。

**Q：哪些参数可以热加载，哪些必须重启？**
A：
- **可热加载（★ 标记）**：`network_limit_kbps` / `scan_*` / `simple_tables_*` / `heartbeat_*` / `retry_*` 等 pacing 字段
- **必须重启**：源库连接配置 / server.address / TLS 配置 / `tables.file_block_tables[]` 结构

详见 [reference/03-pacing-and-rate-control.md · pacing 完整字段](../reference/03-pacing-and-rate-control.md#pacing-完整字段v3x)。

## 3. 报错对照表

### gRPC 层

| 报错 | 原因 | 解决 |
|---|---|---|
| `Unauthenticated` | secret_key 不匹配 | 两端 `SYNC_SECRET_KEY` 必须一致 |
| `recv ack: ResourceExhausted: received message larger than max` | block 体积 > MaxRecvMsgSize | v3.x 已调到 16 MB；如还超就拆 batch 或排除大字段表 |
| `dial tcp ...: connection refused` | 服务端不可达 / Nginx 没起 | `nc -zv <host> <port>` 验证 |
| `the client connection is closing` | TCP 连接被对端 RST | 看 server 端是否 PROTOCOL_ERROR，或服务端正在重启 |
| `context deadline exceeded` | 单次 RPC 超时 | 网络抖动或 heartbeat_timeout_sec 太短 |

### 业务层（服务端日志）

| 报错 | 原因 | 解决 |
|---|---|---|
| `ERROR write block failed ... dial tcp ...:23306` | 目标 MySQL 不可达 | 检查 target 主机状态 |
| `ERROR write block failed ... dial tcp ...:29000` | 目标 MinIO 不可达 | 同上 |
| `WARN block validation failed ... ALREADY_PROCESSED` | 幂等去重 | **不是错误**，忽略 |
| `client marked offline client_id=...` | 90s 没心跳 | 检查客户端进程 / 网络 |
| `schema ensured module=writer table=...` | 首次为该表建库表 | 正常 |

### 客户端日志

| 报错 | 原因 | 解决 |
|---|---|---|
| `send failed, will retry ... attempt=N` | 上传失败 retry 中 | N 持续上涨要警觉，检查根因 |
| `too many consecutive failures, pausing sender count=10` | sender 已进 paused 状态 | 排查根因后重启客户端 |
| `probe failed err=...` | 网络探测失败 | 偶尔出现正常，连续 3 次会 invalidate 连接 |
| `heartbeat failed module=heartbeat err=...` | 心跳 RPC 失败 | 服务端不可达或被 RST |
| `download_failed=true` (block meta) | 源 MinIO 下载文件失败 | 不阻塞块提交，但目标库行 file path 会指向缺失对象 |

## 4. 代码导航（哪里改哪里）

| 想改什么 | 在哪 |
|---|---|
| Scanner 扫表逻辑 | `sync-client/internal/scanner/` |
| file_block 装配 | `sync-client/internal/scanner/file_block_builder.go` |
| Sender 上传 + 重试 | `sync-client/internal/sender/sender.go` |
| Heartbeat 协议 | `sync-client/internal/heartbeat/` + `sync-server/internal/receiver/` |
| gRPC 服务端接收 | `sync-server/internal/receiver/` |
| 写目标 MySQL | `sync-server/internal/writer/` |
| 幂等去重 | `sync-server/internal/validator/` |
| Admin Web | `sync-server/internal/admin/` |
| 全局状态 sync_state.json | `sync-server/internal/syncstate/` |
| 客户端 mode 切换 | `sync-client/internal/modes/registry.go` |
| sqMode chunk upload | `sync-client/internal/sqmode/chunk_uploader.go` |
| 策略热加载 | `sync-client/internal/policy/store.go` |

proto 定义在两端 `proto/sync.proto`，**两份要保持同步**。

## 5. 本地开发

**Q：怎么本地跑起来？**

服务端：
```bash
cd sync-server
go run ./cmd/server -config config/config.yaml
```

客户端：
```bash
cd sync-client
go run ./cmd/client -config config/config.yaml
```

依赖：本地起 MySQL（源 + 目标）+ MinIO（源 + 目标）+ admin Web 看 :8090。

**Q：怎么测某个具体场景？**

`.claude/commands/` 下有几个 skill：
- `runtime-test.md` —— 运行时功能测试
- `pre-release-test.md` —— 发布前测试套件
- `deploy-server.md` —— 服务端部署

直接 `/pre-release-test` 调用。

## 6. 升级与发布

**Q：发版流程？**
A：
1. 在 feat 分支开发 → 测试通过
2. cherry-pick / merge 到 main
3. 本地 `cd sync-server && GOOS=linux GOARCH=amd64 go build -o /tmp/sync-server-new ./cmd/server`
4. 验证 `go tool nm /tmp/sync-server-new | grep <关键符号>` 确认改动在二进制里
5. SCP 到生产 → 备份旧 binary → systemctl stop → swap → start
6. 看服务端 `data/server.log` 有 "started successfully"
7. 看几个关键 client 心跳是否恢复

更详细：[reference/10-deployment-modes.md · 二进制升级流程](../reference/10-deployment-modes.md#c-二进制升级流程)。

## 7. 数据库变更

**Q：源端加了新列怎么办？**
A：服务端 Writer 默认会自动按客户端推过来的 schema 同步。**如果你担心列对齐**，最稳的做法：
1. 先在目标库 ALTER TABLE 加好列
2. 再让源端开始用

**Q：源端改了列类型怎么办？**
A：当前不会自动 ALTER TABLE，需要在目标库手动改。

## 8. 监控告警

**Q：哪些指标值得告警？**
A：
- 客户端 `marked offline` 持续 > 5 分钟 → 客户端进程死 / 网络断
- 服务端 `ERROR write block failed` 短时间内 > 10 次 → 目标基础设施异常
- 客户端 `sender paused` → 单 client 卡死
- catchup 倍率 < 0.5 持续 > 1 小时 → 调速或排查

**Q：有 Prometheus 接入吗？**
A：客户端 status 端口暴露 `/metrics`（v3.2 加的），但**没接完整的 Prometheus stack**。如果业务方有 Prom，可以自己 scrape。

## 9. 速查

| 你想问 | 看哪 |
|---|---|
| 加一张表 | 本文 §1 |
| 改 pacing | 本文 §2 + [reference/03](../reference/03-pacing-and-rate-control.md) |
| 报错查 | 本文 §3 |
| 代码在哪 | 本文 §4 |
| 怎么本地跑 | 本文 §5 |
| 怎么发版 | 本文 §6 + [reference/10](../reference/10-deployment-modes.md) |
| 故障处理 | [scenarios/02](../scenarios/02-incident-debug-workflow.md) |
| 客户端调速 | [scenarios/03](../scenarios/03-tuning-throughput.md) |
