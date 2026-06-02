# FAQ 01 · 客户常问问题（对外可答）

> 这份 FAQ **可以直接发给业务方 / 客户方**。内容只涵盖事实层。不含技术债、不含价格、不含真实客户名。

## 1. 是什么 / 解决什么

**Q：Sync-Data 是干什么的？**
A：把分布在多个门店 / 分支机构的 MySQL 业务数据，加上关联的文件（音频 / 图片等），增量同步到中心机房统一存储和分析。每个门店是一个独立的"客户端"，中心是一个统一的"服务端"。

**Q：和市面上 Canal / Debezium / Otter 有什么区别？**
A：那些工具基于 MySQL binlog 做近实时同步，强项是流式 CDC。Sync-Data 走表扫描 + 块同步路线，强项有三个：
1. **天然支持文件**：业务里的音频 / 图片随主表行一起搬，保持原子一致
2. **多客户端隔离**：每个客户端独立目标库 + 独立 bucket，跨客户端零干扰
3. **弱网友好**：客户端本地块队列做缓冲，断网后自动续传

不冲突，选哪个看场景。详见 [reference/06-competitive-landscape.md](../reference/06-competitive-landscape.md)。

## 2. 同步延迟

**Q：从源端修改一条数据，多久能看到在目标库？**
A：典型延迟 **几分钟级**，由三部分组成：
- 客户端扫描周期（默认 120s，可调）
- 网络传输（取决于上行带宽和 block 大小）
- 服务端写入（毫秒级）

**Q：能做秒级吗？**
A：把 `scan_incremental_interval_sec` 调到 30 秒可以更快，但代价是源库扫描压力上升。当前定位是"分钟级追近"，**不追求 binlog 级实时**。

**Q：网络断了会丢数据吗？**
A：不会。客户端把 block 写到本地磁盘后才删除源端读取游标；网络断时块在本地累积，恢复后自动续传。目标端有 `block_id` 幂等去重，重复推送只 ACK 不重复落库。

## 3. 数据准确性

**Q：怎么保证目标端数据和源端一致？**
A：三层保障：
1. **block 粒度的事务原子性**：一个 block 内的主表 + 关联表 + 文件，要么全部落库要么全部不落
2. **SHA256 checksum**：block 含 checksum，传输错乱会被检出
3. **服务端去重**：同一 `block_id` 二次推送直接 ACK，避免重复

但**目前不做 row-level 一致性校验**。如果业务关心"目标端有 100% 一致的快照"，需要额外的对账工具。

**Q：删除操作会同步吗？**
A：当前版本**只同步 INSERT / UPDATE**，不同步 DELETE。如果源端删除某行，目标端不会自动删除（因为是表扫描模型，没法检测删除）。

**Q：DDL（建表 / 改表）会同步吗？**
A：建表会（首次写入时服务端自动 CREATE TABLE）。ALTER TABLE 不自动同步 —— 列变更需要在目标库手动执行。

## 4. 安全

**Q：客户端连服务端走什么协议？加密吗？**
A：当前默认走 gRPC over HTTP/2 + HMAC token 认证。**传输加密通过 Nginx 反代上层加 TLS 实现**（推荐生产配置）。客户端 / 服务端代码本身也支持原生 TLS，但 Nginx 方案运维更简单。

**Q：客户端能看到其他客户端的数据吗？**
A：**不能**。每个客户端用独立的 `client_id` + 独立 token，服务端按 client_id 路由到独立的目标库 + 独立的 MinIO bucket。客户端 RPC 只能写自己的命名空间。

**Q：服务端凭据怎么管理？**
A：所有密码 / token 通过环境变量注入（如 `SYNC_SECRET_KEY` / `DB_PASSWORD` / `MINIO_SECRET_KEY`）。配置文件里用 `${VAR_NAME:default}` 语法引用。**不能在代码 / 文档里硬编码密钥**。

## 5. 故障处理

**Q：客户端崩溃了怎么办？**
A：systemd / Windows 任务计划会自动重启。重启后客户端：
1. 读 `data/checkpoint.json` 拿到上次同步水位
2. 读 `data/blocks/` 拿到未发完的块
3. 继续从断点扫源端 + 重发块

