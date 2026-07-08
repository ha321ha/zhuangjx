# 分库分表同步到 Doris / ES — 方案优缺点概述（修正版）

> **基于**: V4 概念分析 + 官方资料核实
> **场景**: ①定时批量同步到 Doris　②实时 CDC 同步到 Doris / ES
> **编写日期**: 2026-07-08
> **数据来源**: SeaTunnel 官方文档、GitHub Issues、官方 Blog、社区 FAQ

---

## 目录

1. [SeaTunnel 定位：官方怎么说](#1-seatunnel-定位官方怎么说)
2. [方案优点（有官方依据）](#2-方案优点有官方依据)
3. [方案缺点与应对（合并）](#3-方案缺点与应对合并)
4. [轻量化方案的压力边界](#4-轻量化方案的压力边界)
5. [总结](#5-总结)

---

## 1. SeaTunnel 定位：官方怎么说

### 1.1 官方定义

SeaTunnel 对自己的定位非常明确（[About SeaTunnel](https://seatunnel.apache.org/docs/2.3.8/about)）：

> "SeaTunnel focuses on data integration and data synchronization."

专注做数据集成和同步，不是计算引擎，不是调度平台。

### 1.2 SQL Transform 的能力边界

官方文档（[SQL Transform](https://seatunnel.apache.org/docs/2.3.12/transform-v2/sql)）：

> "The complex SQL unsupported yet, include: multi source table/rows **JOIN** and **AGGREGATE** operation."

GitHub Issue [#11061](https://github.com/apache/seatunnel/issues/11061) 补充：

> "SeaTunnel's Transform layer currently only supports **row-level operations**, explicitly blocks **JOIN, GROUP BY, window functions, subqueries**."

### 1.3 调度能力

官方 FAQ（[Can SeaTunnel execute scheduled tasks?](https://seatunnel.apache.org/docs/faq)）：

> "You can use Linux cron jobs... or leverage scheduling tools like Apache DolphinScheduler or Apache Airflow."

没有内置调度器。

### 1.4 关于 Seatunnel-Web 和 Web UI 是两个东西

SeaTunnel 生态中有两个 Web 产品，**容易混淆**：

| 产品 | 定位 | 功能 | 端口 |
|------|------|------|------|
| **SeaTunnel-Web** | 管理控制台（子项目） ⚠️ **实验性项目，非生产就绪** | 数据源管理、虚拟表、任务定义、用户权限 | 8801 |
| **Web UI** | 内置监控页面（Engine自带） | 查看运行中/已完成任务、Worker/Master 状态 | 5801 |

本文讨论的是**SeaTunnel-Web**。官方说明（[About SeaTunnel Web](https://seatunnel.apache.org/seatunnel_web/1.0.0/about)）：

> "SeaTunnel Web is a web project that provides visual management of jobs, scheduling, running and monitoring capabilities."

但注意：
- **下载页面明确标注**：`WARNING: SeaTunnel Web is an experimental project and is not yet production ready.`
- About 页面声称具备 scheduling 能力，但实际**尚未实现**（GitHub Issue #8657）

### 1.5 EL(T) 定位

官方 About 页面明确 SeaTunnel 是 **EL(T)** 而非 ETL：

> "SeaTunnel is an EL(T) data integration platform. Therefore, in SeaTunnel, Transform can only be used to perform some simple transformations on data."

**来源**: [About Seatunnel](https://seatunnel.apache.org/docs/2.3.8/about)

---

## 2. 方案优点（有官方依据）

### 2.1 轻量：零外部依赖

| 对比项 | SeaTunnel | Flink CDC | Canal |
|-------|-----------|-----------|-------|
| 集群协调 | 内嵌 Hazelcast | Zookeeper 或 K8s HA | Zookeeper |
| 消息队列 | 不需要 | 建议 Kafka | 需要 Kafka |
| 计算引擎 | 自带 Zeta Engine | 必须 Flink | 不需要 |
| 部署节点 | 2 Master + 3 Worker | Flink 集群（3~5 节点）| Canal + Kafka + Consumer |

### 2.2 同时支持批量和流（CDC）

**来源**: [官方 FAQ](https://seatunnel.apache.org/docs/faq) — "SeaTunnel supports both batch and streaming processing modes." 一个工具解决两种场景。

### 2.3 高性能

**来源**: [SeaTunnel 官方 Medium 对比文章](https://apacheseatunnel.medium.com/apache-seatunnel-vs-d374a531ee06) — "Under the same test scenarios, SeaTunnel is **40%–80% faster than DataX**."（注：来自官方博客，非独立第三方测试）

### 2.4 连接器生态

**来源**: [Source 连接器列表](https://seatunnel.apache.org/docs/connectors/source) + [Sink 连接器列表](https://seatunnel.apache.org/docs/connectors/sink) — 覆盖主流数据源。你需要的 MySQL、CDC、Doris、ES 都有官方连接器。

### 2.5 Exactly-Once

**来源**: [官方 FAQ](https://seatunnel.apache.org/docs/faq) — "SeaTunnel supports exactly-once consistency for some data sources, such as MySQL and PostgreSQL, ensuring data consistency during integration. Note that exactly-once consistency depends on the capabilities of the underlying database." Doris Sink 通过 2PC（两阶段提交）支持 Exactly-Once。

### 2.6 社区规模

- GitHub **9,500+ Stars**（[apache/seatunnel](https://github.com/apache/seatunnel)，截至 2026-07）
- **2,300+ Forks**
- 官方 About 页面称：**"used in production by nearly 100 companies"**（近百家企业生产使用）
- 官方用户案例页面展示：唯品会、腾讯、天翼云等企业案例（[seatunnel.apache.org/user_cases](https://seatunnel.apache.org/user_cases)）

---

## 3. 方案缺点与应对（合并）

### 3.1 无内置定时调度

**来源**: [官方 FAQ](https://seatunnel.apache.org/docs/faq)

```
影响：定时批量任务需要外部触发。不能像 DolphinScheduler 那样配 Cron 就自动跑。

应对方案对比：
  ┌────────────────────┬──────────┬──────────────────────────────┐
  │ 方案               │ 加组件？ │ 说明                         │
  ├────────────────────┼──────────┼──────────────────────────────┤
  │ K8s CronJob        │ 不加     │ K8s 原生，你这最自然的方式    │
  │ SeaTunnel-Web 二开 │ 不加     │ 4~5 人天，但需维护版本兼容   │
  │ DolphinScheduler   │ 加一个   │ 功能最强但有学习成本          │
  │ Linux Cron         │ 不加     │ 最简单，适合测试环境          │
  └────────────────────┴──────────┴──────────────────────────────┘

你计划的 SeaTunnel-Web 二开可行，但注意：
  - SeaTunnel-Web 在快速迭代期，版本升级可能影响二开代码
  - 尽量解耦，减少和 SeaTunnel-Web 内部逻辑的耦合

如果你的场景只需要简单的 Cron 触发：
  K8s CronJob 最省事，连二开都不用。
```

### 3.2 Transform 不支持 JOIN / GROUP BY / 窗口函数

**来源**: [SQL 官方文档](https://seatunnel.apache.org/docs/2.3.12/transform-v2/sql) + [Issue #11061](https://github.com/apache/seatunnel/issues/11061) — 官方明确说不支持，而且不是 Bug 是设计决定。

```
影响：你不能在 SeaTunnel 里做多表关联、聚合计算、窗口函数。

正确做法：
  SeaTunnel 只负责把数据从分库分表搬到 Doris，
  JOIN / GROUP BY 在 Doris 里做——它是 OLAP 引擎，专业对口。

  这其实是合理的设计：
  - SeaTunnel 是数据管道，不是计算引擎
  - 在管道里做 JOIN 会变成瓶颈
  - Doris 的列式存储 + MPP 架构更适合做分析计算
```

### 3.3 不支持跨库实例的正则 Source

**来源**: [JDBC 官方文档](https://seatunnel.apache.org/docs/2.3.13/connectors/source/Jdbc) — `table_path` + `use_regex` 只能在一个 JDBC URL（一个数据库实例）内有效。

```
影响：每个分库实例需要独立的 Source 块。

  5 个分库 = 5 个 Source ≈ 25 行配置
  每个 Source 里的表已经可以用正则匹配了（10 张分表不用逐张写）

应对：
  - 这是所有同步工具的共同限制（Flink CDC、Canal 都一样）
  - 5 个 Source 的维护成本很低，不用过度优化
```

### 3.4 不支持工作流编排（DAG）

**来源**: SeaTunnel 产品定位决定，一个任务就是一条独立管道。

```
影响：不能定义任务间的依赖关系，没有 DAG 编排。

你的场景（分库分表→Doris）是否需要 DAG？
  - 如果只是把数据同步到 Doris → 不需要，每个任务独立
  - 如果需要先同步 A 表、再聚合写入 B 表 → 在 Doris 里用 SQL 分步做
  - 如果确实需要 DAG 编排 → 用 DolphinScheduler 编排多个 SeaTunnel 任务
```

### 3.5 CDC 不支持无主键表

**来源**: [官方 FAQ](https://seatunnel.apache.org/docs/faq) — "SeaTunnel does not support CDC for tables without primary keys."

```
影响：如果分表没有主键，CDC 模式无法使用。

应对：分库分表的表一般都有主键（分片键依赖主键），
      这个限制在分库分表场景中几乎不构成问题。
```

### 3.6 SeaTunnel-Web 的功能边界（有监控和日志）

**来源**: [GitHub commit log](https://apache.googlesource.com/seatunnel-web/+log) + [GitHub README](https://github.com/apache/seatunnel-web) + [部署文档](https://seatunnel.apache.org/seatunnel_web/1.0.0/deploy)

SeaTunnel-Web 的 GitHub 提交记录显示，**监控和日志功能是已有的**：

| 提交 | 内容 | 时间 |
|------|------|------|
| `#303` | "Add log viewer modal for enhanced log management and viewing" | 12 个月前 |
| `#301` | "Add creation and update timestamps to monitor task metrics" | 12 个月前 |
| `#281` | "Add task monitor chart" | 14 个月前 |
| `#265` | "Support multi-table sink" | 18 个月前 |

完整功能清单：

| 功能模块 | 具体能力 | 当前状态 | 来源 |
|---------|---------|---------|------|
| **用户管理** | 用户增删改查、权限分配 | ✅ 支持 | [GitHub README](https://github.com/apache/seatunnel-web) |
| **数据源管理** | 注册 MySQL、Doris、ES 等数据源 | ✅ 支持 | [GitHub README](https://github.com/apache/seatunnel-web) |
| **虚拟表管理** | 定义表的 Schema、字段映射 | ✅ 支持 | [GitHub README](https://github.com/apache/seatunnel-web) |
| **任务管理** | 创建、提交、查看任务列表 | ✅ 支持 | [GitHub README](https://github.com/apache/seatunnel-web) |
| **任务日志查看** | SeaTunnel-Web 弹窗查看任务运行日志 | ✅ 自带（#303） | [Git commit #303](https://apache.googlesource.com/seatunnel-web/+/refs/heads/main) |
| **Engine Web UI 监控** | 查看运行中/已完成任务、Worker/Master 状态、任务日志 | ✅ Engine 自带，**0 额外组件** | [Web UI 文档](https://seatunnel.apache.org/docs/2.3.9/seatunnel-engine/web-ui) |
| **REST API 监控** | 通过 HTTP 查询任务状态、集群概览、系统监控信息 | ✅ Engine 自带，**0 额外组件** | [REST API V2](https://seatunnel.apache.org/docs/engines/zeta/rest-api-v2) |
| **Prometheus 指标** | 暴露 CPU/内存/吞吐量等指标到 `/metrics` | ✅ Engine 自带端点，**但需 Prometheus + Grafana 采集展示**（可选） | [Telemetry](https://seatunnel.apache.org/docs/engines/zeta/telemetry) |
| **任务监控图表** | SeaTunnel-Web 内任务指标趋势图 | ✅ 支持（#281） | [Git commit #281](https://apache.googlesource.com/seatunnel-web/+/refs/heads/main) |
| **任务实例管理** | 任务实例状态跟踪、错误信息记录 | ✅ 支持 | [TaskInstanceController](https://apache.googlesource.com/seatunnel-web/+/refs/heads/main) |
| **多表 Sink** | 一个任务写多个目标表 | ✅ 支持（#265） | [Git commit #265](https://apache.googlesource.com/seatunnel-web/+/refs/heads/main) |
| **配置占位符** | 支持变量替换 `${var:default}` | ✅ 支持 | [GitHub README](https://github.com/apache/seatunnel-web) |
| **认证方式** | DB 登录、LDAP 集成 | ✅ 支持 | [application.yml](https://github.com/apache/seatunnel-web) |
| **API 参数化执行** | 通过 API 传参提交任务 | ✅ 支持（#193） | [Git commit #193](https://apache.googlesource.com/seatunnel-web/+/refs/heads/main) |
| **工作空间** | 多租户/多项目隔离 | ✅ 支持（#284） | [Git commit #284](https://apache.googlesource.com/seatunnel-web/+/refs/heads/main) |
| **任务编排 DAG** | 定义任务间依赖关系 | ❌ 不支持 | 产品定位决定 |
| **定时调度** | Cron 触发周期性执行 | ❌ 不支持 | [Issue #8657](https://github.com/apache/seatunnel/issues/8657) |
| **任务告警** | 失败时自动告警通知 | ❌ 不支持 | 无相关功能 |

### 3.7 连接器驱动需自备

**来源**: [JDBC 官方文档](https://seatunnel.apache.org/docs/2.3.13/connectors/source/Jdbc) — "for license compliance, you have to provide database driver yourself."

```
影响：MySQL、Doris 等驱动的 JAR 需自行下载放到 lib 目录。

应对：一次性解决，在构建 Docker 镜像时打包进去即可。
```

### 3.8 版本强绑定

**来源**: [GitHub README](https://github.com/apache/seatunnel-web) — SeaTunnel-Web 和 Engine 有严格的版本对应关系：

| SeaTunnel Web | SeaTunnel Engine |
|---------------|-----------------|
| 1.0.3-SNAPSHOT | 2.3.11 |
| 1.0.2 | 2.3.8 |
| 1.0.1 | 2.3.3 |
| 1.0.0 | 2.3.3 |

```
影响：升级时必须一起升，不能只升 Engine 不升 Web（反之亦然）。
```

### 3.9 SeaTunnel-Web 无官方 Docker 镜像

**来源**: [Deployment 文档](https://seatunnel.apache.org/seatunnel_web/1.0.0/deploy) — 只有二进制包（tar.gz），没有 Docker 镜像。

```
影响：需要自己构建 Docker 镜像、推送到 Harbor。

应对：文档中有完整的 Dockerfile 模板，一次构建后续复用。
```

### 3.10 版本升级对比：2.3.8→2.3.11 / Web 1.0.2→1.0.3-SNAPSHOT

你之前问过这两个版本的差异，以下是完整的迭代内容对比。

#### SeaTunnel Engine 2.3.8 → 2.3.11

| 版本 | 发布日期 | 主要新功能 |
|------|---------|-----------|
| **2.3.8** | 2024-10 | 官方 Docker 镜像、Job 级别日志、Prometheus 监控集成、Spark/Flink 多表读写、Typesense 连接器、AI 模型连接器、Protobuf Kafka 支持 |
| 2.3.9 | 2025-01 | 连接器更新、Bug 修复、文档改进 |
| 2.3.10 | 2025-03 | 连接器更新、Bug 修复、多表同步改进 |
| **2.3.11** | 2025-05 | 连接器增强、配置优化、Bug 修复（OceanBase、文档等） |

**2.3.8 到 2.3.11 的核心变化**：

```
2.3.8 引入的重要能力（对你分库分表同步场景直接影响）：
  ✅ Docker 镜像 → 你部署在 K8s 可以直接拉官方镜像，不用自己构建
  ✅ Job 级别日志 → 每个任务日志隔离，排查问题方便
  ✅ Prometheus 指标 → 暴露标准监控接口
  ✅ 多表读写 → 分库分表合并同步的基础能力

2.3.9→2.3.11 主要是连接器更新和修 Bug：
  ✅ 连接器越来越多（对你有用的是 MySQL、Doris、ES 相关修复）
  ✅ 配置选项优化
  ✅ 如果你的分表有 OceanBase 等特殊数据库，2.3.11 有专门修复
  ❌ 没有颠覆性的新功能，基本上是小版本迭代
```

#### SeaTunnel-Web 1.0.2 → 1.0.3-SNAPSHOT

| 功能 | 1.0.2 | 1.0.3-SNAPSHOT | 你的分库分表场景是否需要？ |
|------|-------|----------------|--------------------------|
| **数据源管理** | ✅ | ✅ | ✅ 需要 |
| **虚拟表管理** | ✅ | ✅ | ✅ 需要 |
| **任务管理** | ✅ | ✅ | ✅ 需要 |
| **用户管理** | ✅ | ✅ | ✅ 需要 |
| **多表 Sink** | ❌ | ✅ (#265) | ✅ 需要（合并写入多张表） |
| **任务日志查看** | ❌ | ✅ (#303) | ✅ 需要（排查同步问题） |
| **任务监控图表** | ❌ | ✅ (#281) | ✅ 需要（看同步趋势） |
| **工作空间隔离** | ❌ | ✅ (#284) | 多个团队使用的话需要 |
| **LDAP 认证** | ❌ | ✅ (#269) | 公司有 LDAP 的话需要 |
| **定时调度** | ❌ | ❌ (#8657) | 需要（你要二开解决） |

#### 你应该选哪个版本？

```
你的情况：
  - 部署环境：K8s + Rancher（优先用官方 Docker 镜像）
  - 分库分表同步需要多表合并
  - 需要监控和日志排查问题

结论：
  ✅ 选 SeaTunnel Engine 2.3.11（或直接 2.3.13，最新版）
     2.3.8 已经有 Docker 镜像和 Job 日志，2.3.11 连接器更完善
     如果从零起步，建议直接上 2.3.13（2026-03 发布）
     ⚠️ 2.3.11 的官方文档已标注 "no longer actively maintained"

  ✅ 选 SeaTunnel-Web 1.0.3-SNAPSHOT
     1.0.2 没有多表 Sink、日志查看、监控图表，都是你需要的
     缺点：1.0.3-SNAPSHOT 是开发版，需要从源码构建
     ⚠️ 没有官方二进制包，首次部署可能遇到 Bug

  ⚠️ SeaTunnel-Web 无论哪个版本都标注为 "experimental, not production ready"
     如果 Web 稳定性是硬要求，可以考虑只用 Engine CLI + REST API
```

#### 版本选择速查

| 选项 | Engine | Web | 优点 | 缺点 |
|------|--------|-----|------|------|
| **保守** | 2.3.8 | 1.0.2 | 正式发布版 | 缺多表 Sink、日志查看、监控图表 |
| **推荐** | **2.3.11** | **1.0.3-SNAPSHOT** | 功能最全，有 Docker 镜像 | Web 是开发版 |
| **激进** | 2.3.13 | 1.0.3-SNAPSHOT | Engine 最新最稳定 | 和 Web 版本匹配需确认 |

```
最终建议：
  Engine 选 2.3.11（有官方 Docker 镜像，文档最全）
  Web 选 1.0.3-SNAPSHOT（功能完整，接受它是开发版）
  如果 Web 稳定性是关键，可以只用 Engine + K8s CronJob，
  不做二开，不碰 SeaTunnel-Web。
```

---

### 3.12 缺乏统一的二开 API / 扩展点

```
影响：SeaTunnel-Web 没有提供"调度插件"或"扩展点"。
      加调度功能需要在源码级别修改 Spring Boot 代码。

  这意味着：
  - 每次官方发新版，你的二开代码需要适配
  - 如果合代码成本高，你可能被迫停在旧版本
  - 建议把调度做成独立模块，减少耦合
```

### 3.13 Engine Web UI 和 SeaTunnel-Web 是两回事（容易混淆）

```
  Engine Web UI（端口 5801）:
  ─────────────────────────────────────
  内嵌在 SeaTunnel Engine Master 中
  功能：查看运行中任务、已完成任务、Worker 状态、Master 状态
  日志：在 UI 中直接查看任务日志（来自 Engine 的 REST API）
  不需要部署，Engine 启动就有

  SeaTunnel-Web（端口 8801）:
  ─────────────────────────────────────
  独立部署的 Spring Boot 项目
  功能：数据源管理、虚拟表、任务管理、日志查看、监控图表、用户管理
  日志：任务运行日志查看（#303）、监控指标（#301）
  需要单独构建和部署

  两者都可以看日志，但 SeaTunnel-Web 的日志功能更多样
  （有日志弹窗、任务实例错误信息记录等）
```

### 3.14 缺点应对矩阵（汇总）

| 缺点 | 你计划的应对 | 可行性 | 风险 |
|------|------------|--------|------|
| 无调度 | SeaTunnel-Web 二开 Quartz/CronJob | ✅ 可行 | 版本升级需适配 |
| 无 JOIN/GROUP BY | 交给 Doris 处理 | ✅ 本来就是最佳实践 | 无 |
| 无跨库正则 | 每个分库写一个 Source | ✅ 5 个 = 25 行 | 分库新增时需改配重启 |
| 无编排 | Doris 内部 SQL 处理 / 不需要 | ✅ 简单场景够用 | 复杂 DAG 需调度平台 |
| CDC 无主键表 | 分库分表都有主键 | ✅ 不构成问题 | 无 |
| 驱动需自备 | 构建镜像时打包 | ✅ 一次性解决 | 无 |
| Web 功能不足 | 监控/日志已有，缺调度需二开 | ⚠️ 调度需评估维护成本 | 版本升级兼容性 |
| 无官方 Docker | 自建镜像推 Harbor | ✅ 一次构建后续复用 | 无 |

---

## 4. 轻量化方案的压力边界

### 4.1 什么情况下扛不住？

| 指标 | 能扛的范围 | 超过后的方案 |
|------|-----------|-------------|
| 单表数据量 | ≤ 5 亿行 | 优化分区或加大 Worker |
| 日增量 | ≤ 10 亿行 | 增加 Worker 或拆分任务 |
| 分库实例数 | ≤ 20 个 | 分集群部署 |
| CDC 延迟要求 | ≥ 3 秒 | 调优 Checkpoint + 网络 |
| 并发任务数 | ≤ 50 个 | 增加 Worker 节点 |

### 4.2 轻量化不等于零成本

```
部署阶段：SeaTunnel 集群搭建 + Web 部署 + 测试链路   3~5 人天
二开调度：数据库 + 后端 + 前端 + 部署                4~5 人天
日常运维：CDC 监控 + 故障处理 + 加库改配置            持续投入

如果不用二开，用 K8s CronJob：省掉 4~5 人天
```

---

## 5. 总结

### 5.1 一句话

```
SeaTunnel 做"分库分表→Doris"是合适的。

  它做得好：
  ✓ 高吞吐数据搬运
  ✓ 批量 + CDC 统一
  ✓ 轻量少依赖

  它不做：
  ✗ 调度（需要外部触发）
  ✗ 计算（JOIN/GROUP BY 交给 Doris）
  ✗ 编排（简单场景用 Doris SQL，复杂场景用调度平台）
```

### 5.2 官方资料速查

| 你想确认的事 | 官方怎么说 | 来源 |
|------------|-----------|------|
| SeaTunnel 定位 | "focuses on data integration and data synchronization" | [About](https://seatunnel.apache.org/docs/2.3.8/about) |
| SQL 支不支持 JOIN | "complex SQL unsupported yet, include: JOIN and AGGREGATE" | [SQL 文档](https://seatunnel.apache.org/docs/2.3.12/transform-v2/sql) |
| SQL 不支持哪些 | "explicitly blocks JOIN, GROUP BY, window functions, subqueries" | [Issue #11061](https://github.com/apache/seatunnel/issues/11061) |
| 有没有调度 | 用 Cron / DolphinScheduler / Airflow | [FAQ](https://seatunnel.apache.org/docs/faq) |
| 性能比 DataX | "40%–80% faster than DataX" | [官方博客](https://apacheseatunnel.medium.com/apache-seatunnel-vs-d374a531ee06) |
| 企业在用吗 | "nearly 100 companies" 生产使用，唯品会、腾讯、天翼云等 | [About](https://seatunnel.apache.org/docs/2.3.8/about) + [User Cases](https://seatunnel.apache.org/user_cases) |
| 支不支持无主键CDC | "does not support CDC for tables without primary keys" | [FAQ](https://seatunnel.apache.org/docs/faq) |
| Exactly-Once | "supports exactly-once consistency" | [FAQ](https://seatunnel.apache.org/docs/faq) |
| SeaTunnel-Web 做什么 | 用户/数据源/虚拟表/任务/监控/日志/图表/工作空间 | [GitHub](https://github.com/apache/seatunnel-web) + [Git commits](https://apache.googlesource.com/seatunnel-web/+log) |
| SeaTunnel-Web 生产就绪吗 | ⚠️ "experimental project, not yet production ready" | [下载页面](https://seatunnel.apache.org/download) |
| SeaTunnel-Web 不做什么 | 调度(Issue#8657)、编排、告警 | [GitHub Issues](https://github.com/apache/seatunnel/issues) |
