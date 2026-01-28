---
title: Flink1.18.1本地idea源码调试环境搭建
abbrlink: d1ff1ddd
date: 2026-01-23 12:35:57
tags:
  - 大数据
categories:
  - 大数据
toc: true
---


Apache Flink 作为业界领先的流处理和批处理统一计算引擎，其强大的功能与复杂的内部机制吸引了无数开发者深入探索。对于后端开发者而言，能够直接在本地 IDE 中调试 Flink 源码，无疑是提升理解、快速定位问题、甚至参与社区贡献的利器。然而，搭建这样一个庞大项目的本地调试环境，尤其是特定版本如 Flink 1.18.1，往往涉及到诸多配置细节，令不少初学者望而却步。本篇博文旨在提供一份详尽的、按部就班的指南，帮助您在 Windows 系统下，使用 IntelliJ IDEA 顺利搭建起 Flink 1.18.1 的源码调试环境。通过本文的指引，您将能够轻松配置项目、编译源码，并成功启动一个可供调试的本地 Flink Standalone 集群，为您的 Flink 深度学习之旅奠定坚实基础。

<!-- more -->

> **温馨提示：**
> 1. 本文所有命令行操作均在 `Git Bash` 中执行。
> 2. 请确保您的网络环境已启用科学上网。

环境准备
---

1. windows/linux
2. 安装jdk1.8并配置好环境变量
3. 安装git
4. 安装idea

项目准备
---

**1. git配置**

**通用配置（在 Windows 和 Linux 下都执行）**
```shell
# [基础身份]
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# [显示优化] 让 git status 显示中文文件名，而不是 \346\265... 乱码
git config --global core.quotepath false

# [行为优化] git pull 时默认使用 rebase 模式
# 作用：避免在拉取代码时产生无意义的 "Merge branch..." 提交记录，保持提交线是一条直线。
git config --global pull.rebase true

# [行为优化] 新建仓库时默认分支名为 main (代替过时的 master)
git config --global init.defaultBranch main

# [效率提升] 常用命令简写别名 (可选，强烈推荐)
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.br branch
# 一个非常酷的 log 查看命令：显示提交图谱、精简信息
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

**仅Windows下执行**
```shell
# [核心兼容] 提交转 LF，检出转 CRLF (适配 Windows 编辑器)
git config --global core.autocrlf true

# [Java开发必备] 开启长路径支持
# Java 项目层级深、类名长，很容易突破 Windows 260 字符限制，此项必须开启
git config --global core.longpaths true

# [权限忽略] 建议开启
# 因为 Windows (NTFS) 和 Linux (Ext4) 的文件权限模型不同 (如 755/644)，
# 防止因为文件权限变动导致 Git 认为文件被修改。
git config --global core.filemode false
```

**仅Linux下执行**
```shell
# [核心兼容] 提交转 LF，检出不转换 (保持 Linux 原生 LF)
git config --global core.autocrlf input

# [安全检查] 开启对 NTFS 文件名的保护 (即使在 Linux 上)
# 作用：防止你在 Linux 上创建了 Windows 无法识别的文件名（如含冒号、尾随空格等），
# 导致你切回 Windows 系统时报错。
git config --global core.protectNTFS true
```

**2. 拉取源码**

```shell
git clone https://github.com/apache/flink.git
```

**3. 切换分支**

```shell
cd ./flink
git checkout -b learning/release-1.18.1 release-1.18.1
```

**4. idea安装scala插件**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605135559429.png)

**5. idea配置maven**

> 因为已经科学上网了，所以本地的maven配置文件`setting.xml`只需要配置一下本地仓库地址即可

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605135724091.png)

**6. idea配置Project Structure**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605135941465.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605140144572.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605140320603.png)

**7. 安装 Python**

```shell
# windows
winget install Python.Python.3

