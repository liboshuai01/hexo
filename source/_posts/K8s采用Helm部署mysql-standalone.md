---
title: K8s采用Helm部署mysql-standalone
tags:
  - Linux
  - K8s
  - Helm
  - Mysql
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250607042527522.png'
toc: true
abbrlink: 1ace4f5
date: 2025-06-11 18:40:02
---

在云原生时代，将有状态应用（如MySQL）部署到Kubernetes集群已成为标准实践。借助Helm这一强大的包管理工具，我们可以极大地简化部署和生命周期管理的复杂性。本文将详细阐述如何利用Bitnami社区维护的Helm Chart，在Kubernetes上部署一个带监控、配置灵活的MySQL单机实例（Standalone）。

我们将采用一种工程化的方式，通过配置文件（`.env`）和脚本（`install.sh`）来分离配置与执行逻辑，实现可重复、可维护的自动化部署。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-standalone), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-standalone)

## 核心思路

1.  **标准化与自动化**：使用业界公认的Bitnami Helm Chart作为部署模板，确保部署的标准化和可靠性。
2.  **配置即代码**：将所有可变配置（如命名空间、密码、资源限制等）集中管理在`.env`文件中，使得环境迁移和复现变得简单。
3.  **声明式部署**：通过`install.sh`脚本封装`helm`命令，实现一键式安装与升级，这是一种典型的声明式操作，符合Kubernetes的设计哲学。
4.  **可观测性集成**：在部署时即开启metrics端点，并自动创建`ServiceMonitor`资源，无缝对接到已有的Prometheus监控体系中，实现开箱即用的监控。
5.  **持久化存储**：为MySQL配置持久化卷（PVC），确保Pod重启或漂移后数据不丢失，这是有状态应用在K8s中运行的基石。

## 项目文件结构解析

在开始部署之前，我们先来理解项目的核心文件。

### 1. 配置文件: `.env`

此文件是所有自定义配置的入口，它定义了从部署环境到MySQL自身认证的全部变量。这种方式将配置与执行脚本分离，是云原生应用部署的最佳实践。

```shell
# 命名空间
NAMESPACE="mysql"
# helm的release名称
RELEASE_NAME="my-mysql-standalone"
# helm的chart版本
CHART_VERSION="10.3.0"
# 存储类名称
STORAGE_CLASS_NAME="nfs"

# MySQL root用户密码
MYSQL_ROOT_PASSWORD="YOUR_PASSWORD"
# MySQL 数据库名称
MYSQL_DATABASE="test"
# MySQL 用户名称
MYSQL_USER="lbs"
# MySQL 用户密码
MYSQL_PASSWORD="YOUR_PASSWORD"

# Prometheus 监控组件所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 ServiceMonitor 的标签值 (通常是 helm release 的名称)
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```
**解读**:
*   `NAMESPACE`, `RELEASE_NAME`: 定义了应用在K8s中的逻辑隔离空间和Helm实例名。
*   `STORAGE_CLASS_NAME`: **关键配置**。指定了用于数据持久化的存储类，你需要确保K8s集群中已存在名为`nfs`（或你自定义的名称）的StorageClass。
*   `auth.*`系列变量: 直接注入到MySQL容器中，用于初始化root密码和创建一个业务用户。
*   `PROMETHEUS_*`系列变量: 这是实现自动化监控的核心。它告诉Helm Chart，在哪个命名空间下创建`ServiceMonitor`，并且为这个`ServiceMonitor`打上什么标签，以便Prometheus Operator能够自动发现并开始采集监控指标。

### 2. 安装脚本: `install.sh`

这个Shell脚本是部署的执行器。它加载`.env`中的配置，并调用`helm`命令来完成MySQL的安装或升级。

```shell
#!/usr/bin/env bash

set -e

# --- 加载变量 ---
if [ -f .env ]; then
    source .env
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

# --- 添加仓库并更新 ---
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# --- 安装 / 升级 ---
helm upgrade --install ${RELEASE_NAME} bitnami/mysql --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --create-namespace \
  \
  --set architecture=standalone \
  --set-string global.storageClass="${STORAGE_CLASS_NAME}" \
  \
  --set-string auth.rootPassword=${MYSQL_ROOT_PASSWORD} \
  --set-string auth.database=${MYSQL_DATABASE} \
  --set-string auth.username=${MYSQL_USERNAME} \
  --set-string auth.password=${MYSQL_PASSWORD} \
  \
  --set primary.persistence.size=16Gi \
  --set primary.resources.requests.cpu=100m \
  --set primary.resources.requests.memory=128Mi \
  --set primary.resources.limits.cpu=512m \
  --set primary.resources.limits.memory=2048Mi \
  \
  --set secondary.resources.requests.cpu=250m \
  --set secondary.resources.requests.memory=512Mi \
  --set secondary.resources.limits.cpu=2000m \
  --set secondary.resources.limits.memory=4096Mi \
  \
  --set rbac.create=true \
  \
  --set metrics.enabled=true \
  --set metrics.serviceMonitor.enabled=true \
  --set metrics.serviceMonitor.namespace="${PROMETHEUS_NAMESPACE}" \
  --set metrics.serviceMonitor.labels.release="${PROMETHEUS_RELEASE_LABEL}" \
  --set metrics.resources.requests.cpu=100m \
  --set metrics.resources.requests.memory=128Mi \
  --set metrics.resources.limits.cpu=256m \
  --set metrics.resources.limits.memory=1024Mi
```
**解读**:
*   `helm upgrade --install`: 这是一个幂等命令，如果`RELEASE_NAME`不存在，它会执行安装；如果已存在，则会执行升级。这是自动化CI/CD流程中的常用技巧。
*   `--create-namespace`: 自动创建命名空间，无需手动操作。
*   `--set architecture=standalone`: 明确指定部署架构为单机模式。Bitnami Chart还支持`replication`等其他模式。
*   `--set primary.persistence.size` 和 `primary.resources.*`: 为MySQL主节点Pod配置了持久化卷大小以及CPU/Memory的请求和限制（Request/Limit），这是保障服务稳定性的重要资源管理措施。
*   `--set metrics.*`: 开启了MySQL Exporter，并启用了`ServiceMonitor`的创建。这使得Prometheus Operator可以自动发现此MySQL实例并将其纳入监控。

