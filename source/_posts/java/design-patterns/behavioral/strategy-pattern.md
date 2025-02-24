---
title: Java设计模式之策略模式详解
date: 2025-02-23 22:20:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 2f3e105f
---

## 什么是策略模式？

策略模式（Strategy Pattern）是一种行为型设计模式，它定义了一系列算法，并将每个算法封装起来，使它们可以相互替换。策略模式让算法独立于使用它的客户端而变化。

## 为什么使用策略模式？

1. 避免使用多重条件语句
2. 算法可以自由切换
3. 扩展性良好
4. 遵循开闭原则

## 实现示例

### 1. 基本实现

```java
// 策略接口
public interface Strategy {
    void execute();
}

// 具体策略A
public class ConcreteStrategyA implements Strategy {
    @Override
    public void execute() {
        System.out.println("执行策略A");
    }
}

// 具体策略B
public class ConcreteStrategyB implements Strategy {
    @Override
    public void execute() {
        System.out.println("执行策略B");
    }
}

// 上下文
public class Context {
    private Strategy strategy;
    
    public Context(Strategy strategy) {
        this.strategy = strategy;
    }
    
    public void setStrategy(Strategy strategy) {
        this.strategy = strategy;
    }
    
    public void executeStrategy() {
        strategy.execute();
    }
}
```

### 2. 支付方式示例

```java
// 支付策略接口
public interface PaymentStrategy {
    void pay(int amount);
}

// 信用卡支付
public class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    private String cvv;
    private String dateOfExpiry;
    
    public CreditCardPayment(String cardNumber, String cvv, String dateOfExpiry) {
        this.cardNumber = cardNumber;
        this.cvv = cvv;
        this.dateOfExpiry = dateOfExpiry;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println("使用信用卡支付：" + amount + "元");
    }
}

// 支付宝支付
public class AlipayPayment implements PaymentStrategy {
    private String emailId;
    
    public AlipayPayment(String emailId) {
        this.emailId = emailId;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println("使用支付宝支付：" + amount + "元");
    }
}

// 微信支付
public class WeChatPayment implements PaymentStrategy {
    private String phoneNumber;
    
    public WeChatPayment(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println("使用微信支付：" + amount + "元");
    }
}

// 购物车
public class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout(int amount) {
        paymentStrategy.pay(amount);
    }
}
```

### 3. 文件压缩示例

```java
// 压缩策略接口
public interface CompressionStrategy {
    void compressFiles(List<File> files);
}

// ZIP压缩
public class ZipCompression implements CompressionStrategy {
    @Override
    public void compressFiles(List<File> files) {
        System.out.println("使用ZIP格式压缩文件");
        // 具体的ZIP压缩实现
    }
}

// RAR压缩
public class RarCompression implements CompressionStrategy {
    @Override
    public void compressFiles(List<File> files) {
        System.out.println("使用RAR格式压缩文件");
        // 具体的RAR压缩实现
    }
}

// 7Z压缩
public class SevenZipCompression implements CompressionStrategy {
    @Override
    public void compressFiles(List<File> files) {
        System.out.println("使用7Z格式压缩文件");
        // 具体的7Z压缩实现
    }
}

// 压缩器
public class Compressor {
    private CompressionStrategy strategy;
    
    public void setCompressionStrategy(CompressionStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void compress(List<File> files) {
        strategy.compressFiles(files);
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        Context context = new Context(new ConcreteStrategyA());
        context.executeStrategy();
        
        context.setStrategy(new ConcreteStrategyB());
        context.executeStrategy();
        
        // 支付方式示例
        ShoppingCart cart = new ShoppingCart();
        
        cart.setPaymentStrategy(new CreditCardPayment("1234-5678", "123", "12/24"));
        cart.checkout(100);
        
        cart.setPaymentStrategy(new AlipayPayment("example@email.com"));
        cart.checkout(200);
        
        cart.setPaymentStrategy(new WeChatPayment("13800138000"));
        cart.checkout(300);
        
        // 文件压缩示例
        Compressor compressor = new Compressor();
        List<File> files = Arrays.asList(new File("file1.txt"), new File("file2.txt"));
        
        compressor.setCompressionStrategy(new ZipCompression());
        compressor.compress(files);
        
        compressor.setCompressionStrategy(new RarCompression());
        compressor.compress(files);
        
        compressor.setCompressionStrategy(new SevenZipCompression());
        compressor.compress(files);
    }
}
```

## 策略模式的优点

1. 算法可以自由切换
2. 避免使用多重条件判断
3. 扩展性良好
4. 遵循开闭原则

## 策略模式的缺点

1. 策略类会增多
2. 所有策略类都需要对外暴露
3. 客户端必须知道所有的策略类

## 适用场景

1. 需要动态地在几种算法中选择一种
2. 有多个类只区别在表现行为不同
3. 算法需要自由切换
4. 避免使用多重条件选择语句

## 与其他模式的关系

1. 状态模式：策略模式是算法的封装，状态模式是对象状态的封装
2. 命令模式：策略模式通常用来封装算法，命令模式用来封装行为或动作
3. 工厂模式：可以使用工厂模式来创建策略对象

## 总结

策略模式是一种非常实用的设计模式，它通过定义一系列算法，把它们一个个封装起来，并且使它们可以相互替换。这种模式在实际开发中经常用到，特别是在需要动态选择算法的场景下，如支付方式选择、文件压缩等。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Spring Framework 文档
- Java核心技术

---

希望这篇文章能帮助您更好地理解Java中的策略模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 