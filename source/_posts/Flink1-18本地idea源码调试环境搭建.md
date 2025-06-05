---
title: Flink1.18本地idea源码调试环境搭建
abbrlink: d1ff1ddd
date: 2025-06-05 16:25:37
tags:
  - Flink
  - 源码
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605163341216.png'
toc: true
---

Apache Flink 作为业界领先的流处理和批处理统一计算引擎，其强大的功能与复杂的内部机制吸引了无数开发者深入探索。对于后端开发者而言，能够直接在本地 IDE 中调试 Flink 源码，无疑是提升理解、快速定位问题、甚至参与社区贡献的利器。然而，搭建这样一个庞大项目的本地调试环境，尤其是特定版本如 Flink 1.18，往往涉及到诸多配置细节，令不少初学者望而却步。本篇博文旨在提供一份详尽的、按部就班的指南，帮助您在 Windows 系统下，使用 IntelliJ IDEA 顺利搭建起 Flink 1.18 的源码调试环境。通过本文的指引，您将能够轻松配置项目、编译源码，并成功启动一个可供调试的本地 Flink Standalone 集群，为您的 Flink 深度学习之旅奠定坚实基础。

<!-- more -->

> **温馨提示：**
> 1. 本文所有命令行操作均在 `Git Bash` 中执行。
> 2. 请确保您的网络环境已启用科学上网。

环境准备
---

在开始之前，请确保您的开发环境满足以下条件：

1.  **操作系统：** Windows 系统
2.  **JDK：** 安装 JDK 1.8 并正确配置 `JAVA_HOME` 环境变量及 Path。
3.  **Git：** 安装 Git 版本控制工具。
4.  **IDE：** 安装 IntelliJ IDEA。

Git 全局配置
---

为了保证后续操作的顺利进行，建议对 Git 进行如下全局配置：

```shell
# 配置全局用户名称
git config --global user.name "Your Name"
# 配置全局用户邮箱
git config --global user.email "your.email@example.com"
# 禁用 NTFS 保护性限制（建议谨慎使用，可能影响文件系统安全性）
git config --global core.protectNTFS false
# 关闭自动行结束符转换，避免跨平台换行符差异导致的问题
git config --global core.autocrlf false
# 允许 Git 正确显示中文文件名，避免乱码
git config --global core.quotepath false
# 在 Windows 系统下，允许处理超过 260 字符的路径
git config --global core.longpaths true
```

拉取 Flink 源码
---

从 Apache Flink 的官方 GitHub 仓库克隆源码：

```shell
git clone https://github.com/apache/flink.git
```

切换到目标分支
---

进入克隆下来的 `flink` 项目根目录，并切换到 `release-1.18` 分支：

```shell
cd flink
git checkout -b flink-1.18 origin/release-1.18
```
*（我们创建了一个本地分支 `flink-1.18` 并跟踪远程的 `origin/release-1.18` 分支）*

配置 Maven 编译 JDK 版本
---

为了确保项目使用正确的 JDK 版本进行编译，需要修改项目根目录下的 `pom.xml` 文件。
找到 `<artifactId>maven-compiler-plugin</artifactId>` 的 `<plugin>` 配置节（大约在1700行附近，具体行数可能因版本细微差异而不同），为其添加 `<configuration>` 部分，指定编译源码和目标字节码的 Java 版本。

修改前（仅作参考，具体以实际 `pom.xml` 为准）：
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <!-- 可能已存在其他配置 -->
</plugin>
```

添加如下 `<configuration>` 内容：
```xml
<configuration>
    <source>${maven.compiler.source}</source>
    <target>${maven.compiler.target}</target>
</configuration>
```

添加后的 `<plugin>` 标签内容示例如下：
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <!-- 新增内容 -->
    <configuration>
        <source>${maven.compiler.source}</source>
        <target>${maven.compiler.target}</target>
    </configuration>
    <!-- 可能已存在其他配置 -->
</plugin>
```
修改 `pom.xml` 后，请记得在 IDEA 右侧的 Maven 面板中点击“Reload All Maven Projects”按钮，以使更改生效。

IDEA 安装 Scala 插件
---

Flink 的部分核心代码使用 Scala 编写，因此需要在 IDEA 中安装 Scala 插件以获得良好的代码支持和编译能力。
您可以通过 `File -> Settings -> Plugins`，在 Marketplace 中搜索 "Scala" 并安装。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605135559429.png)

