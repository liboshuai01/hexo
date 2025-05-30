---
title: K8s采用Helm部署kube-prometheus-stack实战指南
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
abbrlink: 9958a6cd
date: 2025-05-24 18:01:30
---

Kubernetes (K8s) 已成为容器编排的事实标准，而监控则是确保其上运行的应用和服务稳定可靠的关键。`kube-prometheus-stack` 是一个非常受欢迎的 Helm Chart，它打包了 Prometheus Operator、Prometheus、Grafana、Alertmanager 以及众多 Exporter，为 Kubernetes 集群提供了一套全面的监控和告警解决方案。

本文将指导您如何使用 Helm，通过预配置的脚本在 Kubernetes 集群中快速部署 `kube-prometheus-stack`。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/k8s-stack/tree/master/kube-prometheus-stack), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/kube-prometheus-stack)

## 前提准备

在开始之前，请确保您已满足以下条件：

1.  **拥有一个 Kubernetes 集群**：并且 `kubectl` 已配置好，可以与集群通信。
2.  **安装了 Helm v3**：Helm 是 Kubernetes 的包管理器，我们将用它来部署 `kube-prometheus-stack`。
3.  **配置了 Ingress Controller**：例如 Nginx Ingress Controller，因为我们将通过 Ingress 暴露 Prometheus、Grafana 和 Alertmanager 服务。脚本中会使用 `INGRESS_CLASS_NAME` 指定 Ingress 类。
4.  **配置了 StorageClass**：Prometheus、Grafana 和 Alertmanager 都需要持久化存储。脚本中会使用 `STORAGE_CLASS_NAME` 指定存储类，例如 `nfs`、`cephfs` 或云提供商的块存储。

## 1. 配置环境变量

首先，我们需要定义一些环境变量，这些变量将用于自定义 `kube-prometheus-stack` 的安装。创建一个名为 `.env` 的文件，并填入以下内容。您可以根据您的实际环境修改这些值。

```env
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

**重要提示**：请务必将 `GRAFANA_ADMIN_PASSWORD` 的值 `"YOUR_PASSWORD"` 替换为您自己的强密码。

## 2. 编写安装脚本

接下来，我们创建一个安装脚本 `install.sh`。该脚本会加载 `.env` 文件中的变量，添加 Prometheus Helm 仓库，并使用 `helm upgrade --install` 命令来部署或更新 `kube-prometheus-stack`。

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

**脚本内容解析：**

1.  **`#!/usr/bin/env bash`**: Shebang，指定脚本使用 bash解释器执行。
2.  **加载变量**:
    ```shell
    if [ -f .env ]; then
        export $(grep -v '^#' .env | sed 's/\r$//' | xargs)
    else
        echo "错误: .env 文件不存在!"
        exit 1
    fi
    ```
    *   这部分代码检查当前目录下是否存在 `.env` 文件。
    *   如果存在，`grep -v '^#' .env` 会读取文件内容，并排除以 `#` 开头的注释行。
    *   `sed 's/\r$//'` 用于移除 Windows 风格的换行符 `\r` (如果存在)，确保跨平台兼容性。
    *   `xargs` 将处理后的每行内容（`KEY=VALUE` 格式）作为参数传递给 `export` 命令，从而将这些键值对设置为环境变量。
    *   如果 `.env` 文件不存在，则打印错误信息并退出脚本。
3.  **添加仓库并更新**:
    ```shell
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    ```
    *   `helm repo add prometheus-community ...`: 添加 `prometheus-community` 的 Helm Chart 仓库。这是 `kube-prometheus-stack` Chart 的官方来源。
    *   `helm repo update`: 更新本地 Helm 仓库列表，确保能获取到最新的 Chart 信息。
4.  **安装 / 升级**:
    ```shell
    helm upgrade --install ${RELEASE_NAME} prometheus-community/kube-prometheus-stack \
      --version ${CHART_VERSION} --namespace ${NAMESPACE} --create-namespace \
      ... (省略了 --set 参数)
    ```
    *   `helm upgrade --install`: 这是 Helm 的核心命令。
        *   `upgrade`: 如果指定的 `RELEASE_NAME` 已经存在，则升级该 Release。
        *   `--install`: 如果指定的 `RELEASE_NAME` 不存在，则安装它。这个组合使得脚本具有幂等性，可以反复执行。
    *   `${RELEASE_NAME}`: Helm Release 的名称，从 `.env` 文件中加载。
    *   `prometheus-community/kube-prometheus-stack`: 要安装的 Chart 名称，格式为 `<repository_name>/<chart_name>`。
    *   `--version ${CHART_VERSION}`: 指定要安装的 Chart 版本，从 `.env` 文件中加载。
    *   `--namespace ${NAMESPACE}`: 指定 Release 安装到的 Kubernetes 命名空间，从 `.env` 文件中加载。
    *   `--create-namespace`: 如果指定的命名空间不存在，则自动创建它。
