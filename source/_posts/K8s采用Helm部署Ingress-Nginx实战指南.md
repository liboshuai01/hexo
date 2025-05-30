---
title: K8s采用Helm部署Ingress-Nginx实战指南
tags:
  - Linux
  - K8s
  - Helm
  - Nginx
  - Ingress-Nginx
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505061122195.png'
toc: true
abbrlink: ffec2a5
date: 2025-05-06 11:20:16
---

在 Kubernetes (K8s) 集群中，Ingress 是管理外部访问集群内部服务的重要组件，它提供了 HTTP 和 HTTPS 路由。Ingress-Nginx 是目前最受欢迎的 Ingress Controller 实现之一。本文将详细介绍如何使用 Helm 包管理器，采用 `DaemonSet + HostNetwork` 模式在 K8s 集群中部署 Ingress-Nginx Controller，并提供完整的脚本和验证步骤。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-stack/tree/master/ingress-nginx), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/ingress-nginx)

## Ingress-Nginx 部署模式简介

在开始之前，我们先简单回顾一下 Ingress-Nginx 常见的几种部署模式：

1.  **Deployment + LoadBalancer Service:**
    *   原理: Ingress Controller Pods 由 Deployment 管理。创建一个 `type: LoadBalancer` 的 Service 指向这些 Pods。云厂商会自动创建并关联一个外部负载均衡器及公网 IP。
    *   优点: 易于与公有云集成，自动获取公网 IP 和负载均衡。
    *   缺点: 依赖云厂商支持，可能产生额外费用，网络路径相对较长。
    *   适用场景: 公有云环境。

2.  **Deployment + NodePort Service:**
    *   原理: Ingress Controller Pods 由 Deployment 管理。创建一个 `type: NodePort` 的 Service 指向这些 Pods。Ingress Controller 会暴露在集群每个节点的一个静态高位端口上。
    *   优点: 不依赖特定云厂商，部署相对简单。
    *   缺点: NodePort 端口通常在高位范围 (30000-32767)，需要外部负载均衡器将 80/443 端口的流量转发到节点的 NodePort。增加了一层转发。
    *   适用场景: 自建机房或需要手动控制负载均衡器的环境。

3.  **DaemonSet + HostNetwork:**
    *   原理: Ingress Controller Pods 由 DaemonSet 管理，确保在指定的每个节点上都运行一个 Pod。Pod 配置 `hostNetwork: true`，直接使用宿主机的网络命名空间，监听宿主机的 80/443 端口。
    *   优点: 网络路径最短，性能通常最优。
    *   缺点: Pod 直接占用宿主机端口，可能冲突。每个节点只能运行一个监听相同端口的 Ingress Controller Pod。需通过 `nodeSelector` 或 `affinity` 精确控制部署节点。
    *   适用场景: 对性能要求高、网络延迟敏感的生产环境，且有专用节点承载 Ingress 流量。

**本文重点实践 `DaemonSet + HostNetwork` 模式，并通过脚本进行部署。** 这种模式下，Ingress Controller Pod 直接监听宿主机节点的物理网络端口（如80和443），流量直接到达，无需额外的 Service 层转发，从而获得最佳性能和最低延迟。

## 准备工作

在开始部署之前，请确保满足以下条件：

