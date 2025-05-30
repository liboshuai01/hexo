---
title: K8s采用Helm部署redis-cluster实战指南
tags:
  - Linux
  - K8s
  - Helm
  - Redis
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250509140551459.png'
toc: true
abbrlink: 5c36b781
date: 2025-05-09 13:59:31
---

在现代云原生应用架构中，Redis 以其高性能和灵活性成为了缓存、会话管理和消息队列等场景的首选。当需要高可用性和数据分片时，Redis Cluster 模式便派上了用场。Kubernetes (K8s) 作为容器编排的事实标准，为部署和管理 Redis Cluster 提供了强大的平台。Helm，作为 K8s 的包管理器，则进一步简化了复杂应用的部署流程。

本文将详细介绍如何使用 Helm 在 Kubernetes 集群上部署一个高可用的 Redis Cluster。我们将通过一系列脚本来自动化配置、安装、验证和卸载过程。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-stack/tree/master/redis/redis-cluster), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/redis/redis-cluster)

## 前提准备

在开始之前，请确保您已具备以下环境和工具：

1.  **可用的 Kubernetes 集群**：您需要一个正在运行的 K8s 集群，并且 `kubectl` 已配置好访问权限。
2.  **Helm v3 已安装**：Helm是我们将要使用的包管理器。
3.  **存储类 (Storage Class)**：Redis Cluster 需要持久化存储。请确保您的 K8s 集群中有一个可用的 StorageClass (例如 NFS, Ceph, vSphere Volume 等)。本文示例中将使用名为 `nfs` 的 StorageClass，您需要根据实际情况修改。
4.  **脚本文件**：将本文后续提供的 `.env`, `install.sh`, `status.sh`, 和 `uninstall.sh` 文件保存在同一目录下。

## 1. 配置文件 (`.env`)

首先，我们需要一个 `.env` 文件来管理部署过程中使用的变量。这样可以方便地自定义配置，而无需直接修改安装脚本。

```
# 命名空间
NAMESPACE="redis"
# helm的release名称
RELEASE_NAME="my-redis-cluster"
# helm的chart版本
CHART_VERSION="12.0.6"
# 存储类名称
STORAGE_CLASS_NAME="nfs"
# redis密码
REDIS_PASSWORD="YOUR_PASSWORD"
```

**配置解析**:

*   `NAMESPACE`: 指定 Redis Cluster 将被安装到的 Kubernetes 命名空间。如果该命名空间不存在，安装脚本会自动创建它。
*   `RELEASE_NAME`: Helm Release 的名称。这是 Helm 用来追踪和管理已部署应用实例的唯一标识。
*   `CHART_VERSION`: 指定要使用的 Bitnami Redis Cluster Helm Chart 的版本。使用特定版本可以确保部署的可重现性。
*   `STORAGE_CLASS_NAME`: Redis Master 和 Replica 节点持久化数据时将使用的 Kubernetes StorageClass 名称。**请务必将其修改为您的 K8s 集群中实际存在的 StorageClass 名称。**
*   `REDIS_PASSWORD`: 设置 Redis Cluster 的访问密码。**请务必将其更改为一个强密码。**

**操作**:
在开始部署前，请根据您的环境和需求修改`.env`文件中的 `STORAGE_CLASS_NAME` 和 `REDIS_PASSWORD`。其他变量可视情况修改。

## 2. 安装脚本 (`install.sh`)

`install.sh` 脚本负责添加 Helm 仓库、更新仓库信息，并使用 `.env` 文件中定义的配置来安装或升级 Redis Cluster。

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
helm upgrade --install "${RELEASE_NAME}" bitnami/redis-cluster \
  --version "${CHART_VERSION}" --namespace "${NAMESPACE}" --create-namespace \
  --set-string global.storageClass="${STORAGE_CLASS_NAME}" \
  --set-string global.redis.password="${REDIS_PASSWORD}" \
  \
  --set redis.resources.requests.cpu=250m \
  --set redis.resources.requests.memory=512Mi \
  --set redis.resources.limits.cpu=1000m \
  --set redis.resources.limits.memory=2048Mi \
  \
  --set updateJob.resources.requests.cpu=250m \
  --set updateJob.resources.requests.memory=512Mi \
  --set updateJob.resources.limits.cpu=1000m \
  --set updateJob.resources.limits.memory=2048Mi
