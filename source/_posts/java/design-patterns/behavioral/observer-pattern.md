---
title: Java设计模式之观察者模式详解
date: 2025-02-23 21:30:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 7f8e605f
---

## 什么是观察者模式？

观察者模式（Observer Pattern）是一种行为型设计模式，它定义了一种一对多的依赖关系，让多个观察者对象同时监听某一个主题对象。这个主题对象在状态发生变化时，会通知所有观察者对象，使它们能够自动更新自己。

## 为什么使用观察者模式？

1. 实现了表示层和数据逻辑层的分离
2. 支持广播通信
3. 符合开闭原则
4. 建立了触发机制

## 实现示例

### 1. 基本实现

```java
// 观察者接口
public interface Observer {
    void update(String message);
}

// 主题接口
public interface Subject {
    void registerObserver(Observer observer);
    void removeObserver(Observer observer);
    void notifyObservers();
}

// 具体主题
public class ConcreteSubject implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private String message;
    
    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }
    
    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(message);
        }
    }
    
    public void setMessage(String message) {
        this.message = message;
        notifyObservers();
    }
}

// 具体观察者
public class ConcreteObserver implements Observer {
    private String name;
    
    public ConcreteObserver(String name) {
        this.name = name;
    }
    
    @Override
    public void update(String message) {
        System.out.println(name + " received message: " + message);
    }
}
```

### 2. 天气监测站示例

```java
// 天气数据
public class WeatherData implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private float temperature;
    private float humidity;
    private float pressure;
    
    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }
    
    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(temperature, humidity, pressure);
        }
    }
    
    public void setMeasurements(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        notifyObservers();
    }
}

// 显示设备接口
public interface DisplayElement {
    void display();
}

// 当前状况显示
public class CurrentConditionsDisplay implements Observer, DisplayElement {
    private float temperature;
    private float humidity;
    
    @Override
    public void update(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        display();
    }
    
    @Override
    public void display() {
        System.out.printf("Current conditions: %.1f°C and %.1f%% humidity%n", 
                temperature, humidity);
    }
}

// 统计信息显示
public class StatisticsDisplay implements Observer, DisplayElement {
    private List<Float> temperatures = new ArrayList<>();
    
    @Override
    public void update(float temperature, float humidity, float pressure) {
        temperatures.add(temperature);
        display();
    }
    
    @Override
    public void display() {
        float avg = temperatures.stream().reduce(0f, Float::sum) / temperatures.size();
        float max = Collections.max(temperatures);
        float min = Collections.min(temperatures);
        System.out.printf("Avg/Max/Min temperature: %.1f/%.1f/%.1f%n", 
                avg, max, min);
    }
}
```

### 3. 事件监听示例

```java
// 事件对象
public class Event {
    private String type;
    private Object data;
    
    public Event(String type, Object data) {
        this.type = type;
        this.data = data;
    }
    
    public String getType() {
        return type;
    }
    
    public Object getData() {
        return data;
    }
}

// 事件发布者
public class EventPublisher {
    private Map<String, List<EventListener>> listeners = new HashMap<>();
    
    public void addEventListener(String eventType, EventListener listener) {
        listeners.computeIfAbsent(eventType, k -> new ArrayList<>()).add(listener);
    }
    
    public void removeEventListener(String eventType, EventListener listener) {
        if (listeners.containsKey(eventType)) {
            listeners.get(eventType).remove(listener);
        }
    }
    
    public void publishEvent(Event event) {
        if (listeners.containsKey(event.getType())) {
            listeners.get(event.getType()).forEach(listener -> listener.onEvent(event));
        }
    }
}

// 事件监听器接口
public interface EventListener {
    void onEvent(Event event);
}

// 具体监听器
public class LogEventListener implements EventListener {
    @Override
    public void onEvent(Event event) {
        System.out.println("Log: " + event.getType() + " - " + event.getData());
    }
}

public class EmailEventListener implements EventListener {
    @Override
    public void onEvent(Event event) {
        System.out.println("Sending email about: " + event.getType());
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        ConcreteSubject subject = new ConcreteSubject();
        ConcreteObserver observer1 = new ConcreteObserver("Observer 1");
        ConcreteObserver observer2 = new ConcreteObserver("Observer 2");
        
        subject.registerObserver(observer1);
        subject.registerObserver(observer2);
        subject.setMessage("Hello Observers!");
        
        // 天气监测站示例
        WeatherData weatherData = new WeatherData();
        CurrentConditionsDisplay currentDisplay = new CurrentConditionsDisplay();
        StatisticsDisplay statisticsDisplay = new StatisticsDisplay();
        
        weatherData.registerObserver(currentDisplay);
        weatherData.registerObserver(statisticsDisplay);
        
        weatherData.setMeasurements(25.2f, 65.0f, 1013.1f);
        weatherData.setMeasurements(26.5f, 70.0f, 1012.5f);
        
        // 事件监听示例
        EventPublisher publisher = new EventPublisher();
        publisher.addEventListener("login", new LogEventListener());
        publisher.addEventListener("login", new EmailEventListener());
        
        publisher.publishEvent(new Event("login", "User logged in"));
    }
}
```

## 观察者模式的优点

1. 观察者和被观察者之间是松耦合的
2. 支持广播通信
3. 符合开闭原则
4. 可以建立一套触发机制

## 观察者模式的缺点

1. 如果观察者太多，通知所有观察者会花费较多时间
2. 如果观察者和被观察者之间有循环依赖，可能导致系统崩溃
3. 观察者接收通知的顺序是不确定的

## 适用场景

1. 当一个对象的改变需要同时改变其他对象时
2. 当一个对象必须通知其他对象，而它又不知道这些对象是谁时
3. 需要建立一个一对多的依赖关系时
4. 当系统需要分离观察者和被观察者时

## 与其他模式的关系

1. 中介者模式：观察者模式用于对象间的一对多通信，中介者模式用于多对多通信
2. 单例模式：主题对象通常是单例的
3. 策略模式：可以使用策略模式来改变观察者的更新行为

## 总结

观察者模式是一种使用非常广泛的设计模式，它在事件处理系统、用户界面设计、消息推送等场景中都有重要应用。Java的事件处理机制、Swing的事件模型等都使用了观察者模式。在实际开发中，当需要实现对象间的一对多依赖关系时，观察者模式是一个很好的选择。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Java API 文档
- Spring Framework 事件机制

---

希望这篇文章能帮助您更好地理解Java中的观察者模式。如果您有任何问题，欢迎在评论区讨论！ 
abbrlink: '0'
---
 