---
title: Flink on k8s：并行度资源相关配置最佳实践
abbrlink: 8fb5dd5f
date: '2025-07-15T03:23:02.000Z'
tags:
  - 大数据
categories:
  - 大数据
toc: true
---


## 引言：Flink on K8s并行度配置的重要性

Apache Flink作为业界领先的流处理引擎，在实时数据处理领域占据核心地位，其高性能和低延迟特性使其成为众多企业构建实时数据管道的首选。与此同时，Kubernetes作为云原生容器编排平台的普及，使得将Flink作业部署到K8s环境成为主流趋势。这种结合不仅提供了强大的资源管理和弹性伸缩能力，也带来了新的配置挑战。

在Kubernetes环境下部署Flink时，并行度配置不再是简单的参数调整，它直接关系到作业的性能、资源利用率、集群稳定性以及最终的运营成本。不合理的并行度配置可能导致资源浪费、性能瓶颈，甚至引发Kubernetes集群异常。例如，过多的TaskManager Pods可能给Kubernetes API Server、调度器和Kubelet带来巨大负担，而过少的并行度则可能无法充分利用集群资源，导致处理能力不足。

本文旨在深入剖析Flink在Kubernetes上的并行度配置机制。通过结合实际测试案例和详细数据，文章将揭示不同配置参数间的相互作用及其对作业实际运行行为的影响，并提供一套经过验证的最佳实践。这些实践旨在帮助读者避免常见陷阱，实现高效、稳定且资源优化的生产部署。

## Flink on K8s并行度核心概念解析

在深入探讨Flink on K8s的并行度配置之前，理解Flink中与并行度相关的核心概念至关重要。并行度（Parallelism）在Flink中指的是一个算子（Operator）的并行实例数量。一个具有并行度的算子，其逻辑任务会被分成多个子任务，这些子任务可以在不同的任务槽（Slot）中并行执行，从而提高作业的处理能力。

**核心配置项详解**

Flink on K8s环境中，有几个关键配置项共同决定了作业的并行度行为和资源分配：

### taskManager.replicas

taskManager.replicas是Flink官网关于作业并行度配置的说明中列出的一个配置项 。它控制着Flink集群中TaskManager Pods的数量。在Kubernetes环境中，每个TaskManager Pod通常对应一个或多个物理节点上的计算实例，因此它是Kubernetes层面控制计算资源实例数量的关键参数。 然而，通过测试观察发现，taskManager.replicas并非TaskManager数量的硬性上限。例如，在测试1中，尽管YAML配置中taskManager.replicas设置为1，但实际启动了2个TaskManager。类似地，在测试6中， taskManager.replicas也设置为1，但实际启动了4个TaskManager 。这种现象表明，在Flink Kubernetes Operator的Application模式下，Operator会根据作业的总任务槽需求来动态请求TaskManager Pods。如果作业的实际并行度（特别是受到程序中args.parallelism或SlotSharingGroup配置的影响）所需的总任务槽数量，超过了taskManager.replicas * taskmanager.numberOfTaskSlots所能提供的总任务槽数，Operator为了满足作业的任务槽需求，会弹性地启动额外的TaskManager Pods。这意味着用户不能仅依赖taskManager.replicas来确定最终的TaskManager数量，而需要结合作业的实际并行度需求和taskmanager.numberOfTaskSlots来理解其动态行为。taskManager.replicas更像是一个“初始”或“最小”的TaskManager数量，Flink运行时在K8s上具有一定的弹性扩容能力，这对于K8s集群的资源规划至关重要，因为实际Pod数量可能超出用户的初步预期。

### taskmanager.numberOfTaskSlots

taskmanager.numberOfTaskSlots同样是Flink官网关于作业并行度配置的说明中列出的配置项 。它定义了每个TaskManager Pod内部可用的任务槽（Slot）数量。一个任务槽是TaskManager中一个独立的资源单元，用于执行一个或多个任务链。因此，它是控制单个TaskManager内部并行度的关键参数。

