---
title: Pekko快速入门：基于`tell`的异步消息通信
abbrlink: 28c290ea
date: 2025-08-04 03:40:59
tags:
  - 大数据
categories:
  - 大数据
toc: true
---


## 前言

欢迎来到 Pekko 的世界！Pekko 是 Akka 的一个社区驱动的分支，它继承了 Akka 强大的 Actor 模型，为构建高并发、分布式和弹性系统提供了坚实的基础。在 Actor 模型中，所有通信都是通过消息传递完成的。

本篇博文将作为 Pekko 入门系列的第一篇，重点介绍最核心、最常用的消息传递模式：`tell`。我们将通过构建一个简单的客户端-服务端（Client-Server）应用，一步步向您展示如何使用 `tell` 方法进行异步、非阻塞的“发射后不管”（Fire-and-Forget）通信。

<!-- more -->

> 源码：[github](https://github.com/liboshuai01/lbs-demo/tree/master/flink-rpc/base-pekko-tell), [gitee](https://gitee.com/liboshuai01/lbs-demo/tree/master/flink-rpc/base-pekko-tell)

## 项目准备

在开始编码之前，我们需要在 Maven 项目中添加 Pekko 相关的依赖。

**`pom.xml`**

我们主要需要以下几个核心依赖：

* `pekko-actor`: Pekko 的核心 Actor 模块。
* `pekko-remote`: 用于实现 Actor 之间的远程通信。
* `pekko-serialization-jackson`: 一个高效且安全的序列化模块，我们将用它替代 Java 原生序列化。
* `pekko-slf4j`: 用于集成日志框架。

<!-- end list -->

```xml
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <pekko.version>1.0.1</pekko.version>
    <scala.binary.version>2.12</scala.binary.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.apache.pekko</groupId>
        <artifactId>pekko-actor_${scala.binary.version}</artifactId>
        <version>${pekko.version}</version>
    </dependency>

    <dependency>
        <groupId>org.apache.pekko</groupId>
        <artifactId>pekko-remote_${scala.binary.version}</artifactId>
        <version>${pekko.version}</version>
    </dependency>

    <dependency>
        <groupId>org.apache.pekko</groupId>
        <artifactId>pekko-slf4j_${scala.binary.version}</artifactId>
        <version>${pekko.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.pekko</groupId>
        <artifactId>pekko-serialization-jackson_${scala.binary.version}</artifactId>
        <version>${pekko.version}</version>
    </dependency>

    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-simple</artifactId>
        <version>1.7.32</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.30</version>
    </dependency>
</dependencies>
```

## 核心概念：`tell` (或 `!`)

在 Pekko 中，向一个 Actor 发送消息的基本方法是 `tell`。它的方法签名通常是 `actorRef.tell(message, sender)`。

* **`message`**: 你想要发送的任何对象。
* **`sender`**: 发送消息的 Actor 的引用 (`ActorRef`)。接收方可以使用这个引用来回复消息。如果不需要回复，可以传入 `ActorRef.noSender()`。

`tell` 是一种完全异步且非阻塞的操作。当你调用 `tell` 时，消息会被放入目标 Actor 的邮箱（Mailbox），然后你的代码会立即继续执行，无需等待消息被处理。这正是 Actor 模型“发射后不管”（Fire-and-Forget）的精髓所在。

## 代码实战

现在，让我们通过代码来深入理解 `tell` 的工作流程。

### 1\. 定义消息契约与序列化

在分布式系统中，消息需要在网络间传输，因此必须进行序列化。为了安全和高效，我们不直接使用 Java 的原生序列化，而是定义一个标记接口，并配置 Pekko 使用 `jackson-cbor` 进行序列化。

**标记接口 `CborSerializable.java`**

这个接口本身没有任何方法，只用作一个标记，告诉 Pekko 哪些类是允许被序列化的。

```java
package com.liboshuai.demo.common;

import java.io.Serializable;

public interface CborSerializable extends Serializable {
}
```

**请求与响应消息**

我们定义 `RequestData` 和 `ResponseData` 两个类作为我们应用中的消息，它们都实现了 `CborSerializable` 接口。

* **`RequestData.java`**: 客户端发送给服务端。

<!-- end list -->

```java
package com.liboshuai.demo.common;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class RequestData implements CborSerializable{
    private String data;
}
```

* **`ResponseData.java`**: 服务端回复给客户端。

<!-- end list -->

```java
package com.liboshuai.demo.common;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class ResponseData implements CborSerializable {
    private String response;
}
```

### 2\. 配置 ActorSystem

我们需要配置 Pekko 以启用远程通信和我们自定义的序列化规则。这些配置都写在 `resources` 目录下的 `application.conf` 文件中。

**`application.conf`**

```hocon
pekko {
  actor {
    provider = remote # 启用远程 Actor Provider

    serialization-bindings {
      # 将我们的标记接口绑定到 jackson-cbor 序列化器
      "com.liboshuai.demo.common.CborSerializable" = jackson-cbor
    }
  }
  remote {
    artery {
      enabled = on # 启用 Artery TCP 传输协议
      transport = tcp
      canonical.hostname = "127.0.0.1"
      canonical.port = 25520 # 服务端 ActorSystem 的端口
    }
  }
}
```

### 3\. 服务端实现

服务端的职责是接收 `RequestData` 消息，然后回复一个 `ResponseData` 消息。

**`ServerActor.java`**

这个 Actor 的逻辑非常简单：

1.  使用 `receiveBuilder()` 定义消息处理逻辑。
2.  `match(RequestData.class, ...)`: 当接收到的消息是 `RequestData` 类型时，执行相应的逻辑。
3.  在处理逻辑中，我们打印收到的消息。
4.  创建一个 `ResponseData` 作为回复。
5.  使用 `getSender().tell(response, getSelf())` 将回复消息发送给原始发送者。`getSender()` 在这里获取的就是客户端 Actor 的引用。

<!-- end list -->

```java
package com.liboshuai.demo.server;

import com.liboshuai.demo.common.RequestData;
import com.liboshuai.demo.common.ResponseData;
import lombok.extern.slf4j.Slf4j;
import org.apache.pekko.actor.AbstractActor;
import org.apache.pekko.actor.Props;

@Slf4j
public class ServerActor extends AbstractActor {

    public static Props props() {
        return Props.create(ServerActor.class, ServerActor::new);
    }

    @Override
    public Receive createReceive() {
        return receiveBuilder()
                .match(RequestData.class, requestData -> {
                    log.info("收到客户端的消息，内容为：[{}]", requestData.getData());

                    String replyMessage = String.format("消息 [%s] 已收到!", requestData.getData());
                    ResponseData response = new ResponseData(replyMessage);

                    log.info("回应客户端的信息，内容为：[{}]", replyMessage);
                    getSender().tell(response, getSelf());
                })
                .matchAny(o -> log.warn("收到未知类型的消息: {}", o.getClass().getName()))
                .build();
    }
}
```

**`ServerMain.java`**

这是服务端的启动入口。它会创建 `ActorSystem` 和 `ServerActor`。

```java
package com.liboshuai.demo.server;

import com.typesafe.config.ConfigFactory;
import lombok.extern.slf4j.Slf4j;
import org.apache.pekko.actor.ActorRef;
import org.apache.pekko.actor.ActorSystem;

import java.io.IOException;

@Slf4j
public class ServerMain {
    public static void main(String[] args) {
        // 加载 application.conf 配置创建 ActorSystem
        ActorSystem serverSystem = ActorSystem.create("serverSystem", ConfigFactory.load());

        // 创建 ServerActor
        ActorRef serverActor = serverSystem.actorOf(ServerActor.props(), "serverActor");

        log.info("服务端 actor 已经创建完毕，完整路径为: {}", serverActor.path());

        log.info(">>> 按回车键退出 <<<");
        try {
            System.in.read();
        } catch (IOException e) {
            log.error("等待输入时发生错误。", e);
        } finally {
            log.info("服务端正在关闭...");
            serverSystem.terminate();
        }
    }
}
```

### 4\. 客户端实现

客户端负责从控制台读取用户输入，将其封装成 `RequestData` 发送给服务端，并接收服务端的 `ResponseData` 回复。

**`ClientActor.java`**

这个 Actor 的职责是与服务端进行交互：

1.  在构造函数中，通过 `getContext().actorSelection(serverPath)` 获取一个指向远程 `ServerActor` 的 `ActorSelection`。这是一个“指针”，可以用来向远程 Actor 发送消息。
2.  它能处理两种消息：
    * `String`: 来自控制台的输入。它会将这个字符串包装成 `RequestData`，然后通过 `serverActorSelection.tell(...)` 发送给服务端。注意第二个参数是 `getSelf()`，这会将当前 Actor（`ClientActor`）的引用作为发送者传递过去，以便服务端可以回复。
    * `ResponseData`: 从服务端接收到的回复。它会简单地将回复内容打印出来。

<!-- end list -->

```java
package com.liboshuai.demo.client;

import com.liboshuai.demo.common.RequestData;
import com.liboshuai.demo.common.ResponseData;
import lombok.extern.slf4j.Slf4j;
import org.apache.pekko.actor.AbstractActor;
import org.apache.pekko.actor.ActorSelection;
import org.apache.pekko.actor.Props;

@Slf4j
public class ClientActor extends AbstractActor {

    private final ActorSelection serverActorSelection;

    public ClientActor(String serverPath) {
        this.serverActorSelection = getContext().actorSelection(serverPath);
    }

    public static Props props(String serverPath) {
        return Props.create(ClientActor.class, () -> new ClientActor(serverPath));
    }

    @Override
    public Receive createReceive() {
        return receiveBuilder()
                .match(String.class, message -> {
                    // 来自 ClientMain 控制台输入的消息
                    log.info("向服务端发送消息: [{}]", message);
                    serverActorSelection.tell(new RequestData(message), getSelf());
                })
                .match(ResponseData.class, response -> {
                    // 从服务端接收到的消息
                    log.info("接收到服务端响应: [{}]", response.getResponse());
                })
                .matchAny(o -> log.warn("客户端接收到未知消息: {}", o.getClass().getName()))
                .build();
    }
}
```

**`ClientMain.java`**

这是客户端的启动入口。

1.  它首先通过代码的方式修改了端口号为 `25521`，避免和运行在 `25520` 的服务端冲突，然后加载 `application.conf` 的其余配置。
2.  定义了服务端的 Actor 路径：`pekko://serverSystem@127.0.0.1:25520/user/serverActor`。
3.  创建 `ClientActor`，并将服务端路径传递给它。
4.  启动一个循环，不断读取控制台输入，并通过 `clientActor.tell(...)` 将每一行输入发送给 `ClientActor` 处理。

<!-- end list -->

```java
package com.liboshuai.demo.client;

import com.typesafe.config.Config;
import com.typesafe.config.ConfigFactory;
import lombok.extern.slf4j.Slf4j;
import org.apache.pekko.actor.ActorRef;
import org.apache.pekko.actor.ActorSystem;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

@Slf4j
public class ClientMain {
    public static void main(String[] args) {
        Config config = ConfigFactory.parseString(
                "pekko.remote.artery.canonical.port = 25521" // 客户端使用不同端口
        ).withFallback(ConfigFactory.load());
        ActorSystem clientSystem = ActorSystem.create("clientSystem", config);

        String serverPath = "pekko://serverSystem@127.0.0.1:25520/user/serverActor";
        ActorRef clientActor = clientSystem.actorOf(ClientActor.props(serverPath), "clientActor");

        log.info("客户端 actor 已经创建完毕，完整路径为: {}", clientActor.path());
        log.info(">>> 请在控制台输入要发送的消息, 输入 'exit' 退出 <<<");

        try (BufferedReader reader = new BufferedReader(new InputStreamReader(System.in))) {
            String line;
            while ((line = reader.readLine()) != null) {
                if ("exit".equalsIgnoreCase(line.trim())) {
                    break;
                }
                // 将控制台输入发送给 ClientActor
                clientActor.tell(line, ActorRef.noSender());
            }
        } catch (IOException e) {
            log.error("等待输入时发生错误。", e);
        } finally {
            log.info("客户端正在关闭...");
            clientSystem.terminate();
        }
    }
}
```

## 运行与演示

1.  **启动服务端**：运行 `ServerMain.java` 的 `main` 方法。你将看到服务端已启动的日志。

    **服务端控制台输出:**

    ```
    15:30:10.517 [main] INFO com.liboshuai.demo.server.ServerMain - 服务端 actor 已经创建完毕，完整路径为: pekko://serverSystem@127.0.0.1:25520/user/serverActor
    15:30:10.519 [main] INFO com.liboshuai.demo.server.ServerMain - >>> 按回车键退出 <<<
    ```

2.  **启动客户端**：运行 `ClientMain.java` 的 `main` 方法。

    **客户端控制台输出:**

    ```
    15:30:45.112 [main] INFO com.liboshuai.demo.client.ClientMain - 客户端 actor 已经创建完毕，完整路径为: pekko://clientSystem@127.0.0.1:25521/user/clientActor
    15:30:45.113 [main] INFO com.liboshuai.demo.client.ClientMain - >>> 请在控制台输入要发送的消息, 输入 'exit' 退出 <<<
    ```

3.  **发送消息**：在客户端的控制台输入 `Hello Pekko` 并按回车。

    **客户端日志:**

    ```
    // ClientActor 收到来自控制台的消息，并将其发送给服务端
    15:31:12.345 [clientSystem-pekko.actor.default-dispatcher-3] INFO com.liboshuai.demo.client.ClientActor - 向服务端发送消息: [Hello Pekko]

    // 稍后，ClientActor 收到服务端的回复
    15:31:12.456 [clientSystem-pekko.actor.default-dispatcher-3] INFO com.liboshuai.demo.client.ClientActor - 接收到服务端响应: [消息 [Hello Pekko] 已收到!]
    ```

    **服务端日志:**

    ```
    // ServerActor 收到来自客户端的消息
    15:31:12.400 [serverSystem-pekko.actor.default-dispatcher-5] INFO com.liboshuai.demo.server.ServerActor - 收到客户端的消息，内容为：[Hello Pekko]

    // ServerActor 发送回复给客户端
    15:31:12.401 [serverSystem-pekko.actor.default-dispatcher-5] INFO com.liboshuai.demo.server.ServerActor - 回应客户端的信息，内容为：[消息 [Hello Pekko] 已收到!]
    ```

整个过程完美地展示了 `tell` 的异步通信：客户端发送消息后不必等待，可以继续接收新的控制台输入；而服务端的回应则在未来的某个时间点到达，并被客户端 Actor 的另一部分逻辑所处理。

## 总结

通过这个简单的例子，我们学习了 Pekko 中最基础也是最重要的通信方式——`tell`。它体现了 Actor 模型的核心思想：异步、非阻塞的消息驱动。

我们涵盖了从项目配置、消息定义、Actor 实现到远程通信配置的全过程。关键点在于：

* **`tell` 是“发射后不管”的**：发送方不阻塞等待响应。
* **通信是异步的**：消息的发送和接收发生在不同的时间点，由 Pekko 的调度器管理。
* **通过 `getSender()` 回复**：接收方可以通过 `getSender()` 获取发送方的 `ActorRef`，并使用 `tell` 回复消息。

在下一篇博文中，我们将探讨另一种通信模式 `ask`，它适用于需要明确等待并处理返回结果（`Future`）的场景。敬请期待！