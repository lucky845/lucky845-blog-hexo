---
title: 【Java】JMM内存模型详解
date: 2025-3-29 15:00:00
tags:
  - Java
  - 并发编程
  - JMM
  - 内存模型
categories: 
  - Java
  - 技术笔记
  - Java基础
abbrlink: 9e7d3f2a
keywords: Java,JMM,内存模型,volatile,happens-before,内存屏障,原子性,可见性,有序性
description: 本文详细介绍了Java内存模型(JMM)的核心概念、工作原理、三大特性(原子性、可见性、有序性)、happens-before规则、内存屏障以及volatile关键字的实现机制，帮助读者深入理解Java并发编程中的内存一致性问题。
---

# Java内存模型(JMM)详解

## 前言

在多线程编程中，由于线程间的通信方式、缓存一致性问题以及编译器和处理器的优化，导致程序执行的结果可能与预期不符。Java内存模型(Java Memory Model，简称JMM)就是为了解决这些问题而提出的。本文将深入探讨JMM的核心概念、工作原理以及在并发编程中的应用，帮助读者更好地理解和应用Java并发编程。

## 1. JMM基本概念

### 1.1 什么是Java内存模型

Java内存模型(JMM)是一种规范，它定义了Java虚拟机(JVM)在计算机内存(RAM)中的工作方式。JMM规定了一个线程如何以及何时可以看到由其他线程修改过的共享变量的值，以及在必要时如何同步地访问共享变量。

JMM的主要目标是定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。

### 1.2 JMM的抽象结构

JMM定义了一个抽象的内存模型，屏蔽了各种硬件和操作系统的内存访问差异：

```
       ┌─────────────┐          ┌─────────────┐
       │  线程A       │          │  线程B      │
       └─────────────┘          └─────────────┘
             │                        │
             ▼                        ▼
       ┌─────────────┐          ┌─────────────┐
       │ 本地内存A    │          │ 本地内存B   │
       │(工作内存)    │          │(工作内存)   │
       └─────────────┘          └─────────────┘
             │                        │
             │                        │
             ▼                        ▼
       ┌─────────────────────────────────────┐
       │            主内存                    │
       │  (存储Java对象，包括成员变量等)         │
       └─────────────────────────────────────┘
```

在JMM中：

1. **主内存(Main Memory)**：所有线程共享的内存区域，存储所有的变量(包括实例变量和静态变量，不包括局部变量和方法参数)。
2. **工作内存(Working Memory)**：每个线程都有自己的工作内存，线程对变量的所有操作(读取、赋值等)都必须在工作内存中进行，而不能直接操作主内存中的变量。
3. **线程间通信**：线程之间的变量值传递需要通过主内存来完成。

### 1.3 JMM与硬件内存架构的关系

JMM与计算机硬件内存架构之间存在一定的差异：

- 硬件内存架构没有区分线程栈和堆，但JMM抽象了主内存与工作内存的概念
- 硬件内存架构中，所有的共享变量都存在主内存中
- 每个CPU都有自己的缓存，JMM中的工作内存可以类比为CPU的缓存

## 2. JMM的三大特性

### 2.1 原子性(Atomicity)

原子性是指一个操作是不可中断的，要么全部执行成功，要么全部执行失败，不存在部分执行的情况。

在Java中，对基本数据类型的读取和赋值操作是原子性的(除了long和double类型的非volatile变量)。但是像i++这样的操作，它包括读取变量的值、加1、写回主内存三个步骤，这个过程不是原子性的。

**Java提供的原子性保证：**

1. 基本数据类型的读写操作(除了long和double)是原子性的
2. 所有引用类型的读写操作是原子性的
3. volatile修饰的long和double变量的读写是原子性的
4. 原子性操作可以通过synchronized和Lock来保证

```java
public class AtomicityExample {
    private int count = 0;
    
    // 非原子操作
    public void increment() {
        count++; // 读取-修改-写入，非原子操作
    }
    
    // 使用synchronized保证原子性
    public synchronized void safeIncrement() {
        count++;
    }
    
    // 使用AtomicInteger保证原子性
    private AtomicInteger atomicCount = new AtomicInteger(0);
    public void atomicIncrement() {
        atomicCount.incrementAndGet();
    }
}
```

### 2.2 可见性(Visibility)

可见性是指当一个线程修改了共享变量的值，其他线程能够立即得知这个修改。在多核CPU中，每个CPU都有自己的缓存，当多个线程在不同的CPU上运行时，可能会导致缓存不一致的问题。

**Java提供的可见性保证：**

1. volatile关键字可以保证变量的可见性
2. synchronized和Lock在释放锁之前会将工作内存中的变量值刷新到主内存
3. final关键字可以保证初始化完成的对象对其他线程可见

