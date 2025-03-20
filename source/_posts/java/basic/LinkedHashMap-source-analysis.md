---
title: 【Java】LinkedHashMap核心源码解析与实现原理详解
tags:
  - Java集合
  - 源码分析
categories:
  - Java
  - Java基础
abbrlink: 7a8b905g
date: 2025-03-20 12:00:00
---

## 一、底层数据结构

LinkedHashMap是HashMap的子类，它在HashMap的基础上，通过维护一个双向链表，保证了元素的插入顺序或访问顺序：

```java
// JDK 1.8 LinkedHashMap部分源码
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> {
    // 双向链表的头节点
    transient LinkedHashMap.Entry<K,V> head;
    
    // 双向链表的尾节点
    transient LinkedHashMap.Entry<K,V> tail;
    
    // 是否按访问顺序排序，true表示按访问顺序，false表示按插入顺序
    final boolean accessOrder;
    
    // 默认构造函数，按插入顺序排序
    public LinkedHashMap() {
        super();
        accessOrder = false;
    }
    
    // 指定初始容量的构造函数，按插入顺序排序
    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }
    
    // 指定初始容量和负载因子的构造函数，按插入顺序排序
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }
    
    // 指定初始容量、负载因子和排序模式的构造函数
    public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
    
    // 使用已有Map构造，按插入顺序排序
    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super();
        accessOrder = false;
        putMapEntries(m, false);
    }
}
```

### 1.1 Entry节点结构

LinkedHashMap的核心是其Entry节点，它继承自HashMap.Node，并增加了before和after引用，用于维护双向链表：

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after; // 双向链表的前驱和后继节点引用
    
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

这种设计使得LinkedHashMap同时拥有HashMap的高效查找特性和链表的有序遍历特性：

1. **哈希表结构**：继承自HashMap，使用哈希表进行元素存储，保证了O(1)的查找效率
2. **双向链表结构**：通过Entry节点的before和after引用，维护了元素的插入顺序或访问顺序

## 二、关键成员变量

| 变量名 | 类型 | 作用 |
|--------|------|------|
| head | Entry<K,V> | 双向链表的头节点 |
| tail | Entry<K,V> | 双向链表的尾节点 |
| accessOrder | boolean | 决定迭代顺序，true表示访问顺序，false表示插入顺序 |

其中，accessOrder是LinkedHashMap的关键特性：
- 当accessOrder为false时（默认值），元素按照插入顺序排列
- 当accessOrder为true时，元素按照访问顺序排列，最近访问的元素会移动到链表末尾

## 三、核心方法源码解析

### 3.1 链表维护方法

```java
// 将节点链接到双向链表的尾部
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}

// 从双向链表中移除节点
private void transferLinks(LinkedHashMap.Entry<K,V> src, LinkedHashMap.Entry<K,V> dst) {
    LinkedHashMap.Entry<K,V> b = dst.before = src.before;
    LinkedHashMap.Entry<K,V> a = dst.after = src.after;
    if (b == null)
        head = dst;
    else
        b.after = dst;
    if (a == null)
        tail = dst;
    else
        a.before = dst;
}
```

### 3.2 重写HashMap的回调方法

LinkedHashMap重写了HashMap的三个关键回调方法，用于维护双向链表：

```java
// 节点访问后的回调，当accessOrder为true时，将访问的节点移到链表末尾
void afterNodeAccess(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> last;
    // 如果是按访问顺序排序，且当前节点不是尾节点
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e;
        LinkedHashMap.Entry<K,V> b = p.before;
        LinkedHashMap.Entry<K,V> a = p.after;
        
        // 从链表中断开当前节点
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        
        // 将当前节点插入到链表尾部
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}

// 节点插入后的回调，将新节点添加到链表末尾
void afterNodeInsertion(boolean evict) {
    LinkedHashMap.Entry<K,V> first;
    // 如果需要删除最老的元素，且链表不为空
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

// 节点删除后的回调，从链表中移除节点
void afterNodeRemoval(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e;
    LinkedHashMap.Entry<K,V> b = p.before;
    LinkedHashMap.Entry<K,V> a = p.after;
    
    // 清空节点的前驱和后继引用
    p.before = p.after = null;
    
    // 更新链表
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```

### 3.3 get方法实现

```java
public V get(Object key) {
    Node<K,V> e;
    // 调用HashMap的getNode方法获取节点
    if ((e = getNode(hash(key), key)) == null)
        return null;
    // 如果是按访问顺序排序，则将访问的节点移到链表末尾
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

### 3.4 removeEldestEntry方法

```java
// 判断是否应该移除最老的元素，默认返回false
// 子类可以重写此方法实现LRU缓存等功能
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

## 四、LinkedHashMap的特殊应用

### 4.1 实现LRU缓存

