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

在构建高性能、数据驱动的后端系统中，Redis Cluster 扮演着至关重要的角色。它不仅提供了卓越的缓存性能，还通过其分布式架构保证了数据的高可用性和可扩展性。然而，在 Kubernetes (K8s) 环境中手动部署和管理一个健壮的 Redis Cluster 是一项复杂且繁琐的任务。

幸运的是，我们可以借助 Helm——Kubernetes 的包管理器，极大地简化这一过程。通过使用预先配置好的 Helm Chart，我们能够以一种声明式、可重复的方式，快速部署一个生产级的 Redis Cluster。

本文将详细介绍如何使用 Bitnami 提供的优秀 Helm Chart，在 Kubernetes 集群中部署一套包含监控、持久化和高可用特性的 Redis Cluster。

<!-- more -->

> 项目源码: [github](ht/tps://github.com/liboshuai01/k8s-stack/tree/master/redis/redis-cluster), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/redis/redis-cluster)

## 核心优势

使用 Helm 部署 Redis Cluster 有以下几个显著优势：

*   **标准化与可重复性**：将 Redis Cluster 的所有 Kubernetes 资源（Deployments, StatefulSets, Services, ConfigMaps, Secrets, ServiceMonitors 等）打包成一个 Chart，确保每次部署都是一致的。
*   **配置简化**：通过一个中心化的 `.env` 文件和 `install.sh` 脚本，将复杂的配置项抽象为易于管理的变量。
*   **生命周期管理**：Helm 简化了应用的安装、升级、回滚和卸载全过程。
*   **集成监控**：Bitnami Chart 内置了对 Prometheus 的支持，可以轻松地将 Redis 的监控指标集成到现有的监控体系中。

## 项目文件结构

为了实现一键部署，我们采用以下精简的项目结构：

*   `.env`: 存储所有可配置的变量，如命名空间、密码、存储类等。
*   `install.sh`: 核心部署脚本，负责执行 `helm` 命令。
*   `README.md`: 项目说明书，包含了安装、验证、更新和卸载的完整指南。

接下来，我们将深入剖析每个文件的具体内容和作用。

### 1. 配置文件 (.env)

这是我们整个部署的“心脏”，集中管理了所有可变参数。通过修改此文件，我们可以轻松定制部署，而无需触碰核心的安装脚本。

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

**关键参数解析**：
*   `NAMESPACE`: 为 Redis Cluster 创建一个独立的命名空间，便于资源隔离和管理。
*   `RELEASE_NAME`: Helm Release 的名称，是此次部署在 Helm 中的唯一标识。
*   `CHART_VERSION`:锁定 Chart 版本，确保部署的可预测性和稳定性。
*   `STORAGE_CLASS_NAME`: 指定 Redis 数据持久化所使用的存储类。**请确保您的 K8s 集群中存在此 StorageClass。**
*   `REDIS_PASSWORD`: 设置 Redis 集群的访问密码。
*   `PROMETHEUS_NAMESPACE` & `PROMETHEUS_RELEASE_LABEL`: 用于配置 `ServiceMonitor`，使其能够被您集群中的 Prometheus Operator 自动发现，从而实现指标抓取。

### 2. 安装脚本 (install.sh)

此脚本是自动部署的执行者。它加载 `.env` 文件的配置，并执行 `helm upgrade --install` 命令来部署或更新 Redis Cluster。

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

**脚本核心逻辑**：
1.  **加载变量**: `source .env` 将配置注入到脚本的执行环境中。
2.  **准备 Helm 仓库**: 添加并更新 Bitnami 的 Helm 仓库，确保能拉取到最新的 Chart 信息。
3.  **执行部署**: `helm upgrade --install` 是一个幂等操作。如果 Release 不存在，它会执行安装；如果已存在，它会根据新的配置进行升级。
    *   `--create-namespace`: 如果命名空间不存在，则自动创建。
    *   `--set-string`: 用于传递字符串类型的配置，如密码和存储类名。
    *   `persistence.size=8Gi`: 为每个 Redis 节点配置 8Gi 的持久化存储卷。
    *   `resources.*`: 为 Redis Pod 和其更新任务（updateJob）精细地设置了 CPU 和内存的 `requests` 与 `limits`，这是保障集群稳定性的关键实践。
    *   `metrics.enabled=true` & `metrics.serviceMonitor.enabled=true`: 启用 Redis Exporter，并创建一个 `ServiceMonitor` CRD，为 Prometheus 监控铺平了道路。

### 3. 操作指南 (README.md)

`README.md` 文件是这份部署方案的“使用说明书”，它清晰地指导用户完成从准备、安装、验证到卸载的全过程。

> 前提准备
> ---
>
> 修改`.env`文件中配置的变量为自定义内容，如安装的命名空间、helm实例名称、char版本号等（可选）。
>
> 安装应用
> ---
>
> ```shell
> bash install.sh
> ```
>
> 验证应用
> ---
>
> ### 初步验证
>
> ```shell
> bash status.sh
> ```
>
> ### 进阶验证
>
> **1. 首先，获取 Redis 密码 (假设 Release 名称为 redis-cluster，密码 Key 为 redis-password)**
>
> ```shell
> export REDIS_PASSWORD=$(kubectl get secret --namespace "redis" my-redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d)
> ```
>
> **2. 启动一个临时的 Redis 客户端 Pod 来连接集群**
>
> ```shell
> kubectl run --namespace redis my-redis-cluster-client --rm --tty -i --restart='Never' \
>  --env REDIS_PASSWORD=$REDIS_PASSWORD \
> --image docker.io/bitnami/redis-cluster:8.0.2-debian-12-r2 -- bash
> ```
>
> **3. 在临时 Pod 中连接到 Redis 集群**
>
> ```shell
> redis-cli -c -h my-redis-cluster -a $REDIS_PASSWORD
> ```
>
> **4. 连接成功后，您可以执行 Redis 命令来验证集群状态**
>
> ```shell
> # 在 redis-cli 提示符下执行
> > info
> > cluster nodes
> ```
>
> **5. k8s 内部访问 Redis 集群**
>
> ```shell
> # 方式一：<service>.<namespace>.svc.cluster.local:6379（大多数 Redis Cluster 客户端库只需要这个地址和密码即可自动发现所有节点）
> my-redis-cluster.redis.svc.cluster.local:6379
>
> # 方式二：<pod>.<headless-service>.<namespace>.svc.cluster.local:6379
> my-redis-cluster-0.my-redis-cluster-headless.redis.svc.cluster.local:6379
> my-redis-cluster-1.my-redis-cluster-headless.redis.svc.cluster.local:6379
> my-redis-cluster-2.my-redis-cluster-headless.redis.svc.cluster.local:6379
> my-redis-cluster-3.my-redis-cluster-headless.redis.svc.cluster.local:6379
> my-redis-cluster-4.my-redis-cluster-headless.redis.svc.cluster.local:6379
> my-redis-cluster-5.my-redis-cluster-headless.redis.svc.cluster.local:6379
> ```
>
> ### 监控验证
>
> **1. 访问`prometheus`的`/targets`页面，查看`redis-exporter`是否正常 scrape metrics**
>
> **2. 访问`grafana`并导入面板`11835`，查看`redis-exporter`的dashboard是否正常显示。**
>
>
> 更新应用
> ---
>
> 修改`.env`或`install.sh`文件中的内容，后重新执行`install.sh`脚本即可。
>
> 卸载应用
> ---
>
> **1. 执行卸载脚本**
>
> ```shell
> bash uninstall.sh
> ```
>
> **2. （可选）删除pvc**
>
> ```shell
> # 加载变量
> source .env
>
> # 查看pvc
> kubectl get pvc -n ${NAMESPACE}
>
> # 删除pvc（可能有多个pvc要删除）
> kubectl delete pvc [pvc名称] -n ${NAMESPACE}
> ```

**关键验证步骤解读**：
*   **进阶验证**：提供了一个非常实用的调试和验证流程。通过创建一个临时的 Redis 客户端 Pod，我们可以直接在集群内部与 Redis Cluster 交互，使用 `cluster nodes` 命令检查集群的健康状态。
*   **K8s 内部访问**：这部分是后端开发人员最关心的。它清晰地列出了两种服务发现方式：
    1.  **标准 Service 地址**：`my-redis-cluster.redis.svc.cluster.local:6379`。这是推荐的方式。现代的 Redis Cluster 客户端（如 Lettuce、Jedis Cluster）只需要提供这个入口地址和密码，即可自动发现和管理集群中的所有主从节点。
    2.  **Headless Service 地址**：直接解析到每个 Pod 的地址。这种方式较少使用，但在特定调试场景下很有用。
*   **监控验证**：无缝集成了可观察性（Observability）的最佳实践，指导用户验证 Prometheus 是否成功抓取了指标，并推荐了成熟的 Grafana Dashboard (ID: 11835) 进行可视化展示。
*   **卸载**：强调了卸载后需要手动处理持久化存储卷（PVC），这是一个非常重要的提醒，可以防止数据意外丢失或产生不必要的存储成本。

## 总结

通过将配置、执行和文档三者结合，我们构建了一套强大而灵活的 Redis Cluster 部署方案。它不仅遵循了基础设施即代码（IaC）的理念，还融入了资源管理、高可用、持久化和可观察性等生产级系统的核心要素。

对于任何希望在 Kubernetes 上运行 Redis 的团队来说，这套基于 Helm 的方法都可以作为一个坚实的起点，让您从繁琐的运维工作中解放出来，更专注于核心业务逻辑的开发。