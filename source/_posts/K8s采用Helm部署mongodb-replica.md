---
title: K8s采用Helm部署mongodb-replica
abbrlink: 9d2481ad
date: '2025-06-11T19:28:16.000Z'
tags:
  - 容器化
categories:
  - 容器化
toc: true
---


在现代云原生架构中，将有状态应用（如数据库）容器化并部署在 Kubernetes 上已成为主流趋势。这不仅能带来高可用性、弹性伸缩的优势，还能统一应用的运维管理模式。MongoDB 作为业界领先的 NoSQL 数据库，其副本集（Replica Set）模式是保障数据冗余和高可用的生产标准。

本文将以一个实战项目的视角，详细阐述如何利用 Helm——Kubernetes 的包管理器——在 K8s 集群中快速、规范地部署一套生产可用的 MongoDB 副本集，并集成 Prometheus 监控。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/mongodb/mongodb-replica), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/mongodb/mongodb-replica)

## 为什么选择 Helm 部署 MongoDB？

1.  **标准化与可复用**：Helm Chart 将部署 MongoDB 所需的所有 K8s 资源（StatefulSet, Service, Secret, ConfigMap, ServiceMonitor 等）打包管理，实现了“基础设施即代码”（IaC），极大提升了部署的标准化和可复用性。
2.  **简化复杂性**：部署一个高可用的 MongoDB 副本集涉及复杂的网络配置（如 Headless Service）、节点间认证、持久化存储以及监控集成。Helm 将这些复杂配置抽象为可配置的参数，让我们可以通过简单的变量设置来完成部署。
3.  **生命周期管理**：Helm 不仅负责安装，还简化了应用的升级、回滚和卸载流程，使得 MongoDB 集群的维护变得轻而易举。

## 项目准备

为了实现灵活配置与一键部署，我们将项目结构设计为两个核心文件和一个环境变量文件：

*   `.env`: 存储所有可配置的变量，如命名空间、密码、副本数等。
*   `install.sh`: 核心部署脚本，负责拉取 Helm Chart 并根据 `.env` 中的配置执行安装或升级。
*   `uninstall.sh` (未在文中展示，但与 `README.md` 对应): 负责清理和卸载应用。
*   `README.md`: 项目说明与操作指南。

### Step 1: 核心配置 (`.env`)

这是整个部署的“控制中心”。我们将所有需要定制的参数都提取到这个文件中，以便在不修改部署脚本的情况下，适应不同环境的需求。

```shell
# 命名空间
NAMESPACE="mongodb"
# helm的release名称
RELEASE_NAME="my-mongodb-replica"
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

# --- 监控配置 ---
# Prometheus Operator 所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 ServiceMonitor 的标签值
# 通常是 kube-prometheus-stack 的 release 名称, 你的 kubectl get all 输出中release名称为 kube-prom-stack
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```

**关键配置解读**：

*   `STORAGE_CLASS_NAME`: 指定 MongoDB 数据持久化使用的存储类。请确保您的 K8s 集群中已存在名为 `nfs` (或您自定义的名称) 的 `StorageClass`。
*   `REPLICA_COUNT`: 副本集节点数，生产环境强烈建议为奇数（如3, 5）以保证选举机制正常工作。
*   `MONGO_REPLICA_SET_KEY`: 副本集内部成员间通信的认证密钥，是保障安全的重要配置。**在实际部署前，请务必使用 `openssl rand -base64 756` 命令生成一个新的密钥来替换默认值。**
*   `metrics.*` 相关配置：这部分是与 Prometheus Operator 集成的关键，`PROMETHEUS_RELEASE_LABEL` 需要配置为您的 Prometheus Operator 实例的 `release` 标签值，以便 `ServiceMonitor` 能够被正确发现。

### Step 2: 核心安装脚本 (`install.sh`)

这个脚本封装了 `helm upgrade --install` 命令，它是一个幂等操作：如果 Release 不存在，则安装；如果已存在，则升级。这使得初次安装和后续更新都使用同一命令，非常便捷。

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
  --set podAntiAffinityPreset=soft \
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

**关键 Helm 参数解析**：

