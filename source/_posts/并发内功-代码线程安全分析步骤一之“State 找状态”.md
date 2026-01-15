---
title: 并发内功-代码线程安全分析步骤一之“State 找状态”
abbrlink: d9a17448
date: 2025-11-25 09:27:58
tags:
  - Java
categories:
  - Java
toc: true
---


# [并发内功] 代码线程安全分析步骤一之“State 找状态”

## 前言

很多开发者在排查线程安全问题时，习惯直接去找“有没有加锁”。其实，**高手看代码的第一眼，看的不是锁，而是“变量”**。

只要没有“共享的可变状态”，根本就不需要锁。如果连哪里有“鬼”（状态）都不知道，盲目加锁只会导致性能下降或死锁。

本文基于 **S.E.O.P（State, Escape, Operation, Protection）** 排查模型，带你攻克第一关：**Step 1: State (找状态)**。我们将通过 5 个典型案例，训练你“一眼识别危险代码”的本能。

-----

## ⚡️ 精华速查版：三色安全分级表

在阅读详细分析前，请先死死记住这张\*\*“变量红绿灯表”\*\*。当你审查代码（Code Review）时，按此表对变量进行快速定级。

| 🚦 危险等级 | 典型特征 | 审查判定 | 应对策略 |
| :--- | :--- | :--- | :--- |
| 🟢 **绝对安全** | **局部变量**<br>(定义在方法内) | ✅ **栈封闭**<br>天生线程隔离，只要不传出去，绝对安全。 | 无需处理 |
| 🟢 **绝对安全** | **Final 基础类型**<br>(`final int`, `final double`) | ✅ **不可变**<br>不能改，只能读，随便并发。 | 无需处理 |
| 🟡 **隐患区** | **Final 引用类型**<br>(`final List`, `final Map`) | ⚠️ **半个不可变**<br>引用不能变，但**容器里的内容能变**！ | 检查是否有 `add`/`put` 操作，或改用并发容器。 |
| 🟡 **隐患区** | **Final 复杂对象**<br>(`final UserConfig`) | ⚠️ **嵌套可变**<br>对象引用没变，但**对象内部的字段**可能有 Setter。 | 递归审查对象内部代码。 |
| 🔴 **高危区** | **单例中的实例变量**<br>(Spring `@Service` 里的 `int count`) | ❌ **对象级共享**<br>所有请求共享同一个变量，数据必串台。 | **立即移入方法内**，或改用 `ThreadLocal`。 |
| 🔴 **高危区** | **Static 静态变量**<br>(`static Map`, `static SDF`) | ❌ **类级共享**<br>全 JVM 共享，核武级污染源。 | 严禁裸奔，必须加锁或用并发类。 |

-----

## 🔍 核心原理：S.E.O.P 审查模型

所有的线程安全问题，本质上都必须同时满足三个条件：
$$\text{线程不安全} = \text{共享资源} + \text{可变性} + \text{并发访问}$$

**Step 1: State (找状态)** 的核心目标，就是快速识别出代码中符合\*\*“共享 + 可变”\*\*特征的变量。

-----

## ⚔️ 实战案例深度剖析

### 案例一：绝对安全区 —— 局部变量与常量

**场景**：一个简单的订单计算工具类。

```java
public class OrderHelper {
    // A: 静态常量
    public static final double TAX_RATE = 0.09; 

    public double calculateTotal(double price, int quantity) {
        // B: 方法局部变量
        double localTax = price * quantity * TAX_RATE;
        return (price * quantity) + localTax;
    }
}
```

**👀 审查视角：**

* `TAX_RATE`：**安全**。`static` 虽然共享，但 `final` + 基础类型保证了不可变。
* `localTax`：**安全**。这是新手最容易疑惑的地方。局部变量存在于**虚拟机栈（Stack）中，栈是线程私有的。哪怕 100 个线程同时跑这个方法，它们各自拥有独立的 `localTax`，互不干扰。这叫栈封闭（Stack Confinement）**。

-----

### 案例二：Final 的骗局 —— 引用不可变 vs 内容可变

**场景**：小组名单管理。很多开发者看到 `final` 就觉得稳了。

