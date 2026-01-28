---
title: 并发内功-代码线程安全分析步骤三之“Operation 查操作”
abbrlink: 5d726f37
date: 2025-11-25 09:30:50
tags:
  - java
categories:
  - java
toc: true
---


# [并发内功] 代码线程安全分析步骤三之“Operation 查操作”

## 前言

绝大多数并发 Bug 的根源，不在于变量本身，而在于我们\*\*“想当然”\*\*地认为某些操作是瞬间完成的。

你写了一行代码 `count++`，觉得它是一气呵成的。但在 CPU 眼里，那是三条指令。当线程在这些指令的缝隙中切换时，灾难就发生了。

**Step 3: Operation** 专注于审视代码的逻辑动作，寻找\*\*“竞态条件”（Race Condition）\*\*。

-----

## ⚡️ 精华速查版：操作原子性红黑榜

审查代码时，盯着那些**针对共享变量的读写操作**。不要被代码的行数迷惑，要看语义。

| 🕵️‍♂️ 操作类型 | 代码示例 | 判定结果 | 理由 |
| :--- | :--- | :--- | :--- |
| 🟢 **单一读/写** | `flag = true;` <br> `return x;` | ✅ **通常安全** | 基础类型的赋值和读取（非 long/double 64位机）通常是原子的。配合 `volatile` 可保安全。 |
| 🟢 **原子类方法** | `atomic.incrementAndGet();` | ✅ **安全** | JDK 底层 CAS 保证原子性。 |
| 🔴 **读-改-写 (RMW)** | `i++;` <br> `i = i + 1;` | ❌ **竞态条件** | 经典三步曲：读旧值 -\> 改值 -\> 写回。中间会被打断。 |
| 🔴 **先检查后执行 (Check-Then-Act)** | `if (map.containsKey(k)) { ... }` <br> `if (obj == null) { ... }` | ❌ **竞态条件** | **时间差攻击**。检查通过的那一瞬间，条件可能已经被别的线程改了。 |
| 🔴 **容器组合操作** | `if (!concurrentMap.containsKey(k)) { concurrentMap.put(k, v); }` | ❌ **伪安全** | 容器本身的方法是安全的，但**两个方法之间**的缝隙是不安全的。 |

-----

## 🧠 核心原理：原子性与竞态条件

**竞态条件 (Race Condition)**：程序的输出依赖于线程执行的具体时序。
**原子性 (Atomicity)**：一个操作不可分割。要么全做，要么全不做。

**公式回顾**：
$$\text{不安全操作} = \text{多步操作} - \text{同步保护}$$

-----

## ⚔️ 实战案例深度剖析

### 案例一：经典的“读-改-写”陷阱

**场景**：统计网站访问量。很多新手以为加个 `volatile` 就能解决 `count++` 问题。

```java
public class Counter {
    // volatile 只能保证可见性，保证不了原子性！
    private volatile int count = 0;

    public void increment() {
        // 看起来是一行代码
        count++; 
    }
}
```

**👀 Operation 审查：**

* **动作**：`count++`。
* **微观视角**：
    1.  `LOAD count` (从内存读到寄存器，假设读到 10)
    2.  `ADD 1` (寄存器内加 1，变为 11)
    3.  `STORE count` (写回内存)
* **危机**：线程 A 执行完第 2 步被挂起。线程 B 进来读到 10，加到 11，写回。线程 A 醒来，也写回 11。**明明加了两次，结果只加了 1。**
* **结论**：❌ **非原子操作**。
* **修正**：使用 `AtomicInteger` 或 `synchronized`。

-----

### 案例二：隐蔽的“先检查后执行” (Check-Then-Act)

**场景**：单例模式的懒加载，或者业务逻辑中的条件判断。

```java
public class LazyInit {
    private ExpensiveObject instance;

    public ExpensiveObject getInstance() {
        // Step A: 检查
        if (instance == null) {
            // Step B: 行动 (只有 null 时才创建)
            instance = new ExpensiveObject();
        }
        return instance;
    }
}
```

**👀 Operation 审查：**

