---
title: 【Java】HashMap核心源码解析与实现原理详解
tags:
  - Java集合
  - 源码分析
categories:
  - Java
  - Java基础
abbrlink: 7a8b905e
date: 2025-03-10 11:00:00
---

## 一、底层数据结构

HashMap是Java中最常用的Map实现，它基于哈希表实现，在JDK 1.8中采用了数组+链表+红黑树的复合数据结构：

```java
// JDK 1.8 HashMap部分源码
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    // 默认初始容量 - 必须是2的幂
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 即16
    
    // 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    
    // 默认加载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    // 链表转红黑树的阈值
    static final int TREEIFY_THRESHOLD = 8;
    
    // 红黑树转链表的阈值
    static final int UNTREEIFY_THRESHOLD = 6;
    
    // 转红黑树时，数组的最小长度
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    // 存储元素的数组，总是2的幂次倍
    transient Node<K,V>[] table;
    
    // 存放具体元素的集合
    transient Set<Map.Entry<K,V>> entrySet;
    
    // 存放元素的个数，注意这个不等于数组的长度
    transient int size;
    
    // 每次扩容和更改map结构的计数器
    transient int modCount;
    
    // 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
    int threshold;
    
    // 填充因子
    final float loadFactor;
}
```

### 1.1 Node节点结构

HashMap的基本存储单元是Node节点，它实现了Map.Entry接口：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;      // 哈希值
    final K key;         // 键
    V value;             // 值
    Node<K,V> next;      // 指向下一个节点的引用
    
    // 构造函数
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
    
    // 实现Map.Entry接口的方法
    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
    
    // 计算hashCode
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
    
    // 设置新值并返回旧值
    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
    
    // 判断两个节点是否相等
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

### 1.2 TreeNode结构

当链表长度超过阈值（默认为8）时，链表会转换为红黑树，此时使用TreeNode节点：

```java
// 红黑树节点，继承自LinkedHashMap.Entry
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // 父节点
    TreeNode<K,V> left;    // 左子节点
    TreeNode<K,V> right;   // 右子节点
    TreeNode<K,V> prev;    // 删除时需要用到的前一个节点
    boolean red;           // 节点颜色（红/黑）
    
    // 构造函数
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
    
    // 返回当前节点的根节点
    final TreeNode<K,V> root() {
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }
    // ... 其他红黑树操作方法
}
```

## 二、构造方法解析

HashMap提供了多个构造方法，以适应不同的初始化需求：

