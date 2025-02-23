---
title: Java设计模式之组合模式详解
date: 2025-02-23 20:20:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 0f1e905f
---

## 什么是组合模式？

组合模式（Composite Pattern）是一种结构型设计模式，它允许你将对象组合成树形结构来表现"整体/部分"层次结构。组合能让客户以一致的方式处理个别对象以及组合对象。

## 为什么使用组合模式？

1. 需要表示对象的部分-整体层次结构
2. 希望用户忽略组合对象与单个对象的不同
3. 处理树形结构
4. 统一管理容器和叶子节点

## 实现示例

### 1. 基本实现

```java
// 组件抽象类
public abstract class Component {
    protected String name;
    
    public Component(String name) {
        this.name = name;
    }
    
    public abstract void add(Component component);
    public abstract void remove(Component component);
    public abstract void display(int depth);
}

// 叶子节点
public class Leaf extends Component {
    public Leaf(String name) {
        super(name);
    }
    
    @Override
    public void add(Component component) {
        System.out.println("叶子节点不能添加子节点");
    }
    
    @Override
    public void remove(Component component) {
        System.out.println("叶子节点不能删除子节点");
    }
    
    @Override
    public void display(int depth) {
        StringBuilder indent = new StringBuilder();
        for (int i = 0; i < depth; i++) {
            indent.append("-");
        }
        System.out.println(indent.toString() + name);
    }
}

// 组合节点
public class Composite extends Component {
    private List<Component> children = new ArrayList<>();
    
    public Composite(String name) {
        super(name);
    }
    
    @Override
    public void add(Component component) {
        children.add(component);
    }
    
    @Override
    public void remove(Component component) {
        children.remove(component);
    }
    
    @Override
    public void display(int depth) {
        StringBuilder indent = new StringBuilder();
        for (int i = 0; i < depth; i++) {
            indent.append("-");
        }
        System.out.println(indent.toString() + name);
        
        for (Component component : children) {
            component.display(depth + 2);
        }
    }
}
```

### 2. 实际应用示例：文件系统

```java
// 文件系统组件
public abstract class FileSystemComponent {
    protected String name;
    protected int size;
    
    public FileSystemComponent(String name, int size) {
        this.name = name;
        this.size = size;
    }
    
    public abstract int getSize();
    public abstract void show();
}

// 文件
public class File extends FileSystemComponent {
    public File(String name, int size) {
        super(name, size);
    }
    
    @Override
    public int getSize() {
        return size;
    }
    
    @Override
    public void show() {
        System.out.println("File: " + name + " (" + size + "KB)");
    }
}

// 目录
public class Directory extends FileSystemComponent {
    private List<FileSystemComponent> children = new ArrayList<>();
    
    public Directory(String name) {
        super(name, 0);
    }
    
    public void add(FileSystemComponent component) {
        children.add(component);
    }
    
    public void remove(FileSystemComponent component) {
        children.remove(component);
    }
    
    @Override
    public int getSize() {
        int totalSize = 0;
        for (FileSystemComponent component : children) {
            totalSize += component.getSize();
        }
        return totalSize;
    }
    
    @Override
    public void show() {
        System.out.println("Directory: " + name + " (" + getSize() + "KB)");
        for (FileSystemComponent component : children) {
            System.out.print("  ");
            component.show();
        }
    }
}
```

### 3. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 创建文件系统结构
        Directory root = new Directory("root");
        Directory home = new Directory("home");
        Directory docs = new Directory("docs");
        
        File file1 = new File("file1.txt", 100);
        File file2 = new File("file2.doc", 200);
        File file3 = new File("file3.pdf", 300);
        
        root.add(home);
        home.add(docs);
        docs.add(file1);
        docs.add(file2);
        home.add(file3);
        
        // 显示整个文件系统
        root.show();
        
        // 显示总大小
        System.out.println("Total size: " + root.getSize() + "KB");
    }
}
```

## 组合模式的优点

1. 定义了包含基本对象和组合对象的类层次结构
2. 简化了客户端代码，客户端可以一致地使用组合对象和单个对象
3. 使得添加新类型的组件变得容易
4. 符合开闭原则

## 组合模式的缺点

1. 在需要限制组件类型时会较为复杂
2. 在需要遍历时可能需要进行类型判断
3. 可能会使设计变得更加抽象

## 适用场景

1. 表示对象的部分-整体层次结构
2. 希望用户忽略组合对象与单个对象的不同
3. 需要统一处理组合对象和单个对象
4. 需要实现树形结构的场景

## 与其他模式的关系

1. 装饰器模式：组合模式改变结构，装饰器模式添加职责
2. 迭代器模式：可以用来遍历组合结构
3. 访问者模式：可以用来定义对组合结构中的对象的操作

## 总结

组合模式是一种非常实用的设计模式，特别适合用来处理树形结构。它通过将对象组合成树形结构，使得客户端可以统一地处理单个对象和组合对象。在实际开发中，当需要处理树形结构时，组合模式是一个很好的选择。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Java API 文档
- Spring Framework 源码

---

希望这篇文章能帮助您更好地理解Java中的组合模式。如果您有任何问题，欢迎在评论区讨论！ 
abbrlink: '0'
---
 