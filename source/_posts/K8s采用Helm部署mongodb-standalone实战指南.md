---
title: K8s采用Helm部署mongodb-standalone实战指南
date: 2025-06-07 20:54:39
tags:
  - Linux
  - K8s
  - Helm
  - Mongodb
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250607205732464.png'
toc: true
---

在现代后端开发中，将数据库等有状态应用部署在 Kubernetes (K8s) 上已成为主流选择。Kubernetes 提供了强大的编排能力，而 Helm 作为其官方包管理器，极大地简化了复杂应用的部署和生命周期管理。

本文将提供一个详细的实战指南，介绍如何使用 Helm 和一组标准化脚本，在 K8s 集群上快速、可靠地部署一个单节点（Standalone）的 MongoDB 实例。这种方法通过将配置与脚本分离，实现了高度的可复现性和易维护性。

<!-- more -->

## 一、项目结构概览

为了实现标准化部署，我们采用以下文件结构：

-   `.env`: 存储所有可配置的变量，如命名空间、密码和资源名称。
-   `install.sh`: 核心安装/升级脚本，负责执行 Helm 命令。
-   `status.sh`: 用于快速检查部署后应用的状态。
-   `uninstall.sh`: 用于卸载 Helm 应用。
-   `README.md`: 项目的说明文档（本文正是基于其内容扩展而来）。

这种结构使得任何开发人员都可以通过修改 `.env` 文件并执行相应脚本来完成部署，而无需深入了解 Helm 命令的每一个细节。

## 二、前提准备

在开始之前，请确保您已具备以下环境：

1.  一个正在运行的 Kubernetes 集群。
2.  已安装并配置好 `kubectl` 命令行工具，且能正常连接到您的集群。
3.  已安装 Helm v3。

部署的核心配置存放于 `.env` 文件中。在执行安装前，请根据您的环境和需求修改此文件。

> **.env**
```
# 命名空间
NAMESPACE="mongo"
# helm的release名称
RELEASE_NAME="my-mongo-standalone"
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

```
**关键配置说明：**
*   `NAMESPACE`: 定义应用部署的 K8s 命名空间。
*   `RELEASE_NAME`: Helm Release 的名称，用于标识此次部署。
*   `CHART_VERSION`: 锁定 Bitnami MongoDB Chart 的版本，确保部署的一致性。
*   `STORAGE_CLASS_NAME`: 指定用于持久化存储的 StorageClass。请确保您的 K8s 集群中存在该 StorageClass。
*   `MONGO_..._PASSWORD`: **请务必修改为您自己的强密码！**

## 三、安装应用

我们使用 `install.sh` 脚本来执行部署。该脚本会自动加载 `.env` 变量，添加 Bitnami 的 Helm 仓库，并执行 `helm upgrade --install` 命令。

`helm upgrade --install` 是一个幂等操作：如果应用尚未安装，它会执行安装；如果已安装，它会根据新的配置进行升级。这使得初次安装和后续更新都使用同一套流程。

> **install.sh**
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
  --set resources.requests.cpu=250m \
  --set resources.requests.memory=512Mi \
  --set resources.limits.cpu=2000m \
  --set resources.limits.memory=4096Mi

