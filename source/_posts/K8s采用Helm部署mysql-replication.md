---
title: K8s采用Helm部署mysql-replication
tags:
  - Linux
  - K8s
  - Helm
  - Mysql
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250607051028585.png'
toc: true
abbrlink: e9954717
date: 2025-06-11 18:43:46
---

在构建高可用、数据驱动的后端系统中，数据库的稳定性和可扩展性是基石。传统的MySQL主从复制（Replication）是保障数据冗余和读写分离的经典方案。当我们将应用迁移到云原生环境时，如何在Kubernetes上高效、可靠地部署和管理MySQL主从集群，就成了一个重要课题。

本文将以一个后端架构师的视角，分享如何利用Helm这一强大的Kubernetes包管理工具，快速部署一套带监控的MySQL主从复制集群。我们将通过一个标准化的项目结构，实现配置、安装、验证和生命周期管理的全流程自动化。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-replication), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-replication)

## 核心优势

- **声明式部署**：通过Helm Chart和配置文件，精确定义MySQL集群的每一个组件和参数。
- **高可用架构**：一主多从（Master-Secondary）的复制架构，天然支持读写分离和故障转移。
- **自动化管理**：使用简单的Shell脚本封装Helm命令，实现一键式安装、升级和卸载。
- **可观测性**：内置Prometheus Exporter，并自动创建`ServiceMonitor`，无缝对接现有的Prometheus监控体系。
- **配置与逻辑分离**：通过`.env`文件管理所有可变参数，使安装脚本更具通用性和可复用性。

---

## 项目文件结构

一个良好组织的项目结构是高效运维的开端。我们的部署项目包含以下核心文件：

- `.env`: 环境变量文件，用于存放所有自定义配置，如密码、副本数、命名空间等。
- `install.sh`: 核心安装脚本，负责执行Helm命令来部署或升级MySQL集群。
- `uninstall.sh`: 卸载脚本，用于清理集群中的所有相关资源。
- `status.sh`: 状态检查脚本，用于快速验证Pod是否正常运行。
- `README.md`: 项目说明文档，提供完整的操作指引。

接下来，我们将深入解析每个环节的实现细节。

---

## 第一步：环境配置 (`.env`)

配置与代码分离是DevOps的最佳实践。我们将所有可变参数都集中在`.env`文件中，便于管理和修改，而无需触碰核心部署逻辑。

```shell
# 命名空间
NAMESPACE="mysql"
# helm的release名称
RELEASE_NAME="my-mysql-replication"
# helm的chart版本
CHART_VERSION="10.3.0"
# 存储类名称
STORAGE_CLASS_NAME="nfs"

# MySQL root用户密码
MYSQL_ROOT_PASSWORD="YOUR_PASSWORD"
# MySQL 数据库名称
MYSQL_DATABASE="test"
# MySQL 用户名称
MYSQL_USER="lbs"
# MySQL 用户密码
MYSQL_PASSWORD="YOUR_PASSWORD"
# MySQL 复制用户名称
MYSQL_REPLICATION_USERNAME="replicator"
# MySQL 复制用户密码
MYSQL_REPLICATION_PASSWORD="YOUR_PASSWORD"

# MySQL 从节点（secondary）的副本数量
MYSQL_SECONDARY_REPLICAS=2

# Prometheus 监控组件所在的命名空间
PROMETHEUS_NAMESPACE="monitoring"
# Prometheus Operator 用于发现 ServiceMonitor 的标签值 (通常是 helm release 的名称)
PROMETHEUS_RELEASE_LABEL="kube-prom-stack"
```

**关键配置解析**：

- `STORAGE_CLASS_NAME`: **至关重要**。请确保该存储类在您的K8s集群中已存在且可用。有状态应用的数据持久化依赖于此。
- `MYSQL_SECONDARY_REPLICAS`: 灵活配置从节点的数量，轻松实现读能力的水平扩展。
- `PROMETHEUS_*`: 这几个参数是实现自动化监控的关键，它告诉Helm Chart在哪里创建`ServiceMonitor`以及如何让Prometheus Operator发现它。

---

## 第二步：一键安装 (`install.sh`)

