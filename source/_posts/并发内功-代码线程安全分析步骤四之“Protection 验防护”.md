---
title: 并发内功-代码线程安全分析步骤四之“Protection 验防护”
abbrlink: e748168e
date: 2025-11-25 09:32:11
tags:
  - java
categories:
  - java
toc: true
---


# [并发内功] 代码线程安全分析步骤四之“Protection 验防护”

## 前言

加锁很容易，但**加对锁**很难。

在审查代码时，当你看到 `synchronized` 或 `Lock` 时，千万不要掉以轻心。反而要更加警惕：开发者既然加了锁，说明他潜意识里觉得这里有危险。你需要做的，就是验证这把锁是否真的锁住了危险。

**Step 4: Protection** 专注于审视同步机制的**有效性（Effectiveness）和完整性（Completeness）**。

-----

## ⚡️ 精华速查版：防护有效性红黑榜

拿着代码里的锁，去对这张表。只要中了一条红色的，锁就是废的。

| 🛡️ 防护场景 | 代码特征 | 判定结果 | 理由 |
| :--- | :--- | :--- | :--- |
| 🟢 **标准同步** | `synchronized(this)` 保护实例变量 | ✅ **有效** | 锁对象和被保护的数据属于同一个实例生命周期。 |
| 🟢 **全局同步** | `synchronized(Class.class)` 保护 `static` 变量 | ✅ **有效** | 静态变量属于类，锁也属于类，门当户对。 |
| 🔴 **对象不匹配** | `synchronized(this)` 保护 `static` 变量 | ❌ **无效锁** | **经典的“锁自家门，管广场人”。** 实例锁管不住全局变量。 |
| 🔴 **一次性锁** | `synchronized(new Object())` | ❌ **无效锁** | 每次进来都换一把新锁，根本没人排队。 |
| 🔴 **半边锁** | 写操作加锁了，读操作没加 | ❌ **脏读** | 读线程可以看到写线程改了一半的数据（可见性问题）。 |
| 🔴 **范围过小** | `synchronized(map) { if(!map.contains)... }` | ⚠️ **需谨慎** | 如果锁没有覆盖**整个**复合操作逻辑，依然有竞态条件。 |

-----

## 🧠 核心原理：锁的“监视器”法则

要判断锁是否有效，必须回答两个核心问题：

1.  **锁的是谁？(Lock Identity)**：所有访问该资源的线程，抢的是**同一个**对象吗？
2.  **锁了多少？(Lock Scope)**：锁的范围是否覆盖了**整个**“检查-执行”的复合操作？

**黄金法则**：对于同一个共享变量的所有读写操作，必须持有**同一把锁**。

-----

## ⚔️ 实战案例深度剖析

### 案例一：经典的“张冠李戴” —— 锁错对象

**场景**：维护一个全局的计数器，为了线程安全，加了 synchronized。

```java
public class GlobalCounter {
    // 静态变量：全局共享
    private static int count = 0;

    // 错误的做法：锁的是实例对象 (this)
    public synchronized void increment() {
        count++;
    }
    
    // 等价于：
    // public void increment() {
    //     synchronized(this) { count++; }
    // }
}
```

**👀 Protection 审查：**

* **State**: `count` 是 `static` 的，属于 `GlobalCounter.class`。
* **Lock**: `synchronized` 修饰实例方法，锁的是 `this`（当前 `new` 出来的对象）。
* **危机**：
    * 线程 A new 了一个 `GlobalCounter gc1`，持有 `gc1` 的锁，修改 `count`。
    * 线程 B new 了一个 `GlobalCounter gc2`，持有 `gc2` 的锁，修改 `count`。
    * **两把不同的锁，根本互不影响。** 两个人同时进屋砸家具。
* **结论**：❌ **无效锁**。
* **修正**：`static synchronized` 或 `synchronized(GlobalCounter.class)`。

-----

### 案例二：隐蔽的“半边锁” —— 读写不同步

**场景**：为了性能，只在写的时候加锁，读的时候不加。

```java
public class Account {
    private double balance;

    // 写操作：加锁了，很安全
    public synchronized void setBalance(double balance) {
        this.balance = balance;
    }

    // 读操作：没加锁，“为了快”
    public double getBalance() {
        return this.balance;
    }
}
```

**👀 Protection 审查：**

