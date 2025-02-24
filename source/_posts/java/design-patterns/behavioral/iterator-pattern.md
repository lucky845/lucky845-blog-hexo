---
title: Java设计模式之迭代器模式详解
date: 2025-02-23 21:20:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 6f7e505f
---

## 什么是迭代器模式？

迭代器模式（Iterator Pattern）是一种行为型设计模式，它提供一种方法顺序访问一个聚合对象中的各个元素，而又不需要暴露该对象的内部表示。迭代器模式是Java集合框架的核心设计模式之一。

## 为什么使用迭代器模式？

1. 访问一个聚合对象的内容而无需暴露它的内部表示
2. 支持对聚合对象的多种遍历方式
3. 为遍历不同的聚合结构提供一个统一的接口
4. 简化聚合类的设计

## 实现示例

### 1. 基本实现

```java
// 迭代器接口
public interface Iterator<T> {
    boolean hasNext();
    T next();
    void remove();
}

// 容器接口
public interface Container<T> {
    Iterator<T> getIterator();
}

// 具体容器
public class NameRepository implements Container<String> {
    private String[] names = {"Robert", "John", "Julie", "Lora"};
    
    @Override
    public Iterator<String> getIterator() {
        return new NameIterator();
    }
    
    private class NameIterator implements Iterator<String> {
        private int index;
        
        @Override
        public boolean hasNext() {
            return index < names.length;
        }
        
        @Override
        public String next() {
            if (hasNext()) {
                return names[index++];
            }
            return null;
        }
        
        @Override
        public void remove() {
            throw new UnsupportedOperationException();
        }
    }
}
```

### 2. 自定义集合实现

```java
// 自定义集合
public class CustomCollection<T> {
    private List<T> items = new ArrayList<>();
    
    public void addItem(T item) {
        items.add(item);
    }
    
    public Iterator<T> iterator() {
        return new CustomIterator();
    }
    
    public Iterator<T> reverseIterator() {
        return new ReverseIterator();
    }
    
    private class CustomIterator implements Iterator<T> {
        private int currentIndex = 0;
        
        @Override
        public boolean hasNext() {
            return currentIndex < items.size();
        }
        
        @Override
        public T next() {
            if (hasNext()) {
                return items.get(currentIndex++);
            }
            throw new NoSuchElementException();
        }
        
        @Override
        public void remove() {
            items.remove(--currentIndex);
        }
    }
    
    private class ReverseIterator implements Iterator<T> {
        private int currentIndex = items.size() - 1;
        
        @Override
        public boolean hasNext() {
            return currentIndex >= 0;
        }
        
        @Override
        public T next() {
            if (hasNext()) {
                return items.get(currentIndex--);
            }
            throw new NoSuchElementException();
        }
        
        @Override
        public void remove() {
            items.remove(currentIndex + 1);
        }
    }
}
```

### 3. 树形结构遍历示例

```java
// 树节点
public class TreeNode<T> {
    private T data;
    private List<TreeNode<T>> children = new ArrayList<>();
    
    public TreeNode(T data) {
        this.data = data;
    }
    
    public void addChild(TreeNode<T> child) {
        children.add(child);
    }
    
    public T getData() {
        return data;
    }
    
    public List<TreeNode<T>> getChildren() {
        return children;
    }
}

// 树的深度优先遍历迭代器
public class DepthFirstIterator<T> implements Iterator<T> {
    private Stack<TreeNode<T>> stack = new Stack<>();
    
    public DepthFirstIterator(TreeNode<T> root) {
        stack.push(root);
    }
    
    @Override
    public boolean hasNext() {
        return !stack.isEmpty();
    }
    
    @Override
    public T next() {
        if (!hasNext()) {
            throw new NoSuchElementException();
        }
        
        TreeNode<T> node = stack.pop();
        List<TreeNode<T>> children = node.getChildren();
        for (int i = children.size() - 1; i >= 0; i--) {
            stack.push(children.get(i));
        }
        
        return node.getData();
    }
}

// 树的广度优先遍历迭代器
public class BreadthFirstIterator<T> implements Iterator<T> {
    private Queue<TreeNode<T>> queue = new LinkedList<>();
    
    public BreadthFirstIterator(TreeNode<T> root) {
        queue.offer(root);
    }
    
    @Override
    public boolean hasNext() {
        return !queue.isEmpty();
    }
    
    @Override
    public T next() {
        if (!hasNext()) {
            throw new NoSuchElementException();
        }
        
        TreeNode<T> node = queue.poll();
        queue.addAll(node.getChildren());
        
        return node.getData();
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        NameRepository namesRepository = new NameRepository();
        Iterator<String> iterator = namesRepository.getIterator();
        while (iterator.hasNext()) {
            System.out.println("Name: " + iterator.next());
        }
        
        // 自定义集合示例
        CustomCollection<Integer> numbers = new CustomCollection<>();
        numbers.addItem(1);
        numbers.addItem(2);
        numbers.addItem(3);
        
        System.out.println("Forward iteration:");
        Iterator<Integer> forwardIterator = numbers.iterator();
        while (forwardIterator.hasNext()) {
            System.out.println(forwardIterator.next());
        }
        
        System.out.println("Reverse iteration:");
        Iterator<Integer> reverseIterator = numbers.reverseIterator();
        while (reverseIterator.hasNext()) {
            System.out.println(reverseIterator.next());
        }
        
        // 树形结构遍历示例
        TreeNode<String> root = new TreeNode<>("A");
        TreeNode<String> b = new TreeNode<>("B");
        TreeNode<String> c = new TreeNode<>("C");
        TreeNode<String> d = new TreeNode<>("D");
        TreeNode<String> e = new TreeNode<>("E");
        
        root.addChild(b);
        root.addChild(c);
        b.addChild(d);
        b.addChild(e);
        
        System.out.println("Depth-first traversal:");
        Iterator<String> dfsIterator = new DepthFirstIterator<>(root);
        while (dfsIterator.hasNext()) {
            System.out.println(dfsIterator.next());
        }
        
        System.out.println("Breadth-first traversal:");
        Iterator<String> bfsIterator = new BreadthFirstIterator<>(root);
        while (bfsIterator.hasNext()) {
            System.out.println(bfsIterator.next());
        }
    }
}
```

## 迭代器模式的优点

1. 支持以不同的方式遍历一个聚合对象
2. 简化了聚合类的设计
3. 在同一个聚合上可以有多个遍历
4. 迭代器模式使得增加新的聚合类和迭代器类都很方便

## 迭代器模式的缺点

1. 对于比较简单的遍历，使用迭代器模式可能会显得过于复杂
2. 迭代器模式在一定程度上增加了系统的复杂性

## 适用场景

1. 访问一个聚合对象的内容而无需暴露它的内部表示
2. 需要为聚合对象提供多种遍历方式
3. 为遍历不同的聚合结构提供一个统一的接口

## 与其他模式的关系

1. 组合模式：经常和迭代器模式一起使用来遍历复杂的树形结构
2. 工厂方法模式：可以使用工厂方法模式来创建迭代器
3. 备忘录模式：可以使用迭代器来实现备忘录模式

## 总结

迭代器模式是一种使用频率非常高的设计模式，它提供了一种统一的方式来访问集合对象中的元素。Java集合框架大量使用了迭代器模式，使得我们可以用统一的方式来遍历不同类型的集合。在实际开发中，当需要为自定义的集合类提供遍历功能时，迭代器模式是一个很好的选择。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Java Collections Framework 文档
- Java核心技术

---

希望这篇文章能帮助您更好地理解Java中的迭代器模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 