---
title: K8s采用Helm部署Kafka集群实战指南
tags:
  - Linux
  - K8s
  - Helm
  - Kafka
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505130251467.png'
toc: true
abbrlink: 84c192a2
date: 2025-05-09 16:59:31
---

Apache Kafka 是一个高性能、分布式的发布-订阅消息系统，广泛应用于实时数据管道和流式应用。在 Kubernetes (K8s) 环境中部署 Kafka 可以充分利用其弹性伸缩、自我修复和声明式配置等优势。Helm 作为 K8s 的包管理器，则大大简化了 Kafka 集群的部署和管理过程。

本指南将详细介绍如何使用 Helm 和 Bitnami提供的 Kafka Helm Chart，在 Kubernetes 集群上部署一个生产可用的 Kafka 集群（基于 KRaft模式，无需 ZooKeeper）。我们将使用提供的脚本和配置文件来自动化部署流程。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-stack/tree/master/kafka/kafka-cluster), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/kafka/kafka-cluster)

## 前提准备

在开始之前，请确保您已具备以下条件：

1.  **一个正在运行的 Kubernetes 集群**：版本建议 1.20+。
2.  **`kubectl` 命令行工具**：并已配置为指向您的 K8s 集群。
3.  **Helm 客户端**：版本建议 v3+。
4.  **一个可用的 StorageClass**：本指南默认使用名为 `nfs` 的 StorageClass。如果您的集群中没有或使用其他名称，请相应修改 `.env` 文件。
5.  **下载/准备相关脚本**：确保您已拥有以下文件：
    *   `.env`：配置文件
    *   `install.sh`：安装/升级脚本
    *   `status.sh`：状态检查脚本
    *   `uninstall.sh`：卸载脚本

## 项目文件结构与说明

我们的部署将围绕以下几个核心文件展开：

*   `.env`：环境变量配置文件，用于定义命名空间、Helm Release 名称、Chart 版本和存储类等。
*   `install.sh`：核心部署脚本，负责添加 Helm 仓库、更新仓库，并使用 Helm 安装或升级 Kafka 集群。
*   `status.sh`：用于快速检查已部署 Kafka 集群中 Pod 和 PVC 的状态。
*   `uninstall.sh`：用于卸载 Kafka 集群。

下面是这些文件的具体内容。

### 配置文件: `.env`

此文件定义了部署 Kafka 集群所需的关键参数。

```ini
# 命名空间名称
NAMESPACE="kafka"
# helm的release名称
RELEASE_NAME="my-kafka-cluster"
# helm的chart版本
CHART_VERSION="32.2.8"
# 存储类名称
STORAGE_CLASS_NAME="nfs"
```

**参数说明：**

*   `NAMESPACE`：Kafka 集群将要部署到的 Kubernetes 命名空间。如果该命名空间不存在，`install.sh` 脚本会尝试创建它。
*   `RELEASE_NAME`：Helm Release 的名称，用于唯一标识本次部署。
*   `CHART_VERSION`：要使用的 Bitnami Kafka Helm Chart 的版本。
*   `STORAGE_CLASS_NAME`：用于持久化存储的 Kubernetes StorageClass 名称。请确保您的集群中存在此 StorageClass，或者修改为您环境中可用的 StorageClass。

您可以根据您的需求修改这些值。

## 步骤一：配置环境变量

如上所示，请根据您的实际环境修改 `.env` 文件中的值。

## 步骤二：安装 Kafka 集群

配置好 `.env` 文件后，我们可以执行 `install.sh` 脚本来部署 Kafka 集群。

### 安装脚本: `install.sh`

```shell
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    export $(grep -v '^#' .env | sed 's/\r$//' | xargs)
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
  --set defaultInitContainers.prepareConfig.resources.limits.memory=512Mi \
  \
  --set controller.replicaCount=3 \
  --set controller.persistence.enabled=true \
  --set controller.persistence.size=16Gi \
  --set controller.logPersistence.enabled=true \
  --set controller.logPersistence.size=4Gi \
  --set controller.resources.requests.cpu=250m \
  --set controller.resources.requests.memory=512Mi \
  --set controller.resources.limits.cpu=1000m \
  --set controller.resources.limits.memory=2048Mi \
  # 如果需要分离的 Broker 节点 (KRaft 提供的 Dedicated Broker Mode)，请取消注释并配置以下参数 \
  # --set broker.replicaCount=3 \
  # --set broker.persistence.enabled=true \
  # --set broker.persistence.size=32Gi \
  # --set broker.logPersistence.enabled=true \
  # --set broker.logPersistence.size=8Gi \
  # --set broker.resources.requests.cpu=250m \
  # --set broker.resources.requests.memory=512Mi \
  # --set broker.resources.limits.cpu=1000m \
  # --set broker.resources.limits.memory=2048Mi
```

