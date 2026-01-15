---
title: Java CompletableFuture 快速入门指南
abbrlink: bc1559a5
date: 2025-08-04 04:26:33
tags:
  - Java
categories:
  - Java
toc: true
---


## 1. 为什么需要 CompletableFuture？

在 `CompletableFuture` 出现之前，Java 5 引入了 `Future` 接口，用于表示一个异步计算的结果。但 `Future` 的能力非常有限：

*   **无法主动完成**：你无法手动将一个 `Future` 标记为已完成，并设置其结果。它只能被动地等待执行它的线程完成任务。
*   **阻塞式获取结果**：`future.get()` 方法是阻塞的，调用时会暂停当前线程，直到异步任务完成。这违背了异步编程的初衷。
*   **没有回调机制**：你无法在 `Future` 完成时自动触发某个动作（回调函数），只能通过循环调用 `isDone()` 来检查，或者直接阻塞在 `get()` 上。
*   **无法组合**：你很难将多个 `Future` 串联起来，例如当一个 `Future` 完成后，用其结果去执行另一个异步任务。

`CompletableFuture` 扩展了 `Future` 接口，并实现了 `CompletionStage` 接口，彻底解决了以上痛点。它为异步编程提供了一种功能强大、非阻塞、可组合的范式，是现代 Java 异步编程的基石。

**核心思想**：`CompletableFuture` 像一个承诺（Promise），它承诺在未来的某个时刻会有一个结果。你可以基于这个承诺（无论它成功还是失败）来编排一系列后续操作，而无需阻塞等待。

## 2. CompletableFuture 的创建

创建 `CompletableFuture` 通常有两种方式：

1.  **`runAsync(Runnable runnable)` / `runAsync(Runnable runnable, Executor executor)`**
    *   **作用**：执行一个没有返回值的异步任务。
    *   **返回值**：`CompletableFuture<Void>`。
    *   **使用场景**：当你只想异步执行一个操作，不关心其返回值时。例如，异步记录日志、发送通知邮件等。

2.  **`supplyAsync(Supplier<U> supplier)` / `supplyAsync(Supplier<U> supplier, Executor executor)`**
    *   **作用**：执行一个有返回值的异步任务。`Supplier<U>` 是一个函数式接口，代表一个提供类型为 `U` 的结果的函数。
    *   **返回值**：`CompletableFuture<U>`，其中 `U` 是 `supplier` 提供的结果类型。
    *   **使用场景**：这是最常用的创建方式。当你需要异步获取数据（如查询数据库、调用远程API）时使用。

**关于`Executor`参数**：
*   **不带`Executor`的版本**：默认使用 `ForkJoinPool.commonPool()` 作为其线程池。这个线程池是全局共享的，适用于CPU密集型任务。如果用于IO密集型任务（如网络请求、文件读写），可能会因为线程阻塞而耗尽 `commonPool` 的线程，影响整个应用的性能。
*   **带`Executor`的版本**：允许你指定自定义的线程池。**强烈推荐**为IO密集型任务创建专用的线程池，以避免上述问题。

## 3. 核心API详解：回调与链式处理

这是 `CompletableFuture` 最强大的部分，用于处理异步任务的结果。

### 3.1 结果处理（消费或转换）

当一个 `CompletableFuture` 完成后，你可以使用以下方法来处理其结果。这些方法都有一个同步版本（如 `thenApply`）和一个异步版本（如 `thenApplyAsync`）。

*   **同步版本 (e.g., `thenApply`)**：后续任务可能由完成上一个任务的线程执行，也可能由调用 `thenApply` 的主线程执行。它不会切换线程池。
*   **异步版本 (e.g., `thenApplyAsync`)**：后续任务会被提交到线程池（默认是 `ForkJoinPool.commonPool()`，或指定的 `Executor`）中执行，确保了任务的异步性。

