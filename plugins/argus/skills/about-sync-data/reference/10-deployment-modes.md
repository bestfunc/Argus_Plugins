# 10 · 部署模式

## 定义

Sync-Data 是**二进制部署**（无容器要求 / 无运行时依赖）。客户端单文件 + 配置文件 + 服务化脚本即可。服务端略复杂，需要先准备 MySQL 与 MinIO。

## 服务端部署

### 拓扑

```
                          ┌────────────────────┐
                          │    Nginx (可选)     │
                          │  TLS 终结 + 路径分流 │
                          └─────────┬──────────┘
                                    │
                                    ▼  :50051 / :8090
                          ┌────────────────────┐
                          │  sync-server       │
                          │  (单 binary)        │
                          └─────┬──────────┬───┘
                                │          │
                                ▼          ▼
                       目标 MySQL      目标 MinIO
                          :3306          :9000
```

### 必需依赖

- **MySQL 5.7 / 8.0**：每个客户端有独立目标库 `smart_quality_v2_<client_id>`，需要建库权限
- **MinIO**（或 S3 兼容存储）：每个客户端独立 bucket `smart-quality-<client_id>`，需要建桶权限
- **Linux 主机**：amd64 / arm64，至少 2C4G，建议 4C8G

### 部署方式

#### A. systemd 直接运行（生产推荐）

`/etc/systemd/system/sync-server.service`：
```ini
[Unit]
Description=Sync Server
After=network-online.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/root/sync-server
ExecStart=/root/sync-server/bin/sync-server -config config/config.yaml
Restart=on-failure
RestartSec=5s
StandardOutput=append:/root/sync-server/data/server.log
StandardError=append:/root/sync-server/data/server.log

[Install]
WantedBy=multi-user.target
```

操作：
```bash
systemctl daemon-reload
systemctl enable --now sync-server
systemctl status sync-server
```

日志查看：`tail -f /root/sync-server/data/server.log`

#### B. Docker Compose

`sync-server/deployments/docker-compose.yaml` 已提供模板。生产偏向不用 Docker，原因：
- 服务端是单进程，没有复杂 service 拓扑
- `data/*.json` 持久化更适合直接挂磁盘
- 升级时需要 systemd 控制粒度

#### C. 二进制升级流程

参考 session 内实际操作（commit `46e9dee` 部署时用过）：
```bash
# 1. 准备新二进制
scp sync-server-new root@<host>:/root/sync-server/bin/sync-server.new

# 2. 验证可执行
ssh <host> '/root/sync-server/bin/sync-server.new -h'

# 3. 备份 + 切换
ssh <host> '
  ts=$(date +%Y%m%d_%H%M%S)
  cp /root/sync-server/bin/sync-server /root/sync-server/bin/sync-server.bak.${ts}
  systemctl stop sync-server
  mv /root/sync-server/bin/sync-server.new /root/sync-server/bin/sync-server
  chmod +x /root/sync-server/bin/sync-server
  systemctl start sync-server
  systemctl status sync-server
'
```

`.claude/commands/deploy-server.md` 是封装好的部署 skill，可一键完成上述流程。

### 配置文件（关键字段）

`sync-server/config/config.yaml`：
```yaml
server:
  grpc_port: 50051             # gRPC 监听
  max_connections: 50          # gRPC MaxConcurrentStreams
  secret_key: "${SYNC_SECRET_KEY:…}"
  tls:
    enabled: false             # 生产环境改 true 或经 Nginx

target:                        # 目标 MySQL
  type: mysql
  host: ...
  port: 23306
  user: root
  password: "${DB_PASSWORD:…}"
  max_open_conns: 10
  max_idle_conns: 5

target_minio:                  # 目标 MinIO
  endpoint: ...
  access_key: "${MINIO_ACCESS_KEY:…}"
  secret_key: "${MINIO_SECRET_KEY:…}"
  use_ssl: false
  default_bucket: ...

admin:
  http_port: 8090              # Admin Web
  username: admin
  password: admin123           # 生产改强密码

data:
  temp_dir: ./data/temp
  processed_blocks_ttl: 24h
```

## 客户端部署

### Windows 任务计划（生产典型）

部署产物（`sync-client/deploy/windows/`）：
```
sync-client.exe        二进制 (~15 MB strip 后)
config.yaml            配置模板
install-task.ps1       注册任务计划（开机自启 + 失败重启）
uninstall-task.ps1     卸载脚本
```

部署步骤：
1. 把 `sync-client-windows-<branch>-<sha>-<date>.zip` 拷到目标机
2. 解压到 `C:\sync-client\`
3. 修改 `config.yaml`：填 source / server.address / client_id
4. 以管理员权限运行 `install-task.ps1`
5. 任务计划自动启动；日志在 `C:\sync-client\logs\`

打包脚本（在仓库根目录）：
```bash
cd sync-client
GOOS=windows GOARCH=amd64 CGO_ENABLED=0 \
  go build -trimpath -ldflags="-s -w" -o deploy/windows/sync-client.exe ./cmd/client