**执行安装：**

```shell
bash install.sh
```

**脚本解析：**

1.  **加载变量**：从 `.env` 文件中读取配置并将其导出为环境变量。
2.  **添加 Helm 仓库**：添加 Bitnami 的 Helm Chart 仓库。
3.  **更新 Helm 仓库**：确保本地 Helm 仓库信息是最新的。
4.  **安装/升级 Kafka**：使用 `helm upgrade --install` 命令部署 Kafka。
    *   各种 `--set` 参数用于自定义 Kafka Chart 的配置，包括：
        *   设置全局默认的存储类。
        *   为所有监听器（client, controller, interbroker, external）配置为 `PLAINTEXT` 协议，并禁用 SSL 客户端认证。**警告：这简化了初始设置，但在生产环境中，对于外部访问或跨网络通信，应考虑启用 SSL/TLS 加密和认证。**
        *   配置 `prepare-config` 初始化容器的资源请求和限制。
        *   设置 KRaft 控制器节点的副本数为3，并为其启用持久化存储（数据卷和日志卷）。
        *   为控制器节点配置资源请求和限制。
        *   注释部分提供了配置专用 Broker 节点的选项，用于 KRaft 的 "Dedicated Broker Mode"。

等待 Helm 命令执行完成。如果一切顺利，Kafka 集群就会成功部署到您的 Kubernetes 集群中。

## 步骤三：初步验证

部署完成后，我们可以执行 `status.sh` 脚本来快速检查 Kafka 相关资源的状态。

### 状态检查脚本: `status.sh`

```shell
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    export $(grep -v '^#' .env | sed 's/\r$//' | xargs)
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

kubectl get all -n ${NAMESPACE}
kubectl get pvc -n ${NAMESPACE}
```

**执行状态检查：**

```shell
bash status.sh
```

**脚本解析：**

该脚本会执行以下命令：

*   `kubectl get all -n ${NAMESPACE}`：显示指定命名空间下的所有 Kubernetes 资源 (Pods, Services, StatefulSets, etc.)。
*   `kubectl get pvc -n ${NAMESPACE}`：显示指定命名空间下的持久卷声明 (PersistentVolumeClaims)。

您应该能看到类似以下的输出（具体 Pod 名称和数量可能因配置而异）：

*   Controller Pods (例如 `my-kafka-cluster-controller-0`, `my-kafka-cluster-controller-1`, `my-kafka-cluster-controller-2`) 状态为 `Running`。
*   对应的 PVCs 状态为 `Bound`。
*   Kafka 服务（如 `my-kafka-cluster`, `my-kafka-cluster-headless`）已创建。

## 步骤四：进阶验证 (生产与消费消息)

为了进一步确认 Kafka 集群工作正常，我们将创建一个临时的客户端 Pod，并在其中创建 Topic、发送消息和消费消息。

1.  **启动客户端 Pod：**
    请根据您 `install.sh` 执行后，Helm 输出的 NOTES 部分或最新 Bitnami Kafka Chart 文档推荐的镜像版本进行调整。此处的镜像是示例。

    ```shell
    # 将 .env 文件中的 NAMESPACE 变量值替换到下面的命令中
    # 例如 kubectl run kafka-test-client ... --namespace kafka ...
    kubectl run kafka-test-client \
        --restart='Never' \
        --image docker.io/bitnami/kafka:4.0.0-debian-12-r3 \
        --namespace ${NAMESPACE} \
        --command -- sleep infinity
    ```
    **注意：** 请将命令中的 `${NAMESPACE}` 手动替换为 `.env` 文件中 `NAMESPACE` 的实际值，或者在执行前确保 `NAMESPACE` 环境变量已正确导出。

2.  **等待客户端 Pod 启动并进入其 Shell：**

    ```shell
    # 检查 Pod 状态 (替换 ${NAMESPACE})
    kubectl get pod kafka-test-client -n ${NAMESPACE}
    # 进入 Pod Shell (替换 ${NAMESPACE})
    kubectl exec --tty -i kafka-test-client --namespace ${NAMESPACE} -- bash
    ```