## 部署与验证实战

以下步骤将引导您完成整个部署和验证过程。

### 前提准备

在执行安装前，请务必根据您的环境修改`.env`文件。至少需要将`MYSQL_ROOT_PASSWORD`和`MYSQL_PASSWORD`修改为强密码，并确认`STORAGE_CLASS_NAME`在您的K8s集群中是真实存在的。

### 安装应用

只需一行命令，即可启动整个部署流程。

```shell
bash install.sh
```

脚本会自动添加Helm仓库、更新并部署MySQL。

### 验证应用

部署完成后，我们需要从几个层面来验证其正确性。

#### 初步验证

运行状态检查脚本，快速查看Pod是否正常运行。

```shell
bash status.sh
```
*(注：`status.sh`脚本通常包含`kubectl get pods -n <NAMESPACE>`等命令，用于检查Pod状态。)*

#### 进阶验证

我们需要亲自连接到数据库内部，确认其可用性。

> **注意**：默认Chart仅创建了root用户和在`.env`中指定的业务用户。

**1. 获取root用户密码**

此命令从Kubernetes Secret中安全地提取我们之前设置的root密码。

```shell
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql my-mysql-standalone -o jsonpath="{.data.mysql-root-password}" | base64 -d)
```

**2. 启动MySQL客户端Pod**

我们将在集群内部启动一个临时的MySQL客户端Pod，用于连接数据库服务。

```shell
kubectl run my-mysql-standalone-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.37-debian-12-r2 --namespace mysql --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash
```
*   `--rm --restart='Never'`: 确保这个客户端Pod在退出后被自动删除，非常适合临时调试。

**3. 连接MySQL**

在客户端Pod的shell中，使用K8s内部服务发现地址（Service DNS）进行连接。

```shell
mysql -h my-mysql-standalone.mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```
成功连接并看到MySQL提示符，即表明数据库实例已正常工作。

**4. K8s 内部访问 MySQL 实例**

集群内的其他应用可以通过以下地址访问MySQL服务，这是标准的K8s服务发现机制。

```shell
# <service>.<namespace>.svc.cluster.local:<port>
my-mysql-standalone.mysql.svc.cluster.local:3306
```

#### 监控验证

验证我们精心配置的可观测性是否生效。

**1. 访问`prometheus`的`/targets`页面**
在Prometheus的Web UI中，检查`/targets`页面。您应该能看到一个与MySQL相关的target，其状态为`UP`，这表示Prometheus已成功发现并正在抓取（scrape）`mysql-exporter`暴露的指标。

**2. 访问`grafana`并导入面板`14057`**
Grafana是数据可视化的利器。我们可以导入社区提供的优秀Dashboard（ID: `14057`）来直观地查看MySQL的各项性能指标，如QPS、连接数、缓存命中率等。如果图表正常显示数据，则说明监控链路已完全打通。

## 应用管理

### 更新应用

得益于`helm upgrade --install`和`.env`文件的设计，更新应用配置（例如，调整资源限制）变得非常简单：只需修改`.env`或`install.sh`中的相关参数，然后重新执行安装脚本即可。

```shell
bash install.sh
```

### 卸载应用

**1. 执行卸载脚本**

```shell
bash uninstall.sh
```
*(注: `uninstall.sh`脚本的核心是`helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}`命令)*

这会删除由Helm创建的所有K8s资源，如Deployment、Service、Secret等。

**2. （可选）删除持久化卷声明（PVC）**

**这是一个关键且需要谨慎操作的步骤。** Helm默认不会删除PVC，这是为了防止误操作导致数据丢失。如果您确认不再需要这些数据，可以手动删除。

```shell
# 加载变量以便使用 ${NAMESPACE}
source .env

# 查看pvc
kubectl get pvc -n ${NAMESPACE}

# 删除pvc（可能有多个pvc要删除）
kubectl delete pvc [pvc名称] -n ${NAMESPACE}
```

## 总结

通过本文的方案，我们不仅在Kubernetes上成功部署了MySQL单机实例，更重要的是，我们建立了一套**自动化、可配置、可观测**的部署与管理流程。这种将基础设施和应用配置代码化的实践，是现代DevOps和云原生开发的核心思想，它极大地提升了系统的可靠性、可维护性和团队协作效率。希望这套方案能为您的项目提供坚实的数据存储基座。