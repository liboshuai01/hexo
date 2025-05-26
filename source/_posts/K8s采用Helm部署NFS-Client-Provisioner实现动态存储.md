---
title: K8s采用Helm部署NFS-Client-Provisioner实现动态存储
tags:
  - Linux
  - K8s
  - Helm
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425104445050.png'
toc: true
abbrlink: e3673e0e
date: 2025-05-05 10:43:17                                                                                                                                                               
---

在Kubernetes (K8s) 集群中，动态存储卷配置 (Dynamic Volume Provisioning) 是一项核心功能，它允许根据 PersistentVolumeClaim (PVC) 的请求自动创建 PersistentVolume (PV)。`nfs-subdir-external-provisioner` 是一个广受欢迎的外部存储供应器，它利用您现有的NFS服务器，为每个PVC在NFS共享目录下创建一个子目录作为PV。

本文将详细介绍如何准备NFS环境，并使用Helm v3在K8s集群中部署 `nfs-subdir-external-provisioner`，配置其为默认的 StorageClass。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-cookbook), [gitee](https://gitee.com/liboshuai01/k8s-cookbook/tree/master/nfs-subdir-external-provisioner)

## 前提条件

在开始之前，请确保您已具备以下条件：

1.  一个正常运行的 Kubernetes 集群 (本指南中会涉及部分节点操作，如安装NFS客户端和搭建NFS服务端)。
2.  已安装并配置好 Helm v3+ 客户端。
3.  对 `kubectl` 和 `helm` 命令有基本了解。
4.  Linux系统管理基础，特别是 `yum` 包管理和 `systemd` 服务管理。

## 1. NFS 环境准备

`nfs-subdir-external-provisioner` 依赖于一个已配置好的NFS服务器和在Kubernetes节点上安装的NFS客户端。

### 1.1 安装 NFS 客户端

所有 Kubernetes 节点（包括 Master 和 Worker 节点，如果它们都需要挂载NFS卷）上都需要安装 NFS 客户端工具，以便能够挂载 NFS 共享目录。

在所有相关节点上执行：
```bash
yum install -y nfs-utils
```
**命令解析:**
*   `yum install -y nfs-utils`: 使用 `yum` 包管理器安装 `nfs-utils` 包。该包提供了支持NFS所需的工具，如 `mount.nfs`。 `-y` 参数表示自动确认安装。

### 1.2 搭建 NFS 服务器

您可以选择集群中的一个节点或一个独立的服务器作为NFS服务端。本示例选择 `k8s-master` 节点作为NFS服务端进行配置。**请注意，在生产环境中，推荐使用专用的、高可用的NFS服务器。**

#### 1.2.1 安装 rpcbind 服务

NFS 服务器需要依赖 `rpcbind` 服务进行远程过程调用绑定。`rpcbind` 服务将RPC程序号和版本号映射到NFS服务器上的端口号。

在选定的NFS服务器节点上执行：
```bash
yum install -y rpcbind
systemctl enable rpcbind
systemctl start rpcbind
```
**命令解析:**
*   `yum install -y rpcbind`: 安装 `rpcbind` 服务。
*   `systemctl enable rpcbind`: 设置 `rpcbind` 服务开机自启。
*   `systemctl start rpcbind`: 立即启动 `rpcbind` 服务。

#### 1.2.2 创建共享目录

创建您希望通过NFS共享的目录。`nfs-subdir-external-provisioner` 将在此目录下为每个PV创建子目录。

在NFS服务器节点上执行：
```bash
mkdir -p /data/nfs/k8s
```
**命令解析:**
*   `mkdir -p /data/nfs/k8s`: 创建目录 `/data/nfs/k8s`。 `-p` 参数确保如果父目录不存在也会一并创建。

#### 1.2.3 配置导出目录

编辑NFS的导出配置文件 `/etc/exports`，指定要共享的目录以及允许访问的客户端和权限。

在NFS服务器节点上，编辑 `/etc/exports` 文件，添加如下内容：
```bash
/data/nfs/k8s *(insecure,rw,sync,no_root_squash)
```
**文件内容解析:**
*   `/data/nfs/k8s`: 要导出的共享目录路径。
*   `*`: 表示允许任何IP地址的客户端访问此共享。**在生产环境中，强烈建议将其替换为具体的IP地址或网段以增强安全性**，例如 `192.168.1.0/24`。
*   `insecure`: 允许客户端使用大于1024的非特权端口进行连接。某些NFS客户端实现需要此选项。
*   `rw`: 允许客户端对共享目录进行读写操作。
*   `sync`: 数据会同步写入磁盘和内存，确保数据一致性，但可能会略微影响性能。对于持久化存储，通常推荐使用。
*   `no_root_squash`: 当客户端以 `root` 用户身份访问共享时，NFS服务器不会将其映射为匿名用户（通常是 `nfsnobody`）。这意味着客户端 `root` 用户在共享目录中拥有与服务器上 `root` 用户相同的权限。**请谨慎使用此选项，确保了解其安全 implications。**

#### 1.2.4 启动 NFS 服务并设置开机自启

安装并启动NFS服务器主服务。

在NFS服务器节点上执行：
```bash
yum install -y nfs-utils # 如果之前未安装，此步骤也会安装
systemctl enable nfs-server
systemctl start nfs-server
```
**命令解析:**
*   `systemctl enable nfs-server`: 设置 `nfs-server` 服务（在某些系统中也可能叫 `nfs`）开机自启。
*   `systemctl start nfs-server`: 立即启动 `nfs-server` 服务。

确保 `rpcbind` 和 `nfs-server` (或 `nfs`) 两个服务均已成功启动并设置为开机自启。您可以使用 `systemctl status rpcbind` 和 `systemctl status nfs-server` 来检查状态。

#### 1.2.5 重新导出共享目录

修改 `/etc/exports` 文件后，需要让NFS服务器重新加载配置以使其生效。

在NFS服务器节点上执行：
```bash
exportfs -r
exportfs
```
**命令解析:**
*   `exportfs -r`: 重新读取 `/etc/exports` 文件，并重新导出所有目录。它会同步 `/etc/exports` 与 `/var/lib/nfs/etab` 的内容。
*   `exportfs`: 不带参数执行时，会显示当前NFS服务器导出的共享目录列表及其配置，可以用来验证配置是否已正确加载。

### 1.3 验证 NFS 挂载

完成服务器端配置后，应在至少一个Kubernetes工作节点（或任何已安装NFS客户端的机器）上测试NFS共享是否可以成功挂载和访问。

在任意一个Kubernetes工作节点（例如 `k8s-node1`）上执行：
```bash
# 如果客户端上此目录不存在，则创建作为挂载点
mkdir -p /data/nfs/k8s 

# 挂载NFS共享，将 "master" 替换为你的NFS服务器IP或主机名
mount -t nfs master:/data/nfs/k8s /data/nfs/k8s

# 检查挂载是否成功
df -h | grep /data/nfs/k8s
```
**命令解析:**
*   `mkdir -p /data/nfs/k8s`: 在客户端节点上创建一个本地目录作为NFS共享的挂载点。
*   `mount -t nfs master:/data/nfs/k8s /data/nfs/k8s`:
    *   `-t nfs`: 指定文件系统类型为NFS。
    *   `master:/data/nfs/k8s`: NFS服务器的地址（此处为 `master`，即我们之前配置的NFS服务器的主机名或IP）和服务器上共享的路径。
    *   `/data/nfs/k8s`: 客户端本地的挂载点目录。
*   `df -h | grep /data/nfs/k8s`: 显示磁盘空间使用情况，并通过 `grep` 过滤出与我们挂载点相关的信息。如果挂载成功，这里应该能看到NFS共享的空间信息。

您可以在服务器端 `${NFS_PATH}`（即 `/data/nfs/k8s`）下创建一个测试文件，然后在客户端挂载点查看该文件是否存在，以验证读写和同步。

完成验证后，如果不需要保持挂载状态，可以卸载：
```bash
umount /data/nfs/k8s
```
**重要提示**: `nfs-subdir-external-provisioner` 运行时，Kubernetes节点上的kubelet会负责实际的NFS挂载操作，因此上述手动挂载测试仅用于验证NFS环境是否配置正确。

## 2. 配置环境变量

NFS环境准备就绪后，我们开始配置 `nfs-subdir-external-provisioner` 的安装参数。创建一个 `.env` 文件来存放所有配置变量。

> .env
```
# 命名空间
NAMESPACE="kube-system"
# helm的release名称
RELEASE_NAME="nfs-subdir-external-provisioner"
# helm的chart版本
CHART_VERSION="4.0.18"
# nfs服务地址 - 确保此值与您搭建的NFS服务器地址一致
NFS_SERVER="master"
# nfs存储路径 - 确保此值与您NFS服务器上配置的共享路径一致
NFS_PATH="/data/nfs/k8s"
# 存储类名称
STORAGE_CLASS_NAME="nfs"
```

**文件解析:**

*   `NAMESPACE`: 指定 `nfs-subdir-external-provisioner` 将被安装到的Kubernetes命名空间。
*   `RELEASE_NAME`: Helm部署实例的名称。
*   `CHART_VERSION`: 要安装的 `nfs-subdir-external-provisioner` Helm Chart的版本号。
*   `NFS_SERVER`: 您的NFS服务器的地址。**确保这个值与您在步骤 1.2 中配置的NFS服务器主机名或IP地址一致。**
*   `NFS_PATH`: NFS服务器上已导出的共享目录路径。**确保这个值与您在步骤 1.2.2 和 1.2.3 中创建并导出的路径一致。**
*   `STORAGE_CLASS_NAME`: 将要创建的StorageClass的名称。

请根据您的实际NFS环境和偏好修改 `.env` 文件中的值。

## 3. 编写安装脚本 (`install.sh`)

我们将使用一个Shell脚本来自动化 `nfs-subdir-external-provisioner` 的安装过程。

> install.sh
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
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

# --- 安装 / 升级 ---
helm upgrade --install ${RELEASE_NAME} nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --version ${CHART_VERSION} --namespace ${NAMESPACE} --create-namespace \
  --set nfs.server="${NFS_SERVER}" \
  --set nfs.path="${NFS_PATH}" \
  --set storageClass.name="${STORAGE_CLASS_NAME}" \
  --set storageClass.defaultClass=true \
  --set rbac.create=true
```

**脚本解析:**

1.  `#!/usr/bin/env bash`: 指定脚本使用bash解释器。
2.  **加载变量**:
    *   检查 `.env` 文件是否存在。
    *   `export $(grep -v '^#' .env | sed 's/\r$//' | xargs)`: 读取 `.env` 文件（排除注释行和处理Windows换行符），并将定义的变量导出为环境变量，供后续的 `helm` 命令使用。
3.  **添加仓库并更新**:
    *   `helm repo add ...`: 添加 `nfs-subdir-external-provisioner` 的官方Helm Chart仓库。
    *   `helm repo update`: 更新本地Helm仓库缓存。
4.  **安装 / 升级**:
    *   `helm upgrade --install ...`: 如果release已存在则升级，否则安装。
    *   `${RELEASE_NAME}`, `${CHART_VERSION}`, `${NAMESPACE}`, `--create-namespace`: 使用环境变量和指定选项。
    *   `--set nfs.server="${NFS_SERVER}"`: 将 `.env` 中的 `NFS_SERVER` 值传递给Chart。
    *   `--set nfs.path="${NFS_PATH}"`: 将 `.env` 中的 `NFS_PATH` 值传递给Chart。
    *   `--set storageClass.name="${STORAGE_CLASS_NAME}"`: 设置创建的StorageClass的名称。
    *   `--set storageClass.defaultClass=true`: 将此NFS StorageClass设置为集群的默认StorageClass。
    *   `--set rbac.create=true`: 自动创建所需的RBAC资源。

## 4. 执行安装

1.  确保您的 `.env` 文件已根据实际情况配置完毕。
2.  赋予 `install.sh` 执行权限并运行：
    ```shell
    chmod +x install.sh
    bash install.sh
    ```

## 5. 处理默认StorageClass (K3s等环境可选)

在某些Kubernetes发行版（如K3s）中，可能预装了 `local-path` 作为默认的StorageClass。如果我们希望我们创建的 `nfs` StorageClass成为唯一的默认StorageClass，需要进行以下额外步骤：

1.  **取消 `local-path` 存储类的默认值设置:**
    ```shell
    kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
    ```
    这条命令会移除 `local-path` 的默认标记。

2.  **卸载 `local-path` 存储类 (可选):**
    如果您不再需要 `local-path`，可以将其删除：
    ```shell
    kubectl delete storageclass local-path
    ```

3.  **禁用K3s的 `local-path` 组件 (K3s特定):**
    如果您的集群是K3s，并且希望彻底禁用 `local-path` provisioner，可以修改K3s服务配置：
    编辑K3s服务文件（通常在Master节点）：
    ```shell
    # sudo vim /etc/systemd/system/k3s.service
    ```
    在 `ExecStart` 参数的末尾（`server` 或 `agent` 命令之后）添加 `--disable local-storage` 选项，例如：
    ```
    ExecStart=/usr/local/bin/k3s \
        server \
        # ... 其他参数 ...
        --disable local-storage
    ```
    然后重新加载 `systemd` 配置并重启K3s服务：
    ```shell
    sudo systemctl daemon-reload
    sudo systemctl restart k3s
    ```

## 6. 初步验证

为了快速检查StorageClass的配置情况，我们准备一个简单的状态检查脚本。

> status.sh
```shell
#!/usr/bin/env bash

# 获取 StorageClass 列表
kubectl get storageclass
```

**脚本解析:**
*   `kubectl get storageclass`: 该命令会列出集群中所有已配置的StorageClass。

执行脚本：
```shell
bash status.sh
```
您应该能看到名为 `nfs` (或您在 `.env` 中定义的 `STORAGE_CLASS_NAME`) 的StorageClass，并且其 `PROVISIONER` 应能反映 `nfs-subdir-external-provisioner`，`IS-DEFAULT-CLASS` 应该为 `true` (如果成功设为默认)。

## 7. 进阶验证：创建PVC测试

最可靠的验证方法是实际创建一个PVC，并观察其是否能成功绑定到由 `nfs-subdir-external-provisioner` 动态创建的PV。

1.  **编写测试PVC资源配置 (`pvc-test.yaml`)**

    > pvc-test.yaml
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc-test
    spec:
      storageClassName: "nfs" # 存储类名称，应与.env中STORAGE_CLASS_NAME一致
      accessModes:
        - ReadWriteMany # NFS支持多种访问模式，RWM是常用的一种
      resources:
        requests:
          storage: 10Mi # 请求10Mi的存储空间
    ```

2.  **创建测试PVC**
    ```shell
    kubectl apply -f pvc-test.yaml
    ```

3.  **查看测试PVC状态**
    ```shell
    kubectl get pvc pvc-test
    ```
    输出示例：
    ```
    NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    pvc-test   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   10Mi       RWM            nfs            5s
    ```
    如果 `STATUS` 显示为 `Bound`，表示PVC已成功创建并绑定到底层PV。您也可以通过 `kubectl get pv` 查看动态创建的PV，并到NFS服务器的 `${NFS_PATH}`（即您在 `.env` 中配置的路径，例如 `/data/nfs/k8s`）目录下检查是否有名为 `NAMESPACE-pvc-test-pvc-ID` 格式的子目录被创建。

## 8. 更新应用

如果需要更改 `nfs-subdir-external-provisioner` 的配置：

1.  修改 `.env` 文件中对应的变量值，或者直接修改 `install.sh` 文件中 `helm upgrade` 命令的 `--set` 参数。
2.  重新执行安装脚本：
    ```shell
    bash install.sh
    ```
    `helm upgrade --install` 命令会自动处理更新逻辑。

## 9. 卸载应用

如果不再需要 `nfs-subdir-external-provisioner`，可以使用以下脚本进行卸载。

> uninstall.sh
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

**脚本解析:**

1.  **加载变量**: 与 `install.sh` 类似，加载 `.env` 文件中的环境变量。
2.  `helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}`: 卸载指定的Helm release。

执行卸载脚本：
```shell
bash uninstall.sh
```
**注意**: 卸载 `nfs-subdir-external-provisioner` 只会删除其部署的Kubernetes资源。**它不会删除已经通过它动态创建的PV以及这些PV在NFS服务器上对应的子目录和数据。** 如果您也想删除这些数据，需要手动清理NFS服务器上的目录和集群中的PV/PVC对象。

## 总结

通过本文的指南，您已经学会了如何准备NFS环境（包括客户端安装和服务端搭建），并使用Helm在Kubernetes集群中部署和配置 `nfs-subdir-external-provisioner`，以实现基于NFS的动态存储卷供应。这为您的有状态应用提供了灵活可靠的持久化存储方案。确保按照您的实际环境调整 `.env` 文件，并根据需要执行特定Kubernetes发行版（如K3s）的额外步骤。

---