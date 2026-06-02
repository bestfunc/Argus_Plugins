---
name: about-sync-data
display_name: Sync-Data 产品百科
description: Sync-Data 多客户端数据同步系统的方方面面 — 给开发同事 / 实施运维 / 决策者 / 速查者查阅。涵盖架构、模块、配置、出块流程、限速策略、技术演进、典型故障场景。能根据业务需求给出使用建议。
user-invocable: true
---

# Sync-Data 产品百科

这是 **Sync-Data 多客户端数据同步系统** 的**自助百科**。无论你是开发同事 / 实施运维 / 决策者 / 想查个术语，都能从这里查到全部产品知识，并根据业务场景给出合理建议。

不写代码 / 不调用工具。如果需要操作集群，去用 `.claude/commands/deploy-server.md` 等部署命令文档。

---

## 我是谁？我从哪里看起？

### 🧑‍💻 我是开发同事 / 新员工，第一次接手这个项目

按这个顺序看：
1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 5 分钟搞清 sync-data 是干什么的、目标场景、当前 v3.x 状态
2. **[reference/02-architecture.md](./reference/02-architecture.md)** — 端到端数据流：源 MySQL → client → gRPC → server → 目标 MySQL/MinIO；含模块图
3. **[reference/04-key-concepts.md](./reference/04-key-concepts.md)** — file_block / simple_block / pacing / sqMode 等术语和它们之间的关系
4. **[reference/09-tech-stack.md](./reference/09-tech-stack.md)** — Go + gRPC + MySQL + MinIO，看清依赖边界
5. **[scenarios/02-incident-debug-workflow.md](./scenarios/02-incident-debug-workflow.md)** — 顺一遍故障排查流程，能独立处理常见卡死/落后

### 🧑‍🔧 我是实施/运维工程师，要部署或维护一个客户端

按这个顺序看：
1. **[reference/10-deployment-modes.md](./reference/10-deployment-modes.md)** — 客户端二进制部署 + systemd / Windows 任务计划
2. **[reference/03-pacing-and-rate-control.md](./reference/03-pacing-and-rate-control.md)** — 限速 / 退避 / 心跳 / 三类参数体系
3. **[scenarios/01-new-client-onboarding.md](./scenarios/01-new-client-onboarding.md)** — 一个新客户从零到首次同步成功
4. **[scenarios/03-tuning-throughput.md](./scenarios/03-tuning-throughput.md)** — 已上线客户端追不上时怎么调
5. **[faq/02-developer-questions.md](./faq/02-developer-questions.md)** — 部署 / 日志 / 常见报错对照表

### 🧐 我是决策者 / PM / 想了解项目全貌

按这个顺序看：
1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 是什么、解决什么、当前哪些客户在跑
2. **[reference/06-competitive-landscape.md](./reference/06-competitive-landscape.md)** — 与 Debezium / Canal / Maxwell 等 CDC 工具的差异（中性技术对比，非商业竞品）
3. **[reference/08-roadmap.md](./reference/08-roadmap.md)** — v3.0 → v3.2 → v3.3 演进路径与设计动机
4. **[faq/03-decision-talking-points.md](./faq/03-decision-talking-points.md)** *(内部参考，不对外发)* — 为什么选自研而不用 CDC

### 📚 我只是想查个术语 / 概念

直接 → **[glossary/terms.md](./glossary/terms.md)**

---

## 完整索引

### Reference（事实层）— `reference/`

| 文件 | 内容摘要 |
|---|---|
| [01-product-overview.md](./reference/01-product-overview.md) | 是什么、目标场景、当前规模、版本节奏 |
| [02-architecture.md](./reference/02-architecture.md) | 端到端架构图 / 模块图 / RPC 清单 / 存储布局 |
| [03-pacing-and-rate-control.md](./reference/03-pacing-and-rate-control.md) | 限速三层体系 / 退避策略 / 心跳协议 / pacing_version |
| [04-key-concepts.md](./reference/04-key-concepts.md) | block 模型 / mode 体系 / checkpoint / 幂等去重 |
| [05-target-customers.md](./reference/05-target-customers.md) | 典型客户画像（多门店企业 / 边缘机房）+ 部署形态分类 |
| [06-competitive-landscape.md](./reference/06-competitive-landscape.md) | 与 Debezium / Canal / Maxwell / Otter 的事实对比表 |
| [08-roadmap.md](./reference/08-roadmap.md) | v3.0 重构 → v3.2 sqMode → v3.3 pull-based pacing |
| [09-tech-stack.md](./reference/09-tech-stack.md) | Go / gRPC / MySQL / MinIO / 关键依赖版本 |
| [10-deployment-modes.md](./reference/10-deployment-modes.md) | 二进制部署 + systemd / Windows scheduled task + Nginx 反代 |

