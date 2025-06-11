---
title: K8s采用Helm部署nfs-subdir-external-provisioner实现动态存储
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
abbrlink: e3673e0e
date: 2025-05-05 10:43:17                                                                                                                                                               
---

在Kubernetes集群中实现动态存储分配是提高资源利用率和简化应用部署的关键。本文将详细介绍如何使用Helm部署nfs-subdir-external-provisioner，为K8s集群提供基于NFS的动态存储能力。

<-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-stack), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/nfs-subdir-external-provisioner)

## 一、为什么需要动态存储供应

在Kubernetes环境中，**持久化存储**是运行有状态应用的基础。传统的手动创建PV（Persistent Volume）方式存在以下问题：
- 需要管理员预先创建大量PV
- PV与PVC（Persistent Volume Claim）的匹配过程繁琐
- 存储资源利用率低下

nfs-subdir-external-provisioner通过**动态存储供应**机制解决了这些问题：
1. 按需自动创建PV
2. 自动回收不再使用的存储资源
3. 简化存储管理流程
4. 支持多租户环境下的存储隔离

## 二、部署前提准备

### 1. 搭建NFS服务端

在NFS服务器上执行以下命令：

```shell
# 安装必要组件
sudo yum install -y nfs-utils rpcbind

# 启动并启用rpcbind服务
sudo systemctl enable rpcbind
sudo systemctl start rpcbind

# 创建共享目录
sudo mkdir -p /data/nfs/k8s

# 配置共享目录权限
sudo tee -a /etc/exports <<'EOF'
/data/nfs/k8s *(insecure,rw,sync,no_root_squash)
EOF

# 启动NFS服务
sudo systemctl enable nfs-server
sudo systemctl start nfs-server

# 刷新并验证导出
sudo exportfs -r
sudo exportfs
```

### 2. 挂载NFS客户端

在所有Kubernetes节点上挂载NFS共享：

```shell
# 安装客户端工具
sudo yum install -y nfs-utils

# 创建本地挂载点
sudo mkdir -p /data/nfs/k8s 

# 挂载NFS共享
sudo mount -t nfs master:/data/nfs/k8s /data/nfs/k8s

# 验证挂载
df -h | grep /data/nfs/k8s
```

### 3. 配置环境变量（可选）

创建`.env`文件自定义部署参数：

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

## 三、安装nfs-subdir-external-provisioner

### 1. 创建安装脚本

编写`install.sh`脚本：

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

### 2. 执行安装

```shell
# 添加执行权限
chmod +x install.sh

# 运行安装脚本
bash install.sh
```

### 3. 清理现有存储类（如使用k3s）

```shell
# 取消local-path默认存储类
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# 删除local-path存储类
kubectl delete storageclass local-path

# 禁用k3s的local-path组件
sudo vim /etc/systemd/system/k3s.service
# 添加以下参数：
ExecStart=/usr/local/bin/k3s \
  server \
  --disable local-storage

# 重新加载并重启服务
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

## 四、验证安装结果

### 1. 初步验证

```shell
# 检查Pod状态
kubectl get pods -n kube-system -l app=nfs-subdir-external-provisioner

# 检查存储类
kubectl get storageclass

# 验证默认存储类
kubectl get storageclass -o json | jq '.items[] | select(.metadata.annotations."storageclass.kubernetes.io/is-default-class" == "true")'
```

### 2. 进阶验证：测试PVC

创建测试PVC配置文件`pvc-test.yaml`：

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

应用并验证PVC：

```shell
# 创建测试PVC
kubectl apply -f pvc-test.yaml

# 检查PVC状态
kubectl get pvc pvc-test

# 理想输出
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-test   Bound    pvc-5b3d7e1e-8e4a-4b7d-9b3c-3b0b3b3b3b3b   10Mi       RWX            nfs            10s
```

## 五、应用管理与维护

### 1. 更新应用

修改`.env`或`install.sh`中的配置后，重新执行安装脚本：

```shell
bash install.sh
```

### 2. 卸载应用

创建`uninstall.sh`卸载脚本：

```shell
#!/usr/bin/env bash

source .env

helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}
kubectl delete storageclass ${STORAGE_CLASS_NAME}
```

执行卸载：

```shell
bash uninstall.sh
```

## 六、总结

通过本文的部署流程，我们成功实现了：

1. **自动化存储供应**：应用程序只需声明PVC即可自动获得存储资源
2. **资源高效利用**：按需分配存储空间，避免资源浪费
3. **简化运维流程**：通过Helm实现一键部署和升级
4. **多租户支持**：每个命名空间自动创建独立子目录

nfs-subdir-external-provisioner特别适合以下场景：
- 开发测试环境快速搭建
- 中小规模生产环境
- 需要共享存储的应用（如WordPress、GitLab等）
- 资源受限的边缘计算场景
