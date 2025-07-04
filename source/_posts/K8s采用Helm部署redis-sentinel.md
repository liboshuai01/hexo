---
title: K8s采用Helm部署redis-sentinel
tags:
  - Linux
  - K8s
  - Helm
  - Redis
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250611190521489.png'
toc: true
abbrlink: f8f963bc
date: 2025-06-11 18:56:36
---

在现代云原生架构中，缓存系统是提升应用性能、降低后端负载的关键组件。Redis以其卓越的性能和丰富的数据结构，成为了缓存解决方案的首选。然而，在生产环境中，单点的Redis实例存在高可用性风险。为了解决这个问题，Redis Sentinel（哨兵）模式应运而生，它能够自动监控、通知和故障转移，确保Redis服务的连续性。

本文将作为一份实战指南，详细阐述如何利用Kubernetes（K8s）和Helm，快速、标准地部署一个生产级别的高可用Redis Sentinel集群。我们将使用Bitnami提供的优秀Helm Chart，它封装了复杂的配置，让我们能够通过简单的变量定义，实现一主多从、多哨兵、持久化存储以及Prometheus监控的集成。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/redis/redis-sentinel), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/redis/redis-sentinel)

## 项目结构概览

为了实现标准化和可重复的部署，我们采用以下文件结构：

*   `.env`: 核心配置文件，用于定义命名空间、密码、副本数等所有可变参数。
*   `install.sh`: 部署脚本，负责加载配置、更新Helm仓库并执行部署/升级命令。
*   `uninstall.sh`: 卸载脚本，用于清理Helm Release。
*   `status.sh`: 状态检查脚本，用于快速验证部署后的Pod状态。
*   `README.md`: 项目说明文档，提供完整的操作指引。

这种结构将配置与执行逻辑分离，极大地提高了部署的灵活性和可维护性。

## 核心配置 (`.env`)

一切部署始于配置。`.env`文件是我们定义整个Redis集群行为的控制中心。

```shell
# 命名空间
NAMESPACE="redis"
# helm的release名称
RELEASE_NAME="my-redis-ha"
# helm的chart版本
CHART_VERSION="21.2.1"
# 存储类名称
STORAGE_CLASS_NAME="nfs"
# redis密码
REDIS_PASSWORD="YOUR_PASSWORD"

# -- 高可用配置 --
# 从节点/哨兵节点数量 (为了实现3哨兵，这里设置为3)
REPLICA_COUNT=3
# 哨兵仲裁所需的最小投票数 (推荐值为 N/2 + 1)
SENTINEL_QUORUM=2

# Prometheus 监控组件所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 ServiceMonitor 的标签值
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```

**关键配置解析：**

*   `NAMESPACE` & `RELEASE_NAME`: 定义了应用在K8s中的逻辑隔离空间和Helm实例名，便于管理。
*   `STORAGE_CLASS_NAME`: **至关重要**。为Redis Master和Replica节点指定持久化存储类。在生产环境中，这确保了即使Pod重启或漂移，数据也不会丢失。您需要根据自己的K8s集群环境，替换为可用的StorageClass，如`ceph-rbd`, `gp2`等。
*   `REPLICA_COUNT`: 定义了从节点（Replica）和哨兵（Sentinel）的数量。这里设置为`3`，意味着我们将部署一个拥有1个Master、3个Replica和3个Sentinel的集群。这是一个典型的高可用配置，3个哨兵可以有效避免脑裂问题。
*   `SENTINEL_QUORUM`: 哨兵判定Master下线并触发故障转移所需的最小投票数。官方推荐值为`(N/2 + 1)`，其中N是哨兵总数。对于3个哨兵，此值为2，意味着至少需要2个哨兵同意，才能进行主备切换。
*   `PROMETHEUS_*`: 这些配置用于与Prometheus监控系统无缝集成。通过创建`ServiceMonitor`资源，并打上正确的标签，Prometheus Operator可以自动发现并抓取Redis Exporter暴露的监控指标。

## 自动化部署脚本 (`install.sh`)