# debian/ubuntu
sudo apt update
sudo apt install build-essential python3
```

**8. 执行maven打包**

> idea中双击ctrl，调出`Run Anything`，执行下面命令，第一次执行时间可能会比较久

```shell
mvn clean install -DskipTests -Pfast -U
```

**9. 移动conf与lib**

> 在项目根目录下执行

```shell
mkdir -p {conf,distlib}
cp -r ./flink-dist/target/flink-1.18.1-bin/flink-1.18.1/conf/* ./conf
cp -r ./flink-dist/target/flink-1.18.1-bin/flink-1.18.1/lib/* ./distlib
```

启动master
---

> standalone集群模式

**1. 找到启动类**

通过`ctrl + n`查看类`org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint`

**2. 修改启动类启动配置**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605141742395.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250802053232125.png)

> Add vm options: `-Dlog.file=./log/flink-jobmanager-1.local.log -Dlog4j.configuration=./conf/log4j-console.properties -Dlog4j.configurationFile=./conf/log4j-console.properties -Dlogback.configurationFile=./conf/logback-console.xml`

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605151932563.png)

> 修改类加载路径

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152032642.png)‘

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152126871.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152250068.png)

**3. 最后启动主类即可**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152730149.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605152833245.png)

启动TaskManager
---

> standalone集群模式

**1. 找到启动类**

通过`ctrl + n`查看类`org.apache.flink.runtime.taskexecutor.TaskManagerRunner`

**2. 修改启动类启动配置**

和master操作步骤一致即可，除了将add vm options的值修改为: `-Dlog.file=./log/flink-taskmanager-1.local.log -Dlog4j.configuration=./conf/log4j-console.properties -Dlog4j.configurationFile=./conf/log4j-console.properties -Dlogback.configurationFile=./conf/logback-console.xml`

**3. 修改flink-conf配置**

在`conf/flink-conf.yaml`文件中添加如下内容：

```yaml
taskmanager.resource-id: taskmanager_01
taskmanager.cpu.cores: 1
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

**4. 最后启动主类即可**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605162306391.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605162424201.png)

提交作业到flink集群
---

**1. 打包生成示例jar包**

```shell
git clone https://github.com/liboshuai01/learn.git
cd ./learn/flink/flink-example
mvn clean install -DskipTests
```

然后拷贝`target`中的`flink-example-1.0-SNAPSHOT.jar`到flink项目根目录的`distlib`文件夹中。

**2. 找到提交作业入口类**

通过`ctrl + n`查看类`org.apache.flink.client.cli.CliFrontend`

**3. 修改启动类启动配置**

> Add vm options: `-Dlog.file=./log/flink-hunter-client-hunter.log -Dlog4j.configuration=./conf/log4j-cli.properties -Dlog4j.configurationFile=./conf/log4j-cli.properties -Dlogback.configurationFile=./conf/logback-console.xml `
>
> Program arguments: `run -c cn.liboshuai.learn.flink.example.WordCount ./distlib/flink-example-1.0-SNAPSHOT.jar`
>
> Environment variables: `FLINK_CONF_DIR=./conf`

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250605185435332.png)

**4. 使用wsl2创建一个socket监听端口**

> `nc -l 9999`

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250802054817998.png)

**5. 右键启动`CliFrontend`类即可**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250802055036933.png)

> 浏览器访问`http://127.0.0.1:8081`

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250802055137221.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250802055157085.png)


---

结语
---

至此，您已经成功地在 IntelliJ IDEA 中搭建起了 Flink 1.18 的本地源码调试环境。这不仅仅意味着您拥有了一个可以运行和观察 Flink 的平台，更重要的是，您打开了一扇通往 Flink 内核世界的大门。现在，您可以自由地在 JobManager、TaskManager 的核心代码中设置断点，跟踪一个作业的提交流程，观察数据在算子间的流动，或是探究资源调度与状态管理的奥秘。充分利用这个环境，积极动手实践，无论是为了解决实际工作中的疑难杂症，还是为了向 Flink 社区贡献自己的一份力量，都将使您的 Flink 技术水平得到质的飞跃。希望本篇指南能为您的 Flink 之旅提供有力的支持，祝您在探索 Flink 的过程中收获满满，享受编码与调试的乐趣！