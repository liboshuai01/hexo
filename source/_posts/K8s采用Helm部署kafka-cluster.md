---
title: K8s采用Helm部署kafka-cluster
abbrlink: c4730ed2
date: '2025-06-11T19:34:00.000Z'
tags:
  - 容器化
categories:
  - 容器化
toc: true
---


在当今的云原生时代，将像 Kafka 这样的有状态数据系统部署到 Kubernetes (K8s) 上已成为主流实践。Kubernetes 提供了强大的弹性伸缩、故障自愈和资源管理能力，而 Helm 作为 K8s 的包管理器，则极大地简化了复杂应用的部署和生命周期管理。

本文将详细介绍如何利用 Helm 和 Bitnami 提供的优秀 Chart，在 Kubernetes 集群中部署一个高可用的 Kafka 集群。我们将采用 Zookeeper-less 的 KRaft 模式，并通过一个结构化的项目，实现配置与部署逻辑的分离，使其更易于维护和复用。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/kafka/kafka-cluster), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/kafka/kafka-cluster)

## 一、项目结构概览

为了实现标准化和可重复的部署，我们采用以下文件结构：

- `.env`: 环境变量文件，用于存放所有可配置的参数，如命名空间、存储类、副本数等。这实现了配置与执行逻辑的分离。
- `install.sh`: 核心部署脚本，负责加载配置、更新 Helm 仓库并执行 `helm upgrade --install` 命令。
- `uninstall.sh`: 卸载脚本，用于清理 Helm Release 及相关资源。
- `status.sh`: （可选）状态检查脚本，用于快速查看部署后 Pod 和 Service 的状态。

这种结构使得部署过程非常清晰：修改`.env`文件进行配置，执行`install.sh`进行部署。

## 二、环境准备：配置`.env`文件

在开始之前，我们需要根据目标 K8s 集群的实际情况，配置`.env`文件。这个文件是整个部署的配置中心。

```shell
# .env

# 命名空间
NAMESPACE="kafka"
# helm的release名称
RELEASE_NAME="my-kafka-cluster"
# helm的chart版本
CHART_VERSION="32.2.8"
# 存储类名称
STORAGE_CLASS_NAME="nfs"
# controller节点数量
CONTROLLER_REPLICA_COUNT=3

# Prometheus 监控组件所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 ServiceMonitor 的标签值 (通常是 helm release 的名称)
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```

**参数解读:**

- `NAMESPACE`: 为 Kafka 集群创建独立的命名空间，实现资源隔离。
- `RELEASE_NAME`: Helm Release 的名称，用于标识和管理这次部署。
- `CHART_VERSION`:锁定 Bitnami Kafka Chart 的版本，确保部署的可重复性。
- `STORAGE_CLASS_NAME`: 指定用于持久化存储的 `StorageClass`。**请务必确保您的 K8s 集群中存在名为 `nfs` (或您自定义的名称) 的 `StorageClass`**，否则 PVC 将无法绑定。
- `CONTROLLER_REPLICA_COUNT`: Kafka Controller 节点的数量。在 KRaft 模式下，Controller 节点负责集群元数据的管理。设置为3可以提供高可用性。
- `PROMETHEUS_NAMESPACE` 和 `PROMETHEUS_RELEASE_LABEL`:这两个参数用于与 Prometheus Operator 集成。脚本会自动创建一个 `ServiceMonitor` 资源，Prometheus Operator 会根据这些标签发现 Kafka 的 metrics 端点，实现监控数据的自动采集。

## 三、核心部署脚本：`install.sh`详解

