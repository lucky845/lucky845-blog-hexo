---
title: Java设计模式之模板方法模式详解
date: 2025-02-23 21:00:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 4f5e305f
---

## 什么是模板方法模式？

模板方法模式（Template Method Pattern）是一种行为型设计模式，它定义了一个算法的骨架，将一些步骤延迟到子类中实现。模板方法使得子类可以在不改变算法结构的情况下，重新定义算法中的某些步骤。

## 为什么使用模板方法模式？

1. 复用代码，避免重复
2. 控制算法的主要流程
3. 允许子类定制算法的特定步骤
4. 维护一个统一的算法框架

## 实现示例

### 1. 基本实现

```java
// 抽象类
public abstract class AbstractTemplate {
    // 模板方法
    public final void templateMethod() {
        step1();
        step2();
        step3();
        hook();
    }
    
    // 具体方法
    private void step1() {
        System.out.println("AbstractTemplate.step1");
    }
    
    // 抽象方法
    protected abstract void step2();
    
    // 具体方法
    protected void step3() {
        System.out.println("AbstractTemplate.step3");
    }
    
    // 钩子方法
    protected void hook() {
        // 默认空实现
    }
}

// 具体实现类A
public class ConcreteTemplateA extends AbstractTemplate {
    @Override
    protected void step2() {
        System.out.println("ConcreteTemplateA.step2");
    }
    
    @Override
    protected void hook() {
        System.out.println("ConcreteTemplateA.hook");
    }
}

// 具体实现类B
public class ConcreteTemplateB extends AbstractTemplate {
    @Override
    protected void step2() {
        System.out.println("ConcreteTemplateB.step2");
    }
}
```

### 2. 实际应用示例：数据导出

```java
// 数据导出抽象类
public abstract class DataExporter {
    // 模板方法
    public final void export() {
        connectToDataSource();
        extractData();
        transformData();
        exportData();
        closeConnection();
    }
    
    // 连接数据源
    protected abstract void connectToDataSource();
    
    // 提取数据
    protected abstract void extractData();
    
    // 转换数据
    protected void transformData() {
        System.out.println("执行默认的数据转换");
    }
    
    // 导出数据
    protected abstract void exportData();
    
    // 关闭连接
    private void closeConnection() {
        System.out.println("关闭数据源连接");
    }
}

// Excel导出器
public class ExcelExporter extends DataExporter {
    @Override
    protected void connectToDataSource() {
        System.out.println("连接到MySQL数据库");
    }
    
    @Override
    protected void extractData() {
        System.out.println("执行SQL查询提取数据");
    }
    
    @Override
    protected void exportData() {
        System.out.println("将数据导出到Excel文件");
    }
}

// PDF导出器
public class PdfExporter extends DataExporter {
    @Override
    protected void connectToDataSource() {
        System.out.println("连接到Oracle数据库");
    }
    
    @Override
    protected void extractData() {
        System.out.println("执行存储过程提取数据");
    }
    
    @Override
    protected void transformData() {
        System.out.println("执行PDF特定的数据转换");
    }
    
    @Override
    protected void exportData() {
        System.out.println("将数据导出到PDF文件");
    }
}
```

### 3. 饮料制作示例

```java
// 饮料抽象类
public abstract class Beverage {
    // 模板方法
    public final void prepareBeverage() {
        boilWater();
        brew();
        pourInCup();
        if (customerWantsCondiments()) {
            addCondiments();
        }
    }
    
    private void boilWater() {
        System.out.println("将水煮沸");
    }
    
    protected abstract void brew();
    
    private void pourInCup() {
        System.out.println("倒入杯中");
    }
    
    protected abstract void addCondiments();
    
    // 钩子方法
    protected boolean customerWantsCondiments() {
        return true;
    }
}

// 咖啡类
public class Coffee extends Beverage {
    @Override
    protected void brew() {
        System.out.println("用沸水冲泡咖啡");
    }
    
    @Override
    protected void addCondiments() {
        System.out.println("加入糖和牛奶");
    }
}

// 茶类
public class Tea extends Beverage {
    private boolean wantsLemon;
    
    public Tea(boolean wantsLemon) {
        this.wantsLemon = wantsLemon;
    }
    
    @Override
    protected void brew() {
        System.out.println("用沸水浸泡茶叶");
    }
    
    @Override
    protected void addCondiments() {
        System.out.println("加入柠檬");
    }
    
    @Override
    protected boolean customerWantsCondiments() {
        return wantsLemon;
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        AbstractTemplate templateA = new ConcreteTemplateA();
        templateA.templateMethod();
        
        AbstractTemplate templateB = new ConcreteTemplateB();
        templateB.templateMethod();
        
        // 数据导出示例
        DataExporter excelExporter = new ExcelExporter();
        excelExporter.export();
        
        DataExporter pdfExporter = new PdfExporter();
        pdfExporter.export();
        
        // 饮料制作示例
        Beverage coffee = new Coffee();
        coffee.prepareBeverage();
        
        Beverage teaWithLemon = new Tea(true);
        teaWithLemon.prepareBeverage();
        
        Beverage teaWithoutLemon = new Tea(false);
        teaWithoutLemon.prepareBeverage();
    }
}
```

## 模板方法模式的优点

1. 提高代码复用性
2. 提供了一个框架，便于维护
3. 封装不变部分，扩展可变部分
4. 提供了钩子方法，增加了灵活性

## 模板方法模式的缺点

1. 每个不同的实现都需要一个子类，导致类的数量增加
2. 限制了算法的种类和顺序
3. 子类可能会过度依赖父类的实现

## 适用场景

1. 多个类有相似的算法骨架
2. 需要控制子类扩展的时候
3. 一次性实现算法的不变部分
4. 需要统一管理算法的整体流程

## 与其他模式的关系

1. 工厂方法模式：工厂方法是模板方法的一种特殊形式
2. 策略模式：策略模式使用组合改变整个算法，模板方法使用继承改变算法的特定步骤
3. 命令模式：可以结合使用，将命令的执行过程模板化

## 总结

模板方法模式是一种非常实用的设计模式，它通过定义算法骨架并允许子类重写特定步骤来实现代码复用。这种模式在框架设计中被广泛使用，例如Spring框架中的各种Template类。在实际开发中，当发现多个类有相似的算法结构时，可以考虑使用模板方法模式。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Spring Framework 源码
- Java核心技术

---

希望这篇文章能帮助您更好地理解Java中的模板方法模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 