---
title: 【Java】CopyOnWriteArrayList核心源码解析与写时复制机制详解
tags:
  - Java集合
  - 源码分析
  - 并发编程
categories:
  - Java
  - Java基础
abbrlink: 8d7c2f3e
date: 2025-03-20 14:00:00
---

## 一、写时复制机制原理

### 1.1 设计思想
通过写时复制（Copy-On-Write）策略实现并发安全，所有修改操作都会创建底层数组的新副本。这种设计保证：
1. 读操作完全无锁，直接访问当前数组引用
2. 写操作通过ReentrantLock保证原子性
3. 迭代器使用创建时的数组快照，避免ConcurrentModificationException

### 1.2 适用场景
适用于读操作远多于写操作的场景，例如：
- 事件监听器列表管理
- 黑白名单等配置数据的读取
- 需要保证数据最终一致性的场景

不适用于写操作频繁或数据实时性要求高的场景

## 二、核心数据结构

```java
// 使用volatile保证数组引用的可见性
private transient volatile Object[] array;

final transient ReentrantLock lock = new ReentrantLock();
```

## 三、并发控制实现

### 3.1 写操作加锁机制
所有修改操作前必须获取独占锁：
```java
final ReentrantLock lock = this.lock;
lock.lock();
try {
    // 修改操作
} finally {
    lock.unlock();
}
```
保证同一时刻只有一个线程执行修改操作

### 3.2 原子引用更新
volatile修饰的array引用保证：
1. 写线程修改数组引用后，读线程立即可见
2. 配合Unsafe.putObjectVolatile实现原子引用更新
3. 避免指令重排序导致的可见性问题

## 四、关键方法源码解析

### 4.1 add方法实现
```java
public boolean add(E e) {
    public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
}
```

### 4.2 remove方法实现
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = elementAt(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index, numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}

## 五、迭代器实现特点

### 5.1 快照迭代器
迭代器持有创建时的数组快照：
```java
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);
}
```
写操作不会影响已创建的迭代器，但可能导致迭代器数据过时

## 六、性能对比测试

| 操作类型 | 时间复杂度 | 线程安全 |
|---------|------------|----------|
| 读操作 | O(1)       | 无锁访问 |
| 写操作 | O(n)       | 完全同步 |

## 七、最佳实践建议

1. 集合大小保持较小（建议不超过1KB）
2. 写操作完成后及时释放资源
3. 配合版本号实现数据最终一致性检查
4. 避免在循环中执行批量写操作

示例用法：
```java
CopyOnWriteArrayList<String> configList = new CopyOnWriteArrayList<>();
// 写操作
configList.add("new_config"); 
// 读操作
for(String config : configList) {
    processConfig(config);
}
```

## 八、参考资料

1. [Oracle官方文档 - CopyOnWriteArrayList](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CopyOnWriteArrayList.html)
2. 《Java并发编程实战》第5章 并发容器
3. 《Java性能权威指南》第9章 并发编程性能调优

## 九、技术讨论

如果您对本文有任何疑问、建议或发现了错误，欢迎在评论区留言讨论。我们可以一起探讨CopyOnWriteArrayList的实现细节、性能优化或在实际项目中的应用经验。