### job.parallelism

job.parallelism是Flink作业的全局默认并行度设置，通常用于指定整个作业的默认并行度 。然而，测试结论明确指出，此配置项“不起作业，可以忽略” 。在所有测试案例中，无论job.parallelism设置为1还是10，其对最终的“任务并行度”都没有直接影响。这与Flink的传统行为（job.parallelism通常是重要的全局控制参数）存在显著差异。在Flink Kubernetes Operator的Application模式下，Operator更直接地管理TaskManager Pods及其任务槽分配，并可能将JobGraph的并行度与Kubernetes资源配置（如taskManager.replicas和taskmanager.numberOfTaskSlots）紧密绑定。job.parallelism可能在这个资源协调和调度逻辑中被忽略或优先级较低，导致其在实际运行时无法生效。对于习惯于传统Flink部署模式的用户来说，这是一个重要的配置陷阱。在Flink on K8s环境中，应将注意力从job.parallelism转移到更底层的TaskManager和任务槽配置，以及程序层面的算子并行度。

### args.parallelism (程序内算子并行度)

args.parallelism指的是在Flink程序代码中，通过setParallelism()方法为特定算子（Operator）设置的并行度 。在本文的测试案例中，Kafka Source的并行度就是通过程序参数args.parallelism动态传入并设置的。 在测试4、测试5和测试6中，当args.parallelism被设置时，最终的“任务并行度”会直接受到其影响，甚至在某些情况下（如测试4的Max(5,4)/2，测试6的Max(10,5)/2）显示为args.parallelism的值或其与集群默认并行度的最大值。测试结论也明确指出“这些算子的并行度为args.parallelism” 。这表明args.parallelism具有最高的优先级，可以覆盖集群层面的默认并行度设置。Flink的并行度解析遵循一个明确的层级结构：算子级别（setParallelism()）>作业级别（job.parallelism，此处被忽略）>集群级别（由taskManager.replicas和taskmanager.numberOfTaskSlots决定）。当算子明确设置了并行度时，它将尝试获取相应数量的任务槽来满足其并行度需求。如果集群提供的总任务槽不足以满足所有算子的并行度需求，Flink会动态请求更多的TaskManager Pods（如前述对taskManager.replicas的分析所示），以尝试满足这个更高的并行度要求。args.parallelism提供了精细化控制作业中特定算子并行度的能力，这对于优化作业瓶颈、匹配上游数据源分区数（如Kafka topic分区）或满足特定性能要求非常有用。但同时，它也可能成为隐式触发TaskManager扩容的因素，导致资源消耗超出预期。因此，在设置时需要综合考虑集群的任务槽承载能力和Kubernetes集群的资源限制。

### SlotSharingGroup

SlotSharingGroup是Flink中用于控制算子是否共享同一个任务槽的机制 。归属同一个SlotSharingGroup的算子可以共享一个任务槽，而归属不同SlotSharingGroup的算子则会强制占用不同的任务槽。在测试程序中，Kafka Source被设置为slotSharingGroup("slotGroup-" + i)，这意味着不同的Kafka Topic Source实例将被分配到不同的任务槽中，而不是共享 。测试程序中明确使用了slotSharingGroup来隔离不同的Kafka Source，这意味着即使在同一个TaskManager内有空闲任务槽，不同SlotSharingGroup的算子也不会共享，从而增加了总的任务槽消耗。SlotSharingGroup允许开发者在逻辑上将算子分组，从而影响物理任务槽的分配。通过将不同Source设置到不同的SlotSharingGroup，可以确保每个Source实例都拥有独立的任务槽，避免资源争抢和相互影响，尤其是在处理多主题或多数据源的场景下。这种隔离性虽然会增加总的任务槽消耗，但可以提高作业的稳定性和可预测性，特别是在处理高并发或有严格SLA要求的场景。因此，SlotSharingGroup是优化资源利用和任务隔离的重要工具。合理使用它可以避免不必要的任务槽共享导致性能下降，或者强制任务槽隔离以提高稳定性。