*   `--set architecture=replicaset`: 明确指定部署模式为副本集。
*   `--set-string auth.*`: 将 `.env` 中定义的密码、密钥等安全信息传递给 Chart。使用 `--set-string` 可以避免 Helm 对长字符串进行不必要的类型转换。
*   `--set podAntiAffinityPreset=soft`: 设置Pod反亲和性。这是一个非常重要的生产实践，它会“尽力”将 MongoDB 的各个 Pod 调度到不同的物理节点上，从而避免单点故障。
*   `--set resources.*`: 为 MongoDB Pod 和监控 Exporter Pod 设置了明确的 CPU 和内存请求（requests）与限制（limits）。这是保障 K8s 集群资源稳定性和服务质量（QoS）的最佳实践。
*   `--set metrics.enabled=true` 及 `metrics.serviceMonitor.enabled=true`: 启用内置的 Prometheus Exporter Sidecar，并创建一个 `ServiceMonitor` CRD 资源，实现与 Prometheus 的自动集成。

## 部署与验证

遵循 `README.md` 中的指南，我们可以轻松完成部署和验证。

### 1. 安装应用

在项目根目录下，直接执行安装脚本：

```shell
bash install.sh
```

Helm 会开始创建所有必需的 K8s 资源。您可以通过 `kubectl get pods -n mongodb -w` 来观察 Pod 的启动过程。

### 2. 验证应用

#### 初步验证

等待所有 Pod 变为 `Running` 状态后，使用 `helm status my-mongodb-replica -n mongodb` 或 `kubectl get all -n mongodb` 查看所有资源是否都已就绪。

#### 进阶验证：连接数据库

为了确认副本集是否正常工作，我们需要从集群内部连接它。

1.  **获取 Root 密码**：密码被安全地存储在 K8s Secret 中。
    ```shell
    export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace mongodb my-mongodb-replica -o jsonpath="{.data.mongodb-root-password}" | base64 -d)
    ```

2.  **启动一个临时客户端 Pod**：这是在 K8s 中进行内部服务连通性测试的标准方法。
    ```shell
    kubectl run --namespace mongodb my-mongodb-replica-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:8.0.10-debian-12-r1 --command -- bash
    ```

3.  **使用 `mongosh` 连接副本集**：在客户端 Pod 的 shell 中，使用由 Headless Service 提供的稳定网络标识符来连接整个副本集。
    ```shell
    mongosh admin --host "my-mongodb-replica-0.my-mongodb-replica-headless.mongodb.svc.cluster.local:27017,my-mongodb-replica-1.my-mongodb-replica-headless.mongodb.svc.cluster.local:27017,my-mongodb-replica-2.my-mongodb-replica-headless.mongodb.svc.cluster.local:27017" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
    ```
    成功连接并看到副本集的主节点提示符（如 `rs0:PRIMARY>`），表示副本集已成功建立。

#### 监控验证

1.  访问您的 **Prometheus UI**，在 `Status -> Targets` 页面，查找 `mongodb-exporter` 相关的 target，确认其状态为 `UP`。
2.  访问您的 **Grafana**，导入官方推荐的 Dashboard ID `12079` 或 `20867`。如果能看到 MongoDB 的各项性能指标（如连接数、QPS、内存使用率等）被正确渲染，则证明监控通路已完全打通。

## 应用生命周期管理

### 更新
当需要修改配置时（例如，增加副本数、调整资源限制），只需更新 `.env` 文件中的相应变量，然后**重新执行 `install.sh` 脚本**即可。Helm 的幂等性会智能地计算出变更并应用到集群中。

### 卸载
执行卸载脚本 `bash uninstall.sh` (其核心是 `helm uninstall` 命令) 会删除所有由 Helm 创建的资源。

**特别注意**：默认情况下，为了防止数据丢失，Helm 不会删除与 StatefulSet 关联的持久卷声明（PVC）。如果需要彻底清理（包括数据），您需要手动删除这些 PVC。
```shell
# 查看PVC
kubectl get pvc -n mongodb

# 删除PVC (示例)
kubectl delete pvc data-my-mongodb-replica-0 -n mongodb
```

## 总结

通过将 Helm Chart 与清晰的配置文件、自动化的部署脚本相结合，我们构建了一套健壮、可重复且易于维护的 MongoDB 副本集部署方案。该方案不仅覆盖了高可用性、安全性、持久化等生产核心要素，还无缝集成了云原生监控体系。这套方法论充分体现了 DevOps 的精神，将复杂的数据库运维工作转变为简单、可靠的代码化流程，为基于 Kubernetes 的数据驱动型应用提供了坚实的基础。