---
title: K8s采用Helm部署mysql-exporter实战指南
tags:
  - Linux
  - K8s
  - Helm
  - Mysql
  - Mysql-exporter
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250607061634826.png'
toc: true
abbrlink: a67511bd
date: 2025-06-07 06:15:13
---

作为后端开发人员，我们不仅要构建高效可扩展的应用，更要确保其稳定运行。数据库是应用的核心，对其进行全方位的监控至关重要。Prometheus 作为云原生监控领域的标准，结合各种 Exporter 可以轻松实现对各类组件的监控。

本文将提供一个实战指南，详细介绍如何使用 Helm 在 Kubernetes (K8s) 环境中部署 `mysql-exporter`，以采集 MySQL 的性能指标，并通过 Prometheus 实现监控。我们将采用一种**配置与脚本分离**的最佳实践，使整个部署过程更加清晰、可重复和易于维护。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-exporter), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-exporter)

### 项目结构概览

为了实现标准化部署，我们将项目分解为以下几个文件：

*   `.env`: 核心配置文件，用于存储所有可变参数，如命名空间、MySQL 地址、密码等。
*   `install.sh`: 安装和更新脚本，负责执行 Helm 命令。
*   `status.sh`: 状态检查脚本，用于快速验证部署的资源。
*   `uninstall.sh`: 卸载脚本，用于清理部署的资源。
*   `README.md`: 项目说明文档（即本文的基础）。

这种结构使得我们只需要关心 `.env` 文件的配置，而无需修改部署逻辑本身。

### 第一步：环境准备与配置 (`.env`)

在开始之前，我们需要创建一个 `.env` 文件来定义部署所需的所有变量。这使得配置一目了然，并且便于在不同环境中复用。

```
# .env

# --- Exporter 配置 ---
# Exporter将要部署的命名空间
NAMESPACE="mysql"
# Helm Release的名称
RELEASE_NAME="my-mysql-exporter-standalone"
# Helm Chart的版本 (建议指定一个明确的版本以确保可重复部署)
CHART_VERSION="2.10.0"

# --- 目标MySQL配置 ---
# 单机版MySQL的Service地址 (格式: <service-name>.<namespace>.svc.cluster.local)
MYSQL_HOST="my-mysql-standalone.mysql.svc.cluster.local"
# 存储MySQL密码的Kubernetes Secret的名称
MYSQL_SECRET_NAME="my-mysql-standalone"
# Secret中存储密码的Key
MYSQL_SECRET_KEY="mysql-root-password"
# 默认的mysql exporter用户
MYSQL_USER="root"

# --- Prometheus集成配置 ---
# kube-prometheus-stack 所在的命名空间
MONITOR_NAMESPACE="monitoring"
# kube-prometheus-stack 的 Helm Release 名称 (用于匹配ServiceMonitor的标签)
MONITOR_RELEASE_NAME="kube-prom-stack"
```

**配置项解析：**

*   `NAMESPACE` & `RELEASE_NAME`: 定义了 `mysql-exporter` 将被安装在哪个 K8s 命名空间以及 Helm Release 的名称。
*   `CHART_VERSION`: **强烈建议**锁定一个明确的 Chart 版本，这可以避免因上游 Chart 变更导致部署失败，确保环境的一致性和可重复性。
*   `MYSQL_HOST`: 这是 Exporter 需要连接的目标 MySQL 数据库的 K8s Service FQDN (完全限定域名)。
*   `MYSQL_SECRET_NAME` & `MYSQL_SECRET_KEY`: 这是**安全最佳实践**。我们不应将密码硬编码在配置或脚本中。这里我们指定一个已经存在的 K8s Secret 和其中的 `key`，Helm Chart 会自动从中读取密码。
*   `MONITOR_NAMESPACE` & `MONITOR_RELEASE_NAME`: 这部分是与 Prometheus 集成的关键。通过启用 `ServiceMonitor` 并设置正确的 `namespace` 和 `label` (`release=${MONITOR_RELEASE_NAME}`), Prometheus Operator 可以自动发现 `mysql-exporter` 的 Service，并将其纳入监控目标列表。

**前提准备：** 在执行安装前，请确保您已经根据自己的环境修改了 `.env` 文件，特别是目标 MySQL 的地址和密码所在的 Secret 信息。

### 第二步：部署 Exporter (`install.sh`)

安装脚本 `install.sh` 封装了所有 Helm 的复杂操作，使其可以一键执行。

```shell
#!/usr/bin/env bash

# --- 加载变量 ---
if [ -f .env ]; then
    source .env
else
    echo "错误: .env 文件不存在! 请先创建 .env 文件。"
    exit 1
fi

# --- 添加仓库并更新 ---
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# --- 安装 / 升级 ---
helm upgrade --install ${RELEASE_NAME} prometheus-community/prometheus-mysql-exporter \
  --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --create-namespace \
  \
  --set mysql.host="${MYSQL_HOST}" \
  --set mysql.user="${MYSQL_USER}" \
  --set mysql.existingPasswordSecret.name="${MYSQL_SECRET_NAME}" \
  --set mysql.existingPasswordSecret.key="${MYSQL_SECRET_KEY}" \
  \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.namespace="${MONITOR_NAMESPACE}" \
  --set serviceMonitor.additionalLabels.release="${MONITOR_RELEASE_NAME}" \
  \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=128Mi \
  --set resources.limits.cpu=200m \
  --set resources.limits.memory=256Mi
```
**执行安装**：
在项目根目录下，赋予脚本执行权限并运行：
```shell
chmod +x install.sh
bash install.sh
```