| 方法 | 作用 | 入参 | 返回值 | 使用场景 |
| :--- | :--- | :--- | :--- | :--- |
| `thenApply(Function<? super T,? extends U> fn)` | **转换**：接收上一步的结果，并返回一个**新**的结果。同步执行。 | `Function` (T -> U) | `CompletableFuture<U>` | 将一个异步结果（如用户ID）转换为另一种形式（如用户信息对象）。这是最常用的链式操作。 |
| `thenAccept(Consumer<? super T> action)` | **消费**：接收上一步的结果，但**没有**返回值。同步执行。 | `Consumer` (T -> Void) | `CompletableFuture<Void>` | 对异步结果进行最终处理，如打印到控制台、存入数据库，并且不需要返回任何东西。 |
| `thenRun(Runnable action)` | **运行**：不关心上一步的结果，只要上一步完成了，就执行一个 `Runnable`。同步执行。 | `Runnable` (Void -> Void) | `CompletableFuture<Void>` | 在某个异步操作（无论结果是什么）完成后，触发一个没有入参和返回值的动作。 |

**示例理解**：
假设有一个 `CompletableFuture<Long> userIdFuture`。
*   `userIdFuture.thenApply(id -> "用户" + id)` -> 返回 `CompletableFuture<String>`
*   `userIdFuture.thenAccept(id -> System.out.println("成功获取用户ID：" + id))` -> 返回 `CompletableFuture<Void>`
*   `userIdFuture.thenRun(() -> System.out.println("获取ID的任务已完成"))` -> 返回 `CompletableFuture<Void>`

### 3.2 任务编排（串行化）

当一个异步任务的执行依赖于另一个异步任务的结果时，你需要 `thenCompose`。

*   **`thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)`**
    *   **作用**：这是处理**依赖**异步任务的核心。它接收上一步的结果，然后返回**另一个** `CompletionStage`（通常是 `CompletableFuture`）。最终的结果由这个新的 `CompletableFuture` 决定。
    *   **与 `thenApply` 的关键区别**：
        *   `thenApply` 的函数体返回一个**普通值** (`U`)，`thenApply` 自身会把这个值包装成 `CompletableFuture<U>`。
        *   `thenCompose` 的函数体直接返回一个**新的 `CompletableFuture<U>`**。它会将两个 `CompletableFuture` “铺平”（flat），避免出现 `CompletableFuture<CompletableFuture<User>>` 这样的嵌套结构。
    *   **使用场景**：当你有多个需要依次执行的异步步骤时。例如：
        1.  异步获取订单ID。
        2.  使用订单ID，异步获取订单详情。
        3.  使用订单详情，异步获取用户信息。

    ```java
    // 伪代码演示
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "订单ID123") // 步骤1
        .thenCompose(orderId -> supplyAsync(() -> "订单详情 for " + orderId)) // 步骤2
        .thenCompose(orderDetails -> supplyAsync(() -> "用户信息 for " + orderDetails)); // 步骤3
    ```

## 4. 组合多个CompletableFuture

当需要并行执行多个独立的异步任务，并在它们都完成后做一些事情时，使用组合API。

### 4.1 AND 关系（等待两个都完成）

*   **`thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)`**
    *   **作用**：将**两个独立**的 `CompletableFuture` 结果合并。当**两个**任务都成功完成后，将它们的结果作为 `BiFunction` 的参数，并返回一个新的结果。
    *   **使用场景**：需要并行获取两个不相关的数据，然后将它们组合起来。例如，异步获取用户信息和用户权限，然后合并成一个用户视图对象。

*   **`thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)`**
    *   **作用**：与 `thenCombine` 类似，但用于消费结果，不返回值。

*   **`runAfterBoth(CompletionStage<?> other, Runnable action)`**
    *   **作用**：当两个任务都完成后，执行一个 `Runnable`，不关心它们的结果。

### 4.2 OR 关系（等待任意一个完成）

