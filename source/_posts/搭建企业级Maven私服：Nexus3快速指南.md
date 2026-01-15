---
title: 搭建企业级Maven私服：Nexus3快速指南
abbrlink: 3f978590
date: '2025-07-12T14:04:43.000Z'
tags:
  - 杂货小铺
categories:
  - 杂货小铺
toc: true
---


在软件开发中，Maven私服是不可或缺的工具，它能有效管理项目依赖、加速构建过程、统一团队内部组件发布，并提升安全性。本文将详细指导您如何快速安装Nexus Repository Manager 3，并配置常用的仓库类型及Maven客户端，实现高效的内部依赖管理和构件发布。

<!-- more -->

## Nexus3环境准备

我们将使用Docker Compose来部署Nexus3，这能提供极大的便利性和环境隔离。

**1.1. 克隆项目仓库**

首先，从GitHub克隆包含Nexus3 Docker Compose配置的仓库。

```shell
# 如果在国内访问GitHub较慢，可以考虑使用Gitee镜像地址：
# git clone https://gitee.com/liboshuai01/docker-stack.git
git clone https://github.com/liboshuai01/docker-stack
```

**1.2. 进入Nexus3目录**

克隆完成后，进入[`docker-stack/nexus3`](http://docker-stack/nexus3)目录。

```shell
cd docker-stack/nexus3
```

**1.3. 准备环境变量文件**

复制 `.env.example` 文件为 `.env`，这里通常会定义Nexus的数据卷路径、端口映射等。

```shell
cp .env.example .env
```

**1.4. 启动Nexus3容器**

使用`docker-compose`在后台启动Nexus3服务。

```shell
docker-compose up -d
```
启动可能需要一些时间，Nexus3首次启动会进行初始化。

## 获取Nexus3管理员密码

Nexus3首次启动时会生成一个随机的管理员密码，我们需要获取它来登录。

```shell
docker-compose exec nexus3 cat /nexus-data/admin.password
```
执行此命令后，终端会打印出管理员密码，请妥善保存。

## 访问Nexus3管理界面

在浏览器中输入`http://${宿主机IP}:18081`（请将`宿主机IP`替换为您的服务器IP地址或`localhost`），进入Nexus3登录界面。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712131753289.png)

输入用户名为`admin`，密码为您上一步获取到的密码，然后点击“Sign in”登录。登录后会提示您修改admin密码，建议按照提示进行操作。

## 配置Nexus3仓库

Nexus3支持多种仓库类型，我们将创建以下四种常用Maven仓库：
*   **lbs-central (Proxy)**：代理中央Maven仓库和其他公共仓库，如阿里云Maven。
*   **lbs-releases (Hosted)**：存储公司内部发布的Release版本构件。
*   **lbs-snapshots (Hosted)**：存储公司内部开发的Snapshot版本构件。
*   **lbs-public (Group)**：一个仓库组，将上述三个仓库整合在一起，方便Maven客户端配置。

点击左侧导航栏的“**Settings**”（齿轮图标），然后选择“**Repositories**”进入仓库管理页面。点击“**Create repository**”开始创建。

**4.1. 创建 lbs-central (Proxy) 仓库**

这个仓库将代理Maven中央仓库和阿里Maven仓库，加速依赖下载。
*   选择类型：`maven2 (proxy)`
*   名称：`lbs-central`
*   远程存储地址 (Remote storage)：填写 `https://maven.aliyun.com/repository/central`

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712131818169.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712131917184.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712131956719.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712144503974.png)

点击“**Create repository**”完成创建。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712132817188.png)

**4.2. 创建 lbs-releases (Hosted) 仓库**

这个仓库用于存储项目正式发布（Release）版本的构件。
*   选择类型：`maven2 (hosted)`
*   名称：`lbs-releases`
*   Deployment policy（部署策略）：`Allow deployments of release versions`

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133133581.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133159363.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133048407.png)

点击“**Create repository**”完成创建。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133255905.png)

**4.3. 创建 lbs-snapshots (Hosted) 仓库**