1.  **Kubernetes 集群就绪**: 您需要一个正常运行的 K8s 集群。
2.  **Helm 已安装**: Helm 是 K8s 的包管理器，我们将使用它来部署 Ingress-Nginx。如果尚未安装，请参考 [Helm 官方文档](https://helm.sh/docs/intro/install/)进行安装。
3.  **端口检查**: 确保您计划部署 Ingress Controller 的 K8s 节点上的 `80` 和 `443` 端口未被其他应用占用。
4.  **卸载现有 Ingress Controller (如果存在)**: 如果集群中已存在其他 Ingress Controller (如 Traefik)，请先卸载，以避免冲突。
    例如，卸载 Traefik 的命令：
    ```shell
    helm uninstall traefik -n kube-system
    helm uninstall traefik-crd -n kube-system
    ```
5.  **脚本和配置文件**: 准备以下文件，并将它们放在同一个目录下（例如 `ingress-nginx-deploy/`）：
    *   `.env` (环境变量配置文件)
    *   `install.sh` (安装脚本)
    *   `status.sh` (状态检查脚本)
    *   `uninstall.sh` (卸载脚本)

## 部署 Ingress-Nginx

我们将使用预先准备好的脚本和配置文件来完成部署。

### 1. 配置文件 (`.env`)

该文件用于定义部署相关的变量。您可以根据实际需求修改这些值。

```ini
# 命名空间名称
NAMESPACE="ingress-nginx"
# helm的release名称
RELEASE_NAME="ingress-nginx"
# helm的chart版本
CHART_VERSION="4.12.2"
```

### 2. 安装脚本 (`install.sh`)

此脚本负责添加 Helm 仓库、更新仓库，并使用 `.env` 文件中定义的变量和固定的 Helm 参数来安装或升级 Ingress-Nginx。

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
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# --- 安装 / 升级 ---
helm upgrade --install ${RELEASE_NAME} ingress-nginx/ingress-nginx \
  --version ${CHART_VERSION} --namespace ${NAMESPACE} --create-namespace \
  \
  --set controller.hostNetwork=true \
  --set controller.dnsPolicy=ClusterFirstWithHostNet \
  --set-string controller.nodeSelector.ingress="true" \
  --set controller.kind=DaemonSet \
  --set controller.service.enabled=true \
  --set controller.service.type=NodePort
```

**脚本中的 Helm 参数解释:**

*   `--version ${CHART_VERSION}`: 指定安装的 Ingress-Nginx Helm Chart 版本，从 `.env` 文件获取。
*   `--namespace ${NAMESPACE}`: 指定安装的命名空间，从 `.env` 文件获取。如果命名空间不存在，`--create-namespace` 会自动创建它。
*   `--set controller.hostNetwork=true`: 启用 HostNetwork 模式，Ingress Controller Pod 将直接使用宿主机的网络栈，监听宿主机的80/443端口。
*   `--set controller.dnsPolicy=ClusterFirstWithHostNet`: 当 `hostNetwork` 为 `true` 时，为 Pod 设置 DNS 策略。`ClusterFirstWithHostNet` 意味着 Pod 会优先使用集群内部的 DNS 服务，但如果配置了宿主机的 `/etc/resolv.conf`，也会参考它。
*   `--set-string controller.nodeSelector.ingress="true"`: 设置节点选择器。Ingress-Nginx Controller Pods 将仅被调度到那些带有 `ingress=true` 标签的 Kubernetes 节点上。
*   `--set controller.kind=DaemonSet`: 指定 Ingress Controller 的部署类型为 `DaemonSet`。这意味着在每个被选中的节点（通过 `nodeSelector`）上都会运行一个 Controller Pod。
*   `--set controller.service.enabled=true` 和 `--set controller.service.type=NodePort`: 这会为 Ingress Controller 创建一个 `NodePort` 类型的 Service。虽然在 `hostNetwork` 模式下，外部流量可以直接访问节点 IP 的 80/443 端口，这个 Service 仍然可以被创建。它可能用于集群内部服务发现或某些特定场景，但对于典型的 HostNetwork 外部访问不是必需的。

### 3. 执行安装脚本

首先，给 `install.sh` 脚本添加执行权限：
```shell
chmod +x install.sh
```
然后执行脚本进行安装：
```shell
bash install.sh
```

### 4. 为节点打上标签

安装脚本执行后，Ingress-Nginx Controller Pods 需要被调度到特定的节点上。根据 `install.sh` 脚本中的 `--set-string controller.nodeSelector.ingress="true"` 配置，我们需要手动为期望运行 Ingress Controller 的节点打上 `ingress=true` 标签。

执行以下命令，将 `<节点名称>` 替换为您的实际节点名称：
```shell
kubectl label node [节点名称] ingress=true --overwrite

# 例如，为 k8s-node-1 节点打标签
kubectl label node k8s-node-1 ingress=true --overwrite
```
您可以为多个节点打上此标签，DaemonSet 会确保在每个打了标签的节点上都运行一个 Ingress Controller Pod。

## 验证部署

### 1. 查看状态 (`status.sh`)

使用 `status.sh` 脚本来检查 Ingress-Nginx 的部署状态。

```shell
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    export $(grep -v '^#' .env | sed 's/\r$//' | xargs)
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

# --- 执行查询命令 ---
kubectl get all -n ${NAMESPACE}
kubectl get ingressclass
```

首先，给 `status.sh` 脚本添加执行权限：
```shell
chmod +x status.sh
```
然后执行脚本查看状态：
```shell
bash status.sh
```
您应该能看到在 `${NAMESPACE}` (默认为 `ingress-nginx`) 命名空间下的 Pods (特别是 `ingress-nginx-controller-*` Pods) 处于 `Running` 状态，并且 DaemonSet 的 `DESIRED` 和 `READY` 数量与您打标签的节点数量一致。同时，`kubectl get ingressclass` 命令应该显示一个名为 `nginx` 的 IngressClass。

### 2. 安装应用进行测试

为了进一步验证 Ingress-Nginx 是否正常工作，可以部署一个测试应用并通过 Ingress 规则访问它。

使用 Helm 安装一个测试 Nginx 应用：
```shell
helm upgrade --install nginx-test-app bitnami/nginx \
  --version 20.0.3 --namespace default \
  \
  --set service.type=ClusterIP \
  \
  --set ingress.enabled=true \
  --set ingress.ingressClassName=nginx \
  --set ingress.hostname="nginx.lbs.com" \
  --set ingress.path="/" \
  \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=128Mi \
  --set resources.limits.cpu=250m \
  --set resources.limits.memory=512Mi
```
**注意**: 上述 `bitnami/nginx` 的版本 (`20.0.3`) 仅为示例，您可能需要根据 Bitnami Helm 仓库的实际情况调整。确保 `ingress.ingressClassName=nginx` 与 `kubectl get ingressclass` 输出的名称一致。

**配置 Hosts 文件**

在您的本地计算机上编辑 `hosts` 文件（通常位于 Linux/macOS: `/etc/hosts`, Windows: `C:\Windows\System32\drivers\etc\hosts`），添加一条记录，将测试域名 `nginx.lbs.com` 指向**任意一个**运行 Ingress Controller Pod 并被打上 `ingress=true` 标签的节点的 IP 地址。

```
# 将 [任意ingress-nginx节点IP] 替换为实际的节点 IP
nginx.lbs.com  [任意ingress-nginx节点IP]

# 例如:
# nginx.lbs.com  192.168.6.202
```

**访问测试应用**

在浏览器中输入 `http://nginx.lbs.com`。如果能看到 Nginx 的欢迎页面，则表明 Ingress-Nginx 已成功部署并正常工作。

**清理测试应用**

测试完成后，可以通过以下命令卸载测试应用：
```shell
helm uninstall nginx-test-app -n default
```

## 更新 Ingress-Nginx

如果需要更新 Ingress-Nginx 的配置（例如，修改 Helm Chart 中的某些参数）或升级到新版本：

1.  **修改配置**:
    *   若要升级 Chart 版本，请修改 `.env` 文件中的 `CHART_VERSION` 变量。
    *   若要调整 Helm 安装参数 (即 `install.sh` 中 `--set` 或 `--set-string` 定义的那些)，您需要直接修改 `install.sh` 脚本中对应的 `--set` 参数值。
2.  **重新执行安装脚本**:
    ```shell
    bash install.sh
    ```
    Helm 的 `upgrade --install` 命令会智能地应用更改。

## 卸载 Ingress-Nginx

### 1. 卸载脚本 (`uninstall.sh`)

使用 `uninstall.sh` 脚本来移除通过 Helm 部署的 Ingress-Nginx。

```shell
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    export $(grep -v '^#' .env | sed 's/\r$//' | xargs)
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

# --- 执行卸载命令 ---
helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}
```

### 2. 执行卸载脚本

首先，给 `uninstall.sh` 脚本添加执行权限：
```shell
chmod +x uninstall.sh
```
然后执行脚本进行卸载：
```shell
bash uninstall.sh
```
这将卸载名为 `${RELEASE_NAME}` (默认为 `ingress-nginx`) 的 Helm Release，并删除其在 `${NAMESPACE}` (默认为 `ingress-nginx`) 命名空间中创建的所有资源。

### 3. (可选) 移除节点标签

卸载 Ingress-Nginx 후, 如果不再需要在这些节点上运行 Ingress Controller，可以移除之前添加的 `ingress=true` 标签：
```shell
kubectl label node [节点名称] ingress-

# 例如，为 k8s-node-1 节点移除标签
kubectl label node k8s-node-1 ingress-
```
您可以对所有之前打过标签的节点执行此操作。

## 总结

通过本文的指引和提供的脚本，您可以有效地在 Kubernetes 集群中采用 `DaemonSet + HostNetwork` 模式部署 Ingress-Nginx。这种模式因其出色的性能和较低的网络延迟，在对网络敏感的生产环境中是一个优秀的选择。请记得根据您的具体环境和需求，通过修改 `.env` 文件或 `install.sh` 中的 Helm 参数来调整部署配置。

---