### Scenarios（场景层）— `scenarios/`

| 文件 | 内容摘要 |
|---|---|
| [01-new-client-onboarding.md](./scenarios/01-new-client-onboarding.md) | 从 0 部署一个新客户端到 admin 配置完成、首次成功同步 |
| [02-incident-debug-workflow.md](./scenarios/02-incident-debug-workflow.md) | 客户端落后 / 卡死时的诊断 SOP（含真实复盘案例） |
| [03-tuning-throughput.md](./scenarios/03-tuning-throughput.md) | pacing 调速：kbps / batch / 退避 几个旋钮的搭配选择 |

### FAQ（话术层）— `faq/`

| 文件 | 内容摘要 |
|---|---|
| [01-customer-questions.md](./faq/01-customer-questions.md) | 对外可答的常见问题（同步延迟 / 数据准确性 / 安全） |
| [02-developer-questions.md](./faq/02-developer-questions.md) | 开发 / 运维内部问题（如何加新表、报错对照） |
| [03-decision-talking-points.md](./faq/03-decision-talking-points.md) *(内部，不对外)* | 选型理由、技术债清单、未公开计划 |

### Glossary（术语层）— `glossary/`

| 文件 | 内容摘要 |
|---|---|
| [terms.md](./glossary/terms.md) | 全部术语 + 缩写 + 跨文件引用入口 |

---

## 当你被问到的时候

> **新人开发**：sync-data 是干嘛的？跟 binlog 同步有什么区别？
> **你**：先看 `reference/01-product-overview.md` 的"5 分钟搞清"和 `reference/06-competitive-landscape.md` 的对比表，能给出 3 句话回答 + 1 张架构图。

> **运维**：`<client_id>` 这个客户端追不上了，怎么排？
> **你**：照 `scenarios/02-incident-debug-workflow.md` 的步骤走（服务端 log → sync_state.json → 目标库 MAX(createTime) → MinIO 上传记录 → 客户端 sync-client.log），能给出确诊。

> **决策者**：要不要直接换成 Canal/Debezium？
> **你**：去 `reference/06-competitive-landscape.md` 看事实对比，再去 `faq/03-decision-talking-points.md` 看自研保留的理由（含文件场景 + 多源到多目标的隔离需求）。

> **任何人**："block" "checkpoint" "pacing" 这些词到底什么意思？
> **你**：`glossary/terms.md` 全部收录，跨文件链回。

> **PM**：v3.3 在做什么？什么时候上线？
> **你**：`reference/08-roadmap.md` 有 v3.3 的目标 (pull-based pacing) 和当前阶段（设计完，proto/schema 已合入 commit `a84d742`）。路线图章节用"计划/设想"措辞，不承诺时间。

---

## 维护说明（给写 skill 的人看）

- **事实来源**：以 `sync-client/` `sync-server/` 源代码 + `docs/plans/` 设计文档 + `docs/superpowers/plans/` 历史规划 + git log 为准。**不要**采信根目录的 `README.md` / `architecture-docs/` / `模块解释.md` 等 V2 时代文档（V2 模块已在 commit `1dec55b` 删除，那些 .md 没同步更新）。
- **凡涉及"未来路线"** 用 *计划 / 路线图 / 设想 / 在调研* 等词修饰，不用 *承诺 / 一定 / 必须 / 即将上线* 等词。
- **跨文档引用** 一律相对路径，方便 GitHub 渲染。
- **更新内容时** 同步更新 `glossary/terms.md`（如有新概念）。
- **client_id 在文档中使用占位符**（如 `<client_id>` / `<client-a>`），而非真实生产中的门店简称，避免泄露客户标识。
- **生产 IP / 凭证** 永不入文档；所有示例用 `127.0.0.1` / `<host>` / `<password>`。
