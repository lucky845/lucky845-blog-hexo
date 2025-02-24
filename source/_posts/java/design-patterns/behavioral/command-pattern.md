---
title: Java设计模式之命令模式详解
date: 2025-02-23 21:10:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 5f6e405f
---

## 什么是命令模式？

命令模式（Command Pattern）是一种行为型设计模式，它将请求封装成对象，从而可以用不同的请求对客户进行参数化，实现请求的排队、记录日志、撤销等功能。

## 为什么使用命令模式？

1. 将请求发送者和接收者解耦
2. 可以将命令存储和传递
3. 支持撤销/重做操作
4. 可以组合命令实现宏命令

## 实现示例

### 1. 基本实现

```java
// 命令接口
public interface Command {
    void execute();
    void undo();
}

// 接收者
public class Receiver {
    public void action() {
        System.out.println("Receiver执行请求");
    }
    
    public void undoAction() {
        System.out.println("Receiver撤销请求");
    }
}

// 具体命令
public class ConcreteCommand implements Command {
    private Receiver receiver;
    
    public ConcreteCommand(Receiver receiver) {
        this.receiver = receiver;
    }
    
    @Override
    public void execute() {
        receiver.action();
    }
    
    @Override
    public void undo() {
        receiver.undoAction();
    }
}

// 调用者
public class Invoker {
    private Command command;
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void executeCommand() {
        command.execute();
    }
    
    public void undoCommand() {
        command.undo();
    }
}
```

### 2. 实际应用示例：遥控器控制家电

```java
// 电器接口
public interface ElectricAppliance {
    void on();
    void off();
}

// 具体电器：电灯
public class Light implements ElectricAppliance {
    private String location;
    
    public Light(String location) {
        this.location = location;
    }
    
    @Override
    public void on() {
        System.out.println(location + "的灯打开了");
    }
    
    @Override
    public void off() {
        System.out.println(location + "的灯关闭了");
    }
}

// 具体电器：电视
public class Television implements ElectricAppliance {
    @Override
    public void on() {
        System.out.println("电视打开了");
    }
    
    @Override
    public void off() {
        System.out.println("电视关闭了");
    }
}

// 开启命令
public class TurnOnCommand implements Command {
    private ElectricAppliance appliance;
    
    public TurnOnCommand(ElectricAppliance appliance) {
        this.appliance = appliance;
    }
    
    @Override
    public void execute() {
        appliance.on();
    }
    
    @Override
    public void undo() {
        appliance.off();
    }
}

// 关闭命令
public class TurnOffCommand implements Command {
    private ElectricAppliance appliance;
    
    public TurnOffCommand(ElectricAppliance appliance) {
        this.appliance = appliance;
    }
    
    @Override
    public void execute() {
        appliance.off();
    }
    
    @Override
    public void undo() {
        appliance.on();
    }
}

// 遥控器
public class RemoteControl {
    private Command[] onCommands;
    private Command[] offCommands;
    private Command undoCommand;
    
    public RemoteControl() {
        onCommands = new Command[7];
        offCommands = new Command[7];
        
        Command noCommand = new NoCommand();
        for (int i = 0; i < 7; i++) {
            onCommands[i] = noCommand;
            offCommands[i] = noCommand;
        }
        undoCommand = noCommand;
    }
    
    public void setCommand(int slot, Command onCommand, Command offCommand) {
        onCommands[slot] = onCommand;
        offCommands[slot] = offCommand;
    }
    
    public void onButtonWasPushed(int slot) {
        onCommands[slot].execute();
        undoCommand = onCommands[slot];
    }
    
    public void offButtonWasPushed(int slot) {
        offCommands[slot].execute();
        undoCommand = offCommands[slot];
    }
    
    public void undoButtonWasPushed() {
        undoCommand.undo();
    }
}
```

### 3. 文本编辑器示例

```java
// 文本编辑器
public class TextEditor {
    private StringBuilder content = new StringBuilder();
    
    public void insert(String text) {
        content.append(text);
    }
    
    public void delete(int length) {
        content.delete(content.length() - length, content.length());
    }
    
    public String getContent() {
        return content.toString();
    }
}

// 插入命令
public class InsertCommand implements Command {
    private TextEditor editor;
    private String text;
    
    public InsertCommand(TextEditor editor, String text) {
        this.editor = editor;
        this.text = text;
    }
    
    @Override
    public void execute() {
        editor.insert(text);
    }
    
    @Override
    public void undo() {
        editor.delete(text.length());
    }
}

// 命令历史记录
public class CommandHistory {
    private Stack<Command> history = new Stack<>();
    
    public void push(Command command) {
        history.push(command);
    }
    
    public Command pop() {
        return history.isEmpty() ? null : history.pop();
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 遥控器示例
        RemoteControl remote = new RemoteControl();
        
        Light livingRoomLight = new Light("客厅");
        Light kitchenLight = new Light("厨房");
        Television tv = new Television();
        
        Command livingRoomLightOn = new TurnOnCommand(livingRoomLight);
        Command livingRoomLightOff = new TurnOffCommand(livingRoomLight);
        Command kitchenLightOn = new TurnOnCommand(kitchenLight);
        Command kitchenLightOff = new TurnOffCommand(kitchenLight);
        Command tvOn = new TurnOnCommand(tv);
        Command tvOff = new TurnOffCommand(tv);
        
        remote.setCommand(0, livingRoomLightOn, livingRoomLightOff);
        remote.setCommand(1, kitchenLightOn, kitchenLightOff);
        remote.setCommand(2, tvOn, tvOff);
        
        remote.onButtonWasPushed(0);  // 打开客厅灯
        remote.offButtonWasPushed(0); // 关闭客厅灯
        remote.undoButtonWasPushed(); // 撤销上一个操作
        
        // 文本编辑器示例
        TextEditor editor = new TextEditor();
        CommandHistory history = new CommandHistory();
        
        Command insertHello = new InsertCommand(editor, "Hello, ");
        Command insertWorld = new InsertCommand(editor, "World!");
        
        insertHello.execute();
        history.push(insertHello);
        
        insertWorld.execute();
        history.push(insertWorld);
        
        System.out.println(editor.getContent()); // 输出: Hello, World!
        
        Command lastCommand = history.pop();
        lastCommand.undo();
        System.out.println(editor.getContent()); // 输出: Hello, 
    }
}
```

## 命令模式的优点

1. 降低系统的耦合度
2. 新的命令可以很容易地加入到系统中
3. 可以比较容易地设计一个命令队列和宏命令
4. 可以方便地实现对请求的撤销和重做

## 命令模式的缺点

1. 可能会导致某些系统有过多的具体命令类
2. 增加了系统的复杂度

## 适用场景

1. 需要将请求发送者和接收者解耦
2. 需要将请求排队或者记录请求日志
3. 需要支持撤销操作
4. 需要支持事务操作

## 与其他模式的关系

1. 责任链模式：两者都可以处理请求，但命令模式更注重请求本身
2. 备忘录模式：可以结合使用来实现撤销操作
3. 策略模式：命令模式关注请求的封装，策略模式关注算法的封装

## 总结

命令模式是一种非常实用的设计模式，它通过将请求封装成对象，实现了请求发送者和接收者的解耦。这种模式在需要支持撤销、重做、日志记录等功能时特别有用。在实际开发中，当需要将请求参数化并支持这些高级功能时，命令模式是一个很好的选择。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Spring Framework 源码
- Java核心技术

---

希望这篇文章能帮助您更好地理解Java中的命令模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 