5.  **`--set` 参数**:
    这些参数用于覆盖 Chart 中的默认 `values.yaml` 配置。
    *   **Alertmanager 配置**:
        *   `--set alertmanager.enabled=true`: 启用 Alertmanager 组件。
        *   `--set alertmanager.ingress.enabled=true`: 为 Alertmanager 启用 Ingress 资源，使其能通过域名访问。
        *   `--set alertmanager.ingress.ingressClassName=${INGRESS_CLASS_NAME}`: 指定 Ingress Controller 的类名。
        *   `--set alertmanager.ingress.hosts[0]=${ALERTMANAGER_HOST}`: 设置 Alertmanager 的访问域名。
        *   `--set alertmanager.ingress.paths[0]="/"`: 设置 Alertmanager Ingress 的访问路径。
        *   `--set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=${STORAGE_CLASS_NAME}`: 指定 Alertmanager 持久化存储使用的 StorageClass。
        *   `--set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.accessModes[0]="ReadWriteOnce"`: 设置存储卷的访问模式。
        *   `--set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=8Gi`: 设置 Alertmanager 的存储请求大小。
    *   **Prometheus 配置**:
        *   `--set prometheus.enabled=true`: 启用 Prometheus 组件。
        *   `--set prometheus.ingress.enabled=true`: 为 Prometheus 启用 Ingress。
        *   `--set prometheus.ingress.ingressClassName=${INGRESS_CLASS_NAME}`: 指定 Ingress Controller 的类名。
        *   `--set prometheus.ingress.hosts[0]=${PROMETHEUS_HOST}`: 设置 Prometheus 的访问域名。
        *   `--set prometheus.ingress.paths[0]="/"`: 设置 Prometheus Ingress 的访问路径。
        *   `--set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=${STORAGE_CLASS_NAME}`: 指定 Prometheus 持久化存储使用的 StorageClass。
        *   `--set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.accessModes[0]="ReadWriteOnce"`: 设置存储卷的访问模式。
        *   `--set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=32Gi`: 设置 Prometheus 的存储请求大小。
    *   **Grafana 配置**:
        *   `--set grafana.enabled=true`: 启用 Grafana 组件。
        *   `--set grafana.adminPassword=${GRAFANA_ADMIN_PASSWORD}`: 设置 Grafana 的 admin 用户密码。
        *   `--set grafana.ingress.enabled=true`: 为 Grafana 启用 Ingress。
        *   `--set grafana.ingress.ingressClassName=${INGRESS_CLASS_NAME}`: 指定 Ingress Controller 的类名。
        *   `--set grafana.ingress.hosts[0]=${GRAFANA_HOST}`: 设置 Grafana 的访问域名。
        *   `--set grafana.ingress.path="/"`: 设置 Grafana Ingress 的访问路径。
        *   `--set grafana.persistence.enabled=true`: 启用 Grafana 的持久化存储。
        *   `--set grafana.persistence.storageClassName=${STORAGE_CLASS_NAME}`: 指定 Grafana 持久化存储使用的 StorageClass。
        *   `--set grafana.persistence.accessModes[0]="ReadWriteOnce"`: 设置存储卷的访问模式。
        *   `--set grafana.persistence.size=8Gi`: 设置 Grafana 的存储请求大小。
    *   **Prometheus Operator 配置**:
        *   `--set prometheusOperator.enabled=true`: 启用 Prometheus Operator 组件。Prometheus Operator 负责管理 Prometheus 实例、ServiceMonitors、PodMonitors 等自定义资源。

这个脚本通过 `--set` 参数详细配置了 Alertmanager、Prometheus 和 Grafana 的持久化存储、Ingress 规则以及 Grafana 的管理员密码，确保了监控栈的关键组件都能按需部署和访问。

## 3. 执行安装

确保 `.env` 和 `install.sh` 文件在同一目录下，并赋予 `install.sh` 执行权限：

```bash
chmod +x install.sh
```

然后执行安装脚本：

```bash
bash install.sh
```

Helm 命令会创建指定的命名空间（如果不存在），并开始部署所有相关的 Kubernetes 资源。这个过程可能需要几分钟。

## 4. 配置 Hosts 解析 (或 DNS)