IDEA 配置 Maven
---

由于我们已经配置了科学上网，本地 Maven 的 `settings.xml` 文件主要配置本地仓库地址即可。
在 IDEA 中，通过 `File -> Settings -> Build, Execution, Deployment -> Build Tools -> Maven` 进行配置。
确保 "User settings file" 和 "Local repository" 指向您自定义的 Maven 配置文件和仓库位置。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605135724091.png)

IDEA 配置 Project Structure
---

检查并配置项目的 SDK 和语言级别。
1.  **Project SDK:** 打开 `File -> Project Structure -> Project`，确保 "SDK" 设置为您安装的 JDK 1.8。
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605135941465.png)
2.  **Modules JDK:** 切换到 `Modules` 标签页，为主要的 Flink 模块（特别是 `flink-clients`, `flink-core`, `flink-runtime`, `flink-streaming-java` 等）检查其 "Module SDK"，确保继承自项目 SDK 或单独设置为 JDK 1.8。
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605140144572.png)
3.  **Language Level:** 确保各模块的语言级别（Language Level）与 JDK 版本匹配，通常为 "8 - Lambdas, type annotations etc."。
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605140320603.png)

执行 Maven 打包
---

在 IDEA 中，可以通过 `Ctrl + Ctrl` (连按两次) 快捷键调出 "Run Anything" 窗口，然后执行以下 Maven 命令进行项目编译和打包。
首次执行此命令，由于需要下载大量依赖，可能耗时较长，请耐心等待。

```shell
mvn clean install -DskipTests -Pfast -U
```
参数说明：
*   `clean install`: 清理并构建项目，将产物安装到本地 Maven 仓库。
*   `-DskipTests`: 跳过测试用例执行，加快编译速度。
*   `-Pfast`: 激活 "fast" profile，该 profile 通常会跳过一些耗时的插件（如代码风格检查、JavaDoc 生成等）。
*   `-U`: 强制更新 SNAPSHOT 依赖。

移动 Flink 配置与库文件
---

为了方便在 IDEA 中直接启动 Flink Standalone 集群进行调试，我们需要将编译后生成的配置文件和库文件复制到项目根目录下的特定位置。
在项目根目录下（`flink/`）执行以下命令：

```shell
# 创建目标目录
mkdir -p conf distlib
# 复制配置文件
cp -r ./flink-dist/target/flink-*-bin/flink-*/conf/* ./conf
# 复制库文件
cp -r ./flink-dist/target/flink-*-bin/flink-*/lib/* ./distlib
```
*注意：上述命令中的 `flink-*` 是通配符，以适应不同快照版本号。请根据实际 `flink-dist/target/` 下的目录名调整。例如，对于 `flink-1.18-SNAPSHOT-bin/flink-1.18-SNAPSHOT`，命令会是：*
```shell
cp -r ./flink-dist/target/flink-1.18-SNAPSHOT-bin/flink-1.18-SNAPSHOT/conf/* ./conf
cp -r ./flink-dist/target/flink-1.18-SNAPSHOT-bin/flink-1.18-SNAPSHOT/lib/* ./distlib
```

启动 Master (Standalone 集群模式)
---

**1. 找到启动类**

使用 IDEA 的 `Ctrl + N` 快捷键，搜索并打开 `org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint` 类。

**2. 修改启动类运行配置**

右键点击该类中的 `main` 方法，选择 "Modify Run Configuration..." (如果之前未运行过，则为 "Run 'StandaloneSes...'")。

*   **VM options:** 添加以下 JVM 参数，用于指定日志文件位置和日志配置。
    ```
    -Dlog.file=./log/flink-jobmanager-1.local.log -Dlog4j.configuration=./conf/log4j-console.properties -Dlog4j.configurationFile=./conf/log4j-console.properties -Dlogback.configurationFile=./conf/logback-console.xml
    ```
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605141742395.png)

*   **Working directory:** 确保 "Working directory" 设置为 Flink 项目的根目录（例如 `D:\path\to\your\flink`）。
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605151932563.png)

*   **Modify options -> Modify classpath:** 确保 `flink-runtime` 模块的 `classes` 和 `resources` 目录在 classpath 的最前面，以便加载最新的代码和配置。
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152032642.png)
    将 `flink-runtime` 相关的条目（通常是 `flink-runtime_main classes` 或类似名称）移动到列表顶部。
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152126871.png)
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152250068.png)