```

**脚本解析**:

1.  `#!/usr/bin/env bash`: 指定脚本使用 bash 执行。
2.  **加载变量**:
    *   `if [ -f .env ]`: 检查当前目录下是否存在 `.env` 文件。
    *   `export $(grep -v '^#' .env | sed 's/\r$//' | xargs)`: 如果 `.env` 文件存在，则：
        *   `grep -v '^#' .env`: 读取 `.env` 文件内容，并排除以 `#` 开头的注释行。
        *   `sed 's/\r$//'`: 移除 Windows 风格的换行符 `\r`（兼容性处理）。
        *   `xargs`: 将处理后的键值对（如 `NAMESPACE="redis"`）转换为 `export` 命令可以接受的格式，并执行 `export`，从而将这些变量设置为环境变量。
    *   如果 `.env` 文件不存在，则打印错误信息并退出。
3.  **添加仓库并更新**:
    *   `helm repo add bitnami https://charts.bitnami.com/bitnami`: 添加 Bitnami 的 Helm chart 仓库。Bitnami 提供了许多预打包好的、高质量的开源应用 Helm chart。
    *   `helm repo update`: 更新本地的 Helm chart 仓库信息，确保能拉取到最新的 chart 列表。
4.  **安装 / 升级**:
    *   `helm upgrade --install "${RELEASE_NAME}" bitnami/redis-cluster`: 这是核心命令。
        *   `upgrade --install`: 这个 Helm 命令非常方便，如果名为 `${RELEASE_NAME}` 的 Release 已经存在，它会尝试升级该 Release；如果不存在，则会进行新的安装。
        *   `${RELEASE_NAME}`: 使用从 `.env` 文件加载的 Release 名称。
        *   `bitnami/redis-cluster`: 指定要安装的 chart 名称（仓库名/chart名）。
    *   `--version "${CHART_VERSION}"`: 指定使用从 `.env` 加载的 chart 版本。
    *   `--namespace "${NAMESPACE}" --create-namespace`:
        *   `--namespace "${NAMESPACE}"`: 指定在哪个命名空间安装。
        *   `--create-namespace`: 如果该命名空间不存在，则自动创建。
    *   `--set-string global.storageClass="${STORAGE_CLASS_NAME}"`: 通过 `--set-string` (确保值被当作字符串处理) 设置 Helm chart 中的参数 `global.storageClass` 为 `.env` 中定义的 `STORAGE_CLASS_NAME`。
    *   `--set-string global.redis.password="${REDIS_PASSWORD}"`: 设置 Redis 的全局密码。
    *   `--set redis.resources...`: 为 Redis 节点 (StatefulSet) 设置资源请求 (requests) 和限制 (limits)。这有助于 K8s 调度器合理分配资源，并防止单个 Pod 过度消耗资源。
        *   `requests`: Pod 启动时保证获得的最小资源。
        *   `limits`: Pod 能使用的最大资源。
    *   `--set updateJob.resources...`: 为 Bitnami chart 中可能包含的用于更新或初始化的 Job 设置资源请求和限制。

**执行安装**:
确保 `.env` 文件配置正确后，在终端执行：
```shell
bash install.sh
```
等待 Helm 命令执行完成。

## 3. 初步验证 (`status.sh`)

安装完成后，我们可以使用 `status.sh` 脚本来快速查看 Redis Cluster 相关资源的状态。

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

**脚本解析**:

1.  **加载变量**: 同 `install.sh`，首先加载 `.env` 文件中的环境变量，主要是为了获取 `NAMESPACE`。
2.  `kubectl get all -n ${NAMESPACE}`: 列出在指定命名空间 (`${NAMESPACE}`) 下的所有主要 Kubernetes 资源，包括 Pods, Services, Deployments, StatefulSets, ReplicaSets 等。通过这个命令可以快速概览部署状态。
3.  `kubectl get pvc -n ${NAMESPACE}`: 列出在指定命名空间下的所有持久卷声明 (PersistentVolumeClaim)。这可以帮助我们确认 Redis 节点是否成功申请并绑定了持久化存储。