## 并行度配置测试案例与结果分析

本次测试基于Flink 1.13.6版本，部署在Kubernetes集群上，采用Flink Kubernetes Operator的Application模式。测试程序ParallelismTest是一个典型的Flink流处理作业，它从多个Kafka Topic（topic-1, topic-2）消费数据，进行WordCount处理，并将结果写入MySQL 。程序中Kafka Source通过args.parallelism动态设置并行度，并且为每个Kafka Source实例设置了独立的slotSharingGroup，以确保它们不共享任务槽 。 MyKafkaProducer用于向Kafka Topic生成测试数据 。 为了更直观地展示不同配置组合下的实际运行效果，以下表格汇总了所有测试案例的配置参数和实际运行结果：

### Flink on K8s并行度配置测试结果汇总

| 序号  | `taskManager.replicas` | `taskmanager.numberOfTaskSlots` | `job.parallelism` | `args.parallelism` | 启动的 TaskManager | 总 Slot / 消耗 Slot | 任务并行度 / 任务链数       |
|:----|:-----------------------|:--------------------------------|:------------------|:-------------------|:----------------|:-----------------|:-------------------|
| 测试1 | 1                      | 6                               | 10                | 0                  | 2               | 12 / 12          | 6 / 2              |
| 测试2 | 1                      | 11                              | 10                | 0                  | 2               | 22 / 22          | 11 / 2             |
| 测试3 | 4                      | 1                               | 1                 | 0                  | 8               | 8 / 8            | 4 / 2              |
| 测试4 | 4                      | 1                               | 1                 | 5                  | 10              | 10 / 10          | Max(5,4) / 2       |
| 测试5 | 4                      | 3                               | 2                 | 5                  | 8               | 24 / 24          | Max(5, 4*3=12) / 2 |
| 测试6 | 1                      | 5                               | 1                 | 10                 | 4               | 20 / 20          | Max(10,5) / 2      |

### 逐一案例分析

以下对每个测试案例进行简要分析，重点突出其对并行度配置结论的验证作用。

#### 测试1

```
taskManager.replicas=1, taskmanager.numberOfTaskSlots=6, job.parallelism=10, args.parallelism=0
```

分析： 尽管YAML中配置taskManager.replicas为1，但为了满足作业需求（两个Kafka Source实例因SlotSharingGroup而需要独立任务槽），实际启动了2个TaskManager 。 job.parallelism=10被忽略，对实际并行度没有影响 。实际任务并行度为6，总任务槽数12个全部被消耗 。这表明在没有程序内算子并行度设置时，taskmanager.numberOfTaskSlots是决定整体并行度的关键因素。

#### 测试2

```
taskManager.replicas=1, taskmanager.numberOfTaskSlots=11, job.parallelism=10, args.parallelism=0
```

分析： 类似测试1，job.parallelism依然被忽略 。通过将taskmanager.numberOfTaskSlots增加到11，每个TaskManager提供的任务槽数增加，总任务槽数达到22，任务并行度也相应提高到11。这进一步印证了taskmanager.numberOfTaskSlots在没有args.parallelism设置时对并行度的决定作用。

#### 测试3

```
taskManager.replicas=4, taskmanager.numberOfTaskSlots=1, job.parallelism=1, args.parallelism=0
```

分析： job.parallelism再次被忽略 。配置了4个taskManager.replicas，实际启动了8个TaskManager 。每个TaskManager只有1个任务槽，总任务槽数为8，任务并行度为4 。这显示了当taskManager.replicas大于1时，总任务槽数和任务并行度受两者乘积的影响。

#### 测试4

```
taskManager.replicas=4, taskmanager.numberOfTaskSlots=1, job.parallelism=1, args.parallelism=5
```

