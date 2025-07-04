---
title: K8s采用Helm部署redis-standalone
tags:
  - Linux
  - K8s
  - Helm
  - Redis
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250611185238101.png'
toc: true
abbrlink: fb011a6c
date: 2025-06-11 18:48:38
---

在现代微服务架构中，Redis 扮演着至关重要的角色，无论是作为高速缓存、分布式会话存储，还是消息队列，其高性能的读写能力都是后端系统不可或缺的组件。将 Redis 部署在 Kubernetes (K8s) 上，可以充分利用 K8s 的弹性伸缩、自愈和自动化管理能力。而 Helm，作为 K8s 的包管理器，则能将复杂的部署流程标准化、模板化，实现一键式部署与管理。

本文将提供一个生产级的实践指南，详细阐述如何使用 Helm 快速、可靠地在 Kubernetes 集群上部署一个带持久化存储和监控能力的 Redis Standalone（单机模式）实例。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/redis/redis-standalone), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/redis/redis-standalone)

## 核心优势

采用本方案部署 Redis，您将获得：

*   **配置化部署**：所有关键配置（如命名空间、密码、存储类）均通过 `.env` 文件管理，清晰且易于维护。
*   **一键式操作**：通过简单的 Shell 脚本即可完成安装、升级和卸载。
*   **生产级特性**：默认启用持久化存储（PVC），确保数据在 Pod 重启后不丢失。
*   **可观测性**：集成了 Prometheus Metrics Exporter 和 ServiceMonitor，可无缝对接到您的 Prometheus 监控体系，并使用 Grafana 进行可视化。
*   **资源精细化控制**：为 Redis 实例和监控组件都设置了明确的 CPU 和内存 `requests` 与 `limits`，保障集群资源调度的稳定。

## 项目文件结构

我们的部署项目主要由以下三个文件构成，各司其职：

*   `.env`: 存储所有可配置的环境变量，如命名空间、Helm Release 名称、密码等。
*   `install.sh`: 核心的安装与升级脚本，它会读取 `.env` 配置，并执行 `helm upgrade --install` 命令。
*   `README.md`: 项目的说明文档，提供了完整的操作指引。

接下来，我们将分步进行详解。

## 第一步：环境准备与配置

在开始之前，我们需要创建一个 `.env` 文件来定义部署所需的各项参数。这种将配置与脚本分离的方式是 DevOps 的最佳实践，它使得我们的部署脚本更具通用性。

> .env
```shell
# 命名空间
NAMESPACE="redis"
# helm的release名称
RELEASE_NAME="my-redis-standalone"
# helm的chart版本
CHART_VERSION="21.2.1"
# 存储类名称
STORAGE_CLASS_NAME="nfs"
# redis密码
REDIS_PASSWORD="YOUR_PASSWORD"

# Prometheus 监控组件所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 ServiceMonitor 的标签值 (通常是 helm release 的名称)
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```

**参数解析**：

*   `NAMESPACE`: Redis 将被安装到的 K8s 命名空间。
*   `RELEASE_NAME`: 本次 Helm 部署的实例名称，用于后续的管理和追踪。
*   `CHART_VERSION`: 指定使用的 Bitnami Redis Chart 版本，确保部署的可复现性。
*   `STORAGE_CLASS_NAME`: 指定用于持久化存储的 `StorageClass`。请确保您的 K8s 集群中已存在该 `StorageClass`（例如 `nfs`, `ceph`, `longhorn` 或云厂商提供的 `gp2`, `standard` 等）。
*   `REDIS_PASSWORD`: 设置 Redis 的访问密码，**请务必修改为您自己的强密码**。
*   `PROMETHEUS_NAMESPACE` & `PROMETHEUS_RELEASE_LABEL`: 这两个参数用于配置 `ServiceMonitor`，使其能被您集群中的 Prometheus Operator 正确发现。请根据您 Prometheus 的实际部署情况进行修改。

## 第二步：自动化安装脚本

我们使用一个 Shell 脚本 `install.sh` 来封装 Helm 命令，实现自动化部署。

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
helm upgrade --install "${RELEASE_NAME}" bitnami/redis \
  --version "${CHART_VERSION}" --namespace "${NAMESPACE}" --create-namespace \
  \
  --set architecture=standalone \
  \
  --set-string global.redis.password="${REDIS_PASSWORD}" \
  \
  --set master.persistence.enabled=true \
  --set-string master.persistence.storageClass="${STORAGE_CLASS_NAME}" \
  --set master.persistence.size=8Gi \
  \
  --set master.resources.requests.cpu=100m \
  --set master.resources.requests.memory=128Mi \
  --set master.resources.limits.cpu=512m \
  --set master.resources.limits.memory=2048Mi \
  \
  --set replica.resources.requests.cpu=100m \
  --set replica.resources.requests.memory=128Mi \
  --set replica.resources.limits.cpu=512m \
  --set replica.resources.limits.memory=2048Mi \
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

**脚本解析**：

