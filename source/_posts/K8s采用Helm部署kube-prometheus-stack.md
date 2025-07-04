---
title: K8s采用Helm部署kube-prometheus-stack
tags:
  - Linux
  - K8s
  - Helm
  - Prometheus
  - Grafana
  - Alertmanager
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505262315857.png'
toc: true
abbrlink: 32b0bac
date: 2025-06-11 18:30:16
---

在云原生时代，对Kubernetes集群进行全面、实时的监控是确保系统稳定性和性能的关键。Prometheus凭借其强大的数据模型和查询语言，已成为监控领域的标准。`kube-prometheus-stack`项目将Prometheus、Grafana、Alertmanager以及一系列Exporter和CRD（自定义资源定义）打包在一起，提供了一套开箱即用的、与Kubernetes深度集成的监控解决方案。

本文将以一名后端系统架构师的视角，介绍如何利用Helm这一Kubernetes包管理器，实现`kube-prometheus-stack`的自动化、可配置化和可重复部署。我们将通过一套精心设计的脚本和配置文件，快速在任何K8s集群上构建起强大的监控体系。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/kube-prometheus-stack), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/kube-prometheus-stack)

## 一、方案设计：配置与逻辑分离

在进行任何部署之前，一个清晰的架构设计至关重要。为了实现部署的灵活性和可维护性，我们采用**配置与逻辑分离**的最佳实践。

*   **.env 文件**：作为唯一的“配置中心”，它定义了所有可变参数，如命名空间、Helm Release名称、域名、存储类以及凭证等。这使得在不同环境（开发、测试、生产）中部署时，我们只需修改此文件，而无需触碰核心部署脚本。
*   **shell 脚本** (`install.sh`, `uninstall.sh`): 作为“执行逻辑”层，负责读取`.env`中的配置，并调用Helm命令完成应用的安装、升级或卸载。这种方式将复杂的Helm参数固化为脚本，降低了手动操作的复杂性和出错率。

这种设计不仅使部署过程标准化，也为后续将其集成到CI/CD流水线中打下了坚实的基础。

## 二、准备工作：项目文件概览

在开始之前，让我们先看一下我们的项目结构。整个部署方案由以下几个核心文件组成：

*   `README.md`: 项目说明书，为使用者提供快速上手指南。
*   `.env`: 环境配置文件，用于定制化部署。
*   `install.sh`: 核心安装与升级脚本。
*   `uninstall.sh`: 卸载脚本（在此博文中我们将根据`README.md`的描述来理解其功能）。

### 1. 定制化配置 (`.env`)

这是部署的核心配置文件。在执行任何操作前，请根据您的集群环境调整以下参数。

> .env
```shell
# 命名空间
NAMESPACE="monitoring"
# helm的release名称
RELEASE_NAME="kube-prom-stack"
# helm的chart版本
CHART_VERSION="72.6.2"
# 存储类名称
STORAGE_CLASS_NAME="nfs"
# ingress的class名称
INGRESS_CLASS_NAME="nginx"
# prometheus的域名
PROMETHEUS_HOST="prometheus.lbs.com"
# grafana的域名
GRAFANA_HOST="grafana.lbs.com"
# alertmanager的域名
ALERTMANAGER_HOST="alertmanager.lbs.com"
# grafana的admin用户密码
GRAFANA_ADMIN_PASSWORD="YOUR_PASSWORD"
```
**关键参数解析**：
- `STORAGE_CLASS_NAME`: 指定持久化存储所使用的StorageClass。Prometheus、Grafana和Alertmanager都需要持久化存储来保存时序数据、仪表盘配置和告警状态。请确保您的K8s集群中已存在名为`nfs`（或您自定义的）的StorageClass。
- `INGRESS_CLASS_NAME`: 指定暴露服务的Ingress Controller。这里我们使用`nginx`。
- `..._HOST`: 为Prometheus、Grafana、Alertmanager配置的访问域名。后续需要配置DNS或本地`hosts`文件来解析这些域名。
- `GRAFANA_ADMIN_PASSWORD`: 设置Grafana的初始`admin`用户密码，请务必修改为一个强密码。

### 2. 核心安装脚本 (`install.sh`)

此脚本是整个部署过程的执行引擎。它实现了幂等性（Idempotence），无论是首次安装还是后续更新，都可以反复执行此脚本。

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
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# --- 安装 / 升级 ---
helm upgrade --install ${RELEASE_NAME} prometheus-community/kube-prometheus-stack \
  --version ${CHART_VERSION} --namespace ${NAMESPACE} --create-namespace \
  \
  --set alertmanager.enabled=true \
  --set alertmanager.ingress.enabled=true \
  --set alertmanager.ingress.ingressClassName=${INGRESS_CLASS_NAME} \
  --set alertmanager.ingress.hosts[0]=${ALERTMANAGER_HOST} \
  --set alertmanager.ingress.paths[0]="/" \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=${STORAGE_CLASS_NAME} \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.accessModes[0]="ReadWriteOnce" \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=8Gi \
  \
  --set prometheus.enabled=true \
  --set prometheus.ingress.enabled=true \
  --set prometheus.ingress.ingressClassName=${INGRESS_CLASS_NAME} \
  --set prometheus.ingress.hosts[0]=${PROMETHEUS_HOST} \
  --set prometheus.ingress.paths[0]="/" \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=${STORAGE_CLASS_NAME} \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.accessModes[0]="ReadWriteOnce" \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=32Gi \
  \
  --set grafana.enabled=true \
  --set grafana.adminPassword=${GRAFANA_ADMIN_PASSWORD} \
  --set grafana.ingress.enabled=true \
  --set grafana.ingress.ingressClassName=${INGRESS_CLASS_NAME} \
  --set grafana.ingress.hosts[0]=${GRAFANA_HOST} \
  --set grafana.ingress.path="/" \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=${STORAGE_CLASS_NAME} \
  --set grafana.persistence.accessModes[0]="ReadWriteOnce" \
  --set grafana.persistence.size=8Gi \
  \
  --set prometheusOperator.enabled=true
