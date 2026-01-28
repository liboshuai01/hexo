---
title: CompletableFuture中 join() 与 get() 的区别
abbrlink: faf07427
date: 2025-08-04 07:19:41
tags:
  - java
categories:
  - java
toc: true
---


在 Java 异步编程的世界里，`CompletableFuture` 是一个无法绕开的强大工具。当我们启动一个异步任务后，最终总要获取它的结果。这时，`join()` 和 `get()` 两个方法就登场了。它们都能阻塞当前线程直到异步任务完成并返回结果，但它们之间存在一个关键且重要的区别，这个区别直接影响了代码的简洁性和使用场景。

一言以蔽之：**`get()` 抛出受检异常，`join()` 抛出非受检异常。**

让我们深入探讨这个核心差异。

## 1. 核心区别：异常处理机制

### `get()` 方法：传统的异常处理

`get()` 方法继承自 `java.util.concurrent.Future` 接口，它的设计遵循了 Java 传统的异常处理模型，即抛出**受检异常（Checked Exceptions）**。这意味着当你调用 `get()` 时，编译器会强制你处理两种可能发生的异常：

1.  **`ExecutionException`**: 如果异步任务在执行过程中内部抛出了异常，这个异常会被包装在 `ExecutionException` 中抛出。你需要通过 `e.getCause()` 来获取原始的异常信息。
2.  **`InterruptedException`**: 如果等待结果的线程在阻塞期间被其他线程中断 (`interrupt()`)，`get()` 会抛出 `InterruptedException`。

**代码示例：**

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

public class GetExample {
    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            // 模拟一个可能出错的计算
            if (true) { // 为了演示，我们让它总是出错
                throw new RuntimeException("计算过程中发生严重错误！");
            }
            return "一切正常";
        });

        System.out.println("主线程：开始等待异步结果...");

        try {
            // 调用 get() 必须用 try-catch 包裹
            String result = future.get();
            System.out.println("成功获取结果: " + result);
        } catch (InterruptedException e) {
            System.err.println("主线程在等待时被中断了：" + e.getMessage());
            // 重新设置中断状态，是一种好的编程习惯
            Thread.currentThread().interrupt();
        } catch (ExecutionException e) {
            // 异步任务内部的异常被包装在这里
            System.err.println("异步任务执行失败：" + e.getCause().getMessage());
        }

        System.out.println("主线程：执行完毕。");
    }
}
```

**输出：**
```
主线程：开始等待异步结果...
异步任务执行失败：计算过程中发生严重错误！
主线程：执行完毕。
```

如你所见，`try-catch` 块是不可避免的，这在某些情况下会让代码显得有些冗长。

### `join()` 方法：更现代的函数式友好选择

`join()` 是 `CompletableFuture` 类自己独有的方法。它的设计目标之一就是为了更好地融入 Java 8 的函数式编程范式，比如 Lambda 表达式和 Stream API。为了实现这一点，`join()` 选择了抛出**非受检异常（Unchecked Exception）**。

当异步任务出错时，`join()` 会抛出 `CompletionException`，这是一个运行时异常（`RuntimeException` 的子类）。这意味着你**不需要**强制性地编写 `try-catch` 块。

**代码示例：**

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionException;

public class JoinExample {
    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            // 同样模拟一个出错的计算
            throw new RuntimeException("计算过程中发生严重错误！");
        });

        System.out.println("主线程：开始等待异步结果...");

        try {
            // join() 不需要强制的 try-catch，但为了程序健壮性，通常建议捕获
            String result = future.join();
            System.out.println("成功获取结果: " + result);
        } catch (CompletionException e) {
            // 异步任务的异常同样被包装了
            System.err.println("异步任务执行失败：" + e.getCause().getMessage());
        }

        System.out.println("主线程：执行完毕。");
    }
}
```

**输出：**
```
主线程：开始等待异步结果...
异步任务执行失败：计算过程中发生严重错误！
主线程：执行完毕。
```
虽然结果看起来一样，但 `join()` 的调用在语法上更灵活。这种灵活性在 Stream API 中体现得淋漓尽致。

## 2. 场景对决：`join()` 在 Stream API 中的优势

假设我们有一组 `CompletableFuture`，想要等待所有任务完成并收集它们的结果。

**使用 `join()`（推荐方式）**

代码极其简洁、流畅，充满了函数式编程的美感。

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

public class StreamWithJoin {
    public static void main(String[] args) {
        List<CompletableFuture<String>> futures = List.of(
                CompletableFuture.supplyAsync(() -> "任务A的结果"),
                CompletableFuture.supplyAsync(() -> "任务B的结果"),
                CompletableFuture.supplyAsync(() -> "任务C的结果")
        );

        // 使用 join()，可以通过方法引用，代码非常干净
        List<String> results = futures.stream()
                                      .map(CompletableFuture::join)
                                      .collect(Collectors.toList());

        System.out.println("所有任务结果: " + results);
    }
}
```

**使用 `get()`（繁琐方式）**

由于 `get()` 抛出受检异常，而 Lambda 表达式体默认不能抛出受检异常，这迫使我们编写一个笨拙的 `try-catch` 块，或者定义一个包装方法来转换异常。

```java
// ...
List<String> results = futures.stream()
    .map(future -> {
        try {
            // get() 迫使你在这里处理异常，代码变得臃肿
            return future.get();
        } catch (InterruptedException | ExecutionException e) {
            // 在 Lambda 中处理受检异常非常不便
            // 通常只能抛出一个非受检异常来中断流操作
            throw new RuntimeException(e);
        }
    })
    .collect(Collectors.toList());
// ...
```
两种写法的对比，高下立判。`join()` 显然是为这种现代 Java 编程风格而生的。

## 总结

| 特性                 | `get()`                            | `join()`                               |
| -------------------- | ---------------------------------- | -------------------------------------- |
| **定义来源**         | `java.util.concurrent.Future` 接口 | `java.util.concurrent.CompletableFuture` 类 |
| **异常类型**         | 受检异常 (Checked Exception)         | 非受检异常 (Unchecked Exception)         |
| **主要抛出异常**     | `ExecutionException`, `InterruptedException` | `CompletionException`                  |
| **是否强制`try-catch`** | 是                                 | 否（但推荐捕获）                       |
| **Stream/Lambda 友好** | 否，代码会变得繁琐                 | 是，代码简洁流畅                       |

### 我们的建议

*   **优先使用 `join()`**：在绝大多数直接使用 `CompletableFuture` 的场景中，`join()` 都是更好的选择。它让你的代码，尤其是在结合 Stream API 时，更加简洁和易读。
*   **何时使用 `get()`**：当你需要与一些只接受 `Future` 接口的旧代码库或第三方库交互时，你只能使用 `get()`。此外，如果你有非常明确的需求去分别处理 `InterruptedException`，`get()` 也提供了这种可能。

总而言之，`join()` 可以看作是 `CompletableFuture` 为我们提供的更现代化、更便捷的 `get()` 版本，它更好地拥抱了 Java 8 之后函数式编程的浪潮。