**3. 启动 Master**

完成上述配置后，点击 "Apply" 和 "OK"，然后点击运行或调试按钮启动 Master 进程。
成功启动后，您应该能在控制台看到类似 JobManager 启动成功的日志。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152730149.png)

访问 Flink Web UI (默认为 `http://localhost:8081`)，可以看到 Master 已经运行。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152833245.png)

启动 TaskManager (Standalone 集群模式)
---

**1. 找到启动类**

使用 `Ctrl + N` 快捷键，搜索并打开 `org.apache.flink.runtime.taskexecutor.TaskManagerRunner` 类。

**2. 修改启动类运行配置**

参照 Master 的配置步骤，为 `TaskManagerRunner` 创建或修改运行配置。关键区别在于：

*   **VM options:** 修改为 TaskManager 的日志配置。
    ```
    -Dlog.file=./log/flink-taskmanager-1.local.log -Dlog4j.configuration=./conf/log4j-console.properties -Dlog4j.configurationFile=./conf/log4j-console.properties -Dlogback.configurationFile=./conf/logback-console.xml
    ```
*   **Working directory:** 同样设置为 Flink 项目的根目录。
*   **Modify options -> Modify classpath:** 同样确保 `flink-runtime` 模块的 `classes` 和 `resources` 目录在 classpath 的最前面。

**3. 修改 `flink-conf.yaml` 配置**

为了让本地 TaskManager 能够成功注册到 Master 并模拟一定的资源，我们需要在项目根目录的 `conf/flink-conf.yaml` 文件中添加（或修改）以下基础配置。如果文件不存在，可以从 `flink-dist/src/main/resources/flink-conf.yaml` 复制一份模板过来。

```yaml
# (确保之前已有的 rest.port, jobmanager.rpc.address 等配置正确或使用默认)

# TaskManager 资源配置 (示例)
taskmanager.resource-id: taskmanager_01 # 为 TaskManager 指定一个唯一的 ID
taskmanager.numberOfTaskSlots: 2        # 配置 TaskManager 的槽位数

# 以下为更细致的内存配置示例，根据需要调整
taskmanager.cpu.cores: 1.0
taskmanager.memory.task.heap.size: 512m
taskmanager.memory.managed.size: 512m
taskmanager.memory.network.min: 128m
taskmanager.memory.network.max: 128m
taskmanager.memory.task.off-heap.size: 0m
taskmanager.memory.framework.heap.size: 256m
taskmanager.memory.framework.off-heap.size: 128m
taskmanager.memory.jvm-metaspace.size: 128m
taskmanager.memory.jvm-overhead.max: 128m
taskmanager.memory.jvm-overhead.min: 128m
```
*请确保 `jobmanager.rpc.address` 配置为 `localhost` (或您Master实际监听的地址)，`rest.port` 指向 Master 的 Web UI 端口（默认为 8081）。如果 `flink-conf.yaml` 中没有这些配置，Flink 会使用默认值。对于本地调试，通常默认值即可。*

**4. 启动 TaskManager**

配置完成后，启动 `TaskManagerRunner` 的 `main` 方法。
成功启动后，控制台会显示 TaskManager 相关的日志。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605162306391.png)

此时刷新 Flink Web UI (`http://localhost:8081`)，应该能看到 TaskManager 已经注册上来，并且有可用的 Task Slots。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605162424201.png)


---

结语
---

至此，您已经成功地在 IntelliJ IDEA 中搭建起了 Flink 1.18 的本地源码调试环境。这不仅仅意味着您拥有了一个可以运行和观察 Flink 的平台，更重要的是，您打开了一扇通往 Flink 内核世界的大门。现在，您可以自由地在 JobManager、TaskManager 的核心代码中设置断点，跟踪一个作业的提交流程，观察数据在算子间的流动，或是探究资源调度与状态管理的奥秘。充分利用这个环境，积极动手实践，无论是为了解决实际工作中的疑难杂症，还是为了向 Flink 社区贡献自己的一份力量，都将使您的 Flink 技术水平得到质的飞跃。希望本篇指南能为您的 Flink 之旅提供有力的支持，祝您在探索 Flink 的过程中收获满满，享受编码与调试的乐趣！
