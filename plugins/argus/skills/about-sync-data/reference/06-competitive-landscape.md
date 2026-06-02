# 06 · 技术方案对比（中性）

## 定义

本文档**不是商业竞品对比**，而是给"了解项目的同事 / 决策者"理清：在"MySQL 跨网络同步"这个问题域内，Sync-Data 跟主流开源方案的**事实层差异**。**写法保持中性、不贬低任何方案**。

## 对比维度

我们关心 7 个维度：
1. **同步源头**：binlog vs 表扫描
2. **目标端形态**：是否带文件、是否带 schema 自动同步
3. **网络模型**：客户端推 vs 服务端拉
4. **延迟典型值**：近实时 / 分钟级 / 小时级
5. **可观测与运维**：是否有 admin 面板、是否能远程调度
6. **多源多目标隔离**：能否做到 per-client database 隔离
7. **依赖外部组件**：是否需要 ZK / Kafka / Redis 等

## 事实对比表

| 维度 | **Sync-Data (本项目)** | **Debezium** | **Canal** | **Maxwell** | **Otter** |
|---|---|---|---|---|---|
| **同步源头** | 表扫描（`SELECT ... WHERE updateTime > ckpt`） | MySQL binlog | MySQL binlog | MySQL binlog | binlog + 反向解析 |
| **延迟典型值** | 分钟级（120s 扫描间隔默认） | 秒级 | 秒级 | 秒级 | 秒级 |
| **文件 / 二进制依赖** | **原生支持**（file_block 含 MinIO 文件） | 不支持 | 不支持 | 不支持 | 不支持（需自行配合） |
| **目标端 schema 自动建表** | **支持**（每客户端独立目标库 + 自动 DDL） | 不直接支持（输出到 Kafka） | 不直接支持 | 不直接支持 | 半支持 |
| **多源多目标隔离** | **强**（per-client database + bucket + pacing） | 弱（按 connector 配多个） | 弱（多 instance） | 弱 | 较强 |
| **网络模型** | **客户端推**（client → server gRPC） | 服务端从 MySQL 拉 binlog → Kafka | 服务端从 MySQL 拉 binlog | 类似 Canal | 双向通道 |
| **运维面板** | **内置 Admin Web :8090** | 通过 Kafka Connect REST | 有 admin (canal-admin) | 简陋 | 完整 manager 端 |
| **客户端能力** | 客户端就在源端机房 · 本地块队列离线缓冲 | 不分客户端（中心化） | 不分客户端 | 不分客户端 | 较中心化 |
| **依赖外部组件** | 仅目标 MySQL + 目标 MinIO | Kafka + Kafka Connect + ZK | ZK (HA 时) + (可选) RocketMQ/Kafka | 仅 Kafka | ZK + manager DB |
| **典型部署复杂度** | 客户端单 binary · 服务端单 binary · 简 | 高（Kafka stack） | 中 | 低（无 admin 时） | 中高 |
| **HA 支持** | **无**（服务端单点） | 有（Kafka 复制） | 有（基于 ZK） | 弱 | 有 |
| **写入语义** | At-least-once + 幂等去重（block_id） | 取决于下游 | At-least-once | At-least-once | Exactly-once（配置） |

## Sync-Data 的独特定位

> 把 4 件事**捆绑在一起**做的方案，本身不多见：

1. **表扫描 + 文件依赖一致性**：file_block 模型保证 "1 条 detect 行 + 4 条 detect_channel + 3 个 .wav 文件"作为一个 block 原子提交，**文件下载失败回放不丢任何关联**
2. **per-client 隔离**：每个客户端独立目标库 + 独立 bucket + 独立 pacing，跨客户端零干扰
3. **服务端可下发限速**：admin Web 改 `network_limit_kbps` 后客户端心跳周期内热生效，无需登客户端机器
4. **客户端本地块队列**：网络断开 / 服务端故障时块在客户端磁盘累积，恢复后自动追上

binlog 类工具在"近实时 + 流式 ETL + 大量小事务"场景更优；Sync-Data 在"业务复合数据 + 多源到隔离目标 + 弱网"场景更适合。

## 为什么不直接用 binlog 工具

事实记录在 [faq/03-decision-talking-points.md](../faq/03-decision-talking-points.md)。简要：

1. **文件场景**：源端 MinIO 里的 .wav / 图片需要随主表行一起搬，binlog 工具不能拿到关联文件
2. **目标库隔离**：要求每个客户端落到**独立的 MySQL 库 + 独立 MinIO bucket**，方便后续合规审计与数据迁移 —— binlog 工具的多 sink 配置复杂
3. **弱网环境**：客户端需要本地磁盘缓冲、断点续传、退避重试 —— binlog 工具的客户端模型不天然带这些
4. **跨网部署**：客户端在客户机房、服务端在中心机房，中间还有 Nginx 反代 / 公网穿透 —— gRPC 比 Kafka Connect 更轻
5. **没有 binlog 权限**：部分客户的源 MySQL 不开放 binlog（运维约束）

## 何时该考虑替换为 binlog 工具

如果未来出现以下情况之一，可以考虑评估迁移：

| 情况 | 推荐方案 |
|---|---|
| 延迟要求 < 10s | Canal / Debezium |
| 需要支持 PostgreSQL / Oracle 源 | Debezium |
| 单源数据量 > 1 TB / 天 | Debezium + Kafka |
| 业务需要 CDC 流到多个异构下游（Kafka、ES、ClickHouse） | Debezium |
| 不再有"文件 + 行"复合一致性需求 | 任何 binlog 方案 |

## 路线图相关

v3.3 在做 **pull-based pacing**，进一步缩短限速生效延迟。详见 [08-roadmap.md](./08-roadmap.md)。

## 速查

| 问 | 答 |
|---|---|
| 我们这个跟 Canal 是什么关系 | 互补不替代 · 表扫描 vs binlog · 强项不同 |
| 切到 Debezium 难吗 | 难 · 文件场景需要额外开发 |
| 哪些场景不该用 Sync-Data | 上面"何时该考虑替换"那张表 |
