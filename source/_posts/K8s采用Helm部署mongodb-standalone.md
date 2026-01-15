---
title: K8s采用Helm部署mongodb-standalone
abbrlink: f2948077
date: '2025-06-11T19:07:39.000Z'
tags:
  - 容器化
categories:
  - 容器化
toc: true
---


在现代云原生架构中，将有状态应用（如数据库）容器化并部署在 Kubernetes 上已成为主流实践。Kubernetes 提供了强大的编排能力，而 Helm 作为其官方包管理器，极大地简化了复杂应用的部署和生命周期管理。

本文将作为一篇实战指南，详细阐述如何利用 Helm 将一个生产级的单节点 MongoDB (Standalone) 实例高效、可复现地部署到 Kubernetes 集群中。我们将采用广泛使用的 Bitnami MongoDB Chart，并通过脚本化的方式实现配置、安装、验证与监控的全链路自动化。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/mongodb/mongodb-standalone), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/mongodb/mongodb-standalone)

### 核心优势

*   **声明式与可复现**：通过 `.env` 配置文件和 `install.sh` 脚本，我们将部署过程代码化，确保了环境的一致性和部署的可复现性。
*   **配置解耦**：将所有可变配置（如密码、命名空间、资源限制等）提取到 `.env` 文件中，使安装脚本保持通用，便于在不同环境（开发、测试、生产）中复用。
*   **集成监控**：无缝集成了 Prometheus 和 Grafana 监控，通过 `ServiceMonitor` 自动发现并采集 MongoDB 的性能指标，为运维提供了开箱即用的可观测性。
*   **生命周期管理**：提供了完整的安装、更新、验证和卸载脚本，覆盖了应用的整个生命周期。

## 准备工作：项目结构与配置

为了实现自动化和配置分离，我们采用以下文件结构：

*   `.env`: 存储所有环境变量和配置。
*   `install.sh`: 核心安装/更新脚本。
*   `README.md`: 项目说明与使用指南 (本文内容的核心来源)。

### 第一步：定义配置文件 (`.env`)

在开始之前，我们需要创建一个 `.env` 文件来定义部署所需的所有参数。这种方式将配置与执行逻辑完全分离，非常清晰。

```shell
# .env

# 命名空间
NAMESPACE="mongodb"
# helm的release名称
RELEASE_NAME="my-mongodb-standalone"
# helm chart版本 (建议指定一个明确的版本以保证部署的可复现性)
CHART_VERSION="16.5.20"
# 存储类名称
STORAGE_CLASS_NAME="nfs"

# MongoDB root用户密码
MONGO_ROOT_PASSWORD="YOUR_PASSWORD"
# MongoDB 应用数据库名称
MONGO_DATABASE="test"
# MongoDB 应用用户名称
MONGO_USER="lbs"
# MongoDB 应用用户密码
MONGO_PASSWORD="YOUR_PASSWORD"

# --- 监控配置 ---
# Prometheus Operator 所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 ServiceMonitor 的标签值
# 通常是 kube-prometheus-stack 的 release 名称, 你的 kubectl get all 输出中release名称为 kube-prom-stack
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```
**配置解析**：
*   **基础配置**: 定义了部署的命名空间、Helm Release 名称、Chart 版本和持久化存储所需的 `StorageClass`。锁定 `CHART_VERSION` 是一个非常好的实践，可以避免因 Chart 更新导致非预期的变更。
*   **认证配置**: 设置了 MongoDB 的 `root` 用户密码，并预先创建了一个应用数据库及对应的用户和密码。
*   **监控配置**: 这是集成可观测性的关键。`PROMETHEUS_NAMESPACE` 和 `PROMETHEUS_RELEASE_LABEL` 告诉 Helm Chart 如何创建一个 `ServiceMonitor` 资源，并让 Prometheus Operator（通常由 kube-prometheus-stack 部署）能够自动发现它。

**注意**：在实际使用前，请务必将 `YOUR_PASSWORD` 等占位符替换为强密码。

### 第二步：编写安装脚本 (`install.sh`)

这个脚本是整个部署流程的核心，它负责加载配置、更新 Helm 仓库，并执行 `helm upgrade --install` 命令。

