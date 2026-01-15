---
title: Pekko快速入门：基于`ask`的异步消息通信
abbrlink: 341ca284
date: 2025-08-04 03:41:50
tags:
  - 大数据
categories:
  - 大数据
toc: true
---


欢迎来到 Pekko 的世界！本教程将通过一个具体的示例，向您展示如何使用 Pekko 构建一个简单的分布式应用程序。我们将创建两个独立的 Java 应用：一个"远程系统"（服务端），它会等待请求；以及一个"客户端"，它会向远程系统发送一个问候请求，并异步地等待回复。

这个过程主要利用了 Pekko 中非常重要的 **Ask 模式 (Ask Pattern)**。

<!-- more -->

> 源码: [github](https://github.com/liboshuai01/lbs-demo/tree/master/flink-rpc/base-pekko-ask), [gitee](https://gitee.com/liboshuai01/lbs-demo/tree/master/flink-rpc/base-pekko-ask)

## 核心概念简介

在深入代码之前，我们先了解几个 Pekko 的核心概念：

1.  **Actor (演员)**: Actor 是 Pekko 的基本计算单元。它是一个对象，封装了状态（State）和行为（Behavior）。Actor 之间通过发送异步消息进行通信，这是它们唯一的通信方式。每个 Actor 都有一个"邮箱"（Mailbox）用来接收消息。
2.  **ActorSystem (演员系统)**: 这是一个重量级的结构，是所有 Actor 的家。它管理着 Actor 的生命周期、调度、配置和线程池等资源。一个应用程序通常只有一个 ActorSystem。
3.  **Message (消息)**: Actor 之间传递的数据。消息应该是不可变（Immutable）的对象，以避免并发问题。在我们的示例中，`AskForGreeting` 和 `GreetingReply` 就是消息。
4.  **Pekko Remoting (远程通信)**: 这是 Pekko 的一个模块，它允许位于不同 JVM（甚至不同机器）上的 ActorSystem 中的 Actor 进行透明通信。您向一个远程 Actor 发送消息，就如同向一个本地 Actor 发送消息一样简单。
5.  **Ask Pattern (Ask 模式)**: 当您向一个 Actor 发送消息并期望得到一个回复时，就可以使用 Ask 模式。它与"Tell" (`tell`)模式（即"fire-and-forget"，只管发送不关心回复）相对。Ask 模式会返回一个 `Future` (在 Java 中体现为 `CompletionStage`)，代表未来的某个时刻会有一个响应。

-----

## 第一步：定义通信协议 (消息)

任何通信都需要预先定义好协议。在 Pekko 中，协议就是消息的格式。我们定义了两种消息：

1.  **请求消息**: `AskForGreeting.java`
    这个类代表客户端向服务端发起的请求。它包含一个 `name` 字段，表示希望被问候的人。
    ```java
    // 文件: AskForGreeting.java
    package com.liboshuai.demo;

    import lombok.AllArgsConstructor;
    import lombok.Data;
    import lombok.NoArgsConstructor;

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public class AskForGreeting implements CborSerializable {
        public String name;
    }
    ```
2.  **回复消息**: `GreetingReply.java`
    这个类代表服务端对请求的回复。它包含一个 `message` 字段，即问候语。
    ```java
    // 文件: GreetingReply.java
    package com.liboshuai.demo;

    import lombok.AllArgsConstructor;
    import lombok.Data;
    import lombok.NoArgsConstructor;

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public class GreetingReply implements CborSerializable {
        public String message;
    }
    ```

### 关键点：序列化标记接口 `CborSerializable`

请注意，这两个消息类都实现了一个名为 `CborSerializable` 的接口。

```java
// 文件: CborSerializable.java
package com.liboshuai.demo;

import java.io.Serializable;

/**
 * 一个标记接口，用于所有希望通过 Pekko Jackson CBOR 序列化的类。
 * 这是一种安全最佳实践...
 */
public interface CborSerializable extends Serializable {
    // 这是一个标记接口，所以内部不需要任何方法。
}
```

这是一个**标记接口**。它的作用是告诉 Pekko 的序列化系统：“凡是实现了这个接口的类，都应该使用 `jackson-cbor` 序列化器进行转换”。这是一种安全和推荐的做法，避免了将宽泛的 `java.io.Serializable` 直接绑定到序列化器，从而精确控制哪些类是允许被序列化的。

-----

## 第二步：创建响应请求的 Actor

现在我们来创建服务端的 Actor，它负责接收请求并作出回复。

```java
// 文件: GreeterActorWithReply.java
package com.liboshuai.demo;

import org.apache.pekko.actor.AbstractActor;
import org.apache.pekko.actor.Props;
// ...

public class GreeterActorWithReply extends AbstractActor {

    // ...

    public static Props props() {
        return Props.create(GreeterActorWithReply.class, GreeterActorWithReply::new);
    }

    @Override
    public Receive createReceive() {
        return receiveBuilder()
                .match(AskForGreeting.class, request -> {
                    log.info("从发送者 {} 收到了问候请求：'{}'", getSender(), request.name);

                    String replyMessage = String.format("你好，%s！这是来自 GreeterActor 的回复。", request.name);
                    GreetingReply reply = new GreetingReply(replyMessage);

                    // **核心：将回复消息发送给原始的发送者**
                    getSender().tell(reply, getSelf());
                })
                .matchAny(o -> log.warn("收到了未知类型的消息：{}", o.getClass().getName()))
                .build();
    }
}
```

**代码解析**:

* `extends AbstractActor`: 定义了这是一个 Actor。
* `createReceive()`: 这是 Actor 的核心。它定义了 Actor 如何处理接收到的不同类型的消息。
* `receiveBuilder().match(AskForGreeting.class, ...)`: 这部分是行为定义。它表示：“如果收到的消息是 `AskForGreeting` 类型的，就执行后面的 Lambda 表达式”。
* `getSender()`: 这是一个非常重要的方法，它返回发送当前消息的 Actor 的引用 (`ActorRef`)。
* `getSender().tell(reply, getSelf())`: **这是实现 Ask 模式的关键**。我们将 `GreetingReply` 消息通过 `tell` 方法发送回给了原始的请求者 (`getSender()`)。`getSelf()` 指的是当前 Actor 自身的引用。

-----

## 第三步：配置并启动远程系统 (服务端)

为了让我们的 `GreeterActorWithReply` 能够被网络上的其他系统访问，我们需要配置 Pekko Remoting。

**1. 配置文件: `application.conf`**

这个文件告诉 Pekko 如何运行。

```conf
# 文件: application.conf
pekko {
  actor {
    provider = remote // 1. 启用远程通信模块

    serialization-bindings {
      // 2. 将我们的标记接口绑定到 CBOR 序列化器
      "com.liboshuai.demo.CborSerializable" = jackson-cbor
    }
  }
  remote {
    artery {
      enabled = on
      transport = tcp
      canonical.hostname = "127.0.0.1" // 3. 远程系统的主机名
      canonical.port = 25522           // 4. 远程系统的端口
    }
  }
}
```

**配置解析**:

1.  `provider = remote`: 激活 Pekko 的远程通信能力。
2.  `serialization-bindings`: 正如前面所说，将实现了 `CborSerializable` 接口的类与 `jackson-cbor` 序列化器关联起来。
3.  `canonical.hostname` 和 `canonical.port`: 定义了此 ActorSystem 在网络上的唯一地址。客户端将使用这个地址来找到它。

**2. 启动类: `RemoteSystemMainWithAsk.java`**

这个类负责启动服务端的 ActorSystem 并创建我们的 Greeter Actor。

```java
// 文件: RemoteSystemMainWithAsk.java
public class RemoteSystemMainWithAsk {
    // ...
    public static void main(String[] args) {
        // 使用 application.conf 配置启动远程系统
        final ActorSystem system = ActorSystem.create("RemoteSystem", ConfigFactory.load());
        log.info("远程系统 'ask' 演示已准备就绪。");

        // 创建一个可被远程访问的、能够回复消息的 Actor
        // 我们给它一个名字 "greeterWithReply"
        final ActorRef remoteGreeter = system.actorOf(GreeterActorWithReply.props(), "greeterWithReply");

        log.info("已创建远程问候 Actor 并支持回复。完整路径: {}", remoteGreeter.path());
        // ...
    }
}
```

**代码解析**:

* `ActorSystem.create("RemoteSystem", ...)`: 创建一个名为 "RemoteSystem" 的 ActorSystem。这个名字会成为其地址的一部分。
* `system.actorOf(..., "greeterWithReply")`: 在系统中创建 `GreeterActorWithReply` 的一个实例，并给它命名为 "greeterWithReply"。这个名字同样会成为其地址的一部分。
* `remoteGreeter.path()`: 这会打印出 Actor 的完整路径，格式为 `pekko://ActorSystemName@hostname:port/user/actorName`。在我们的例子中，它将是 `pekko://RemoteSystem@127.0.0.1:25522/user/greeterWithReply`。客户端需要这个路径来定位 Actor。

-----

## 第四步：实现 Ask 客户端

现在我们来构建客户端，它将通过网络发送请求并等待回复。

```java
// 文件: AskDemoClient.java
public class AskDemoClient {
    // ...
    public static void main(String[] args) throws InterruptedException {
        // ...
        final ActorSystem system = ActorSystem.create("ClientSystem", config);

        // **重要**：路径指向我们新创建的 Actor "greeterWithReply"
        final String remotePath = "pekko://RemoteSystem@127.0.0.1:25522/user/greeterWithReply";
        final ActorSelection remoteActor = system.actorSelection(remotePath);

        // 创建请求消息
        AskForGreeting request = new AskForGreeting("Pekko Ask Pattern");
        Duration timeout = Duration.ofSeconds(5);

        // **核心：使用 Patterns.ask 发送消息**
        CompletionStage<Object> responseFuture = Patterns.ask(remoteActor, request, timeout);
        log.info("Ask 请求已发送。正在等待响应...");

        // ... 使用 CountDownLatch 等待异步结果 ...

        // 处理 CompletionStage 的结果
        responseFuture.whenComplete((response, error) -> {
            if (error != null) {
                log.error("Ask 请求失败！", error);
            } else {
                if (response instanceof GreetingReply) {
                    GreetingReply reply = (GreetingReply) response;
                    log.info(">>> 成功收到回复：'{}'", reply.getMessage());
                } else {
                    // ...
                }
            }
            latch.countDown();
        });

        latch.await();
        system.terminate();
    }
}
```

**代码解析**:

* `ActorSystem.create("ClientSystem", ...)`: 客户端也需要自己的 ActorSystem。
* `system.actorSelection(remotePath)`: 这是客户端定位远程 Actor 的方式。它使用我们在服务端看到的完整路径创建了一个 `ActorSelection`。
* `Patterns.ask(remoteActor, request, timeout)`: **这就是 Ask 模式的调用**。
    * `remoteActor`: 目标 Actor。
    * `request`: 要发送的消息。
    * `timeout`: 一个超时期限。如果在这个时间内没有收到回复，`Future` 将会以 `AskTimeoutException` 失败。
    * 它返回一个 `CompletionStage<Object>`，这是一个对未来结果的引用。
* `responseFuture.whenComplete(...)`: 由于 `ask` 是异步的，我们不能立即获得结果。我们注册一个回调，当 `Future` 完成时（无论是成功还是失败），这个回调函数就会被执行。
* `if (response instanceof GreetingReply)`: 在成功的回调中，我们检查收到的 `response` 是否是我们期望的 `GreetingReply` 类型，然后处理它。
* `CountDownLatch`: 在这个示例中，`main` 线程会立即执行完毕，可能导致JVM在异步回调完成前就退出了。`CountDownLatch` 是一种简单的同步工具，它阻塞 `main` 线程 (`latch.await()`) 直到异步回调被执行并调用 `latch.countDown()`。

-----

## 第五步：运行与观察

现在，让我们把所有部分组合起来运行。

1.  **启动服务端**:

    * 运行 `RemoteSystemMainWithAsk.java` 的 `main` 方法。
    * 你会在控制台看到类似以下的输出，表明服务端已准备就绪：
      ```
      [main] INFO com.liboshuai.demo.RemoteSystemMainWithAsk - 远程系统 'ask' 演示已准备就绪。
      [main] INFO com.liboshuai.demo.RemoteSystemMainWithAsk - 已创建远程问候 Actor 并支持回复。完整路径: pekko://RemoteSystem@127.0.0.1:25522/user/greeterWithReply
      [main] INFO com.liboshuai.demo.RemoteSystemMainWithAsk - >>> 按 ENTER 退出 <<<
      ```

2.  **启动客户端**:

    * 在服务端运行的同时，运行 `AskDemoClient.java` 的 `main` 方法。
    * 客户端的控制台会显示：
      ```
      [main] INFO com.liboshuai.demo.AskDemoClient - 客户端系统已启动，用于 'ask' 模式演示。
      [main] INFO com.liboshuai.demo.AskDemoClient - 正在使用 'ask' 模式向 pekko://RemoteSystem@127.0.0.1:25522/user/greeterWithReply 发送消息。
      [main] INFO com.liboshuai.demo.AskDemoClient - Ask 请求已发送。正在等待响应...
      ```

3.  **观察结果**:

    * 几乎在客户端发送消息的同时，**服务端的控制台**会打印出它接收到消息的日志：
      ```
      [RemoteSystem-pekko.remote.internal.Remoting-1] INFO com.liboshuai.demo.GreeterActorWithReply - 从发送者 Actor[pekko://ClientSystem@127.0.0.1:25523/temp/$a] 收到了问候请求：'Pekko Ask Pattern'
      ```
    * 紧接着，**客户端的控制台**会打印出成功接收到回复的日志，然后程序终止：
      ```
      [ClientSystem-pekko.actor.default-dispatcher-3] INFO com.liboshuai.demo.AskDemoClient - >>> 成功收到回复：'你好，Pekko Ask Pattern！这是来自 GreeterActor 的回复。'
      [main] INFO com.liboshuai.demo.AskDemoClient - 客户端系统已终止。
      ```

## 总结

恭喜！您已经成功地使用 Pekko 构建并运行了一个跨 JVM 的请求-响应应用。

通过本教程，您学习了：

* **如何定义 Actor 和消息**，构建通信的基础。
* **如何配置 Pekko Remoting**，让不同的系统可以互相通信。
* **如何启动 ActorSystem 并创建 Actor 实例**。
* **如何使用 `ActorSelection` 定位一个远程 Actor**。
* **如何使用 `Patterns.ask` 实现异步的请求-响应交互**，并处理返回的 `CompletionStage`。

这只是 Pekko 功能的冰山一角。基于这些基础，您可以构建出高度并发、可扩展且具备容错能力的复杂分布式系统。