cd ..
branch=$(git rev-parse --abbrev-ref HEAD)
sha=$(git rev-parse --short HEAD)
date=$(date +%Y%m%d)
zipname="sync-client-windows-${branch}-${sha}-${date}.zip"
(cd sync-client/deploy && zip -r "../../${zipname}" windows/)
```

### Linux systemd

`/etc/systemd/system/sync-client.service`（参考服务端写法，改路径与二进制）：
```ini
[Unit]
Description=Sync Client (<client_id>)
After=network-online.target

[Service]
Type=simple
User=sync
Group=sync
WorkingDirectory=/opt/sync-client
ExecStart=/opt/sync-client/sync-client -config config/config.yaml
Restart=on-failure
RestartSec=5s
StandardOutput=append:/opt/sync-client/logs/client.log
StandardError=append:/opt/sync-client/logs/client.log

[Install]
WantedBy=multi-user.target
```

### macOS（调试）

直接 `./sync-client -config config/config.yaml`，前台跑，方便看输出。不建议生产用 macOS。

### 客户端配置（关键字段）

```yaml
server:
  address: "<sync-server>:30020"   # Nginx 反代地址 (生产)
  secret_key: "${SYNC_SECRET_KEY:...}"
  path_prefix: "/grpc"             # 经 Nginx 时设置
  tls:
    enabled: false

source:                            # 源 MySQL
  type: mysql
  host: 127.0.0.1
  port: 3306
  user: root
  database: <source_database>
  max_open_conns: 2
  max_idle_conns: 1

source_minio:                      # 源 MinIO (file_block 需要)
  endpoint: 127.0.0.1:9002
  access_key: "${MINIO_ACCESS_KEY:...}"
  secret_key: "${MINIO_SECRET_KEY:...}"

data:
  block_dir: ./data/blocks
  checkpoint_file: ./data/checkpoint.json
  client_id: "<client_id>"         # 关键 · 必须填或留空让首次启动生成
  client_id_file: ./data/client_id.dat
```

### 客户端启停

```bash
# Linux
systemctl start sync-client
systemctl stop sync-client
systemctl restart sync-client
journalctl -u sync-client -f

# Windows
Start-ScheduledTask -TaskName "SyncClient"
Stop-ScheduledTask -TaskName "SyncClient"
# 查日志: C:\sync-client\logs\client.log
```

## Nginx 反代配置（生产参考）

`/etc/nginx/conf.d/sync.conf`：
```nginx
server {
  listen 30020 http2;
  server_name <sync-server>;

  # TLS (可选)
  # ssl_certificate ...
  # ssl_certificate_key ...

  # /grpc 路径分流到 gRPC
  location /grpc/ {
    grpc_pass grpc://127.0.0.1:50051;
    grpc_set_header Host $host;
    grpc_read_timeout 600s;
    grpc_send_timeout 600s;
    client_max_body_size 32m;     # 略大于 gRPC 16MB 上限
  }

  # /admin 路径分流到 admin HTTP (可选)
  location /admin/ {
    proxy_pass http://127.0.0.1:8090/;
    proxy_set_header Host $host;
  }
}
```

客户端配置对应：
```yaml
server:
  address: "<domain>:30020"
  path_prefix: "/grpc"
```

服务端代码会处理 `/grpc` 路径分流（commit `738c828`）。

## 部署检查清单

### 服务端首次部署

- [ ] MySQL 已启动，root 有 CREATE DATABASE 权限
- [ ] MinIO 已启动，access_key 有 CREATE BUCKET 权限
- [ ] 防火墙开放 `:50051`（直连）或 Nginx 端口
- [ ] `config.yaml` 中环境变量 `SYNC_SECRET_KEY` 等已配置
- [ ] `data/` 目录存在且有写权限
- [ ] systemd unit 启用 + 启动后看 `data/server.log` 有 "sync-server v3.x.x started successfully"

### 客户端首次部署

- [ ] 源 MySQL 可连（用 `mysql -h ... -u ... -p` 验证）
- [ ] 源 MinIO 可连（如果配置了）
- [ ] `client_id` 已在 admin 后台预先创建 + 分配 token / secret
- [ ] 配置文件里 `server.address` 与服务端实际对应（直连 / Nginx）
- [ ] 服务化脚本（systemd / 任务计划）已注册
- [ ] 启动后 admin 看板能看到该 client_id 上线
- [ ] Admin 把 `enabled: false` 切到 `true` 触发首次同步

## 速查

| 想干嘛 | 看哪 |
|---|---|
| 部署一个新服务端 | 本文 "服务端部署" |
| 给客户出 Windows 包 | 本文 "Windows 任务计划" + 打包脚本 |
| 升级线上服务端 | 本文 "B. Docker / C. 二进制升级流程" + `.claude/commands/deploy-server.md` |
| 接入新 client | [scenarios/01](../scenarios/01-new-client-onboarding.md) |
| Nginx 怎么配 | 本文 "Nginx 反代配置" |
