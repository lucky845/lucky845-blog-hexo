---
title: 【Java】ArrayBlockingQueue核心源码与阻塞队列实现机制详解
tags:
  - Java集合
  - 源码分析
  - 并发编程
categories:
  - Java
  - Java基础
abbrlink: 9e4d7f2a
date: 2025-03-21 12:00:00
---

## 一、底层数据结构与设计原理

### 1.1 基本概念

ArrayBlockingQueue是Java并发包中的一个实现，它是一个基于数组的有界阻塞队列，遵循FIFO（先进先出）原则。其核心特性包括：

1. **有界性**：创建时必须指定容量，一旦创建容量不可变
2. **阻塞操作**：当队列满时，入队操作阻塞；当队列空时，出队操作阻塞
3. **线程安全**：通过ReentrantLock保证并发安全
4. **公平性选择**：支持公平/非公平锁模式

### 1.2 底层数据结构

```java
// JDK 1.8 ArrayBlockingQueue部分源码
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    // 存储元素的数组
    final Object[] items;

    // 下一个待取出元素的索引
    int takeIndex;

    // 下一个可插入元素的索引
    int putIndex;

    // 队列中的元素数量
    int count;

    // 用于所有访问控制的锁
    final ReentrantLock lock;

    // 等待获取元素的条件队列
    private final Condition notEmpty;

    // 等待插入元素的条件队列
    private final Condition notFull;
}
```

## 二、构造方法解析

```java
// 创建指定容量的队列，默认非公平锁
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}

// 创建指定容量的队列，可选择公平/非公平锁
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull = lock.newCondition();
}

// 创建指定容量的队列，并用集合c初始化，可选择公平/非公平锁
public ArrayBlockingQueue(int capacity, boolean fair,
                          Collection<? extends E> c) {
    this(capacity, fair);
    final ReentrantLock lock = this.lock;
    lock.lock(); // 加锁确保可见性和原子性
    try {
        int i = 0;
        try {
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}
```

## 三、核心成员变量与锁机制

### 3.1 关键成员变量

- **items**：存储队列元素的数组，容量固定
- **takeIndex**：下一个出队元素的索引
- **putIndex**：下一个入队元素的索引
- **count**：当前队列中的元素数量
- **lock**：控制并发访问的ReentrantLock锁
- **notEmpty**：队列非空条件，用于通知等待获取元素的线程
- **notFull**：队列未满条件，用于通知等待插入元素的线程

### 3.2 锁与条件变量机制

```java
// 锁机制初始化
lock = new ReentrantLock(fair);
notEmpty = lock.newCondition();
notFull = lock.newCondition();
```

ArrayBlockingQueue使用单一的ReentrantLock来保护所有访问，并使用两个条件变量来管理线程等待状态：

1. **notEmpty条件**：当队列为空时，消费者线程在此条件上等待
2. **notFull条件**：当队列已满时，生产者线程在此条件上等待

这种设计确保了线程安全，同时通过条件变量实现了高效的线程等待和唤醒机制。

## 四、核心方法源码解析

### 4.1 入队操作

#### 4.1.1 put方法（阻塞）

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();  // 队列已满，等待空间
        enqueue(e);  // 入队
    } finally {
        lock.unlock();
    }
}

// 入队辅助方法
private void enqueue(E x) {
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;  // 循环队列，索引到达末尾时重置为0
    count++;
    notEmpty.signal();  // 唤醒等待获取元素的线程
}
```

#### 4.1.2 offer方法（非阻塞）

```java
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)
            return false;  // 队列已满，直接返回false
        else {
            enqueue(e);  // 入队
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```

#### 4.1.3 offer方法（超时版本）

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    checkNotNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length) {
            if (nanos <= 0)
                return false;  // 超时返回false
            nanos = notFull.awaitNanos(nanos);  // 等待指定时间
        }
        enqueue(e);  // 入队
        return true;
    } finally {
        lock.unlock();
    }
}
```

### 4.2 出队操作

#### 4.2.1 take方法（阻塞）

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();  // 队列为空，等待数据
        return dequeue();  // 出队
    } finally {
        lock.unlock();
    }
}

