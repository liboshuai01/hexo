---
title: K8s采用Helm部署mongodb-sharded
abbrlink: 4463c703
date: 2025-06-11 19:22:50
tags:
  - Linux
  - K8s
  - Helm
  - Mongodb
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250611192424146.png'
toc: true
---

在构建高性能、数据驱动的后端系统时，数据库的选择与部署是至关重要的一环。MongoDB 作为一个灵活、可扩展的 NoSQL 数据库，其分片集群（Sharded Cluster）架构能够为海量数据提供出色的水平扩展能力和高可用性。然而，手动部署和管理一个完整的分片集群（包含 Config Servers, Shards, Mongos Routers）相当复杂。

幸运的是，借助 Kubernetes (K8s) 的声明式能力和 Helm 的包管理机制，我们可以将这个复杂的过程自动化、标准化。本文将详细介绍如何使用 Bitnami 提供的 Helm Chart，通过一套可配置、可复用的脚本，在 K8s 上高效、可靠地部署一个生产级的 MongoDB Sharded Cluster。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/mongodb/mongodb-sharded), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/mongodb/mongodb-sharded)

## 核心优势

采用此方案，您将获得：

*   **基础设施即代码 (IaC)**：所有配置和部署逻辑均通过代码（`.env` 文件和 shell 脚本）管理，实现版本控制和可重复部署。
*   **高度可配置**：通过一个简单的 `.env` 文件即可调整集群的命名空间、认证信息、分片数量、副本数、存储等核心参数。
*   **生产级特性**：该方案默认启用了认证、副本集、资源限制，并集成了 Prometheus 监控，为生产环境打下坚实基础。
*   **简化运维**：使用简单的命令即可完成安装、升级、验证和卸载，极大降低了管理 MongoDB 分片集群的复杂度。

## 项目结构概览

我们的部署项目主要由以下几个文件构成：

*   `.env`: 核心配置文件，用于定义所有可变参数。
*   `install.sh`: 安装和升级应用的主脚本。
*   `status.sh`: 用于快速检查部署状态的辅助脚本（内容未展示，通常包含`kubectl get pods`等命令）。
*   `uninstall.sh`: 卸载应用的主脚本。

## 第一步：环境配置 (`.env`)

在开始之前，我们需要创建一个 `.env` 文件来集中管理我们所有的配置。这种方式将配置与执行逻辑分离，使得调整部署参数变得非常清晰和安全。

```shell
# 命名空间
NAMESPACE="mongodb"
# helm的release名称
RELEASE_NAME="my-mongodb-sharded"
# helm chart版本 (请确认适用于mongodb-sharded的chart版本，例如 5.0.3)
CHART_VERSION="9.3.5"
# 存储类名称
STORAGE_CLASS_NAME="nfs"

# --- 全局认证配置 ---
# MongoDB root用户密码 (用于mongos)
MONGO_ROOT_PASSWORD="YOUR_PASSWORD"
# 副本集内部通信认证密钥 (用于Config Server副本集和每个Shard副本集内部节点认证)
MONGO_REPLICA_SET_KEY="k/ftJNlwgcIFGjx2GHujlV4yFr9Ee5Qwq59EpfZWNozs/MsSu7BsNOFKDKA2TWmV
                       fzuJ3ybYGajZCt7Vst7Qyff3O3NNOG7/jqLNmUE0x2LN10lD6tARmdCk1WofuPaK
                       bs8uiesiVk+dFXc8mIRlWhuB4WO420FTyHsWlxSVZvV7UNqZcy/oPKk8MuZZuJvQ
                       JhkZNW6YeLK7Pn0IOqaGCeRZZNbnDieC9MBf3OEvThoirCsheSpQYiAE1Dp/zDCn
                       f5E0W9uBB/MiXgiHiPqBxjlmC2RkBtYSYDHYzNij41DN+38sdzyHixzE02mWoUU/
                       Wp8BK2i1tkoWhtbtBl2h09cj0xj/43/7rK8pjL625/jooMoD0j3WLsdPV20jYGXy
                       Nx8/7q05a+wSqoo7tU/TCECbtskdgH4c/GY9PdOdNnFenOGvL1TCsCnLJS3jE6ts

                       Rj+mvKy9xqsAzde+QKbHY7CxQP6Aah5zx400LIjQELYDeqXKG5Xmt1jfKWh1uZQv
                       cD0bDUVJbcNJGe6UUY//0D9woqslcDLZl55lWB/AL7Ndl700s1SFzaDUSmwYjYqF
                       vnYkJoU1PVzTrKu8K7mGzBBRgS97+FXHLvt1y420+AKgHQeFeYU1x8qA3P0Xz7Lb
                       5nDz5mb2IUUwV4bYJjueP+Ixixr78aqYIYKyHtCcKSsVzhAzxA2ycoWDWYzd49em
                       dJrtnWMj5ZNTpXzR4dVm2Br55p7TZRQtVGXC+2nqj+jQ/Icx1rZ7DA0U/n8ne1n0
                       Y/p/0mhbm72lN2Jap0PEf69dWpsiqIRIDtm+hbWMvdwGVpsg0wU6uTklCE7E5U52
                       +YlJzVKjc9+AyKigpd69Igc1A6G25/Nq8eneeBsKwbVSKwyaYaJshtdi77U+xDfL
                       E0fKN0rg787ItQC8cgGPGBZQKZ60TlkoxmbBDD6dIlmA24EQoBq95GgLFyQeWkOn
                       /o3BKkLqGQPmy9qX2eRW+W/MHIkfHvkT2T8NKJLZkpvfjHNQ"

# --- 分片集群架构配置 ---
# Shard (分片) 的数量
SHARD_COUNT=2
# 每个Shard副本集中的数据节点数量 (建议为奇数, e.g., 3)
SHARDSVR_REPLICA_COUNT=3
# Config Server (配置服务器) 副本集的节点数量 (生产环境强烈建议为 3)
CONFIGSVR_REPLICA_COUNT=3
# Mongos (查询路由) 的实例数量
MONGOS_REPLICA_COUNT=2

# --- 监控配置 ---
# Prometheus Operator 所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 PodMonitor 的标签值
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```