`install.sh` 脚本是部署工作的核心，它封装了所有 Helm 操作。

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
helm upgrade --install ${RELEASE_NAME} bitnami/kafka --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --create-namespace \
  \
  --set-string global.defaultStorageClass="${STORAGE_CLASS_NAME}" \
  \
  --set listeners.client.protocol=PLAINTEXT \
  --set listeners.client.sslClientAuth=none \
  --set listeners.controller.protocol=PLAINTEXT \
  --set listeners.controller.sslClientAuth=none \
  --set listeners.interbroker.protocol=PLAINTEXT \
  --set listeners.interbroker.sslClientAuth=none \
  --set listeners.external.protocol=PLAINTEXT \
  --set listeners.external.sslClientAuth=none \
  \
  --set defaultInitContainers.prepareConfig.resources.requests.cpu=100m \
  --set defaultInitContainers.prepareConfig.resources.requests.memory=128Mi \
  --set defaultInitContainers.prepareConfig.resources.limits.cpu=250m \
  --set defaultInitContainers.prepareConfig.resources.limits.memory=1024Mi \
  \
  --set controller.replicaCount=${CONTROLLER_REPLICA_COUNT} \
  --set controller.persistence.enabled=true \
  --set controller.persistence.size=16Gi \
  --set controller.logPersistence.enabled=true \
  --set controller.logPersistence.size=4Gi \
  --set controller.resources.requests.cpu=100m \
  --set controller.resources.requests.memory=128Mi \
  --set controller.resources.limits.cpu=512m \
  --set controller.resources.limits.memory=2048Mi \
  \
  --set rbac.create=true \
  \
  --set metrics.jmx.enabled=true \
  --set metrics.jmx.resources.requests.cpu=100m \
  --set metrics.jmx.resources.requests.memory=128Mi \
  --set metrics.jmx.resources.limits.cpu=256m \
  --set metrics.jmx.resources.limits.memory=1024Mi \
  --set metrics.serviceMonitor.enabled=true \
  --set metrics.serviceMonitor.namespace="${NAMESPACE}" \
  --set metrics.serviceMonitor.selector.release="${PROMETHEUS_RELEASE_LABEL}" \
  --set metrics.resources.requests.cpu=100m \
  --set metrics.resources.requests.memory=128Mi \
  --set metrics.resources.limits.cpu=256m \
  --set metrics.resources.limits.memory=1024Mi

  # 如果需要分离的 Broker 节点 (KRaft 提供的 Dedicated Broker Mode)，请取消注释并配置以下参数 \
  # --set broker.replicaCount=3 \
  # --set broker.persistence.enabled=true \
  # --set broker.persistence.size=32Gi \
  # --set broker.logPersistence.enabled=true \
  # --set broker.logPersistence.size=8Gi \
  # --set broker.resources.requests.cpu=250m \
  # --set broker.resources.requests.memory=512Mi \
  # --set broker.resources.limits.cpu=1000m \
  # --set broker.resources.limits.memory=2048Mi \
```

**脚本关键配置解析:**

1.  **`helm upgrade --install`**: 这是一个幂等操作。如果 Release 不存在，它会执行安装；如果已存在，它会进行升级。这使得脚本可以同时用于初始安装和后续更新。
2.  `--create-namespace`: 如果`.env`中指定的命名空间不存在，Helm 会自动创建它。
3.  **KRaft 模式与监听器配置**:
    - 此 Chart 默认启用 KRaft 模式，不再需要 Zookeeper。Controller 节点同时扮演 Broker 的角色。
    - `listeners.*.protocol=PLAINTEXT`: 为了简化部署示例，我们将所有内部和外部通信协议设置为 `PLAINTEXT`（明文）。**在生产环境中，强烈建议配置 `SASL` 或 `SSL/TLS` 以保证通信安全。**
4.  **持久化配置**:
    - `controller.persistence.enabled=true`: 为 Controller 节点启用持久化存储，用于存放 Kafka 的数据。
    - `controller.logPersistence.enabled=true`: 为 Controller 节点的日志也启用持久化。
    - `size`: 分别为数据和日志指定了 PVC 的大小。
5.  **资源配置 (`resources`)**: 为 `initContainers`、`controller` 节点和 `metrics` sidecar 精细地配置了 CPU 和 Memory 的 `requests` 和 `limits`，这是保障服务稳定性的关键。
6.  **监控集成 (`metrics.*`)**:
    - `metrics.jmx.enabled=true`: 启用 JMX Exporter，它会作为一个 sidecar 容器运行在 Kafka Pod 中，将 JMX 指标转换为 Prometheus 可识别的格式。
    - `metrics.serviceMonitor.enabled=true`: 指示 Helm Chart 创建一个 `ServiceMonitor` CRD 实例。
    - `metrics.serviceMonitor.selector.release`: 正是这个 selector，让 Prometheus Operator 知道应该由哪个 Prometheus 实例来抓取这个 `ServiceMonitor` 定义的指标。
7.  **专用 Broker 模式 (Dedicated Broker Mode)**:
    - 脚本中注释掉的部分是用于部署专用 Broker 节点的。默认情况下，Controller 节点也作为 Broker 使用。如果业务负载很高，需要将元数据管理（Controller）和数据处理（Broker）的职责分离，可以取消注释并配置该部分，以获得更好的性能和资源隔离。

## 四、部署与验证

遵循 `README.md` 中定义的清晰流程，我们可以轻松完成部署和验证。

### 1. 安装应用

只需在项目根目录下执行脚本：
```shell
bash install.sh
```

### 2. 初步验证

等待几分钟让所有 Pod 启动并就绪后，执行状态检查脚本：
```shell
bash status.sh
```
（这个脚本通常包含 `kubectl get pods -n kafka` 和 `helm status my-kafka-cluster -n kafka` 等命令）
您应该能看到3个 `my-kafka-cluster-controller-x` Pod 处于 `Running` 状态。

### 3. 进阶验证

为了确认 Kafka 集群功能完全正常，我们可以在集群内部启动一个临时的客户端 Pod，进行生产和消费测试。

**a. 启动临时 Pod**
```shell
kubectl run my-kafka-cluster-client --rm --tty -i --restart='Never' --image docker.io/bitnami/kafka:4.0.0-debian-12-r5 --namespace kafka --command -- bash
```
> 这条命令会创建一个临时的 Pod，并进入其 shell 环境。`--rm` 标志确保退出后 Pod 会被自动删除。

**b. 在临时 Pod 内创建 Topic**
```shell
kafka-topics.sh \
    --create \
    --bootstrap-server my-kafka-cluster:9092 \
    --topic test_topic \
    --partitions 6 \
    --replication-factor 3
