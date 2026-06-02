# 08 · 路线图

> 本文档讲已完成的演进 + 当前在做的事 + 设想中的方向。**未来部分一律用"设想 / 计划 / 在调研"措辞**。

## 定义

Roadmap 把 Sync-Data 的演进分三段：**已上线（fact）/ 在做（in-progress）/ 设想（concept）**。事实层依据是 `git log`、`docs/plans/`、`docs/superpowers/plans/`，不臆测时间表。

## 一、已上线（fact）

### V2 时代（2026-01 ~ 2026-02）

来自旧 commit 与 `architecture-docs/`：

- 设计了基于 Redis Stream 7.x 的事务级聚合管道
- 64-slot V4 路由调度
- React + Vue 双前端管理界面
- PostgreSQL 14+ 双目标支持

**该架构整体在 commit `1dec55b` 删除**：留下的代码已不能反映 V2 设计。`README.md` / `architecture-docs/` 仍是 V2 时代文档，**已不准确**。

### V3.0 完整重构（2026-03 ~ 2026-05）

关键 commits：
- `1dec55b refactor: 删除 V2 模块，准备 V3.0.0 重构`
- `7705b31 feat: V3 gRPC 协议 + Block 数据结构`
- `870f28c feat(client): V3 配置 + Checkpoint + ResourceLimiter`
- `5b71eb6 feat(client): BlockStore 本地块存储 + SchemaCache 表结构缓存`
- `f22821b feat(client): Scanner 增量扫描 + 文件块构建 + 普通表自动发现`
- `5717790 feat(client): 简化 NetworkMonitor + 重写 Sender 块发送`
- `190a6b5 feat: 客户端/服务端 V3 main.go + Heartbeat 客户端`
- `ee84b70 feat(server): BlockReceiver + V3配置 + Validator + BlockWriter`
- `7b7dfeb feat: V3.0.0 客户端/服务端完整重构 + 运行时测试验证`

成果：
- 去掉 Redis Stream / 64-slot 路由 / 双前端，全部砍掉
- 服务端单 binary 单进程；客户端单 binary 单进程
- block 模型 + 三层 pacing + 心跳 + Admin Web 全部到位
- 一句话总结：**简单可控、好部署、可维护**

### V3.1（2026-05 中）

- blockstore 按 `YYYY-MM-DD` 子目录分类（避免单目录万级条目）
- 自动迁移老格式 block 到 `legacy/`
- 新增 BlockStore goroutine 维护索引刷盘

### V3.2 - sqMode（2026-05 中下旬）

关键 commit：`594dc71 feat(sqmode v3.2): 上传重试 + status HTTP + 目标库统一 + 前端 sqMode 视图`

成果：
- 引入 **modes 注册表** + `default` / `sqMode` 两种运行模式
- 新增 **ChunkUpload** RPC：MinIO multipart 中继通道，8 MiB 分片
- 引入 `pacing_version` 协议，服务端可远程切 mode
- status HTTP 端口暴露 `/healthz` `/metrics`
- 服务端按 `block.Mode` 路由 + bucket 自动创建

V3.2 主要是为 SmartQuality 客户端（含大量音频文件、文件大小不均一）做的专门通道。

### V3.2.x（2026-05 ~ 2026-06）补强

- `46e9dee fix(sync-server): 将 grpc Max{Recv,Send}MsgSize 提到 16MB`（解大字段表卡死）
- `8df830d fix(client): probe 连续失败时主动 invalidate 连接，防止 sender 死锁`
- `27eb91c test(client): 抽出 probe 失败计数器到 probetracker 子包并补单测`
- `6012bce fix(sync-server): clientstore offline 阈值单位 ns→s（90→90*time.Second）`

## 二、在做（in-progress）

### V3.3 - Pull-based pacing & 隔离基础设施

设计文档：`docs/plans/2026-06-02-v3.3-pull-based-pacing.md`

**已合入 main 的 batch**：
- `a84d742 feat(sqmode v3.3): batch 1 — proto/schema + 隔离基础设施 + 错误分类器`
- `694c4c2 feat(sync-server): knownTables 缓存加 30 分钟 TTL + flush API（v3.3）`
- `b5be099 docs(plan): v3.3 pull-based pacing 设计文档定稿`

**计划方向**（措辞 = 设计中）：

1. **客户端主动 pull 策略**：客户端按需向服务端请 pacing（带版本号 + 弱 hash），无变化时短返回
2. **服务端可发广播事件**：客户端立刻拉策略 + 立刻刷新 knownTables 缓存
3. **错误分类器**：服务端根据错误类型自动调整 pacing（如 `ResourceExhausted` 自动 `batch_size /= 2`）
4. **隔离基础设施**：把不同 mode / 不同 client 的逻辑更彻底分开，避免 v3.2 时代某个 mode 改动影响其他

### 同期质量补强

- 测试覆盖率提升（commit `79c3bc2 test: 修复 sender_test 与 limiter_test 预存编译错误`）
- 死锁 / 卡死场景的根因修复

## 三、设想（concept）

这部分**不承诺、不排期**，仅作信息分享：

- **客户端崩溃恢复** 更稳：当前 blockstore 索引刷盘有一定延迟，崩溃可能丢未刷盘的索引；设想用 WAL 风格补强
- **PostgreSQL 源支持**：当前只跑 MySQL；如果未来有客户用 PG 源端，需要重写 scanner 的 schema 反射部分
- **更细粒度幂等**：当前 `processed_blocks.json` 是 block 级粒度；设想按 row hash 做 row 级幂等，但成本待评估
- **服务端高可用**：当前单点 · 设想主备 + 共享 sync_state.json on NFS 或 etcd · 待业务规模驱动

## 进度速查表

| 阶段 | 状态 | 代表 commit / 文档 |
|---|---|---|
| V1 ~ V2 | 已废弃 | commit `1dec55b` 删除 |
| V3.0 重构 | ✅ 上线 | `7b7dfeb` |
| V3.1 blockstore 优化 | ✅ 上线 | (无明确 tag) |
| V3.2 sqMode | ✅ 上线 | `594dc71` |
| V3.2 补强 | ✅ 上线 | `46e9dee` 等 |
| V3.3 pull-based pacing | 🔧 在做 | `a84d742` (batch 1) |
| WAL / PG / HA | 💭 设想 | — |

## 推荐阅读

- 历史决策细节：`docs/superpowers/plans/2026-05-20-sqmode-v3.2.md` (sqMode 设计) 和 `docs/superpowers/plans/2026-05-20-sqmode-v3.2-r2.md` (二轮设计)
- v3.3 详细计划：`docs/plans/2026-06-02-v3.3-pull-based-pacing.md`
- v3 重构溯源：`docs/superpowers/plans/2026-03-30-v3-refactor.md`
