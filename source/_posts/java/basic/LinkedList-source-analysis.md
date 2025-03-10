---
title: 【Java】LinkedList核心源码解析与实现原理详解
tags:
  - Java集合
  - 源码分析
categories:
  - Java
  - Java基础
abbrlink: 7a8b905d
date: 2025-03-10 10:00:00
---

## 一、底层数据结构

LinkedList是基于双向链表实现的List接口，其核心数据结构是内部的Node节点类：

```java
// JDK 1.8 LinkedList部分源码
private static class Node<E> {
    E item;           // 节点存储的元素
    Node<E> next;     // 后继节点引用
    Node<E> prev;     // 前驱节点引用

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

与ArrayList基于数组实现不同，LinkedList通过节点之间的相互引用，形成了一个双向链表结构：

```java
transient int size = 0;        // 链表大小
transient Node<E> first;       // 头节点引用
transient Node<E> last;        // 尾节点引用
```

## 二、构造方法解析

```java
// 无参构造
public LinkedList() {
}

// 集合参数构造
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

与ArrayList不同，LinkedList的构造方法非常简单，无参构造方法仅创建一个空链表，不需要初始化容量。集合参数构造方法则是通过addAll方法将集合中的元素添加到链表中。

## 三、关键成员变量

| 变量名 | 作用 |
|--------|------|
| first | 指向链表的第一个节点 |
| last | 指向链表的最后一个节点 |
| size | 链表中的元素数量 |
| modCount | 结构性修改计数器（用于快速失败机制） |

## 四、核心方法源码解析

### 1. add方法执行流程

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

**执行流程：**
1. 创建一个新节点，前驱指向当前的last节点，后继为null
2. 将last引用指向新节点
3. 如果链表为空（l == null），则first也指向新节点
4. 否则，将原last节点的next指向新节点
5. 更新size和modCount

### 2. addFirst方法执行流程

```java
public void addFirst(E e) {
    linkFirst(e);
}

private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

**执行流程：**
1. 创建一个新节点，前驱为null，后继指向当前的first节点
2. 将first引用指向新节点
3. 如果链表为空（f == null），则last也指向新节点
4. 否则，将原first节点的prev指向新节点
5. 更新size和modCount

### 3. get方法执行流程

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

Node<E> node(int index) {
    // 二分查找优化：根据index位置决定从头还是从尾开始查找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

**执行流程：**
1. 检查索引是否越界
2. 调用node(index)方法获取对应位置的节点
3. 返回节点的item值

**优化策略：**
- 根据索引位置，选择从头节点还是尾节点开始查找，减少遍历次数
- 如果索引小于size/2，从头开始查找；否则从尾开始查找

### 4. remove方法执行流程

```java
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

**执行流程：**
1. 检查索引是否越界
2. 调用node(index)方法获取对应位置的节点
3. 调用unlink方法删除节点并返回节点的值
4. unlink方法中：
   - 保存节点的值、前驱和后继
   - 根据节点位置（是否为头/尾节点）调整链表引用
   - 将被删除节点的引用置为null，帮助GC
   - 更新size和modCount
   - 返回被删除的元素

## 五、LinkedList作为队列和栈的实现

LinkedList实现了Deque接口，可以作为队列和栈使用：

```java
// 作为队列使用
Queue<String> queue = new LinkedList<>();
queue.offer("A");    // 入队
queue.poll();       // 出队

// 作为栈使用
Deque<String> stack = new LinkedList<>();
stack.push("A");    // 入栈
stack.pop();        // 出栈
```

相关方法对应关系：

| 队列方法 | 栈方法 | LinkedList方法 | 说明 |
|---------|-------|---------------|------|
| offer(e) | push(e) | addLast(e)/addFirst(e) | 添加元素 |
| poll() | pop() | removeFirst() | 移除并返回元素 |
| peek() | peek() | getFirst() | 获取但不移除元素 |

## 六、迭代器实现原理

```java
private class ListItr implements ListIterator<E> {
    private Node<E> lastReturned;
    private Node<E> next;
    private int nextIndex;
    private int expectedModCount = modCount;

    // 迭代器方法实现...
    
    public E next() {
        checkForComodification();
        if (!hasNext())
            throw new NoSuchElementException();

        lastReturned = next;
        next = next.next;
        nextIndex++;
        return lastReturned.item;
    }
    
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

LinkedList的迭代器实现与ArrayList类似，都使用了快速失败机制，但内部实现有所不同：
- LinkedList迭代器直接持有节点引用，而不是索引
- 通过节点引用的next和prev进行遍历，不需要通过索引计算

## 七、性能对比分析

| 操作 | LinkedList | ArrayList | 说明 |
|------|------------|-----------|------|
| add(E) | O(1) | 平均O(1)，最坏O(n) | LinkedList直接添加到末尾；ArrayList可能需要扩容 |
| add(int, E) | O(n) | O(n) | 都需要找到指定位置，但LinkedList还需要遍历 |
| get(int) | O(n) | O(1) | ArrayList直接通过索引访问；LinkedList需要遍历 |
| remove(int) | O(n) | O(n) | LinkedList需要先遍历找到位置，但删除操作本身是O(1) |

**内存占用对比：**
- ArrayList: 仅存储元素引用和少量管理变量
- LinkedList: 每个元素额外存储两个引用（prev和next），内存占用较大

## 八、使用场景建议

**适合使用LinkedList的场景：**
1. 频繁在两端添加/删除元素
2. 需要实现队列或栈的数据结构
3. 不需要频繁随机访问元素
4. 对内存占用不敏感

**不适合使用LinkedList的场景：**
1. 需要频繁随机访问元素
2. 对内存占用敏感
3. 数据量较大且操作主要是遍历

## 九、源码实现的关键设计思想

1. **双向链表设计**：通过双向链表实现，支持从两端高效操作
2. **二分查找优化**：get方法中根据索引位置决定从头还是从尾开始查找
3. **接口多态性**：实现多个接口（List, Deque），提供多种数据结构功能
4. **快速失败机制**：通过modCount实现迭代过程中的并发修改检测

## 参考资料

- [Oracle Java Collections Framework官方文档](https://docs.oracle.com/javase/8/docs/technotes/guides/collections/overview.html)
- 《Effective Java（第三版）》 第4章 类和接口
- 《Java核心技术 卷Ⅰ》 第9章 集合
- [Google Guava Collections指南](https://github.com/google/guava/wiki/CollectionUtilitiesExplained)

---

本文详细介绍了LinkedList核心源码解析与实现原理，希望对您有所帮助。如果您有任何问题，欢迎在评论区讨论！