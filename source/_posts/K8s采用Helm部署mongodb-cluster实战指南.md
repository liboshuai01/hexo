---
title: K8s采用Helm部署mongodb-cluster实战指南
tags:
  - Linux
  - K8s
  - Helm
  - Mongodb
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250607212309238.png'
toc: true
abbrlink: 8d4c3fee
date: 2025-06-07 21:21:40
---

作为后端开发人员，我们深知一个稳定、可扩展且高可用的数据库是现代应用程序的基石。在基于 Kubernetes 的微服务架构中，手动管理数据库集群既繁琐又容易出错。幸运的是，我们可以借助 Helm 这一强大的包管理工具，实现 MongoDB 副本集（Replica Set）的自动化部署和生命周期管理。

本文将为您提供一份端到端的实战指南，通过一套精心设计的脚本和配置文件，在 Kubernetes 上快速部署一个生产级的 MongoDB 副本集。

<!-- more -->

## 核心优势

*   **高可用性**: 通过部署副本集（Replica Set）并启用 Pod 反亲和性（Anti-Affinity），确保集群在节点故障时依然可用。
*   **自动化**: 使用 Helm Chart 和 Shell 脚本，将部署、更新和卸载流程完全自动化。
*   **可配置性**: 将所有可变配置项抽离到 `.env` 文件中，轻松适应不同环境。
*   **数据持久化**: 利用 Kubernetes 的持久卷（Persistent Volumes）确保数据在 Pod 重启或迁移后不丢失。

## 前提准备

在开始之前，请确保您已具备以下环境：

1.  一个正在运行的 Kubernetes 集群。
2.  `kubectl` 命令行工具已配置并能够连接到您的集群。
3.  已安装 Helm v3。
4.  集群中已配置好一个可用的 `StorageClass` 用于动态创建持久卷（本指南示例使用 `nfs`）。

## 项目文件结构

我们将采用以下文件结构来组织我们的部署项目，实现配置与逻辑的分离。

```
.
├── .env           # 环境变量与配置中心
├── install.sh     # 安装与更新脚本
├── status.sh      # 状态检查脚本
└── uninstall.sh   # 卸载脚本
```

下面我们来详细解读每个文件的内容。

### 1. 配置中心：`.env`

这个文件是所有可配置参数的集合。在部署前，您应该根据您的环境和需求修改此文件。

> .env
```
# 命名空间
NAMESPACE="mongo"
# helm的release名称
RELEASE_NAME="my-mongo-cluster"
# helm chart版本
CHART_VERSION="16.5.20"
# 存储类名称
STORAGE_CLASS_NAME="nfs"

# --- 副本集特定配置 ---
# 副本集名称
MONGO_REPLICA_SET_NAME="rs0"
# 副本集的节点数量 (建议为奇数，例如 3 或 5)
REPLICA_COUNT=3
# 副本集内部通信认证密钥 (必须提供，用于节点间安全通信)
# 请使用下面的命令生成一个安全的密钥并替换它：
# openssl rand -base64 756
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

# --- 认证和数据库配置 (与单机版相同) ---
# MongoDB root用户密码
MONGO_ROOT_PASSWORD="YOUR_PASSWORD"
# MongoDB 应用数据库名称
MONGO_DATABASE="test"
# MongoDB 应用用户名称
MONGO_USER="lbs"
# MongoDB 应用用户密码
MONGO_PASSWORD="YOUR_PASSWORD"
```
**关键配置说明:**
*   `MONGO_REPLICA_SET_KEY`: 副本集成语之间通信的共享密钥，保障内部通信安全。**在生产环境中，务必使用 `openssl rand -base64 756` 命令重新生成一个新的密钥。**
*   `REPLICA_COUNT`: 副本集节点数，建议设为3、5等奇数，以保证选举机制正常工作。
*   `STORAGE_CLASS_NAME`: 必须是您 K8s 集群中真实存在的 StorageClass 名称。
*   `MONGO_ROOT_PASSWORD` / `MONGO_PASSWORD`: 请务必修改为强密码。

### 2. 部署脚本：`install.sh`

这个脚本是部署的核心。它会读取 `.env` 文件的配置，通过 `helm upgrade --install` 命令来安装或更新 MongoDB 集群。

> install.sh
```shell
#!/usr/bin/env bash

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
  --set architecture=replicaset \
  --set replicaCount=${REPLICA_COUNT} \
  --set replicaSetName="${MONGO_REPLICA_SET_NAME}" \
  --set-string global.storageClass="${STORAGE_CLASS_NAME}" \
  \
  --set-string auth.replicaSetKey="${MONGO_REPLICA_SET_KEY}" \
  --set-string auth.rootPassword="${MONGO_ROOT_PASSWORD}" \
  --set-string auth.databases[0]="${MONGO_DATABASE}" \
  --set-string auth.usernames[0]="${MONGO_USER}" \
  --set-string auth.passwords[0]="${MONGO_PASSWORD}" \
  \
  --set podAntiAffinityPreset=hard \
  \
  --set persistence.size=16Gi \
  \
  --set resources.requests.cpu=250m \
  --set resources.requests.memory=512Mi \
  --set resources.limits.cpu=1000m \
  --set resources.limits.memory=2048Mi

```
**关键 Helm 参数解析:**
*   `helm upgrade --install`: 一个非常实用的命令，如果 Release 不存在，则执行安装；如果已存在，则执行升级。这使得脚本可以重复执行，实现了声明式部署。
*   `--create-namespace`: 如果命名空间不存在，则自动创建。
*   `architecture=replicaset`: 明确告诉 Helm Chart 我们要部署的是一个副本集架构，而不是单机版。
*   `podAntiAffinityPreset=hard`: **这是一个至关重要的生产级配置**。它强制 Kubernetes 调度器将 MongoDB 的各个 Pod 分散到不同的物理节点上。这样，即使某个节点宕机，其他节点上的副本依然存活，保障了服务的高可用性。
*   `persistence.size`: 为每个副本分配的持久化存储空间大小。
*   `resources`: 为 Pod 设置的 CPU 和内存的请求（requests）与限制（limits），是保障服务稳定性的关键 QoS 配置。

