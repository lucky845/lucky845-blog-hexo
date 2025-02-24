---
title: Java设计模式之中介者模式详解
date: 2025-02-23 21:40:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 8f9e705f
---

## 什么是中介者模式？

中介者模式（Mediator Pattern）是一种行为型设计模式，它用一个中介对象来封装一系列对象之间的交互。中介者使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。

## 为什么使用中介者模式？

1. 降低系统对象之间的耦合度
2. 简化对象之间的交互
3. 集中管理对象之间的交互
4. 提高系统的可维护性

## 实现示例

### 1. 基本实现

```java
// 中介者接口
public interface Mediator {
    void sendMessage(String message, Colleague colleague);
}

// 抽象同事类
public abstract class Colleague {
    protected Mediator mediator;
    
    public Colleague(Mediator mediator) {
        this.mediator = mediator;
    }
    
    public abstract void send(String message);
    public abstract void receive(String message);
}

// 具体中介者
public class ConcreteMediator implements Mediator {
    private ConcreteColleagueA colleagueA;
    private ConcreteColleagueB colleagueB;
    
    public void setColleagueA(ConcreteColleagueA colleague) {
        this.colleagueA = colleague;
    }
    
    public void setColleagueB(ConcreteColleagueB colleague) {
        this.colleagueB = colleague;
    }
    
    @Override
    public void sendMessage(String message, Colleague colleague) {
        if (colleague == colleagueA) {
            colleagueB.receive(message);
        } else {
            colleagueA.receive(message);
        }
    }
}

// 具体同事类A
public class ConcreteColleagueA extends Colleague {
    public ConcreteColleagueA(Mediator mediator) {
        super(mediator);
    }
    
    @Override
    public void send(String message) {
        mediator.sendMessage(message, this);
    }
    
    @Override
    public void receive(String message) {
        System.out.println("同事A收到消息：" + message);
    }
}

// 具体同事类B
public class ConcreteColleagueB extends Colleague {
    public ConcreteColleagueB(Mediator mediator) {
        super(mediator);
    }
    
    @Override
    public void send(String message) {
        mediator.sendMessage(message, this);
    }
    
    @Override
    public void receive(String message) {
        System.out.println("同事B收到消息：" + message);
    }
}
```

### 2. 聊天室示例

```java
// 聊天室中介者
public interface ChatMediator {
    void sendMessage(String message, User user);
    void addUser(User user);
}

// 聊天室实现
public class ChatRoom implements ChatMediator {
    private List<User> users = new ArrayList<>();
    
    @Override
    public void addUser(User user) {
        users.add(user);
    }
    
    @Override
    public void sendMessage(String message, User user) {
        for (User u : users) {
            if (u != user) {
                u.receive(message);
            }
        }
    }
}

// 用户抽象类
public abstract class User {
    protected ChatMediator mediator;
    protected String name;
    
    public User(ChatMediator mediator, String name) {
        this.mediator = mediator;
        this.name = name;
    }
    
    public abstract void send(String message);
    public abstract void receive(String message);
}

// 普通用户
public class NormalUser extends User {
    public NormalUser(ChatMediator mediator, String name) {
        super(mediator, name);
    }
    
    @Override
    public void send(String message) {
        System.out.println(name + " 发送消息: " + message);
        mediator.sendMessage(message, this);
    }
    
    @Override
    public void receive(String message) {
        System.out.println(name + " 收到消息: " + message);
    }
}
```

### 3. 航空管制示例