1.  `set -e`: 确保脚本在任何命令出错时立即退出，增强了脚本的健壮性。
2.  `source .env`: 从 `.env` 文件加载我们之前定义的所有变量。
3.  `helm repo add/update`: 添加并更新 Bitnami 的 Helm 仓库，以获取最新的 Chart 信息。
4.  `helm upgrade --install`: 这是核心命令。
    *   `--install` 参数使其具备幂等性：如果 Release 不存在，则执行安装；如果已存在，则执行升级。这使得同一个脚本可以同时用于初始安装和后续更新。
    *   `--namespace "${NAMESPACE}" --create-namespace`: 在指定的命名空间中进行部署，如果命名空间不存在则自动创建。
    *   `--set ...`: 通过这些参数，我们动态地覆盖了 Chart 的默认值，实现了高度定制化：
        *   `architecture=standalone`: 明确指定部署单机模式的 Redis。
        *   `global.redis.password`: 将密码注入到 Chart 中。
        *   `master.persistence.*`: 启用主节点的持久化，并指定了 `StorageClass` 和 PVC 的大小。
        *   `*.resources.*`: 为 Redis 主节点、副本（此处单机模式不启用副本，但配置保留）和 metrics-exporter 设置了资源请求与限制，这是保障 K8s 集群稳定性的关键配置。
        *   `metrics.*`: 启用了 Metrics Exporter，并创建了一个 `ServiceMonitor` 资源，配置了正确的命名空间和标签，以便 Prometheus Operator 可以自动发现它。

现在，只需在终端执行 `bash install.sh`，一个配置完善的 Redis 实例就会被自动部署到您的集群中。

## 第三步：部署后验证

部署完成后，我们需要进行验证以确保一切正常。请参考项目中的 `README.md`，它提供了详细的验证步骤。

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

**1. 首先，获取 Redis 密码 (假设 Release 名称为 my-redis-standalone，密码 Key 为 redis-password)**

```shell
export REDIS_PASSWORD=$(kubectl get secret --namespace redis my-redis-standalone -o jsonpath="{.data.redis-password}" | base64 -d)
```

**2. 启动一个临时的 Redis 客户端 Pod 来连接实例**

```shell
kubectl run my-redis-standalone-client --namespace redis --rm --tty -i \
--env REDIS_PASSWORD_ENV="$REDIS_PASSWORD" \
--image docker.io/bitnami/redis:8.0.2-debian-12-r3 \
-- bash
```

**3. 在临时 Pod 中连接到 Redis 实例**

```shell
redis-cli -c -h my-redis-standalone-master -a "$REDIS_PASSWORD_ENV"
```

**4. 连接成功后，您可以执行 Redis 命令来验证实例状态**

```shell
# 在 redis-cli 提示符下执行
> info
```

**5. k8s 内部访问 Redis 实例**

```shell
# 方式一：<service>.<namespace>.svc.cluster.local:6379（大多数 Redis Cluster 客户端库只需要这个地址和密码即可自动发现所有节点）
my-redis-standalone-master.redis.svc.cluster.local:6379

# 方式二：<pod>.<headless-service>.<namespace>.svc.cluster.local:6379
my-redis-standalone-master-0.my-redis-standalone-headless.redis.svc.cluster.local:6379
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

**验证重点解读**：

*   **进阶验证**：这部分演示了如何在集群内部连接并测试 Redis。它首先从 K8s Secret 中安全地获取密码，然后启动一个临时的 Redis 客户端 Pod，并使用 `redis-cli` 连接到 Redis 服务。这是验证服务连通性和认证功能的标准方法。
*   **服务发现**：文档中提供了两种 K8s 内部的服务 DNS 地址，这对于其他微服务如何连接 Redis 提供了清晰的指引。
*   **监控验证**：这是验证我们可观测性配置是否成功的关键。
    1.  检查 Prometheus 的 Targets 页面，确认 `redis-exporter` 已经被成功发现并处于 `UP` 状态。
    2.  在 Grafana 中导入社区推荐的 Redis Dashboard (ID: 11835)，如果能看到数据，则证明从指标采集到数据可视化的整条链路已经完全打通。

## 第四步：更新与卸载

*   **更新**：得益于 `helm upgrade --install` 的幂等性，更新应用变得极其简单。只需修改 `.env` 文件中的配置（例如，想增加 `master.persistence.size`），然后重新执行 `bash install.sh` 即可。Helm 会智能地计算出变更，并只应用差异部分。
*   **卸载**：执行 `uninstall.sh` (该脚本通常包含 `helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}`) 会删除所有由 Helm 创建的 K8s 资源。**特别注意**，默认情况下 Helm 不会删除 PVC，这是一个防止数据误删的安全设计。如果确认不再需要数据，需要根据 `README.md` 的指引手动删除 PVC。

## 总结

通过结合 Helm、标准化的 Chart 以及自动化的部署脚本，我们构建了一套高效、可靠且生产就绪的 Redis 部署方案。该方案不仅简化了初始部署，更为后续的配置变更、升级和监控集成提供了强大的支持。作为后端系统工程师，掌握这种云原生时代的标准部署模式，将极大地提升我们构建和维护数据驱动型应用的能力。您可以基于此方案，轻松地将其适配到您的项目中。