**执行验证**:
```shell
bash status.sh
```
**期望输出**:
您应该能看到类似如下的输出 (具体数量和名称可能因配置而异，例如默认 Redis Cluster 会创建3个 Master 和3个 Replica)：
*   StatefulSet (如 `my-redis-cluster-master`, `my-redis-cluster-replica`) 的 `DESIRED`, `CURRENT`, `READY` 状态应该是匹配的。
*   Pods (如 `my-redis-cluster-master-0/1/2`, `my-redis-cluster-replica-0/1/2`) 的状态应该是 `Running`，并且 `READY` 列显示 `1/1`。
*   Services (如 `my-redis-cluster`, `my-redis-cluster-headless`) 应该已创建并分配了 ClusterIP (除非是 Headless Service)。
*   PVCs (如 `data-my-redis-cluster-master-0`) 的状态应该是 `Bound`。

如果所有 Pods 都是 `Running` 并且 PVCs 都是 `Bound`，那么初步部署是成功的。

## 4. 进阶验证

初步验证通过后，我们可以进一步连接到 Redis Cluster 内部进行功能性测试。

1.  **获取 Redis 密码** (如果忘记了在 `.env` 中设置的密码，可以通过 K8s Secret 获取)：
    ```shell
    export REDIS_PASSWORD=$(kubectl get secret --namespace ${NAMESPACE} ${RELEASE_NAME} -o jsonpath="{.data.redis-password}" | base64 --decode)
    echo "Redis Password: $REDIS_PASSWORD"
    ```
    *   `kubectl get secret ...`: 从 Kubernetes 中获取名为 `${RELEASE_NAME}` (此处对应 Helm Release 的名称) 的 Secret。Bitnami chart 通常会将密码存储在这个 Secret 中。
    *   `--namespace ${NAMESPACE}`: 指定 Secret 所在的命名空间。
    *   `-o jsonpath="{.data.redis-password}"`: 使用 JSONPath 提取 Secret 数据中 `redis-password` 键对应的值。这个值是 Base64 编码的。
    *   `| base64 --decode`: 将提取到的 Base64 编码的密码进行解码。
    *   `export REDIS_PASSWORD=...`: 将解码后的密码存入环境变量 `REDIS_PASSWORD`。

2.  **启动一个临时的 Redis 客户端 Pod 来连接集群**:
    ```shell
    kubectl run redis-client --namespace ${NAMESPACE} --rm --tty -i \
      --env REDIS_PASSWORD_ENV="$REDIS_PASSWORD" \
      --image docker.io/bitnami/redis-cluster:7.0.15 \
      -- bash
    ```
    *   `kubectl run redis-client ...`: 启动一个名为 `redis-client` 的临时 Pod。
    *   `--namespace ${NAMESPACE}`: 在我们部署 Redis Cluster 的相同命名空间中运行客户端 Pod。
    *   `--rm`: Pod 退出后自动删除。
    *   `--tty -i`: 分配一个伪终端并保持标准输入打开，使我们可以交互式操作。
    *   `--env REDIS_PASSWORD_ENV="$REDIS_PASSWORD"`: 将上一步获取的 Redis 密码作为环境变量 `REDIS_PASSWORD_ENV` 注入到客户端 Pod 中。
    *   `--image docker.io/bitnami/redis-cluster:7.0.15`: 使用 Bitnami 提供的包含 `redis-cli` 的 Redis Cluster 镜像。版本号可以根据需要调整，但最好与集群版本接近。
    *   `-- bash`: 在 Pod 启动后执行 `bash` 命令，进入 Pod 的 shell。

3.  **在临时 Pod 中连接到 Redis 集群**:
    当您进入 `redis-client` Pod 的 shell 后，执行以下命令：
    ```shell
    # 在 redis-client Pod 内部执行（my-redis-cluster为clusterIP类型的service）
    redis-cli -c -h ${RELEASE_NAME} -a "$REDIS_PASSWORD_ENV"
    ```
    *   `redis-cli`: Redis 命令行界面工具。
    *   `-c`: 启用集群模式。这对于与 Redis Cluster 交互至关重要，`redis-cli` 会自动处理 MOVED/ASK 重定向。
    *   `-h ${RELEASE_NAME}`: 指定 Redis Cluster 的 Service 名称。Helm chart 默认会创建一个与 Release 名称同名的 Service (类型为 ClusterIP)，作为集群的入口点。
    *   `-a "$REDIS_PASSWORD_ENV"`: 使用之前注入到 Pod 中的环境变量 `REDIS_PASSWORD_ENV` 作为密码进行认证。