```java
public class VisibilityExample {
    // 不保证可见性
    private boolean flag = false;
    
    // 使用volatile保证可见性
    private volatile boolean volatileFlag = false;
    
    public void writer() {
        flag = true; // 修改后对其他线程不一定立即可见
        volatileFlag = true; // 修改后对其他线程立即可见
    }
    
    public void reader() {
        // flag可能读取到旧值
        while (!flag) {
            // 循环等待
        }
        
        // volatileFlag一定能读取到最新值
        while (!volatileFlag) {
            // 循环等待
        }
    }
}
```

### 2.3 有序性(Ordering)

有序性是指程序执行的顺序按照代码的先后顺序执行。在JMM中，为了提高性能，编译器和处理器常常会对指令进行重排序，导致程序实际执行的顺序可能与代码顺序不同。

指令重排序的类型：
1. **编译器优化重排序**：编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序
2. **指令级并行重排序**：现代处理器采用了指令级并行技术来将多条指令重叠执行
3. **内存系统重排序**：由于处理器使用缓存和读写缓冲区，使得加载和存储操作看上去可能是在乱序执行

**Java提供的有序性保证：**

1. volatile关键字禁止指令重排序
2. synchronized和Lock保证同一时刻只有一个线程执行同步代码，相当于是让线程顺序执行同步代码
3. happens-before规则保证特定情况下的有序性

```java
public class OrderingExample {
    private int a = 0;
    private boolean flag = false;
    
    // 可能发生指令重排序
    public void writer() {
        a = 1;          // 1
        flag = true;    // 2
        // 1和2可能会重排序
    }
    
    // 使用volatile防止指令重排序
    private int b = 0;
    private volatile boolean volatileFlag = false;
    
    public void safeWriter() {
        b = 1;              // 3
        volatileFlag = true; // 4
        // 由于volatileFlag是volatile变量，3一定会在4之前执行
    }
}
```

## 3. Happens-Before原则

Happens-Before是JMM中非常重要的概念，它定义了两个操作之间的内存可见性。如果操作A happens-before 操作B，那么操作A的结果对操作B是可见的，即操作A的执行结果将对操作B可见，且A的执行顺序排在B之前。

### 3.1 Happens-Before的规则

1. **程序顺序规则**：在一个线程内，按照程序代码顺序，前面的操作happens-before后面的操作
2. **监视器锁规则**：对一个锁的解锁happens-before于随后对这个锁的加锁
3. **volatile变量规则**：对一个volatile变量的写操作happens-before于后面对这个变量的读操作
4. **传递性规则**：如果A happens-before B，且B happens-before C，那么A happens-before C
5. **线程启动规则**：Thread对象的start()方法happens-before于此线程的每一个动作
6. **线程终止规则**：线程中的所有操作都happens-before于其他线程检测到该线程已经终止
7. **线程中断规则**：对线程interrupt()方法的调用happens-before于被中断线程的代码检测到中断事件的发生
8. **对象终结规则**：一个对象的初始化完成happens-before于它的finalize()方法的开始

```java
public class HappenBeforeExample {
    private int value = 0;
    private volatile boolean flag = false;
    
    public void writer() {
        value = 42;    // 1
        flag = true;   // 2
    }
    
    public void reader() {
        if (flag) {    // 3
            // 由于volatile的happens-before规则，如果3读取到flag为true
            // 那么1对value的写入对4的读取一定可见
            int r = value;  // 4 - 这里一定能读到42
        }
    }
}
```

## 4. 内存屏障

内存屏障(Memory Barrier)是一种CPU指令，用于控制特定条件下的重排序和内存可见性。JMM使用内存屏障来实现volatile的内存语义。

### 4.1 内存屏障的分类

1. **LoadLoad屏障**：确保Load1数据的装载先于Load2及后续装载指令的装载
2. **StoreStore屏障**：确保Store1数据对其他处理器可见（刷新到内存）先于Store2及后续存储指令的存储
3. **LoadStore屏障**：确保Load1数据装载先于Store2及后续的存储指令刷新到内存
4. **StoreLoad屏障**：确保Store1数据对其他处理器变得可见（指刷新到内存）先于Load2及后续装载指令的装载

### 4.2 内存屏障与volatile的实现

JMM针对编译器制定的volatile重排序规则：

1. 当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序
2. 当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序
3. 当第一个操作是volatile写，第二个操作是volatile读时，不能重排序

在实现上，volatile的内存语义通过内存屏障来保证：

- 在每个volatile写操作前插入StoreStore屏障
- 在每个volatile写操作后插入StoreLoad屏障
- 在每个volatile读操作后插入LoadLoad屏障
- 在每个volatile读操作后插入LoadStore屏障

```
// volatile写
StoreStore屏障
volatile写操作
StoreLoad屏障

// volatile读
volatile读操作
LoadLoad屏障
LoadStore屏障
```

