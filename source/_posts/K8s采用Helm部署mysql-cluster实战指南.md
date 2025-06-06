---
title: K8s采用Helm部署mysql-cluster实战指南
tags:
  - Linux
  - K8s
  - Helm
  - mysql
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250607051028585.png'
toc: true
abbrlink: 99c439e4
date: 2025-06-07 05:07:21
---

在构建现代化、可扩展的 Web 应用程序时，一个高可用的数据库是不可或缺的基石。MySQL 的主从（Primary-Secondary）复制架构是实现读写分离、负载均衡和故障转移的经典方案。然而，在 Kubernetes (K8s) 环境中手动部署和管理这样一个有状态的集群，会涉及复杂的配置、网络和存储管理。

幸运的是，[Helm](https://helm.sh/) 作为 K8s 的包管理器，极大地简化了这一过程。通过使用经过社区验证的 Helm Chart，我们可以实现一键式、可重复、易于管理的 MySQL 集群部署。本文将基于 [Bitnami MySQL Chart](https://github.com/bitnami/charts/tree/main/bitnami/mysql)，提供一份详尽的实战指南，带您一步步在 K8s 上部署一个生产可用的 MySQL 主从复制集群。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-cluster), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-cluster)

### 准备工作

在开始之前，请确保您已具备以下环境：

1.  一个正常运行的 Kubernetes 集群。
2.  `kubectl` 命令行工具已安装并配置好集群访问权限。
3.  `helm` v3 命令行工具已安装。
4.  集群中已配置一个可用的 `StorageClass` 用于持久化存储。本指南示例使用 `nfs`，您需要根据自己的环境进行调整。

### 部署策略与项目结构

为了使部署过程标准化和可复用，我们采用脚本化的方式进行管理。首先，创建以下项目文件结构：

```
mysql-k8s-deployment/
├── .env           # 环境变量配置文件
├── install.sh     # 安装与更新脚本
├── status.sh      # 状态检查脚本
└── uninstall.sh   # 卸载脚本
```

这种结构将配置（`.env`）与操作（`.sh` 脚本）分离，使部署流程更加清晰和易于维护。

### 第一步：定义环境变量 (`.env`)

创建 `.env` 文件，用于集中管理所有可配置的参数。这使得修改配置时无需触碰核心部署脚本。

```ini
# .env

# Kubernetes 命名空间
NAMESPACE="mysql"
# Helm Release 名称
RELEASE_NAME="my-mysql-cluster"
# Helm Chart 版本
CHART_VERSION="10.3.0"
# 持久化存储类名称
STORAGE_CLASS_NAME="nfs"

# MySQL root 用户密码
MYSQL_ROOT_PASSWORD="YOUR_STRONG_PASSWORD"
# 默认创建的业务数据库名称
MYSQL_DATABASE="test"
# 默认创建的业务用户名称
MYSQL_USER="lbs"
# 业务用户密码
MYSQL_PASSWORD="YOUR_STRONG_PASSWORD"
# MySQL 复制用户名称
MYSQL_REPLICATION_USERNAME="replicator"
# MySQL 复制用户密码
MYSQL_REPLICATION_PASSWORD="YOUR_STRONG_PASSWORD"

# MySQL 从节点（secondary）的副本数量
MYSQL_SECONDARY_REPLICAS=2
```

**说明:**
*   **安全提示**: 请务必将 `YOUR_STRONG_PASSWORD` 替换为高强度的密码。在生产环境中，推荐使用如 Vault 或 K8s 外部密钥管理服务来管理敏感信息，而非直接存储在文件中。
*   `STORAGE_CLASS_NAME`: 请确保该存储类在您的 K8s 集群中真实存在。
*   `MYSQL_SECONDARY_REPLICAS`: 定义了从节点的数量，您可以根据读取压力轻松地对其进行扩缩容。

### 第二步：编写安装脚本 (`install.sh`)

此脚本是部署的核心，它利用 `helm upgrade --install` 命令，该命令是幂等的——如果 Release 不存在，则执行安装；如果已存在，则执行升级。这使得初次部署和后续更新可以使用同一个脚本。

```shell
#!/usr/bin/env bash
# install.sh

# --- 加载变量 ---
if [ -f .env ]; then
    source .env
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

# --- 添加 Bitnami Helm 仓库并更新 ---
echo ">>> 添加并更新 Helm 仓库..."
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# --- 安装 / 升级 MySQL 集群 ---
echo ">>> 开始部署/更新 MySQL 集群: ${RELEASE_NAME} 到命名空间: ${NAMESPACE}..."
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
  --set secondary.resources.limits.memory=4096Mi

echo ">>> 部署完成！"
```

**关键参数解析:**
*   `--create-namespace`: 如果命名空间不存在，会自动创建。
*   `--set architecture=replication`: 这是最重要的参数，它告诉 Helm Chart 我们要部署的是主从复制架构，而非单点模式。
*   `--set-string global.storageClass`: 为所有 Pod (主、从) 的持久化卷声明 (PVC) 指定统一的 `StorageClass`。
*   `auth.*`: 设置所有相关的数据库用户和密码。
*   `primary.*` & `secondary.*`: 分别为主节点和从节点配置持久化存储大小 (`persistence.size`) 和 CPU/内存资源 (`resources`)。设置合理的 `requests` 和 `limits` 对保障 K8s 集群的稳定性至关重要。

### 第三步：执行安装

现在，赋予脚本执行权限并运行它。

```shell
chmod +x install.sh
bash install.sh
```

Helm 将开始工作，您会看到它下载 Chart 并向 K8s API 提交资源创建请求。这可能需要几分钟时间，因为 K8s 需要拉取 MySQL 镜像、创建 `StatefulSet`、并通过 `StorageClass` 动态分配 `PersistentVolume`。

### 第四步：部署验证

部署完成后，我们需要验证集群是否按预期正常运行。

#### 1. 初步状态检查

我们准备一个简单的状态检查脚本 `status.sh`。

```shell
#!/usr/bin/env bash
# status.sh

# --- 加载变量 ---
if [ -f .env ]; then
    source .env
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

echo "--- 查看 ${NAMESPACE} 命名空间下的所有资源 ---"
kubectl get all -n ${NAMESPACE}

echo -e "\n--- 查看 ${NAMESPACE} 命名空间下的 PVC 状态 ---"
kubectl get pvc -n ${NAMESPACE}
```

运行它：

```shell
bash status.sh
```

您应该能看到：
*   一个名为 `my-mysql-cluster-primary-0` 的主节点 Pod 正在 `Running`。
*   两个名为 `my-mysql-cluster-secondary-0` 和 `my-mysql-cluster-secondary-1` 的从节点 Pod 正在 `Running`。
*   对应的 `StatefulSet`、`Service` 和 `ConfigMap`。
*   三个 `PersistentVolumeClaim` (PVC) 均处于 `Bound` 状态。

#### 2. 进阶连接验证

为了确保主从复制正常工作，我们需要直接连接到数据库进行测试。

**获取 Root 密码**
密码被安全地存储在 K8s Secret 中，执行以下命令获取：
```shell
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql my-mysql-cluster -o jsonpath="{.data.mysql-root-password}" | base64 -d)
```

**启动一个临时的 MySQL 客户端 Pod**
我们从集群内部发起连接，这是最可靠的测试方式。
```shell
kubectl run my-mysql-cluster-client --rm --tty -i --restart='Never' \
  --image  docker.io/bitnami/mysql:8.0.37-debian-12-r2 \
  --namespace mysql \
  --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD \
  --command -- bash
```
> **提示**: `--rm` 参数意味着当您退出这个 Pod 的 shell 时，它会自动被删除，非常适合临时任务。

**连接到主节点（写操作）**
在刚刚启动的客户端 Pod shell 中，执行：
```shell
# 服务名称格式: {release-name}-primary.{namespace}.svc.cluster.local
mysql -h my-mysql-cluster-primary.mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```
连接成功后，尝试创建一个表并插入数据：
```sql
CREATE DATABASE IF NOT EXISTS replication_test;
USE replication_test;
CREATE TABLE messages (id INT AUTO_INCREMENT PRIMARY KEY, message VARCHAR(255));
INSERT INTO messages (message) VALUES ('Hello from Primary!');
exit;
```

**连接到从节点（读操作）**
在同一个客户端 Pod shell 中，连接到从节点服务。这个 Service 会将请求负载均衡到所有健康的从节点上。
```shell
# 服务名称格式: {release-name}-secondary.{namespace}.svc.cluster.local
mysql -h my-mysql-cluster-secondary.mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```
连接成功后，验证数据是否已同步：
```sql
USE replication_test;
SELECT * FROM messages;
```
如果您能看到 `"Hello from Primary!"` 这条记录，恭喜您，主从复制已成功建立！

### 更新与卸载

#### 更新应用
得益于 `helm upgrade --install` 的幂等性，更新集群配置变得非常简单。例如，要将从节点数量从 2 个增加到 3 个：

1.  修改 `.env` 文件：`MYSQL_SECONDARY_REPLICAS=3`。
2.  重新运行 `install.sh` 脚本：`bash install.sh`。

Helm 会智能地对比当前状态和目标状态，并只对发生变化的资源（在这里是 `secondary` 的 `StatefulSet`）进行更新。

#### 卸载应用
我们同样使用脚本来完成卸载。

**创建 `uninstall.sh`**:
```shell
#!/usr/bin/env bash
# uninstall.sh

# --- 加载变量 ---
if [ -f .env ]; then
    source .env
else
    echo "错误: .env 文件不存在!"
    exit 1
fi

echo ">>> 正在卸载 Helm Release: ${RELEASE_NAME}..."
helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}
echo ">>> 卸载完成。注意：PVC 可能需要手动删除！"
```

**执行卸载**:
```shell
bash uninstall.sh
```
此命令会删除由 Helm 创建的所有 K8s 资源，如 `StatefulSet`, `Service`, `Pod`等。

**（可选）清理持久化卷 (PVC)**
为防止数据丢失，Helm 默认不会删除 PVC。如果您确认不再需要这些数据，可以手动删除它们：

```shell
# 1. 查看剩余的 PVC
kubectl get pvc -n mysql

# 2. 逐个删除 PVC (此操作不可逆！)
kubectl delete pvc data-my-mysql-cluster-primary-0 -n mysql
kubectl delete pvc data-my-mysql-cluster-secondary-0 -n mysql
# ... 删除所有相关的 PVC
```

### 总结

通过本篇指南，我们利用 Helm 和 Bitnami 的优秀 Chart，成功地在 Kubernetes 上部署并验证了一个高可用的 MySQL 主从复制集群。这种脚本化、配置化的方法不仅极大地提升了部署效率，还保证了环境的一致性和可重复性，为后端应用提供了稳定可靠的数据支持。

接下来，您可以基于此部署集成监控（如 Prometheus + Grafana）、备份（如 Velero）等方案，进一步完善您的数据库基础设施。希望这篇实战指南能为您的云原生之旅提供有力的帮助。