这个仓库用于存储项目开发中（Snapshot）版本的构件。
*   选择类型：`maven2 (hosted)`
*   名称：`lbs-snapshots`
*   Deployment policy（部署策略）：`Allow deployments of snapshot versions`

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133432079.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133455271.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133617090.png)

点击“**Create repository**”完成创建。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133647212.png)

**4.4. 创建 lbs-public (Group) 仓库**

这个仓库组将上述三个仓库（lbs-central, lbs-releases, lbs-snapshots）整合在一起，Maven客户端只需要配置这一个URL即可。
*   选择类型：`maven2 (group)`
*   名称：`lbs-public`
*   在“Group”配置中，将 `lbs-central`、`lbs-releases`、`lbs-snapshots` 移入“Members”列表中。请注意顺序，通常代理仓库在前。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133733954.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133757184.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133908388.png)

点击“**Create repository**”完成创建。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712133956888.png)

至此，Nexus3的仓库配置已完成。

## 配置本地Maven的`settings.xml`

为了让本地Maven项目能够从Nexus3下载依赖并发布构件，我们需要修改Maven的全局配置文件`settings.xml`。该文件通常位于`C:\Users\YourUser\.m2\`（Windows）或`~/.m2/`（Linux/macOS）目录下。

**重要提示：**
*   将`YOUR_PASSWORD`替换为Nexus3 `admin`用户的真实密码。
*   将`http://master:18081`替换为您的Nexus3实际访问地址，例如`http://192.168.X.X:18081`。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">

    <!-- 本地Maven仓库路径：所有下载的依赖都会存储在这里 -->
    <localRepository>C:\Users\redmi\.m2\repository</localRepository>

    <!-- 是否是交互模式（通常保持true） -->
    <interactiveMode>true</interactiveMode>
    <!-- 是否离线模式（设置为false以允许访问网络仓库） -->
    <offline>false</offline>
    <!-- 插件组配置（通常不需直接配置） -->
    <pluginGroups/>
    <!-- 代理服务器配置（如果您的网络需要通过HTTP代理才能访问外部网络，可以在这里配置） -->
    <proxies/>

    <!-- 服务器认证信息：用于配置发布到私服的凭据。这里的ID必须与pom.xml中distributionManagement的id对应。 -->
    <servers>
        <server>
            <id>lbs-public</id> <!-- 服务器ID，与mirrors中的ID对应，用于拉取依赖认证（如果需要） -->
            <username>admin</username>
            <password>YOUR_PASSWORD</password>
        </server>
        <server>
            <id>lbs-releases</id> <!-- 发布到lbs-releases仓库的认证ID -->
            <username>admin</username>
            <password>YOUR_PASSWORD</password>
        </server>
        <server>
            <id>lbs-snapshots</id> <!-- 发布到lbs-snapshots仓库的认证ID -->
            <username>admin</username>
            <password>YOUR_PASSWORD</password>
        </server>
    </servers>

    <!-- 镜像配置：用于 Maven 从何处下载依赖。Maven会优先从镜像下载，如果镜像中找不到再从原始仓库下载。 -->
    <mirrors>
        <mirror>
            <id>lbs-public</id> <!-- 镜像ID，与servers中的ID对应 -->
            <mirrorOf>external:*</mirrorOf> <!-- 匹配所有外部仓库，意味着Maven的所有下载请求都会先去lbs-public私服 -->
            <name>Nexus Public Repository</name>
            <url>http://master:18081/repository/lbs-public/</url> <!-- 私服公共库组的地址 -->
        </mirror>
    </mirrors>

    <!-- 配置文件组：用于激活特定环境或工具链。这里配置了JDK 1.8的编译参数。 -->
    <profiles>
        <profile>
            <id>jdk-1.8</id> <!-- Profile ID -->
            <activation>
                <activeByDefault>true</activeByDefault> <!-- 默认激活此Profile -->
                <jdk>1.8</jdk> <!-- 当JDK版本为1.8时激活 -->
            </activation>
            <properties>
                <!-- 编译Java源代码和目标字节码版本 -->
                <maven.compiler.source>1.8</maven.compiler.source>
                <maven.compiler.target>1.8</maven.compiler.target>
                <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
            </properties>
        </profile>
    </profiles>

    <!-- 激活的Profile列表：这里可以手动激活profile，但如果activation中配置了activeByDefault，则无需在此处配置。 -->
    <activeProfiles/>