3.  **在客户端 Pod 内，创建一个测试 Topic：**
    Kafka 的 Bootstrap Server 地址通常是 `<RELEASE_NAME>:<PORT>` 或 `<RELEASE_NAME>-headless:<PORT>`。对于我们的配置，Client Listener 默认监听在 9092 端口。

    ```shell
    # 将 .env 文件中的 RELEASE_NAME 变量值替换到下面的命令中
    # 例如 --bootstrap-server my-kafka-cluster:9092
    # 对于3节点的combined mode集群，副本因子最大为3
    kafka-topics.sh \
        --create \
        --bootstrap-server ${RELEASE_NAME}:9092 \
        --topic test_topic \
        --partitions 6 \
        --replication-factor 3
    ```
    **注意：** 请将命令中的 `${RELEASE_NAME}` 手动替换为 `.env` 文件中 `RELEASE_NAME` 的实际值，或者在执行前确保 `RELEASE_NAME` 环境变量已正确导出。
    如果命令成功，会提示 `Created topic test_topic.`。

4.  **在客户端 Pod 内，启动生产者发送消息：**

    ```shell
    # 将 .env 文件中的 RELEASE_NAME 变量值替换到下面的命令中
    kafka-console-producer.sh \
        --bootstrap-server ${RELEASE_NAME}:9092 \
        --topic test_topic
    ```
    替换 `${RELEASE_NAME}`。然后，您可以输入一些消息，例如：
    `>Hello Kafka from K8s!`
    `>This is a test message.`
    按 `Ctrl+C` 退出生产者。

5.  **（新开一个终端）在客户端 Pod 内，启动消费者接收消息：**
    为了同时观察生产者和消费者，请打开一个新的本地终端窗口，然后再次 `kubectl exec` 进入同一个 `kafka-test-client` Pod。

    ```shell
    # 在新终端执行 (替换 ${NAMESPACE}):
    kubectl exec --tty -i kafka-test-client --namespace ${NAMESPACE} -- bash

    # 在 Pod 内执行 (替换 ${RELEASE_NAME}):
    kafka-console-consumer.sh \
        --bootstrap-server ${RELEASE_NAME}:9092 \
        --topic test_topic \
        --from-beginning
    ```
    您应该能看到之前通过生产者发送的消息。

6.  **删除客户端 Pod：**
    验证完成后，可以删除测试用的客户端 Pod。

    ```shell
    # 替换 ${NAMESPACE}
    kubectl delete pod kafka-test-client --namespace ${NAMESPACE}
    ```

## 步骤五：更新应用

如果您需要修改 Kafka 集群的配置（例如，调整资源限制、更改副本数、升级 Chart 版本等），可以：

1.  修改 `.env` 文件中的变量（如 `CHART_VERSION`）。
2.  或直接修改 `install.sh` 文件中的 `--set` 参数。
3.  然后重新执行 `install.sh` 脚本：

    ```shell
    bash install.sh
    ```
    Helm 的 `upgrade --install` 命令会智能地应用更改，如果 Release 不存在则安装，如果存在则升级。

## 步骤六：卸载应用

如果不再需要 Kafka 集群，可以按照以下步骤卸载：

### 卸载脚本: `uninstall.sh`

```shell
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

**执行卸载：**

1.  **执行卸载脚本：**

    ```shell
    bash uninstall.sh
    ```
    此脚本会执行 `helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}`，它将删除所有由该 Helm Release 创建的 Kubernetes 资源（StatefulSets, Services, ConfigMaps 等）。

2.  **（可选）删除持久卷声明 (PVCs)：**
    默认情况下，Helm 卸载操作不会删除 PVCs，以防止数据丢失。如果您确定不再需要这些数据，可以手动删除它们。

    ```shell
    # 查看指定命名空间下的 PVCs (替换 ${NAMESPACE})
    kubectl get pvc -n ${NAMESPACE}
   
    # 删除pvc (替换 ${NAMESPACE} 和 [pvc名称])
    # 例如: kubectl delete pvc data-my-kafka-cluster-controller-0 -n kafka
    kubectl delete pvc [pvc名称] -n ${NAMESPACE}
    ```
    请**谨慎操作**此步骤，确保数据已备份或不再需要。

## 总结

通过本指南，我们学习了如何使用提供的脚本和配置文件，借助 Helm 在 Kubernetes 上快速部署和管理一个基于 KRaft 模式的 Kafka 集群。这种方法不仅简化了初始部署，也方便了后续的配置更新和集群维护。

您可以进一步探索 Bitnami Kafka Helm Chart 的丰富配置选项，以满足更复杂的生产需求，例如启用 TLS 加密、配置认证授权、集成监控和日志系统等。祝您在 Kubernetes 上使用 Kafka 顺利！

---