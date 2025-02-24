---
title: Java设计模式之建造者模式详解
date: 2025-02-23 19:20:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 4d5e302f
---

## 什么是建造者模式？

建造者模式（Builder Pattern）是一种创建型设计模式，它允许您分步骤创建复杂对象。该模式允许您使用相同的创建代码生成不同类型和形式的对象。

## 为什么使用建造者模式？

1. 需要创建的对象具有复杂的内部结构
2. 需要生成的对象内部属性之间的建造顺序有依赖关系
3. 对象的创建过程独立于创建该对象的类
4. 隐藏对象的创建细节

## 实现示例

让我们通过一个计算机组装的例子来理解建造者模式。

### 1. 产品类

```java
public class Computer {
    private String cpu;        // CPU
    private String motherboard;// 主板
    private String memory;     // 内存
    private String storage;    // 存储
    private String gpu;        // 显卡
    private String power;      // 电源

    public void setCpu(String cpu) {
        this.cpu = cpu;
    }

    public void setMotherboard(String motherboard) {
        this.motherboard = motherboard;
    }

    public void setMemory(String memory) {
        this.memory = memory;
    }

    public void setStorage(String storage) {
        this.storage = storage;
    }

    public void setGpu(String gpu) {
        this.gpu = gpu;
    }

    public void setPower(String power) {
        this.power = power;
    }

    @Override
    public String toString() {
        return "Computer{" +
                "cpu='" + cpu + '\'' +
                ", motherboard='" + motherboard + '\'' +
                ", memory='" + memory + '\'' +
                ", storage='" + storage + '\'' +
                ", gpu='" + gpu + '\'' +
                ", power='" + power + '\'' +
                '}';
    }
}
```

### 2. 抽象建造者

```java
public abstract class ComputerBuilder {
    protected Computer computer = new Computer();

    public abstract void buildCPU();
    public abstract void buildMotherboard();
    public abstract void buildMemory();
    public abstract void buildStorage();
    public abstract void buildGPU();
    public abstract void buildPower();

    public Computer getResult() {
        return computer;
    }
}
```

### 3. 具体建造者

```java
// 游戏电脑建造者
public class GamingComputerBuilder extends ComputerBuilder {
    @Override
    public void buildCPU() {
        computer.setCpu("Intel i9 12900K");
    }

    @Override
    public void buildMotherboard() {
        computer.setMotherboard("ROG MAXIMUS Z690");
    }

    @Override
    public void buildMemory() {
        computer.setMemory("32GB DDR5 6000MHz");
    }

    @Override
    public void buildStorage() {
        computer.setStorage("2TB NVMe SSD");
    }

    @Override
    public void buildGPU() {
        computer.setGpu("NVIDIA RTX 4090");
    }

    @Override
    public void buildPower() {
        computer.setPower("1000W 金牌电源");
    }
}

// 办公电脑建造者
public class OfficeComputerBuilder extends ComputerBuilder {
    @Override
    public void buildCPU() {
        computer.setCpu("Intel i5 12400");
    }

    @Override
    public void buildMotherboard() {
        computer.setMotherboard("B660M");
    }

    @Override
    public void buildMemory() {
        computer.setMemory("16GB DDR4 3200MHz");
    }

    @Override
    public void buildStorage() {
        computer.setStorage("512GB SSD");
    }

    @Override
    public void buildGPU() {
        computer.setGpu("Intel UHD 730");
    }

    @Override
    public void buildPower() {
        computer.setPower("450W 铜牌电源");
    }
}
```

### 4. 指挥者

```java
public class Director {
    private ComputerBuilder builder;

    public Director(ComputerBuilder builder) {
        this.builder = builder;
    }

    public void constructComputer() {
        builder.buildCPU();
        builder.buildMotherboard();
        builder.buildMemory();
        builder.buildStorage();
        builder.buildGPU();
        builder.buildPower();
    }

    public Computer getComputer() {
        return builder.getResult();
    }
}
```

### 5. 客户端使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 创建游戏电脑
        ComputerBuilder gamingBuilder = new GamingComputerBuilder();
        Director director = new Director(gamingBuilder);
        director.constructComputer();
        Computer gamingComputer = director.getComputer();
        System.out.println("游戏电脑配置：" + gamingComputer);

        // 创建办公电脑
        ComputerBuilder officeBuilder = new OfficeComputerBuilder();
        director = new Director(officeBuilder);
        director.constructComputer();
        Computer officeComputer = director.getComputer();
        System.out.println("办公电脑配置：" + officeComputer);
    }
}
```

## 建造者模式的变体：链式调用

在实际开发中，我们经常会看到更简洁的建造者模式实现，特别是在配置对象时：

```java
public class Computer {
    private String cpu;
    private String motherboard;
    private String memory;
    // ... 其他属性

    public static class Builder {
        private Computer computer = new Computer();

        public Builder cpu(String cpu) {
            computer.cpu = cpu;
            return this;
        }

        public Builder motherboard(String motherboard) {
            computer.motherboard = motherboard;
            return this;
        }

        public Builder memory(String memory) {
            computer.memory = memory;
            return this;
        }

        public Computer build() {
            return computer;
        }
    }
}

// 使用示例
Computer computer = new Computer.Builder()
    .cpu("Intel i7")
    .motherboard("Z690")
    .memory("32GB")
    .build();
```

## 优点

1. 可以精细地控制产品的创建过程
2. 将复杂产品的创建步骤分解
3. 可以复用相同的创建代码
4. 遵循单一职责原则

## 缺点

1. 需要创建多个类，增加代码复杂度
2. 与工厂模式相比，更加重量级

## 适用场景

1. 需要创建的对象具有复杂的内部结构
2. 需要生成的对象内部属性之间的建造顺序有依赖关系
3. 对象的创建过程独立于创建该对象的类
4. 需要对象的创建过程具有更好的可控性

## 实际应用示例

1. StringBuilder类
2. Lombok的@Builder注解
3. Spring框架中的BeanDefinitionBuilder
4. Apache Camel的RouteBuilder

## 总结

建造者模式是一种非常实用的设计模式，特别适合用于创建复杂对象。它可以让我们更好地控制对象的创建过程，并且使代码更加清晰易读。在实际开发中，我们经常会使用其变体形式（链式调用），这种形式更加简洁优雅。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Effective Java》第三版
- Spring Framework 源码
- Lombok 文档

---

希望这篇文章能帮助您更好地理解Java中的建造者模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 