</settings>
```

## 配置项目Maven的`pom.xml`

在需要发布到私服的Maven项目的`pom.xml`文件中，添加`<distributionManagement>`标签，指定Release和Snapshot版本分别发布到哪个Nexus仓库。

**重要提示：**
*   `id`必须与`settings.xml`中`<servers>`下定义的`<server>`的`id`完全一致。
*   `url`必须是对应的Nexus仓库的实际URL。

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 项目基本信息 -->
    <groupId>com.liboshuai.demo</groupId> <!-- 组织或公司ID -->
    <artifactId>lbs-demo</artifactId> <!-- 项目ID -->
    <version>0.1.0-SNAPSHOT</version> <!-- 项目版本，SNAPSHOT表示开发中版本 -->
    <packaging>pom</packaging> <!-- 项目打包类型，pom表示聚合项目 -->

    <name>lbs-demo</name> <!-- 项目名称 -->
    <!-- 聚合模块列表（多模块项目）：如果这是一个多模块的父pom，则列出子模块 -->
    <modules>
        <module>maven-multi-env</module>
        <module>springboot-redis</module>
        <module>springboot-kafka</module>
        <module>flink-wordcount</module>
        <module>springboot-mongodb</module>
        <module>springboot-sharding</module>
        <module>springboot-mybatisplus</module>
        <module>springboot-mybatisplus-dynamic</module>
    </modules>

    <!-- 项目属性配置 -->
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding> <!-- 项目编码 -->
    </properties>

    <!-- 发布管理：定义项目发布到远程仓库的配置。这是将构件上传到Nexus私服的关键配置。 -->
    <distributionManagement>
        <repository>
            <id>lbs-releases</id> <!-- 发布版库的ID，需与settings.xml中server ID对应，用于Release版本 -->
            <name>Nexus Release Repository</name>
            <url>http://master:18081/repository/lbs-releases/</url> <!-- 发布版库地址 -->
        </repository>
        <snapshotRepository>
            <id>lbs-snapshots</id> <!-- 快照版库的ID，需与settings.xml中server ID对应，用于Snapshot版本 -->
            <name>Nexus Snapshot Repository</name>
            <url>http://master:18081/repository/lbs-snapshots/</url> <!-- 快照版库地址 -->
        </snapshotRepository>
    </distributionManagement>
</project>
```

## 验证测试Nexus3配置

完成Maven配置后，我们可以尝试部署一个项目到Nexus3来验证配置是否成功。

**7.1. 执行Maven部署命令**

打开您的IDE（如IntelliJ IDEA），在终端或Maven工具窗口中执行以下命令。`deploy`命令会将项目构件发布到`distributionManagement`中配置的私服仓库。

```shell
mvn clean deploy -Dmaven.test.skip=true
```
`-Dmaven.test.skip=true` 参数用于跳过单元测试，加快部署过程。

**7.2. 查看Maven构建日志**

观察IDE的Maven构建日志。如果部署成功，您会看到类似“BUILD SUCCESS”的提示，并且日志中会显示上传构件到Nexus3的URL信息。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712134932507.png)

**7.3. 登录Nexus3验证仓库**

登录Nexus3管理界面，点击左侧导航栏的“**Browse**”，然后选择您发布到的仓库（例如，如果您发布的是Snapshot版本，则选择`lbs-snapshots`）。您应该能看到刚刚部署的构件已成功出现在列表中。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250712135854841.png)

恭喜您！至此，您已成功搭建并配置了企业级的Maven私服Nexus3，并完成了Maven客户端的配置和构件的发布验证。现在，您的团队可以更高效地管理和共享内部构件了。