## 5. volatile关键字详解

volatile是Java提供的一种轻量级的同步机制，它保证了变量的可见性和有序性，但不保证原子性。

### 5.1 volatile的内存语义

1. **可见性**：对一个volatile变量的写，总是能立即被其他线程看到
2. **有序性**：禁止指令重排序优化

### 5.2 volatile的适用场景

1. **状态标志**：用作状态标志，标示某种状态是否发生变化
2. **双重检查锁定**：在单例模式的双重检查锁定中，使用volatile修饰单例对象
3. **独立观察**：一个线程修改值，其他线程通过读取这个值来感知变化

### 5.3 volatile的局限性

1. 不能保证复合操作的原子性，如i++
2. 不适合做计数器，因为不保证原子性

```java
public class VolatileExample {
    // 用作状态标志
    private volatile boolean flag = false;
    
    public void setFlag() {
        flag = true;
    }
    
    public void doIfFlagSet() {
        if (flag) {
            // 执行操作
        }
    }
    
    // 双重检查锁定单例模式
    private static volatile VolatileExample instance;
    
    public static VolatileExample getInstance() {
        if (instance == null) { // 第一次检查
            synchronized (VolatileExample.class) {
                if (instance == null) { // 第二次检查
                    instance = new VolatileExample();
                }
            }
        }
        return instance;
    }
}
```

## 6. 实际应用中的JMM问题

### 6.1 单例模式与JMM

在单例模式的实现中，如果不正确处理JMM相关问题，可能会导致线程安全问题：

```java
public class Singleton {
    private static Singleton instance;
    
    private Singleton() {}
    
    // 非线程安全的懒汉式
    public static Singleton getInstance() {
        if (instance == null) { // 可能多个线程同时进入
            instance = new Singleton();
        }
        return instance;
    }
    
    // 线程安全的双重检查锁定
    private static volatile Singleton safeInstance;
    
    public static Singleton getSafeInstance() {
        if (safeInstance == null) {
            synchronized (Singleton.class) {
                if (safeInstance == null) {
                    safeInstance = new Singleton();
                    // 上面这行代码实际上包含三个步骤：
                    // 1. 分配内存空间
                    // 2. 初始化对象
                    // 3. 将引用指向内存空间
                    // 如果没有volatile，2和3可能会重排序，导致其他线程看到一个未完全初始化的对象
                }
            }
        }
        return safeInstance;
    }
}
```

### 6.2 并发容器与JMM

Java中的并发容器，如ConcurrentHashMap，通过合理使用volatile和CAS等机制，在不加锁的情况下实现了线程安全：

```java
public class ConcurrentHashMapExample {
    public void example() {
        ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
        map.put("key", "value"); // 线程安全的操作
        String value = map.get("key"); // 线程安全的操作
    }
}
```

### 6.3 无锁编程与JMM

使用AtomicInteger等原子类可以在不使用锁的情况下实现线程安全的计数器：

```java
public class AtomicExample {
    private AtomicInteger counter = new AtomicInteger(0);
    
    public void increment() {
        counter.incrementAndGet(); // 原子操作，线程安全
    }
    
    public int getCount() {
        return counter.get();
    }
}
```

## 7. JMM与多线程开发最佳实践

### 7.1 正确使用同步机制

1. 优先使用java.util.concurrent包中的并发工具类
2. 其次考虑使用synchronized和volatile关键字
3. 尽量避免使用低级别的同步原语，如Thread.sleep()、Thread.yield()等

### 7.2 避免共享可变状态

1. 尽量使用不可变对象
2. 使用线程封闭技术，如ThreadLocal
3. 使用线程安全的集合类

### 7.3 正确发布对象

1. 在构造函数返回前不要暴露this引用
2. 使用工厂方法安全发布对象
3. 使用final字段保证初始化安全性

```java
public class SafePublication {
    // 使用final字段保证初始化安全性
    private final int value;
    
    private SafePublication(int value) {
        this.value = value;
    }
    
    // 使用工厂方法安全发布对象
    public static SafePublication create(int value) {
        return new SafePublication(value);
    }
}
```

## 总结

Java内存模型(JMM)是Java并发编程的基础，它定义了线程如何与内存交互以及线程之间如何通信。理解JMM对于编写正确的并发程序至关重要。

本文详细介绍了JMM的基本概念、三大特性(原子性、可见性、有序性)、happens-before规则、内存屏障以及volatile关键字的实现机制。通过这些知识，我们可以更好地理解Java并发编程中的内存一致性问题，从而编写出更加高效、安全的并发程序。

在实际开发中，我们应该遵循多线程开发的最佳实践，正确使用同步机制，避免共享可变状态，并确保对象的安全发布。只有这样，才能充分利用多核处理器的优势，同时避免并发编程中的各种陷阱。