```
**脚本核心参数解析：**
*   `--create-namespace`: 如果命名空间不存在，则自动创建。
*   `--set architecture=standalone`: 指明我们部署的是单节点架构。
*   `--set useStatefulSet=true`: **关键！** 对于数据库等有状态应用，应使用 `StatefulSet` 来确保 Pod 拥有稳定的网络标识和持久化存储。
*   `--set-string auth.*`: 在 MongoDB 首次初始化时，自动创建 root 用户、指定的应用数据库和用户。
*   `--set persistence.size`: 设置持久卷（PV）的大小。
*   `--set resources.*`: 为 MongoDB Pod 设置资源请求（requests）和限制（limits），这是保障 K8s 集群稳定性的重要实践。

要开始安装，只需在终端中执行：
```shell
bash install.sh
```

## 四、部署验证

部署完成后，我们需要验证 MongoDB 是否正常运行。

### 4.1 初步状态检查

`status.sh` 脚本可以快速列出在指定命名空间中创建的所有相关资源，包括 Pod、Service 和 PVC (PersistentVolumeClaim)。

> **status.sh**
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

执行脚本：
```shell
bash status.sh
```
您应该能看到一个名为 `my-mongo-standalone-mongodb-0` 的 Pod 处于 `Running` 状态，并且对应的 PVC 处于 `Bound` 状态。

### 4.2 进阶连接测试

为了最终确认数据库可用，我们将启动一个临时的 MongoDB 客户端 Pod，并尝试连接到我们刚刚部署的实例。

> **README.md 中提供的进阶验证步骤**

**1. 获取 root 用户密码**

MongoDB 的 root 密码被安全地存储在 K8s Secret 中。我们首先需要获取它：
```shell
export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace mongo my-mongo-standalone-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 -d)
```
*此命令会从 Secret 中提取密码，进行 Base64 解码，并将其设置为环境变量。*

**2. 启动 MongoDB 客户端 Pod**

我们使用 `kubectl run` 创建一个一次性的客户端 Pod，并进入其 shell 环境：
```shell
kubectl run --namespace mongo my-mongo-standalone-mongodb-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:8.0.10-debian-12-r1 --command -- bash
```
*   `--rm`: Pod 退出后自动删除。
*   `-i --tty`: 提供一个交互式终端。
*   `--restart='Never'`: 这是一个临时任务，不需要重启。
*   `--env`: 将上一步获取的密码注入到客户端 Pod 的环境变量中。

**3. 连接 MongoDB**

在客户端 Pod 的 shell 中，执行以下命令连接数据库：
```shell
mongosh admin --host "my-mongo-standalone-mongodb" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
```
*   `--host`: `my-mongo-standalone-mongodb` 是 Helm Chart 创建的 Service 名称，它为我们的 Pod 提供了稳定的内部 DNS 记录。
*   如果连接成功并看到 MongoDB shell 提示符，则证明部署完全成功！

## 五、更新应用

如果需要修改配置，例如增加存储空间或调整 CPU/Memory 限制，只需：
1.  修改 `.env` 文件或 `install.sh` 中的 `--set` 参数。
2.  重新执行 `install.sh` 脚本。

```shell
bash install.sh
```
Helm 将会自动计算差异并只更新需要变更的部分，这就是声明式管理的魅力所在。

## 六、卸载应用

当不再需要该 MongoDB 实例时，可以进行清理。

### 6.1 执行卸载脚本

`uninstall.sh` 脚本会调用 `helm uninstall` 来移除所有由该 Chart 创建的 K8s 资源（Pod, Service, StatefulSet, Secret等）。

> **uninstall.sh**
```shell
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; a
    source .env
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}
```

执行卸载：
```shell
bash uninstall.sh
```

### 6.2 （可选）删除 PVC

**重要提示：** 为了防止数据丢失，Helm 默认不会删除与应用关联的 `PersistentVolumeClaim` (PVC)。这意味着即使你卸载了应用，数据仍然保留在持久卷中。如果你确认不再需要这些数据，可以手动删除 PVC。

> **README.md 中提供的 PVC 清理步骤**
```shell
# 加载变量
source .env

# 查看pvc
kubectl get pvc -n ${NAMESPACE}

# 删除pvc（根据上一步查到的名称进行替换）
kubectl delete pvc data-my-mongo-standalone-mongodb-0 -n ${NAMESPACE}
```
**请谨慎操作，此步骤将永久删除所有 MongoDB 数据！**

## 总结

通过本文介绍的脚本化和配置化方法，我们实现了一个强大且易于管理的 K8s MongoDB 部署流程。这种方式不仅降低了部署的复杂度，还通过版本化的 Chart 和显式的配置文件，确保了环境的一致性和可追溯性。作为后端开发人员，掌握这类云原生部署技能将极大地提升开发和运维效率。