分析： 引入了程序层面的args.parallelism=5 。Kafka Source算子的并行度被设置为5，这超出了单个TaskManager的任务槽数（1）和初始配置的replicas*slots总和（4*1=4）。为了满足Source算子的5并行度，系统动态启动了10个TaskManager，提供了10个任务槽 。最终任务并行度显示为Max(5,4)/2，表明算子并行度（5）和集群默认并行度（4）中的最大值决定了任务链的有效并行度。

#### 测试5

```
taskManager.replicas=4, taskmanager.numberOfTaskSlots=3, job.parallelism=2, args.parallelism=5
```

分析： args.parallelism=5对Source算子生效 。集群配置提供了4*3=12个任务槽。实际启动8个TaskManager，总任务槽达到24 。任务并行度为Max(5,12)/2 ，这表明当集群提供的任务槽数（12）能够满足甚至超过args.parallelism（5）时，作业的实际并行度会倾向于利用更多的可用任务槽。

#### 测试6

```
taskManager.replicas=1, taskmanager.numberOfTaskSlots=5, job.parallelism=1, args.parallelism=10
```

分析： args.parallelism=10对Source算子生效 。尽管taskManager.replicas设置为1，但由于Source算子需要10个并行度，而单个TaskManager只能提供5个任务槽，系统动态启动了4个TaskManager，提供了20个任务槽以满足需求。任务并行度为Max(10,5)/2 ，再次验证了args.parallelism的优先级以及它如何驱动TaskManager的弹性扩容。

### 基于测试的并行度配置核心结论

通过上述测试案例的详细分析，可以得出以下关于Flink on K8s并行度配置的核心结论：

#### 结论1：job.parallelism在Flink on K8s中的实际作用

根据所有测试案例的结果，可以明确得出结论：在Flink Kubernetes Operator的Application模式下，job.parallelism配置项对作业的实际并行度没有影响，可以忽略。这意味着用户不应依赖此参数来控制作业的整体并行度。

#### 结论2：taskManager.replicas的默认行为及对TaskManager数量的影响

如果在作业的YAML配置文件中没有显式设置taskManager.replicas，其值可以理解为默认的1 。然而，测试也揭示了taskManager.replicas并非TaskManager数量的严格上限。Flink运行时会根据作业的实际任务槽需求（特别是当args.parallelism或SlotSharingGroup导致更高任务槽需求时）动态请求更多的TaskManager Pods，从而可能导致实际启动的TaskManager数量超过配置值。

#### 结论3：在程序未设置算子并行度时，整体并行度如何由taskManager.replicas和taskmanager.numberOfTaskSlots共同决定

如果taskManager.replicas为1（或未设置），且作业没有在程序层面设置算子的并行度，那么程序整体的并行度将等于taskmanager.numberOfTaskSlots 。

如果taskManager.replicas大于1，且作业没有在程序层面设置算子的并行度，那么程序整体的并行度将等于taskManager.replicas * taskmanager.numberOfTaskSlots 。

这两种情况通过测试1、2、3得到了验证，表明在没有显式算子并行度的情况下，集群配置直接决定了作业的默认并行度。

#### 结论4：当程序层面设置了算子并行度时，其与集群配置的交互逻辑

如果作业在程序层面对某些算子设置了并行度（例如本示例中的args.parallelism），那么这些算子的并行度将由args.parallelism决定，优先级最高。其他没有设置并行度的算子，其并行度仍将由集群配置（即taskManager.replicas * taskmanager.numberOfTaskSlots）决定 。测试4、5、6清晰地展示了args.parallelism如何覆盖默认并行度，并可能导致TaskManager的动态扩容以满足更高的算子并行度需求。

## TaskManager部署策略：少TaskManager多Slot vs 多TaskManager少Slot

在Flink on K8s环境中，TaskManager的部署策略通常可以归结为两种主要模式：“少TaskManager多Slot”和“多TaskManager少Slot”。

- 少TaskManager多Slot： 指的是配置较少的TaskManager Pods，但每个TaskManager拥有更多的任务槽（Slot）。例如，1个TaskManager配置20个任务槽。