```java
public class TeamConfig {
    // ！！！陷阱！！！
    private final List<String> members = new ArrayList<>();

    public void addMember(String name) {
        members.add(name); // 并发写
    }
}
```

**👀 审查视角：**

* `members`：**🔴 危险**。
* **分析**：`final` 只保证了 `members` 这个引用（钥匙）不会指向别的 List，但它**没有锁住 List 内部的数据**（房子里的家具）。
* **后果**：`ArrayList` 本身不是线程安全的，多线程并发调用 `add` 会导致越界异常或数据覆盖。
* **修正**：使用 `CopyOnWriteArrayList` 或 `Collections.synchronizedList`。

-----

### 案例三：Spring 开发者的噩梦 —— 单例下的实例变量

**场景**：这是 Web 开发中**最高频、最致命**的错误。

```java
@Service // 默认是 Singleton（单例）
public class ReportService {
    
    // ！！！致命错误！！！
    // 开发者想暂存一下当前的 ID
    private int processId = 0; 

    public void generate(String orderId) {
        this.processId = Integer.parseInt(orderId); // A 线程写
        // ... 模拟耗时操作 ...
        System.out.println("Processing: " + this.processId); // B 线程读
    }
}
```

**👀 审查视角：**

* `processId`：**🔴 极度危险**。
* **分析**：Spring 的 Bean 默认是单例的。全系统只有一个 `ReportService` 对象，意味着全系统所有的请求都在抢这一个 `processId` 变量。
* **后果**：**数据串台**。请求 A 的订单号，可能被请求 B 覆盖，导致 A 处理了 B 的数据。
* **修正**：**必须**将 `processId` 移入 `generate` 方法内部，变为局部变量。

-----

### 案例四：全局污染源 —— Static 的陷阱

**场景**：为了方便，搞了一个全局静态缓存或工具对象。

```java
public class StringUtils {
    // 场景 A: 所谓的“性能优化”
    private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

    // 场景 B: 简单的全局缓存
    public static Map<String, String> CACHE = new HashMap<>();
    
    // ... 方法省略
}
```

**👀 审查视角：**

* `sdf`：**💀 必死无疑**。`SimpleDateFormat` 内部有成员变量（Calendar），它是**有状态**的。多线程共用一个 static 实例，时间会乱套。-\> **修正**：用 `DateTimeFormatter` (Java 8+)。
* `CACHE`：**🔴 严重隐患**。`HashMap` 在多线程并发扩容时会导致死循环（JDK7）或数据丢失（JDK8）。static 变量意味着所有线程都在“裸奔”访问同一个容器。-\> **修正**：用 `ConcurrentHashMap`。

-----

### 案例五：隐形炸弹 —— 嵌套对象的审查

**场景**：字段看起来是 final 的，但类型是自定义对象。

```java
public class UserSession {
    private final UserConfig config; // 看起来很安全？

    public void updateTheme(String theme) {
        // ！！！钻进去看！！！
        config.setTheme(theme); 
    }
}
```

**👀 审查视角：**

* `config`：**🟡 隐患**。
* **递归审查 (Drill Down)**：不要只看 `UserSession`，要点进 `UserConfig` 类里看。如果 `UserConfig` 里面有 `setTheme` 这种修改内部状态的方法，那么 `UserSession` 就是线程不安全的。
* **原理**：**可变性的传递**。如果不安全的对象被你持有了，那你也不安全了。

-----

## 🎯 总结与下一步

通过 **Step 1: State**，我们学会了用“扫描仪”般的眼光去看变量：

1.  **无视** 局部变量和常量。
2.  **警惕** `final` 修饰的集合和对象（检查内容是否可变）。
3.  **封杀** 单例中的实例变量和裸奔的静态变量。

**如果你发现变量是安全的（比如局部变量），那么后续的步骤都不用看了，直接 Pass。**
但如果变量被标记为“危险”，是不是就一定会有 Bug 呢？
不一定。如果这个危险变量**从来没有逃出过你的手掌心**（没有被 return 出去，也没有被传给别的线程），它依然可能是安全的。

这就引出了下一篇的核心内容 —— **Step 2: Escape & Share (看范围)**。敬请期待！