这个脚本是整个自动化部署的核心。它封装了所有`helm`命令和参数，使得部署过程可重复且不易出错。

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
helm upgrade --install ${RELEASE_NAME} bitnami/mysql --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --create-namespace \
  \
  --set architecture=replication \
  --set-string global.storageClass="${STORAGE_CLASS_NAME}" \
  \
  --set-string auth.rootPassword="${MYSQL_ROOT_PASSWORD}" \
  --set-string auth.database="${MYSQL_DATABASE}" \
  --set-string auth.username="${MYSQL_USER}" \
  --set-string auth.password="${MYSQL_PASSWORD}" \
  --set-string auth.replicationUser="${MYSQL_REPLICATION_USERNAME}" \
  --set-string auth.replicationPassword="${MYSQL_REPLICATION_PASSWORD}" \
  \
  --set primary.persistence.size=16Gi \
  --set primary.resources.requests.cpu=250m \
  --set primary.resources.requests.memory=512Mi \
  --set primary.resources.limits.cpu=2000m \
  --set primary.resources.limits.memory=4096Mi \
  \
  --set secondary.replicaCount=${MYSQL_SECONDARY_REPLICAS} \
  --set secondary.persistence.size=16Gi \
  --set secondary.resources.requests.cpu=250m \
  --set secondary.resources.requests.memory=512Mi \
  --set secondary.resources.limits.cpu=2000m \
  --set secondary.resources.limits.memory=4096Mi \
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

**脚本亮点分析**：

- `helm upgrade --install`: 这是一个幂等操作。如果Release不存在，它会执行安装；如果已存在，它会执行升级。这使得同一个脚本可以同时用于初始部署和后续更新。
- `--create-namespace`: 自动创建目标命名空间，无需手动操作。
- `--set architecture=replication`: 明确指示Bitnami MySQL Chart使用主从复制架构。
- `--set-string`: 用于传递密码等字符串类型的值，避免Helm将其解析为数字。
- `metrics.serviceMonitor.enabled=true`: 启用`ServiceMonitor`的创建，这是与Prometheus Operator集成的标准方式。

**执行安装**:

只需在项目根目录下执行脚本即可。

```shell
bash install.sh
```

---

## 第三步：多维度验证

部署完成后，我们需要从不同层面验证集群的健康状态。

### 1. 初步验证

运行`status.sh`脚本（内容通常是`kubectl get pods -n ${NAMESPACE}`）来快速查看所有Pod是否都处于`Running`状态。

```shell
bash status.sh
```

### 2. 进阶验证：数据库连接测试

我们需要进入集群内部，亲自连接主从节点，以确认复制链路是否正常。

**获取root密码**

```shell
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql my-mysql-replication -o jsonpath="{.data.mysql-root-password}" | base64 -d)
```

**启动一个临时的MySQL客户端**

```shell
kubectl run my-mysql-replication-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.37-debian-12-r2 --namespace mysql --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash
```

**连接主节点和从节点**

在临时客户端的Pod内，分别执行以下命令：

```shell
# 连接主节点
mysql -h my-mysql-replication-primary.mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"

# 连接从节点
mysql -h my-mysql-replication-secondary.mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```

**K8s内部服务地址**

您的Java、Go或任何其他微服务可以通过以下地址访问数据库：
- **主节点 (写)**: `my-mysql-replication-primary.mysql.svc.cluster.local:3306`
- **从节点 (读)**: `my-mysql-replication-secondary.mysql.svc.cluster.local:3306` (这是一个Headless Service，会轮询到所有从节点)

### 3. 监控验证

检查与Prometheus的集成是否成功。

1.  访问您的Prometheus UI，在`Status -> Targets`页面，查找名为`serviceMonitor/mysql/my-mysql-replication-metrics`的条目，确保其状态为`UP`。
2.  访问您的Grafana，导入官方推荐或社区提供的Dashboard（例如ID: `14057`），查看MySQL的各项性能指标是否正常显示。

---

## 第四步：应用生命周期管理

### 更新应用
当需要调整配置时（例如，增加从节点副本数），只需修改`.env`文件中的`MYSQL_SECONDARY_REPLICAS`值，然后重新执行`install.sh`即可。Helm会自动计算差异并应用更新。

### 卸载应用
我们提供了专门的卸载脚本。

**1. 执行卸载脚本**

```shell
bash uninstall.sh
```
该脚本通常包含`helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}`命令。

**2. （可选）手动清理PVC**
Helm默认不会删除`PersistentVolumeClaim`（PVC），这是为了防止数据意外丢失。如果确认不再需要这些数据，可以手动删除。

```shell
# 加载变量
source .env

# 查看pvc
kubectl get pvc -n ${NAMESPACE}

# 删除pvc（可能有多个pvc要删除）
kubectl delete pvc [pvc名称] -n ${NAMESPACE}
```

---

## 总结

通过结合Helm、标准化的配置文件和自动化脚本，我们成功地在Kubernetes上部署了一套健壮、可观测、易于管理的MySQL主从复制集群。这种模式不仅极大地提升了部署效率，也为后续的系统维护和扩展奠定了坚实的基础。

作为后端系统构建者，掌握此类云原生环境下的数据解决方案是至关重要的。它使我们能够将更多精力聚焦于应用层开发，同时确保底层数据服务的高性能与高可用性。