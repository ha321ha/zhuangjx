# SeaTunnel Engine Rancher 部署实操

> **目标**: 在 Rancher 管理的 K8s 集群 `ccs-sit` 命名空间下部署 SeaTunnel Engine（Master 2 副本 + Worker 3 副本）
>
> **环境**: Rancher v2.6.9 / RKE1 / Windows 本机
>
> **镜像**: 已构建好的 `reg.meicloud.com:40443/ccs-std/prd/seatunnel-engine:2.3.11`
>
> **来源**: 官方 Helm chart `oci://docker.m.daocloud.io/apache/seatunnel-helm:2.3.10`（DaoCloud 国内镜像）

---

## 目录

- [前置条件](#前置条件)
- [第 1 步：下载 Helm](#第-1-步下载-helm)
- [第 2 步：拉取 SeaTunnel 官方 Helm chart](#第-2-步拉取-seatunnel-官方-helm-chart)
- [第 3 步：创建自定义 values 文件](#第-3-步创建自定义-values-文件)
- [第 4 步：渲染 YAML](#第-4-步渲染-yaml)
- [第 5 步：在 Rancher 中导入 YAML](#第-5-步在-rancher-中导入-yaml)
- [第 6 步：验证部署](#第-6-步验证部署)
- [第 7 步：配置 Ingress 访问 Web UI](#第-7-步配置-ingress-访问-web-ui)
- [排错](#排错)

---

## 构建 Engine 镜像

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


---

## 第 1 步：下载 Helm

> **为什么需要下载？** Rancher 集群不能连公网，需要在本地渲染 YAML 再导入 Rancher。流程是：**本地下载 Helm → 下载 chart → 渲染 YAML → 导入 Rancher**。Rancher 自带的 Helm 在这个流程中用不上，所以本地必须备一份。

### 1.1 下载压缩包

浏览器打开：

```
https://github.com/helm/helm/releases/latest
```

Assets 列表中不会直接看到 `.zip`（GitHub 只挂了签名文件），实际安装包在 `get.helm.sh`。直接点这个链接下载：

```
https://get.helm.sh/helm-v4.2.2-windows-amd64.zip
```

> **注：** `amd64` = Windows **x64**，同一架构的不同叫法，放心下载。

或者用 PowerShell 下载：

```powershell
curl.exe -Lo D:\MyData\zhuangjx\helm\helm-v4.2.2-windows-amd64.zip `
  https://get.helm.sh/helm-v4.2.2-windows-amd64.zip
```




### 1.2 解压

```powershell
# 创建 helm 目录
mkdir D:\MyData\zhuangjx\helm -Force

# 解压
Expand-Archive -Path D:\MyData\zhuangjx\helm\helm-v4.2.2-windows-amd64.zip `
  -DestinationPath D:\MyData\zhuangjx\helm\

# 确认解压后文件
ls D:\MyData\zhuangjx\helm\windows-amd64\helm.exe
```

### 1.3 验证

```powershell
 .\helm.exe version
```

预期输出（版本号以实际下载为准）：
```
version.BuildInfo{Version:"v4.2.2", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.26.4"}
```

> 输出中的 Version 要 ≥ v3.8，否则 OCI 功能不可用。

---

## 第 2 步：下载 SeaTunnel 官方 Helm chart

> Docker Hub (`registry-1.docker.io`) 在国内被屏蔽，改为使用 DaoCloud 提供的国内镜像源。
>
> **注：** Helm chart 版本（2.3.10）和 SeaTunnel 引擎版本（2.3.11）是独立的，chart 只定义 K8s 资源，引擎版本在 values.yaml 中通过 `image.tag` 指定，互不影响。

### 2.1 拉取 chart

DaoCloud 维护了 Docker Hub 的国内镜像 `docker.m.daocloud.io`，直接拉取：

```powershell
# 进入 Helm 目录
cd D:\MyData\zhuangjx\helm\windows-amd64

# 使用 DaoCloud 国内镜像拉取（无需翻墙）
# .\helm.exe pull oci://registry-1.docker.io/apache/seatunnel-helm --version 2.3.11
.\helm.exe pull oci://docker.m.daocloud.io/apache/seatunnel-helm --version 2.3.10
```

成功输出：
```
Pulled: docker.m.daocloud.io/apache/seatunnel-helm:2.3.10
Digest: sha256:92aad7c076c13d167e9851a8884f946486cf7085a6aea631da3872bbddbde8c8
```

### 2.2 解压

```powershell
# 进入工作目录
cd D:\MyData\zhuangjx

# 解压
tar -xvf seatunnel-helm-2.3.10.tgz

# 确认解压成功
ls seatunnel-helm
```

预期看到：

```
    Directory: D:\MyData\zhuangjx\seatunnel-helm

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         2026/7/9      xx:xx                conf
d-----         2026/7/9      xx:xx                templates
-a----         2026/7/9      xx:xx           1250 Chart.yaml
-a----         2026/7/9      xx:xx           2657 values.yaml
```

---

## 第 3 步：创建自定义 values 文件

在 `D:\MyData\zhuangjx\seatunnel-helm\` 目录下创建 `values-custom.yaml`。

**方法：** 用记事本或 VS Code 创建，内容如下：

```yaml
# ──────────────────────────────────────────
# SeaTunnel Engine 自定义部署配置
# 命名空间: ccs-sit
# 镜像: reg.meicloud.com:40443/ccs-std/prd/seatunnel-engine:2.3.11
# ──────────────────────────────────────────

# 镜像配置
image:
  registry: reg.meicloud.com:40443/ccs-std/prd/seatunnel-engine
  tag: "2.3.11"
  pullPolicy: Always

# 目标命名空间
namespace: ccs-sit

# 全局环境变量
env:
  - name: TZ
    value: Asia/Shanghai

# Master 配置
master:
  replicas: 2
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"

# Worker 配置
worker:
  replicas: 3
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"

# Ingress（Web UI 外部访问）
ingress:
  enabled: true
  host: "seatunnel-sit.meicloud.com"
```

---

## 第 4 步：渲染 YAML

```powershell
# 确保当前在 seatunnel-helm 目录下
cd D:\MyData\zhuangjx\seatunnel-helm

# 渲染 YAML（注意 helm.exe 在 windows-amd64 子目录下）
D:\MyData\zhuangjx\Desktop\新建文件夹\helm-v4.2.2-windows-amd64\windows-amd64\helm.exe template seatunnel . `--namespace ccs-sit `-f values-custom.yaml ` --api-versions networking.k8s.io/v1/Ingress ` > seatunnel-rendered.yaml

```

### 检查渲染结果

```powershell
# 查看文件大小（应该有几百行）
ls seatunnel-rendered.yaml

# 查看包含的资源类型
Select-String -Path seatunnel-rendered.yaml -Pattern "^kind:" | Select-Object -Unique
```

预期输出（顺序严格按官方 chart 排列）：
共 **9 个资源**：

### ingress修改

> - apiVersion:修改：`extensions/v1beta1` → `networking.k8s.io/v1`（K8s v1.22+ 要求）
> - spec:添加 `ingressClassName: nginx`（匹配集群已有的 Ingress Controller）


---

## 第 5 步：在 Rancher 中导入 YAML

> **前提：** 以下操作全部在 Rancher 网页界面中完成，**不需要登录任何主机**。

`seatunnel-rendered.yaml` 包含 9 个不同类型的 K8s 资源（ingress、ServiceAccount、Role、RoleBinding、ConfigMap、Service ×2、Deployment ×2）。下面提供两种方式导入，推荐方式一。

---

### 方式一：用 Kubectl Shell 一次性导入（推荐，无需登录主机）

Kubectl Shell 是 Rancher **内置的网页终端**，在浏览器中直接运行，不需要 SSH 登录任何主机，也不需要安装任何工具。

#### 第 1 步：复制 YAML 内容

用记事本打开 `D:\MyData\zhuangjx\seatunnel-helm\seatunnel-rendered.yaml`，全选（Ctrl+A）→ 复制（Ctrl+C）。

#### 第 2 步：打开 Kubectl Shell

```
 "Kubectl Shell" 按钮（图标是 >_）
```

页面底部会弹出一个黑色的终端窗口。

#### 第 3 步：粘贴并执行 YAML


**此命令需要管理员账号执行**

```bash
cat << 'EOF' | kubectl apply -n ccs-sit -f -
<这里放渲染好的 YAML # seatunnel-rendered.yaml>
EOF
```

然后按回车。kubectl 会自动执行，你应该看到类似这样的输出：

```
serviceaccount/seatunnel created
role.rbac.authorization.k8s.io/seatunnel created
rolebinding.rbac.authorization.k8s.io/seatunnel created
configmap/seatunnel-configs created
service/seatunnel created
service/seatunnel-master created
deployment.apps/seatunnel-master created
deployment.apps/seatunnel-worker created
```

> **排错：** 如果提示 `Error from server (AlreadyExists)`，说明资源已存在，先执行 `kubectl delete -n ccs-sit --all serviceaccount,role,rolebinding,configmap,service,deployment -l app.kubernetes.io/instance=seatunnel` 清理后再重试。

## 第 6 步：验证部署

在 Kubectl Shell 中依次执行。

### 6.1 检查 Pod 状态

```bash
kubectl get pods -n ccs-sit -l app.kubernetes.io/instance=seatunnel
```

预期 5 个 Pod 全部 `Running`、`1/1`。

### 6.2 验证集群组建

```bash
kubectl logs -n ccs-sit deployment/seatunnel-master --tail=200 | grep -i members
```

预期看到 `size:5`：

```
Members {size:5, ver:5} [
        Member [10.42.x.x]:5801 - ... [active master]
        Member [10.42.x.x]:5801 - ... [master node]
        Member [10.42.x.x]:5801 - ... [worker node]
        Member [10.42.x.x]:5801 - ... [worker node]
        Member [10.42.x.x]:5801 - ... [worker node]
]
```

2 Master + 3 Worker 全部加入即部署成功。

### 6.3 Web UI 功能概览

SeaTunnel Engine 内置 Web UI（2.3.x 起自带），提供**纯监控**能力（只读，不能提交或停止任务）。打开后包含三个模块：

| 模块 | 功能 |
|------|------|
| **Jobs** | 查看运行中 / 已完成的作业。点进详情可见 DAG 图、任务分布、资源用量、日志、实时吞吐指标 |
| **Workers** | 每个 Worker 节点的地址、运行状态、CPU / 内存使用率、正在执行的任务数 |
| **Master** | Master 节点地址、运行状态、调度职责、集群资源分配 |

> **注意：** 这是 SeaTunnel Engine 自带的 Web UI（`apache/seatunnel` 项目），与 `seatunnel-web`（独立项目，需要额外部署 + MySQL，提供任务创建/调度/管理功能）是两回事。

```bash
kubectl exec -n ccs-sit deployment/seatunnel-worker -- wget -S -O- http://seatunnel-master:8080/ 2>&1 | head -30
```
---