**关键配置解析：**

*   **基础配置**: 定义了部署的K8s `NAMESPACE`、Helm `RELEASE_NAME`、Chart版本以及PVC使用的`STORAGE_CLASS_NAME`。
*   **认证配置**:
    *   `MONGO_ROOT_PASSWORD`: 这是您连接到集群的 `root` 用户密码，务必修改为一个强密码。
    *   `MONGO_REPLICA_SET_KEY`: 这是副本集内部节点间通信认证的密钥。使用一个长而随机的字符串是安全最佳实践，可以由 `openssl rand -base64 756` 生成。
*   **架构配置**: 这是分片集群的核心。
    *   `SHARD_COUNT`: 定义分片的数量，决定了数据水平扩展的粒度。
    *   `SHARDSVR_REPLICA_COUNT`: 每个分片内部的副本数。设置为 `3` 可以保证分片级别的高可用。
    *   `CONFIGSVR_REPLICA_COUNT`: 配置服务器副本数，存储集群元数据，生产环境强烈建议为 `3`。
    *   `MONGOS_REPLICA_COUNT`: 查询路由 `mongos` 的实例数，是应用的入口，可以根据查询负载进行扩展。
*   **监控配置**: 用于与 Prometheus Operator 集成。`PROMETHEUS_NAMESPACE` 和 `PROMETHEUS_RELEASE_LABEL` 需要与您环境中已安装的 Prometheus Operator 配置相匹配，这样才能自动发现并抓取 MongoDB 的监控指标。

## 第二步：安装脚本详解 (`install.sh`)

这个脚本是部署的核心，它读取 `.env` 配置，并执行 `helm upgrade --install` 命令。这个命令是幂等的，意味着无论集群是首次安装还是需要更新，都可以安全地反复执行。

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

# --- 安装 / 升级 MongoDB Sharded Cluster ---
helm upgrade --install ${RELEASE_NAME} bitnami/mongodb-sharded --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --create-namespace \
  \
  --set-string global.storageClass="${STORAGE_CLASS_NAME}" \
  --set auth.enabled=true \
  --set-string auth.rootPassword="${MONGO_ROOT_PASSWORD}" \
  --set-string auth.replicaSetKey="${MONGO_REPLICA_SET_KEY}" \
  \
  --set shards=${SHARD_COUNT} \
  \
  --set mongos.replicaCount=${MONGOS_REPLICA_COUNT} \
  --set mongos.podAntiAffinityPreset=soft \
  --set mongos.resources.requests.cpu=100m \
  --set mongos.resources.requests.memory=128Mi \
  --set mongos.resources.limits.cpu=512m \
  --set mongos.resources.limits.memory=2048Mi \
  \
  --set configsvr.replicaCount=${CONFIGSVR_REPLICA_COUNT} \
  --set configsvr.podAntiAffinityPreset=soft \
  --set configsvr.persistence.enabled=true \
  --set configsvr.persistence.size=8Gi \
  --set configsvr.resources.requests.cpu=100m \
  --set configsvr.resources.requests.memory=128Mi \
  --set configsvr.resources.limits.cpu=512m \
  --set configsvr.resources.limits.memory=2048Mi \
  \
  --set shardsvr.dataNode.replicaCount=${SHARDSVR_REPLICA_COUNT} \
  --set shardsvr.dataNode.podAntiAffinityPreset=soft \
  --set shardsvr.persistence.enabled=true \
  --set shardsvr.persistence.size=16Gi \
  --set shardsvr.dataNode.resources.requests.cpu=100m \
  --set shardsvr.dataNode.resources.requests.memory=128Mi \
  --set shardsvr.dataNode.resources.limits.cpu=512m \
  --set shardsvr.dataNode.resources.limits.memory=2048Mi \
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

**脚本关键指令解析：**