LinkedHashMap可以很容易地实现LRU（Least Recently Used，最近最少使用）缓存：

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity; // 缓存容量
    
    public LRUCache(int capacity) {
        // 初始容量、负载因子、访问顺序
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }
    
    // 重写removeEldestEntry方法，当缓存满时移除最老的元素
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

使用示例：

```java
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "one");
cache.put(2, "two");
cache.put(3, "three");

// 访问键1，将其移到链表末尾
cache.get(1);

// 添加新元素，会导致最久未使用的元素（键2）被移除
cache.put(4, "four");

System.out.println(cache.keySet()); // 输出：[3, 1, 4]
```

### 4.2 保持插入顺序的Map

```java
Map<String, Integer> orderedMap = new LinkedHashMap<>();
orderedMap.put("one", 1);
orderedMap.put("two", 2);
orderedMap.put("three", 3);

// 遍历时会按照插入顺序输出
for (Map.Entry<String, Integer> entry : orderedMap.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
// 输出：
// one: 1
// two: 2
// three: 3
```

## 五、性能分析与使用建议

### 5.1 性能特点

| 操作 | 时间复杂度 | 说明 |
|------|------------|------|
| get/put/remove | O(1) | 继承自HashMap的高效哈希查找 |
| 遍历 | O(n) | 通过双向链表实现有序遍历 |

LinkedHashMap相比HashMap增加了维护双向链表的开销，但提供了有序性保证：
- 空间开销：每个节点增加了两个引用（before和after）
- 时间开销：插入、删除、访问操作需要额外维护双向链表

### 5.2 使用建议

1. **需要有序Map时使用**：当需要按照插入顺序或访问顺序遍历Map时，应优先考虑LinkedHashMap
2. **实现LRU缓存**：通过设置accessOrder为true并重写removeEldestEntry方法，可以轻松实现LRU缓存
3. **避免频繁修改**：如果应用场景中Map会频繁修改但不需要有序性，使用HashMap可能更高效
4. **初始容量设置**：合理设置初始容量可以减少扩容次数，提高性能

## 六、性能优化建议

### 6.1 性能对比

| 操作类型 | HashMap | LinkedHashMap |
|---------|---------|---------------|
| 插入操作 | O(1)    | O(1)+链表维护 |
| 查询操作 | O(1)    | O(1)+访问顺序维护 |
| 遍历操作 | O(n)    | O(n)（有序遍历） |

### 6.2 内存优化技巧
1. **合理设置初始容量**：避免频繁扩容，建议根据业务场景预估容量
2. **使用基本数据类型**：尽量使用`LinkedHashMap<Integer, String>`而非`LinkedHashMap<Long, String>`
3. **及时清理无效数据**：对LRU缓存实现要设置合理的过期策略
4. **避免冗余顺序维护**：不需要顺序特性时改用HashMap

## 七、常见问题解答

### 7.1 并发修改异常
```java
// 多线程环境下需要同步处理
Map<String, Integer> syncMap = Collections.synchronizedMap(new LinkedHashMap<>());
```

### 7.2 LRU缓存容量设置
- 建议设置初始容量为缓存最大容量的1.25倍
- 示例：最大缓存1000条数据时，设置`new LRUCache(1250)`

### 7.3 顺序不一致问题
可能原因及解决方案：
1. **意外修改accessOrder**：确认构造函数参数是否设置正确
2. **并发环境下无序访问**：使用同步包装器或ConcurrentLinkedHashMap
3. **哈希冲突导致桶顺序变化**：重写key对象的hashCode()保证良好分布

## 八、总结

LinkedHashMap通过巧妙地结合HashMap的哈希表结构和双向链表，在保证高效查找的同时，提供了可预测的迭代顺序。它的主要特点包括：

1. **继承自HashMap**：拥有HashMap的所有特性，如高效的查找、插入和删除操作
2. **维护元素顺序**：通过双向链表维护元素的插入顺序或访问顺序
3. **灵活的排序策略**：可以选择按插入顺序（默认）或访问顺序排序
4. **支持LRU机制**：通过重写removeEldestEntry方法，可以实现LRU缓存等功能

LinkedHashMap是HashMap和LinkedList的完美结合，为需要有序性的Map应用场景提供了理想的解决方案。

## 九、参考资料

1. [Oracle官方文档 - LinkedHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashMap.html)
2. 《Effective Java（第3版）》 第13章 集合
3. [Java Collections Framework官方指南](https://docs.oracle.com/javase/tutorial/collections/index.html)
4. [LinkedHashMap源码分析（JDK 17）](https://hg.openjdk.java.net/jdk-updates/jdk17u/file/tip/src/java.base/share/classes/java/util/LinkedHashMap.java)

## 十、讨论与交流

如果您对本文有任何疑问、建议或发现了错误，欢迎在评论区留言讨论。我们可以一起探讨LinkedHashHashMap的实现细节、性能优化或在实际项目中的应用经验。

