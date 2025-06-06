---
title: K8s采用Helm部署mysql-standalone实战指南
tags:
  - Linux
  - K8s
  - Helm
  - mysql
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250607042527522.png'
toc: true
abbrlink: a668abcf
date: 2025-06-07 04:23:29
---

在现代云原生应用开发中，Kubernetes (K8s) 已成为容器编排的事实标准。而 Helm 作为 K8s 的包管理器，极大地简化了复杂应用的部署和管理。本文将详细介绍如何使用 Helm 在 Kubernetes 集群中快速部署一个单节点 MySQL 实例 (mysql-standalone)，并提供相关的配置、验证及管理脚本。

作为后端开发人员，我们经常需要在开发和测试环境中快速搭建数据库服务。通过 Helm Chart，我们可以实现 MySQL 的声明式部署，版本控制以及轻松的配置管理。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-standalone), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/mysql/mysql-standalone)

## 前提条件

在开始之前，请确保您已具备以下环境和工具：

1.  **Kubernetes 集群**: 一个正常运行的 K8s 集群。
2.  **kubectl**: K8s 命令行工具，已配置好与集群的连接。
3.  **Helm**: Helm v3 或更高版本已安装。
4.  **NFS (或其他) StorageClass**: 示例中使用了名为 `nfs` 的 StorageClass，请确保您的集群中已配置可用的持久化存储类。如果使用其他存储类，请相应修改配置。

## 项目文件概览

我们将使用以下文件来管理我们的 MySQL 部署：

*   `.env`: 存储部署所需的配置变量，如命名空间、密码、存储类等。
*   `install.sh`: Helm 安装和升级脚本。
*   `status.sh`: 查看 MySQL 部署状态的脚本。
*   `uninstall.sh`: 卸载 MySQL 应用的脚本。
*   `README.md`: (本文档的原始来源) 项目说明。

## 步骤一：配置环境变量

首先，我们需要根据实际情况修改 `.env` 文件。这个文件定义了部署 MySQL 实例所需的各种参数。

```ini
# .env
# 命名空间
NAMESPACE="mysql"
# helm的release名称
RELEASE_NAME="my-mysql-standalone"
# helm的chart版本
CHART_VERSION="10.3.0"
# 存储类名称
STORAGE_CLASS_NAME="nfs"

# MySQL root用户密码
MYSQL_ROOT_PASSWORD="YOUR_SECURE_PASSWORD" # 请替换为强密码
# MySQL 数据库名称
MYSQL_DATABASE="test"
# MySQL 用户名称
MYSQL_USER="lbs"
# MySQL 用户密码
MYSQL_PASSWORD="YOUR_SECURE_USER_PASSWORD" # 请替换为强密码
```

**重要提示**:
*   请务必将 `MYSQL_ROOT_PASSWORD` 和 `MYSQL_PASSWORD` 替换为强密码。
*   `NAMESPACE` 是 MySQL 将被部署到的 K8s 命名空间。
*   `RELEASE_NAME` 是 Helm release 的名称，用于标识此次部署。
*   `STORAGE_CLASS_NAME` 必须是您 K8s 集群中已存在的 StorageClass 名称。

## 步骤二：安装 MySQL 应用

配置好 `.env` 文件后，我们可以执行 `install.sh` 脚本来部署 MySQL。

`install.sh` 脚本内容如下：
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
helm upgrade --install ${RELEASE_NAME} bitnami/mysql --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --create-namespace \
  \
  --set-string global.storageClass="${STORAGE_CLASS_NAME}" \
  \
  --set-string auth.rootPassword=${MYSQL_ROOT_PASSWORD} \
  --set-string auth.database=${MYSQL_DATABASE} \
  --set-string auth.username=${MYSQL_USER} \
  --set-string auth.password=${MYSQL_PASSWORD} \
  \
  --set primary.persistence.size=16Gi \
  \
  --set primary.resources.requests.cpu=250m \
  --set primary.resources.requests.memory=512Mi \
  --set primary.resources.limits.cpu=2000m \
  --set primary.resources.limits.memory=4096Mi
