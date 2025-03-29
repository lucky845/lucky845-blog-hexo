---
title: 【Java】CAS机制详解
date: 2025-03-29 10:00:00
tags:
  - Java
  - 并发编程
  - JUC
  - 原子操作
categories: 
  - Java
  - 技术笔记
  - Java基础
abbrlink: 7e8f9a2d
keywords: Java,CAS,并发编程,原子操作,ABA问题
description: 本文详细介绍了Java中的CAS(Compare And Swap)机制，包括其基本原理、底层实现、应用场景、ABA问题及解决方案，以及与传统锁机制的对比分析。
---

# Java CAS机制详解

## 前言

在Java并发编程中，锁机制是保证线程安全的常用手段。然而，传统的锁机制（如synchronized）存在性能开销大、可能导致死锁等问题。为了解决这些问题，Java引入了一种基于硬件原语的轻量级同步机制——CAS（Compare And Swap，比较并交换）。本文将深入探讨CAS的原理、实现和应用，帮助读者全面理解这一重要的并发控制机制。

## 1. CAS基本概念

### 1.1 什么是CAS

CAS是一种无锁算法，全称为Compare And Swap（比较并交换）。它是一种原子操作，可以将指定内存位置的值与预期值进行比较，只有当它们相同时，才将该内存位置的值修改为新的值。整个比较并交换的操作是一个原子操作，不会被线程调度机制打断。

CAS操作包含三个操作数：
- **内存位置V**：要更新的变量的内存地址
- **预期值A**：更新前，变量预期的值
- **新值B**：要设置的新值

CAS的伪代码表示如下：

```
function CAS(V, A, B) {
    if (V == A) {
        V = B;
        return true; // 更新成功
    } else {
        return false; // 更新失败
    }
}
```

### 1.2 CAS的特点

1. **原子性**：CAS操作是一个原子操作，执行过程不会被中断。
2. **非阻塞性**：线程执行CAS操作失败时，可以立即得知结果并决定后续操作，不需要被挂起等待。
3. **乐观性**：CAS是一种乐观锁的实现，假设数据在大多数情况下不会发生冲突。
4. **轻量级**：相比传统锁机制，CAS的性能开销更小。

## 2. CAS的底层实现

### 2.1 硬件层面的支持

CAS操作的原子性是由CPU硬件指令保证的。现代处理器都支持CAS操作，如：
- x86架构：CMPXCHG指令
- SPARC架构：CAS指令
- IA64架构：CASA指令

这些指令能够在硬件层面保证比较和交换操作的原子性，不会被线程调度打断。

### 2.2 Java中的实现 - Unsafe类

Java中的CAS操作主要通过`sun.misc.Unsafe`类实现。Unsafe类提供了一系列的compareAndSwap*方法：

```java
public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object update);
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int update);
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long update);
```

这些方法都是native方法，它们通过JNI（Java Native Interface）调用底层的C++代码，最终调用CPU的原子指令来完成CAS操作。

下面是一个使用Unsafe实现CAS操作的示例：

```java
import sun.misc.Unsafe;
import java.lang.reflect.Field;

public class CASDemo {
    private volatile int value;
    private static final Unsafe unsafe;
    private static final long valueOffset;
    
    static {
        try {
            // 通过反射获取Unsafe实例
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            unsafe = (Unsafe) field.get(null);
            
            // 获取value字段的内存偏移量
            valueOffset = unsafe.objectFieldOffset(
                CASDemo.class.getDeclaredField("value"));
        } catch (Exception ex) {
            throw new Error(ex);
        }
    }
    
    public final int get() {
        return value;
    }
    
    public final boolean compareAndSet(int expectedValue, int newValue) {
        return unsafe.compareAndSwapInt(this, valueOffset, expectedValue, newValue);
    }
    
    public final void increment() {
        int current;
        do {
            current = get();
        } while (!compareAndSet(current, current + 1));
    }
}
```

## 3. Java并发包中的CAS应用

### 3.1 原子类（Atomic*）

Java并发包（java.util.concurrent.atomic）提供了一系列原子类，它们内部使用CAS操作来保证线程安全：