*   `source .env`: 加载我们的配置文件。
*   `helm upgrade --install ...`:
    *   `--create-namespace`: 如果命名空间不存在，会自动创建。
    *   `--set-string ...` 和 `--set ...`: 这是 Helm 的核心功能，用于将 shell 变量动态地传递给 Chart 的 `values.yaml`。我们通过这种方式注入了认证、存储、架构、资源和监控的所有配置。
    *   `podAntiAffinityPreset=soft`: 设置软反亲和性，K8s会尽量将副本调度到不同的节点上，提高物理可用性。
    *   `persistence.enabled=true`: 为数据节点和配置节点启用持久化存储（PVC）。
    *   `metrics.serviceMonitor.enabled=true`: 创建一个 `ServiceMonitor` CRD 实例，允许 Prometheus Operator 自动发现和抓取监控数据。

## 第三步：部署与验证

现在，一切准备就绪。按照以下步骤进行部署和验证。

### 1. 安装应用

在项目根目录下，确保您的 `.env` 文件已按需修改，然后执行安装脚本：

```shell
bash install.sh
```

Helm 会开始创建所有必要的 K8s 资源，包括 `StatefulSet` (用于 Shard-Svr 和 Config-Svr)、`Deployment` (用于 Mongos)、`Service`、`Secret` 和 `PVC`。

### 2. 初步验证

等待几分钟，让所有 Pod 进入 `Running` 状态。您可以执行 `status.sh` 或手动检查 Pod 状态：

```shell
# 假设 status.sh 包含以下内容
kubectl get pods -n mongodb -w
```

### 3. 进阶验证：连接数据库

为了确认集群功能正常，我们可以启动一个临时的 MongoDB客户端 Pod 并连接到集群。

**a. 获取 root 密码**

密码被安全地存储在 K8s Secret 中。使用以下命令获取并导出为环境变量：

```shell
export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace mongodb my-mongodb-sharded -o jsonpath="{.data.mongodb-root-password}" | base64 -d)
```

**b. 启动客户端 Pod**

```shell
kubectl run --namespace mongodb my-mongodb-sharded-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:8.0.10-debian-12-r1 --command -- bash
```

此命令会启动一个临时的 Pod，并进入其 shell 环境。`--rm` 标志确保在退出 shell 后该 Pod 会被自动删除。

**c. 连接 Mongos**

在客户端 Pod 的 shell 中，使用 `mongosh` 连接到 `mongos` 服务：

```shell
mongosh admin --host my-mongodb-sharded --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
```

如果连接成功并看到 MongoDB 的 shell 提示符，说明您的分片集群已成功部署并可接受连接！

**d. 集群内部访问地址**

在 K8s 集群内部，其他应用可以通过以下 DNS 地址连接到 MongoDB Sharded 集群：

```
# 格式: <service>.<namespace>.svc.cluster.local:<port>
my-mongodb-sharded.mongodb.svc.cluster.local:27017
```

### 4. 监控验证

如果您的集群中已部署 Prometheus Operator，可以进行以下验证：

1.  访问 Prometheus UI 的 `/targets` 页面，检查 `mongodb-exporter` 相关的 target 是否处于 `UP` 状态。
2.  访问 Grafana，导入官方推荐的 `mongodb-exporter` Dashboard (例如 ID: `12079` 或 `20867`)，查看集群的各项性能指标是否正常显示。

## 第四步：应用的更新与卸载

### 更新应用

得益于 Helm 和 `.env` 的设计，更新应用配置（例如增加分片数量、调整资源限制）变得非常简单：

1.  修改 `.env` 文件中对应的变量值。
2.  重新执行 `install.sh` 脚本。

Helm 会智能地计算出变更，并对 K8s 集群执行滚动更新。

### 卸载应用

**1. 执行卸载脚本**

```shell
bash uninstall.sh
```

该脚本通常包含 `helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}` 命令，它会删除所有由 Helm 创建的 K8s 资源。

**2. （可选）手动删除持久卷声明 (PVC)**

为了防止数据丢失，Helm 默认不会删除 PVC。如果您确认不再需要这些数据，可以手动删除。

```shell
# 加载变量
source .env

# 查看与此部署相关的PVC
kubectl get pvc -n ${NAMESPACE}

# 逐个删除PVC (请谨慎操作！)
# kubectl delete pvc <pvc-name-1> -n ${NAMESPACE}
# kubectl delete pvc <pvc-name-2> -n ${NAMESPACE}
# ...
```

## 总结

通过本文介绍的方法，我们利用 Helm Chart 和一套标准化的脚本，成功地在 Kubernetes 上部署了一个功能完备、配置灵活且具备生产级特性的 MongoDB Sharded Cluster。这种基础设施即代码（IaC）的实践不仅极大地提升了部署效率和可靠性，也为后续的集群管理、扩展和监控奠定了坚实的基础。对于任何需要构建可扩展、数据驱动的后端系统的团队来说，这都是一套值得借鉴和采纳的优秀实践。