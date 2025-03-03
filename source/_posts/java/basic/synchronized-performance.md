---
title: Java synchronized 性能优化全面解析
date: 2024-01-09 14:30:00
tags:
  - Java
  - 并发编程
  - 性能优化
categories: Java
abbrlink: e55fb501
---

# Java synchronized 性能优化全面解析

在Java并发编程中，synchronized 是最基础也是最重要的同步机制之一。虽然它使用方便，但在高并发场景下可能会成为性能瓶颈。本文将深入分析 synchronized 的实现原理，并介绍各种优化手段。

## synchronized 的实现原理

### 对象头与 Mark Word

synchronized 的实现依赖于对象头中的 Mark Word。在64位JVM中，Mark Word 占用8个字节，其存储内容会随着锁的状态变化而改变：

- 无锁状态：存储对象的 hashCode、分代年龄等
- 偏向锁：记录持有锁的线程ID
- 轻量级锁：指向线程栈中 Lock Record 的指针
- 重量级锁：指向互斥量（重量级锁）的指针

### 锁的升级过程

#### 1. 偏向锁

偏向锁是针对于一个线程多次获取同一个锁的情况。当一个线程访问同步块时，会在对象头中记录线程ID，下次该线程进入时可以直接获取锁，而无需进行CAS操作。

```java
public class BiasedLockExample {
    private static Object lock = new Object();
    
    public void biasedLock() {
        synchronized(lock) {
            // 同步块
        }
    }
}
```

#### 2. 轻量级锁

当有其他线程尝试获取偏向锁时，偏向锁就会升级为轻量级锁。轻量级锁采用CAS操作来获取锁，适用于线程交替执行同步块的场景。

```java
public class LightweightLockExample {
    private static Object lock = new Object();
    
    public void method1() {
        synchronized(lock) {
            // 线程1的操作
        }
    }
    
    public void method2() {
        synchronized(lock) {
            // 线程2的操作
        }
    }
}
```

#### 3. 重量级锁

当轻量级锁的CAS操作失败次数超过阈值时，锁会升级为重量级锁。重量级锁会导致线程阻塞和唤醒，涉及到操作系统的用户态和内核态切换，性能开销较大。

## synchronized 性能优化策略

### 1. 减小锁粒度

将大对象锁拆分成小对象锁，减少锁竞争。

```java
// 优化前
public class CoarseSync {
    private List<String> list1 = new ArrayList<>();
    private List<String> list2 = new ArrayList<>();
    
    public synchronized void addToList1(String item) {
        list1.add(item);
    }
    
    public synchronized void addToList2(String item) {
        list2.add(item);
    }
}

// 优化后
public class FineSync {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
    private List<String> list1 = new ArrayList<>();
    private List<String> list2 = new ArrayList<>();
    
    public void addToList1(String item) {
        synchronized(lock1) {
            list1.add(item);
        }
    }
    
    public void addToList2(String item) {
        synchronized(lock2) {
            list2.add(item);
        }
    }
}
```

### 2. 锁消除

JVM的逃逸分析可以判断同步块中的对象是否只能被一个线程访问，如果是，则可以消除锁。

```java
public class LockElimination {
    public String concatString(String s1, String s2) {
        StringBuffer sb = new StringBuffer();
        // StringBuffer的append方法是同步的
        // 但是sb对象不会被其他线程访问
        // JVM会自动消除锁
        sb.append(s1);
        sb.append(s2);
        return sb.toString();
    }
}
```

### 3. 锁粗化

如果一系列连续的同步操作都对同一个对象反复加锁和解锁，频繁的加锁操作会导致性能损耗。

```java
// 优化前
public class LockCoarsening {
    private static Object lock = new Object();
    
    public void method() {
        for(int i = 0; i < 100; i++) {
            synchronized(lock) {
                // 操作1
            }
            synchronized(lock) {
                // 操作2
            }
        }
    }
}

// 优化后
public class LockCoarseningOptimized {
    private static Object lock = new Object();
    
    public void method() {
        synchronized(lock) {
            for(int i = 0; i < 100; i++) {
                // 操作1
                // 操作2
            }
        }
    }
}
```

### 4. 适时使用其他锁

在某些场景下，可以考虑使用其他锁机制来替代 synchronized：

- ReentrantLock：可以实现公平锁、可中断锁
- ReadWriteLock：读多写少场景的优选
- StampedLock：JDK 8 引入的新锁，在读多写少场景下性能更优

```java
public class LockAlternative {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    private Map<String, String> map = new HashMap<>();
    
    public String read(String key) {
        readLock.lock();
        try {
            return map.get(key);
        } finally {
            readLock.unlock();
        }
    }
    
    public void write(String key, String value) {
        writeLock.lock();
        try {
            map.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

## 性能对比

下面是一个简单的性能测试，比较不同场景下的锁性能：

```java
public class LockPerformanceTest {
    private static final int THREAD_COUNT = 10;
    private static final int LOOP_COUNT = 100000;
    
    // 粗粒度锁
    private static void testCoarseLock() {
        CoarseSync coarse = new CoarseSync();
        long start = System.nanoTime();
        
        Thread[] threads = new Thread[THREAD_COUNT];
        for(int i = 0; i < THREAD_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for(int j = 0; j < LOOP_COUNT; j++) {
                    coarse.addToList1("item");
                    coarse.addToList2("item");
                }
            });
            threads[i].start();
        }
        
        // 等待所有线程完成
        for(Thread t : threads) {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        
        long end = System.nanoTime();
        System.out.println("Coarse lock time: " + (end - start) / 1000000 + "ms");
    }
    
    // 细粒度锁
    private static void testFineLock() {
        FineSync fine = new FineSync();
        long start = System.nanoTime();
        
        Thread[] threads = new Thread[THREAD_COUNT];
        for(int i = 0; i < THREAD_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for(int j = 0; j < LOOP_COUNT; j++) {
                    fine.addToList1("item");
                    fine.addToList2("item");
                }
            });
            threads[i].start();
        }
        
        // 等待所有线程完成
        for(Thread t : threads) {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        
        long end = System.nanoTime();
        System.out.println("Fine lock time: " + (end - start) / 1000000 + "ms");
    }
    
    public static void main(String[] args) {
        testCoarseLock();
        testFineLock();
    }
}
```

在10个线程同时执行10万次操作的测试中，细粒度锁的性能通常比粗粒度锁提升40%以上。

## 总结

要提高 synchronized 的性能，可以从以下几个方面入手：

1. 理解锁升级机制，合理利用偏向锁和轻量级锁
2. 减小锁粒度，避免锁竞争
3. 利用JVM的锁优化（锁消除、锁粗化）
4. 根据场景选择合适的锁机制

在实际应用中，应该根据具体场景和性能需求，选择合适的优化策略。同时要注意，过度优化可能会导致代码复杂度增加，应该在性能和可维护性之间找到平衡点。

---

希望这篇文章能帮助您解决synchronized性能优化问题。如果您有任何问题，欢迎在评论区讨论！