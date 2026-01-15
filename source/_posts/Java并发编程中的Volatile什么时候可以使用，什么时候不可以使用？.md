---
title: Java并发编程中的Volatile什么时候可以使用，什么时候不可以使用？
abbrlink: 164a1666
date: 2025-08-05 23:40:59
tags:
  - Java
categories:
  - Java
toc: true
---


在Java并发编程中，`volatile`是一个既强大又容易被误用的关键字。它像一把轻巧的锁，提供了比`synchronized`更低的开销，但适用场景也更为受限。《Java并发编程实战》一书为我们指明了方向，提出了使用`volatile`必须同时满足的三个条件。

然而，原书中的表述对于初学者来说可能有些晦涩。本文旨在用最通俗的语言和最直观的代码，带你彻底搞懂`volatile`的正确用法。

> **核心回顾：`volatile`的作用**
>
> 在深入探讨之前，我们先快速回顾一下`volatile`的两个核心作用：
>
> 1.  **可见性（Visibility）**: 当一个线程修改了`volatile`变量的值，这个新值对其他线程是立即可见的。它通过防止编译器和CPU的指令重排序、并确保修改后的值立即写回主内存来实现。
> 2.  **有序性（Ordering）**: 在一定程度上防止指令重排序。具体来说，它能确保`volatile`变量之前的代码先执行，`volatile`变量之后的代码后执行，但不能保证`volatile`代码块内部的指令不被重排序。
>
> **关键点**：`volatile`**不保证原子性**。像`count++`这样的复合操作，它无法保证其执行过程不被打断。

现在，让我们逐一剖析使用`volatile`的三条黄金法则。

## **法则一：写入不依赖当前值，或只有单线程写入**

**原文**

> 对变量的写入操作不依赖变量的当前值,或者你能确保只有单个线程更新变量的值。

**通俗解读**
这条法则是关于**原子性**的。它告诉我们，如果你对变量的操作不是“原子”的，那么`volatile`就帮不了你，除非你确保只有一个线程在“搞事情”（写入）。

什么是“写入不依赖当前值”？
简单的赋值操作就是，例如 `running = false;` 或者 `value = 10;`。这个操作本身就是原子的，`volatile`可以确保这个赋值结果能被所有线程立刻看到。

什么是“写入依赖当前值”？
最典型的例子就是 `count++`。这个操作实际上包含了三个步骤：

1.  读取 `count` 的当前值。
2.  在读取到的值上加1。
3.  将新值写回 `count`。

在多线程环境下，线程A可能刚读取完`count`（比如是10），还没来得及加1写回，线程B也读取了`count`（也是10）。然后A和B各自加1，都把11写回。结果是`count`只增加了1，而不是2。`volatile`无法阻止这种线程间的交错执行。

**代码示例**

**场景一：错误使用 `volatile`**

我们来看一个多线程计数器，这正是“写入依赖当前值”的典型场景。

```java
public class VolatileCounter {
    // 错误用法：多线程对 volatile 变量进行 read-modify-write 操作
    public volatile int count = 0;

    public void increment() {
        count++; // 这不是原子操作！
    }

    public static void main(String[] args) throws InterruptedException {
        VolatileCounter counter = new VolatileCounter();
        Thread[] threads = new Thread[10];

        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter.increment();
                }
            });
            threads[i].start();
        }

        for (Thread t : threads) {
            t.join(); // 等待所有线程执行完毕
        }

        // 期望结果是 10000，但实际运行结果几乎总是一个小于10000的数字
        System.out.println("Final count is: " + counter.count);
    }
}
```

**为什么错？**
即使`count`是`volatile`的，保证了每次读取都从主内存获取最新值，但`++`操作的“读-改-写”过程不是原子的。多个线程可能同时读取到同一个旧值，导致部分自增操作丢失。在这种场景下，你应该使用`AtomicInteger`或者`synchronized`。

**场景二：正确使用 `volatile`**

现在来看一个适合`volatile`的场景：一个线程发出“关闭”信号，其他线程读取这个信号。

```java
public class ShutdownSwitch {
    // 正确用法：作为状态标志，由一个线程写入，其他线程读取
    public volatile boolean shutdownRequested = false;

    public void shutdown() {
        System.out.println("Shutdown signal received!");
        shutdownRequested = true; // 简单的原子赋值
    }

    public void doWork() {
        while (!shutdownRequested) {
            // 持续工作...
            System.out.println("Worker thread is running...");
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        System.out.println("Worker thread stopped.");
    }

    public static void main(String[] args) throws InterruptedException {
        ShutdownSwitch switcher = new ShutdownSwitch();
        Thread worker = new Thread(switcher::doWork);
        worker.start();

        Thread.sleep(2000); // 让工作线程运行一会儿

        // 由主线程（单个线程）发起关闭信号
        switcher.shutdown();
    }
}
```

**为什么对？**
这里，`shutdownRequested`变量只有一个写入者（`main`线程），但有多个读取者（`worker`线程）。写入操作`shutdownRequested = true;`是原子的，不依赖其当前值。`volatile`完美地确保了`worker`线程能立即看到`shutdownRequested`的变化，从而安全地退出循环。

## **法则二：变量不与其它状态变量构成“不变性条件”**

**原文**

> 该变量不会与其他状态变量一起纳入不变性条件中。

**通俗解读**
这条法则是关于**多个变量的原子性绑定**的。如果一个变量的合法性依赖于另一个变量的值（它们共同构成一个“不变的规则”），那么`volatile`就无能为力了。

“不变性条件”（Invariant）指的是一个或多个变量之间必须始终保持的某种关系。例如，在一个表示范围的类中，`lower`（下界）必须始终小于等于`upper`（上界）。这就是一个不变性条件：`lower <= upper`。

