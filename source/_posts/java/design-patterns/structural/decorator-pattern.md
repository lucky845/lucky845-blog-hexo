---
title: Java设计模式之装饰器模式详解
date: 2025-02-23 20:10:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 9f0e805f
---

## 什么是装饰器模式？

装饰器模式（Decorator Pattern）是一种结构型设计模式，它允许向一个现有的对象添加新的功能，同时又不改变其结构。这种模式创建了一个装饰类，用来包装原有的类，并在保持类方法签名完整性的前提下，提供了额外的功能。

## 为什么使用装饰器模式？

1. 在不改变原有对象的情况下动态地给对象添加功能
2. 比继承更加灵活
3. 可以实现不同功能的组合
4. 符合开闭原则

## 实现示例

### 1. 基本实现

```java
// 组件接口
public interface Component {
    void operation();
}

// 具体组件
public class ConcreteComponent implements Component {
    @Override
    public void operation() {
        System.out.println("具体组件的基本功能");
    }
}

// 装饰器抽象类
public abstract class Decorator implements Component {
    protected Component component;
    
    public Decorator(Component component) {
        this.component = component;
    }
    
    @Override
    public void operation() {
        component.operation();
    }
}

// 具体装饰器A
public class ConcreteDecoratorA extends Decorator {
    public ConcreteDecoratorA(Component component) {
        super(component);
    }
    
    @Override
    public void operation() {
        super.operation();
        addedBehavior();
    }
    
    private void addedBehavior() {
        System.out.println("装饰器A添加的功能");
    }
}

// 具体装饰器B
public class ConcreteDecoratorB extends Decorator {
    public ConcreteDecoratorB(Component component) {
        super(component);
    }
    
    @Override
    public void operation() {
        super.operation();
        addedBehavior();
    }
    
    private void addedBehavior() {
        System.out.println("装饰器B添加的功能");
    }
}
```

### 2. 实际应用示例：咖啡订单系统

```java
// 饮料抽象类
public abstract class Beverage {
    protected String description = "Unknown Beverage";
    
    public String getDescription() {
        return description;
    }
    
    public abstract double cost();
}

// 具体饮料：浓缩咖啡
public class Espresso extends Beverage {
    public Espresso() {
        description = "浓缩咖啡";
    }
    
    @Override
    public double cost() {
        return 15.0;
    }
}

// 调味品装饰器抽象类
public abstract class CondimentDecorator extends Beverage {
    protected Beverage beverage;
    
    public abstract String getDescription();
}

// 具体装饰器：牛奶
public class Milk extends CondimentDecorator {
    public Milk(Beverage beverage) {
        this.beverage = beverage;
    }
    
    @Override
    public String getDescription() {
        return beverage.getDescription() + ", 加牛奶";
    }
    
    @Override
    public double cost() {
        return beverage.cost() + 5.0;
    }
}

// 具体装饰器：摩卡
public class Mocha extends CondimentDecorator {
    public Mocha(Beverage beverage) {
        this.beverage = beverage;
    }
    
    @Override
    public String getDescription() {
        return beverage.getDescription() + ", 加摩卡";
    }
    
    @Override
    public double cost() {
        return beverage.cost() + 3.0;
    }
}
```

### 3. Java I/O中的装饰器模式

```java
// 文件读取示例
public class IODecoratorExample {
    public void readFile() {
        try {
            // 基本的文件输入流
            InputStream fileStream = new FileInputStream("test.txt");
            
            // 添加缓冲功能
            InputStream bufferedStream = new BufferedInputStream(fileStream);
            
            // 添加数据转换功能
            Reader reader = new InputStreamReader(bufferedStream, "UTF-8");
            
            // 添加缓冲读取功能
            BufferedReader bufferedReader = new BufferedReader(reader);
            
            String line;
            while ((line = bufferedReader.readLine()) != null) {
                System.out.println(line);
            }
            
            bufferedReader.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        Component component = new ConcreteComponent();
        Component decoratorA = new ConcreteDecoratorA(component);
        Component decoratorB = new ConcreteDecoratorB(decoratorA);
        
        decoratorB.operation();
        
        // 咖啡订单示例
        Beverage beverage = new Espresso();
        System.out.println(beverage.getDescription() + " ￥" + beverage.cost());
        
        beverage = new Milk(beverage);
        beverage = new Mocha(beverage);
        System.out.println(beverage.getDescription() + " ￥" + beverage.cost());
    }
}
```

## 装饰器模式的优点

1. 比继承更加灵活
2. 可以动态地添加或删除功能
3. 可以实现不同功能的组合
4. 符合开闭原则

## 装饰器模式的缺点

1. 可能会产生很多小类
2. 增加了系统的复杂度
3. 装饰器的顺序可能会影响结果

## 适用场景

1. 需要动态地给对象添加功能
2. 需要在不影响其他对象的情况下，给单个对象添加功能
3. 需要对功能进行组合的场景
4. 继承关系过于复杂的场景

## 与其他模式的关系

1. 适配器模式：适配器改变接口，装饰器增强功能
2. 代理模式：代理控制访问，装饰器添加职责
3. 组合模式：可以与装饰器模式一起使用，增强树形结构中的对象

## 总结

装饰器模式是一种非常实用的设计模式，它提供了比继承更加灵活的扩展对象功能的方式。Java的I/O系统大量使用了装饰器模式，这使得我们可以通过组合不同的装饰器来实现各种I/O功能。在实际开发中，当需要动态地给对象添加功能时，装饰器模式是一个很好的选择。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Java API 文档
- Spring Framework 源码

---

希望这篇文章能帮助您更好地理解Java中的装饰器模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 