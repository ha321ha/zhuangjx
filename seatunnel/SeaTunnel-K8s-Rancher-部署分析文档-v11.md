# SeaTunnel + SeaTunnel-Web 生产部署文档 v11（Rancher 版）

> **版本**: SeaTunnel 2.3.11 / SeaTunnel-Web 1.0.3-SNAPSHOT  
> **场景**: MySQL ↔ Doris 数据同步  
> **日期**: 2026-07-09  
> **依据**: 官方 Helm chart（`apache/seatunnel` → `deploy/kubernetes/seatunnel` @ dev 分支）

---

## 关于版本兼容性的说明（必读）

这份文档依赖一个核心操作：**用 `helm template` 渲染官方 chart 为 YAML**。这个操作需要：

- ✅ 一台能联网的机器（你的笔记本、跳板机都行）
- ✅ 这台机器上有 `helm` 3.8+
- ✅ 这台机器上有 `kubectl` 能连到你的 Rancher 集群

**这个操作跟你的 Rancher 版本无关。** `helm pull oci://...` 是 helm 命令行操作，不需要 Rancher 支持 OCI。你在自己的机器上执行一次，生成 YAML 文件，然后把 YAML 导入 Rancher 就行。

> 你之前听到我前后矛盾是因为：
> - **Rancher UI 的 "Apps → Repositories → OCI 类型"** 需要 Rancher v2.9+ → 你的 v2.6.9 不支持
> - **但 `helm pull oci://...`** 是 Helm CLI，跟 Rancher 版本无关 → 你的 v2.6.9 完全可以用
>
> 这是两个不同的功能，后面会详细说。

---

## 版本检查清单

在开始之前，先确认你的环境版本：

### 1. 查看 Rancher 版本

```
Rancher UI → 左下角 → 显示版本号
```
或执行：
```bash
# 如果你有 kubectl 权限
kubectl get settings.management.cattle.io server-version -o jsonpath='{.value}'
```

### 2. 确认 Rancher 功能支持

| 功能 | 你的版本 v2.6.9 | 要求版本 |
|------|----------------|---------|
| Apps → Charts 浏览安装 | ✅ 支持 | 所有版本 |
| Apps → Repositories 添加 HTTP 仓库 | ✅ 支持 | 所有版本 |
| **Apps → Repositories 添加 OCI 仓库** | ❌ **不支持** | **v2.9+** |
| **"Install Helm" 输入 OCI URL** | ❌ **不支持** | **v2.9+** |
| **Import YAML** | ✅ **支持** | **所有版本** ← 本文档用这个 |
| Workloads → Deployment | ✅ 支持 | 所有版本 |

### 3. 确认 Helm 版本（在你的机器上）

```bash
helm version
# 需要 ≥ v3.8.0 才能使用 OCI 功能
# 检查方法：输出中应该有 "v3.8" 或更高
```

如果版本太低，升级：
```bash
# Windows (choco)
choco upgrade kubernetes-helm

# Windows (winget)  
winget upgrade Helm.Helm

# macOS
brew upgrade helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 4. 确认 kubectl 能连到你的集群

```bash
kubectl cluster-info
kubectl get nodes
# 应该能看到你的 RKE1 节点列表
```

---

## 目录

- [第一章：SeaTunnel Engine 部署](#第一章seatunnel-engine-部署)
  - [1.1 架构一句话](#11-架构一句话)
  - [1.2 构建 Engine 镜像](#12-构建-engine-镜像)
  - [1.3 用官方 helm template 生成 YAML](#13-用官方-helm-template-生成-yaml)
  - [1.4 在 Rancher 中导入 YAML](#14-在-rancher-中导入-yaml)
  - [1.5 验证集群](#15-验证集群)
- [第二章：SeaTunnel-Web 部署](#第二章seatunnel-web-部署)
  - [2.1 准备 MySQL](#21-准备-mysql)
  - [2.2 构建 Web 镜像](#22-构建-web-镜像)
  - [2.3 部署 Web](#23-部署-web)
- [验证](#验证)
- [排错](#排错)
- [附录：手动编写 YAML（无 Helm 方案）](#附录手动编写-yaml无-helm-方案)

---

# 第一章：SeaTunnel Engine 部署

## 1.1 架构一句话

SeaTunnel Engine 采用 Master-Worker 分离架构：

- **Master**（2 个）：负责任务调度、REST API。一主一备做 HA
- **Worker**（3 个）：实际干活，执行数据同步任务。可以随时扩容
- 节点间通过内嵌 Hazelcast 自动组建集群，**不需要额外安装 ZooKeeper**
- 服务发现使用 **Hazelcast Kubernetes discovery plugin**（官方 chart 默认方式）

## 1.2 构建 Engine 镜像

```bash
mkdir seatunnel-engine-build
cd seatunnel-engine-build
```

创建 `Dockerfile`：

```dockerfile
FROM openjdk:11-jdk-slim