*   **`applyToEither(CompletionStage<? extends T> other, Function<? super T, U> fn)`**
    *   **作用**：两个 `CompletableFuture`，**任意一个**先完成，就将其结果作为 `Function` 的参数，并返回一个新的结果。另一个未完成的任务将被忽略。
    *   **使用场景**：从多个数据源（如主/备数据库，不同地区的缓存）获取相同的数据，哪个快就用哪个。

*   **`acceptEither(...)` 和 `runAfterEither(...)`**：与 `applyToEither` 类似，分别用于消费和运行。

### 4.3 等待所有/任意任务（静态方法）

*   **`allOf(CompletableFuture<?>... cfs)`**
    *   **作用**：等待**所有**给定的 `CompletableFuture` 都完成。
    *   **返回值**：`CompletableFuture<Void>`。**注意**：这个 `allOf` 本身不提供所有结果的聚合。它只是一个完成信号。
    *   **如何获取所有结果？** 你需要先调用 `allOf(...).join()`（或 `.get()`），然后再遍历原始的 `CompletableFuture` 列表，分别调用 `.join()` 来获取它们各自的结果。因为此时可以确定它们都已完成，所以 `.join()` 不会阻塞。
    *   **使用场景**：需要并行执行大量独立的异步任务，并等待它们全部结束后再进行下一步。例如，批量发送邮件。

*   **`anyOf(CompletableFuture<?>... cfs)`**
    *   **作用**：等待**任意一个**给定的 `CompletableFuture` 完成。
    *   **返回值**：`CompletableFuture<Object>`。返回的是第一个完成的任务的结果。
    *   **使用场景**：与 `applyToEither` 类似，但可以处理两个以上的任务。例如，向多个服务发起同一个请求，谁先响应就用谁的数据。

## 5. 异常处理

异步代码的异常处理至关重要。`CompletableFuture` 提供了一套优雅的机制。

*   **`exceptionally(Function<Throwable, ? extends T> fn)`**
    *   **作用**：类似于 `try-catch` 中的 `catch` 块。如果异步链中任何一步出现异常，`exceptionally` 会捕获它，并提供一个**备用/默认值**作为后续链的结果。
    *   **返回值**：`CompletableFuture<T>`。它可以让一个失败的流程“恢复正常”。

*   **`whenComplete(BiConsumer<? super T, ? super Throwable> action)`**
    *   **作用**：类似于 `try-catch-finally` 中的 `finally` 块（但更像一个观察者）。无论成功还是失败，它都会被执行。`BiConsumer` 接收两个参数：`result` 和 `exception`，其中一个必然为 `null`。
    *   **关键**：`whenComplete` **不能改变**计算的结果。它只是一个“窥视”的机会，通常用于记录日志、资源清理等副作用操作。如果上游是成功的，它下游仍然是成功的；如果上游是失败的，它下游仍然是失败的。

*   **`handle(BiFunction<? super T, Throwable, ? extends U> fn)`**
    *   **作用**：`whenComplete` 和 `exceptionally` 的结合体。它既能处理正常结果，也能处理异常，并且**可以改变最终的结果**。
    *   `BiFunction` 接收 `result` 和 `exception` 两个参数。你可以根据这两个参数返回一个全新的结果。
    *   **使用场景**：需要根据成功或失败返回不同类型或状态的结果时，`handle` 是最灵活的选择。

**选择建议**：
*   如果想在出错时提供一个默认值，让流程继续下去 -> `exceptionally`。
*   如果只想记录日志或做清理，不影响结果 -> `whenComplete`。
*   如果需要根据成功或失败，对结果进行复杂的转换和处理 -> `handle`。

## 6. 手动完成 Future

*   **`complete(T value)`**：如果 Future 尚未完成，则将其结果设置为 `value`。如果已完成，则调用无效。
*   **`completeExceptionally(Throwable ex)`**：如果 Future 尚未完成，则使其以异常 `ex` 结束。

