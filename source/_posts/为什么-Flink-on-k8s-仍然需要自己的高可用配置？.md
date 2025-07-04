---
title: 为什么 Flink on k8s 仍然需要自己的高可用配置？
tags:
  - K8s
  - Flink
  - 大数据
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250614063036471.png'
toc: true
abbrlink: 61a88c8b
date: 2025-06-14 06:28:36
---

当我们将有状态的流处理应用 Flink 部署到强大的容器编排平台 Kubernetes (K8s) 上时，一个常见的疑问便浮出水面：K8s 自身已经具备了强大的故障恢复和自愈能力，为什么我们还需要为 Flink 单独配置高可用（HA）呢？本文将深入剖析 K8s HA 与 Flink HA 的关系，阐明它们在保障应用稳定运行中各自扮演的角色，并解释为何两者是相辅相成、缺一不可的“双层保险”。

<!-- more -->

---

## 引言：一个普遍的困惑

想象一下这个场景：你已经成功地将 Flink 作业打包成镜像，并使用 Flink Operator 或 Helm Charts 将其部署到了 K8s 集群中。你深知 K8s 的强大之处——当一个节点宕机，K8s 会自动将运行其上的 Pod 调度到其他健康节点上；当一个 Pod 因故崩溃，Deployment 会立即创建一个新的 Pod 来替代它。

这一切听起来都像是完美的高可用方案。于是，当你看到 Flink 的 `FlinkDeployment` YAML 中出现如下配置时，你可能会感到困惑：

```yaml
# ...
spec:
  # ...
  flinkConfiguration:
    high-availability.type: kubernetes
    high-availability: org.apache.flink.kubernetes.highavailability.KubernetesHaServicesFactory
    high-availability.storageDir: file:///opt/flink/flink_recovery 
    state.checkpoints.dir: file:///opt/flink/checkpoints
# ...
```

既然 K8s 都能重启我的 Flink 进程了，为什么还要多此一举配置 Flink 自己的 HA 呢？

答案的核心在于：**Kubernetes 的高可用和 Flink 的高可用，解决的是不同层面的问题。** K8s 负责基础设施层的“进程存活”，而 Flink HA 负责应用层的“状态与记忆恢复”。

## 第一层保险：Kubernetes 的承诺（基础设施层 HA）

首先，让我们肯定 K8s 所扮演的重要角色。作为 Flink 集群的运行底座，K8s 提供了至关重要的基础设施级别的高可用性：

1.  **Pod 自动治愈**：如果 Flink 的 JobManager 或 TaskManager Pod 因为程序 Bug、OOMKilled 等内部原因意外退出，K8s 的控制器（如 Deployment、StatefulSet 或 Flink Operator 管理的资源）会立刻侦测到，并迅速启动一个新的 Pod 实例来维持期望的副本数。

2.  **节点故障转移**：如果某个 K8s 工作节点（Node）因为硬件故障或网络问题而宕机，K8s 会将该节点上的所有 Pods 重新调度到集群中其他健康的节点上。

K8s 的这些能力保证了 Flink 集群的**组件进程是健壮的**。它就像一个尽职尽责的急救团队，无论你的程序进程（Pod）因为什么原因“心跳停止”，它都会第一时间赶到并进行“心肺复苏”，让进程重新运行起来。

## “失忆”的悲剧：K8s 高可用的局限性

既然急救团队如此给力，问题又出在哪里呢？问题在于，K8s 在“救活”一个 Pod 时，它只是简单粗暴地**启动了一个全新的、空白的进程**。 K8s 对 Pod 内部运行的应用程序的状态一无所知，它眼里的 Pod 只是一个需要保持运行的黑盒。

让我们设想一下，在**没有配置 Flink HA** 的情况下，JobManager Pod 发生故障后的连锁反应：

1.  **故障发生**：JobManager Pod 因节点宕机而消失。
2.  **K8s 介入**：K8s 发现 JobManager Pod 不在了，于是在一个新节点上启动了一个全新的 JobManager Pod。
3.  **灾难降临**：这个新启动的 JobManager 是一个**“失忆”的 JobManager**。它完全不知道：
    *   之前集群正在运行哪个 Flink 作业？
    *   这个作业的执行图（JobGraph）是什么样的？
    *   作业已经完成了哪些 Checkpoint？最新的 Checkpoint 元数据在哪里？
    *   还有哪些 TaskManager 存活并等待它的指令？