为了能够通过域名访问 Prometheus、Grafana 和 Alertmanager，您需要在能够访问 Kubernetes Ingress Controller 的机器上配置 `/etc/hosts` 文件 (或者在您的 DNS 服务器上添加相应的 A 记录)。

获取您的 Ingress Controller 服务的外部 IP 地址（通常是 LoadBalancer 类型 Service 的 EXTERNAL-IP，或者 NodePort 模式下任一集群节点的 IP）。

```bash
kubectl get svc -n <your-ingress-controller-namespace>
```

假设您的 Ingress Controller 节点 IP 是 `192.168.6.202`，并且您在 `.env` 中定义的域名是 `prometheus.lbs.com`, `grafana.lbs.com`, `alertmanager.lbs.com`。编辑您的本地 `hosts` 文件 (Linux/macOS: `/etc/hosts`, Windows: `C:\Windows\System32\drivers\etc\hosts`)，添加如下内容：

```
192.168.6.202 prometheus.lbs.com grafana.lbs.com alertmanager.lbs.com
```
请将 `192.168.6.202` 替换为您实际的 Ingress 节点 IP，并将域名替换为您在 `.env` 文件中定义的域名。

## 5. 初步验证

为了快速检查部署状态，我们可以创建一个 `status.sh` 脚本来查看相关 Pod、Service 和 PVC 的状态。

创建 `status.sh` 文件：
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
赋予执行权限并运行：
```bash
chmod +x status.sh
bash status.sh
```
您应该能看到所有相关的 Pod 都处于 `Running` 状态，并且 PVCs 已经成功绑定。

## 6. 进阶验证

现在，您可以通过浏览器访问之前配置的域名来验证各个组件是否正常工作：

1.  **Prometheus**: 访问 `http://prometheus.lbs.com` (替换为您的 Prometheus 域名)。您应该能看到 Prometheus UI。
2.  **Grafana**: 访问 `http://grafana.lbs.com` (替换为您的 Grafana 域名)。使用用户名 `admin` 和您在 `.env` 文件中设置的 `GRAFANA_ADMIN_PASSWORD` 登录。
3.  **Alertmanager**: 访问 `http://alertmanager.lbs.com` (替换为您的 Alertmanager 域名)。您应该能看到 Alertmanager UI。

如果所有页面都能正常访问，那么恭喜您，`kube-prometheus-stack` 已成功部署！

## 7. 更新应用

如果您需要修改配置（例如，更改存储大小、更新版本号或调整任何 Helm Chart 的值），可以按以下步骤操作：

1.  修改 `.env` 文件中的相关变量。
2.  如果需要调整 `install.sh` 脚本中的 `--set` 参数，也一并修改。
3.  重新执行安装脚本：
    ```bash
    bash install.sh
    ```
    Helm 的 `upgrade --install` 命令是幂等的，它会自动应用更改。

## 8. 卸载应用

如果您需要卸载 `kube-prometheus-stack`，可以创建一个 `uninstall.sh` 脚本。

创建 `uninstall.sh` 文件：
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

赋予执行权限并运行：
```bash
chmod +x uninstall.sh
bash uninstall.sh
```
这将卸载由 Helm Chart 创建的所有 Kubernetes 资源。

**（可选）删除 PVC**

默认情况下，Helm 卸载时不会删除 PersistentVolumeClaims (PVCs)，以防止数据丢失。如果您确定不再需要这些数据，可以手动删除它们：

1.  查看 PVC：
    ```bash
    kubectl get pvc -n monitoring
    ```
    (将 `monitoring` 替换为您在 `.env` 文件中定义的 `NAMESPACE`)

2.  删除 PVC (请将 `[pvc名称]` 替换为实际的 PVC 名称)：
    ```bash
    kubectl delete pvc [pvc名称] -n monitoring
    ```
    例如：
    ```bash
    kubectl delete pvc prometheus-kube-prom-stack-prometheus-db-prometheus-kube-prom-stack-prometheus-0 -n monitoring
    kubectl delete pvc storage-kube-prom-stack-grafana -n monitoring
    kubectl delete pvc alertmanager-kube-prom-stack-alertmanager-db-alertmanager-kube-prom-stack-alertmanager-0 -n monitoring
    ```

## 总结

通过本文提供的脚本和步骤，您可以轻松地在 Kubernetes 集群中部署、管理和维护 `kube-prometheus-stack`。这套监控方案为您的集群和应用提供了强大的可观测性，有助于及时发现和解决问题，保障业务的稳定运行。您可以进一步探索 Grafana 的仪表盘定制、Prometheus 的查询语言 PromQL 以及 Alertmanager 的告警规则配置，以充分发挥其监控能力。

---