**使用场景**：将一个非异步的、基于回调的API（例如某些老旧的库）适配到 `CompletableFuture` 模型中。你可以先创建一个 `CompletableFuture`，在回调的成功方法中调用 `complete()`，在失败方法中调用 `completeExceptionally()`。

## 7. 代码实践

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

/**
 * CompletableFuture 快速入门指南 (Java 1.8)
 * <p>
 * 本指南通过一系列模拟生产环境的场景，展示 CompletableFuture 的核心用法。
 * 每个 "场景" 都是一个可独立运行的 main 方法，建议按顺序学习。
 *
 * @author Your Guide
 */
public class CompletableFutureQuickStartGuide {

    // 模拟一个RPC/数据库调用的线程池，生产环境中应当自定义并精细化配置
    private static final ExecutorService bizExecutor = Executors.newFixedThreadPool(10, r -> {
        Thread thread = new Thread(r);
        thread.setName("业务线程-" + thread.getId());
        return thread;
    });

    /**
     * 模拟一个耗时操作，例如一次RPC调用、数据库查询等
     *
     * @param operationName 操作名称
     * @param seconds       耗时（秒）
     * @return 操作结果
     */
    private static String mockRpcCall(String operationName, int seconds) {
        try {
            System.out.printf("线程 [%s] 开始执行耗时操作: %s...%n", Thread.currentThread().getName(), operationName);
            TimeUnit.SECONDS.sleep(seconds);
            System.out.printf("线程 [%s] 完成了耗时操作: %s%n", Thread.currentThread().getName(), operationName);
            return String.format("'%s' 的结果", operationName);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("模拟操作被中断", e);
        }
    }

    public static void main(String[] args) throws Exception {
        System.out.println("--- 场景1: 异步执行一个耗时任务 ---");
        scene1_runAsync();
        Thread.sleep(2100); // 等待异步任务执行完毕

//        System.out.println("\n--- 场景2: 异步执行并获取返回值，并对结果进行转换 ---");
//        scene2_thenApply();
//
//        System.out.println("\n--- 场景3: 消费上一步结果，执行一个动作 (无返回值) ---");
//        scene3_thenAccept();
//        Thread.sleep(2100);
//
//        System.out.println("\n--- 场景4: RPC风格的链式调用 (关键API: thenCompose) ---");
//        scene4_thenCompose();
//
//        System.out.println("\n--- 场景5: 并行执行两个任务，并在都完成后组合结果 ---");
//        scene5_thenCombine();
//
//        System.out.println("\n--- 场景6: 处理异步任务中的异常 ---");
//        scene6_exceptionally();
//
//        System.out.println("\n--- 场景7: 竞赛模式，谁快用谁的结果 ---");
//        scene7_applyToEither();
//
//        System.out.println("\n--- 最终整合场景: 模拟一个复杂的用户看板数据加载流程 ---");
//        scene8_comprehensive();

        // 在实际应用中 (如Web服务器)，不需要手动关闭线程池
        // 这里为了演示程序能正常退出而关闭
        bizExecutor.shutdown();
    }

    /**
     * 场景1: 提交一个异步任务，不关心返回结果。
     * API: runAsync(Runnable)
     */
    public static void scene1_runAsync() {
        // 使用 runAsync 异步执行一个没有返回值的任务
        CompletableFuture.runAsync(() -> {
            mockRpcCall("异步记录日志", 2);
        }, bizExecutor);
        System.out.println("主线程继续执行其他任务，日志记录已在后台异步进行...");
    }

