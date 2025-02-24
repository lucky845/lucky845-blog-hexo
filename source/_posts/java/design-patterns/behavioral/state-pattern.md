---
title: Java设计模式之状态模式详解
date: 2025-02-23 22:10:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 1f2e005f
---

## 什么是状态模式？

状态模式（State Pattern）是一种行为型设计模式，它允许一个对象在其内部状态改变时改变它的行为。状态模式将状态封装成独立的类，并将与状态相关的行为委托给代表当前状态的对象。

## 为什么使用状态模式？

1. 将状态相关的行为局部化
2. 使状态转换显式化
3. 消除庞大的条件分支语句
4. 更好地管理状态之间的转换

## 实现示例

### 1. 基本实现

```java
// 状态接口
public interface State {
    void handle();
}

// 具体状态A
public class ConcreteStateA implements State {
    private Context context;
    
    public ConcreteStateA(Context context) {
        this.context = context;
    }
    
    @Override
    public void handle() {
        System.out.println("当前是状态A");
        context.setState(new ConcreteStateB(context));
    }
}

// 具体状态B
public class ConcreteStateB implements State {
    private Context context;
    
    public ConcreteStateB(Context context) {
        this.context = context;
    }
    
    @Override
    public void handle() {
        System.out.println("当前是状态B");
        context.setState(new ConcreteStateA(context));
    }
}

// 环境类
public class Context {
    private State state;
    
    public Context() {
        state = new ConcreteStateA(this);
    }
    
    public void setState(State state) {
        this.state = state;
    }
    
    public void request() {
        state.handle();
    }
}
```

### 2. 自动售货机示例

```java
// 售货机状态接口
public interface VendingMachineState {
    void insertCoin();
    void ejectCoin();
    void selectProduct();
    void dispense();
}

// 没有硬币状态
public class NoCoinState implements VendingMachineState {
    private VendingMachine machine;
    
    public NoCoinState(VendingMachine machine) {
        this.machine = machine;
    }
    
    @Override
    public void insertCoin() {
        System.out.println("投入硬币");
        machine.setState(new HasCoinState(machine));
    }
    
    @Override
    public void ejectCoin() {
        System.out.println("没有硬币可退");
    }
    
    @Override
    public void selectProduct() {
        System.out.println("请先投入硬币");
    }
    
    @Override
    public void dispense() {
        System.out.println("请先投入硬币");
    }
}

// 有硬币状态
public class HasCoinState implements VendingMachineState {
    private VendingMachine machine;
    
    public HasCoinState(VendingMachine machine) {
        this.machine = machine;
    }
    
    @Override
    public void insertCoin() {
        System.out.println("已经投入了硬币");
    }
    
    @Override
    public void ejectCoin() {
        System.out.println("退回硬币");
        machine.setState(new NoCoinState(machine));
    }
    
    @Override
    public void selectProduct() {
        System.out.println("选择商品");
        machine.setState(new SoldState(machine));
    }
    
    @Override
    public void dispense() {
        System.out.println("请先选择商品");
    }
}

// 售出状态
public class SoldState implements VendingMachineState {
    private VendingMachine machine;
    
    public SoldState(VendingMachine machine) {
        this.machine = machine;
    }
    
    @Override
    public void insertCoin() {
        System.out.println("请等待商品发放");
    }
    
    @Override
    public void ejectCoin() {
        System.out.println("交易已经开始，无法退币");
    }
    
    @Override
    public void selectProduct() {
        System.out.println("已经选择了商品");
    }
    
    @Override
    public void dispense() {
        System.out.println("发放商品");
        machine.setState(new NoCoinState(machine));
    }
}

// 售货机
public class VendingMachine {
    private VendingMachineState state;
    
    public VendingMachine() {
        state = new NoCoinState(this);
    }
    
    public void setState(VendingMachineState state) {
        this.state = state;
    }
    
    public void insertCoin() {
        state.insertCoin();
    }
    
    public void ejectCoin() {
        state.ejectCoin();
    }
    
    public void selectProduct() {
        state.selectProduct();
    }
    
    public void dispense() {
        state.dispense();
    }
}
```

### 3. 订单状态示例

```java
// 订单状态接口
public interface OrderState {
    void nextState();
    void cancel();
    String getStatus();
}

// 新建订单状态
public class NewOrderState implements OrderState {
    private Order order;
    
    public NewOrderState(Order order) {
        this.order = order;
    }
    
    @Override
    public void nextState() {
        System.out.println("订单已支付");
        order.setState(new PaidOrderState(order));
    }
    
    @Override
    public void cancel() {
        System.out.println("取消订单");
        order.setState(new CancelledOrderState(order));
    }
    
    @Override
    public String getStatus() {
        return "新建";
    }
}

// 已支付状态
public class PaidOrderState implements OrderState {
    private Order order;
    
    public PaidOrderState(Order order) {
        this.order = order;
    }
    
    @Override
    public void nextState() {
        System.out.println("订单已发货");
        order.setState(new ShippedOrderState(order));
    }
    
    @Override
    public void cancel() {
        System.out.println("申请退款");
        order.setState(new RefundingOrderState(order));
    }
    
    @Override
    public String getStatus() {
        return "已支付";
    }
}

// 订单类
public class Order {
    private OrderState state;
    private String orderNumber;
    
    public Order(String orderNumber) {
        this.orderNumber = orderNumber;
        this.state = new NewOrderState(this);
    }
    
    public void setState(OrderState state) {
        this.state = state;
    }
    
    public void nextState() {
        state.nextState();
    }
    
    public void cancel() {
        state.cancel();
    }
    
    public String getStatus() {
        return state.getStatus();
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        Context context = new Context();
        context.request();  // 输出：当前是状态A
        context.request();  // 输出：当前是状态B
        context.request();  // 输出：当前是状态A
        
        // 售货机示例
        VendingMachine machine = new VendingMachine();
        machine.insertCoin();
        machine.selectProduct();
        machine.dispense();
        
        // 订单状态示例
        Order order = new Order("ORDER001");
        System.out.println("订单状态：" + order.getStatus());  // 新建
        
        order.nextState();
        System.out.println("订单状态：" + order.getStatus());  // 已支付
        
        order.nextState();
        System.out.println("订单状态：" + order.getStatus());  // 已发货
    }
}
```

## 状态模式的优点

1. 封装了状态的转换规则
2. 消除了庞大的条件分支语句
3. 将状态相关的行为局部化
4. 使状态转换显式化

## 状态模式的缺点

1. 可能会导致状态类的数量增多
2. 状态之间的转换关系容易变得复杂
3. 可能会增加系统的复杂度

## 适用场景

1. 对象的行为取决于其状态
2. 代码中包含大量与对象状态有关的条件语句
3. 需要在运行时根据状态改变对象的行为
4. 状态转换规则比较复杂

## 与其他模式的关系

1. 策略模式：状态模式是策略模式的一种变体
2. 单例模式：状态对象通常可以被共享，使用单例模式
3. 备忘录模式：可以使用备忘录模式保存状态对象的状态

## 总结

状态模式是一种用于管理对象状态及其相关行为的设计模式。它通过将状态相关的行为封装在独立的状态类中，使得对象在不同状态下能够改变其行为，同时又不会导致代码的混乱。这种模式在处理复杂的状态转换逻辑时特别有用。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Spring Statemachine 文档
- Java核心技术

---

希望这篇文章能帮助您更好地理解Java中的状态模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 