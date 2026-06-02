# 05 · 目标客户与部署形态

## 定义

Sync-Data 的"客户"指**部署 sync-client 客户端那一方**。每个客户在自己机房有一套源 MySQL + 业务系统，需要把数据同步到中心机房。**百科文档统一用 `<client_id>` 占位真实客户标识。**

## 典型客户画像

### 共同特征

- 自有源端 MySQL（5.7 / 8.0），数据量在 GB ~ 数百 GB 级
- 业务里有"主表 + 关联表 + 文件"的复合结构（如 检测主表 + 检测通道 + 音频文件）
- 边缘网络环境：带宽可能受限（家庭宽带 / 4G）、可能不稳定（短时断网常态）
- 不希望中心机房有人远程登录到他们机器上做日常运维

### 行业分布

| 行业类型 | 特征 | 典型同步需求 |
|---|---|---|
| 检测 / 质量管控 | 大量音频 / 图像文件 + 关联检测记录 | 文件 + 主表事务级一致 |
| 连锁零售 / 餐饮 | 多门店 POS 数据汇聚 | 偏小表多 · 高频低量 |
| 工业 IoT | 设备数据 + 工艺参数 + 报表 | 历史数据回填 + 实时增量 |
| 政企内网 | 内部业务库到中心数据仓 | 安全合规要求高、不接公网 |

> 上述行业是基于代码里看到的 schema 推测（如 `detect`、`detect_channel`、`audio_ai_req_log` 这些表名），不代表当前实际客户分布。

## 部署形态分类

### A. 客户端运行平台

| 平台 | 服务化方式 | 部署产物 |
|---|---|---|
| **Windows** | 任务计划程序（`install-task.ps1`）· 开机自启 + 失败重启 | `sync-client.exe` + `config.yaml` + `install-task.ps1` |
| **Linux** | systemd service · `WantedBy=multi-user.target` | `sync-client` 二进制 + systemd unit |
| **macOS** | 开发调试用 · 直接命令行运行 | `sync-client` 二进制 |

详细部署步骤见 [10-deployment-modes.md](./10-deployment-modes.md)。

### B. 网络接入方式

| 方式 | 适用场景 | 配置 |
|---|---|---|
| **直连** | 同一内网 / 测试环境 | `server.address: 127.0.0.1:50051` |
| **Nginx 反代** | 跨网段 / 经公网 | `server.address: <domain>:30020` + `server.path_prefix: /grpc` |
| **TLS 直连** | 生产跨网（替代 Nginx） | `tls.enabled: true` + `tls.ca_file` 等 |

### C. 源库部署关系

| 模型 | 描述 | 配置示例 |
|---|---|---|
| **单源 · 单客户端** | 一个客户端一个源库 · 一对一 | 1 个 sync-client 进程 |
| **单源 · 多端口客户端** | 同机部署多 mode 实例（不常见） | 用不同 `data_dir` 区分 |
| **多源 · 各自客户端** | 一家企业多套源库 · 每套一个客户端实例（不同 `client_id`） | 每个 client 独立配置 |

## 服务端端到端隔离

每个客户端的数据**完全隔离**：

| 资源 | 命名规则 |
|---|---|
| 目标 MySQL 库 | `smart_quality_v2_<client_id>` |
| 目标 MinIO bucket | `smart-quality-<client_id>` |
| 服务端 pacing 配置 | `sync_state.json[clients][<client_id>]` |
| 服务端 token | `tokens[<client_id>]` 或 HMAC 签名 |
| 服务端 admin 看板 | 每客户端独立卡片 |

这意味着：
- 跨客户端**不会出现数据串库**
- 单个客户端的 pacing / mode 调整**不影响其他客户端**
- 单个客户端可以独立 `enabled: false` 暂停（不影响别人）

## 客户接入要点

详见 [scenarios/01-new-client-onboarding.md](../scenarios/01-new-client-onboarding.md)。简要清单：

1. **在 admin 后台预先创建客户端记录**：分配 `client_id` 和 token
2. **生成客户端配置**：填入源库连接信息 + `client_id` + 服务端地址
3. **打包二进制**：把 `sync-client.exe` + `config.yaml` 一起发给客户
4. **客户机器部署**：装 systemd / 任务计划
5. **首次启动**：客户端注册到服务端，admin 看板看到上线
6. **admin 启用同步**：把 `enabled: false` 切到 `true`
7. **观察首次扫描**：admin 看板会显示 "catchup 倍率"，正常情况下应稳步增长

## 适用范围限制

| 不适用情况 | 原因 |
|---|---|
| 源是 PostgreSQL / Oracle / SQL Server | v3.x 只实现了 MySQL driver |
| 要求亚秒级延迟 | scanner 默认 120s 一次扫描 · 不读 binlog |
| 单表行数 > 10 亿 | scanner 是 `LIMIT` 分页，没有大表特殊优化 |
| 跨地域目标多机房 | 服务端是单进程 · 无主从无切换 · 设计上只有一个中心 |
| 客户端机器没磁盘 / 嵌入式 | blockstore 要本地磁盘缓冲，没磁盘走不通 |

## 当前生产规模（v3.x）

> 数据来自服务端 `sync_state.json` 公共字段，不含任何客户标识。

- **已部署客户端数**：10+ 个 active
- **服务端单实例累计同步**：100+ GB / 月（按所有客户端总量）
- **典型单客户端**：每天源端新增 ~150 MB（含文件），需要 ~6 小时上传完毕（按 50 kbps 限速）
- **运行时长**：服务端单实例稳态运行 3 周以上无重启

## 速查

| 你问 | 看哪 |
|---|---|
| 多客户端怎么不串数据 | 本文 "服务端端到端隔离" |
| 一个客户端怎么从零接入 | [scenarios/01](../scenarios/01-new-client-onboarding.md) |
| 我的客户网络不稳能用吗 | 本文 "典型客户画像" + [04 概念 · checkpoint](./04-key-concepts.md#3-checkpoint同步断点) |
| 客户用 macOS / Windows / Linux 都行吗 | 本文 "部署形态分类 · A" |