```

执行安装命令：
```bash
bash install.sh
```
这个脚本会执行以下操作：
1.  加载 `.env` 文件中的环境变量。
2.  添加 Bitnami Helm 仓库并更新。
3.  使用 `helm upgrade --install` 命令部署或升级 MySQL。
    *   `--create-namespace`: 如果指定的命名空间不存在，则创建它。
    *   `--set-string`: 用于传递 Helm Chart 的配置参数，如存储类、认证信息、持久化存储大小以及资源请求和限制。

## 步骤三：初步验证部署状态

安装完成后，我们可以使用 `status.sh` 脚本来检查 MySQL Pod 和持久卷声明 (PVC) 的状态。

`status.sh` 脚本内容如下：
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

执行状态检查命令：
```bash
bash status.sh
```
您应该能看到类似以下的输出（具体名称会根据您的 `.env` 配置而变化）：
*   一个名为 `${RELEASE_NAME}-0` (例如 `my-mysql-standalone-0`) 的 Pod 处于 `Running` 状态。
*   一个名为 `data-${RELEASE_NAME}-0` (例如 `data-my-mysql-standalone-0`) 的 PVC 处于 `Bound` 状态。
*   相关的 Service 和 StatefulSet。

## 步骤四：进阶验证 - 连接到 MySQL

为了进一步验证 MySQL 是否正常工作，我们可以连接到数据库实例。

1.  **获取 root 用户密码**
    密码存储在 Kubernetes Secret 中。使用以下命令获取并解码密码（请确保您的 `.env` 文件中的 `NAMESPACE` 和 `RELEASE_NAME` 与此处命令匹配）：

    ```bash
    # 确保 .env 文件中的变量已加载到当前 shell 环境
    source .env
    
    # 获取密码
    ACTUAL_MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace ${NAMESPACE} ${RELEASE_NAME} -o jsonpath="{.data.mysql-root-password}" | base64 -d)
    echo "获取到的 Root 密码: ${ACTUAL_MYSQL_ROOT_PASSWORD}" 
    # 注意：此处获取的是 helm chart 创建的 secret 中的密码，它应该与 .env 中设置的 MYSQL_ROOT_PASSWORD 一致。
    ```

2.  **启动 MySQL 客户端 Pod**
    我们将启动一个临时的 MySQL 客户端 Pod 来连接到我们部署的 MySQL 服务。

    ```bash
    kubectl run ${RELEASE_NAME}-client --rm --tty -i --restart='Never' \
      --image docker.io/bitnami/mysql:8.0.37-debian-12-r2 \
      --namespace ${NAMESPACE} \
      --env MYSQL_ROOT_PASSWORD=$ACTUAL_MYSQL_ROOT_PASSWORD \
      --command -- bash
    ```
    这条命令会：
    *   创建一个名为 `${RELEASE_NAME}-client` 的 Pod。
    *   `--rm`: Pod 退出后自动删除。
    *   `--tty -i`: 分配一个 TTY 并保持 stdin 打开，允许交互式会话。
    *   `--restart='Never'`: Pod 退出后不重启。
    *   `--image`: 使用 Bitnami 提供的 MySQL 客户端镜像。
    *   `--namespace`: 在我们部署 MySQL 的相同命名空间中运行。
    *   `--env MYSQL_ROOT_PASSWORD=...`: 将 root 密码传递给客户端环境（可选，便于后续连接）。
    *   `--command -- bash`: 在 Pod 内启动一个 bash shell。

3.  **连接 MySQL**
    在 MySQL 客户端 Pod 的 bash shell 中，执行以下命令连接到 MySQL 服务：
    (服务名称通常是 `${RELEASE_NAME}.${NAMESPACE}.svc.cluster.local`)

    ```bash
    # 在 ${RELEASE_NAME}-client Pod 的 shell 内部执行：
    mysql -h ${RELEASE_NAME}.${NAMESPACE}.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
    ```
    系统会提示您输入密码，输入之前获取的 `ACTUAL_MYSQL_ROOT_PASSWORD` (或者您在 `.env` 中设置的 `MYSQL_ROOT_PASSWORD`)。
    连接成功后，您会看到 `mysql>` 提示符。可以执行一些简单的 SQL 命令来验证，例如：
    ```sql
    SHOW DATABASES;
    USE test; -- 如果 .env 中 MYSQL_DATABASE="test"
    SHOW TABLES;
    CREATE TABLE IF NOT EXISTS hello_k8s (message VARCHAR(255));
    INSERT INTO hello_k8s (message) VALUES ('Hello from MySQL on Kubernetes!');
    SELECT * FROM hello_k8s;
    ```
    验证完毕后，输入 `exit` 退出 MySQL 客户端，再输入 `exit` 退出 Pod 的 bash shell，Pod 将会自动删除。

## 更新应用

如果需要修改 MySQL 的配置（例如，资源限制、版本等），您可以：
1.  修改 `.env` 文件中的相关变量。
2.  或者直接修改 `install.sh` 脚本中 `--set` 参数的值。
3.  然后重新执行 `install.sh` 脚本：
    ```bash
    bash install.sh
    ```
    Helm 的 `upgrade --install` 命令会智能地应用更改。如果 release 不存在，它会执行安装；如果已存在，则会执行升级。

## 卸载应用

如果不再需要该 MySQL 实例，可以执行卸载操作。

1.  **执行卸载脚本**
    `uninstall.sh` 脚本内容如下：
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
    执行卸载命令：
    ```bash
    bash uninstall.sh
    ```
    这将卸载 Helm release，并删除相关的 K8s 资源，如 StatefulSet、Service、Pod 等。

2.  **（可选）删除持久卷声明 (PVC)**
    默认情况下，Helm 卸载后可能会保留 PVC，以防止数据意外丢失。如果您确认不再需要这些数据，可以手动删除 PVC。

    首先，查看 PVC：
    ```bash
    # 确保 .env 文件中的变量已加载到当前 shell 环境
    source .env
    kubectl get pvc -n ${NAMESPACE}
    ```
    您会看到类似 `data-${RELEASE_NAME}-0` 的 PVC。

    然后，删除指定的 PVC：
    ```bash
    kubectl delete pvc [pvc名称] -n ${NAMESPACE}
    # 例如: kubectl delete pvc data-my-mysql-standalone-0 -n mysql
    ```

## 关键 Helm Chart 参数说明

在 `install.sh` 脚本中，我们使用了一些重要的 `--set` 参数来配置 Bitnami MySQL Chart：

*   `global.storageClass="${STORAGE_CLASS_NAME}"`: 指定用于持久化存储的 StorageClass。
*   `auth.rootPassword=${MYSQL_ROOT_PASSWORD}`: 设置 MySQL root 用户的密码。
*   `auth.database=${MYSQL_DATABASE}`: 创建一个指定名称的初始数据库。
*   `auth.username=${MYSQL_USER}`: 创建一个具有指定名称的普通用户。
*   `auth.password=${MYSQL_PASSWORD}`: 设置该普通用户的密码。此用户将拥有对 `auth.database` 的完全访问权限。
*   `primary.persistence.size=16Gi`: 设置主节点 PVC 的大小为 16GiB。
*   `primary.resources.requests.cpu` & `primary.resources.requests.memory`: Pod 的 CPU 和内存请求量。K8s 会确保 Pod 至少能获得这些资源。
*   `primary.resources.limits.cpu` & `primary.resources.limits.memory`: Pod 的 CPU 和内存上限。Pod 使用的资源不能超过这个限制。

您可以查阅 Bitnami MySQL Chart 的官方文档，了解更多可配置的参数。

## 总结

通过 Helm 和预定义的脚本，我们可以非常高效地在 Kubernetes 上部署和管理 MySQL 实例。这种方式不仅简化了初始部署，也使得后续的配置更新、版本升级和应用卸载变得更加标准化和便捷。希望本指南能帮助您快速上手，在 K8s 环境中自如地使用 MySQL。

---
