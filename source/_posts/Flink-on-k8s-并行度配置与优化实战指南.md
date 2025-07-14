---
title: Flink on k8s：并行度配置与优化实战指南
tags:
  - Flink
  - K8s
  - Kubernetes
  - 并行度
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250715032618358.png'
toc: true
abbrlink: 8fb5dd5f
date: 2025-07-15 03:23:02
---

Apache Flink 在 Kubernetes (K8s) 上部署时，并行度是影响作业性能和资源利用率的关键因素。本文将深入探讨 Flink on K8s 环境中并行度相关的配置项，并通过测试案例总结其作用机制，最后给出实际应用中的优化建议。

<!-- more -->

## Flink 作业并行度相关配置项

在 Flink on K8s 中，有几个核心参数控制着作业的并行度：

1.  [cite_start]**`taskManager.replicas`**： 此参数定义了 TaskManager 副本的数量。如果定义了此值，它将优先于 `parallelism` 参数设置的并行度 [cite: 202]。

2.  [cite_start]**`taskmanager.numberOfTaskSlots`**： 该参数表示单个 TaskManager 可运行的并行操作符或用户函数实例的数量 [cite: 203][cite_start]。当此值大于 1 时，一个 TaskManager 可以运行多个函数或操作符实例，从而利用多个 CPU 核心。通常，此值与 TaskManager 机器的物理 CPU 核心数成比例（例如，等于核心数或核心数的一半） [cite: 203]。

3.  [cite_start]**`job.parallelism`**： 这是 Flink 作业的并行度，定义了整个作业的默认并行执行数量 [cite: 204]。

4.  [cite_start]**`args.parallelism`**： 这是在程序代码中为特定算子（Operator）设置的并行度 [cite: 205]。

5.  **并行度、Slot、SlotSharingGroup**：
    * [cite_start]**Slot**：是 TaskManager 上的一个执行资源单位，可以执行一个任务或一个管道 [cite: 390]。
    * **SlotSharingGroup**：允许将属于同一 SlotSharingGroup 的任务链中的多个算子共享同一个 TaskSlot。
        * [cite_start]**归属于同一个 SlotSharingGroup**： 如果 Task Chain 中的所有算子都属于同一个 `SlotSharingGroup`，则作业所需的 Slot 数量等于该 Task Chain 中并行度最大的算子的并行度。例如，在一个包含 Source（并行度 10）、Map（并行度 10）和 Sink（并行度 5）的 `SlotShareGroup-1` 中，作业并行度为 `Max(10, 5) = 10`，所需 Slot 数量为 10 [cite: 207]。
        * [cite_start]**归属于不同的 SlotSharingGroup**： 如果不同的 Task Chain 属于不同的 `SlotSharingGroup`，则作业所需的总 Slot 数量是每个 `SlotSharingGroup` 所需 Slot 数量的总和。例如，如果 `SlotShareGroup-1` 需要 10 个 Slot，`SlotShareGroup-2` 需要 10 个 Slot，那么总共需要 $10 + 10 = 20$ 个 Slot [cite: 208]。

## 测试案例与结论

为了更好地理解这些参数的相互作用，我们进行了一系列测试，并总结出以下结论：

| 序号 | `taskManager.replicas` | `taskmanager.numberOfTaskSlots` | `job.parallelism` | `args.parallelism` | 启动的 TaskManager | 总 Slot / 消耗 Slot | 任务并行度 / 任务链数           |
| :--- | :--------------------- | :------------------------------ | :---------------- | :----------------- | :----------------- | :------------------ | :------------------------------- |
| 测试1 | 1                      | 6                               | 10                | 0                  | 2                  | 12 / 12             | 6 / 2                            |
| 测试2 | 1                      | 11                              | 10                | 0                  | 2                  | 22 / 22             | 11 / 2                           |
| 测试3 | 4                      | 1                               | 1                 | 0                  | 8                  | 8 / 8               | 4 / 2                            |
| 测试4 | 4                      | 1                               | 1                 | 5                  | 10                 | 10 / 10             | Max(5,4) / 2                     |
| 测试5 | 4                      | 3                               | 2                 | 5                  | 8                  | 24 / 24             | Max(5, 4*3=12) / 2               |
| 测试6 | 1                      | 5                               | 1                 | 10                 | 4                  | 20 / 20             | Max(10,5) / 2                    |