```shell
#!/usr/bin/env bash

set -e

# --- 加载变量 ---
if [ -f .env ]; then
    # shellcheck disable=SC1091
    source .env
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

# --- 添加仓库并更新 ---
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# --- 安装 / 升级 ---
helm upgrade --install ${RELEASE_NAME} bitnami/mongodb --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --create-namespace \
  \
  --set architecture=standalone \
  --set useStatefulSet=true \
  --set-string global.storageClass="${STORAGE_CLASS_NAME}" \
  \
  --set-string auth.rootPassword="${MONGO_ROOT_PASSWORD}" \
  --set-string auth.databases[0]="${MONGO_DATABASE}" \
  --set-string auth.usernames[0]="${MONGO_USER}" \
  --set-string auth.passwords[0]="${MONGO_PASSWORD}" \
  \
  --set persistence.size=16Gi \
  \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=128Mi \
  --set resources.limits.cpu=512m \
  --set resources.limits.memory=2048Mi \
  \
  --set arbiter.resources.requests.cpu=100m \
  --set arbiter.resources.requests.memory=128Mi \
  --set arbiter.resources.limits.cpu=512m \
  --set arbiter.resources.limits.memory=2048Mi \
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
**脚本解析**:
*   `set -e`: 保证脚本在任何命令出错时立即退出。
*   `source .env`: 加载我们的配置文件。
*   `helm upgrade --install`: 这是一个幂等操作。如果 Release 不存在，它会执行安装 (`install`)；如果已存在，它会进行升级 (`upgrade`)。这使得同一个脚本可以用于初始安装和后续更新。
*   `--create-namespace`: 如果命名空间不存在，会自动创建。
*   `--set` 参数详解:
    *   `architecture=standalone`: 明确指定部署单节点模式。
    *   `useStatefulSet=true`: **至关重要**。对于数据库等有状态应用，应使用 `StatefulSet` 而不是 `Deployment`，它能提供稳定的网络标识符和持久化存储。
    *   `auth.*`: 将 `.env` 中的认证信息传递给 Chart。
    *   `resources.*`: 为 MongoDB 容器和监控 Exporter 容器设置了明确的 CPU 和内存的 `requests` 与 `limits`，这是保障 K8s 集群稳定性和资源调度公平性的最佳实践。
    *   `metrics.*`: 启用 metrics exporter，并配置 `ServiceMonitor` 的创建，使其能被 Prometheus 自动纳管。

## 执行与验证

### 安装应用

将 `.env` 和 `install.sh` 置于同一目录下，赋予脚本执行权限后运行：

```shell
chmod +x install.sh
bash install.sh
```

Helm 将会开始部署 MongoDB，整个过程可能需要几分钟，具体取决于镜像拉取速度和存储卷的创建速度。

### 验证应用

部署完成后，我们需要进行验证以确保一切正常。

#### 1. 初步验证

可以先通过 `kubectl` 查看 Pod 和 Service 的状态。一个简单的 `status.sh` 脚本可以封装如下命令：

```shell
# status.sh (示例)
source .env
echo "--- Pods in namespace ${NAMESPACE} ---"
kubectl get pods -n ${NAMESPACE} -o wide
echo "\n--- Services in namespace ${NAMESPACE} ---"
kubectl get svc -n ${NAMESPACE}
echo "\n--- PVC in namespace ${NAMESPACE} ---"
kubectl get pvc -n ${NAMESPACE}
```

执行 `bash status.sh`，如果看到 Pod 处于 `Running` 状态，Service 和 PVC 都已成功创建，则初步验证通过。

#### 2. 进阶验证：连接数据库

光看 Pod 状态还不够，我们需要实际连接到数据库内部进行验证。

**a. 获取 root 密码**

MongoDB 的 root 密码被安全地存储在 Kubernetes Secret 中。我们通过以下命令获取并解码：
```shell
export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace mongodb my-mongodb-standalone -o jsonpath="{.data.mongodb-root-password}" | base64 -d)
```

**b. 启动临时客户端 Pod**

为了从集群内部访问 MongoDB，我们启动一个临时的、包含 `mongosh` 客户端的 Pod：
```shell
kubectl run --namespace mongodb my-mongodb-standalone-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:8.0.10-debian-12-r1 --command -- bash
```

**c. 连接 MongoDB 实例**

在临时客户端 Pod 的 shell 中，执行连接命令：
```shell
# 主机名 "my-mongodb-standalone" 就是 Helm 创建的 Service 名称
mongosh admin --host "my-mongodb-standalone" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
```
如果成功进入 `mongosh` 命令行，并可以执行 `show dbs;` 等命令看到我们创建的 `test` 数据库，则证明数据库实例功能完全正常。

**d. 集群内部访问地址**

其他部署在同一 K8s 集群内的应用（如您的 Spring Boot 微服务）可以通过以下地址连接到 MongoDB：
```
# 格式: <service-name>.<namespace>.svc.cluster.local:<port>
my-mongodb-standalone.mongodb.svc.cluster.local:27017
```

#### 3. 监控验证

最后，验证我们的可观测性配置是否生效。

1.  **Prometheus Targets**: 访问 Prometheus UI，在 "Status" -> "Targets" 页面，应该能找到一个名为 `serviceMonitor/mongodb/my-mongodb-standalone-metrics` 的条目，且其状态为 `UP`。
2.  **Grafana Dashboard**: 访问 Grafana UI，通过 "Import dashboard" 功能导入官方推荐的 MongoDB Exporter Dashboard，ID 为 `12079` 或 `20867`。如果面板能成功导入并显示出数据，则监控链路完全打通。

## 应用的生命周期管理

### 更新应用

得益于 `helm upgrade --install` 的幂等性，更新应用变得非常简单。例如，若要将持久卷大小从 `16Gi` 调整到 `32Gi`，只需修改 `install.sh` 中 `--set persistence.size=32Gi`，然后 **重新执行** `bash install.sh` 即可。Helm 会智能地计算出变更并应用到集群中。

### 卸载应用

**a. 执行卸载脚本**

一个简单的 `uninstall.sh` 脚本会包含以下命令：
```shell
# uninstall.sh (示例)
source .env
helm uninstall ${RELEASE_NAME} --namespace ${NAMESPACE}
```
执行 `bash uninstall.sh` 会删除所有由 Helm 创建的资源（StatefulSet, Service, Secret 等）。

**b. (可选) 删除 PVC**

**请特别注意**：为了防止数据丢失，Helm 在卸载时默认会保留 `PersistentVolumeClaim` (PVC)。如果您确认不再需要这些数据，需要手动删除它们。

```shell
# 加载变量
source .env

# 查看pvc
kubectl get pvc -n ${NAMESPACE}

# 确认pvc名称后，执行删除
kubectl delete pvc data-my-mongodb-standalone-0 -n ${NAMESPACE}
```

## 总结

通过结合 Helm Chart、配置文件（`.env`）和自动化脚本（`install.sh`），我们实现了一个健壮、可配置且易于管理的 Kubernetes MongoDB 部署方案。该方案不仅覆盖了从安装到卸载的完整生命周期，还内置了与 Prometheus 和 Grafana 的集成，为生产环境提供了必要的全方位支持。这种方法论同样适用于在 Kubernetes 上部署其他任何复杂的有状态或无状态应用，是云原生 DevOps 实践中的典范。