    /**
     * 场景2: 异步获取一个结果，并对结果进行同步转换。
     * API: supplyAsync(Supplier), thenApply(Function)
     * thenApply: 当上一步完成后，使用其结果作为输入，执行一个【同步】的转换函数，并返回一个新的CompletableFuture。
     *          注意：thenApply中的代码会由执行上一步的线程或者主线程来执行，它本身不是异步的。
     */
    public static void scene2_thenApply() {
        CompletableFuture<String> userInfoFuture = CompletableFuture.supplyAsync(() -> {
            return mockRpcCall("获取用户信息", 2);
        }, bizExecutor);

        // 使用 thenApply 对上一步的结果进行加工
        CompletableFuture<String> welcomeMessageFuture = userInfoFuture.thenApply(userInfo -> {
            System.out.printf("线程 [%s] 正在加工用户信息...%n", Thread.currentThread().getName());
            return "欢迎您, " + userInfo;
        });

        System.out.println("主线程已提交任务，现在等待最终结果...");
        // .join() 是一个阻塞方法，会等待 Future 完成并返回结果
        String result = welcomeMessageFuture.join();
        System.out.println("主线程获取到最终结果: " + result);
    }

    /**
     * 场景3: 异步获取一个结果，并消费它，不产生新的返回值。
     * API: thenAccept(Consumer)
     */
    public static void scene3_thenAccept() {
        CompletableFuture<Void> notificationFuture = CompletableFuture.supplyAsync(() ->
                        mockRpcCall("获取用户手机号", 2), bizExecutor)
                .thenAccept(phoneNumber -> {
                    // thenAccept 用于消费上一步的结果，它本身没有返回值 (返回 CompletableFuture<Void>)
                    System.out.printf("线程 [%s] 正在发送短信给: %s%n", Thread.currentThread().getName(), phoneNumber);
                });

        System.out.println("主线程已提交发送短信任务，不关心发送结果...");
        // 这里为了演示，主线程可以继续做别的事
    }

    /**
     * 场景4: 异步任务的链式调用，这是 Flink RPC 或任何服务间异步调用的核心。
     * 需求: 1. 先异步获取用户ID -> 2. 根据用户ID再异步获取用户订单。
     * 关键API: thenCompose(Function<T, CompletionStage<U>>)
     * <p>
     * 对比 thenApply:
     * - thenApply(Function<T, U>): 输入T，返回一个普通值U。用于同步转换。
     * - thenCompose(Function<T, CompletionStage<U>>): 输入T，返回一个 CompletableFuture<U>。用于将两个异步操作连接起来。
     * 使用 thenApply 会导致 CompletableFuture<CompletableFuture<Order>> 的嵌套地狱。
     */
    public static void scene4_thenCompose() {
        CompletableFuture<String> finalResultFuture = CompletableFuture.supplyAsync(() -> {
            // 第一个异步调用：获取用户ID
            return mockRpcCall("获取用户ID", 1);
        }, bizExecutor).thenCompose(userId -> {
            // thenCompose接收上一步的结果(userId), 返回一个新的CompletableFuture
            // 这使得我们可以将两个异步调用平滑地串联起来
            System.out.printf("线程 [%s] 拿到了用户ID: %s, 准备发起第二次调用获取订单...%n", Thread.currentThread().getName(), userId);
            // 第二个异步调用：根据用户ID获取订单
            return CompletableFuture.supplyAsync(() -> mockRpcCall("获取 " + userId + " 的订单", 2), bizExecutor);
        });

        System.out.println("主线程已提交【获取用户->获取订单】的链式任务...");
        System.out.println("主线程最终获取到结果: " + finalResultFuture.join());
    }

    /**
     * 场景5: 并行执行两个独立的异步任务，并在它们都完成后，合并结果。
     * 需求: 同时获取用户的 "基本信息" 和 "账户余额"，然后合并成一个视图对象。
     * API: thenCombine(CompletionStage, BiFunction)
     */
    public static void scene5_thenCombine() {
        // 任务1: 异步获取用户基本信息
        CompletableFuture<String> profileFuture = CompletableFuture.supplyAsync(() ->
                mockRpcCall("获取用户基本信息", 2), bizExecutor);

        // 任务2: 异步获取用户账户余额
        CompletableFuture<String> balanceFuture = CompletableFuture.supplyAsync(() ->
                mockRpcCall("获取用户账户余额", 3), bizExecutor);

        System.out.println("主线程提交了两个并行的任务：获取用户信息和余额...");

        // 使用 thenCombine 合并两个任务的结果
        CompletableFuture<String> combinedFuture = profileFuture.thenCombine(balanceFuture, (profile, balance) -> {
            System.out.printf("线程 [%s] 正在合并用户信息和余额...%n", Thread.currentThread().getName());
            return "用户信息: [" + profile + "] | 账户余额: [" + balance + "]";
        });

        System.out.println("主线程等待合并结果...");
        System.out.println("主线程获取到最终合并结果: " + combinedFuture.join());
    }