// 出队辅助方法
private E dequeue() {
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;  // 帮助GC
    if (++takeIndex == items.length)
        takeIndex = 0;  // 循环队列，索引到达末尾时重置为0
    count--;
    if (itrs != null)
        itrs.elementDequeued();  // 通知迭代器元素已出队
    notFull.signal();  // 唤醒等待插入元素的线程
    return x;
}
```

#### 4.2.2 poll方法（非阻塞）

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
```

#### 4.2.3 poll方法（超时版本）

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0)
                return null;  // 超时返回null
            nanos = notEmpty.awaitNanos(nanos);  // 等待指定时间
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

### 4.3 查看操作

```java
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex);  // 只查看不移除
    } finally {
        lock.unlock();
    }
}

// 获取指定位置元素的辅助方法
@SuppressWarnings("unchecked")
final E itemAt(int i) {
    return (E) items[i];
}
```

## 五、循环数组实现原理

ArrayBlockingQueue使用循环数组实现队列，这种设计有以下特点：

```java
// 入队索引循环
if (++putIndex == items.length)
    putIndex = 0;
    
// 出队索引循环
if (++takeIndex == items.length)
    takeIndex = 0;
```

1. **空间复用**：通过循环使用数组空间，避免了数据搬移
2. **边界处理**：当索引到达数组末尾时，自动重置为0继续使用
3. **高效实现**：入队和出队操作的时间复杂度均为O(1)

这种循环数组的实现使得ArrayBlockingQueue在有限的空间内能够高效地处理元素的添加和移除。

## 六、公平性与非公平性对比

ArrayBlockingQueue支持两种锁模式：

### 6.1 公平锁模式

```java
// 创建公平锁模式的队列
ArrayBlockingQueue<String> fairQueue = new ArrayBlockingQueue<>(100, true);
```

- **优点**：按照线程等待的时间顺序获取锁，避免线程饥饿
- **缺点**：需要维护一个有序队列，性能相对较低

### 6.2 非公平锁模式（默认）

```java
// 创建非公平锁模式的队列（默认）
ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<>(100);
```

- **优点**：吞吐量更高，性能更好
- **缺点**：可能导致某些线程长时间等待（饥饿）

## 七、性能特性与最佳实践

### 7.1 性能特点

1. **有界性能优势**：预分配固定大小的数组，避免动态扩容
2. **单锁设计**：所有操作共用一把锁，简化实现但可能成为性能瓶颈
3. **数组访问效率**：基于数组实现，随机访问效率高
4. **内存占用**：相比链表实现的队列，内存占用更紧凑

### 7.2 适用场景

- **生产者-消费者模型**：典型的阻塞队列应用场景
- **线程池工作队列**：用于存储等待执行的任务
- **数据缓冲区**：在异步处理系统中作为数据缓冲
- **流量控制**：利用有界特性进行流量整形和控制

### 7.3 使用建议

1. **合理设置容量**：根据生产和消费速率设置合适的队列容量
2. **选择合适的API**：根据需求选择阻塞(put/take)或非阻塞(offer/poll)方法
3. **注意锁竞争**：高并发场景下考虑使用LinkedBlockingQueue或ConcurrentLinkedQueue
4. **避免在锁内执行耗时操作**：减少锁持有时间，提高并发性

## 八、源码实现的关键设计思想

1. **单锁 + 双条件变量**：使用一个ReentrantLock和两个Condition实现线程协作
2. **循环数组**：通过循环索引实现高效的FIFO队列
3. **阻塞与非阻塞API**：提供多种操作方式满足不同需求
4. **公平性选择**：支持公平与非公平锁模式，满足不同场景需求

## 九、与其他阻塞队列的对比

### 9.1 ArrayBlockingQueue vs LinkedBlockingQueue