**Q：服务端崩溃了怎么办？**
A：systemd 自动重启。重启后服务端：
1. 读 `data/sync_state.json` 恢复每客户端配置
2. 读 `data/processed_blocks.json` 恢复去重表
3. 客户端心跳进来时正常处理

期间客户端的块会在本地累积，服务端恢复后自动追上。**服务端当前是单点**，没有主备切换（架构设计上的 trade-off，详见 [reference/06-competitive-landscape.md](../reference/06-competitive-landscape.md)）。

**Q：客户端数据落后好几天怎么办？**
A：先按 [scenarios/02 故障排查 SOP](../scenarios/02-incident-debug-workflow.md) 确诊根因，再按 [scenarios/03 调速](../scenarios/03-tuning-throughput.md) 调高带宽追上。

## 6. 部署与运维

**Q：服务端要部署什么前置依赖？**
A：MySQL（5.7 / 8.0） + MinIO（或 S3 兼容存储）。除此之外**没有 Kafka / ZK / Redis 这类重型组件**。详见 [reference/10-deployment-modes.md](../reference/10-deployment-modes.md)。

**Q：客户端可以装在 Windows 上吗？**
A：可以。提供 Windows 任务计划部署脚本（`install-task.ps1`），开机自启 + 失败自动重启。详见 [reference/10-deployment-modes.md · Windows 任务计划](../reference/10-deployment-modes.md#windows-任务计划生产典型)。

**Q：客户端要常驻、还是可以按需启停？**
A：建议常驻（systemd / 任务计划）。按需启停理论上可以，但会失去"准实时"特性。

**Q：服务端能跑多少个客户端？**
A：当前生产规模 10+ 个客户端稳定运行。设计上单服务端进程通过 `MaxConcurrentStreams=50` 支撑数十客户端没问题。如果未来需要数百 / 数千客户端，需要重新设计为分片或多实例。

## 7. 演进与版本

**Q：当前是什么版本？什么时候出新版本？**
A：当前主流版本是 **v3.2**（含 sqMode 大文件中继）。**v3.3 在开发中**（pull-based pacing + 错误分类器），具体上线时间没承诺，看测试与回归。详见 [reference/08-roadmap.md](../reference/08-roadmap.md)。

**Q：升级会停服多久？**
A：服务端升级走 stop → 备份旧二进制 → 替换 → start 流程，**典型停机 5-10 秒**。停机期间客户端日志会出现一次心跳失败，对客户端无破坏 —— 它会自动重连。

**Q：v3.x 还会兼容 V2 吗？**
A：V2 已在 commit `1dec55b` 删除，**不兼容**。如果你看到老的 README / 架构文档提到 Redis Stream / 64-slot 路由 / React 后台等，那些都是 V2 时代的，**当前代码不再有**。

## 8. 通用

**Q：源端 MySQL 一定要给 root 权限吗？**
A：不一定。只需要：
- `SELECT` 业务表
- `SHOW CREATE TABLE` 拉 schema（schemacache 需要）
- `SELECT MAX(updateTime)` 做 checkpoint sampling

最小权限可以用 readonly user。

**Q：可以同步到 PostgreSQL 吗？**
A：**当前不支持**。代码里只实现了 MySQL driver 适配。PostgreSQL 支持是设想中方向。

**Q：可以同步元数据（用户表 / 配置表）吗？**
A：可以，这正是 simple_block 模型的用途。在 `tables.simple_tables.include[]` 里配置即可。

**Q：能加自定义字段过滤吗？**
A：当前不支持 WHERE 过滤。scanner 是 `SELECT * FROM <table> WHERE updateTime > <ckpt>`，全列拉。如果有过滤需求是 *设想中* 的方向。

## 速查

| 你问 | 看哪 |
|---|---|
| 是什么 / 选不选 | 本文 §1 / [01 产品总览](../reference/01-product-overview.md) |
| 延迟多大 | 本文 §2 |
| 数据准吗 | 本文 §3 |
| 客户端能不能装 | 本文 §6 |
| 出问题怎么办 | 本文 §5 + [scenarios/02](../scenarios/02-incident-debug-workflow.md) |
