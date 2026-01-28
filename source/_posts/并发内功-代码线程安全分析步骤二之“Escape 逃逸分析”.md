---
title: 并发内功-代码线程安全分析步骤二之“Escape 逃逸分析”
abbrlink: 6b9ba70b
date: 2025-11-25 09:33:02
tags:
  - java
categories:
  - java
toc: true
---


# [并发内功] 代码线程安全分析步骤二之“Escape 逃逸分析”

## 前言

这是 S.E.O.P 系列的第二篇。

在 Step 1 中，我们学会了识别“炸弹”（可变状态）。但是，**如果不小心把炸弹扔到了人堆里，那才是灾难。** 如果你把炸弹锁在只有你一个人有钥匙的保险柜里，那它就是绝对安全的。

Step 2 的核心任务就是：**判断变量的作用域（Scope），看它是否逃离了当前线程的控制（Escape）。**

在找出了代码中所有的“可变变量”（Step 1）之后，不要急着加锁。如果这个变量**只有当前线程能看到**，那么恭喜你，你省下了一次加锁的开销。

**Step 2: Escape & Share (看范围)** 专注于分析对象的“生存空间”。只要把状态限制在局部，多线程问题就会自动消失。

-----

## ⚡️ 精华速查版：逃逸判定红黑榜

看代码时，盯着那些在 Step 1 中被标记为“可变”的对象，问自己：**它跑到哪里去了？**

| 🕵️‍♂️ 场景 | 判定结果 | 理由 | 应对策略 |
| :--- | :--- | :--- | :--- |
| 🟢 **方法内自生自灭** | ✅ **安全 (栈封闭)** | 对象在方法内 New 出来，用完就扔，没 return，没传给别人。 | 放心裸奔，无需同步。 |
| 🟢 **ThreadLocal** | ✅ **安全 (线程封闭)** | 放在 `ThreadLocal` 里的对象，虽然看起来像全局变量，但每个线程只能看到自己的副本。 | 记得 `remove()` 防止内存泄漏。 |
| 🔴 **作为返回值 Return** | ⚠️ **潜在危险** | 一旦 return 出去，外部调用者可能把它传给别的线程。 | 尽量返回**不可变对象**或**深拷贝副本**。 |
| 🔴 **作为参数传递** | ⚠️ **潜在危险** | 把它传给了 `new Thread()` 或线程池，或者传给了作为一个单例对象的成员变量。 | 必须认为它已共享，需加锁。 |
| 🔴 **构造函数中赋值给 This** | ❌ **构造逸出** | 在构造函数还没跑完时，就把 `this` 传给了外部监听器。 | **严禁**在构造函数中启动线程或注册监听器。 |
| 🔴 **赋值给 Static 变量** | ❌ **全局逃逸** | 只要挂到了 `static` 字段上，全世界都能看见它。 | 必须使用并发容器或加锁。 |

-----

## 🧠 核心原理：封闭与逃逸

我们的目标是打破并发三要素中的“共享资源”：
$$\text{线程安全} = \text{不共享 (No Share)} + \text{可变性} + \text{并发访问}$$

只要实现了\*\*“不共享”\*\*，就没有线程安全问题。

-----

## ⚔️ 实战案例深度剖析

### 案例一：绝对的安全堡垒 —— 栈封闭 (Stack Confinement)

**场景**：在方法内部处理复杂的临时数据。

```java
public void processOrder() {
    // 1. 在方法内部创建对象
    // HashMap 是非线程安全的，但在这里用安全吗？
    Map<String, Object> tempContext = new HashMap<>(); 
    
    tempContext.put("timestamp", System.currentTimeMillis());
    doSomeCalculation(tempContext);
    
    // 方法结束，tempContext 随之销毁
}
```

**👀 Escape 审查：**

* **变量**：`tempContext` (HashMap)。
* **分析**：它是在 Stack（栈）上引用的。虽然 pass 给了 `doSomeCalculation`，但只要那个方法不把 `tempContext` 存到某个全局变量里，它就永远在当前线程的控制下。
* **结论**：**🟢 安全**。即使 HashMap 本身不安全，但在这种用法下是安全的。

-----

### 案例二：隐秘的越狱 —— 返回值导致的逃逸

**场景**：一个看起来封装很好的类。

```java
public class UserCache {
    private final Map<String, User> cache = new HashMap<>();

    // ... put 方法省略 ...

    // ！！！隐患在这里！！！
    public Map<String, User> getAllUsers() {
        return cache;
    }
}
```

