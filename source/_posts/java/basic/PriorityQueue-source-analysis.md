---
title: 【Java】PriorityQueue优先队列核心源码详解
tags:
  - Java集合
  - 源码分析
categories:
  - Java
  - Java基础
abbrlink: 8f3e2c7a
date: 2024-03-04 11:30:00
---

## 一、底层数据结构与设计原理

### 1.1 基本概念

PriorityQueue 是 Java 集合框架中的优先队列实现，它是一个基于优先级堆的无界队列。其核心特性包括：

1. **优先级排序**：元素按照自然顺序或自定义比较器排序
2. **堆实现**：底层使用二叉堆数据结构
3. **非线程安全**：不支持并发访问
4. **不允许null**：不允许插入null元素

### 1.2 底层数据结构

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {

    // 默认初始容量
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    // 存储元素的数组
    transient Object[] queue;

    // 优先队列中的元素个数
    private int size = 0;

    // 比较器
    private final Comparator<? super E> comparator;
}
```

## 二、构造方法解析

```java
// 创建一个默认初始容量(11)的优先队列
public PriorityQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}

// 创建指定初始容量的优先队列
public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}

// 创建指定初始容量和比较器的优先队列
public PriorityQueue(int initialCapacity,
                     Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
```

## 三、核心实现机制

### 3.1 二叉堆实现

PriorityQueue 使用完全二叉树来实现二叉堆，并用数组表示这个完全二叉树。对于数组中的任意位置 k：

- 父节点位置：(k - 1) >>> 1
- 左子节点位置：2 * k + 1
- 右子节点位置：2 * k + 2

这种实现方式使得：
1. 空间效率高：不需要存储节点之间的引用
2. 定位效率高：可以通过简单的数学计算找到任意节点的父节点和子节点
3. 维护成本低：只需要在添加和删除时进行上浮或下沉操作

## 四、核心方法源码解析

### 4.1 添加元素（offer方法）

```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    // 检查容量是否需要扩容
    if (i >= queue.length)
        grow(i + 1);
    size = i + 1;
    // 如果是第一个元素，直接放入
    if (i == 0)
        queue[0] = e;
    else
        // 否则，向上调整堆
        siftUp(i, e);
    return true;
}

// 向上调整实现
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}

private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        // 找到父节点位置
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        // 如果当前值大于父节点，结束循环
        if (key.compareTo((E) e) >= 0)
            break;
        // 否则交换位置
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}
```

### 4.2 获取并删除堆顶元素（poll方法）

```java
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];
    E x = (E) queue[s];
    queue[s] = null;
    if (s != 0)
        // 向下调整堆
        siftDown(0, x);
    return result;
}

// 向下调整实现
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}

private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    int half = size >>> 1;
    while (k < half) {
        // 找到左子节点
        int child = (k << 1) + 1;
        Object c = queue[child];
        int right = child + 1;
        // 如果右子节点更小，则选择右子节点
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        // 如果当前值小于子节点值，结束循环
        if (key.compareTo((E) c) <= 0)
            break;
        // 否则交换位置
        queue[k] = c;
        k = child;
    }
    queue[k] = key;
}
```

## 五、性能分析与使用建议

### 5.1 时间复杂度

PriorityQueue 的主要操作时间复杂度如下：
- 插入操作（offer）：O(log n)
- 删除操作（poll）：O(log n)
- 获取堆顶元素（peek）：O(1)
- 查找元素（contains）：O(n)

### 5.2 使用建议

1. **场景选择**
   - 任务调度：按优先级处理任务
   - 图算法：如Dijkstra最短路径算法
   - 事件处理：按时间顺序处理事件

2. **注意事项**
   - 非线程安全，并发场景需要外部同步
   - 不允许插入null元素
   - 元素必须可比较（实现Comparable或提供Comparator）
   - 遍历顺序不一定是优先级顺序

### 5.3 使用示例

```java
// 创建优先队列
PriorityQueue<Integer> pq = new PriorityQueue<>();

// 添加元素
pq.offer(5);
pq.offer(2);
pq.offer(8);
pq.offer(1);

// 按优先级顺序处理元素
while (!pq.isEmpty()) {
    System.out.print(pq.poll() + " "); // 输出：1 2 5 8
}
```

## 六、扩容机制详解

### 6.1 扩容触发条件

PriorityQueue 在添加元素时，如果当前元素数量达到或超过数组容量，就会触发扩容操作：

```java
public boolean offer(E e) {
    // 省略其他代码...
    int i = size;
    // 检查容量是否需要扩容
    if (i >= queue.length)
        grow(i + 1);
    // 省略其他代码...
}
```

### 6.2 grow方法实现

```java
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // 如果旧容量小于64，扩容为旧容量的2倍 + 2
    // 如果旧容量大于等于64，扩容为旧容量的1.5倍
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                     (oldCapacity + 2) :
                                     (oldCapacity >> 1));
    // 检查是否溢出
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    queue = Arrays.copyOf(queue, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // 溢出
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

### 6.3 扩容策略分析

PriorityQueue 的扩容策略有以下特点：

1. **分段扩容比例**：
   - 当容量小于64时：新容量 = 旧容量 * 2 + 2
   - 当容量大于等于64时：新容量 = 旧容量 * 1.5

2. **容量上限处理**：
   - 最大容量为 `Integer.MAX_VALUE`
   - 通过 `hugeCapacity` 方法处理接近上限时的情况

3. **数组复制**：
   - 使用 `Arrays.copyOf` 创建新数组并复制元素
   - 这是一个相对昂贵的操作，尤其是队列较大时

这种扩容策略在小容量时增长较快，大容量时增长较慢，平衡了空间利用率和扩容频率。

## 七、参考资料

1. [Oracle官方文档 - PriorityQueue](https://docs.oracle.com/javase/8/docs/api/java/util/PriorityQueue.html)
2. 《Java编程思想（第4版）》第17章 容器深入研究
3. 《Effective Java（第3版）》第47条：优先使用集合而非数组
4. [Java Collections Framework官方指南](https://docs.oracle.com/javase/tutorial/collections/index.html)
5. [PriorityQueue源码分析（JDK 17）](https://hg.openjdk.java.net/jdk-updates/jdk17u/file/tip/src/java.base/share/classes/java/util/PriorityQueue.java)

## 八、讨论与交流

如果您对本文有任何疑问、建议或发现了错误，欢迎在评论区留言讨论。我们可以一起探讨PriorityQueue的实现细节、性能优化或在实际项目中的应用经验。