```
**脚本解析**：
1.  **加载变量**: 从`.env`文件中加载配置。
2.  **Helm仓库管理**: 添加`prometheus-community`的Helm仓库并更新，确保能拉取到最新的Chart。
3.  **安装/升级**:
    *   `helm upgrade --install`: 这是一个非常实用的命令。如果名为`${RELEASE_NAME}`的Release不存在，它会执行`install`；如果已存在，则执行`upgrade`。
    *   `--namespace ${NAMESPACE} --create-namespace`: 在指定的命名空间中进行操作，如果该命名空间不存在，则自动创建。
    *   `--set ...`: 这是Helm的核心功能，用于在运行时覆盖Chart中`values.yaml`的默认值。我们通过它将`.env`中的配置动态传入，实现了对Ingress、持久化存储(PVC)、组件启用/禁用等关键属性的精确控制。

## 三、部署与验证

遵循以下步骤，即可完成监控系统的部署。

### 1. 执行安装

在项目根目录下，直接运行安装脚本：

```shell
bash install.sh
```

Helm会根据脚本内容，开始在Kubernetes集群中创建所有必需的资源，包括Deployments, StatefulSets, Services, ConfigMaps, Ingresses, PVCs等。

### 2. 配置域名解析

为了通过浏览器访问我们部署的服务，需要将域名指向Ingress Controller所在的节点。编辑您本地机器的`hosts`文件（在Linux/macOS上是`/etc/hosts`，在Windows上是`C:\Windows\System32\drivers\etc\hosts`），添加如下内容：

```
# 格式：[任意ingress-nginx节点IP] [域名1] [域名2] ...
# 例如：
192.168.6.202 prometheus.lbs.com grafana.lbs.com alertmanager.lbs.com
```
其中`192.168.6.202`需要替换成您环境中`ingress-nginx-controller`服务暴露的IP地址。

### 3. 验证应用

**初步验证**：
通过`kubectl`检查Pod的运行状态，确保所有组件都处于`Running`状态。

```shell
# 加载变量以便在命令行中使用
source .env

# 查看monitoring命名空间下的所有Pod
kubectl get pods -n ${NAMESPACE}
```

**进阶验证**：
打开浏览器，分别访问之前配置的三个域名：

1.  **Prometheus**: `http://prometheus.lbs.com`
    *   如果能看到Prometheus的Web UI，说明Prometheus已成功部署和暴露。您可以在`Status -> Targets`页面查看它自动发现的监控目标。
2.  **Grafana**: `http://grafana.lbs.com`
    *   如果能看到Grafana的登录页面，说明Grafana部署成功。使用用户名`admin`和您在`.env`中设置的`GRAFANA_ADMIN_PASSWORD`登录。`kube-prometheus-stack`已经预置了大量实用的仪表盘。
3.  **Alertmanager**: `http://alertmanager.lbs.com`
    *   如果能看到Alertmanager的UI，说明告警管理器也已就绪。

## 四、应用管理

### 更新应用

得益于`helm upgrade`和`.env`文件的设计，更新应用配置变得非常简单。例如，当需要为Prometheus增加存储空间时：

1.  修改`.env`文件或直接调整`install.sh`中`--set`参数的值。
2.  重新执行`install.sh`脚本。

```shell
bash install.sh
```
Helm会自动计算出变更的差异（diff），并只更新发生变化的Kubernetes资源。

### 卸载应用

当不再需要该监控栈时，可以执行卸载脚本。

**1. 执行卸载脚本**

```shell
bash uninstall.sh
```
这个脚本内部通常会执行`helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}`，它会删除由Helm创建的所有资源。

**2. （可选）清理持久化数据 (PVC)**

为了防止数据丢失，Helm在卸载时默认会保留PVC（Persistent Volume Claim）。如果您确定不再需要这些监控数据，需要手动删除它们。

```shell
# 加载变量
source .env

# 1. 查看当前命名空间下的PVC
kubectl get pvc -n ${NAMESPACE}

# 2. 逐个删除需要清理的PVC
kubectl delete pvc [pvc名称] -n ${NAMESPACE}
```

## 五、总结

通过将`kube-prometheus-stack`的部署过程封装为脚本和配置文件，我们不仅大大简化了在Kubernetes上搭建复杂监控系统的过程，还获得了极高的可维护性和可重复性。这种“基础设施即代码”（IaC）的思路，是现代后端系统和DevOps实践的核心。

您现在拥有了一套功能强大、生产就绪的Kubernetes监控解决方案。下一步，您可以开始探索如何配置自定义的抓取任务（ServiceMonitor）、定义告警规则（PrometheusRule）以及创建个性化的Grafana仪表盘，从而让这套系统更好地服务于您的业务。