该脚本会：
1.  加载 `.env` 变量。
2.  添加 `prometheus-community` 的 Helm 仓库。
3.  执行 `helm upgrade --install` 命令。这个命令是幂等的，意味着无论是首次安装还是后续更新，都可以安全地重复执行。
4.  通过 `--set` 参数，将我们 `.env` 文件中定义的变量动态传入 Helm Chart，完成 `mysql-exporter` 的定制化部署。

### 第三步：状态验证

部署完成后，我们需要验证 `mysql-exporter` 是否正常运行并被 Prometheus 正确发现。

#### 3.1 初步验证

使用 `status.sh` 脚本可以快速查看在指定命名空间中创建的 K8s 资源。

```shell
#!/usr/bin/env bash
# status.sh
if [ -f .env ]; then source .env; fi
kubectl get all -n ${NAMESPACE}
```

**执行验证：**
```shell
bash status.sh
```

您应该能看到类似 `Pod`, `Service`, `Deployment` 等资源，并且 `Pod` 的状态为 `Running`。

#### 3.2 进阶验证

初步验证只能说明 Pod 启动了，我们还需要确保它能正确连接 MySQL 并暴露指标。

**1. 访问 Metrics 端点**

我们可以临时启动一个带 `curl` 工具的 Pod，来模拟集群内部的网络访问。

```shell
# 启动一个临时的调试Pod
kubectl run -i --tty --rm debug --image=curlimages/curl --restart=Never -- sh
```

进入临时 Pod 的 Shell 后，执行以下 `curl` 命令：
```shell
# 注意: Service 名称由 Helm Release 名称和 Chart 子名称构成
# 格式: curl http://${RELEASE_NAME}-prometheus-mysql-exporter.${NAMESPACE}.svc.cluster.local:9104/metrics

curl http://my-mysql-exporter-standalone-prometheus-mysql-exporter.mysql.svc.cluster.local:9104/metrics
```
如果您看到大量以 `# HELP` 和 `# TYPE` 开头的，类似 `mysql_global_status_threads_running...` 的文本输出，那么恭喜您，Exporter 已经成功连接到 MySQL 并暴露了指标数据！

**2. 检查 Prometheus Targets**

登录您的 Prometheus UI，导航到 **Status -> Targets** 页面。由于我们配置了 `ServiceMonitor`，Prometheus 应该会自动发现我们的 `mysql-exporter`。您应该能在列表中找到一个对应的 Target，其状态为 **UP**。

 <!-- 使用一个通用的示例图片 -->

**3. 在 Grafana 中实现可视化**

监控的最终目的是为了可视化和告警。Grafana 是展示 Prometheus 数据的最佳搭档。

1.  登录您的 Grafana。
2.  进入 **Dashboards -> Import**。
3.  在 "Import via grafana.com" 输入框中填入ID `14057`，这是社区提供的一个非常优秀的 MySQL 监控面板。
4.  点击 "Load"，并在下一步中选择您的 Prometheus 数据源。
5.  导入后，您将看到一个信息丰富的 MySQL 监控仪表盘，所有数据均来自我们刚刚部署的 `mysql-exporter`！

### 第四步：更新与卸载

#### 应用更新

得益于 `helm upgrade --install` 的幂等性，更新应用变得异常简单。当您需要修改配置（例如更换目标 MySQL，或调整资源限制）时：

1.  直接修改 `.env` 文件或 `install.sh` 中的 `--set` 参数。
2.  重新执行 `bash install.sh` 即可。

Helm 会智能地计算出变更差异 (diff)，并只更新需要改变的 K8s 资源。

#### 应用卸载

如果不再需要监控，可以使用 `uninstall.sh` 脚本来彻底清理所有相关资源。

```shell
#!/usr/bin/env bash
# uninstall.sh
if [ -f .env ]; then source .env; fi
helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}
```

**执行卸载：**
```shell
bash uninstall.sh
```
该命令会干净地移除由本次 Helm部署创建的所有 K8s 资源，包括 Deployment, Service, ServiceMonitor 等。

### 总结

通过这套基于 Helm、`.env` 文件和 Shell 脚本的标准化流程，我们实现了一个健壮、可维护的 `mysql-exporter` 部署方案。这种方法将配置与执行逻辑解耦，不仅降低了手动操作的风险，也极大地提升了在多环境（开发、测试、生产）中部署和管理监控组件的效率。作为后端开发人员，掌握此类云原生运维技能，将使我们能够更好地保障服务的稳定性和可靠性。