| 特性 | ArrayBlockingQueue | LinkedBlockingQueue |
| --- | --- | --- |
| 底层实现 | 循环数组 | 链表 |
| 容量限制 | 创建时固定 | 可选（默认Integer.MAX_VALUE） |
| 锁机制 | 单锁 | 双锁（分离的读写锁） |
| 内存占用 | 紧凑，预分配 | 按需分配，额外引用开销 |
| 适用场景 | 容量可预知，读写频率接近 | 容量不可预知，读写速率不匹配 |

### 9.2 ArrayBlockingQueue vs ConcurrentLinkedQueue

| 特性 | ArrayBlockingQueue | ConcurrentLinkedQueue |
| --- | --- | --- |
| 阻塞特性 | 支持阻塞操作 | 非阻塞（无等待） |
| 底层实现 | 数组 + 锁 | 链表 + CAS |
| 容量限制 | 有界 | 无界 |
| 性能特点 | 中等并发性能 | 高并发性能 |
| 适用场景 | 需要阻塞语义的场景 | 高吞吐、低延迟场景 |

### 9.3 ArrayBlockingQueue vs DelayQueue

| 特性 | ArrayBlockingQueue | DelayQueue |
| --- | --- | --- |
| 元素特性 | 普通元素 | 延迟元素（需实现Delayed接口） |
| 出队顺序 | FIFO | 按延迟时间排序 |
| 应用场景 | 普通生产-消费模型 | 定时任务、缓存过期 |

## 十、讨论与交流

### 10.1 常见问题解答

1. **为什么ArrayBlockingQueue使用单锁而不是分离锁？**
   - 设计简单性：单锁实现更简单，代码更易维护
   - 数组特性：数组结构使得头尾操作相互影响，分离锁收益有限
   - 避免死锁：单锁避免了复杂的死锁处理逻辑

2. **ArrayBlockingQueue是否线程安全？**
   - 是的，通过ReentrantLock保证了所有操作的线程安全性
   - 所有修改操作都在锁的保护下进行，确保了可见性和原子性

3. **如何选择合适的队列容量？**
   - 分析生产和消费速率，确保队列不会成为性能瓶颈
   - 考虑内存限制，避免过大的队列占用过多内存
   - 通常设置为生产速率与消费速率之差的峰值乘以服务时间

### 10.2 性能优化建议

1. **减少锁竞争**
   ```java
   // 使用多个队列分散负载
   ArrayBlockingQueue<Task>[] queues = new ArrayBlockingQueue[nThreads];
   for (int i = 0; i < nThreads; i++) {
       queues[i] = new ArrayBlockingQueue<>(capacity);
   }
   
   // 根据某种策略选择队列
   int index = task.hashCode() % nThreads;
   queues[index].put(task);
   ```

2. **批量操作**
   ```java
   // 使用drainTo批量处理元素
   List<Task> tasks = new ArrayList<>(batchSize);
   queue.drainTo(tasks, batchSize);
   for (Task task : tasks) {
       process(task);
   }
   ```

3. **合理使用超时方法**
   ```java
   // 避免无限等待
   Task task = queue.poll(timeout, TimeUnit.MILLISECONDS);
   if (task != null) {
       process(task);
   } else {
       // 处理超时情况
   }
   ```

### 10.3 参考资料

1. Java SE Documentation - [java.util.concurrent.ArrayBlockingQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ArrayBlockingQueue.html)
2. 《Java并发编程实战》- Brian Goetz等
3. 《Java并发编程的艺术》- 方腾飞等
4. Doug Lea - [《Concurrent Programming in Java》](http://gee.cs.oswego.edu/dl/cpj/index.html)
5. OpenJDK源码 - [ArrayBlockingQueue.java](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/concurrent/ArrayBlockingQueue.java)

通过本文的分析，我们深入了解了ArrayBlockingQueue的实现原理、性能特点和最佳实践。在实际应用中，根据具体场景选择合适的阻塞队列实现，可以显著提升系统的性能和可靠性。