[cite_start]*Source: [cite: 381]*

**测试结论：**

* [cite_start]**`job.parallelism` 的作用**： 实际上，`job.parallelism` 参数在 Flink on K8s 的 Application 模式下，对于作业的实际并行度影响不大，可以忽略 [cite: 383]。
* [cite_start]**`taskManager.replicas` 的默认值**： 如果在作业 YAML 文件中没有明确设置 `taskManager.replicas`，其值可以理解为 1 [cite: 384]。
* **无 `args.parallelism` 设置的情况**：
    * [cite_start]当 `taskManager.replicas` 未设置或为 1 时，并且作业没有在程序层面设置算子并行度 (`args.parallelism` 为 0)，那么整个程序的并行度将由 `taskmanager.numberOfTaskSlots` 决定 [cite: 385]。
    * [cite_start]当 `taskManager.replicas` 大于 1 时，且作业没有在程序层面设置算子并行度 (`args.parallelism` 为 0)，那么程序的整体并行度将是 `taskManager.replicas * taskmanager.numberOfTaskSlots` [cite: 386]。
* **设置了 `args.parallelism` 的情况**：
    * 如果 `taskManager.replicas` 大于 1，并且作业在程序层面对某些算子设置了并行度 (`args.parallelism`)，那么这些算子的并行度将是 `args.parallelism`。
    * [cite_start]其他没有设置并行度的算子，它们的并行度将是 `taskManager.replicas * taskmanager.numberOfTaskSlots` [cite: 387]。

**三、少 TaskManager 多 Slot vs. 多 TaskManager 少 Slot**

在 Flink 官方文档中，对于 `taskmanager.numberOfTaskSlots` 参数的解释提到：

* [cite_start]运行更多小型的 TaskManager（每个 TaskManager 只有一个 Slot）是实现任务之间最佳隔离的良好起点 [cite: 390]。
* [cite_start]将相同的资源分配给更少、更大、拥有更多 Slot 的 TaskManager 有助于提高资源利用率，但代价是任务之间的隔离性较弱（更多任务共享同一个 JVM） [cite: 390]。

**其他考量因素：**

* [cite_start]**过多 Slot**： 可能导致频繁的上下文切换，从而影响性能。但它可以分摊 JVM、应用程序库或网络连接的固定开销 [cite: 392]。
* [cite_start]**过多 TaskManager**： 彼此之间需要通过网络交换信息，这也会影响性能 [cite: 393]。

## 优化建议

在实际应用中，我们给出以下建议：

* [cite_start]**遵循单一职责小作业原则**： 建议作业的并行度不宜过高，所需 Slot 数量相对较少（建议在 16 个以内）[cite: 395]。
* [cite_start]**控制 TaskManager 数量**： 为了避免 TaskManager 数量过多导致 K8s 异常，建议始终将 `taskManager.replicas` 设置为 1 [cite: 397]。
* [cite_start]**根据需求调整 `taskmanager.numberOfTaskSlots`**： 根据作业实际所需的 Slot 数量来调整 `taskmanager.numberOfTaskSlots`。例如，如果作业需要 100 个 Slot，可以考虑将 `taskmanager.numberOfTaskSlots` 设置为 10 或 20，从而将 TaskManager 数量控制在 10 个以内 [cite: 397]。
* [cite_start]**Slot 数量与 CPU 核数匹配**： 尽管 Slot 数量与 CPU 核数没有严格的对应关系，但建议 `taskmanager.numberOfTaskSlots` 的数量与 TaskManager 的 CPU 数量保持一致，或者保持 2:1 的比例 [cite: 398]。
