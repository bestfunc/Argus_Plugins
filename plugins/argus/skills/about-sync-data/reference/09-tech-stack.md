# 09 · 技术栈

## 定义

Sync-Data 的技术栈刻意保持**最小可行集合**。整体只依赖 Go 标准库 + 几个稳定第三方库 + MySQL + MinIO。**没有**消息队列、ZooKeeper、Etcd、Redis、Kafka 这些常见的"重型中间件"。

## 核心栈

| 分类 | 服务端 | 客户端 |
|---|---|---|
| **语言** | Go 1.24 | Go 1.25 |
| **构建** | `go build` · `CGO_ENABLED=0` · 静态二进制 | 同 |
| **传输** | gRPC over HTTP/2 (h2c) | 同 |
| **目标数据库** | MySQL 5.7 / 8.0+ | (不直连) |
| **目标对象存储** | MinIO | (源端 MinIO 拉文件) |
| **配置格式** | YAML | YAML |
| **状态持久化** | JSON (本地文件) | JSON (本地文件) |
| **日志** | `log/slog` (标准库) | 同 |

## 第三方依赖（go.mod）

服务端 `sync-server/go.mod`：
```
google.golang.org/grpc v1.79.3       // gRPC 实现
google.golang.org/protobuf v1.36.10  // protobuf 运行时
github.com/go-sql-driver/mysql v1.9.3 // MySQL driver
github.com/minio/minio-go/v7 v7.0.66  // MinIO SDK
golang.org/x/net (http2/h2c)         // h2c 处理
gopkg.in/yaml.v3                     // YAML 解析
```

客户端 `sync-client/go.mod` 大致同名集合 + 额外的 modes / sqmode 内部依赖。

**特点**：每个第三方包都是"工业级稳定 + 单一职责"，没引入框架（无 echo / gin / kratos），整个 RPC 层贴 gRPC 标准库写。

## 标准库使用

| 标准库包 | 用途 |
|---|---|
| `context` | 全链路上下文传递 + 取消 |
| `log/slog` | 结构化日志（key-value） |
| `database/sql` | MySQL 抽象层 |
| `net/http` + `golang.org/x/net/http2` | h2c 服务端 |
| `os/signal` + `syscall` | 优雅退出（SIGTERM / SIGINT） |
| `path/filepath` + `os` | 块文件系统操作 |
| `sync` + `atomic` | 并发原语（避免引入第三方并发库） |
| `time` | 心跳定时 / 退避 |
| `runtime/debug` | `SetMemoryLimit` 内存限制 |
| `crypto/sha256` | block checksum |
| `encoding/json` | 状态文件序列化 |

## 协议层

### gRPC

- 单一 service：`syncpb.SyncService`
- 已定义 RPC 见 [02-architecture.md · gRPC 接口清单](./02-architecture.md#grpc-接口清单)
- proto 文件位置：`sync-client/proto/sync.proto` 和 `sync-server/proto/sync.proto`（保持手动同步）
- 拦截器：服务端 `AuthStreamInterceptor / AuthUnaryInterceptor` 做 token 校验

### HTTP

- 仅服务端 admin：`:8090`
- 框架：`net/http` 标准库 ServeMux
- 内容：JSON API + 静态 HTML（admin Web）

### h2c（HTTP/2 over cleartext）

服务端开启方式（`sync-server/cmd/server/main.go`）：
```go
h2s := &http2.Server{}
httpServer := &http.Server{
    Handler: h2c.NewHandler(grpcHandler, h2s),
}
```

好处：避免 TLS 证书运维负担。**生产强烈建议在 Nginx 反代上层做 TLS**。

## 持久化

### 服务端

| 文件 | 内容 | 写入频率 |
|---|---|---|
| `data/sync_state.json` | 每客户端配置 + 统计 + pacing | admin 写时 + 心跳触发 |
| `data/processed_blocks.json` | 幂等去重表 | 块完成时增量写 + 24h TTL 清理 |
| `data/checkpoint_samples.json` | admin 进度面板样本 | 每 60s 写 |

读写**都用普通文件 + `os.Rename`** 做原子替换，不引数据库。

### 客户端

| 文件 | 内容 |
|---|---|
| `data/blocks/YYYY-MM-DD/<id>/meta.json` | block 元数据 |
| `data/blocks/YYYY-MM-DD/<id>/files/*` | 关联文件 |
| `data/blockstore_state.json` | rootIndex 索引快照 |
| `data/checkpoint.json` | 每表同步水位 |
| `data/client_id.dat` | 持久化 client_id |

## 部署产物

| 产物 | 大小 (典型 strip 后) | 内容 |
|---|---|---|
| `sync-server-linux` | ~24 MB | Linux amd64 静态二进制 |
| `sync-client.exe` | ~15 MB | Windows amd64 静态二进制 (`-ldflags "-s -w"`) |
| `sync-client-windows-<branch>-<sha>-<date>.zip` | ~5.8 MB | Windows 分发包（含 install/uninstall 脚本） |

## 不使用的技术（刻意选择）

| 类型 | 不用什么 | 原因 |
|---|---|---|
| **消息队列** | Kafka / RabbitMQ / Redis Stream | 客户端推模式 + 本地块队列 = 已有缓冲 |
| **协调器** | ZK / Etcd | 服务端单点 · 不需要分布式协调 |
| **缓存** | Redis / Memcached | sync_state.json 内存够 |
| **DI 框架** | wire / fx | 项目规模没到 |
| **ORM** | gorm / ent | `database/sql` 加手写 SQL 足够 |
| **配置中心** | Apollo / Nacos | 配置量小 · 本地 YAML + 服务端下发 pacing 已覆盖 |
| **追踪 / Tracing** | OpenTelemetry / Jaeger | 当前有 `traceid` 内部包，未引入完整 OTel |
| **TLS 内建** | mTLS / SPIFFE | h2c + Nginx 上层做 TLS 更简单 |

## 与其他常见组件的关系

| 组件 | 当前状态 | 备注 |
|---|---|---|
| MySQL | 必需（源 + 目标） | 5.7 / 8.0 都行 |
| MinIO | 必需（目标）/ 可选（源） | 也可用 S3 兼容存储 |
| Nginx | 可选（生产推荐） | 跨网部署 + TLS 终结 |
| Docker | 服务端有 Dockerfile | `sync-server/Dockerfile` + `sync-server/deployments/docker-compose.yaml` |
| systemd | 客户端 Linux 部署用 | 在 [10-deployment-modes.md](./10-deployment-modes.md) |
| Windows 任务计划 | 客户端 Windows 部署用 | 同上 |
| Prometheus | 暂无 metrics 端点 | 客户端 status 端口有 `/metrics`，未完整集成 |

## 资源占用（实测）

服务端：
- 内存稳态：~100 MB（10 客户端规模）
- 内存峰值：~180 MB（追积压时）
- CPU：稳态 < 5%（10 客户端规模）

客户端：
- 内存上限：`memory_limit_mb=200`（pacing 默认）
- 块磁盘占用：因网络通畅程度而异，正常 < 100 MB；离线时累积

## 速查

| 想知道 | 答 |
|---|---|
| Go 版本要求 | 服务端 ≥ 1.24，客户端 ≥ 1.25 |
| 要不要装 Redis | 不要 |
| 数据库一定 MySQL 吗 | 当前是 |
| 怎么做 TLS | Nginx 上层做（推荐）或 `tls.enabled: true` 直连 |
| 重要依赖 | gRPC, MySQL driver, MinIO SDK, YAML |
