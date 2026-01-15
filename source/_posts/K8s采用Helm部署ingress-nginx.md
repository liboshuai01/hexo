---
title: K8s采用Helm部署ingress-nginx
abbrlink: ad80c258
date: '2025-06-11T18:26:25.000Z'
tags:
  - 容器化
categories:
  - 容器化
toc: true
---


在Kubernetes（K8s）生态中，`Ingress`是管理集群外部访问HTTP/S路由的核心资源。而`ingress-nginx`作为官方维护的实现，凭借其高性能和稳定性，成为了事实上的标准。本文将介绍一种生产级的、可重复的部署方式——使用Helm结合脚本，将`ingress-nginx`以`DaemonSet`的形式部署在指定节点上，并利用`hostNetwork`模式实现高效的网络通信。

<!-- more -->

> 项目源码: [github](https://github.com/liboshuai01/k8s-stack/tree/master/ingress-nginx), [gitee](https://gitee.com/liboshuai01/k8s-stack/tree/master/ingress-nginx)

## Ingress-Nginx 部署模式简介

在开始之前，我们先简单回顾一下 Ingress-Nginx 常见的几种部署模式：

1.  **Deployment + LoadBalancer Service:**
    *   原理: Ingress Controller Pods 由 Deployment 管理。创建一个 `type: LoadBalancer` 的 Service 指向这些 Pods。云厂商会自动创建并关联一个外部负载均衡器及公网 IP。
    *   优点: 易于与公有云集成，自动获取公网 IP 和负载均衡。
    *   缺点: 依赖云厂商支持，可能产生额外费用，网络路径相对较长。
    *   适用场景: 公有云环境。

2.  **Deployment + NodePort Service:**
    *   原理: Ingress Controller Pods 由 Deployment 管理。创建一个 `type: NodePort` 的 Service 指向这些 Pods。Ingress Controller 会暴露在集群每个节点的一个静态高位端口上。
    *   优点: 不依赖特定云厂商，部署相对简单。
    *   缺点: NodePort 端口通常在高位范围 (30000-32767)，需要外部负载均衡器将 80/443 端口的流量转发到节点的 NodePort。增加了一层转发。
    *   适用场景: 自建机房或需要手动控制负载均衡器的环境。

3.  **DaemonSet + HostNetwork:**
    *   原理: Ingress Controller Pods 由 DaemonSet 管理，确保在指定的每个节点上都运行一个 Pod。Pod 配置 `hostNetwork: true`，直接使用宿主机的网络命名空间，监听宿主机的 80/443 端口。
    *   优点: 网络路径最短，性能通常最优。
    *   缺点: Pod 直接占用宿主机端口，可能冲突。每个节点只能运行一个监听相同端口的 Ingress Controller Pod。需通过 `nodeSelector` 或 `affinity` 精确控制部署节点。
    *   适用场景: 对性能要求高、网络延迟敏感的生产环境，且有专用节点承载 Ingress 流量。

**本文重点实践 `DaemonSet + HostNetwork` 模式，并通过脚本进行部署。** 这种模式下，Ingress Controller Pod 直接监听宿主机节点的物理网络端口（如80和443），流量直接到达，无需额外的 Service 层转发，从而获得最佳性能和最低延迟。

## 核心思路

我们的目标是构建一个健壮、高可用的入口流量解决方案。核心思路如下：

1.  **配置化与自动化**：将命名空间、版本等易变配置抽离到`.env`文件中，安装、卸载过程由Shell脚本执行，实现一键部署与清理。
2.  **高可用部署**：采用`DaemonSet`模式，确保`ingress-nginx-controller`在所有打了特定标签的节点上都运行一个实例。这天然地实现了负载均衡和高可用，任何一个节点宕机，其他节点依然可以处理流量。
3.  **高性能网络**：启用`hostNetwork=true`，让Pod直接使用宿主机的网络命名空间。这避免了通过K8s Service（如`NodePort`或`LoadBalancer`）进行端口转发所带来的额外网络开销和NAT性能损耗，实现了流量的直接、高效路由。

## 部署实践

下面，我们将通过具体的项目文件和步骤，来完成`ingress-nginx`的部署。

### 前提准备

在开始之前，请确保您的K8s集群满足以下条件。这些是确保`ingress-nginx`成功运行的关键前提。

> ```markdown
> 前提准备
> ---
>
> 1. 确保`k8s`集群中的各节点中的`80/443`端口均没有被占用。
> 2. 如果应该存在了其他`ingress controller`请先卸载，下面为卸载`traefik`的示例命令。
>     ```shell
>     helm uninstall traefik -n kube-system
>     helm uninstall traefik-crd -n kube-system
>     ```
> 3. 修改`.env`文件中配置的变量为自定义内容，如安装的命名空间、helm实例名称、char版本号等（可选）。
> ```

**专家解读**：

*   **端口独占**：由于我们后续会采用`hostNetwork`模式，`ingress-nginx`的Pod会直接监听宿主机节点的`80`和`443`端口。因此，必须保证这些端口未被主机上的其他任何进程（例如一个裸机部署的Nginx）占用。
*   **避免冲突**：集群中同时运行多个`Ingress Controller`会导致`Ingress`资源的归属权混乱和严重的端口冲突，务必保证单一控制器的原则。

### 项目文件概览

我们的部署方案主要由两个文件构成：一个用于存放配置变量，一个用于执行安装逻辑。

#### 1. 配置文件: `.env`

该文件集中管理了部署所需的所有可变参数，便于维护和修改。

> ```shell
> # 命名空间
> NAMESPACE="ingress-nginx"
> # helm的release名称
> RELEASE_NAME="ingress-nginx"
> # helm的chart版本
> CHART_VERSION="4.12.2"
> ```

#### 2. 安装脚本: `install.sh`

此脚本负责执行完整的Helm安装或升级流程。

> ```shell
> #!/usr/bin/env bash
> 
> set -e
> 
> # --- 加载变量 ---
> if [ -f .env ]; then
>     source .env
> else
>     echo "错误: .env 文件不存在!"
>     exit 1
> fi
> 
> # --- 添加仓库并更新 ---
> helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
> helm repo update
> 
> # --- 安装 / 升级 ---
> helm upgrade --install ${RELEASE_NAME} ingress-nginx/ingress-nginx \
>   --version ${CHART_VERSION} --namespace ${NAMESPACE} --create-namespace \
>   \
>   --set controller.hostNetwork=true \
>   --set controller.dnsPolicy=ClusterFirstWithHostNet \
>   --set-string controller.nodeSelector.ingress="true" \
>   --set controller.kind=DaemonSet \
>   --set controller.service.enabled=true \
>   --set controller.service.type=NodePort
> ```

**专家解读`helm upgrade`关键参数**：

*   `--install`：如果Release不存在，则执行安装；如果已存在，则执行升级。这让脚本可以重复执行，具备幂等性。
*   `--create-namespace`：如果指定的命名空间不存在，会自动创建。
*   `controller.hostNetwork=true`：**核心配置**。让Controller Pod使用宿主机的网络，性能最高。
*   `controller.dnsPolicy=ClusterFirstWithHostNet`：当启用`hostNetwork`时，必须为Pod指定DNS策略，`ClusterFirstWithHostNet`是标准选择。
*   `controller.nodeSelector.ingress="true"`：**调度策略**。指定Controller Pod只能被调度到包含`ingress=true`标签的节点上。这给予了我们精确控制入口流量节点的权力。
*   `controller.kind=DaemonSet`：**部署模式**。确保每个满足`nodeSelector`的节点上都且仅都运行一个Controller Pod实例。
*   `controller.service.type=NodePort`：虽然我们使用了`hostNetwork`，外部流量直接访问节点IP即可。但保留一个`Service`对象（即使是`NodePort`类型）对于Kubernetes内部的服务发现和某些监控工具的集成可能仍然有益，它不会影响`hostNetwork`的流量路径。

### 安装与验证

现在，让我们按照操作手册执行部署和验证。

#### 安装应用

> ```markdown
> 安装应用
> ---
> 
> **1. 执行安装脚本**
> 
> ```shell
> bash install.sh
> ```
>
> **2. 给对应节点打调度标签**
>
> 执行完`install.sh`脚本后，请手动执行下面的命令选择指定节点让`ingress controller`调度到对应的节点。
> ```shell
> kubectl label node [节点名称] ingress=true --overwrite
> 
> # 例如
> # kubectl label node k8s-node-1 ingress=true --overwrite
> ```
> ```

**操作说明**：先运行安装脚本创建`DaemonSet`等资源，此时因为没有节点满足`nodeSelector`，Pod会处于`Pending`状态。接着，为期望作为Ingress流量入口的节点（通常是Master或专用的边缘节点）打上标签，K8s调度器会自动将Pod调度到这些节点上并启动。

#### 验证应用

部署完成后，我们需要进行验证以确保一切工作正常。

> ```markdown
> 验证应用
> ---
> 
> ### 初步验证
> 
> ```shell
> bash status.sh
> ```
>
> ### 进阶验证
>
> **1. 进一步验证，可以创建一个测试应用，并使用`ingress`访问**
>
> ```shell
> helm upgrade --install nginx-test-app bitnami/nginx \
>   --version 20.0.3 --namespace default \
>   \
>   --set service.type=ClusterIP \
>   \
>   --set ingress.enabled=true \
>   --set ingress.ingressClassName=nginx \
>   --set ingress.hostname="nginx.lbs.com" \
>   --set ingress.path="/" \
>   \
>   --set resources.requests.cpu=100m \
>   --set resources.requests.memory=128Mi \
>   --set resources.limits.cpu=250m \
>   --set resources.limits.memory=512Mi
> ```
>
> **2. 配置`hosts`文件，添加一下内容**
>
> ```
> [任意ingress-nginx节点IP]    nginx.lbs.com
> 
> # 例如
> # 192.168.6.202    nginx.lbs.com
> ```
>
> **3. 访问`nginx.lbs.com`，如果访问成功，则说明安装成功**
>
> **4. 测试成功后，删除测试应用**
>
> ```shell
> helm uninstall nginx-test-app -n default
> ```
> ```

**验证流程解析**：
这个进阶验证是一个完整的端到端测试。它创建了一个Nginx测试应用，并为其配置了一个`Ingress`规则。该规则声明 `nginx.lbs.com` 的流量应由名为 `nginx` 的`IngressClass`（即我们刚刚部署的`ingress-nginx`）来处理。

通过修改本地`hosts`文件，我们模拟了DNS解析，将域名指向任意一个运行着`ingress-nginx` Pod的节点IP。当您在浏览器访问该域名时，请求会直接发送到该节点的80端口，被`ingress-nginx`接收，然后根据`Ingress`规则，将请求准确地转发到后端的`nginx-test-app`服务。访问成功，即证明整个流量链路畅通无阻。

### 应用生命周期管理

最后，我们也提供了标准的更新与卸载流程。

> ```markdown
> 更新应用
> ---
> 
> 修改`.env`或`install.sh`文件中的内容，后重新执行`install.sh`脚本即可。
> 
> 卸载应用
> ---
> 
> **1. 执行卸载脚本**
> 
> ```shell
> bash uninstall.sh
> ```
>
> **2. （可选）将`ingress`标签从节点上删除**
>
> ```shell
> kubectl label node [节点名称] ingress-
> 
> # 例如
> kubectl label node k8s-node-1 ingress-
> ```
> ```

## 总结

通过本文介绍的方案，我们利用Helm、Shell脚本和精心选择的部署策略（`DaemonSet` + `hostNetwork`），实现了一套专业、高效且易于管理的`ingress-nginx`部署方案。这种方法不仅保证了入口流量层的高性能和高可用性，还通过脚本化和配置化大大提升了运维效率和部署的可靠性，是现代云原生后端系统架构中的优秀实践。