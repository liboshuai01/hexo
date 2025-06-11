---
title: K8s采用Helm部署nfs-subdir-external-provisioner
tags:
  - Linux
  - K8s
  - Helm
  - Nfs
  - StorageClass
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425104445050.png'
toc: true
abbrlink: cebdd494
date: 2025-06-11 18:14:21
---

在Kubernetes（K8s）集群中，为应用提供持久化存储是一个核心需求。虽然K8s本身提供了多种存储卷类型，但对于需要多节点读写（`ReadWriteMany`）的场景，或者希望在私有化环境中快速搭建一个可靠共享存储的场景，`hostPath`或K3s默认的`local-path-provisioner`便显得力不从心。它们的存储与特定节点绑定，一旦节点故障，数据访问便会中断，甚至有丢失风险。

为了解决这个问题，NFS（Network File System）提供了一个经典且高效的解决方案。它允许我们在网络中共享一个目录，让集群中的所有节点都能访问，从而为Pod提供真正的共享持久化存储。

然而，手动为每个应用创建NFS对应的`PersistentVolume`（PV）既繁琐又容易出错。这时，`nfs-subdir-external-provisioner`就派上了用场。它是一个动态存储制备器（Dynamic Provisioner），可以监听`PersistentVolumeClaim`（PVC）的创建请求，并自动在NFS服务器上创建一个子目录，然后将其注册为PV，最后与PVC进行绑定。整个过程无需人工干预。

本文将以一个后端系统专家的视角，详细介绍如何通过Helm这一强大的K8s包管理器，结合自动化脚本，快速、可靠地在K8s集群中部署`nfs-subdir-external-provisioner`，为我们的数据驱动应用提供坚实的存储基础。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/nfs-subdir-external-provisioner), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/nfs-subdir-external-provisioner)

## 前提准备

在部署Provisioner之前，我们需要先准备好NFS环境，并确保K8s节点可以访问它。

### 1. 搭建NFS服务端

首先，我们需要一台服务器作为NFS服务端。这里以CentOS为例，执行以下命令来安装并配置NFS服务。

```shell
sudo yum install -y nfs-utils rpcbind
sudo systemctl enable rpcbind
sudo systemctl start rpcbind

sudo mkdir -p /data/nfs/k8s
sudo tee -a /etc/exports <<'EOF'
/data/nfs/k8s *(insecure,rw,sync,no_root_squash)
EOF

sudo systemctl enable nfs-server
sudo systemctl start nfs-server

sudo exportfs -r
sudo exportfs
```

**解析：**
*   我们安装`nfs-utils`和`rpcbind`。
*   创建了`/data/nfs/k8s`目录作为NFS的共享根目录。
*   通过修改`/etc/exports`文件，我们将该目录共享给所有客户端（`*`），并赋予读写（`rw`）、同步写入（`sync`）和允许客户端以root身份访问（`no_root_squash`）的权限。
*   最后，启动NFS服务并使配置生效。

### 2. 在K8s节点挂载NFS客户端

为了让Kubelet能够管理NFS卷，**每一个运行Pod的K8s节点**都需要安装NFS客户端工具，并能够访问NFS共享。

> **注意**：以下命令需要在**所有**需要使用NFS存储的K8s Worker节点上执行。

```shell
sudo yum install -y nfs-utils

# 此处的master应替换为你的NFS服务端IP或主机名
sudo mkdir -p /data/nfs/k8s 

sudo mount -t nfs master:/data/nfs/k8s /data/nfs/k8s

# 验证挂载是否成功
df -h | grep /data/nfs/k8s
```
成功挂载后，我们就可以开始部署核心的Provisioner了。

## 安装应用

为了实现标准化和可重复部署，我们将所有配置和命令封装成文件。

### 1. 配置文件 `.env`

创建一个`.env`文件来集中管理所有可变配置，这使得环境迁移和参数调整变得非常简单。

> .env
```shell
# 命名空间
NAMESPACE="kube-system"
# helm的release名称
RELEASE_NAME="nfs-subdir-external-provisioner"
# helm的chart版本
CHART_VERSION="4.0.18"
# nfs服务地址
NFS_SERVER="master"
# nfs存储路径
NFS_PATH="/data/nfs/k8s"
# 存储类名称
STORAGE_CLASS_NAME="nfs"
```
**说明：**
*   `NFS_SERVER`：请修改为你的NFS服务器的实际IP或主机名。
*   `NFS_PATH`：必须与NFS服务端导出的路径一致。
*   其他变量如命名空间、版本号等可根据需求自定义。