**最终结果是：整个 Flink 作业宣告失败。** 虽然 JobManager *进程*被 K8s 恢复了，但 Flink 作业的“大脑”已经丢失了所有关于作业状态和进度的记忆。你唯一的选择就是手动重新提交作业，并且所有从上一个 Checkpoint 到故障发生时刻之间的计算进度都将付之一炬。

## 第二层保险：Flink 的自我救赎（应用层 HA）

这就是 Flink 自身 HA 机制登场的时刻。它的核心使命就是解决 JobManager 的“失忆”问题，确保在主 JobManager 发生故障切换后，新的 Leader 能够完整地继承“记忆”，从而恢复整个分布式作业。

回头看我们最初的配置：

```yaml
high-availability.type: kubernetes
high-availability.storageDir: file:///opt/flink/flink_recovery
```

这两行配置告诉 Flink 两件重要的事情：

1.  **元数据持久化**：启用 HA 后，JobManager 会像写日记一样，将所有关键的元数据——例如作业图、已完成的 Checkpoint 的指针和计数器、Leader 选举信息等——持续写入到一个**高可用的外部存储**中。这个存储位置就是由 `high-availability.storageDir` 指定的。

2.  **Leader 选举**：当现有的 JobManager Leader 失联时，备用的 JobManager 实例或新启动的 JobManager 实例会通过 K8s 的原生机制（通常是利用 ConfigMaps 的原子操作）进行 Leader 选举，以确保同一时间只有一个活跃的“大脑”。

现在，让我们再次模拟 JobManager 故障，但这次**配置了 Flink HA**：

1.  **故障发生**：JobManager Pod 挂掉。
2.  **K8s 介入**：K8s 同样会重启一个新的 JobManager Pod。
3.  **恢复记忆**：新的 JobManager 启动后，它的首要任务不是茫然四顾，而是立刻去 `high-availability.storageDir` 指定的位置**读取“工作日志”**。
4.  **恢复作业**：通过读取元数据，新 JobManager 瞬间“恢复记忆”。它知道了要运行哪个作业，找到了最后一个成功 Checkpoint 的位置（Checkpoint 数据本身存储在 `state.checkpoints.dir`），重新连接上所有 TaskManager，并指挥它们从这个 Checkpoint 恢复状态。
5.  **无缝衔接**：作业从中断处继续运行，数据流得以恢复，实现了真正的故障恢复和 Exactly-Once 语义保证。

## 关键前提：持久化存储的重要性

> **注意**：在本文引用的示例配置中，`storageDir` 和 `checkpoints.dir` 都使用了 `file:///` 协议。这表示数据被存储在 Pod 的本地文件系统中，这**仅适用于本地测试**！在生产环境中，当 Pod 被删除重建后，本地数据会随之丢失，这将导致 Flink HA 机制因找不到恢复数据而失效。
>
> **在生产环境中，`high-availability.storageDir` 和 `state.checkpoints.dir` 必须指向一个真正高可用的持久化存储卷（Persistent Volume），例如 NFS、HDFS、S3、GCS 或 CephFS 等。**

## 结论：K8s HA + Flink HA = 真正的弹性

现在，我们可以清晰地总结两者之间的关系了：

| 特性         | Kubernetes HA (基础设施层)                                 | Flink HA (应用层)                               |
| :----------- | :----------------------------------------------------------- | :---------------------------------------------- |
| **关注对象**   | Pod, Container, Node (进程和硬件)                            | Flink Job, 元数据, Checkpoint (应用状态和逻辑)  |
| **解决的问题** | “我的程序进程挂了，请帮我重启一个新的。”                     | “我的 Flink 大脑挂了，请让新大脑恢复记忆，从上次存档点继续。” |
| **核心机制**   | Pod 健康检查, 副本控制器, 调度器                             | Leader 选举, 元数据持久化, 状态恢复             |
| **形象比喻**   | **医院的急救团队**：负责在病人（Pod）心跳停止时进行电击除颤，让其恢复心跳。 | **病人的病历和记忆**：确保被救活的病人不会失忆，知道自己是谁，之前在做什么。 |

最终，K8s 为 Flink 提供了一个坚实的、具备自愈能力的运行环境，这是高可用的**地基**。而 Flink 自身的 HA 机制，则在这个地基之上，构建了针对有状态流计算的、精细化的**业务容错逻辑**。

因此，当你下一次在 K8s 上部署 Flink 时，请记住，启用并正确配置 Flink 的高可用性并非冗余，而是为你宝贵的数据和业务连续性购买的一份至关重要的“双层保险”。