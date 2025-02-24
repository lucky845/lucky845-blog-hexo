---
title: Java设计模式之访问者模式详解
date: 2025-02-23 22:40:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 4f5e305f
---

## 什么是访问者模式？

访问者模式（Visitor Pattern）是一种行为型设计模式，它允许你在不改变各元素类的前提下定义作用于这些元素的新操作。这种模式将操作与对象结构分离，使得我们可以在不修改对象结构的情况下，增加新的操作。

## 为什么使用访问者模式？

1. 实现对象结构和操作的分离
2. 增加新的操作很方便
3. 集中相关的操作
4. 访问者可以积累状态

## 实现示例

### 1. 基本实现

```java
// 访问者接口
public interface Visitor {
    void visit(ConcreteElementA element);
    void visit(ConcreteElementB element);
}

// 元素接口
public interface Element {
    void accept(Visitor visitor);
}

// 具体元素A
public class ConcreteElementA implements Element {
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
    
    public String operationA() {
        return "ElementA";
    }
}

// 具体元素B
public class ConcreteElementB implements Element {
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
    
    public String operationB() {
        return "ElementB";
    }
}

// 具体访问者1
public class ConcreteVisitor1 implements Visitor {
    @Override
    public void visit(ConcreteElementA element) {
        System.out.println("访问者1访问" + element.operationA());
    }
    
    @Override
    public void visit(ConcreteElementB element) {
        System.out.println("访问者1访问" + element.operationB());
    }
}

// 具体访问者2
public class ConcreteVisitor2 implements Visitor {
    @Override
    public void visit(ConcreteElementA element) {
        System.out.println("访问者2访问" + element.operationA());
    }
    
    @Override
    public void visit(ConcreteElementB element) {
        System.out.println("访问者2访问" + element.operationB());
    }
}

// 对象结构
public class ObjectStructure {
    private List<Element> elements = new ArrayList<>();
    
    public void attach(Element element) {
        elements.add(element);
    }
    
    public void detach(Element element) {
        elements.remove(element);
    }
    
    public void accept(Visitor visitor) {
        for (Element element : elements) {
            element.accept(visitor);
        }
    }
}
```

### 2. 文件系统示例

```java
// 文件系统元素接口
public interface FileSystemElement {
    void accept(FileVisitor visitor);
}

// 文件访问者接口
public interface FileVisitor {
    void visit(File file);
    void visit(Directory directory);
}

// 文件类
public class File implements FileSystemElement {
    private String name;
    private int size;
    
    public File(String name, int size) {
        this.name = name;
        this.size = size;
    }
    
    @Override
    public void accept(FileVisitor visitor) {
        visitor.visit(this);
    }
    
    public String getName() {
        return name;
    }
    
    public int getSize() {
        return size;
    }
}

// 目录类
public class Directory implements FileSystemElement {
    private String name;
    private List<FileSystemElement> elements = new ArrayList<>();
    
    public Directory(String name) {
        this.name = name;
    }
    
    public void add(FileSystemElement element) {
        elements.add(element);
    }
    
    @Override
    public void accept(FileVisitor visitor) {
        visitor.visit(this);
        for (FileSystemElement element : elements) {
            element.accept(visitor);
        }
    }
    
    public String getName() {
        return name;
    }
    
    public List<FileSystemElement> getElements() {
        return elements;
    }
}

// 大小计算访问者
public class SizeCalculator implements FileVisitor {
    private int totalSize = 0;
    
    @Override
    public void visit(File file) {
        totalSize += file.getSize();
    }
    
    @Override
    public void visit(Directory directory) {
        // 目录本身不占用空间
    }
    
    public int getTotalSize() {
        return totalSize;
    }
}

// 文件列表打印访问者
public class FileLister implements FileVisitor {
    private int indent = 0;
    
    @Override
    public void visit(File file) {
        System.out.println(" ".repeat(indent) + "- " + file.getName());
    }
    
    @Override
    public void visit(Directory directory) {
        System.out.println(" ".repeat(indent) + "+ " + directory.getName());
        indent += 2;
    }
}
```

### 3. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        ObjectStructure structure = new ObjectStructure();
        structure.attach(new ConcreteElementA());
        structure.attach(new ConcreteElementB());
        
        Visitor visitor1 = new ConcreteVisitor1();
        Visitor visitor2 = new ConcreteVisitor2();
        
        structure.accept(visitor1);
        structure.accept(visitor2);
        
        // 文件系统示例
        Directory root = new Directory("root");
        Directory home = new Directory("home");
        Directory docs = new Directory("docs");
        
        root.add(home);
        home.add(docs);
        docs.add(new File("resume.doc", 1000));
        docs.add(new File("photo.jpg", 2000));
        
        SizeCalculator sizeCalc = new SizeCalculator();
        root.accept(sizeCalc);
        System.out.println("Total size: " + sizeCalc.getTotalSize() + " bytes");
        
        FileLister lister = new FileLister();
        root.accept(lister);
    }
}
```

## 访问者模式的优点

1. 符合单一职责原则
2. 优秀的扩展性
3. 集中相关的操作
4. 对象结构可以复用

## 访问者模式的缺点

1. 增加新的元素类型困难
2. 破坏封装
3. 违反依赖倒置原则

## 适用场景

1. 对象结构中的元素类型固定
2. 需要对一个对象结构中的对象进行很多不同操作
3. 需要避免在元素类中添加新的操作
4. 对象结构包含多个类型的对象

## 与其他模式的关系

1. 组合模式：访问者模式经常与组合模式一起使用
2. 迭代器模式：可以使用迭代器来遍历对象结构
3. 解释器模式：可以使用访问者来解释语法树

## 总结

访问者模式是一种用于分离对象结构和操作的设计模式。它在需要对固定的对象结构执行多种不同操作时特别有用。虽然这种模式增加了系统的复杂度，但它提供了很好的扩展性和灵活性。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Java ASM 框架
- Spring Framework 源码

---

希望这篇文章能帮助您更好地理解Java中的访问者模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 