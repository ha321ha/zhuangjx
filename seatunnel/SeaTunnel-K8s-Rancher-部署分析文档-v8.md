# SeaTunnel + SeaTunnel-Web 生产部署文档（Rancher 版）

> **版本**: SeaTunnel 2.3.11 / SeaTunnel-Web 1.0.3-SNAPSHOT
> **场景**: MySQL ↔ Doris 数据同步
> **日期**: 2026-07-08

---

## 目录

- [前置准备](#前置准备)
- [第一章：SeaTunnel Engine 部署](#第一章seatunnel-engine-部署)
- [第二章：SeaTunnel-Web 部署](#第二章seatunnel-web-部署)
- [验证](#验证)
- [排错](#排错)

---

## 前置准备

### 环境要求

| 资源                                    | 说明                                 |
| ------------------------------------- | ---------------------------------- |
| K8s 集群（Rancher 管理）                    | ≥ 1.23                             |
| Harbor 镜像仓库                           | 存放自建镜像。你的平台可能已经集成了 Harbor 拉取能力，见下文 |
| MySQL 5.7+/8.0+                       | SeaTunnel-Web 元数据库                 |
| **一台安装了 Podman Desktop 的 Windows 机器** | 构建镜像用，只用一次，不需要安装 Docker            |

---

# 第一章：SeaTunnel Engine 部署

## 1.1 架构一句话

SeaTunnel Engine 采用 Master-Worker 分离架构：

- **Master**（2 个）：负责任务调度、REST API。一主一备做 HA
- **Worker**（3 个）：实际干活，执行数据同步任务。可以随时扩容
- 节点间通过内嵌 Hazelcast 自动组建集群，**不需要额外安装 ZooKeeper**

## 1.2 构建 Engine 镜像

### 为什么要自己构建？

官方 `apache/seatunnel` 镜像**只包含 Engine 本身**，不包含任何连接器（connector-jdbc、connector-doris 等）和 JDBC 驱动（mysql-connector-j 等）。这两样东西需要自己加进去。

### 镜像里加了什么

构建这个镜像就是在官方基础上加了 3 样东西：

```
官方镜像 apache/seatunnel
  + connector-jdbc-2.3.11.jar          ← JDBC 通用连接器（MySQL、PostgreSQL、Oracle 等都可以用）
  + connector-doris-2.3.11.jar         ← Doris 专用连接器（Stream Load 方式写入）
  + connector-cdc-mysql-2.3.11.jar     ← MySQL CDC 连接器（增量同步用）
  + mysql-connector-j-8.0.33.jar       ← MySQL JDBC 驱动（放 lib/ 目录，Doris 也用它）
  ────────────────────────────────────────────
  = 自建镜像 harbor.xxx/seatunnel/seatunnel-engine:2.3.11
```

每个 jar 都是从 Maven 中央仓库直接下载的，不需要编译源码。

### 操作步骤

构建目录和 Dockerfile 的准备与操作系统无关，在 Windows 终端（cmd / PowerShell / Git Bash 均可）中执行：

```bash
mkdir seatunnel-engine-build
cd seatunnel-engine-build
```

创建文件 `Dockerfile`，内容如下。每行都有注释说明"这步在干什么"：

```dockerfile
# ── 基础镜像：JDK 11（SeaTunnel 官方推荐）──
FROM openjdk:11-jdk-slim

ARG VERSION=2.3.11
ENV SEATUNNEL_HOME=/opt/seatunnel

# ── 第 1 步：下载 SeaTunnel 二进制包并解压到 /opt/seatunnel ──
RUN apt-get update && apt-get install -y wget && rm -rf /var/lib/apt/lists/* && \
    wget -q https://dlcdn.apache.org/seatunnel/${VERSION}/apache-seatunnel-${VERSION}-bin.tar.gz \
      -O /tmp/seatunnel.tar.gz && \
    mkdir -p ${SEATUNNEL_HOME} && \
    tar -xzf /tmp/seatunnel.tar.gz -C /tmp/ && \
    mv /tmp/apache-seatunnel-${VERSION}/* ${SEATUNNEL_HOME}/ && \
    rm -rf /tmp/*

# ── 第 2 步：安装连接器（从 Maven 中央仓库下载 jar）──
# 连接器放在 /opt/seatunnel/connectors/ 下，Engine 启动时自动加载
RUN mkdir -p ${SEATUNNEL_HOME}/connectors && \
    wget -q https://repo1.maven.org/maven2/org/apache/seatunnel/connector-jdbc/${VERSION}/connector-jdbc-${VERSION}.jar \
      -O ${SEATUNNEL_HOME}/connectors/connector-jdbc-${VERSION}.jar && \
    wget -q https://repo1.maven.org/maven2/org/apache/seatunnel/connector-doris/${VERSION}/connector-doris-${VERSION}.jar \
      -O ${SEATUNNEL_HOME}/connectors/connector-doris-${VERSION}.jar && \
    wget -q https://repo1.maven.org/maven2/org/apache/seatunnel/connector-cdc-mysql/${VERSION}/connector-cdc-mysql-${VERSION}.jar \
      -O ${SEATUNNEL_HOME}/connectors/connector-cdc-mysql-${VERSION}.jar

# ── 第 3 步：安装 JDBC 驱动（放 lib/ 目录，不是 connectors/）──
# 注意：JDBC 驱动必须放 lib/，放 connectors/ 不会被加载
# MySQL 驱动同时也是 Doris 驱动，因为 Doris 兼容 MySQL 协议
RUN wget -q https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar \
      -O ${SEATUNNEL_HOME}/lib/mysql-connector-j-8.0.33.jar

WORKDIR ${SEATUNNEL_HOME}
```

> **关于 Dockerfile 在 Windows 上的注意事项**：Dockerfile 本身是跨平台的纯文本文件，用记事本、VS Code 等任意编辑器创建即可，不需要任何特殊环境。Podman Desktop 会处理内部的 Linux 构建逻辑。

### 使用 Podman Desktop 构建并推送

**第一步：构建镜像**

```bash
cd seatunnel-engine-build
docker build -t reg.meicloud.com:40443/ccs-std/prd/seatunnel-engine:2.3.11 .
```

Podman 输出示例（首次构建需下载基础镜像，约 1-2 分钟）：

```
STEP 1/5: FROM openjdk:11-jdk-slim
STEP 2/5: ARG VERSION=2.3.11
...
STEP 5/5: WORKDIR /opt/seatunnel
COMMIT harbor.你的公司.com/seatunnel/seatunnel-engine:2.3.11
--> 8a2b3c4d5e6f
Successfully tagged ...
```

**第二步：推送镜像到 Harbor**

```
# 导出镜像
docker save -o seatunnel-engine-2.3.11.tar \
  reg.meicloud.com:40443/ccs-std/prd/seatunnel-engine:2.3.11

# 压缩（减少传输体积，1.73GB → 约 400~700MB）
gzip seatunnel-engine-2.3.11.tar


# 在目标机器上解压
gunzip seatunnel-engine-2.3.11.tar.gz
```





**第三步：推送镜像到 Harbor**

```bash
# 在 Windows 终端中执行（cmd / PowerShell / Git Bash 均可）
podman login harbor.你的公司.com
# 提示输入 Harbor 用户名和密码

podman push harbor.你的公司.com/seatunnel/seatunnel-engine:2.3.11
```

**验证镜像内容**（在 Windows 上执行）：

```bash
podman run --rm harbor.你的公司.com/seatunnel/seatunnel-engine:2.3.11 ls /opt/seatunnel/connectors/
# 应该看到 3 个 jar：connector-jdbc-2.3.11.jar、connector-doris-2.3.11.jar、connector-cdc-mysql-2.3.11.jar
```

### Podman Desktop 常见问题

| 问题                              | 解决方法                                                                                  |
| ------------------------------- | ------------------------------------------------------------------------------------- |
| `podman` 命令找不到                  | 确保 Podman Desktop 已启动（系统托盘有企鹅图标），重启终端                                                 |
| 构建时磁盘空间不足                       | Podman 默认镜像存储在 `C:\Users\xxx\AppData\Local\containers`，清理旧镜像：`podman system prune -a` |
| 构建速度慢                           | 首次构建需下载基础镜像，后续构建会利用缓存。如果公司网络限制，挂代理或使用镜像加速器                                            |
| Harbor 登录报错 `x509: certificate` | 让运维确认 Harbor 证书可信，或在 Podman Desktop Settings → Registries 中添加 insecure registry       |

**结论**：除非你要修改 SeaTunnel 源码（比如改某个连接器的逻辑），否则直接 `wget` 下载二进制包 + 下载连接器 jar，比编译源码快得多（几分钟 vs 半小时以上）。

## 1.3 在 Rancher 中部署 Engine

### 前提确认

部署前确认已在 Rancher 中切换到**目标命名空间**（你的现有命名空间）：

```
集群 → Projects/Namespaces → 找到你的命名空间并选中
→（后续操作均在该命名空间下进行）
```

### 用 Helm 安装

```
Apps → Charts → 右上角 "Install Helm"
```

填写：

| 字段                  | 值                                                  |
| ------------------- | -------------------------------------------------- |
| Name                | `seatunnel`                                        |
| Namespace           | `$(NAMESPACE)` — 选你现有的命名空间                         |
| Create namespace    | **取消勾选**（命名空间已经存在）                                 |
| Helm Repository URL | `oci://registry-1.docker.io/apache/seatunnel-helm` |
| Version             | `2.3.11`                                           |

点击 Next，在 YAML 编辑器中粘贴：

```yaml
# 使用 Harbor 中的自定义镜像
image:
  repository: harbor.你的公司.com/seatunnel/seatunnel-engine
  tag: "2.3.11"
  pullPolicy: Always

master:
  replicas: 2
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"

worker:
  replicas: 3
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"

# 如果平台已经集成了 Harbor，不需要 imagePullSecrets，删掉或注释掉
# imagePullSecrets:
#   - name: harbor-secret

seatunnel:
  config:
    seatunnel.yaml: |
      seatunnel:
        engine:
          history-job-expire-minutes: 1440
          backup-count: 1
          queue-type: blockingqueue
          classloader-cache-mode: true
          slot-service:
            dynamic-slot: true
          checkpoint:
            interval: 300000
            timeout: 60000
            storage:
              type: hdfs
              max-retained: 3
              plugin-config:
                storage.type: hdfs
                fs.defaultFS: file:///tmp/seatunnel/checkpoint/
                namespace: /tmp/seatunnel/checkpoint/

ingress:
  enabled: false
```

点击 Install。Helm Chart 会在你的目标命名空间中创建 `seatunnel-master` 和 `seatunnel-worker` 两个 StatefulSet。

> **关于 imagePullSecrets**：如果你的平台（如海豚）已经集成了 Harbor 拉取能力，Pod 不需要额外配置 Secret 就能拉镜像，那就把 `imagePullSecrets` 那几行删掉。不确定的话先不删，如果 Pod 能正常启动（不报 ImagePullBackOff），下次可以删掉。

### 验证部署

```
Workloads → StatefulSet →
  ├─ seatunnel-master: 2/2 Running  ✓
  └─ seatunnel-worker: 3/3 Running  ✓
```

点 `seatunnel-master-0` → View Logs，搜 `Members`，应该看到：

```
Members {size:5, ver:3} [...]
```

`size:5` 表示 2 Master + 3 Worker 全部加入集群。

点 `seatunnel-worker-0` → Execute Shell，验证连接器：

```bash
ls /opt/seatunnel/connectors/
# 应有：connector-jdbc-2.3.11.jar, connector-doris-2.3.11.jar, connector-cdc-mysql-2.3.11.jar

ls /opt/seatunnel/lib/
# 应有：mysql-connector-j-8.0.33.jar
```

---

# 第二章：SeaTunnel-Web 部署

## 2.1 Web 是什么

SeaTunnel-Web 是一个 Spring Boot 项目，提供浏览器界面来管理数据源、配置任务、查看运行状态。它需要：

- **MySQL**：存元数据
- **Engine 客户端**：通过 Hazelcast 跟 Engine 集群通信

## 2.2 准备 MySQL

### 建库

在 MySQL 中执行：

```sql
CREATE DATABASE IF NOT EXISTS seatunnel CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER IF NOT EXISTS 'seatunnel'@'%' IDENTIFIED BY '你的密码';
GRANT ALL PRIVILEGES ON seatunnel.* TO 'seatunnel'@'%';
FLUSH PRIVILEGES;
```

### 导入表结构

建表 SQL 在 SeaTunnel-Web 源码中：`seatunnel-server/seatunnel-app/src/main/resources/script/seatunnel_server_mysql.sql`

```bash
# 下载建表 SQL（不需要下载整个源码）
wget https://raw.githubusercontent.com/apache/seatunnel-web/1.0.3-SNAPSHOT/seatunnel-server/seatunnel-app/src/main/resources/script/seatunnel_server_mysql.sql

# 导入
mysql -h 你的MySQL地址 -u seatunnel -p -D seatunnel < seatunnel_server_mysql.sql

# 验证
mysql -h 你的MySQL地址 -u seatunnel -p -D seatunnel -e "SHOW TABLES;"
# 应看到：user, datasource, job_definition, job_instance 等
```

### 在 K8s 中映射 MySQL 地址

如果 MySQL 在 K8s 外部，需要创建一个 Service 把它映射进来。

**Rancher 操作路径**：

```
Import YAML
```

```yaml
# 有域名用 ExternalName
apiVersion: v1
kind: Service
metadata:
  name: external-mysql
  namespace: $(NAMESPACE)                # 改为你的现有命名空间
spec:
  type: ExternalName
  externalName: mysql.你的公司.com    # 替换为你的 MySQL 域名
```

```yaml
# 只有 IP 用 Endpoints
apiVersion: v1
kind: Service
metadata:
  name: external-mysql
  namespace: $(NAMESPACE)                # 改为你的现有命名空间
spec:
  ports:
    - port: 3306
      targetPort: 3306
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-mysql
  namespace: $(NAMESPACE)                # 改为你的现有命名空间
subsets:
  - addresses:
      - ip: 192.168.1.100               # 替换为你的 MySQL IP
    ports:
      - port: 3306
```

创建之后，K8s 内部就可以用 `external-mysql.$(NAMESPACE).svc.cluster.local:3306` 访问你的 MySQL。

## 2.3 构建 Web 镜像

### 获取 Web 编译产物

需要先获得 `apache-seatunnel-web-1.0.3-SNAPSHOT.tar.gz`：

```bash
# 这一步可以在任意有 JDK 11 的机器上执行
git clone https://github.com/apache/seatunnel-web.git
cd seatunnel-web
git checkout 1.0.3-SNAPSHOT
sh build.sh code
# 产物：seatunnel-web-dist/target/apache-seatunnel-web-1.0.3-SNAPSHOT.tar.gz
cp seatunnel-web-dist/target/apache-seatunnel-web-1.0.3-SNAPSHOT.tar.gz /你的目录/
```

### Dockerfile

```dockerfile
FROM openjdk:11-jdk-slim

ARG VERSION=2.3.11
ARG ST_WEB_VERSION=1.0.3-SNAPSHOT

ENV SEATUNNEL_HOME=/opt/seatunnel
ENV ST_WEB_HOME=/opt/seatunnel-web

RUN apt-get update && apt-get install -y wget && rm -rf /var/lib/apt/lists/* && \
    mkdir -p ${SEATUNNEL_HOME} ${ST_WEB_HOME}/logs

# ── 第 1 步：SeaTunnel Engine（Web 需要用它的客户端跟 Engine 集群通信）──
RUN wget -q https://dlcdn.apache.org/seatunnel/${VERSION}/apache-seatunnel-${VERSION}-bin.tar.gz \
      -O /tmp/seatunnel.tar.gz && \
    tar -xzf /tmp/seatunnel.tar.gz -C /tmp/ && \
    mv /tmp/apache-seatunnel-${VERSION}/* ${SEATUNNEL_HOME}/ && \
    rm -rf /tmp/*

# ── 第 2 步：SeaTunnel-Web ──
COPY apache-seatunnel-web-${ST_WEB_VERSION}.tar.gz /tmp/
RUN tar -xzf /tmp/apache-seatunnel-web-${ST_WEB_VERSION}.tar.gz -C /tmp/ && \
    mv /tmp/apache-seatunnel-web-${ST_WEB_VERSION}/* ${ST_WEB_HOME}/ && \
    rm -rf /tmp/*

# ── 第 3 步：MySQL JDBC 驱动（Web 连接元数据库必须，没有它启动就报错）──
RUN mkdir -p ${ST_WEB_HOME}/libs && \
    wget -q https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar \
      -O ${ST_WEB_HOME}/libs/mysql-connector-j-8.0.33.jar

# ── 第 4 步：配置 Hazelcast Client（告诉 Web Engine 集群在哪）──
# DNS 规则：<pod名>.<headless service名>.<命名空间>.svc.cluster.local
RUN echo 'hazelcast-client:\n\
  cluster-name: seatunnel\n\
  properties:\n\
    hazelcast.logging.type: log4j2\n\
  network:\n\
    cluster-members:\n\
      - seatunnel-master-0.seatunnel-hl.$(NAMESPACE).svc.cluster.local:5801\n\
      - seatunnel-master-1.seatunnel-hl.$(NAMESPACE).svc.cluster.local:5801' \
  > ${ST_WEB_HOME}/conf/hazelcast-client.yaml

# ── 第 5 步：复制插件映射文件（让 Web 知道有哪些数据源可用）──
RUN if [ -f ${SEATUNNEL_HOME}/connectors/plugin-mapping.properties ]; then \
      cp ${SEATUNNEL_HOME}/connectors/plugin-mapping.properties ${ST_WEB_HOME}/conf/; \
    fi

# ── 第 6 步：配置 MySQL 连接信息 ──
# 密码写 CHANGE_ME 占位，部署时通过环境变量 DATASOURCE_PASSWORD 传入真实密码
RUN echo 'server:\n\
  port: 8801\n\
spring:\n\
  datasource:\n\
    url: jdbc:mysql://external-mysql.$(NAMESPACE).svc.cluster.local:3306/seatunnel?useSSL=false&useUnicode=true&characterEncoding=utf-8&allowMultiQueries=true&allowPublicKeyRetrieval=true\n\
    username: seatunnel\n\
    password: CHANGE_ME\n\
    driver-class-name: com.mysql.cj.jdbc.Driver\n\
  jackson:\n\
    date-format: yyyy-MM-dd HH:mm:ss\n\
    time-zone: GMT+8\n\
seatunnel:\n\
  engine:\n\
    home: /opt/seatunnel' \
  > ${ST_WEB_HOME}/conf/application.yml

EXPOSE 8801 7890

CMD ["sh", "-c", "${ST_WEB_HOME}/bin/seatunnel-backend-daemon.sh start && tail -f ${ST_WEB_HOME}/logs/seatunnel-web-server.log"]
```

> **注意事项**：
> 
> - Dockerfile 中的 `$(NAMESPACE)` 在构建时会被当作字面量写入配置文件，实际的命名空间由部署时的 K8s DNS 解析决定，因此构建时不需要替换。
> - 其他应用部署在**同一个命名空间**中，所以集群内部 DNS 地址中的命名空间部分要一致。

### 使用 Podman Desktop 构建并推送

```bash
# 确保 apache-seatunnel-web-1.0.3-SNAPSHOT.tar.gz 在当前目录
podman build -t harbor.你的公司.com/seatunnel/seatunnel-web:1.0.3 .
podman push harbor.你的公司.com/seatunnel/seatunnel-web:1.0.3
```

> 如果之前已经登录过 Harbor，推送不需要重新登录。如果 Token 过期，重新执行 `podman login harbor.你的公司.com`。

## 2.4 在 Rancher 中部署 Web

### 创建 Deployment

```
Workloads → Deployments → Create
```

**基本信息**：

| 字段                | 值                                               |
| ----------------- | ----------------------------------------------- |
| Name              | `seatunnel-web`                                 |
| Namespace         | `$(NAMESPACE)` — 选择你现有的命名空间                     |
| Container Image   | `harbor.你的公司.com/seatunnel/seatunnel-web:1.0.3` |
| Image Pull Policy | Always                                          |

> 如果平台集成了 Harbor，Pull Secrets 不用填。否则选 `harbor-secret`。

**端口**：点 Add Port，加两个：

| Service Type | Port | Container Port | Protocol |
| ------------ | ---- | -------------- | -------- |
| ClusterIP    | 8801 | 8801           | TCP      |
| ClusterIP    | 7890 | 7890           | TCP      |

**环境变量**：点 Add Environment Variable

| Key                   | Value       |
| --------------------- | ----------- |
| `DATASOURCE_PASSWORD` | 你的 MySQL 密码 |

**健康检查** → Readiness Probe：

```
Type: HTTP
Request Path: /api/health
Port: 7890
Initial Delay (s): 60
Period (s): 10
```

Liveness Probe 同上，Initial Delay 改 120。

**资源**：

```
CPU Request: 1    Memory Request: 2Gi
CPU Limit:   2    Memory Limit:   4Gi
```

点击 Launch。

### 验证

Pod Ready 后，点 Pod → View Logs，看到：

```
o.s.b.w.embedded.tomcat.TomcatWebServer: Tomcat started on port(s): 8801 (http)
```

说明启动成功。健康检查：

```
curl http://127.0.0.1:7890/api/health
# {"status":"UP"}
```

### 创建 Ingress 对外暴露

```
Service Discovery → Ingresses → Create
  → Name: seatunnel-web-ingress
  → Namespace: $(NAMESPACE)              ← 选择你现有的命名空间
  → Rules:
    → Host: seatunnel-web.你的公司.com
    → Path: /
    → Target Service: seatunnel-web:8801
  → Create
```

浏览器访问 `http://seatunnel-web.你的公司.com`，默认账号 **admin / admin**。

---

# 验证

## 验证连接数据源

登录 Web UI 后：

```
数据源 → 添加数据源
  → 选 MySQL → 填你的 MySQL 地址 → 测试连接 → ✅

数据源 → 添加数据源
  → 选 Doris → 填 Doris FE 地址和端口 → 测试连接 → ✅
```

## 验证 MySQL → Console 任务

```
任务管理 → 创建任务
  → Source: 选 MySQL 表
  → Sink: Console
  → 运行 → 查看日志 → 看到数据行 → ✅
```

## 验证 MySQL → Doris 同步

```
任务管理 → 创建任务
  → Source: 选 MySQL 表
  → Sink: 选 Doris 表（如果表不存在，勾选自动建表）
  → 运行
  → 到 Doris 查询确认数据已同步 → ✅
```

任务配置参考（如果需要在 Web 中手动写 HOCON）：

```hocon
env {
  parallelism = 2
  job.mode = "BATCH"
}

source {
  Jdbc {
    url = "jdbc:mysql://你的MySQL:3306/你的库"
    driver = "com.mysql.cj.jdbc.Driver"
    user = "你的用户"
    password = "你的密码"
    query = "SELECT * FROM 你的表"
  }
}

sink {
  Doris {
    fenodes = "你的DorisFE:8030"
    username = "root"
    password = ""
    database = "目标库"
    table = "目标表"
    sink.label-prefix = "sync-1"
    schema_save_mode = "CREATE_SCHEMA_WHEN_NOT_EXIST"
    doris.config = {
      format = "csv"
      column_separator = ","
    }
  }
}
```

---

# 排错

## Engine 集群组建失败

日志看到 `timeout waiting for members` 或 `Members {size:1}`。

进入 Master Pod → Execute Shell：

```bash
# 检查 DNS 能否解析到其他 Pod
nslookup seatunnel-hl.$(NAMESPACE).svc.cluster.local

# 检查网络能否通
nc -zv seatunnel-worker-0.seatunnel-hl.$(NAMESPACE).svc.cluster.local 5802
```

**原因排查**：

- Headless Service 的 `clusterIP` 是不是 `None`？
- 所有节点的 `cluster-name` 是不是一致？
- 网络策略/防火墙有没有放开 5801、5802？

## Web 连不上 Engine

Web 日志出现 `Cannot connect to cluster`。

进入 Web Pod → Execute Shell：

```bash
cat /opt/seatunnel-web/conf/hazelcast-client.yaml
nc -zv seatunnel-master-0.seatunnel-hl.$(NAMESPACE).svc.cluster.local 5801
```

**原因排查**：

- `cluster-members` 的 DNS 地址写对了没有（命名空间部分要匹配实际部署的命名空间）？
- `cluster-name` 和 Engine 的是否一致？

## ImagePullBackOff

Pod 拉不到镜像。点 Pod 看 Events。

**原因排查**：

- 镜像标签在 Harbor 中存不存在？
- 如果平台没集成 Harbor，有没有配 `imagePullSecrets`？
- Secret 里的账号密码对不对？

## 任务报 ClassNotFoundException

Worker Pod 执行任务时报类找不到。

进入 Worker Pod → Execute Shell：

```bash
ls /opt/seatunnel/connectors/
ls /opt/seatunnel/lib/
```

**原因**：连接器或 JDBC 驱动缺失。**解决**：回 1.2 节重新构建 Engine 镜像，确保 Dockerfile 中下载了需要的 jar。

---

## 附录

### 镜像清单

| 镜像     | Harbor 地址                                      | 装了啥                                                                             |
| ------ | ---------------------------------------------- | ------------------------------------------------------------------------------- |
| Engine | `harbor.xxx/seatunnel/seatunnel-engine:2.3.11` | Engine + connector-jdbc + connector-doris + connector-cdc-mysql + MySQL JDBC 驱动 |
| Web    | `harbor.xxx/seatunnel/seatunnel-web:1.0.3`     | Web + Engine Client + Hazelcast 配置 + application.yml + MySQL JDBC 驱动            |

### 端口

| 组件       | 端口   | 干什么用                     |
| -------- | ---- | ------------------------ |
| Master   | 5801 | 集群通信 + REST API          |
| Worker   | 5802 | 集群通信                     |
| Web UI   | 8801 | 浏览器访问                    |
| Web 健康检查 | 7890 | Readiness/Liveness Probe |

### 目录

| Pod 内路径                      | 里面放什么                                     |
| ---------------------------- | ----------------------------------------- |
| `/opt/seatunnel/connectors/` | 连接器 jar（connector-jdbc、connector-doris 等） |
| `/opt/seatunnel/lib/`        | JDBC 驱动 jar（mysql-connector-j 等）          |
| `/opt/seatunnel-web/libs/`   | Web 数据源驱动 jar                             |
| `/opt/seatunnel-web/conf/`   | application.yml、hazelcast-client.yaml     |

### v8 变更记录

与 v7 相比，v8 做了以下调整：

| 变更项       | v7                      | v8                                |
| --------- | ----------------------- | --------------------------------- |
| 构建工具      | Docker（Linux/Mac）       | Podman Desktop（Windows）           |
| 命名空间策略    | 新建 `seatunnel` 命名空间     | 使用现有命名空间，用 `$(NAMESPACE)` 占位      |
| Docker 命令 | `docker build/push/run` | `podman build/push/run`           |
| 环境要求      | 一台有 Docker + JDK 11 的机器 | 一台安装了 Podman Desktop 的 Windows 机器 |
| Helm 安装   | Create namespace ✅ 勾选   | **取消勾选**（命名空间已存在）                 |
| 新增章节      | —                       | Podman Desktop 常见问题、v8 变更记录       |