- 多TaskManager少Slot： 指的是配置更多的TaskManager Pods，但每个TaskManager只拥有少量（例如1个）任务槽。例如，20个TaskManager，每个配置1个任务槽。

### 官方解析与优缺点

根据官方说明，运行更多小型TaskManager（每个任务一个任务槽）是实现任务之间最佳隔离的良好起点。将相同的资源分配给更少、更大、任务槽更多的TaskManager有助于提高资源利用率，但代价是任务之间的隔离较弱（更多任务共享相同的JVM）。

具体来说：

#### 多TaskManager少Slot的优点：

更好的任务隔离性： 每个任务或少量任务共享一个JVM，可以有效避免“吵闹的邻居”问题，即一个任务的资源消耗（如CPU、内存）影响到同一TaskManager内的其他任务。

#### 多TaskManager少Slot的缺点：

- Kubernetes Pod管理开销大： 大量的TaskManager Pod会增加Kubernetes API
  Server、Scheduler和Kubelet的负担，可能导致Kubernetes集群自身的性能问题，甚至引发“Kubernetes异常” 。

- JVM开销高： 每个TaskManager Pod都需要启动一个独立的JVM，这意味着更多的固定内存和CPU开销，降低了整体资源利用率。

- 网络通信频繁： 更多的TaskManager实例意味着任务之间需要通过网络交换信息时，涉及的节点数量更多，可能增加网络延迟和带宽消耗。

#### 少TaskManager多Slot的优点：

- 提高资源利用率： 可以更好地摊销JVM、应用程序库加载以及网络连接的固定开销，从而提高单个TaskManager的资源利用效率。

- 降低Kubernetes管理开销： 减少TaskManager Pod的数量，减轻Kubernetes集群的调度和管理负担，提高集群稳定性。

- 简化运维： 更少的Pod数量意味着更少的日志需要收集、更少的指标需要监控，简化了故障排查和日常运维。

- 少TaskManager多Slot的缺点：

- 任务隔离性较弱： 更多的任务共享同一个JVM，一个任务的异常（如内存泄漏）可能会影响到同一TaskManager内的其他任务。

### 结合生产实践的策略选择

文档的“建议”部分明确推荐“建议始终将taskManager.replicas设置为1，根据作业实际的任务槽需求数量调大或调小taskmanager.numberOfTaskSlots”。这与“少TaskManager多Slot”策略高度一致。

在Kubernetes这种资源共享且Pod管理存在开销的环境下，为了集群的整体稳定性和资源效率，通常更推荐“少TaskManager多Slot”策略。Kubernetes中，Pod是基本的调度和管理单元。大量的Pod会给API Server、Scheduler、Kubelet带来显著的额外负担，可能导致Kubernetes自身的运维复杂度和资源消耗增加。将更多任务槽集中在少数TaskManager中，可以更好地摊销Flink TaskManager的JVM启动、公共库加载等固定开销，提高整体资源利用率，从而降低成本。尽管Flink内部有高效的网络传输机制，但减少TaskManager间的网络通信节点数量，通常能降低整体网络延迟和带宽消耗，尤其对于数据量大的Shuffle操作。此外，这种策略也契合“单一职责小作业原则”，即使是“小作业”，如果其并行度需求较高，通过“少TaskManager多Slot”也能在少量TaskManager内满足，避免TaskManger数量爆炸。对于大型作业，这种策略更是控制Kubernetes Pod数量的关键。

尽管“多TaskManager少Slot”提供了更好的隔离性，但在大多数Flink on K8s生产场景下，为了集群的整体稳定性、资源效率和运维便利性，通常更推荐“少TaskManager多Slot”策略。仅当单个TaskManager的资源瓶颈（如CPU或内存）成为问题，或者需要极端严格的资源隔离以避免“吵闹的邻居”问题时，才应考虑“多TaskManager少Slot”策略，但需权衡其带来的额外开销。

## 生产环境Flink on K8s并行度配置最佳实践

在生产环境中配置Flink on K8s的并行度，需要综合考虑作业特性、资源利用率和集群稳定性。以下是基于测试结果和生产经验的最佳实践建议：

