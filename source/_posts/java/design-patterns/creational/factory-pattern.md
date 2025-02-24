---
title: Java设计模式之工厂模式详解
date: 2025-02-23 19:30:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 5f6e403f
---

## 什么是工厂模式？

工厂模式（Factory Pattern）是 Java 中最常用的设计模式之一，它提供了一种创建对象的最佳方式。在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。

## 工厂模式的三种类型

1. 简单工厂模式（Simple Factory Pattern）
2. 工厂方法模式（Factory Method Pattern）
3. 抽象工厂模式（Abstract Factory Pattern）

## 1. 简单工厂模式

### 实现示例

让我们通过一个计算器的例子来理解简单工厂模式：

```java
// 操作接口
public interface Operation {
    double getResult(double numberA, double numberB);
}

// 加法操作
public class AddOperation implements Operation {
    @Override
    public double getResult(double numberA, double numberB) {
        return numberA + numberB;
    }
}

// 减法操作
public class SubtractOperation implements Operation {
    @Override
    public double getResult(double numberA, double numberB) {
        return numberA - numberB;
    }
}

// 乘法操作
public class MultiplyOperation implements Operation {
    @Override
    public double getResult(double numberA, double numberB) {
        return numberA * numberB;
    }
}

// 除法操作
public class DivideOperation implements Operation {
    @Override
    public double getResult(double numberA, double numberB) {
        if (numberB == 0) {
            throw new IllegalArgumentException("除数不能为0");
        }
        return numberA / numberB;
    }
}

// 简单工厂类
public class OperationFactory {
    public static Operation createOperation(String operator) {
        switch (operator) {
            case "+":
                return new AddOperation();
            case "-":
                return new SubtractOperation();
            case "*":
                return new MultiplyOperation();
            case "/":
                return new DivideOperation();
            default:
                throw new IllegalArgumentException("不支持的操作符");
        }
    }
}

// 客户端使用示例
public class Client {
    public static void main(String[] args) {
        Operation operation = OperationFactory.createOperation("+");
        double result = operation.getResult(10, 5);
        System.out.println("10 + 5 = " + result);  // 输出：10 + 5 = 15
    }
}
```

## 2. 工厂方法模式

工厂方法模式是简单工厂模式的进阶版本。

### 实现示例

```java
// 抽象产品
public interface Product {
    void operation();
}

// 具体产品A
public class ConcreteProductA implements Product {
    @Override
    public void operation() {
        System.out.println("ConcreteProductA operation");
    }
}

// 具体产品B
public class ConcreteProductB implements Product {
    @Override
    public void operation() {
        System.out.println("ConcreteProductB operation");
    }
}

// 抽象工厂
public interface Factory {
    Product createProduct();
}

// 具体工厂A
public class ConcreteFactoryA implements Factory {
    @Override
    public Product createProduct() {
        return new ConcreteProductA();
    }
}

// 具体工厂B
public class ConcreteFactoryB implements Factory {
    @Override
    public Product createProduct() {
        return new ConcreteProductB();
    }
}

// 客户端使用示例
public class Client {
    public static void main(String[] args) {
        Factory factoryA = new ConcreteFactoryA();
        Product productA = factoryA.createProduct();
        productA.operation();

        Factory factoryB = new ConcreteFactoryB();
        Product productB = factoryB.createProduct();
        productB.operation();
    }
}
```

## 实际应用示例：日志记录器

```java
// 日志记录接口
public interface Logger {
    void log(String message);
}

// 文件日志记录器
public class FileLogger implements Logger {
    @Override
    public void log(String message) {
        System.out.println("File Logger: " + message);
    }
}

// 数据库日志记录器
public class DatabaseLogger implements Logger {
    @Override
    public void log(String message) {
        System.out.println("Database Logger: " + message);
    }
}

// 日志记录器工厂接口
public interface LoggerFactory {
    Logger createLogger();
}

// 文件日志记录器工厂
public class FileLoggerFactory implements LoggerFactory {
    @Override
    public Logger createLogger() {
        return new FileLogger();
    }
}

// 数据库日志记录器工厂
public class DatabaseLoggerFactory implements LoggerFactory {
    @Override
    public Logger createLogger() {
        return new DatabaseLogger();
    }
}

// 使用示例
public class LoggerClient {
    public static void main(String[] args) {
        LoggerFactory factory = new FileLoggerFactory();
        Logger logger = factory.createLogger();
        logger.log("这是一条测试日志");
    }
}
```

## 工厂模式的优点

1. 封装对象的创建过程
2. 降低代码耦合度
3. 符合开闭原则
4. 提供统一的创建对象的接口

## 工厂模式的缺点

1. 增加系统的复杂度
2. 需要创建大量的类
3. 增加了系统的抽象性和理解难度

## 适用场景

1. 不知道具体需要创建什么对象
2. 需要解耦对象的创建和使用
3. 需要系统具有较好的扩展性
4. 需要屏蔽产品的具体实现

## Spring框架中的工厂模式

Spring框架中大量使用了工厂模式，比如：

```java
// BeanFactory接口
public interface BeanFactory {
    Object getBean(String name);
    <T> T getBean(String name, Class<T> requiredType);
    <T> T getBean(Class<T> requiredType);
}
```

## 总结

工厂模式是一种非常实用的创建型设计模式，它提供了一种创建对象的最佳方式。在实际开发中，我们可以根据具体需求选择使用简单工厂、工厂方法或抽象工厂模式。

工厂模式的核心思想是：
1. 封装对象的创建过程
2. 解耦对象的创建和使用
3. 提供统一的对象创建接口

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Spring Framework 源码
- Java核心技术

---

希望这篇文章能帮助您更好地理解Java中的工厂模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 