### 2. 安装脚本 `install.sh`

接下来，编写一个强大的安装脚本，它能加载配置、添加Helm仓库并执行安装或升级操作。

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
**脚本亮点：**
*   `helm upgrade --install`：这是一个幂等操作。如果Release不存在，它会执行安装；如果已存在，则执行升级。这使得脚本可以重复运行。
*   `--set ...`：通过命令行参数，将`.env`文件中的配置动态地传递给Helm Chart，实现了配置与执行的分离。
*   `storageClass.defaultClass=true`：我们将新创建的NFS存储类（StorageClass）设置为集群的**默认**存储类。这样，当应用创建PVC时不指定`storageClassName`，将自动使用NFS。

### 3. 执行安装

现在，只需一步即可完成部署：

```shell
bash install.sh
```

### 4. 处理K3s环境下的存储类冲突 (可选)

如果您的K8s集群是K3s，它默认会安装`local-path`作为默认的`StorageClass`。为了避免冲突，并让我们的NFS成为唯一的默认存储，需要执行以下步骤。

**1. 取消`local-path`存储类的默认值设置**
```shell
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

**2. （可选）卸载`local-path`存储类**
如果不再需要，可以彻底删除它。
```shell
kubectl delete storageclass local-path
```

**3. （推荐）禁用k3s的`local-path`组件**
为了一劳永逸，可以直接在K3s的启动参数中禁用它，防止K3s重启后再次创建。

修改K3s服务配置文件，例如 `/etc/systemd/system/k3s.service`：
```
# sudo vim /etc/systemd/system/k3s.service
ExecStart=/usr/local/bin/k3s \
  server \
  # 添加此行到最后（不要添加注释）
  --disable local-storage \

# 重新加载 systemd 配置并重启 K3s 服务
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

## 验证应用

部署完成后，我们需要验证Provisioner是否正常工作。

### 初步验证

运行一个状态检查脚本（`status.sh`，其内容通常是 `helm status $RELEASE_NAME -n $NAMESPACE` 或 `kubectl get pods -n $NAMESPACE -l app=nfs-subdir-external-provisioner`）来查看Provisioner Pod是否正常运行。

```shell
bash status.sh
```
看到Pod处于`Running`状态即表示初步成功。

### 进阶验证

真正的考验是创建一个PVC，看系统是否能为其自动创建PV。

**1. 编写测试 PVC 资源配置**
> pvc-test.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  storageClassName: "nfs" # 存储类名称
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
```

**2. 创建测试 PVC**
```shell
kubectl apply -f pvc-test.yaml
```

**3. 查看测试 PVC 状态**
```shell
kubectl get pvc pvc-test
```
如果一切顺利，您将看到`STATUS`列为`Bound`。
```
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   10Mi       RWX            nfs            5s
```
`Bound`状态表明，`nfs-subdir-external-provisioner`已成功监听到PVC请求，在NFS服务器的`/data/nfs/k8s`目录下创建了一个新的子目录，并创建了对应的PV与`pvc-test`绑定。

## 应用生命周期管理

### 更新应用

得益于我们的自动化脚本，更新非常简单。只需修改`.env`文件（例如升级`CHART_VERSION`）或`install.sh`中的`--set`参数，然后重新执行安装脚本即可。
```shell
bash install.sh
```

### 卸载应用

我们同样可以创建一个`uninstall.sh`脚本（内容为`helm uninstall $RELEASE_NAME -n $NAMESPACE`），或者直接执行卸载命令。
```shell
bash uninstall.sh
```
**重要**：卸载操作会删除Provisioner的Pod、ServiceAccount、RBAC规则以及`StorageClass`，但**不会删除NFS服务器上的任何数据**。这正是持久化存储的意义所在，应用与数据生命周期分离。

## 总结

通过结合Helm和自动化Shell脚本，我们成功地为Kubernetes集群部署了一套健壮、自动化的NFS动态存储解决方案。这种方法不仅极大地简化了初始部署，更为后续的运维、升级和迁移提供了极大的便利。它为我们部署需要共享存储的微服务（如高可用WordPress、分布式文件系统），或是需要稳定持久化层的数据处理应用（如数据库、消息队列）奠定了坚实的基础。这是构建高性能、数据驱动后端系统不可或缺的一环。