```java
// 航空管制中介者
public interface AirTrafficControl {
    void registerFlight(Flight flight);
    void requestLanding(Flight flight);
    void requestTakeoff(Flight flight);
}

// 航空管制塔
public class ControlTower implements AirTrafficControl {
    private List<Flight> flights = new ArrayList<>();
    private Queue<Flight> landingQueue = new LinkedList<>();
    private Queue<Flight> takeoffQueue = new LinkedList<>();
    
    @Override
    public void registerFlight(Flight flight) {
        flights.add(flight);
    }
    
    @Override
    public void requestLanding(Flight flight) {
        landingQueue.offer(flight);
        processNextOperation();
    }
    
    @Override
    public void requestTakeoff(Flight flight) {
        takeoffQueue.offer(flight);
        processNextOperation();
    }
    
    private void processNextOperation() {
        if (!landingQueue.isEmpty()) {
            Flight flight = landingQueue.poll();
            System.out.println("允许" + flight.getFlightNumber() + "降落");
            flight.land();
        } else if (!takeoffQueue.isEmpty()) {
            Flight flight = takeoffQueue.poll();
            System.out.println("允许" + flight.getFlightNumber() + "起飞");
            flight.takeoff();
        }
    }
}

// 航班类
public class Flight {
    private String flightNumber;
    private AirTrafficControl controlTower;
    
    public Flight(String flightNumber, AirTrafficControl controlTower) {
        this.flightNumber = flightNumber;
        this.controlTower = controlTower;
        controlTower.registerFlight(this);
    }
    
    public void requestLanding() {
        System.out.println(flightNumber + "请求降落");
        controlTower.requestLanding(this);
    }
    
    public void requestTakeoff() {
        System.out.println(flightNumber + "请求起飞");
        controlTower.requestTakeoff(this);
    }
    
    public void land() {
        System.out.println(flightNumber + "已降落");
    }
    
    public void takeoff() {
        System.out.println(flightNumber + "已起飞");
    }
    
    public String getFlightNumber() {
        return flightNumber;
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        ConcreteMediator mediator = new ConcreteMediator();
        ConcreteColleagueA colleagueA = new ConcreteColleagueA(mediator);
        ConcreteColleagueB colleagueB = new ConcreteColleagueB(mediator);
        
        mediator.setColleagueA(colleagueA);
        mediator.setColleagueB(colleagueB);
        
        colleagueA.send("你好，B!");
        colleagueB.send("你好，A!");
        
        // 聊天室示例
        ChatMediator chatRoom = new ChatRoom();
        User user1 = new NormalUser(chatRoom, "张三");
        User user2 = new NormalUser(chatRoom, "李四");
        User user3 = new NormalUser(chatRoom, "王五");
        
        chatRoom.addUser(user1);
        chatRoom.addUser(user2);
        chatRoom.addUser(user3);
        
        user1.send("大家好！");
        
        // 航空管制示例
        AirTrafficControl controlTower = new ControlTower();
        Flight flight1 = new Flight("CA1234", controlTower);
        Flight flight2 = new Flight("MU5678", controlTower);
        
        flight1.requestLanding();
        flight2.requestTakeoff();
    }
}
```

## 中介者模式的优点

1. 减少了对象之间的耦合，使得对象易于独立地改变和复用
2. 将对象间的交互封装到中介者中，使得交互逻辑集中化
3. 简化了对象间的关系，将多对多转化为一对多
4. 提高了系统的可维护性

## 中介者模式的缺点

1. 中介者可能会变得过于复杂
2. 中介者承担了较多的责任，可能会导致中介者类变得庞大
3. 如果设计不当，中介者对象本身就可能产生过度耦合

## 适用场景

1. 一组对象以定义良好但复杂的方式进行通信
2. 一些对象之间的通信方式必须定制化
3. 多个类相互耦合形成了网状结构
4. 想要集中管理对象间的交互

## 与其他模式的关系

1. 观察者模式：中介者模式通常用观察者模式实现
2. 外观模式：中介者是对等的对象之间的交互，外观是单向的访问
3. 命令模式：可以结合使用，中介者作为命令的接收者

## 总结

中介者模式是一种用于降低多个对象之间复杂关系的设计模式。它通过引入一个中介者对象，将系统中对象之间的直接交互转化为中介者对象协调的间接交互。这种模式在处理复杂的对象间通信时特别有用，如聊天室、航空管制等场景。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Java API 文档
- Spring Framework 源码

---

希望这篇文章能帮助您更好地理解Java中的中介者模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 