    /**
     * 场景6: 优雅地处理异步执行过程中的异常。
     * 需求: 获取用户信息时可能抛出异常 (如用户不存在)，此时我们不希望整个流程中断，而是返回一个默认的 "游客" 信息。
     * API: exceptionally(Function)
     */
    public static void scene6_exceptionally() {
        CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() -> {
            // 模拟一个有50%几率失败的操作
            if (ThreadLocalRandom.current().nextBoolean()) {
                throw new RuntimeException("数据库连接超时，无法获取用户信息");
            }
            return mockRpcCall("获取尊贵VIP用户信息", 1);
        }, bizExecutor).exceptionally(ex -> {
            // exceptionally 捕获上一步的异常，并提供一个备用结果
            System.err.printf("线程 [%s] 捕获到异常: %s, 返回默认值%n", Thread.currentThread().getName(), ex.getMessage());
            return "默认游客信息";
        });

        System.out.println("主线程已提交任务，并设置了异常处理...");
        System.out.println("主线程获取到的用户信息: " + userFuture.join());
    }

    /**
     * 场景7: 竞赛模式，两个任务并行，谁先完成就用谁的结果。
     * 需求: 从两个不同的API源获取商品价格，使用先返回的那个。
     * API: applyToEither(CompletionStage, Function)
     */
    public static void scene7_applyToEither() {
        CompletableFuture<String> sourceAFuture = CompletableFuture.supplyAsync(() ->
                        mockRpcCall("从数据源A获取价格", ThreadLocalRandom.current().nextInt(1, 4)),
                bizExecutor
        );

        CompletableFuture<String> sourceBFuture = CompletableFuture.supplyAsync(() ->
                        mockRpcCall("从数据源B获取价格", ThreadLocalRandom.current().nextInt(1, 4)),
                bizExecutor
        );

        System.out.println("主线程已提交两个竞速任务...");

        // applyToEither: 将一个函数应用到两个CF中先完成的那一个的结果上。
        CompletableFuture<String> finalPriceFuture = sourceAFuture.applyToEither(sourceBFuture, price -> {
            System.out.printf("线程 [%s] 正在处理胜出的结果...%n", Thread.currentThread().getName());
            return "最终采纳价格: " + price;
        });

        System.out.println("主线程最终获取的结果: " + finalPriceFuture.join());
    }

    /**
     * 最终整合场景: 模拟一个复杂的用户看板数据加载流程
     * 1. 异步获取用户ID
     * 2. 并行获取用户的 "订单列表" 和 "优惠券列表"
     * 3. 在两者都获取到后，将它们组合起来
     * 4. 如果中途有任何异常，则返回一个 "加载失败" 的提示
     * 5. 最后，将看板信息展示出来
     */
    public static void scene8_comprehensive() {
        // 开始节点
        CompletableFuture<String> start = CompletableFuture.completedFuture("user-123");

        CompletableFuture<String> dashboardFuture = start
                // 1. 链式调用：拿到ID后，触发两个并行任务
                .thenCompose(userId -> {
                    System.out.printf("线程 [%s] 获取到用户ID: %s, 开始并行加载订单和优惠券%n", Thread.currentThread().getName(), userId);

                    // 2a. 并行任务一: 获取订单列表
                    CompletableFuture<String> ordersFuture = CompletableFuture.supplyAsync(
                            () -> mockRpcCall("获取 " + userId + " 的订单列表", 2), bizExecutor);

                    // 2b. 并行任务二: 获取优惠券列表
                    CompletableFuture<String> couponsFuture = CompletableFuture.supplyAsync(
                            () -> mockRpcCall("获取 " + userId + " 的优惠券列表", 3), bizExecutor);

                    // 模拟优惠券服务异常
                    if (ThreadLocalRandom.current().nextBoolean()) {
                        couponsFuture = CompletableFuture.supplyAsync(() -> {
                            throw new RuntimeException("优惠券服务暂时不可用");
                        });
                    }


                    // 3. 组合两个并行任务的结果
                    return ordersFuture.thenCombine(couponsFuture, (orders, coupons) -> {
                        System.out.printf("线程 [%s] 成功组合了订单和优惠券信息%n", Thread.currentThread().getName());
                        return "看板数据 -> [ " + orders + " | " + coupons + " ]";
                    });
                })
                // 4. 设置全局异常处理
                .exceptionally(ex -> {
                    System.err.printf("线程 [%s] 在加载看板数据时发生严重错误: %s%n", Thread.currentThread().getName(), ex.getMessage());
                    return "看板数据加载失败，请稍后重试！";
                });

        // 5. 最终消费结果
        dashboardFuture.thenAccept(dashboardData -> {
            System.out.println("================= 用户看板最终呈现 ==================");
            System.out.println(dashboardData);
            System.out.println("===================================================");
        }).join(); // join() 等待最终的accept执行完成
    }
}
```

1. 保存代码: 将以上代码保存为 CompletableFutureQuickStartGuide.java 文件。
2. 编译运行: 使用您的IDE（如IntelliJ IDEA, Eclipse）或命令行 javac 和 java 命令来编译和运行这个类。
3. 观察控制台输出:
   - 注意线程名称: [业务线程-xx] 和 [main] 的交替出现，直观地展示了任务是在哪个线程上执行的，帮您理解异步和并行的概念。
   - 注意执行顺序: 主线程的日志总是会先打印出来，表明任务提交后主线程没有被阻塞，可以继续执行。
   - 注意结果: 通过 .join() 获取并打印最终结果，验证整个异步流程的正确性。
4. 逐个场景学习:
   - 从 main 方法开始，它依次调用了各个场景。
   - 您可以注释掉 main 方法中的部分调用，只关注某一个特定场景，仔细阅读该场景的代码和注释。
   - 尝试修改 mockRpcCall 的耗时，观察对最终结果的影响，加深理解。
   - 在 scene6 和 scene8 中，由于异常是随机发生的，可以多次运行程序，观察在正常和异常情况下的不同输出。

## 总结与最佳实践

1.  **为IO密集型任务提供专用线程池**：不要阻塞 `ForkJoinPool.commonPool()`，这是最重要的实践之一。
2.  **理解`thenApply` vs `thenCompose`**：`thenApply` 用于同步转换值，`thenCompose` 用于链接依赖的异步任务，这是区分中级和初级使用者的关键。
3.  **明智地选择同步与异步回调**：如果回调中的任务非常快（如简单的内存操作），使用同步版本（`thenApply`）可以减少线程切换开销。如果任务耗时或可能阻塞，请务必使用异步版本（`thenApplyAsync`）。
4.  **优先使用`handle`进行全面的结果处理**：它比 `exceptionally` 和 `whenComplete` 更通用和强大。
5.  **使用 `join()` 代替 `get()`**：在主线程等待最终结果时，`join()` 抛出的是非受检异常（`CompletionException`），代码更简洁。`get()` 抛出的是受检异常（`InterruptedException`, `ExecutionException`），需要显式 `try-catch`。

掌握了以上概念和API，您就已经具备了在实际项目中熟练运用 `CompletableFuture` 的能力。祝您学习愉快！