ARG VERSION=2.3.11
ENV SEATUNNEL_HOME=/opt/seatunnel

RUN apt-get update && apt-get install -y wget && rm -rf /var/lib/apt/lists/* && \
    wget -q https://dlcdn.apache.org/seatunnel/${VERSION}/apache-seatunnel-${VERSION}-bin.tar.gz \
      -O /tmp/seatunnel.tar.gz && \
    mkdir -p ${SEATUNNEL_HOME} && \
    tar -xzf /tmp/seatunnel.tar.gz -C /tmp/ && \
    mv /tmp/apache-seatunnel-${VERSION}/* ${SEATUNNEL_HOME}/ && \
    rm -rf /tmp/*

# 连接器
RUN mkdir -p ${SEATUNNEL_HOME}/connectors && \
    wget -q https://repo1.maven.org/maven2/org/apache/seatunnel/connector-jdbc/${VERSION}/connector-jdbc-${VERSION}.jar \
      -O ${SEATUNNEL_HOME}/connectors/connector-jdbc-${VERSION}.jar && \
    wget -q https://repo1.maven.org/maven2/org/apache/seatunnel/connector-doris/${VERSION}/connector-doris-${VERSION}.jar \
      -O ${SEATUNNEL_HOME}/connectors/connector-doris-${VERSION}.jar && \
    wget -q https://repo1.maven.org/maven2/org/apache/seatunnel/connector-cdc-mysql/${VERSION}/connector-cdc-mysql-${VERSION}.jar \
      -O ${SEATUNNEL_HOME}/connectors/connector-cdc-mysql-${VERSION}.jar

# JDBC 驱动
RUN wget -q https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar \
      -O ${SEATUNNEL_HOME}/lib/mysql-connector-j-8.0.33.jar

WORKDIR ${SEATUNNEL_HOME}
```

构建推送：

```bash
# 在 Windows（Podman Desktop）上执行
podman build -t harbor.你的公司.com/seatunnel/seatunnel-engine:2.3.11 .
podman push harbor.你的公司.com/seatunnel/seatunnel-engine:2.3.11
```

---

## 1.3 用官方 helm template 生成 YAML

这是**唯一官方**的部署方式。你在自己的机器上执行，不需要在 Rancher 中操作 Helm。

### 前提

- ✅ 机器能访问互联网（或能连到 Harbor）
- ✅ `helm version` ≥ 3.8
- ✅ `kubectl` 能连到你的 K8s 集群

### 操作步骤

```bash
# ──────────────────────────────────
# 第 1 步：拉取官方 chart
# ──────────────────────────────────
helm pull oci://registry-1.docker.io/apache/seatunnel-helm --version 2.3.11

# 解压
tar -xvf seatunnel-helm-2.3.11.tgz
cd seatunnel-helm
```

```bash
# ──────────────────────────────────
# 第 2 步：创建自定义 values 文件
# ──────────────────────────────────
cat > values-custom.yaml << 'EOF'
# ── 镜像 ──
image:
  registry: harbor.你的公司.com/seatunnel/seatunnel-engine
  tag: "2.3.11"
  pullPolicy: Always
  # pullSecret: "harbor-secret"   # 需要认证时取消注释

# ── 命名空间（必须改为你的实际命名空间）──
namespace: $(NAMESPACE)

# ── 环境变量 ──
env:
  - name: TZ
    value: Asia/Shanghai

# ── Master 配置 ──
master:
  replicas: 2
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"

# ── Worker 配置（官方默认 2，生产建议 3）──
worker:
  replicas: 3
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"

# ── Ingress ──
ingress:
  enabled: false
EOF
```

```bash
# ──────────────────────────────────
# 第 3 步：渲染 YAML
# ──────────────────────────────────
helm template seatunnel . \
  --namespace $(NAMESPACE) \
  -f values-custom.yaml \
  > seatunnel-rendered.yaml

# 确认生成成功
wc -l seatunnel-rendered.yaml
# 应该有 200-300 行

head -20 seatunnel-rendered.yaml
# 第一行应该看到 ---
# 然后看到 # Source: seatunnel-helm/templates/...
```

> **为什么不直接在 Rancher 中执行 helm？**
>
> 因为 Rancher v2.6.9 不支持 OCI Helm 仓库。`helm pull oci://...` 需要在 Helm CLI 中执行。你的机器上有 Helm 就够了，跟 Rancher 版本无关。

### 渲染出的 YAML 包含什么？

| 资源 | 名称 | 用途 |
|------|------|------|
| `ServiceAccount` | `seatunnel` | Hazelcast 查询 K8s API 用 |
| `Role` | `seatunnel` | 允许 `get/watch/list configmaps` |
| `RoleBinding` | `seatunnel` | 绑定 SA 到 Role |
| `ConfigMap` | `seatunnel-configs` | 包含所有配置文件 |
| `Service` (#1) | `seatunnel` | Headless（clusterIP: None），端口 5801，Hazelcast 发现用 |
| `Service` (#2) | `seatunnel-master` | Headless（clusterIP: None），端口 8080，REST API 用 |
| `Deployment` | `seatunnel-master` | 2 副本 |
| `Deployment` | `seatunnel-worker` | 3 副本 |

---

## 1.4 在 Rancher 中导入 YAML

```bash
# ──────────────────────────────────
# 第 4 步（方案 A）：通过 Rancher UI 导入
# ──────────────────────────────────
# seatunnel-rendered.yaml 已经在你的机器上了，在 Rancher 中：
```

**Rancher 操作：**
```
进入你的目标集群 → 右上角 "Import YAML"
  → 点击 "Read from a file" 上传 seatunnel-rendered.yaml
  → 确认 Namespace 选中你的现有命名空间
  → 点击 Import
```

```bash
# ──────────────────────────────────
# 第 4 步（方案 B）：通过 kubectl 导入
# ──────────────────────────────────
# 如果你在的机器有 kubectl 权限，直接：
kubectl apply -f seatunnel-rendered.yaml -n $(NAMESPACE)
```

### 验证导入成功

```
Workloads → Deployments →
  ├─ seatunnel-master: 2/2 Running  ✓
  └─ seatunnel-worker: 3/3 Running  ✓
```

---

## 1.5 验证集群

点任一 Master Pod → View Logs，搜索 `Members`：

```
Members {size:5, ver:3} [...]
# size:5 = 2 Master + 3 Worker 全部加入集群
```

进入 Worker Pod → Execute Shell：

```bash
ls /opt/seatunnel/connectors/
# 应有：connector-jdbc-2.3.11.jar, connector-doris-2.3.11.jar, connector-cdc-mysql-2.3.11.jar

ls /opt/seatunnel/lib/
# 应有：mysql-connector-j-8.0.33.jar
```

---

# 第二章：SeaTunnel-Web 部署

## 2.1 准备 MySQL

```sql
CREATE DATABASE IF NOT EXISTS seatunnel CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER IF NOT EXISTS 'seatunnel'@'%' IDENTIFIED BY '你的密码';
GRANT ALL PRIVILEGES ON seatunnel.* TO 'seatunnel'@'%';
FLUSH PRIVILEGES;
```

```bash
wget https://raw.githubusercontent.com/apache/seatunnel-web/1.0.3-SNAPSHOT/seatunnel-server/seatunnel-app/src/main/resources/script/seatunnel_server_mysql.sql
mysql -h 你的MySQL地址 -u seatunnel -p -D seatunnel < seatunnel_server_mysql.sql
```

### 在 K8s 中映射 MySQL

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-mysql
  namespace: $(NAMESPACE)
spec:
  type: ExternalName
  externalName: mysql.你的公司.com
```

## 2.2 构建 Web 镜像

```dockerfile
FROM openjdk:11-jdk-slim

ARG VERSION=2.3.11
ARG ST_WEB_VERSION=1.0.3-SNAPSHOT

ENV SEATUNNEL_HOME=/opt/seatunnel
ENV ST_WEB_HOME=/opt/seatunnel-web

RUN apt-get update && apt-get install -y wget && rm -rf /var/lib/apt/lists/* && \
    mkdir -p ${SEATUNNEL_HOME} ${ST_WEB_HOME}/logs

# SeaTunnel Engine
RUN wget -q https://dlcdn.apache.org/seatunnel/${VERSION}/apache-seatunnel-${VERSION}-bin.tar.gz \
      -O /tmp/seatunnel.tar.gz && \
    tar -xzf /tmp/seatunnel.tar.gz -C /tmp/ && \
    mv /tmp/apache-seatunnel-${VERSION}/* ${SEATUNNEL_HOME}/ && \
    rm -rf /tmp/*

# SeaTunnel-Web
COPY apache-seatunnel-web-${ST_WEB_VERSION}.tar.gz /tmp/
RUN tar -xzf /tmp/apache-seatunnel-web-${ST_WEB_VERSION}.tar.gz -C /tmp/ && \
    mv /tmp/apache-seatunnel-web-${ST_WEB_VERSION}/* ${ST_WEB_HOME}/ && \
    rm -rf /tmp/*

# MySQL JDBC 驱动
RUN mkdir -p ${ST_WEB_HOME}/libs && \
    wget -q https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar \
      -O ${ST_WEB_HOME}/libs/mysql-connector-j-8.0.33.jar

# Hazelcast Client 配置
# 重要：Web 通过 headless service DNS seatunnel.NAMESPACE.svc.cluster.local:5801 连 Engine
RUN echo 'hazelcast-client:\n\
  cluster-name: seatunnel\n\
  properties:\n\
    hazelcast.logging.type: log4j2\n\
  connection-strategy:\n\
    connection-retry:\n\
      cluster-connect-timeout-millis: 3000\n\
  network:\n\
    cluster-members:\n\
      - seatunnel.$(NAMESPACE).svc.cluster.local:5801' \
  > ${ST_WEB_HOME}/conf/hazelcast-client.yaml

# 插件映射
RUN if [ -f ${SEATUNNEL_HOME}/connectors/plugin-mapping.properties ]; then \
      cp ${SEATUNNEL_HOME}/connectors/plugin-mapping.properties ${ST_WEB_HOME}/conf/; \
    fi

# application.yml
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

构建推送：

```bash
podman build -t harbor.你的公司.com/seatunnel/seatunnel-web:1.0.3 .
podman push harbor.你的公司.com/seatunnel/seatunnel-web:1.0.3
```

## 2.3 部署 Web

创建 `seatunnel-web.yaml`，在 Rancher 中 Import YAML：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seatunnel-web
  namespace: $(NAMESPACE)
  labels:
    app: seatunnel-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: seatunnel-web
  template:
    metadata:
      labels:
        app: seatunnel-web
    spec:
      containers:
        - name: seatunnel-web
          image: harbor.你的公司.com/seatunnel/seatunnel-web:1.0.3
          imagePullPolicy: Always
          ports:
            - containerPort: 8801
              name: http
            - containerPort: 7890
              name: health
          env:
            - name: DATASOURCE_PASSWORD
              value: "你的MySQL密码"
          startupProbe:
            httpGet:
              path: /api/health
              port: 7890
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /api/health
              port: 7890
            initialDelaySeconds: 120
            periodSeconds: 15
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: seatunnel-web
  namespace: $(NAMESPACE)
spec:
  selector:
    app: seatunnel-web
  ports:
    - name: http
      port: 8801
      targetPort: 8801
    - name: health
      port: 7890
      targetPort: 7890
```

### 创建 Ingress

```
Service Discovery → Ingresses → Create
  → Name: seatunnel-web-ingress
  → Namespace: $(NAMESPACE)
  → Rules:
    → Host: seatunnel-web.你的公司.com
    → Path: /
    → Target Service: seatunnel-web:8801
  → Create
```

浏览器访问 `http://seatunnel-web.你的公司.com`，默认账号 **admin / admin**。

---

# 验证

```
数据源 → 添加数据源 → 选 MySQL → 测试连接 → ✅
数据源 → 添加数据源 → 选 Doris → 测试连接 → ✅
任务管理 → 创建任务 → Source: MySQL → Sink: Console → 运行 → ✅
```

---

# 排错

### Engine 集群组建失败

```bash
# 检查 Pod 是否 Running
kubectl get pods -n $(NAMESPACE) -l app.kubernetes.io/instance=seatunnel

# 查看 Master 日志（看 Hazelcast 发现过程）
kubectl logs -n $(NAMESPACE) -l app.kubernetes.io/component=master --tail=50

# 确认 RBAC 已创建（Hazelcast 需要查 API）
kubectl get sa,role,rolebinding -n $(NAMESPACE) | grep seatunnel
```

### ImagePullBackOff

- 镜像在 Harbor 中存在吗？
- 需要 `imagePullSecrets` 吗？
- 查看 Events：`kubectl describe pod -n $(NAMESPACE) seatunnel-master-xxx`

### Web 连不上 Engine

```bash
kubectl exec -n $(NAMESPACE) deploy/seatunnel-web -- cat /opt/seatunnel-web/conf/hazelcast-client.yaml
# 确认 cluster-members 地址和 cluster-name 正确
```

---

# 附录：手动编写 YAML（无 Helm 方案）

如果你完全无法使用 Helm（既没有 Helm CLI，也无法在机器上安装 Helm），可以手动创建以下 YAML 文件。

> **注意**：手动 YAML 基于官方 chart 结构，但将 Hazelcast 发现方式改为 **TCP/IP member-list**（不依赖 `hazelcast-kubernetes` 插件），因此不需要 RBAC。

创建 `seatunnel-engine-manual.yaml`：

```yaml
---
# ── ConfigMap ──
apiVersion: v1
kind: ConfigMap
metadata:
  name: seatunnel-configs
  namespace: $(NAMESPACE)
data:
  hazelcast-master.yaml: |
    hazelcast:
      cluster-name: seatunnel
      network:
        rest-api:
          enabled: true
          endpoint-groups:
            CLUSTER_WRITE:
              enabled: true
            DATA:
              enabled: true
        join:
          tcp-ip:
            enabled: true
            member-list:
              - seatunnel-hl
        port:
          auto-increment: false
          port: 5801
      properties:
        hazelcast.logging.type: log4j2

  hazelcast-worker.yaml: |
    hazelcast:
      cluster-name: seatunnel
      network:
        rest-api:
          enabled: true
          endpoint-groups:
            CLUSTER_WRITE:
              enabled: true
            DATA:
              enabled: true
        join:
          tcp-ip:
            enabled: true
            member-list:
              - seatunnel-hl
        port:
          auto-increment: false
          port: 5801
      properties:
        hazelcast.logging.type: log4j2

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
          timeout: 10000
          storage:
            type: hdfs
            max-retained: 3
            plugin-config:
              storage.type: hdfs
              fs.defaultFS: file:///tmp/seatunnel/checkpoint/

---
# ── Headless Service（Hazelcast 发现用）──
apiVersion: v1
kind: Service
metadata:
  name: seatunnel-hl
  namespace: $(NAMESPACE)
spec:
  clusterIP: None
  selector:
    app.kubernetes.io/instance: seatunnel
  ports:
    - name: hazelcast
      port: 5801

---
# ── Master Deployment ──
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seatunnel-master
  namespace: $(NAMESPACE)
  labels:
    app.kubernetes.io/instance: seatunnel
    app.kubernetes.io/component: master
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/instance: seatunnel
      app.kubernetes.io/component: master
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: seatunnel
        app.kubernetes.io/component: master
    spec:
      containers:
        - name: seatunnel-engine
          image: harbor.你的公司.com/seatunnel/seatunnel-engine:2.3.11
          imagePullPolicy: Always
          ports:
            - containerPort: 5801
              name: hazelcast
            - containerPort: 8080
              name: rest-api
          command:
            - /bin/sh
            - -c
            - /opt/seatunnel/bin/seatunnel-cluster.sh -r master
          env:
            - name: TZ
              value: Asia/Shanghai
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          volumeMounts:
            - name: config
              mountPath: /opt/seatunnel/config/hazelcast-master.yaml
              subPath: hazelcast-master.yaml
            - name: config
              mountPath: /opt/seatunnel/config/hazelcast-worker.yaml
              subPath: hazelcast-worker.yaml
            - name: config
              mountPath: /opt/seatunnel/config/seatunnel.yaml
              subPath: seatunnel.yaml
          livenessProbe:
            tcpSocket:
              port: 5801
            initialDelaySeconds: 30
            periodSeconds: 30
      volumes:
        - name: config
          configMap:
            name: seatunnel-configs

---
# ── Worker Deployment ──
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seatunnel-worker
  namespace: $(NAMESPACE)
  labels:
    app.kubernetes.io/instance: seatunnel
    app.kubernetes.io/component: worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/instance: seatunnel
      app.kubernetes.io/component: worker
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: seatunnel
        app.kubernetes.io/component: worker
    spec:
      containers:
        - name: seatunnel-engine
          image: harbor.你的公司.com/seatunnel/seatunnel-engine:2.3.11
          imagePullPolicy: Always
          ports:
            - containerPort: 5801
              name: hazelcast
          command:
            - /bin/sh
            - -c
            - /opt/seatunnel/bin/seatunnel-cluster.sh -r worker
          env:
            - name: TZ
              value: Asia/Shanghai
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
            limits:
              cpu: "4"
              memory: "8Gi"
          volumeMounts:
            - name: config
              mountPath: /opt/seatunnel/config/hazelcast-master.yaml
              subPath: hazelcast-master.yaml
            - name: config
              mountPath: /opt/seatunnel/config/hazelcast-worker.yaml
              subPath: hazelcast-worker.yaml
            - name: config
              mountPath: /opt/seatunnel/config/seatunnel.yaml
              subPath: seatunnel.yaml
          livenessProbe:
            tcpSocket:
              port: 5801
            initialDelaySeconds: 30
            periodSeconds: 30
      volumes:
        - name: config
          configMap:
            name: seatunnel-configs
```

> **手动 YAML 与官方 chart 的差异**：
> 1. Hazelcast 发现方式改为 **TCP/IP** + Headless Service DNS（`seatunnel-hl`），不依赖 `hazelcast-kubernetes` 插件
> 2. 不需要 RBAC（因为 TCP/IP 发现不查询 K8s API）
> 3. 配置更精简，只挂了必要文件

部署：

```bash
# Rancher Import YAML 或 kubectl
kubectl apply -f seatunnel-engine-manual.yaml -n $(NAMESPACE)
```

---

## 附录 B：官方文件核对清单

以下文件来自 `apache/seatunnel` 仓库 `dev` 分支的 `deploy/kubernetes/seatunnel/`：

| 文件 | 内容 | 已核对 |
|------|------|--------|
| `Chart.yaml` | chart 名 `seatunnel-helm`，version 2.3.10 | ✅ |
| `values.yaml` | 默认镜像 `apache/seatunnel`，默认 worker 2 副本 | ✅ |
| `templates/deployment-seatunnel-master.yaml` | `kind: Deployment`，端口 5801+8080 | ✅ |
| `templates/deployment-seatunnel-worker.yaml` | `kind: Deployment`，端口 5801 | ✅ |
| `templates/configmap.yaml` | 挂载 `conf/` 目录所有文件 | ✅ |
| `templates/service-headless.yaml` | `clusterIP: None`，端口 5801 | ✅ |
| `templates/service-master-headless.yaml` | `clusterIP: None`，端口 8080 | ✅ |
| `templates/rbac.yaml` | ServiceAccount + Role + RoleBinding | ✅ |
| `templates/ingress.yaml` | 支持 networking.k8s.io/v1 | ✅ |
| `conf/hazelcast-master.yaml` | `join.kubernetes.enabled: true`，端口 5801 | ✅ |
| `conf/hazelcast-worker.yaml` | `join.kubernetes` + `member-attributes: rule=worker` | ✅ |
| `conf/hazelcast-client.yaml` | 连 headless service DNS `name.ns.svc.cluster.local:5801` | ✅ |
| `conf/seatunnel.yaml` | engine 参数配置 | ✅ |
| `conf/jvm_master_options` | `-Xms2g -Xmx2g -XX:+UseG1GC` | ✅ |
| `conf/jvm_worker_options` | 同上 | ✅ |

---

> **总结**：`helm template` 方法是官方唯一推荐的 K8s 部署方式。对于 Rancher v2.6.9，你在自己的机器上执行 `helm template` 生成 YAML，然后通过 Rancher 的 **Import YAML** 功能导入。不需要 Rancher 支持 OCI，你只需要自己机器上有 Helm 3.8+ 即可。
