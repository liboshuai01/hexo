---
title: K8s采用Helm部署redis-exporter实战指南
tags:
  - Linux
  - K8s
  - Helm
  - Redis
  - Redis-exporter
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505270144511.png'
toc: true
abbrlink: 34b93d5c
date: 2025-05-27 01:40:21
---

在Kubernetes (K8s)环境中监控Redis集群的性能和健康状况至关重要。`redis-exporter`是一个流行的Prometheus Exporter，它能够从Redis实例中抓取指标，以便Prometheus进行收集和存储，并最终通过Grafana等工具进行可视化。Helm作为K8s的包管理器，极大地简化了应用的部署和管理。

本文将详细介绍如何使用Helm在K8s集群中部署`redis-exporter`，并将其配置为由`kube-prometheus-stack`（或任何遵循Prometheus Operator规范的Prometheus部署）自动发现。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-cookbook/tree/master/redis/redis-exporter), [gitee](https://gitee.com/liboshuai01/k8s-cookbook/tree/master/redis/redis-exporter)

## 前提准备

在开始之前，请确保您已准备好以下环境：

1.  一个正在运行的Kubernetes集群。
2.  `kubectl`命令行工具已配置并连接到您的集群。
3.  Helm V3已安装。
4.  一个正在运行的Redis实例或集群（本文将配置`redis-exporter`连接到它）。
5.  （可选但推荐）已安装`kube-prometheus-stack`或类似的Prometheus监控解决方案，以便利用`ServiceMonitor`。

## 配置文件 (`.env`)

我们使用一个`.env`文件来管理部署所需的配置变量，方便修改和维护。

```
# 命名空间
NAMESPACE="redis"
# helm的release名称
RELEASE_NAME="my-redis-exporter"
# helm的chart版本
CHART_VERSION="6.10.3"
# redis地址
REDIS_ADDRESS="redis://my-redis-cluster:6379"
# redis密码的secret名称
REDIS_SECRET_NAME="my-redis-cluster"
# redis密码的secret的key名称
REDIS_SECRET_KEY="redis-password"
# kube-prometheus-stack的命名空间
MONITOR_NAMESPACE="monitoring"
# kube-prometheus-stack的release名称
MONITOR_RELEASE_NAME="kube-prom-stack"
```

**配置说明：**

*   `NAMESPACE`: `redis-exporter`将部署到的Kubernetes命名空间。
*   `RELEASE_NAME`: Helm部署的实例名称。
*   `CHART_VERSION`: `prometheus-redis-exporter` Helm Chart的版本号。
*   `REDIS_ADDRESS`: 目标Redis实例的连接地址。如果是Redis集群，通常指向其中一个节点或哨兵（根据Exporter的连接方式）。
*   `REDIS_SECRET_NAME`: 存储Redis密码的Kubernetes Secret的名称。
*   `REDIS_SECRET_KEY`: 上述Secret中，包含实际密码的键名。
*   `MONITOR_NAMESPACE`: `kube-prometheus-stack`（或您的Prometheus Operator）所在的命名空间。
*   `MONITOR_RELEASE_NAME`: `kube-prometheus-stack`的Helm Release名称。这用于正确标记`ServiceMonitor`，以便Prometheus可以发现它。

请根据您的实际环境修改此文件中的值。

## 安装脚本 (`install.sh`)

此脚本负责添加Helm仓库、更新仓库信息，并使用`.env`文件中的配置来安装或升级`redis-exporter`。

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
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# --- 安装 / 升级 ---
helm upgrade --install "${RELEASE_NAME}" prometheus-community/prometheus-redis-exporter \
  --version "${CHAR_VERSION}" --namespace "${NAMESPACE}" --create-namespace \
  \
  --set redisAddress="${REDIS_ADDRESS}" \
  \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.namespace="${MONITOR_NAMESPACE}" \
  --set serviceMonitor.labels.release="${MONITOR_RELEASE_NAME}" \
  \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=128Mi \
  --set resources.limits.cpu=1000m \
  --set resources.limits.memory=2048Mi \
  \
  --set auth.enabled=true \
  --set auth.secret.name="${REDIS_SECRET_NAME}" \
  --set auth.secret.key="${REDIS_SECRET_KEY}"
```

**脚本解析：**

1.  **加载变量**:
    *   `if [ -f .env ]; then ... fi`: 检查当前目录下是否存在`.env`文件。
    *   `export $(grep -v '^#' .env | sed 's/\r$//' | xargs)`: 如果文件存在，则读取文件内容，忽略以`#`开头的注释行，移除Windows换行符`\r`（如果存在），然后将每一行 `KEY=VALUE` 格式的内容导出为环境变量。
    *   如果`.env`文件不存在，则打印错误信息并退出。
2.  **添加仓库并更新**:
    *   `helm repo add prometheus-community ...`: 添加`prometheus-community`的Helm Chart仓库，这是`prometheus-redis-exporter` Chart的来源。
    *   `helm repo update`: 更新本地的Helm仓库索引。
3.  **安装/升级**:
    *   `helm upgrade --install "${RELEASE_NAME}" ...`: 这是核心命令。
        *   `upgrade --install`: 如果名为`${RELEASE_NAME}`的Release已存在，则升级它；如果不存在，则安装新的Release。
        *   `prometheus-community/prometheus-redis-exporter`: 指定要安装的Chart。
        *   `--version "${CHAR_VERSION}"`: 指定Chart的版本。**(注意：这里脚本中使用了`${CHAR_VERSION}`，而`.env`文件中定义的是`CHART_VERSION`。这可能是一个笔误，如果`${CHAR_VERSION}`未被其他方式设置，Helm可能会尝试安装最新版本或报错。通常应与`.env`文件中的变量名保持一致，即`${CHART_VERSION}`。)**
        *   `--namespace "${NAMESPACE}" --create-namespace`: 指定部署到的命名空间，如果该命名空间不存在，则创建它。
        *   `--set redisAddress="${REDIS_ADDRESS}"`: 设置`redis-exporter`要连接的Redis地址。
        *   `--set serviceMonitor.enabled=true`: 启用`ServiceMonitor`的创建。`ServiceMonitor`是Prometheus Operator CRD，用于声明式地配置Prometheus如何抓取目标。
        *   `--set serviceMonitor.namespace="${MONITOR_NAMESPACE}"`: 指定`ServiceMonitor`资源本身创建在哪个命名空间（通常是Prometheus Operator所在的命名空间）。
        *   `--set serviceMonitor.labels.release="${MONITOR_RELEASE_NAME}"`: 为`ServiceMonitor`设置特定的标签。`kube-prometheus-stack`通常会配置Prometheus Operator来发现带有特定`release`标签（例如，`kube-prom-stack`）的`ServiceMonitor`。
        *   `--set resources.*`: 设置`redis-exporter` Pod的资源请求（requests）和限制（limits）。
        *   `--set auth.enabled=true`: 启用Redis密码认证。
        *   `--set auth.secret.name="${REDIS_SECRET_NAME}"`: 指定包含Redis密码的K8s Secret的名称。
        *   `--set auth.secret.key="${REDIS_SECRET_KEY}"`: 指定Secret中密码所在的键。

执行此脚本来部署`redis-exporter`：

```shell
bash install.sh
```

## 初步验证 (`status.sh`)

部署完成后，可以使用以下脚本快速检查`redis-exporter`相关资源的状态。

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

**脚本解析：**

1.  **加载变量**: 同`install.sh`，从`.env`文件加载配置。
2.  **获取资源**:
    *   `kubectl get all -n ${NAMESPACE}`: 列出在`${NAMESPACE}`（即我们部署`redis-exporter`的命名空间）中的所有核心资源，如Pods, Services, Deployments, ReplicaSets。您应该能看到`redis-exporter`的Pod正在运行，以及相关的Service。
    *   `kubectl get pvc -n ${NAMESPACE}`: 列出该命名空间下的PersistentVolumeClaims。对于`redis-exporter`本身，通常不需要PVC，但这是一个通用的检查命令。

执行验证脚本：

```shell
bash status.sh
```

您应该看到`redis-exporter`的Pod处于`Running`状态，并且相关的Service已创建。

## 进阶验证

初步验证只是确保Pod正常运行，我们还需要进一步确认`redis-exporter`是否能正确暴露指标并被Prometheus抓取。

1.  **创建临时应用，访问`redis-exporter`的metrics地址**

    我们可以在集群内部署一个临时的Pod，并从该Pod内部尝试访问`redis-exporter`暴露的`/metrics`端点。

    ```shell
    # 启动一个临时pod用于测试 (例如在 default 命名空间)
    kubectl run -i --tty --rm debug --image=curlimages/curl --restart=Never -- sh
    ```

    进入临时Pod的shell后，执行以下命令（请根据您的`.env`配置替换`${RELEASE_NAME}`和`${NAMESPACE}`）：

    ```shell
    # 格式为：curl http://${RELEASE_NAME}-prometheus-redis-exporter.${NAMESPACE}.svc.cluster.local:9121/metrics
    # 示例（使用.env中的默认值 my-redis-exporter 和 redis）：
    curl http://my-redis-exporter-prometheus-redis-exporter.redis.svc.cluster.local:9121/metrics
    ```
    如果看到大量的Prometheus格式的指标输出（以`# HELP`或`# TYPE`开头），则表示`redis-exporter`正在正常工作并暴露指标。

2.  **访问`prometheus`的`/targets`页面**

    打开您的Prometheus UI（通常通过端口转发或Ingress访问），导航到 "Status" -> "Targets" 页面。
    由于我们启用了`ServiceMonitor`并正确设置了标签，Prometheus Operator应该会自动配置Prometheus实例来抓取`redis-exporter`。您应该能在Target列表中找到与`redis-exporter`相关的条目，并且其状态为`UP`。

3.  **访问`grafana`并导入面板**

    打开您的Grafana UI，导入社区推荐的Redis Dashboard。一个常用的Dashboard ID是 `11835` (Redis Exporter Dashboard)。
    *   在Grafana左侧菜单，点击 "+" -> "Import"。
    *   在 "Import via grafana.com" 字段输入 `11835` 并点击 "Load"。
    *   选择您的Prometheus数据源。
    *   点击 "Import"。
        如果一切配置正确，您应该能看到关于Redis实例的各种监控图表。

## 更新应用

如果需要修改配置（例如，更改Redis地址、资源限制或Chart版本），您可以：

1.  修改`.env`文件中的相应变量。
2.  如果需要更改`install.sh`中的Helm `set`参数，也一并修改。
3.  重新执行`install.sh`脚本：

    ```shell
    bash install.sh
    ```
    Helm的`upgrade --install`命令会智能地应用更改。

## 卸载应用 (`uninstall.sh`)

如果不再需要`redis-exporter`，可以使用以下脚本将其卸载。

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

**脚本解析：**

1.  **加载变量**: 同上，从`.env`文件加载配置。
2.  **卸载Release**:
    *   `helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}`: 使用Helm卸载在`${NAMESPACE}`命名空间中名为`${RELEASE_NAME}`的Release。这将删除由该Chart创建的所有K8s资源（Deployment, Service, ServiceMonitor等）。

执行卸载脚本：

```shell
bash uninstall.sh
```

## 总结

通过使用Helm和精心编写的辅助脚本，我们可以非常方便地在Kubernetes集群中部署、配置和管理`redis-exporter`。结合`ServiceMonitor`与`kube-prometheus-stack`，可以轻松实现对Redis的自动化监控。这种声明式和可重复的方法是云原生时代管理应用的最佳实践之一。

---