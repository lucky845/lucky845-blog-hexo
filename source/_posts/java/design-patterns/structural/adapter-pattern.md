---
title: Java设计模式之适配器模式详解
date: 2025-02-23 19:50:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 7f8e605f
---

## 什么是适配器模式？

适配器模式（Adapter Pattern）是一种结构型设计模式，它作为两个不兼容接口之间的桥梁，将一个类的接口转换成客户希望的另外一个接口。适配器模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

## 适配器模式的类型

1. 类适配器（使用继承）
2. 对象适配器（使用组合）

## 实现示例

### 1. 类适配器模式

```java
// 目标接口
public interface Target {
    void request();
}

// 被适配的类
public class Adaptee {
    public void specificRequest() {
        System.out.println("适配者的特殊请求");
    }
}

// 类适配器
public class ClassAdapter extends Adaptee implements Target {
    @Override
    public void request() {
        specificRequest();  // 调用父类的方法
    }
}
```

### 2. 对象适配器模式

```java
// 目标接口
public interface Target {
    void request();
}

// 被适配的类
public class Adaptee {
    public void specificRequest() {
        System.out.println("适配者的特殊请求");
    }
}

// 对象适配器
public class ObjectAdapter implements Target {
    private Adaptee adaptee;

    public ObjectAdapter(Adaptee adaptee) {
        this.adaptee = adaptee;
    }

    @Override
    public void request() {
        adaptee.specificRequest();  // 调用被适配对象的方法
    }
}
```

## 实际应用示例

### 1. 电源适配器示例

```java
// 美式电源接口
public interface AmericanSocket {
    void supplyPowerAt110V();
}

// 欧式电源接口
public interface EuropeanSocket {
    void supplyPowerAt220V();
}

// 美式电源实现
public class AmericanSocketImpl implements AmericanSocket {
    @Override
    public void supplyPowerAt110V() {
        System.out.println("提供110V电压");
    }
}

// 电源适配器
public class PowerAdapter implements EuropeanSocket {
    private AmericanSocket americanSocket;

    public PowerAdapter(AmericanSocket americanSocket) {
        this.americanSocket = americanSocket;
    }

    @Override
    public void supplyPowerAt220V() {
        System.out.println("使用适配器进行电压转换");
        americanSocket.supplyPowerAt110V();
        System.out.println("将110V转换为220V");
    }
}
```

### 2. 数据格式转换示例

```java
// JSON数据格式
public class JsonData {
    private String jsonString;

    public JsonData(String jsonString) {
        this.jsonString = jsonString;
    }

    public String getJsonString() {
        return jsonString;
    }
}

// XML数据格式
public interface XmlData {
    String getXmlString();
}

// JSON到XML的适配器
public class JsonToXmlAdapter implements XmlData {
    private JsonData jsonData;

    public JsonToXmlAdapter(JsonData jsonData) {
        this.jsonData = jsonData;
    }

    @Override
    public String getXmlString() {
        // 实际项目中这里会有真实的JSON到XML转换逻辑
        String json = jsonData.getJsonString();
        return String.format("<xml>%s</xml>", json);
    }
}
```

### 3. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 电源适配器示例
        AmericanSocket americanSocket = new AmericanSocketImpl();
        EuropeanSocket adapter = new PowerAdapter(americanSocket);
        adapter.supplyPowerAt220V();

        // 数据格式转换示例
        JsonData jsonData = new JsonData("{name: 'John', age: 30}");
        XmlData xmlAdapter = new JsonToXmlAdapter(jsonData);
        System.out.println(xmlAdapter.getXmlString());
    }
}
```

## 在Java标准库中的应用

Java标准库中有许多使用适配器模式的例子：

```java
// 输入流适配器示例
public class InputStreamExample {
    public void readFile() {
        try {
            // 将FileInputStream适配成Reader
            Reader reader = new InputStreamReader(new FileInputStream("test.txt"));
            BufferedReader bufferedReader = new BufferedReader(reader);
            String line = bufferedReader.readLine();
            System.out.println(line);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 适配器模式的优点

1. 将目标类和适配者类解耦
2. 增加了类的透明性
3. 提高了类的复用性
4. 灵活性好

## 适配器模式的缺点

1. 过多使用适配器会让系统变得凌乱
2. 可能会增加系统的复杂性
3. 适配器可能会增加系统的开销

## 适用场景

1. 系统需要使用现有的类，但这些类的接口不符合系统的需要
2. 想要建立一个可以重复使用的类，用于与一些彼此之间没有太大关联的类一起工作
3. 需要统一多个类的接口设计时
4. 旧系统改造和升级时

## 与其他模式的关系

1. 装饰器模式：装饰器模式更注重于动态地增加功能
2. 代理模式：代理模式不会改变接口，而适配器模式会改变接口
3. 外观模式：外观模式定义了一个新的接口，而适配器模式复用一个原有的接口

## 总结

适配器模式是一种使用频率很高的设计模式，它主要用于接口转换，使得原本由于接口不兼容而不能一起工作的类可以协同工作。在实际开发中，经常会遇到需要适配接口的情况，这时适配器模式就能派上用场。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Java API 文档
- Spring Framework 源码

---

希望这篇文章能帮助您更好地理解Java中的适配器模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 