4.  **连接成功后，您可以执行 Redis 命令来验证集群状态**:
    在 `redis-cli` 提示符下 (`127.0.0.1:6379>`) 执行：
    ```
    > cluster info
    # 期望看到: cluster_state:ok, cluster_slots_assigned:16384, cluster_size:3 (或您配置的主节点数), ...

    > cluster nodes
    # 期望看到: 列出所有节点的信息，及其角色 (master/slave) 和连接状态。

    > set mykey "Hello K8s Redis Cluster"
    # OK
    > GET mykey
    # "Hello K8s Redis Cluster"

    > exit
    ```
    如果 `cluster info` 显示 `cluster_state:ok` 并且所有哈希槽 (16384) 都已分配，同时 `set` 和 `get` 命令能够正常工作，那么您的 Redis Cluster 已成功部署并正常运行。

5.  **退出临时 Pod**:
    在 `redis-client` Pod 的 bash shell 中输入 `exit`，即可退出并自动删除该 Pod。

## 5. K8s 集群内部访问

对于部署在同一 Kubernetes 集群内的其他应用程序，它们可以通过 Redis Cluster 的 Service 名称来访问。
格式通常为：`<service-name>.<namespace>.svc.cluster.local`。
对于本文的配置，即为：`${RELEASE_NAME}.${NAMESPACE}.svc.cluster.local` (例如：`my-redis-cluster.redis.svc.cluster.local`)。

大多数现代 Redis Cluster 客户端库只需要提供这个入口地址和密码，它们就能自动发现集群中的所有节点并与之通信。

## 6. 更新应用

如果需要更新 Redis Cluster 的配置（例如，更改资源限制、升级 Chart 版本或修改 `.env` 文件中的其他 Helm 参数），可以：

1.  修改 `.env` 文件中的相关变量。
2.  或者，如果需要更改 `install.sh` 脚本中直接 `--set` 的参数，则修改 `install.sh` 文件。
3.  然后重新执行安装脚本：
    ```shell
    bash install.sh
    ```
    由于 `install.sh` 使用的是 `helm upgrade --install` 命令，Helm 会智能地比较当前状态和目标状态，并应用必要的更改。

## 7. 卸载应用

如果不再需要 Redis Cluster，可以使用 `uninstall.sh` 脚本来清理所有相关的 Kubernetes 资源。

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

**脚本解析**:

1.  **加载变量**: 同上，加载 `.env` 文件中的环境变量，主要是为了获取 `RELEASE_NAME` 和 `NAMESPACE`。
2.  `helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}`: Helm 命令，用于卸载指定的 Release。
    *   `${RELEASE_NAME}`: 要卸载的 Release 名称。
    *   `-n ${NAMESPACE}`: 指定 Release 所在的命名空间。
        此命令会删除由该 Helm chart 创建的所有 Kubernetes 资源，包括 StatefulSets, Pods, Services, ConfigMaps, Secrets 等。

**执行卸载**:
```shell
bash uninstall.sh
```
**注意**: 默认情况下，`helm uninstall` **不会删除 PersistentVolumeClaims (PVCs)**，这是为了防止数据意外丢失。如果您也想删除 PVCs 和它们关联的 PersistentVolumes (PVs)，您需要手动执行 `kubectl delete pvc <pvc-name> -n ${NAMESPACE}`。PV 的回收策略（`ReclaimPolicy`）将决定底层存储是否被真正释放。

## 总结

通过使用 Helm 和精心编写的辅助脚本，我们可以高效、可靠地在 Kubernetes 上部署和管理 Redis Cluster。这种方法不仅简化了初始部署，也为后续的更新和维护提供了便利。希望本指南能帮助您成功地将 Redis Cluster 引入到您的云原生架构中。

---