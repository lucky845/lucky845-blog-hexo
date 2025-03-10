---
title: 【Java】ConcurrentHashMap核心源码与底层数据结构分析
tags:
  - Java集合
  - 源码分析
  - 并发编程
categories:
  - Java
  - Java基础
abbrlink: 7a8b905f
date: 2025-03-10 14:00:00
---

## 一、底层数据结构

ConcurrentHashMap是Java中高性能的线程安全哈希表实现，它在JDK 1.8中采用了数组+链表+红黑树的复合数据结构，同时结合CAS和synchronized实现了高效的并发控制：

```java
// JDK 1.8 ConcurrentHashMap部分源码
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V> implements ConcurrentMap<K,V>, Serializable {
    // 最大容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    
    // 默认初始容量
    private static final int DEFAULT_CAPACITY = 16;
    
    // 默认并发级别，已不再使用，但为了兼容性保留
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    
    // 负载因子，当前使用固定值
    private static final float LOAD_FACTOR = 0.75f;
    
    // 链表转红黑树阈值
    static final int TREEIFY_THRESHOLD = 8;
    
    // 红黑树转链表阈值
    static final int UNTREEIFY_THRESHOLD = 6;
    
    // 转红黑树时表的最小容量
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    // 每个线程处理的最小批量数
    private static final int MIN_TRANSFER_STRIDE = 16;
    
    // sizeCtl中记录stamp的位数
    private static int RESIZE_STAMP_BITS = 16;
    
    // 可以帮助扩容的最大线程数
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
    
    // sizeCtl中记录size大小的偏移量
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
    
    // 节点哈希值的特殊标记
    static final int MOVED     = -1; // 表示正在转移
    static final int TREEBIN   = -2; // 表示已经转为红黑树
    static final int RESERVED  = -3; // 表示节点的哈希值为-3，保留
    
    // 可用处理器数量
    static final int NCPU = Runtime.getRuntime().availableProcessors();
    
    // 存储元素的数组，大小总是2的幂次方
    transient volatile Node<K,V>[] table;
    
    // 扩容时用于存储元素的临时数组
    private transient volatile Node<K,V>[] nextTable;
    
    // 记录容量，初始化和扩容控制
    // 负数：表示正在初始化或扩容，-1表示正在初始化，-N表示有N-1个线程正在进行扩容
    // 正数：表示下一次扩容的阈值
    private transient volatile int sizeCtl;
    
    // 扩容时存储每个桶元素的转移进度
    private transient volatile int transferIndex;
    
    // 元素个数计数器，基于CounterCell实现
    private transient volatile long baseCount;
    
    // 元素计数器数组，用于分散计数
    private transient volatile CounterCell[] counterCells;
    
    // 视图相关字段
    private transient KeySetView<K,V> keySet;
    private transient ValuesView<K,V> values;
    private transient EntrySetView<K,V> entrySet;
}
```

### 1.1 Node节点结构

ConcurrentHashMap的基本存储单元是Node节点，它与HashMap的Node类似，但使用volatile修饰value和next引用，保证了可见性：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;      // 哈希值
    final K key;         // 键
    volatile V value;    // 值，使用volatile修饰
    volatile Node<K,V> next; // 下一个节点的引用，使用volatile修饰
    
    // 构造函数
    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = val;
        this.next = next;
    }
    
    // 实现Map.Entry接口的方法
    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final int hashCode()    { return key.hashCode() ^ value.hashCode(); }
    
    // 设置新值并返回旧值
    public final V setValue(V newValue) {
        throw new UnsupportedOperationException(); // 不支持直接修改值
    }
    
    // 判断两个节点是否相等
    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = value) || v.equals(u)));
    }
    
    // 查找方法，用于在链表中查找指定key的节点
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

### 1.2 TreeNode和TreeBin结构

当链表长度超过阈值（默认为8）时，链表会转换为红黑树，此时使用TreeNode和TreeBin节点：