```java
// 默认构造函数
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}

// 指定初始容量的构造函数
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

// 指定初始容量和加载因子的构造函数
public HashMap(int initialCapacity, float loadFactor) {
    // 检查参数合法性
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
    
    this.loadFactor = loadFactor;
    // 计算阈值，注意这里并没有初始化table数组
    this.threshold = tableSizeFor(initialCapacity);
}

// 使用已有Map创建新的HashMap
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

其中，`tableSizeFor`方法用于计算大于等于给定值的最小2的幂：

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

## 三、核心方法源码解析

### 3.1 put方法

put方法是HashMap最常用的方法之一，用于添加或更新键值对：

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// 计算key的哈希值
static final int hash(Object key) {
    int h;
    // 如果key为null，返回0；否则，将key的hashCode与其右移16位的值进行异或运算
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// put的核心实现
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 如果table为空或长度为0，则进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 计算索引位置，如果该位置为空，则直接创建新节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 如果第一个节点就是要找的键，则记录该节点
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果是红黑树节点，则调用红黑树的插入方法
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 遍历链表
            for (int binCount = 0; ; ++binCount) {
                // 到达链表尾部，创建新节点
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 如果链表长度达到阈值，则转为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到相同的key，跳出循环
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // 如果找到了对应的key，则更新值并返回旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    // 修改计数加1
    ++modCount;
    // 如果大小超过阈值，则扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### 3.2 get方法

get方法用于根据键获取对应的值：

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    
    // 如果table不为空且长度大于0，且对应索引位置的节点不为空
    if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
        // 检查第一个节点是否匹配
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        
        // 如果有下一个节点
        if ((e = first.next) != null) {
            // 如果是红黑树节点，则使用红黑树的查找方法
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            
            // 遍历链表查找
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

### 3.3 resize方法

resize方法用于初始化或扩容哈希表：

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    // 如果旧容量大于0
    if (oldCap > 0) {
        // 如果旧容量已达到最大值，则不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 否则，新容量为旧容量的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 新阈值也翻倍
    }
    // 如果旧阈值大于0（使用带参构造函数但尚未初始化）
    else if (oldThr > 0)
        newCap = oldThr;
    // 如果旧容量和旧阈值都为0（使用无参构造函数）
    else {
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    // 计算新阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                 (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    
    // 创建新数组
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    
    // 如果旧表不为空，则将旧表中的元素移动到新表中
    if (oldTab != null) {
        // 遍历旧表
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 如果当前位置有元素
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null; // 释放旧表引用
                // 如果只有一个节点，直接放入新表
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果是红黑树节点，则拆分红黑树
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // 如果是链表
                    // 将链表分成两部分，一部分保持原索引，另一部分移动到原索引+oldCap的位置
                    Node<K,V> loHead = null, loTail = null; // 保持原索引的链表头尾
                    Node<K,V> hiHead = null, hiTail = null; // 移动到新索引的链表头尾
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 判断节点应该放在原索引还是原索引+oldCap的位置
                        // 这里利用了哈希值的特性，扩容后，节点要么在原位置，要么在原位置+oldCap
                        if ((e.hash & oldCap) == 0) { // 放在原索引
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else { // 放在原索引+oldCap的位置
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    // 将两个链表放入新表对应位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

### 3.4 remove方法

remove方法用于从哈希表中删除指定键的映射：

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 如果表不为空且对应位置有节点
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 检查第一个节点是否匹配
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            // 如果是红黑树节点，则使用红黑树的查找方法
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 遍历链表查找
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        
        // 如果找到了节点，且不需要匹配值或值匹配
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            // 如果是红黑树节点，则使用红黑树的删除方法
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 如果是链表的第一个节点
            else if (node == p)
                tab[index] = node.next;
            // 如果是链表中间的节点
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

## 四、HashMap的性能优化

### 4.1 哈希算法优化

HashMap的哈希算法经过了精心设计，以减少哈希冲突：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这个算法将哈希值的高16位与低16位进行异或运算，主要有两个目的：

1. **充分利用哈希值的所有位**：由于HashMap的容量通常不会很大，计算索引时只会用到哈希值的低位。通过将高16位与低16位异或，可以让高位也参与索引计算，减少冲突。

2. **提高随机性**：增加哈希值的随机性，使得数据分布更均匀。

### 4.2 红黑树优化

JDK 1.8中引入了红黑树来优化链表过长的问题：

1. **链表转红黑树的条件**：
   - 链表长度达到TREEIFY_THRESHOLD（默认为8）
   - 数组长度达到MIN_TREEIFY_CAPACITY（默认为64）

2. **红黑树转链表的条件**：
   - 红黑树节点数量小于UNTREEIFY_THRESHOLD（默认为6）

这种优化使得在哈希冲突严重的情况下，查找的时间复杂度从O(n)降低到O(log n)。

### 4.3 扩容优化

JDK 1.8中对扩容过程也进行了优化：

1. **容量始终为2的幂**：这样可以通过位运算快速计算索引，提高效率。

2. **元素重新分配**：扩容时，元素要么在原位置，要么在原位置+oldCap的位置，避免了重新计算哈希值。

## 五、HashMap的使用注意事项

### 5.1 初始容量设置

合理设置初始容量可以减少扩容次数，提高性能：

```java
// 如果预计存储100个元素，设置初始容量为100/0.75 ≈ 134
Map<String, Object> map = new HashMap<>(134);
```

### 5.2 线程安全问题

HashMap不是线程安全的，在多线程环境下可能导致数据不一致甚至死循环。解决方案：

1. 使用Collections.synchronizedMap()包装：

```java
Map<String, Object> synchronizedMap = Collections.synchronizedMap(new HashMap<>());
```

2. 使用ConcurrentHashMap：

```java
ConcurrentHashMap<String, Object> concurrentMap = new ConcurrentHashMap<>();
```

### 5.3 自定义对象作为键

当使用自定义对象作为键时，必须正确重写equals()和hashCode()方法：

```java
public class Person {
    private String name;
    private int age;
    
    // 构造函数、getter和setter省略
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return age == person.age && Objects.equals(name, person.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

## 六、总结

HashMap是Java中使用最广泛的Map实现，它通过数组+链表+红黑树的复合数据结构，实现了高效的键值对存储和查找。

主要特点：

1. **底层结构**：数组+链表+红黑树
2. **初始容量**：默认16，总是2的幂
3. **加载因子**：默认0.75
4. **扩容机制**：当元素数量超过容量*加载因子时，容量扩大为原来的2倍
5. **红黑树优化**：当链表长度超过8且数组长度超过64时，链表转换为红黑树

HashMap的源码设计体现了空间与时间的权衡，以及对各种边界情况的处理。通过深入理解HashMap的实现原理，我们不仅能够更加高效地使用这一集合类，还能从中学习到许多优秀的设计思想和编程技巧，这对于提升Java编程能力和系统设计水平都有很大帮助。

## 七、参考资料

1. **官方文档**
   - [Java SE 17 API Documentation - HashMap](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/HashMap.html)
   - [Java SE 8 API Documentation - HashMap](https://docs.oracle.com/javase/8/docs/api/java/util/HashMap.html)

2. **书籍**
   - Joshua Bloch. *Effective Java (3rd Edition)*. Addison-Wesley Professional, 2018.
   - Cay S. Horstmann. *Core Java Volume I—Fundamentals (11th Edition)*. Prentice Hall, 2018.
   - Brian Goetz. *Java Concurrency in Practice*. Addison-Wesley Professional, 2006.

3. **技术博客与文章**
   - [Java HashMap工作原理及实现](https://tech.meituan.com/2016/06/24/java-hashmap.html) - 美团技术团队
   - [HashMap源码分析](https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20容器.md#hashmap) - CS-Notes

4. **JDK源码**
   - [JDK 1.8 HashMap源码](https://github.com/openjdk/jdk/blob/jdk8-b120/src/share/classes/java/util/HashMap.java)

## 八、讨论与交流

如果您对本文有任何疑问、建议或发现了错误，欢迎在评论区留言讨论。我们可以一起探讨HashMap的实现细节、性能优化或在实际项目中的应用经验。
