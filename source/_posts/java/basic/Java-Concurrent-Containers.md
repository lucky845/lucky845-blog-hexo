---
title: 【Java】并发容器总结：线程安全的集合框架详解
tags:
  - Java集合
  - 并发编程
categories:
  - Java
  - Java基础
abbrlink: 7a8b905g
date: 2025-06-05 09:00:00
---

## 一、并发容器概述

Java并发容器是Java集合框架中专门为多线程环境设计的线程安全容器。它们通过不同的并发控制机制，确保在多线程访问时的数据一致性和线程安全性。

### 1.1 主要并发容器分类

1. **并发Map**
   - ConcurrentHashMap
   - ConcurrentSkipListMap

2. **并发Queue**
   - ConcurrentLinkedQueue
   - BlockingQueue接口及其实现类
   - TransferQueue接口及其实现类

3. **并发Set**
   - ConcurrentSkipListSet
   - CopyOnWriteArraySet

4. **并发List**
   - CopyOnWriteArrayList

## 二、ConcurrentHashMap详解

### 2.1 核心特性

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    
    // 默认初始容量
    private static final int DEFAULT_CAPACITY = 16;
    
    // 默认并发级别
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    
    // 默认加载因子
    private static final float DEFAULT_LOAD_FACTOR = 0.75f;
}
```

ConcurrentHashMap的主要特点：

1. **分段锁机制**：JDK 1.7采用Segment分段锁
2. **CAS + synchronized**：JDK 1.8采用CAS + synchronized保证并发安全
3. **并发度可调**：支持自定义并发级别
4. **弱一致性**：迭代器不会抛出ConcurrentModificationException

### 2.2 核心方法实现

```java
// 插入或更新元素
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            synchronized (f) {
                // 链表或红黑树的插入逻辑
            }
        }
    }
    return null;
}
```

## 三、BlockingQueue详解

### 3.1 主要实现类

1. **ArrayBlockingQueue**：有界阻塞队列
2. **LinkedBlockingQueue**：可选有界阻塞队列
3. **PriorityBlockingQueue**：无界优先级阻塞队列
4. **DelayQueue**：延迟队列
5. **SynchronousQueue**：同步队列

### 3.2 核心方法

```java
public interface BlockingQueue<E> extends Queue<E> {
    // 添加元素，如果队列满则阻塞
    void put(E e) throws InterruptedException;
    
    // 获取元素，如果队列空则阻塞
    E take() throws InterruptedException;
    
    // 添加元素，如果队列满则返回false
    boolean offer(E e);
    
    // 获取元素，如果队列空则返回null
    E poll();
}
```

## 四、CopyOnWriteArrayList详解

### 4.1 实现原理

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    
    // 使用volatile保证可见性
    private transient volatile Object[] array;
    
    // 写操作时加锁
    final transient ReentrantLock lock = new ReentrantLock();
}
```

CopyOnWriteArrayList的特点：

1. **写时复制**：修改时复制整个数组
2. **读操作无锁**：读操作不需要加锁
3. **弱一致性**：迭代器不会抛出ConcurrentModificationException
4. **适合读多写少**：写操作性能较差

### 4.2 核心方法实现

```java
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
```

## 五、并发容器的使用建议

### 5.1 选择建议

1. **Map的选择**
   - 需要高并发：使用ConcurrentHashMap
   - 需要排序：使用ConcurrentSkipListMap

2. **Queue的选择**
   - 需要阻塞操作：使用BlockingQueue
   - 需要优先级：使用PriorityBlockingQueue
   - 需要延迟处理：使用DelayQueue

3. **List的选择**
   - 读多写少：使用CopyOnWriteArrayList
   - 写多读少：使用Collections.synchronizedList

### 5.2 性能优化建议

1. **合理设置并发级别**
   ```java
   // 根据实际并发量设置并发级别
   ConcurrentHashMap<String, Object> map = 
       new ConcurrentHashMap<>(16, 0.75f, 32);
   ```

2. **避免频繁扩容**
   ```java
   // 预估容量，避免频繁扩容
   BlockingQueue<String> queue = 
       new ArrayBlockingQueue<>(1000);
   ```

3. **合理使用批量操作**
   ```java
   // 批量操作减少锁竞争
   map.putAll(anotherMap);
   ```

## 六、常见问题与解决方案

### 6.1 死锁问题

```java
// 错误示例：可能发生死锁
synchronized (map1) {
    synchronized (map2) {
        // 操作
    }
}

// 正确示例：使用ConcurrentHashMap避免死锁
ConcurrentHashMap<String, Object> map = new ConcurrentHashMap<>();
```

### 6.2 性能问题

1. **避免频繁的写操作**
   ```java
   // 错误示例：频繁写操作
   for (int i = 0; i < 1000; i++) {
       list.add(i);
   }
   
   // 正确示例：批量操作
   list.addAll(Arrays.asList(0, 1, 2, ..., 999));
   ```

2. **合理使用迭代器**
   ```java
   // 错误示例：在迭代过程中修改
   for (String s : list) {
       list.remove(s);  // 可能抛出ConcurrentModificationException
   }
   
   // 正确示例：使用迭代器的remove方法
   Iterator<String> it = list.iterator();
   while (it.hasNext()) {
       String s = it.next();
       it.remove();
   }
   ```

## 七、总结

Java并发容器通过不同的并发控制机制，为多线程环境提供了线程安全的集合实现。选择合适的并发容器，可以显著提高程序的并发性能和可靠性。

主要特点：

1. **线程安全**：通过锁机制或CAS保证线程安全
2. **高性能**：针对不同场景优化
3. **弱一致性**：迭代器不会抛出ConcurrentModificationException
4. **可扩展性**：支持自定义并发级别和容量

## 八、参考资料

1. **官方文档**
   - [Java SE 17 API Documentation - Concurrent Collections](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/package-summary.html)

2. **书籍**
   - Brian Goetz. *Java Concurrency in Practice*. Addison-Wesley Professional, 2006.
   - Doug Lea. *Concurrent Programming in Java*. Addison-Wesley Professional, 1999.

3. **技术博客**
   - [Java并发编程实战](https://www.infoq.cn/article/java-concurrency-programming)
   - [深入理解Java并发容器](https://www.jianshu.com/p/c0642afe03e0)

## 九、讨论与交流

如果您对本文有任何疑问、建议或发现了错误，欢迎在评论区留言讨论。我们可以一起探讨Java并发容器的实现细节、性能优化或在实际项目中的应用经验。 