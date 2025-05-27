---
title: K8s采用Helm部署kafka-exporter实战指南
abbrlink: 64683bd3
date: 2025-05-26 22:27:59
tags:
  - Linux
  - K8s
  - Helm
  - kafka
  - Kafka-exporter
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505262241780.png'
toc: true
---

Apache Kafka 作为一款高性能的分布式流处理平台，在现代数据架构中扮演着至关重要的角色。为了确保 Kafka 集群的稳定运行和性能优化，对其进行有效的监控是必不可少的。`kafka-exporter` 是一个流行的 Prometheus exporter，它能够从 Kafka 集群中抓取各种指标，并通过 HTTP 端点暴露给 Prometheus 进行采集和展示。

本文将详细介绍如何在 Kubernetes (K8s) 环境中，利用 Helm 包管理器快速部署和配置 `kafka-exporter`，并将其集成到已有的 Prometheus监控体系中。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-cookbook/tree/master/kafka/kafka-exporter), [gitee](https://gitee.com/liboshuai01/k8s-cookbook/tree/master/redis/redis-exporter)

## 前提条件

在开始之前，请确保您已具备以下环境和工具：

1.  **Kubernetes 集群**: 一个正在运行的 K8s 集群。
2.  **Helm**: Helm v3 或更高版本已安装并配置完毕。
3.  **kubectl**: `kubectl` 命令行工具已配置并能够访问您的 K8s 集群。
4.  **Kafka 集群**: K8s 集群内部或外部署有一个可访问的 Kafka 集群。
5.  **(可选) Prometheus 监控栈**: 为了实现 ServiceMonitor 自动发现，建议已在 K8s 中部署 Prometheus Operator (例如通过 `kube-prometheus-stack` Helm chart)。

## 准备工作

### 1. 文件结构

建议创建一个专门的目录来存放相关配置文件，例如 `kafka-exporter-deploy`：

```
kafka-exporter-deploy/
├── .env
├── install.sh
├── status.sh
└── uninstall.sh
```

### 2. 配置 `.env` 文件

`.env` 文件用于存储部署过程中需要的变量。请根据您的实际环境修改这些值。

```ini
# 命名空间
NAMESPACE="kafka"
# helm的release名称
RELEASE_NAME="my-kafka-exporter"
# helm的chart版本
CHART_VERSION="2.12.1"
# kafka集群地址
KAFKA_SERVER_0="my-kafka-cluster-controller-0.my-kafka-cluster-controller-headless.kafka.svc.cluster.local:9092"
KAFKA_SERVER_1="my-kafka-cluster-controller-1.my-kafka-cluster-controller-headless.kafka.svc.cluster.local:9092"
KAFKA_SERVER_2="my-kafka-cluster-controller-2.my-kafka-cluster-controller-headless.kafka.svc.cluster.local:9092"
# kube-prometheus-stack的命名空间
MONITOR_NAMESPACE="monitoring"
# kube-prometheus-stack的release名称
MONITOR_RELEASE_NAME="kube-prom-stack"
```

**关键变量说明:**

*   `NAMESPACE`: `kafka-exporter` 将被安装到的 K8s 命名空间。
*   `RELEASE_NAME`: Helm 发行版的名称。
*   `CHART_VERSION`: `prometheus-kafka-exporter` Helm chart 的版本号。
*   `KAFKA_SERVER_0`, `KAFKA_SERVER_1`, `KAFKA_SERVER_2`: 您的 Kafka brokers 的连接地址。**请务必根据您的 Kafka 集群实际情况修改这些值。** 如果您的 Kafka brokers 数量不同，请相应调整后续 `install.sh` 脚本中的 `--set kafkaServer[X]` 参数。
*   `MONITOR_NAMESPACE`: Prometheus Operator 所在的命名空间。`ServiceMonitor` 资源将创建在此命名空间下。
*   `MONITOR_RELEASE_NAME`: 如果您使用 `kube-prometheus-stack`，这通常是其 Helm release 名称。`ServiceMonitor` 会带上 `release: ${MONITOR_RELEASE_NAME}` 标签，以便 Prometheus Operator 能够发现它。

### 3. 准备安装脚本 `install.sh`

该脚本负责添加 Helm 仓库、更新仓库，并使用 `.env` 文件中定义的变量来安装或升级 `kafka-exporter`。

```bash
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    # 确保 .env 文件中的变量导出到环境变量
    export $(grep -v '^#' .env | sed 's/\r$//' | xargs)
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

# --- 添加仓库并更新 ---
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# --- 安装 / 升级 ---
helm upgrade --install ${RELEASE_NAME} prometheus-community/prometheus-kafka-exporter \
  --version ${CHART_VERSION} --namespace ${NAMESPACE} --create-namespace \
  \
  --set kafkaServer[0]="${KAFKA_SERVER_0}" \
  --set kafkaServer[1]="${KAFKA_SERVER_1}" \
  --set kafkaServer[2]="${KAFKA_SERVER_2}" \
  --set prometheus.serviceMonitor.enabled=true \
  --set prometheus.serviceMonitor.namespace="${MONITOR_NAMESPACE}" \
  --set prometheus.serviceMonitor.additionalLabels.release="${MONITOR_RELEASE_NAME}" \
  \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=128Mi \
  --set resources.limits.cpu=1000m \
  --set resources.limits.memory=2048Mi
```

**脚本解析:**

1.  **加载变量**: 从 `.env` 文件中读取配置并将其导出为环境变量。
2.  **添加/更新 Helm 仓库**: 添加 `prometheus-community` 仓库，这里包含了 `prometheus-kafka-exporter` chart。
3.  **安装/升级**: 使用 `helm upgrade --install` 命令进行部署。
    *   `--install`: 如果 release 不存在则安装，存在则升级。
    *   `${RELEASE_NAME}`: 使用 `.env` 中定义的 release 名称。
    *   `prometheus-community/prometheus-kafka-exporter`: 指定要安装的 chart。
    *   `--version ${CHART_VERSION}`: 指定 chart 版本。
    *   `--namespace ${NAMESPACE} --create-namespace`: 指定命名空间，如果不存在则创建。
    *   `--set kafkaServer[0/1/2]`: 设置 Kafka broker 地址。**如果您的 Kafka broker 数量或地址与 `.env` 中定义的不同，请务必调整这里的参数数量和值。**
    *   `--set prometheus.serviceMonitor.enabled=true`: 启用 `ServiceMonitor` 的创建。
    *   `--set prometheus.serviceMonitor.namespace="${MONITOR_NAMESPACE}"`: 指定 `ServiceMonitor` 创建在哪个命名空间（通常是 Prometheus Operator 所在的命名空间）。
    *   `--set prometheus.serviceMonitor.additionalLabels.release="${MONITOR_RELEASE_NAME}"`: 为 `ServiceMonitor` 添加额外的标签。这对于让 Prometheus Operator（特别是 `kube-prometheus-stack` 部署的）自动发现此 `ServiceMonitor` 至关重要。Prometheus 通常会根据 `release` 标签来筛选 `ServiceMonitor`。
    *   `--set resources...`: 配置 `kafka-exporter` Pod 的资源请求和限制。

### 4. 准备状态查看脚本 `status.sh`

此脚本用于快速查看 `kafka-exporter` 相关资源在 K8s 中的状态。

```bash
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    export $(grep -v '^#' .env | sed 's/\r$//' | xargs)
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

kubectl get all -n ${NAMESPACE}
```

### 5. 准备卸载脚本 `uninstall.sh`

此脚本用于从 K8s 集群中卸载 `kafka-exporter`。

```bash
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    export $(grep -v '^#' .env | sed 's/\r$//' | xargs)
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}
```

## 安装应用

1.  确保您已根据实际环境修改了 `.env` 文件，特别是 `KAFKA_SERVER_*`, `MONITOR_NAMESPACE`, 和 `MONITOR_RELEASE_NAME`。
2.  为脚本添加执行权限：
    ```bash
    chmod +x install.sh status.sh uninstall.sh
    ```
3.  执行安装脚本：
    ```bash
    bash install.sh
    ```

等待 Helm 命令执行完毕。

## 初步验证

安装完成后，执行状态查看脚本来检查部署情况：

```bash
bash status.sh
```

您应该能看到类似以下的输出（具体名称会根据您的 `.env` 配置而有所不同）：

```
--- 在命名空间 kafka 中获取所有资源 ---
NAME                                                              READY   STATUS    RESTARTS   AGE
pod/my-kafka-exporter-prometheus-kafka-exporter-xxxxxxxxxx-yyyyy   1/1     Running   0          2m

NAME                                                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/my-kafka-exporter-prometheus-kafka-exporter                ClusterIP   10.100.x.x   <none>        9308/TCP   2m

NAME                                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-kafka-exporter-prometheus-kafka-exporter   1/1     1            1           2m

NAME                                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/my-kafka-exporter-prometheus-kafka-exporter-xxxxxxxxxx   1         1         1       2m

--- 在命名空间 kafka 中获取持久卷声明 (PVC) ---
No resources found in kafka namespace.
```

关键点是确保 `pod/my-kafka-exporter-...` 处于 `Running` 状态，并且 `deployment.apps/my-kafka-exporter-...` 的 `AVAILABLE` 副本数为 1。

## 进阶验证

### 1. 访问 Exporter 端点

`kafka-exporter` 默认在 `9308` 端口暴露指标。您可以通过其 Service 在集群内部访问。Service 的名称通常是 `${RELEASE_NAME}-prometheus-kafka-exporter`。

例如，根据我们的 `.env` 文件，Service 的 FQDN (完全限定域名) 可能是：
`my-kafka-exporter-prometheus-kafka-exporter.kafka.svc.cluster.local:9308`

您可以从集群内的一个 Pod 中尝试 `curl` 这个地址：
```bash
# 启动一个临时pod用于测试 (例如在 default 命名空间)
# kubectl run -i --tty --rm debug --image=curlimages/curl --restart=Never -- sh

# 在临时pod的shell中执行
curl http://my-kafka-exporter-prometheus-kafka-exporter.kafka.svc.cluster.local:9308/metrics
```
如果看到一长串 Prometheus 格式的指标输出，则说明 `kafka-exporter` 正在正常工作并连接到了您的 Kafka 集群。

### 2. 检查 Prometheus ServiceMonitor

如果您配置了 `prometheus.serviceMonitor.enabled=true` 并且正确设置了 `MONITOR_NAMESPACE` 和 `MONITOR_RELEASE_NAME`，Prometheus Operator 应该会自动发现并开始抓取 `kafka-exporter` 的指标。

*   **检查 ServiceMonitor 是否创建成功**:
    ```bash
    kubectl get servicemonitor -n ${MONITOR_NAMESPACE} # 将 ${MONITOR_NAMESPACE} 替换为 .env 中的值
    ```
    您应该能看到名为 `${RELEASE_NAME}-prometheus-kafka-exporter` 的 `ServiceMonitor`。

*   **检查 Prometheus Targets**:
    登录到您的 Prometheus UI，导航到 "Status" -> "Targets"。您应该能在列表中找到一个指向 `kafka-exporter` 的 target，并且其状态为 `UP`。如果未找到或状态为 `DOWN`，请检查 `ServiceMonitor` 的配置、标签以及 Prometheus Operator 的日志。

### 3. 导入 Grafana 面板

> 注意：必须先使用非控制台生产者、消费者进行生产和消费数据，否则grafana面板会无数据。

在 Prometheus UI 中，您可以导入 `kafka-exporter` 的 Grafana 面板，面板ID为`7589`，查看面板是否正常显示数据。

## 更新应用

如果您需要修改 `kafka-exporter` 的配置（例如，更新 Kafka broker 地址、调整资源限制或升级 chart 版本）：

1.  修改 `.env` 文件中的相应变量。
2.  如果需要调整传递给 `helm upgrade` 的参数（例如添加或修改 `--set` 值），请直接修改 `install.sh` 脚本。
3.  重新执行安装脚本：
    ```bash
    bash install.sh
    ```
    Helm 的 `upgrade --install` 机制会智能地应用更改。

## 卸载应用

当不再需要 `kafka-exporter` 时，可以执行卸载脚本：

```bash
bash uninstall.sh
```
这将使用 Helm 从集群中移除 `kafka-exporter` 的所有相关资源（Deployment, Service, ServiceMonitor 等）。

如果 `NAMESPACE` 是专门为 `kafka-exporter` 创建且不再需要，您可以手动删除该命名空间：
```bash
# kubectl delete ns kafka  # 将 kafka 替换为您的命名空间
```
**请谨慎操作，确保该命名空间下没有其他重要应用。**

## 总结

通过本指南，您学习了如何使用 Helm 在 Kubernetes 上快速部署 `kafka-exporter`，并将其配置为可被 Prometheus 自动发现。这种自动化的部署和管理方式大大简化了 Kafka 监控的设置过程，使您能够更专注于从指标中获取洞察，保障 Kafka 集群的健康运行。利用提供的脚本，您可以轻松地进行安装、验证、更新和卸载操作。

---