```
> `bootstrap-server` 使用了 Kubernetes 的 Service DNS 名称 (`<release-name>.<namespace>`)。`replication-factor` 设置为3，与我们的 Controller 节点数一致，以实现高可用。

**c. 生产与消费消息**

- **打开一个终端，启动生产者并发送消息：**
  ```shell
  kafka-console-producer.sh \
      --bootstrap-server my-kafka-cluster:9092 \
      --topic test_topic
  ```
  输入 `hello kafka` 等消息后回车。

- **另开一个终端（需要再次 `kubectl exec` 进入同一个临时 Pod），启动消费者：**
  ```shell
  kafka-console-consumer.sh \
      --bootstrap-server my-kafka-cluster:9092 \
      --topic test_topic \
      --from-beginning
  ```
  如果能成功接收到刚才发送的 `hello kafka` 消息，说明集群工作正常。

### 4. K8s 内部访问地址

在K8s集群内部的其他应用需要连接此Kafka时，可以使用以下两种方式：

1.  **通过 Service (负载均衡):**
    `my-kafka-cluster.kafka.svc.cluster.local:9092`
    这是最常用的方式，通过标准的 Service 入口进行访问，流量会被负载均衡到后端的某个 Broker Pod。

2.  **通过 Headless Service (直连 Pod):**
    `my-kafka-cluster-controller-0.my-kafka-cluster-controller-headless.kafka.svc.cluster.local:9092`
    这种方式允许你直接连接到特定的 Pod。Kafka 客户端足够智能，它会首先连接到任何一个 Broker，然后获取整个集群的元数据（包括所有 Broker 的地址），后续的通信可能会直接与其他 Broker Pod 进行。

## 五、监控验证

1.  **Prometheus Targets**: 访问您的 Prometheus UI，导航到 "Status" -> "Targets" 页面。您应该能找到一个与 Kafka 相关的 endpoint，其状态为 `UP`，这表明 Prometheus 已成功发现并抓取 Kafka 的 JMX 指标。
2.  **Grafana Dashboard**: 访问您的 Grafana，点击 "Import dashboard"，输入 ID `7589`。这是一个社区提供的优秀 Kafka Exporter Dashboard。导入后，选择正确的数据源（您的 Prometheus），您就能看到一个包含吞吐量、延迟、分区状态等关键指标的可视化监控面板。

## 六、应用更新与卸载

### 更新应用
得益于 `helm upgrade --install` 的幂等性，更新非常简单。只需修改 `.env` 文件（例如，调整资源限制）或 `install.sh` 中的 `--set` 参数，然后重新执行：
```shell
bash install.sh
```
Helm 会智能地计算出变更并应用到集群中。

### 卸载应用
**1. 执行卸载脚本**
```shell
bash uninstall.sh
```
> 该脚本的核心命令是 `helm uninstall my-kafka-cluster -n kafka`。

**2. （可选）删除 PVC**
Helm 为了保护数据，默认不会删除 `PersistentVolumeClaim` (PVC)。如果确认不再需要 Kafka 中的数据，需要手动删除它们。
```shell
# 加载变量
source .env

# 查看pvc
kubectl get pvc -n ${NAMESPACE}

# 删除pvc（可能有多个pvc要删除）
kubectl delete pvc <pvc名称> -n ${NAMESPACE}
```

## 总结

通过本文的方案，我们实现了一个结构化、可配置、自动化的 Kafka on Kubernetes 部署流程。利用 Helm Chart，我们轻松地处理了 KRaft 模式的配置、持久化存储、资源管理和监控集成等复杂问题。这种将基础设施即代码（IaC）的理念应用于数据系统部署的方式，是现代后端系统和 DevOps 实践的最佳体现，为构建高性能、数据驱动的应用奠定了坚实可靠的基础。