### 3. 状态检查脚本：`status.sh`

部署完成后，使用此脚本快速查看集群中相关的资源状态。

> status.sh
```shell
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    source .env
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

kubectl get all -n ${NAMESPACE}
kubectl get pvc -n ${NAMESPACE}

```
它会列出 `mongo` 命名空间下的所有 Pod、Service、StatefulSet 等资源，以及持久卷声明（PVC），方便我们进行初步的健康检查。

### 4. 卸载脚本：`uninstall.sh`

用于清理通过 Helm 部署的 MongoDB 集群。

> uninstall.sh
```shell
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    source .env
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}

```
注意：`helm uninstall` 默认不会删除 PVC，这是为了防止数据被误删。数据的清理需要手动操作，我们将在后续步骤中说明。

## 部署与验证流程

以下是完整的部署、验证和管理流程。

### 步骤一：修改配置并执行安装

1.  打开 `.env` 文件，根据您的实际需求修改其中的变量，特别是密码和 StorageClass。
2.  在终端中执行安装脚本：
    ```shell
    bash install.sh
    ```
    Helm 将开始在您的 K8s 集群中创建所有必要的资源。这个过程可能需要几分钟，因为要拉取镜像并等待 Pod 启动和副本集初始化。

### 步骤二：初步验证

等待安装命令执行完毕后，运行状态检查脚本：

```shell
bash status.sh
```

您应该能看到类似下面的输出（以 `REPLICA_COUNT=3` 为例）：
*   3 个 `my-mongo-cluster-mongodb-x` 的 Pod 处于 `Running` 状态。
*   1 个名为 `my-mongo-cluster-mongodb-headless` 的 Headless Service。
*   1 个名为 `my-mongo-cluster-mongodb` 的 StatefulSet。
*   3 个状态为 `Bound` 的 PVC。

### 步骤三：进阶验证（连接测试）

为了最终确认副本集工作正常，我们将启动一个临时的 MongoDB 客户端 Pod，并连接到集群内部。

**1. 获取 root 用户密码**

密码被安全地存储在 Kubernetes Secret 中。我们通过以下命令获取并解码它：

```shell
export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace mongo my-mongo-cluster-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 -d)
```

**2. 启动 MongoDB 客户端 Pod**

这个命令会创建一个临时的 Pod，并进入其交互式 Shell：

```shell
kubectl run --namespace mongo my-mongo-cluster-mongodb-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:8.0.10-debian-12-r1 --command -- bash
```

**3. 连接 MongoDB 副本集**

在客户端 Pod 的 Shell 中，执行以下命令。注意这个连接字符串的格式，它包含了所有副本集成员的地址，这是副本集连接的标准方式。

```shell
mongosh admin --host "my-mongo-cluster-mongodb-0.my-mongo-cluster-mongodb-headless.mongo.svc.cluster.local:27017,my-mongo-cluster-mongodb-1.my-mongo-cluster-mongodb-headless.mongo.svc.cluster.local:27017,my-mongo-cluster-mongodb-2.my-mongo-cluster-mongodb-headless.mongo.svc.cluster.local:27017" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
```

如果一切正常，您将看到 MongoDB Shell 的提示符，并且提示符中会显示副本集名称和当前节点角色（例如 `rs0 [primary] >`），这表明您已成功连接到副本集的主节点！

## 应用生命周期管理

### 更新应用

当您需要修改配置时（例如，增加存储空间 `persistence.size` 或调整 `resources`），只需：

1.  修改 `.env` 或 `install.sh` 文件中的相应参数。
2.  重新执行 `bash install.sh`。

Helm 会智能地计算出变更，并对集群资源进行滚动更新，确保服务的连续性。

### 卸载应用

**1. 执行卸载脚本**

这将删除由 Helm 创建的所有 K8s 资源（Pod, Service, StatefulSet, Secret 等）。

```shell
bash uninstall.sh
```

**2. （可选）删除 PVC**

如前所述，为保护数据，PVC 不会被自动删除。如果您确认不再需要这些数据，可以手动删除它们。

```shell
# 加载变量
source .env

# 查看pvc
kubectl get pvc -n ${NAMESPACE}

# 逐个删除pvc（请将[pvc名称]替换为实际名称）
kubectl delete pvc data-my-mongo-cluster-mongodb-0 -n ${NAMESPACE}
kubectl delete pvc data-my-mongo-cluster-mongodb-1 -n ${NAMESPACE}
kubectl delete pvc data-my-mongo-cluster-mongodb-2 -n ${NAMESPACE}
```
**警告：删除 PVC 将导致其关联的持久化数据永久丢失，请谨慎操作！**

## 总结

通过遵循本指南，您已经掌握了利用 Helm 和脚本在 Kubernetes 上部署和管理高可用 MongoDB 副本集的核心技能。这种基础设施即代码（IaC）的方法不仅极大地提升了部署效率和可靠性，还使得整个流程标准化、可重复，是云原生时代后端开发与运维的必备实践。现在，您可以放心地将这个强大的数据库集群集成到您的下一代可扩展 Web 应用中了。