```java
// TreeNode是红黑树的节点
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // 父节点
    TreeNode<K,V> left;    // 左子节点
    TreeNode<K,V> right;   // 右子节点
    TreeNode<K,V> prev;    // 删除时需要用到的前一个节点
    boolean red;           // 节点颜色（红/黑）
    
    // 构造函数
    TreeNode(int hash, K key, V val, Node<K,V> next,
             TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }
    
    // 返回根节点
    final TreeNode<K,V> root() {
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }
    // ... 其他红黑树操作方法
}

// TreeBin是ConcurrentHashMap特有的，封装了红黑树的根节点
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;           // 红黑树的根节点
    volatile TreeNode<K,V> first; // 链表的第一个节点
    volatile Thread waiter;       // 最近的一个设置写锁的线程
    volatile int lockState;       // 锁状态
    
    // 锁状态枚举
    static final int WRITER = 1;  // 获取写锁的标识
    static final int WAITER = 2;  // 等待写锁的标识
    static final int READER = 4;  // 增加读锁的单位值
    
    // 构造函数，传入TreeNode链表的第一个节点，构建红黑树
    TreeBin(TreeNode<K,V> b) {
        super(TREEBIN, null, null, null); // TREEBIN是特殊的哈希值
        this.first = b;
        TreeNode<K,V> r = null;
        for (TreeNode<K,V> x = b, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            if (r == null) {
                x.parent = null;
                x.red = false;
                r = x;
            }
            else {
                // 构建红黑树的过程
                // ...
            }
        }
        this.root = r;
    }
    // ... 其他红黑树操作方法和锁控制方法
}
```

### 1.3 ForwardingNode节点

ForwardingNode是一种特殊的节点，在扩容时使用，它的哈希值为MOVED（-1），指向nextTable：

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable; // 扩容时的新表引用
    
    // 构造函数
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null); // MOVED是特殊的哈希值
        this.nextTable = tab;
    }
    
    // 在ForwardingNode中查找元素，直接到nextTable中查找
    Node<K,V> find(int h, Object k) {
        // 循环查找，应对多层forwarding的情况
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```

## 二、并发控制机制

### 2.1 CAS操作

ConcurrentHashMap大量使用CAS（Compare And Swap）操作来保证线程安全，避免使用全局锁：

```java
// 获取数组中的节点（volatile读）
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

// 利用CAS设置数组中的节点
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                  Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

// 设置数组中的节点（volatile写）
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

### 2.2 synchronized锁

ConcurrentHashMap在JDK 1.8中使用synchronized锁定链表或红黑树的首节点，实现了细粒度的锁控制：

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable(); // 初始化表
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 桶为空，CAS插入新节点
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f); // 协助扩容
        else {
            V oldVal = null;
            // 对首节点加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // 链表
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i); // 链表转红黑树
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount); // 更新计数
    return null;
}
```

### 2.3 读操作无锁化

ConcurrentHashMap的get操作不需要加锁，利用volatile保证了可见性：

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) { // 检查第一个节点
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0) // 特殊节点（ForwardingNode或TreeBin）
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) { // 遍历链表
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

## 三、核心方法实现

### 3.1 初始化方法

ConcurrentHashMap使用懒加载策略，在第一次插入元素时才初始化table：

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0) // 有其他线程在初始化
            Thread.yield(); // 让出CPU时间片
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // CAS设置sizeCtl为-1，表示正在初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY; // 默认容量16
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2); // 设置阈值为0.75n
                }
            } finally {
                sizeCtl = sc; // 恢复sizeCtl为阈值
            }
            break;
        }
    }
    return tab;
}
```

### 3.2 扩容方法