### 核心原则：遵循单一职责小作业原则

在实际应用中，建议遵循单一职责小作业原则 。这意味着将大型复杂作业拆分为多个单一职责的小型作业，每个作业处理特定的数据流和业务逻辑。这不仅有助于简化并行度配置和资源管理，还能提高作业的韧性和可维护性。对于大多数“小作业”，其所需的任务槽数量通常不会太高，建议在16个以内 。

### TaskManager数量控制：强烈建议将taskManager.replicas设置为1

这是本文最核心的生产建议之一。为了控制Flink作业的TaskManager数量，避免TaskManager数量爆炸导致Kubernetes异常，建议始终将taskManager.replicas设置为1。通过将taskManager.replicas固定为1，可以有效避免在作业并行度需求较高时，Flink Kubernetes Operator动态创建过多的TaskManager Pods，从而导致Kubernetes集群过载、调度效率下降甚至异常。这种策略强制采用“少TaskManager多Slot”模式，最大化单个TaskManager的资源利用率。

### 任务槽数量调整：根据作业实际需求，灵活调整taskmanager.numberOfTaskSlots

在taskManager.replicas=1的前提下，taskmanager.numberOfTaskSlots成为主要控制作业并行度的参数。根据作业的实际吞吐量、延迟要求和算子特性（如Kafka Source的分区数、KeyBy后的数据倾斜情况）来估算所需的总任务槽数 。例如，如果作业需要100个任务槽，可以考虑将taskmanager.numberOfTaskSlots设置为10或20，这样总的TaskManager数量（即使动态扩容）也能控制在合理范围内 。

### 任务槽与CPU配比：给出taskmanager.numberOfTaskSlots与TaskManager CPU数量的合理配比建议

尽管任务槽数量与CPU核数没有严格的对应要求，但建议taskmanager.numberOfTaskSlots的数量与TaskManager的CPU数量保持一致，或者是2:1的比例 。

- 1:1比例 (任务槽:CPU)： 适用于CPU密集型作业。每个任务槽拥有独立的CPU核心，最大化并行计算能力，减少上下文切换，从而提供更好的性能隔离和可预测性。

- 2:1比例 (任务槽:CPU)： 适用于I/O密集型或混合型作业。当任务在等待I/O（如从Kafka读取数据、写入数据库）时，CPU可以切换到另一个任务槽的任务，从而提高CPU利用率。这种配置在资源有限的情况下，可以提高整体吞吐量，但可能引入更多的上下文切换开销。

此外，需要注意的是，每个任务槽都会消耗内存。当taskmanager.numberOfTaskSlots设置得很高时，必须确保TaskManager的内存资源（taskManager.resource.memory）足够支撑所有任务槽的运行。否则，即使CPU充足，也可能因内存不足导致性能问题或OutOfMemoryError。建议根据单个任务的内存需求和numberOfTaskSlots来估算总内存。资源配比不是一成不变的，需要根据作业的实际工作负载特性（CPU密集、I/O密集、内存需求）进行调整。这是一个迭代优化的过程，需要结合实际压测和监控数据进行调整。

### JobManager资源配置

JobManager通常不执行计算任务，主要负责作业协调、调度、状态管理和Checkpoint协调。其资源需求相对较低，但需要保证高可用性。

- CPU： 建议至少配置1-2核。

- 内存： 建议至少2-4GB，具体取决于作业数量、状态信息量以及Checkpoint元数据的大小。对于管理大量作业或大型状态的场景，可能需要更多内存。

- 高可用（HA）： 生产环境务必配置JobManager HA，通过设置jobManager.replicas: 2或更高来确保JobManager的健壮性。

### 整体资源规划与优化

