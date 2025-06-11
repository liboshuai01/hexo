---
title: K8s采用Helm部署nginx
tags:
  - Linux
  - K8s
  - Helm
  - Nginx
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250612033146788.png'
toc: true
abbrlink: fdb2064a
date: 2025-06-12 03:30:40
---

在云原生时代，Kubernetes (K8s) 已成为容器编排的事实标准，而Nginx作为高性能的反向代理和Web服务器，是K8s生态中最常部署的应用之一。直接编写和管理K8s的YAML清单文件可能变得复杂和繁琐，尤其是在涉及多环境配置、应用更新和依赖管理时。

Helm，作为Kubernetes的包管理器，极大地简化了这一过程。它允许我们将应用打包成可重用的“Chart”，通过版本控制和简单的配置覆盖，实现一键式部署、升级和回滚。

本文将详细介绍一种生产级的实践：如何利用Bitnami提供的优秀Nginx Helm Chart，结合外部化配置 (`.env`) 和自动化脚本 (`.sh`)，在K8s集群中快速部署一个高度可定制、且集成了Prometheus监控的Nginx服务。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/nginx), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/nginx)

## 项目结构概览

为了实现自动化和可配置性，我们的项目由以下几个核心文件组成：

*   `.env`: 环境变量文件，用于存放所有可自定义的配置，如命名空间、副本数、域名等。这使得我们的部署逻辑与配置彻底分离。
*   `install.sh`: 核心安装脚本，它会读取`.env`中的配置，并执行`helm`命令来完成Nginx的部署与更新。
*   `README.md`: 项目说明书，为使用者提供清晰的安装、验证、卸载指引。

接下来，我们将深入解析这些文件的实现细节。

## 一、核心配置文件解析 (`.env`)

将配置与代码分离是工程化的最佳实践。`.env`文件使我们无需修改安装脚本，即可轻松调整部署参数。

```shell
# 命名空间
NAMESPACE="nginx"
# helm的release名称
RELEASE_NAME="my-nginx"
# helm的chart版本
CHART_VERSION="20.1.0"

# 副本数
REPLICA_COUNT="1"

# Service 类型
SERVICE_TYPE="ClusterIP"
# 是否启用 Ingress
INGRESS_ENABLED="true"
# 指定 Ingress Controller 的 class name
INGRESS_CLASS_NAME="nginx"
# Nginx 对外暴露的域名
NGINX_HOST="nginx.lbs.com"

# Prometheus 监控组件所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 ServiceMonitor 的标签值 (通常是 helm release 的名称)
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```

这个配置文件主要分为三个部分：

1.  **Helm基础配置**:
    *   `NAMESPACE`, `RELEASE_NAME`, `CHART_VERSION`: 定义了Nginx将安装在哪个命名空间，Helm Release的名称是什么，以及使用哪个版本的Chart。这对于环境隔离和版本管理至关重要。

2.  **应用运行时配置**:
    *   `REPLICA_COUNT`: 控制Nginx Pod的副本数量，便于实现高可用和负载均衡。
    *   `SERVICE_TYPE`: 定义Service的类型。此处使用`ClusterIP`，意味着Nginx服务仅在集群内部暴露。外部访问将通过下一步的Ingress来管理。
    *   `INGRESS_ENABLED` & 相关配置: 这是暴露服务的关键。我们启用了Ingress，并指定了处理流量的`IngressClassName`（通常是您集群中已安装的Ingress Controller，如nginx-ingress-controller），同时定义了对外服务的域名`NGINX_HOST`。

3.  **可观测性配置**:
    *   `PROMETHEUS_NAMESPACE`, `PROMETHEUS_RELEASE_LABEL`: 这两个参数是实现与Prometheus自动集成的核心。它会指示Helm Chart创建一个`ServiceMonitor`资源，并打上正确的标签，以便部署在`monitoring`命名空间下的Prometheus Operator能够自动发现这个Nginx服务并开始抓取其性能指标。

## 二、自动化安装脚本 (`install.sh`)

这个脚本是整个自动化部署的执行者。它逻辑清晰，将复杂的`helm`命令封装成简单的一步执行。

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
helm upgrade --install ${RELEASE_NAME} bitnami/nginx \
  --version ${CHART_VERSION} --namespace ${NAMESPACE} --create-namespace \
  \
  --set replicaCount=${REPLICA_COUNT} \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=128Mi \
  --set resources.limits.cpu=512m \
  --set resources.limits.memory=2048Mi \
  \
  --set service.type=${SERVICE_TYPE} \
  \
  --set ingress.enabled=${INGRESS_ENABLED} \
  --set ingress.ingressClassName=${INGRESS_CLASS_NAME} \
  --set ingress.hostname=${NGINX_HOST} \
  --set ingress.path="/" \
  \
  --set cloneStaticSiteFromGit.gitSync.resources.requests.cpu=100m \
  --set cloneStaticSiteFromGit.gitSync.resources.requests.memory=128Mi \
  --set cloneStaticSiteFromGit.gitSync.resources.limits.cpu=256m \
  --set cloneStaticSiteFromGit.gitSync.resources.limits.memory=1024Mi \
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