* **动作**：`if (instance == null) ... instance = ...`
* **危机**：
    * 线程 A 检查 `instance` 为 null。
    * **（切换）** 线程 B 检查 `instance` 也为 null。
    * 线程 B 创建对象 `Obj1`，赋值给 `instance`。
    * **（切换）** 线程 A 继续执行（因为它记得刚才检查结果是 null），创建对象 `Obj2`，覆盖了 `instance`。
* **结论**：❌ **竞态条件**。这是典型的 TOCTOU (Time Of Check to Time Of Use) 漏洞。

-----

### 案例三：并发容器的“组合拳”失效

**场景**：这是**高手**也容易犯的错。使用了线程安全的 `ConcurrentHashMap`，就以为高枕无忧了。

```java
public class Dictionary {
    // 线程安全的容器
    private final Map<String, Integer> wordCounts = new ConcurrentHashMap<>();

    public void addWord(String word) {
        // 目标：如果单词不存在，存入 1；如果存在，+1。
        
        Integer count = wordCounts.get(word); // 操作 1
        if (count == null) {
            wordCounts.put(word, 1);          // 操作 2
        } else {
            wordCounts.put(word, count + 1);  // 操作 3
        }
    }
}
```

**👀 Operation 审查：**

* **动作**：`get` -\> `check` -\> `put`。
* **分析**：
    * `wordCounts.get()` 是线程安全的（原子的）。
    * `wordCounts.put()` 是线程安全的（原子的）。
    * **但是！** 两个操作合在一起，就**不是**原子的。
* **危机**：线程 A 发现 "Java" 不在，准备 put 1。此时线程 B 也发现 "Java" 不在，也 put 1。结果 "Java" 的计数是 1，而不是 2。
* **结论**：❌ **组合操作不安全**。
* **修正**：使用并发容器提供的**原子组合方法**，如 `putIfAbsent` 或 `compute`。
  ```java
  // 正确写法：利用 compute 原子指令
  wordCounts.compute(word, (k, v) -> (v == null) ? 1 : v + 1);
  ```

-----

### 案例四：不安全的控制流

**场景**：使用一个布尔标记来控制初始化。

```java
public class Initializer {
    private boolean initialized = false;

    public void init() {
        // 如果已经初始化，就抛异常
        if (initialized) { 
            throw new RuntimeException("Already initialized");
        }
        
        // 执行真正的初始化
        doHeavyInit();
        
        initialized = true;
    }
}
```

**👀 Operation 审查：**

* **动作**：`check flag` -\> `act` -\> `set flag`。
* **分析**：两个线程可能同时通过 `if (initialized)` 的检查，然后都去执行 `doHeavyInit()`。这可能导致资源重复加载甚至严重错误。
* **结论**：❌ **竞态条件**。`boolean` 标记的读写虽然是原子的，但逻辑流不是。
* **修正**：使用 `AtomicBoolean.compareAndSet(false, true)`。

-----

## 🎯 总结与下一步

**Step 3: Operation** 告诉我们：**永远不要相信两行代码之间没有空隙。**

当你看到代码中有以下结构时，警报必须拉响：

1.  **Read-Modify-Write**: 像 `i++` 这种累加操作。
2.  **Check-Then-Act**: `if (x == null) init();` 这种条件初始化。
3.  **Composite Actions**: `map.get` 后再 `map.put`，即使是 ConcurrentMap 也不行。

**到了这一步，我们已经完整确诊了病情：**

* Step 1: 发现了病毒（共享变量）。
* Step 2: 确认病毒已扩散（变量逃逸）。
* Step 3: 观察到病毒在攻击细胞（非原子操作导致数据损坏）。

接下来，就是治病救人的时刻了。我们必须施加手段来保护这些脆弱的代码。
这就引出了 **S.E.O.P** 的最后一环：**Step 4: Protection (验防护)**。

* 用 `synchronized` 还是 `Lock`？
* 锁对象是 `this` 还是 `class`？
* 为什么我的锁加了还是没用（锁的覆盖范围不对）？