* **Lock**: 写加锁，读裸奔。
* **危机**：
    * **可见性问题**：在 Java 内存模型 (JMM) 中，如果没有同步（锁或 volatile），线程 A 写入的值，线程 B **不一定能立马看见**。线程 B 可能读到很久以前的旧值。
    * **原子性问题（针对 long/double）**：在 32 位虚拟机上，64 位的 `double` 读写不是原子的。读线程可能读到“高 32 位是新的，低 32 位是旧的”这种畸形数据。
* **结论**：❌ **不安全**。
* **修正**：给 `getBalance` 也加上 `synchronized`，或者将 `balance` 声明为 `volatile`（如果只是简单的赋值）。

-----

### 案例三：可笑的“一次性锁”

**场景**：新手在方法内部定义锁对象。

```java
public class ErrorLock {
    private int items = 0;

    public void add() {
        // ！！！每一次调用，都new一个新的锁对象！！！
        Object lock = new Object(); 
        
        synchronized (lock) {
            items++;
        }
    }
}
```

**👀 Protection 审查：**

* **Lock**: `lock` 是局部变量。
* **危机**：每个线程进来，都自己造一把锁，然后锁住自己。这就像每个人都带了一扇自己的门，关上门自己玩，根本起不到“互斥”的作用。
* **结论**：❌ **完全无效**。
* **修正**：`lock` 必须是 `private final Object lock = new Object();` 定义为成员变量。

-----

### 案例四：遗憾的“漏网之鱼” —— 范围过小

**场景**：使用了线程安全的容器，且加了锁，但锁的范围不对。

```java
public class Cache {
    // 即使是线程安全的 Map
    private Map<String, String> map = new ConcurrentHashMap<>();

    public void putIfMissing(String key, String value) {
        // 锁住了，但只锁住了 containsKey 这一步？
        // 或者锁错了对象？
        
        // 假设这里想做原子操作
        synchronized (this) {
            if (!map.containsKey(key)) {
                // ！！！这里的空隙！！！
                
                // 假设在这里释放了锁（当然 synchronized 不会这样，但如果是手动 Lock 容易犯错）
                // 或者说，如果 containsKey 和 put 是两个独立的加锁方法，
                // 那么它们之间就是裸奔的。
                map.put(key, value);
            }
        }
    }
}
```

*注：上面的 `synchronized(this)` 其实是涵盖了整个逻辑的，是安全的（虽然性能可能低）。我们来看一个更典型的**错误示范**：*

```java
// 错误示范：Vector 是线程安全的，但组合操作不是
public void processVector(Vector<Integer> v) {
    // 这里的 v.size() 是安全的
    // 这里的 v.get(i) 是安全的
    for (int i = 0; i < v.size(); i++) {
        // 但在这个循环过程中，别的线程可能删除了元素！
        // 导致 v.get(i) 抛出 IndexOutOfBoundsException
        doSomething(v.get(i)); 
    }
}
```

**👀 Protection 审查：**

* **Lock Scope**: 容器内部的方法是加锁的，但**遍历容器**这个动作本身没有整体加锁。
* **结论**：❌ **复合操作未保护**。
* **修正**：在循环外部加锁 `synchronized(v) { for... }`。

-----

## 🏆 S.E.O.P 终极复盘

恭喜你！你已经掌握了并发代码审查的\*\*“三维审视法”\*\*全套内功。

让我们最后一次回顾这套排查连招：

1.  **Step 1: State (找状态)**

    * *口诀*：盯着变量看。
    * *目标*：找出所有的“共享可变变量”。
    * *结果*：如果都是局部变量/不可变，直接 Pass。否则进入下一步。

2.  **Step 2: Escape (看范围)**

    * *口诀*：盯着 return 和传参。
    * *目标*：确认变量是否逃出了当前线程。
    * *结果*：如果是栈封闭或 ThreadLocal，直接 Pass。否则进入下一步。

3.  **Step 3: Operation (查操作)**

    * *口诀*：盯着 `++`、`if` 和组合逻辑。
    * *目标*：确认是否存在非原子的竞态条件。
    * *结果*：如果是单步原子操作，Pass。否则进入下一步。

4.  **Step 4: Protection (验防护)**

    * *口诀*：盯着 `synchronized` 和锁对象。
    * *目标*：确认锁是否\*\*“同体”**（同一个对象）且**“全包”\*\*（覆盖全过程）。
    * *结果*：如果锁没问题，那么代码安全。

**记住：并发编程没有玄学，只有严谨的逻辑。** 当你学会用 S.E.O.P 的视角看代码时，那些潜伏的 Bug 将在你的注视下无所遁形。