- **AtomicBoolean**：原子更新布尔类型
- **AtomicInteger**：原子更新整型
- **AtomicLong**：原子更新长整型
- **AtomicReference**：原子更新引用类型
- **AtomicIntegerArray**：原子更新整型数组
- **AtomicLongArray**：原子更新长整型数组
- **AtomicReferenceArray**：原子更新引用类型数组

以AtomicInteger为例，其incrementAndGet方法的实现如下：

```java
public final int incrementAndGet() {
    for (;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
            return next;
    }
}
```

这个方法使用了CAS操作和自旋（循环尝试）的方式来实现原子递增。

### 3.2 并发容器

Java中的许多并发容器也使用了CAS操作，如ConcurrentHashMap：

```java
// JDK 1.8 ConcurrentHashMap中的部分代码
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ...
    for (Node<K,V>[] tab = table;;) {
        // ...
        if ((f = tabAt(tab, i)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;
        }
        // ...
    }
    // ...
}
```

### 3.3 锁实现

Java中的显式锁（如ReentrantLock）也在内部使用了CAS操作：

```java
// AbstractQueuedSynchronizer中的部分代码
public final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

## 4. CAS的问题与解决方案

### 4.1 ABA问题

ABA问题是CAS操作的一个典型问题：如果一个值原来是A，变成了B，又变回了A，那么CAS操作检查时会认为这个值没有被修改过，但实际上它已经经历了A->B->A的变化。

例如：
1. 线程1读取值A
2. 线程1被挂起
3. 线程2将值从A修改为B，再修改回A
4. 线程1恢复执行，发现值仍然是A，CAS操作成功

但实际上，值已经被修改过了，这可能导致程序逻辑错误。

### 4.2 解决ABA问题 - 版本号/时间戳

解决ABA问题的一种常用方法是使用版本号或时间戳。每次修改值的同时增加版本号，这样即使值本身变回了原来的值，版本号也是不同的。

Java提供了AtomicStampedReference和AtomicMarkableReference类来解决ABA问题：

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABADemo {
    private static AtomicStampedReference<Integer> atomicStampedRef = 
        new AtomicStampedReference<>(100, 0);
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            // 获取当前版本号
            int stamp = atomicStampedRef.getStamp();
            System.out.println("t1 第一次读取：" + atomicStampedRef.getReference() + ", 版本号：" + stamp);
            
            // 暂停一秒，让t2执行
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            
            // CAS操作，同时检查值和版本号
            boolean success = atomicStampedRef.compareAndSet(
                100, 101, stamp, stamp + 1);
            System.out.println("t1 CAS操作结果：" + success);
        });
        
        Thread t2 = new Thread(() -> {
            // 获取当前版本号
            int stamp = atomicStampedRef.getStamp();
            System.out.println("t2 第一次读取：" + atomicStampedRef.getReference() + ", 版本号：" + stamp);
            
            // 执行ABA操作
            atomicStampedRef.compareAndSet(100, 101, stamp, stamp + 1);
            System.out.println("t2 将值改为：" + atomicStampedRef.getReference() + ", 版本号：" + atomicStampedRef.getStamp());
            
            atomicStampedRef.compareAndSet(101, 100, atomicStampedRef.getStamp(), atomicStampedRef.getStamp() + 1);
            System.out.println("t2 将值改回：" + atomicStampedRef.getReference() + ", 版本号：" + atomicStampedRef.getStamp());
        });
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

运行结果：
```
t1 第一次读取：100, 版本号：0
t2 第一次读取：100, 版本号：0
t2 将值改为：101, 版本号：1
t2 将值改回：100, 版本号：2
t1 CAS操作结果：false
```

可以看到，虽然值变回了100，但由于版本号已经从0变成了2，所以t1的CAS操作失败了。

### 4.3 循环时间长开销大

CAS操作如果长时间不成功，会导致循环时间长，CPU开销大。解决方法：

1. **限制自旋次数**：设置一个自旋次数的上限，超过后使用传统锁。
2. **使用退避策略**：如指数退避，每次失败后等待时间增加。
3. **合理设计并发粒度**：减少竞争，如ConcurrentHashMap的分段锁设计。

### 4.4 只能保证一个共享变量的原子操作

CAS只能保证对单个变量的原子操作。如果需要对多个变量进行原子操作，可以：

1. **使用AtomicReference包装多个变量**：将多个变量封装在一个对象中，然后使用AtomicReference。
2. **使用锁**：对于复杂的原子操作，可以使用传统锁机制。

## 5. CAS与锁的对比

### 5.1 性能对比

在低竞争环境下，CAS的性能通常优于传统锁：

1. **无需线程切换**：CAS失败时不会导致线程阻塞和唤醒，避免了线程切换的开销。
2. **无需操作系统介入**：CAS是用户态操作，不需要进入内核态。
3. **适合短时间操作**：对于执行时间短的操作，CAS的自旋等待比线程阻塞更高效。

但在高竞争环境下，CAS可能导致大量的自旋和重试，反而不如传统锁。

### 5.2 使用场景对比

**CAS适合的场景**：
- 竞争不激烈的环境
- 只需要对单个变量进行原子操作
- 操作执行时间短

**传统锁适合的场景**：
- 竞争激烈的环境
- 需要对多个变量进行原子操作
- 操作执行时间长
- 需要公平性保证

## 6. 实践应用

### 6.1 实现一个线程安全的计数器

```java
import java.util.concurrent.atomic.AtomicLong;