ConcurrentHashMap的扩容操作支持多线程并发执行，大大提高了扩容效率：

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 计算每个线程处理的桶区间跨度，最小为16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;
    
    // 初始化nextTab
    if (nextTab == null) {
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1]; // 容量翻倍
            nextTab = nt;
        } catch (Throwable ex) {
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n; // 初始化转移下标
    }
    
    int nextn = nextTab.length;
    // 创建ForwardingNode，标记已经处理过的桶
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true; // 是否继续处理下一个桶
    boolean finishing = false; // 是否完成扩容
    
    // 多线程并发扩容的核心循环
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 确定当前线程负责的桶区间
        while (advance) {
            // ... 计算下一个要处理的桶索引
        }
        
        if (i < 0 || i >= n || i + n >= nextn) {
            // 扩容结束
            if (finishing) {
                nextTable = null;
                table = nextTab; // 更新table引用
                sizeCtl = (n << 1) - (n >>> 1); // 更新阈值
                return;
            }
            // ... 检查是否所有桶都已处理完毕
        }
        else if ((f = tabAt(tab, i)) == null) {
            // 桶为空，放置ForwardingNode标记
            if (casTabAt(tab, i, null, fwd))
                advance = true; // 处理下一个桶
        }
        else if ((fh = f.hash) == MOVED) {
            // 已经是ForwardingNode，说明这个桶已经被处理过了
            advance = true;
        }
        else {
            // 对桶首节点加锁，处理该桶的元素
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // ... 将元素分配到新桶中（高位桶和低位桶）
                }
            }
        }
    }
}
```

### 3.3 size计算方法

ConcurrentHashMap使用分段计数的方式实现高效的size计算：

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

## 四、与HashMap的对比

### 4.1 线程安全性

ConcurrentHashMap与HashMap最大的区别在于线程安全性：

1. **HashMap**：非线程安全，在多线程环境下可能导致死循环、数据丢失等问题
2. **ConcurrentHashMap**：线程安全，采用CAS+synchronized实现并发控制

### 4.2 实现机制对比

| 特性 | HashMap | ConcurrentHashMap |
| --- | --- | --- |
| 底层结构 | 数组+链表+红黑树 | 数组+链表+红黑树 |
| 线程安全 | 否 | 是 |
| 锁机制 | 无 | JDK 1.7：分段锁<br>JDK 1.8：CAS+synchronized |
| 允许null键值 | 是 | 否 |
| 扩容方式 | 单线程扩容 | 多线程并发扩容 |
| 计数方式 | 直接维护size字段 | 分段计数（CounterCell数组） |

### 4.3 性能对比

1. **单线程环境**：HashMap性能略高于ConcurrentHashMap
2. **多线程环境**：
   - ConcurrentHashMap性能远高于HashMap
   - ConcurrentHashMap性能远高于使用Collections.synchronizedMap()包装的HashMap
   - ConcurrentHashMap性能远高于Hashtable

## 五、JDK 1.7与JDK 1.8的区别

### 5.1 数据结构变化

1. **JDK 1.7**：
   - 采用Segment数组+HashEntry数组的结构
   - Segment继承自ReentrantLock，实现分段锁
   - 每个Segment维护一个HashEntry数组

2. **JDK 1.8**：
   - 采用Node数组+链表+红黑树的结构
   - 去除了Segment分段锁的设计
   - 使用CAS+synchronized实现更细粒度的锁控制

### 5.2 锁实现变化

1. **JDK 1.7**：
   - 使用分段锁Segment，每个Segment独立加锁
   - 最大并发度受Segment数组大小限制

2. **JDK 1.8**：
   - 使用CAS+synchronized锁定桶的首节点
   - 锁粒度更细，并发度更高
   - 利用synchronized的优化（偏向锁、轻量级锁、重量级锁）提升性能

### 5.3 扩容机制变化

1. **JDK 1.7**：
   - 每个Segment独立扩容
   - 扩容时，其他Segment不受影响

2. **JDK 1.8**：
   - 支持多线程并发扩容
   - 使用ForwardingNode标记已处理的桶
   - 扩容效率大幅提升

## 六、总结

ConcurrentHashMap是Java中高性能的线程安全哈希表实现，它通过精心设计的数据结构和并发控制机制，实现了高效的并发访问。

主要特点：

1. **底层结构**：数组+链表+红黑树
2. **并发控制**：CAS+synchronized
3. **读操作**：完全无锁，利用volatile保证可见性
4. **写操作**：细粒度的锁控制，只锁定当前桶的首节点
5. **扩容机制**：支持多线程并发扩容，提高扩容效率
6. **计数方式**：使用分段计数，避免计数时的竞争

ConcurrentHashMap的设计充分考虑了并发环境下的性能和安全性，是Java并发编程中的经典实现，也是学习并发数据结构设计的优秀范例。

## 七、参考资料

1. **官方文档**
   - [Java SE 17 API Documentation - ConcurrentHashMap](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
   - [Java SE 8 API Documentation - ConcurrentHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html)

2. **书籍**
   - Brian Goetz等. *Java并发编程实战*. 机械工业出版社, 2012.
   - 周志明. *深入理解Java虚拟机（第3版）*. 机械工业出版社, 2019.
   - Doug Lea. *Java并发编程：设计原则与模式*. 机械工业出版社, 2004.

3. **技术博客与文章**
   - [深入浅出ConcurrentHashMap1.8](https://www.jianshu.com/p/c0642afe03e0)
   - [ConcurrentHashMap源码分析](https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20并发.md#concurrenthashmap)
   - [聊聊并发（四）——深入分析ConcurrentHashMap](https://www.infoq.cn/article/ConcurrentHashMap/)

4. **JDK源码**
   - [JDK 1.8 ConcurrentHashMap源码](https://github.com/openjdk/jdk/blob/jdk8-b120/src/share/classes/java/util/concurrent/ConcurrentHashMap.java)
   - [JDK 17 ConcurrentHashMap源码](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/concurrent/ConcurrentHashMap.java)

## 八、讨论与交流

如果您对ConcurrentHashMap的实现有任何疑问或见解，欢迎在评论区留言讨论。您的反馈和建议将帮助我不断改进文章内容。