此脚本是部署的核心执行者，它将`.env`中的配置转化为具体的Helm部署参数。

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
echo "添加并更新 Bitnami Helm 仓库..."
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# --- 安装 / 升级 ---
echo "开始部署 Redis (一主${REPLICA_COUNT}从, ${REPLICA_COUNT}哨兵)..."
helm upgrade --install "${RELEASE_NAME}" bitnami/redis \
  --version "${CHART_VERSION}" --namespace "${NAMESPACE}" --create-namespace \
  \
  --set-string global.redis.password="${REDIS_PASSWORD}" \
  \
  --set architecture=replication \
  \
  --set master.persistence.enabled=true \
  --set-string master.persistence.storageClass="${STORAGE_CLASS_NAME}" \
  --set master.persistence.size=8Gi \
  --set master.resources.requests.cpu=100m \
  --set master.resources.requests.memory=128Mi \
  --set master.resources.limits.cpu=512m \
  --set master.resources.limits.memory=2048Mi \
  \
  --set replica.replicaCount=${REPLICA_COUNT} \
  --set replica.persistence.enabled=true \
  --set-string replica.persistence.storageClass="${STORAGE_CLASS_NAME}" \
  --set replica.persistence.size=8Gi \
  --set replica.resources.requests.cpu=100m \
  --set replica.resources.requests.memory=128Mi \
  --set replica.resources.limits.cpu=512m \
  --set replica.resources.limits.memory=2048Mi \
  \
  --set sentinel.enabled=true \
  --set sentinel.persistence.enabled=true \
  --set sentinel.persistence.size=1Gi \
  --set sentinel.quorum=${SENTINEL_QUORUM} \
  --set sentinel.resources.requests.cpu=100m \
  --set sentinel.resources.requests.memory=128Mi \
  --set sentinel.resources.limits.cpu=512m \
  --set sentinel.resources.limits.memory=2048Mi \
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

**脚本关键点解析:**

*   `helm upgrade --install`: 这是一个幂等操作。如果Release不存在，它会执行安装；如果已存在，它会执行升级。这使得我们可以用同一个脚本进行首次部署和后续的配置更新。
*   `--create-namespace`: 如果命名空间不存在，则自动创建，非常方便。
*   `architecture=replication`: 明确告诉Helm Chart我们要部署的是主从复制架构，而非单机版或集群版（Redis Cluster）。
*   `master.persistence.enabled=true` / `replica.persistence.enabled=true`: 为主从节点均开启持久化。这对于数据恢复和高可用性至关重要。
*   `sentinel.enabled=true`: 启用哨兵模式。这是构建高可用Redis的核心。
*   `resources.*`: 为Master、Replica、Sentinel和Metrics Exporter等所有组件都定义了资源请求（requests）和限制（limits）。这是K8s生产实践的最佳做法，能保证服务质量（QoS）并避免资源抢占。
*   `metrics.enabled=true` and `metrics.serviceMonitor.enabled=true`: 启用内置的Redis Exporter，并创建`ServiceMonitor` CRD，使其能够被Prometheus Operator自动纳管。

## 部署与验证

现在，让我们遵循`README.md`中的步骤，完成从部署到验证的全过程。

> README.md
```markdown
前提准备
---

修改`.env`文件中配置的变量为自定义内容，如安装的命名空间、helm实例名称、char版本号等（可选）。

安装应用
---

```shell
bash install.sh
```

验证应用
---

### 初步验证

```shell
bash status.sh
```

### 进阶验证

**1. 首先，获取 Redis 密码 (假设 Release 名称为 my-redis-sentinel，密码 Key 为 redis-password)**

```shell
export REDIS_PASSWORD=$(kubectl get secret --namespace redis my-redis-ha -o jsonpath="{.data.redis-password}" | base64 -d)
```

**2. 启动一个临时的 Redis 客户端 Pod 来连接实例**

```shell
kubectl run my-redis-sentinel-client --namespace redis --rm --tty -i \
--env REDIS_PASSWORD_ENV="$REDIS_PASSWORD" \
--image docker.io/bitnami/redis:8.0.2-debian-12-r3 \
-- bash
```

**3. 在临时 Pod 中连接到 Redis 实例**

```shell
# 连接到只读节点
redis-cli -h my-redis-ha -p 6379 -a "$REDIS_PASSWORD_ENV"
# 连接到哨兵节点
redis-cli -h my-redis-ha -p 26379 -a "$REDIS_PASSWORD_ENV"
```

**4. 连接成功后，您可以执行 Redis 命令来验证实例状态**

```shell
# 连接到只读节点后，在 redis-cli 提示符下执行
> info