让我们来剖析这个脚本的关键部分：

1.  **加载配置**: `source .env`命令将`.env`文件中的变量加载到当前shell会话，使得后续的`helm`命令可以直接使用 `${NAMESPACE}` 这样的语法。
2.  **Helm仓库管理**: `helm repo add` 和 `helm repo update` 确保我们使用的是最新的Bitnami Chart列表。
3.  **核心部署命令 `helm upgrade --install`**:
    *   `upgrade --install`: 这是一个幂等操作。如果名为`${RELEASE_NAME}`的Release不存在，它会执行安装；如果已存在，它会执行升级。这使得同一个脚本可以同时用于初始安装和后续更新。
    *   `--namespace ${NAMESPACE} --create-namespace`: 指定安装的命名空间，如果该命名空间不存在，则自动创建。
    *   `--set ...`: 这是Helm最强大的功能之一。通过`--set`标志，我们将`.env`文件中加载的变量值动态地传递给Helm Chart，覆盖其默认配置。
        *   **资源配额 (`resources`)**: 我们为Nginx主容器、git-sync边车容器以及metrics-exporter边车容器都精细地设置了CPU和内存的`requests`（请求值）和`limits`（限制值）。这是保障集群稳定性和资源公平性的重要措施。
        *   **网络配置 (`service`, `ingress`)**: 将`.env`中的网络相关配置应用到Chart中。
        *   **监控配置 (`metrics`)**: 通过设置`metrics.enabled=true`来启用Nginx自带的Exporter，并通过`metrics.serviceMonitor.enabled=true`来创建`ServiceMonitor`，从而无缝对接到Prometheus监控体系。

## 三、部署与验证流程

遵循`README.md`中的指引，我们可以轻松完成整个流程。

**1. 前提准备**

确保您的环境已安装`kubectl`和`helm`，并且已正确配置好K8s集群的访问凭证。

**2. 安装应用**

首先，根据您的实际环境（如Ingress Controller的Class Name，期望的域名等）修改`.env`文件。然后直接执行安装脚本：

```shell
bash install.sh
```

**3. 配置`hosts`文件**

为了能在本地通过域名访问，需要编辑本地的`hosts`文件（Linux/macOS位于`/etc/hosts`，Windows位于`C:\Windows\System32\drivers\etc\hosts`），添加如下映射：

```
[任意ingress-nginx节点IP] nginx.lbs.com

# 例如
# 192.168.6.202 nginx.lbs.com
```
> **提示**: 您可以通过 `kubectl get svc -n <ingress-controller-namespace>` 查看Ingress Controller的外部IP。

**4. 验证应用**

*   **初步验证**: 访问`http://nginx.lbs.com`，看到Nginx的欢迎页面即表示Ingress和Nginx服务本身已正常工作。
*   **监控验证**:
    1.  访问您的Prometheus UI界面。
    2.  导航到 "Status" -> "Targets" 页面。
    3.  在列表中查找与Nginx相关的Target，它的状态应该是`UP`，这表明Prometheus已成功发现并开始抓取Nginx的指标。

## 四、应用更新与卸载

*   **更新应用**: 得益于`helm upgrade`的幂等性，更新应用非常简单。只需修改`.env`文件中的配置（例如，将`REPLICA_COUNT`改为`3`），然后重新执行`install.sh`即可。Helm会自动计算差异并应用变更。

*   **卸载应用**: 执行提供的卸载脚本（`uninstall.sh`，其内容通常为`helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}`），即可清理所有由该Chart创建的K8s资源。需要注意的是，Helm默认不会删除持久卷声明（PVC），以防数据丢失，您可以根据需要手动清理。

## 总结

通过将Helm Chart、外部化配置（`.env`）和自动化脚本（`.sh`）相结合，我们构建了一套健壮、灵活且易于维护的Nginx部署方案。该方案不仅实现了业务应用的快速部署，更重要的是，它从一开始就集成了资源管理和可观测性，完全符合云原生时代对后端系统高可用、高性能的要求。

这种模式可以被广泛应用到任何其他需要在Kubernetes上部署的应用，是每一位后端及DevOps工程师都应该掌握的实用技能。