# 场景 01 · 接入一个新客户端

## 角色与目标

**角色**：实施工程师 / 运维 / 客户成功

**目标**：从零部署一个新客户端，让它在中心机房 admin 看板上"上线 + 首次同步成功"。

**前提**：
- 服务端已正常运行（参考 [reference/10-deployment-modes.md](../reference/10-deployment-modes.md)）
- 客户机器可访问中心 sync-server 网络（直连或经 Nginx）
- 客户机有访问源端 MySQL + MinIO 的网络通路

## 全流程（约 30-60 分钟）

```
[1] 分配 client_id & token  ──►  [2] 准备客户端配置  ──►  [3] 打包客户端二进制
                                                              │
                                                              ▼
[6] admin 启用 + 调速      ◄──  [5] 首次扫描观察    ◄──  [4] 客户端机器部署 + 启动
```

## 步骤详解

### 1. 在 admin 后台预先创建客户端记录

进 admin Web（`http://<sync-server>:8090/`），点 "新增客户端"，填：

| 字段 | 说明 | 示例 |
|---|---|---|
| `client_id` | 全局唯一标识，建议 kebab-case | `<store-region-code>` |
| `target_database` | 目标 MySQL 库名 | `smart_quality_v2_<client_id>` |
| `minio_bucket` | 目标 MinIO bucket | `smart-quality-<client_id>` |
| `enabled` | **先保持 false**（首次启动只想注册不开始同步） | `false` |
| `pacing` | 用默认即可，后面调 | 见 [reference/03](../reference/03-pacing-and-rate-control.md) |

保存后服务端在 `sync_state.json` 创建一条记录。

### 2. 准备客户端配置文件

复制模板 `sync-client/config/config.yaml`，修改：

```yaml
server:
  address: "<sync-server-host>:30020"   # Nginx 反代或直连
  secret_key: "${SYNC_SECRET_KEY}"      # 环境变量注入
  path_prefix: "/grpc"                  # 经 Nginx 时设置

source:
  type: mysql
  host: <source-mysql-host>
  port: 3306
  user: <source-user>
  password: "${DB_PASSWORD}"
  database: <source-database>
  max_open_conns: 2

source_minio:                            # 如果业务有文件场景
  endpoint: <source-minio-host>:9002
  access_key: "${MINIO_ACCESS_KEY}"
  secret_key: "${MINIO_SECRET_KEY}"

tables:
  file_block_tables:                     # 关键: 配业务表结构
    - root_table: <root_table>
      time_field: updateTime
      id_field: id
      related:
        - table: <related_1>
          fk_field: <fk>
          fk_ref: id
          file_fields:                   # 如有文件字段
            - field: audioPath
              bucket_field: audioBucket

  simple_tables:                         # 非 file_block 的小表
    auto_discover: false
    include:
      - <simple_table_1>
    exclude:
      - migrations
      - audio_ai_req_log                 # ★ 大字段表必须排除

data:
  block_dir: ./data/blocks
  checkpoint_file: ./data/checkpoint.json
  client_id: "<client_id>"               # 跟 admin 创建的对齐
```

### 3. 打包客户端二进制

按 [reference/10-deployment-modes.md · 打包脚本](../reference/10-deployment-modes.md#windows-任务计划生产典型)：

```bash
cd sync-client
GOOS=windows GOARCH=amd64 CGO_ENABLED=0 \
  go build -trimpath -ldflags="-s -w" -o deploy/windows/sync-client.exe ./cmd/client

cd ..
zipname="sync-client-windows-main-$(git rev-parse --short HEAD)-$(date +%Y%m%d).zip"
(cd sync-client/deploy && zip -r "../../${zipname}" windows/)
```

把产出的 `.zip` 发给客户。

### 4. 客户机部署 + 启动

**Windows**：
1. 解压到 `C:\sync-client\`
2. 修改 `config.yaml`（按第 2 步生成的内容）
3. 用管理员 PowerShell：
   ```powershell
   cd C:\sync-client\windows
   .\install-task.ps1
   ```
4. 任务计划程序里查看"SyncClient"已注册并运行

**Linux**：
1. 拷 binary + config 到 `/opt/sync-client/`
2. 装 systemd unit（参考 [10-deployment-modes.md](../reference/10-deployment-modes.md#linux-systemd)）
3. `systemctl enable --now sync-client`

### 5. 首次扫描观察

**客户端侧**（看日志）：
```
INFO sync-client v3.x.x starting
INFO client registered to server client_id=<client_id>
INFO mode registry initialized current_mode=default
INFO scanner initialized incremental_interval_sec=120
WARN sync disabled by server, scanner paused   ← 因为 admin 那边 enabled=false
```

**服务端侧**（在 admin Web 或 `data/server.log`）：
```
INFO client registered client_id=<client_id>
INFO pacing policy pushed module=receiver client_id=<client_id> from_version=0 to_version=1
INFO client config updated client_id=<client_id> target_database=... minio_bucket=...
```

确认 client_id 在 admin 看板里**绿点显示 online**。

### 6. admin 启用同步 + 首次扫描

回到 admin Web 找到该客户端 → 把 `enabled` 切到 `true` → 保存。

观察接下来 1-3 分钟内：
- 服务端日志开始出现 `receiving block module=receiver block_id=fb-... client_id=<client_id>`
- 服务端开始 `INFO schema ensured module=writer table=smart_quality_v2_<client_id>.<table>`（首次会建表）
- 客户端日志开始 `INFO block sent and deleted module=sender block_id=...`
- 目标 MySQL 库 `smart_quality_v2_<client_id>` 出现表 + 第一批行

## 常见首次接入问题对照

| 现象 | 原因 | 解决 |
|---|---|---|
| 客户端启动后日志 `connection refused` | server.address 错或防火墙 | 用 `nc -zv <host> 30020` 验证可达 |
| `Unauthenticated` 错误 | secret_key 不匹配 | 服务端 + 客户端 `SYNC_SECRET_KEY` 必须相同 |
| admin 看板没看到 client_id | client_id 配置和 admin 预填的不一致 | 检查 `data/client_id.dat` vs admin 显示 |
| 启用 `enabled` 后没动静 | scanner 还在等下一个 interval | 默认 120s,耐心等 |
| 服务端 `block validation failed ALREADY_PROCESSED` | 客户端重启过 + 服务端去重生效 | 正常 · 是健康信号 |
| 第一次扫描就报 `received message larger than max` | block 体积超过 16 MB | `simple_tables_batch_size` 调小,或排除大字段表 |

## 检查点 / 验收清单

- [ ] admin 看板该 client_id 显示 online（绿点）
- [ ] 服务端 `data/sync_state.json` 里能查到该 client_id 配置
- [ ] 目标 MySQL `smart_quality_v2_<client_id>` 库存在且有表
- [ ] 至少有 1 个 `fb-*` 或 `sb-*` block 成功 written
- [ ] 客户机 `data/blocks/` 目录有内容（说明 scanner 在工作）
- [ ] 客户机 `data/checkpoint.json` 有水位记录

## 速查

| 想问 | 答 |
|---|---|
| 接入要多久 | 实施手熟 30 分钟内,新人 1 小时内 |
| 第一次扫描扫多少 | scanner 默认按 `batch_size=10` 走,前几小时主要在补历史 |
| 如何确认成功 | 目标库出现表 + 至少 1 个 block_written |
| 出问题怎么排 | [scenarios/02](./02-incident-debug-workflow.md) |