# 连接到哨兵节点后，在 redis-cli 提示符下执行
> sentinel masters
> sentinel master mymaster
> sentinel slaves mymaster
```

**5. k8s 内部应用访问 Redis 哨兵集群**

```
# 方式一：<service>.<namespace>.svc.cluster.local:26379（大多数 Redis 客户端库只需要这个地址和密码即可自动发现所有节点）
- sentinel-master名称：mymaster
- 地址：my-redis-ha.redis.svc.cluster.local:26379

# 方式二：<pod>.<headless-service>.<namespace>.svc.cluster.local:26379
- sentinel-master名称：mymaster
- 地址：
my-redis-ha-node-0.my-redis-ha-headless.redis.svc.cluster.local:26379
my-redis-ha-node-1.my-redis-ha-headless.redis.svc.cluster.local:26379
my-redis-ha-node-2.my-redis-ha-headless.redis.svc.cluster.local:26379
```

### 监控验证

**1. 访问`prometheus`的`/targets`页面，查看`redis-exporter`是否正常 scrape metrics**

**2. 访问`grafana`并导入面板`11835`，查看`redis-exporter`的dashboard是否正常显示。**


更新应用
---

修改`.env`或`install.sh`文件中的内容，后重新执行`install.sh`脚本即可。

卸载应用
---

**1. 执行卸载脚本**

```shell
bash uninstall.sh
```

**2. （可选）删除pvc**

```shell
# 加载变量
source .env

# 查看pvc
kubectl get pvc -n ${NAMESPACE}

# 删除pvc（可能有多个pvc要删除）
kubectl delete pvc [pvc名称] -n ${NAMESPACE}
```

**重点解读：**

*   **连接方式**: 对于需要连接此Redis集群的Java应用（或其他后端服务），**强烈推荐使用方式一**。现代的Redis客户端库（如Jedis, Lettuce）都支持哨兵模式。您只需在配置中提供哨兵的Service地址 (`my-redis-ha.redis.svc.cluster.local:26379`)、Master的名称 (`mymaster`) 和密码。客户端会自动连接哨兵，获取当前Master节点的地址，并建立连接。当发生故障转移时，客户端也能自动感知并重连到新的Master，实现了应用层的无感切换。
*   **监控验证**: `Grafana`面板`11835`是一个社区验证过非常优秀的Redis Exporter Dashboard。成功导入并看到数据，是验证我们监控链路配置正确的最终凭证。
*   **卸载**: `helm uninstall`默认不会删除由StatefulSet创建的PVC。这是一个安全特性，防止误操作导致数据丢失。如果确认不再需要这些数据，需要手动删除PVC。

## 结论

通过本文的方案，我们利用Helm和一份精心设计的部署脚本，在Kubernetes上成功构建了一个稳健、可观测、易于管理的高可用Redis Sentinel集群。该方案具备以下优点：

1.  **自动化与标准化**：所有配置和部署逻辑代码化，保证了环境的一致性和部署的可重复性。
2.  **高可用性**：通过哨兵模式和持久化存储，有效应对节点故障，保障了服务的连续性和数据的安全性。
3.  **可观测性**：无缝集成了Prometheus和Grafana，为性能监控、问题排查和容量规划提供了强大的数据支持。
4.  **易于维护**：无论是更新配置还是卸载应用，都只需执行一条简单的命令，极大降低了运维成本。

这套实践不仅适用于Redis，其“配置与逻辑分离”、“声明式部署”、“集成可观测性”的思想，同样是构建任何高性能、数据驱动的云原生后端系统的基石。