public class ConcurrentCounter {
    private AtomicLong count = new AtomicLong(0);
    
    public long increment() {
        return count.incrementAndGet();
    }
    
    public long decrement() {
        return count.decrementAndGet();
    }
    
    public long get() {
        return count.get();
    }
    
    public static void main(String[] args) throws InterruptedException {
        final ConcurrentCounter counter = new ConcurrentCounter();
        final int threadCount = 100;
        final int incrementsPerThread = 10000;
        
        Thread[] threads = new Thread[threadCount];
        for (int i = 0; i < threadCount; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < incrementsPerThread; j++) {
                    counter.increment();
                }
            });
            threads[i].start();
        }
        
        for (Thread thread : threads) {
            thread.join();
        }
        
        System.out.println("Expected: " + (threadCount * incrementsPerThread));
        System.out.println("Actual: " + counter.get());
    }
}
```

### 6.2 实现一个简单的自旋锁

```java
import java.util.concurrent.atomic.AtomicReference;

public class SimpleSpinLock {
    private AtomicReference<Thread> owner = new AtomicReference<>();
    
    public void lock() {
        Thread currentThread = Thread.currentThread();
        // 自旋直到获取锁
        while (!owner.compareAndSet(null, currentThread)) {
            // 可以添加一些退避策略，如Thread.yield()
        }
    }
    
    public void unlock() {
        Thread currentThread = Thread.currentThread();
        owner.compareAndSet(currentThread, null);
    }
    
    public static void main(String[] args) {
        final SimpleSpinLock lock = new SimpleSpinLock();
        final int[] count = {0};
        
        Runnable runnable = () -> {
            for (int i = 0; i < 10000; i++) {
                lock.lock();
                try {
                    count[0]++;
                } finally {
                    lock.unlock();
                }
            }
        };
        
        Thread t1 = new Thread(runnable);
        Thread t2 = new Thread(runnable);
        
        t1.start();
        t2.start();
        
        try {
            t1.join();
            t2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        System.out.println("Final count: " + count[0]);
    }
}
```

## 7. 总结

CAS是Java并发编程中的一种重要机制，它通过硬件原语实现了无锁的原子操作，在许多场景下可以替代传统的锁机制，提高并发性能。CAS的核心思想是乐观并发控制，它假设冲突很少发生，只在数据真正被修改时才采取措施。

虽然CAS存在ABA问题、自旋开销大、只能保证单变量原子操作等局限性，但通过合理的设计和使用，这些问题都可以得到有效解决。在实际应用中，我们应该根据具体场景选择合适的并发控制机制，在低竞争环境下优先考虑CAS，在高竞争或复杂操作场景下考虑传统锁机制。

理解和掌握CAS机制，对于编写高效的并发程序至关重要。希望本文能帮助读者深入理解CAS的原理和应用，为并发编程实践提供指导。

## 参考资料

1. 《Java并发编程实战》
2. 《深入理解Java虚拟机》
3. Oracle官方文档：java.util.concurrent.atomic包
4. Doug Lea. The java.util.concurrent Synchronizer Framework.

---

本文详细介绍了 CAS 详解，希望对您有所帮助。如果您有任何问题，欢迎在评论区讨论！