如果你将`lower`和`upper`都声明为`volatile`，当一个线程要同时更新它们时（比如从`[0, 5]`更新到`[3, 8]`），这个更新过程不是原子的。它会分两步：

1.  `this.lower = 3;`
2.  `this.upper = 8;`

在第一步和第二步之间，另一个线程可能会介入，读取到一个`lower`为3，但`upper`仍然是旧值5的**临时非法状态**。`volatile`只能保证单个变量的可见性，无法将多个变量的修改“打包”成一个原子操作。

**代码示例**

**场景一：错误使用 `volatile`**

```java
public class NumberRange {
    // 错误用法：lower 和 upper 共同构成不变性条件 (lower <= upper)
    public volatile int lower = 0;
    public volatile int upper = 5;

    public void setLower(int i) {
        // 检查 i <= upper 并不安全，因为在你检查后，upper可能被其他线程改变
        if (i > upper) {
            throw new IllegalArgumentException("Lower bound cannot be greater than upper bound");
        }
        lower = i;
    }

    public void setUpper(int i) {
        // 同样，这里的检查也不安全
        if (i < lower) {
            throw new IllegalArgumentException("Upper bound cannot be less than lower bound");
        }
        upper = i;
    }
    
    // 假设一个检查方法
    public boolean isInRange(int i) {
        return (i >= lower && i <= upper);
    }
}
```

**为什么错？**
想象线程A调用`setLower(4)`，同时线程B调用`setUpper(3)`。由于没有锁，它们的执行可能交错，导致最终范围变为`[4, 3]`，这显然是错误的。`volatile`无法保证这两个`set`方法之间的原子性，也无法保护`lower <= upper`这个不变性条件。

**场景二：正确的做法（使用`synchronized`）**

```java
public class SafeNumberRange {
    // 正确做法：使用锁来保护不变性条件
    private int lower = 0;
    private int upper = 5;

    // 使用 synchronized 关键字将对 lower 和 upper 的访问打包成原子操作
    public synchronized void setRange(int newLower, int newUpper) {
        if (newLower > newUpper) {
            throw new IllegalArgumentException("Range is invalid");
        }
        lower = newLower;
        upper = newUpper;
    }

    public synchronized boolean isInRange(int i) {
        return (i >= lower && i <= upper);
    }
}
```

**为什么对？**
通过`synchronized`关键字，我们确保了任何时候只有一个线程可以进入`setRange`或`isInRange`方法。当一个线程在修改`lower`和`upper`时，其他线程必须等待。这样就保证了其他线程永远不会看到`lower > upper`的中间状态，不变性条件得到了保护。

## **法则三：访问变量时不需要加锁**

**原文**

> 在访问变量时不需要加锁。

**通俗解读**
这更像是一条工程实践原则。它的意思是：如果你在代码的某个地方**无论如何都需要一个锁**（比如`synchronized`），那么就没必要再给这个锁保护的变量加上`volatile`了。

`synchronized`块已经同时提供了**可见性**和**原子性**。当线程退出`synchronized`块时，它会把所有在块内修改过的共享变量的值刷新回主内存；当线程进入`synchronized`块时，它会从主内存重新加载共享变量的值。

所以，`synchronized`已经包含了`volatile`的内存可见性功能，并且功能更强大（还提供了原子性）。在同一个地方同时使用两者是多余的，徒增代码的复杂性。

**代码示例**

```java
public class RedundantVolatile {
    private final Object lock = new Object();
    
    // 冗余用法：既然已经有锁了，volatile 就是不必要的
    private volatile int value; 

    public void updateValue(int newValue) {
        // 这个锁已经保证了 value 的可见性和操作的原子性
        synchronized (lock) {
            this.value = newValue;
        }
    }

    public int getValue() {
        synchronized (lock) {
            return this.value;
        }
    }
}
```

**为什么冗余？**
这里的`volatile`是完全多余的。`synchronized(lock)`已经确保了`value`的修改对所有后续进入`synchronized(lock)`块的线程都是可见的。去掉`volatile`，程序的正确性不会有任何改变。

### **总结：什么时候才应该用 `volatile`？**

结合以上三条法则，我们可以总结出`volatile`的最佳使用场景：**简单的、独立的、原子赋值的状态标志**。

| 何时使用 `volatile`？ (必须同时满足)                                | 何时**不应**使用 `volatile`？ (任一情况)                           | 替代方案                                     |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------- |
| 1. 写入是原子的 (如 `flag = true`) 或仅由单线程写入。               | 1. 写入依赖旧值 (如 `count++`) 且有多线程写入。                        | `java.util.concurrent.atomic.*` (如 `AtomicInteger`) |
| 2. 该变量是独立的，不与其他变量构成不变性条件 (如 `lower <= upper`)。 | 2. 变量的有效性依赖其他变量 (如范围`[lower, upper]`)。               | `synchronized` 或 `ReentrantLock`                |
| 3. 访问该变量的代码块本身不需要其他原因的锁。                       | 3. 访问该变量的代码无论如何都需要一个锁来保护其他资源。                | `synchronized` 或 `ReentrantLock`                |

**最终建议**
`volatile`是优化并发性能的利器，但它是一把需要精准使用的手术刀。如果你对它的使用场景有任何一丝不确定，那么更安全、更通用的`synchronized`或`java.util.concurrent`包下的工具类（如`AtomicInteger`, `ReentrantLock`）通常是更好的选择。

希望这篇博文能帮助你彻底掌握`volatile`的精髓，写出更健壮、更高效的并发代码！

-----