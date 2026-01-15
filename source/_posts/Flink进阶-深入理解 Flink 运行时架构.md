---
title: Flink进阶-深入理解 Flink 运行时架构
abbrlink: d86881fa
date: 2025-12-16 17:09:28
tags:
  - 大数据
categories:
  - 大数据
toc: true
---


在使用 Apache Flink 进行大数据流处理时，理解其底层的运行时架构（Runtime Architecture）是进行性能调优和源码阅读的基础。Flink的架构遵循经典的 **Master-Slave** 模式，但其内部组件的分工非常精细。

本文将深入剖析 Flink 核心架构中的关键名词：**Dispatcher**、**WebMonitorEndpoint**、**ResourceManager**、**JobMaster**、*
*TaskManager**、**Slot** 以及 **SubTask**，并梳理它们之间的协作流程。

---

## 一、 核心组件概览图

在进入细节之前，我们需要建立一个宏观的认知。Flink 的运行时架构主要分为两大阵营：

1. **JobManager (Master)**：负责作业管理和资源调度（包含 Dispatcher, ResourceManager, JobMaster, WebMonitorEndpoint）。
2. **TaskManager (Slave)**：负责具体的任务执行（包含 Slot, SubTask）。

---

## 二、 集群入口与资源管理 (Cluster Entry & Resource Management)

这部分组件负责处理外部请求和全局资源的分配。

### 1. WebMonitorEndpoint (WebMonitor)

* **角色定位**：**REST 接口与 Web 面板的后端**。
* **核心职能**：
    * 它是 Flink 集群与外部交互的主要窗口。
    * 负责暴露 REST API，供 Flink Web UI 查询作业状态、指标、日志等信息。
    * 在客户端提交作业时，它也负责接收 JAR 包和请求（如果是通过 REST 方式提交）。
* **通俗理解**：它是 Flink 的“门户网站”和“API 网关”。

### 2. Dispatcher

* **角色定位**：**作业提交的“前台接待”**。
* **核心职能**：
    * 接收客户端提交的作业（JobGraph）。
    * 为每一个提交的作业启动一个专属的 **JobMaster**。
    * 运行 Flink Web UI（在 Session 模式下）。
    * 在作业失败时负责恢复作业（取决于具体的高可用模式）。
* **通俗理解**：就像公司的前台，有人来办事（提交作业），它负责登记并找一个专门的项目经理（JobMaster）来对接。

### 3. ResourceManager (RM)

* **角色定位**：**资源的“人事部”**。
* **核心职能**：
    * 负责管理 Flink 集群中的计算资源——**Task Slot（槽位）**。
    * 当 JobMaster 申请资源时，RM 会查看是否有空闲的 Slot；如果没有，它会向外部资源管理器（如 YARN, Kubernetes, Mesos）申请启动新的
      TaskManager 容器。
    * 它还要处理 TaskManager 的注册和心跳，监控从节点的健康状况。
* **通俗理解**：它是掌握公司所有工位（Slot）资源的人事总监。项目经理（JobMaster）缺人手，必须找它要。

---

## 三、 作业管理 (Job Management)

### 4. JobMaster

* **角色定位**：**单个作业的“项目经理”**。
* **核心职能**：
    * **这是 Flink 中非常关键的组件**。每个正在运行的作业都有一个独立的 JobMaster。
    * 它负责将 **JobGraph**（逻辑视图）转化为 **ExecutionGraph**（物理执行图）。
    * 向 ResourceManager 申请 Slot。
    * 将具体的计算任务（Task）分发给 TaskManager 执行。
    * 协调 Checkpoint（检查点）的触发和状态快照。
    * 处理任务失败后的重启策略。
* **通俗理解**：它是专门负责某个具体项目（Job）的经理，全权负责该项目的执行、进度监控和容错，直到项目结束。

---

## 四、 任务执行 (Task Execution)

这部分是实际干活的地方，位于从节点上。

### 5. TaskManager (TM)

* **角色定位**：**实际工作的“工人/工作进程”**。
* **核心职能**：
    * TaskManager 是一个 JVM 进程。
    * 它启动时会将自己的资源（Slot）注册到 ResourceManager。
    * 接收来自 JobMaster 的 Task 部署请求。
    * 负责数据流的交换（Network Stack），即不同 TaskManager 之间的数据传输。
* **通俗理解**：它是具体的施工队，里面包含了很多具体的工位（Slot）。

### 6. Slot (TaskSlot)

* **角色定位**：**资源的“容器/工位”**。
* **核心职能**：
    * Slot 是 TaskManager 资源管理的最小单位。
    * 它主要隔离的是 **内存（Managed Memory）**，但默认情况下**不隔离 CPU**。这意味着一个 Slot 中的多个 SubTask 会共享 CPU
      时间片。
    * 一个 TaskManager 有多少个 Slot，通常就意味着它能支持多少并发度（Parallelism）。
* **通俗理解**：这是给工人干活的工位。一个工位虽然只能坐一个人，但 Flink 允许“插槽共享”（Slot Sharing），即不同任务的子任务可以挤在一个工位上（比如
  Source 和 Map 可以在同一个 Slot 运行，节省资源）。

### 7. SubTask

* **角色定位**：**执行的具体“线程”**。
* **核心职能**：
    * SubTask 是 Flink 任务执行的最小单元。
    * 如果你写了一个算子（Operator），比如 `map()`，并且设置并行度为 2，那么运行时就会有两个 SubTask（`map_1`, `map_2`）。
    * 每个 SubTask 都是一个独立的线程（Thread）。
* **通俗理解**：这是真正干活的“人”。如果说 Slot 是工位，SubTask 就是坐在工位上处理数据的具体线程。

---

## 五、 总结：从提交到执行的全流程

为了串联上述概念，我们看一个作业的生命周期：

1. **提交**：用户通过 Flink Client 提交作业，请求发送给 **WebMonitorEndpoint** 或直接给 **Dispatcher**。
2. **启动 Master**：**Dispatcher** 接收请求，为该作业启动一个专属的 **JobMaster**。
3. **资源申请**：**JobMaster** 分析作业需要多少资源，向 **ResourceManager** 请求 Slot。
4. **资源分配**：
    * 如果 **ResourceManager** 发现有空闲的 **TaskManager**，就命令 TM 将 Slot 分配给 JobMaster。
    * 如果不够，RM 会向 YARN/K8s 申请启动新的 TM。
5. **部署**：**TaskManager** 向 **JobMaster** 提供 Slot。JobMaster 将 **SubTask**（具体的代码逻辑）部署到这些 **Slot** 中。
6. **执行**：**SubTask** 在 Slot 中运行（线程启动），开始消费和处理数据。

---

### 关键点对比表

| 组件                  | 层级       | 关键职责                  | 备注        |
|---------------------|----------|-----------------------|-----------|
| **Dispatcher**      | Cluster  | 接收作业，拉起 JobMaster     | 集群的大门     |
| **ResourceManager** | Cluster  | 管理 Slot，统筹资源          | 资源的分配者    |
| **JobMaster**       | Job      | 管理单个作业的执行图、Checkpoint | 单个作业的大脑   |
| **TaskManager**     | Node     | JVM 进程，持有 Slot        | 资源的提供者    |
| **Slot**            | Resource | 静态资源隔离（内存）            | 并发能力的量度   |
| **SubTask**         | Runtime  | 实际运行的线程               | 真正处理数据的单元 |

希望这篇博文能帮助你彻底厘清 Flink 运行时各组件的关系！