**👀 Escape 审查：**

* **变量**：`cache` (HashMap)。
* **动作**：`return cache;`
* **分析**：虽然 `cache` 是 private 的，但 `getAllUsers()` 把它**交了出去**。外部调用者拿到这个 Map 后，可以随意在别的线程里执行 `map.clear()` 或 `map.put()`。你的封装形同虚设。
* **结论**：**🔴 逃逸 (Unsafe)**。
* **修正**：返回防御性副本 `return new HashMap<>(cache);` 或不可变视图 `Collections.unmodifiableMap(cache);`。

-----

### 案例三：主动投送 —— 跨线程传递

**场景**：主线程创建对象，交给子线程处理。

```java
public void asyncProcess() {
    // 1. 创建局部变量
    StringBuilder data = new StringBuilder("Start");

    // 2. 启动新线程
    new Thread(() -> {
        // ！！！注意！！！
        // 这里的 data 已经逃出了主线程的栈帧
        // 跑到了子线程的栈帧中
        data.append(" -> Processing"); 
    }).start();
    
    // 3. 主线程同时也修改它
    data.append(" -> Main");
}
```

**👀 Escape 审查：**

* **变量**：`data` (StringBuilder)。
* **动作**：作为闭包（Closure）变量被 Lambda 表达式捕获，并传给了新线程。
* **分析**：虽然 `data` 定义在方法里，但它跨越了线程边界。主线程和子线程都在修改同一个 StringBuilder。
* **结论**：**🔴 逃逸 (Unsafe)**。StringBuilder 不是线程安全的，这里会发生竞态条件。

-----

### 案例四：最阴险的陷阱 —— 构造逸出 (This Escape)

**场景**：在 GUI 编程或事件驱动模型中常见。

```java
public class EventListener {
    private int count = 0;

    public EventListener(EventSource source) {
        // ！！！致命错误！！！
        // 注册自己到事件源
        source.register(new Callback() {
            public void onEvent(Event e) {
                // 此时外部线程可能回调这里
                // 但 EventListener 的构造函数还没跑完！
                // count 可能还没有初始化完成
                doSomething(e); 
            }
        });
        
        // 模拟一些初始化耗时
        this.count = 100;
    }
}
```

**👀 Escape 审查：**

* **变量**：`this` (当前对象实例)。
* **动作**：在构造函数完成之前，把 `this` (通过内部类) 暴露给了 `source` 对象。
* **分析**：如果不巧，`source` 在另一个线程立刻触发了回调，那个线程会看到一个\*\*“半成品”\*\*的对象（`count` 可能是 0，而不是 100）。
* **结论**：**🔴 构造逸出 (High Risk)**。
* **修正**：不要在构造函数里做注册。写一个 `init()` 方法，等构造完了再调用。

-----

### 案例五：合法的“作弊” —— ThreadLocal

**场景**：Web 框架中存储当前用户 UserInfo。

```java
public class UserContext {
    // 静态变量，但是是 ThreadLocal
    private static final ThreadLocal<User> holder = new ThreadLocal<>();

    public static void set(User user) { holder.set(user); }
    public static User get() { return holder.get(); }
}
```

**👀 Escape 审查：**

* **变量**：`holder` 是 static 的（看起来是全局）。
* **分析**：`ThreadLocal` 的魔法在于它内部为每个线程维护了一个独立的 Map。线程 A 存进去的东西，线程 B 根本看不到。
* **结论**：**🟢 线程封闭 (Safe)**。
* **注意**：这不是为了解决共享冲突，而是为了**数据隔离**。

-----

## 🎯 总结与下一步

通过 **Step 2: Escape & Share**，我们对 Step 1 中发现的变量进行了二次过滤：

1.  **安全判定**：如果变量老老实实待在**方法栈**里，或者待在 **ThreadLocal** 里，直接放行，它是安全的。
2.  **危险判定**：如果变量被 `return` 出去、被传给 `new Thread`、或者赋值给了 `static` 字段，那么它就**真的共享**了。

现在，我们面对的是\*\*“真正共享且可变”**的变量了。逃无可逃，避无可避。
既然必须共享，我们就必须**“正确地操作”\*\*它。

这就引出了下一篇的核心内容：**Step 3: Operation (查操作)**。

* 仅仅加了锁就安全了吗？
* 为什么 `i++` 加了 volatile 还是不安全？
* 为什么用了 `ConcurrentHashMap` 依然会发生并发 Bug？