Flink on K8s的并行度配置是一个系统工程，需要从作业的实际业务需求出发，层层推导至Kubernetes的资源配置，并进行迭代优化。作业的并行度（无论是全局设置还是算子级别设置）直接决定了所需的总任务槽数。总任务槽数和每个TaskManager的任务槽数决定了所需的TaskManager Pod数量。每个TaskManager Pod的任务槽数又决定了其所需的CPU和内存资源。JobManager需要足够的资源来管理这些TaskManager和作业状态，确保调度的顺畅和Checkpoint的成功。最终，所有这些资源需求都必须在Kubernetes集群的整体容量和资源配额之内。仅仅调整某个参数而不考虑其对其他部分的影响，可能导致次优甚至失败的部署。因此，建议用户在配置时进行整体考量，并结合实际的性能压测和监控数据进行持续调整，以达到最佳的性能、稳定性和成本效益。

## 总结与生产配置速查表

本文深入探讨了Flink on K8s环境下并行度配置的关键概念和实践。通过一系列测试案例验证了核心结论：job.parallelism在Flink Kubernetes Operator的Application模式下通常无效，而作业的实际并行度主要受taskManager.replicas、taskmanager.numberOfTaskSlots以及程序内算子并行度（args.parallelism）的共同控制。

对于生产环境，强烈建议遵循“单一职责小作业原则”，并采取“少TaskManager多Slot”的部署策略，即始终将taskManager.replicas设置为1，并通过灵活调整taskmanager.numberOfTaskSlots来满足作业所需的总任务槽数。同时，应根据作业类型合理配置任务槽与CPU的比例（1:1或2:1），并确保TaskManager具备足够的内存资源。正确的并行度配置是确保Flink作业在Kubernetes上高效、稳定运行的关键，它能够有效控制TaskManager数量，避免Kubernetes集群异常，并优化整体资源利用率。

为了方便读者快速参考和应用于实际生产环境，以下将关键配置项及其推荐值或配置逻辑汇总为速查表。

**Flink on K8s并行度生产配置速查表**

| 配置项                                                             | 推荐值/配置逻辑                                                           | 说明                                                      |
|-----------------------------------------------------------------|--------------------------------------------------------------------|---------------------------------------------------------|
| `job.parallelism`                                               | 忽略（或不设置）                                                           | 在Flink on K8s Application模式下，对实际并行度无影响。                 |
| `taskManager.replicas`                                          | 1                                                                  | **强烈推荐**：控制TaskManager Pod数量，避免K8s集群过载及TaskManager数量爆炸。 |
| `taskmanager.numberOfTaskSlots`                                 | 根据作业总任务槽需求调整（例如：作业需100任务槽，可设为10或20）                                | 每个TaskManager的任务槽数，与 `replicas=1` 协同控制作业总任务槽数。          |
| `args.parallelism`（程序内全局算子并行度）                                  | 根据Flink应用需求设置，用于决定Flink并行度的关键参数（通过参数传递，需要在代码中手动设置）。                | 算子级别并行度，优先级最高，会覆盖集群默认设置，需确保总任务槽能满足。                     |
| `SlotSharingGroup`                                              | 灵活使用                                                               | 用于优化任务槽利用率或强制任务隔离，特别适用于多Source或强隔离需求场景。                 |
| `taskmanager.numberOfTaskSlots` 与 `taskManager.resource.cpu` 比例 | 1:1 或 2:1                                                          | 优化CPU利用率：1:适用于CPU密集型，2:适用于I/O密集型或混合型。                   |
| `taskManager.resource.memory`                                   | 根据 `taskmanager.numberOfTaskSlots` 和单任务槽内存需求计算（推荐与slot:memory为1:4） | 确保TaskManager有足够内存支撑所有任务槽运行，避免OOM。                      |
| `jobManager.replicas`                                           | >=2（推荐2）                                                           | 确保JobManager高可用性。                                       |
| `jobManager.resource.cpu`                                       | 1-2核                                                               | JobManager协调和调度所需CPU资源。                                 |
| `jobManager.resource.memory`                                    | 2-4GB+（根据作业数量和状态信息量调整）                                             | JobManager协调和调度所需内存资源。                                  |
|                                                                 |                                                                    |                                                         |