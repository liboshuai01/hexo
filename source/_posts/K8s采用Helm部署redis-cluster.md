---
title: K8s采用Helm部署redis-cluster
tags:
  - Linux
  - K8s
  - Helm
  - Redis
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250611190214435.png'
toc: true
abbrlink: 2a489a94
date: 2025-06-11 19:01:53
---

在现代云原生架构中，Redis 以其卓越的性能成为缓存、消息队列和会话存储的首选方案。然而，在 Kubernetes 环境中部署一个高可用的 Redis 集群并非易事，它涉及到状态管理、节点发现、配置一致性和故障转移等复杂问题。幸运的是，[Helm](https://helm.sh/) 作为 Kubernetes 的包管理器，极大地简化了这一过程。

本文将提供一个完整且生产就绪的指南，介绍如何使用 Bitnami 的 Helm Chart 在 Kubernetes 集群上快速部署一个高可用、可监控的 Redis Cluster。我们将采用一种结构化的方法，通过配置文件 (`.env`) 和部署脚本 (`install.sh`) 将配置与执行逻辑分离，实现标准化、可重复的部署。

<!-- more -->

> 项目源码: [github](ht/tps://github.com/liboshuai01/k8s-stack/tree/master/redis/redis-cluster), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/redis/redis-cluster)

## 一、环境与项目准备

在开始之前，请确保您的环境中已安装 `kubectl` 和 `helm` 命令行工具，并已正确配置好对目标 Kubernetes 集群的访问权限。

为了更好地管理部署，我们创建一个专门的项目目录，并包含两个核心文件：
1.  `.env`：用于存放所有可配置的变量，如命名空间、密码、资源规格等。
2.  `install.sh`：部署执行脚本，负责从 `.env` 加载配置并执行 Helm 命令。

### **1. 配置 .env 文件**

将配置外部化到 `.env` 文件是一个非常好的实践，它使得我们的部署脚本更加通用，便于在不同环境（开发、测试、生产）中复用，只需切换不同的 `.env` 文件即可。

这是我们的 `.env` 配置文件内容：
> .env
```shell
# 命名空间
NAMESPACE="redis"
# helm的release名称
RELEASE_NAME="my-redis-cluster"
# helm的chart版本
CHART_VERSION="12.0.8"
# 存储类名称
STORAGE_CLASS_NAME="nfs"
# redis密码
REDIS_PASSWORD="YOUR_PASSWORD"

# Prometheus 监控组件所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 ServiceMonitor 的标签值 (通常是 helm release 的名称)
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```
**关键变量解析：**
*   `NAMESPACE` & `RELEASE_NAME`：定义了 Redis Cluster 将被安装在哪个命名空间以及 Helm Release 的名称，便于资源隔离和管理。
*   `CHART_VERSION`：锁定 Chart 版本，确保了部署的可重复性，避免因 Chart 版本更新导致非预期的变更。
*   `STORAGE_CLASS_NAME`：指定 Kubernetes 的 StorageClass，用于为 Redis 节点动态创建持久化存储（PV/PVC）。
*   `REDIS_PASSWORD`：为集群设置访问密码，是生产环境安全的基石。
*   `PROMETHEUS_*` 变量：用于与 Prometheus Operator 监控体系集成，后面会详细介绍。

### **2. 编写 install.sh 部署脚本**

此脚本是整个部署流程的核心，它封装了所有 Helm 操作，实现了自动化部署。

> install.sh
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
helm upgrade --install "${RELEASE_NAME}" bitnami/redis-cluster \
  --version "${CHART_VERSION}" --namespace "${NAMESPACE}" --create-namespace \
  --set-string global.storageClass="${STORAGE_CLASS_NAME}" \
  --set-string global.redis.password="${REDIS_PASSWORD}" \
  \
  --set persistence.size=8Gi \
  \
  --set redis.resources.requests.cpu=100m \
  --set redis.resources.requests.memory=128Mi \
  --set redis.resources.limits.cpu=512m \
  --set redis.resources.limits.memory=2048Mi \
  \
  --set updateJob.resources.requests.cpu=100m \
  --set updateJob.resources.requests.memory=128Mi \
  --set updateJob.resources.limits.cpu=512m \
  --set updateJob.resources.limits.memory=2048Mi \
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
**脚本关键点解析：**
*   `set -e`：确保脚本在任何命令失败时立即退出，增强了脚本的健壮性。
*   `helm upgrade --install`：一个幂等操作，如果 Release 不存在，则执行安装；如果已存在，则执行升级。这使得同一个脚本可以同时用于初始安装和后续更新。
*   `--create-namespace`：如果命名空间不存在，会自动创建，简化了前置操作。
*   `--set-string`：用于传递字符串类型的参数，特别是密码这类需要明确为字符串的值。
*   **资源定义 (`resources`)**：为 Redis Pod 和更新任务（`updateJob`）精细地设置了 CPU 和内存的 `requests` 与 `limits`，这是保障服务质量（QoS）和集群稳定性的重要措施。
*   **监控集成 (`metrics.*`)**：这是实现可观测性的关键。
    *   `metrics.enabled=true`：启用 Redis Exporter，它会作为一个 sidecar 容器与每个 Redis 实例一起部署，用于收集 Redis 指标。
    *   `metrics.serviceMonitor.enabled=true`：自动创建一个 `ServiceMonitor` CRD 资源。
    *   `metrics.serviceMonitor.labels.release`：设置 `ServiceMonitor` 的标签，以便让 Prometheus Operator 能够发现它，并自动将其添加到抓取目标列表中。

## 二、安装与验证

准备工作就绪后，部署和验证过程变得非常简单。

### 1. 执行安装

在项目目录下，赋予脚本执行权限并运行：

```shell
chmod +x install.sh
bash install.sh
```

Helm 将根据脚本中的定义，开始在 Kubernetes 集群中创建所有必需的资源，包括 StatefulSet、Services、Secrets、ServiceMonitor 等。

### 2. 初步验证

等待几分钟让所有 Pods 启动并准备就绪。你可以使用 `status.sh` 脚本（通常包含 `kubectl get pods -n <namespace>` 等命令）或直接执行以下命令来检查 Pods 状态：

```shell
# 替换为你的.env中配置的NAMESPACE和RELEASE_NAME
kubectl get pods -n redis -l app.kubernetes.io/instance=my-redis-cluster
```

您应该能看到多个 Redis 节点 Pods (如 `my-redis-cluster-0`, `my-redis-cluster-1`...) 都处于 `Running` 状态。

### 3. 进阶连接与集群功能验证

Pod 正常运行只是第一步，我们还需要验证 Redis 集群本身是否正常工作。

**a. 获取密码并启动客户端**
首先，从 K8s Secret 中获取 Redis 密码，然后启动一个临时的 Redis 客户端 Pod 用于连接测试。

```shell
# 1. 获取密码
export REDIS_PASSWORD=$(kubectl get secret --namespace "redis" my-redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d)

# 2. 运行临时客户端 Pod，并通过环境变量传入密码
kubectl run --namespace redis my-redis-cluster-client --rm --tty -i --restart='Never' \
 --env REDIS_PASSWORD=$REDIS_PASSWORD \
--image docker.io/bitnami/redis-cluster:8.0.2-debian-12-r2 -- bash
```

**b. 连接集群并验证**
在临时 Pod 的 shell 中，使用 `redis-cli` 连接到集群。`-c` 参数表示以集群模式连接。

```shell
# 在临时 Pod 的 bash 提示符下执行
redis-cli -c -h my-redis-cluster -a $REDIS_PASSWORD
```

连接成功后，执行以下命令验证集群状态：
```redis
# 查看集群信息
> info
# 查看集群节点拓扑和主从关系
> cluster nodes
```
如果 `cluster nodes` 命令能正确显示所有主从节点信息，则证明集群已成功组建并正常运行。

### 4. K8s 内部服务发现验证

在 Kubernetes 集群内部，其他应用可以通过标准的 DNS 名称访问 Redis 集群：

*   **推荐方式 (Cluster-IP Service)**：`my-redis-cluster.redis.svc.cluster.local:6379`
    大多数支持集群模式的 Redis 客户端库，只需要提供这个入口地址和密码，它们会自动通过 `CLUSTER NODES` 命令发现集群中的所有节点，并智能地将读写请求路由到正确的节点。

*   **节点直连方式 (Headless Service)**：
    Chart 还会创建一个 Headless Service，允许你直接解析到每个 Pod 的地址，例如 `my-redis-cluster-0.my-redis-cluster-headless.redis.svc.cluster.local:6379`。这在某些特定的调试或运维场景下很有用。

### 5. 监控验证

部署时我们已经启用了监控集成，现在来验证一下。

1.  **Prometheus Targets**: 访问您的 Prometheus UI，导航到 "Status" -> "Targets" 页面。您应该能看到一个名为 `serviceMonitor/<redis-namespace>/my-redis-cluster-metrics` 的目标，并且其所有 Endpoints (每个 Redis 节点) 的状态都是 "UP"。

2.  **Grafana Dashboard**: 访问您的 Grafana UI，点击 "Dashboards" -> "Import"，输入面板ID `11835`。这是社区为 Redis Exporter 提供的一个非常出色的 Grafana 面板。导入后，选择正确的数据源（Prometheus），您将看到一个包含丰富指标的仪表盘，全面展示了 Redis 集群的健康状况、性能和资源使用情况。

## 三、应用更新与卸载

### 更新应用

得益于 `helm upgrade --install` 的幂等性，更新变得非常简单。例如，若要增加持久化存储大小，只需修改 `.env` 或 `install.sh` 中的 `persistence.size` 值，然后重新执行：

```shell
bash install.sh
```
Helm 会智能地计算出变更，并只更新需要改动的 Kubernetes 资源。

### 卸载应用

**1. 执行卸载脚本**
项目提供了一个 `uninstall.sh` 脚本（通常内容为 `helm uninstall ${RELEASE_NAME} --namespace ${NAMESPACE}`）来清理所有由 Helm 创建的资源。

```shell
bash uninstall.sh
```

**2. （可选）清理持久化存储**
默认情况下，Helm 不会删除 PVC，这是为了防止意外的数据丢失。如果确认不再需要这些数据，您需要手动删除它们。

```shell
# 加载变量
source .env
# 查看命名空间下的所有 PVC
kubectl get pvc -n ${NAMESPACE}
# 逐个删除 PVC
kubectl delete pvc data-my-redis-cluster-0 -n ${NAMESPACE}
kubectl delete pvc data-my-redis-cluster-1 -n ${NAMESPACE}
# ... 以此类推
```

## 总结

通过结合 Helm、结构化的配置文件 (`.env`) 和自动化脚本 (`install.sh`)，我们实现了一个强大、灵活且易于维护的 Redis Cluster 部署方案。这种模式不仅适用于 Redis，也可以推广到其他任何需要在 Kubernetes 上部署的复杂有状态应用。

该方案的核心优势在于：
*   **标准化与可重复**：版本化的 Chart 和外部化的配置确保了在任何环境都能得到一致的部署结果。
*   **生命周期管理**：清晰地覆盖了安装、验证、更新和卸载的全过程。
*   **生产就绪**：内置了持久化、高可用、安全（密码）和可观测性（Prometheus/Grafana）等生产环境必备要素。

希望这篇指南能帮助您在 Kubernetes 的旅程中更轻松、更自信地驾驭像 Redis Cluster 这样的有状态服务。