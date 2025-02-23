---
title: Java设计模式之责任链模式详解
date: 2025-02-23 22:30:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 3f4e205f
---

## 什么是责任链模式？

责任链模式（Chain of Responsibility Pattern）是一种行为型设计模式，它通过为请求创建一个接收者对象的链来避免请求发送者与接收者耦合。这种模式将接收对象连成一条链，并沿着这条链传递请求，直到有一个对象处理它为止。

## 为什么使用责任链模式？

1. 降低耦合度
2. 简化对象间的连接
3. 增强给对象指派职责的灵活性
4. 增加新的请求处理类很方便

## 实现示例

### 1. 基本实现

```java
// 处理器接口
public abstract class Handler {
    protected Handler successor;
    
    public void setSuccessor(Handler successor) {
        this.successor = successor;
    }
    
    public abstract void handleRequest(Request request);
}

// 请求类
public class Request {
    private int type;
    private String content;
    
    public Request(int type, String content) {
        this.type = type;
        this.content = content;
    }
    
    public int getType() {
        return type;
    }
    
    public String getContent() {
        return content;
    }
}

// 具体处理器A
public class ConcreteHandlerA extends Handler {
    @Override
    public void handleRequest(Request request) {
        if (request.getType() == 1) {
            System.out.println("处理器A处理请求：" + request.getContent());
        } else if (successor != null) {
            successor.handleRequest(request);
        }
    }
}

// 具体处理器B
public class ConcreteHandlerB extends Handler {
    @Override
    public void handleRequest(Request request) {
        if (request.getType() == 2) {
            System.out.println("处理器B处理请求：" + request.getContent());
        } else if (successor != null) {
            successor.handleRequest(request);
        }
    }
}
```

### 2. 日志级别示例

```java
// 日志处理器
public abstract class LogHandler {
    protected LogHandler nextHandler;
    protected int level;
    
    public void setNextHandler(LogHandler nextHandler) {
        this.nextHandler = nextHandler;
    }
    
    public void logMessage(int level, String message) {
        if (this.level <= level) {
            write(message);
        }
        if (nextHandler != null) {
            nextHandler.logMessage(level, message);
        }
    }
    
    protected abstract void write(String message);
}

// 控制台日志处理器
public class ConsoleLogger extends LogHandler {
    public ConsoleLogger(int level) {
        this.level = level;
    }
    
    @Override
    protected void write(String message) {
        System.out.println("Console::Logger: " + message);
    }
}

// 文件日志处理器
public class FileLogger extends LogHandler {
    public FileLogger(int level) {
        this.level = level;
    }
    
    @Override
    protected void write(String message) {
        System.out.println("File::Logger: " + message);
    }
}

// 错误日志处理器
public class ErrorLogger extends LogHandler {
    public ErrorLogger(int level) {
        this.level = level;
    }
    
    @Override
    protected void write(String message) {
        System.out.println("Error::Logger: " + message);
    }
}
```

### 3. 请求处理示例

```java
// 请求处理器
public abstract class RequestHandler {
    protected RequestHandler next;
    
    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next;
    }
    
    public abstract void handleRequest(HttpRequest request);
}

// 认证处理器
public class AuthenticationHandler extends RequestHandler {
    @Override
    public void handleRequest(HttpRequest request) {
        if (!request.isAuthenticated()) {
            System.out.println("认证失败");
            return;
        }
        System.out.println("认证成功");
        if (next != null) {
            next.handleRequest(request);
        }
    }
}

// 授权处理器
public class AuthorizationHandler extends RequestHandler {
    @Override
    public void handleRequest(HttpRequest request) {
        if (!request.hasPermission()) {
            System.out.println("没有权限");
            return;
        }
        System.out.println("授权成功");
        if (next != null) {
            next.handleRequest(request);
        }
    }
}

// 缓存处理器
public class CacheHandler extends RequestHandler {
    @Override
    public void handleRequest(HttpRequest request) {
        if (request.existsInCache()) {
            System.out.println("从缓存返回数据");
            return;
        }
        System.out.println("继续处理请求");
        if (next != null) {
            next.handleRequest(request);
        }
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        Handler handlerA = new ConcreteHandlerA();
        Handler handlerB = new ConcreteHandlerB();
        handlerA.setSuccessor(handlerB);
        
        handlerA.handleRequest(new Request(1, "请求1"));
        handlerA.handleRequest(new Request(2, "请求2"));
        
        // 日志级别示例
        LogHandler errorLogger = new ErrorLogger(LogLevel.ERROR);
        LogHandler fileLogger = new FileLogger(LogLevel.DEBUG);
        LogHandler consoleLogger = new ConsoleLogger(LogLevel.INFO);
        
        errorLogger.setNextHandler(fileLogger);
        fileLogger.setNextHandler(consoleLogger);
        
        errorLogger.logMessage(LogLevel.ERROR, "错误信息");
        errorLogger.logMessage(LogLevel.DEBUG, "调试信息");
        
        // 请求处理示例
        RequestHandler authentication = new AuthenticationHandler();
        RequestHandler authorization = new AuthorizationHandler();
        RequestHandler cache = new CacheHandler();
        
        authentication.setNext(authorization).setNext(cache);
        
        HttpRequest request = new HttpRequest();
        authentication.handleRequest(request);
    }
}
```

## 责任链模式的优点

1. 降低耦合度
2. 简化了对象之间的连接
3. 增强了给对象指派职责的灵活性
4. 增加新的处理类很方便

## 责任链模式的缺点

1. 不能保证请求一定被处理
2. 系统性能会受到影响
3. 可能不容易观察运行时的特征

## 适用场景

1. 有多个对象可以处理同一个请求，具体由哪个对象处理该请求待运行时确定
2. 在不明确指定接收者的情况下，向多个对象中的一个提交一个请求
3. 可动态指定一组对象处理请求
4. 需要在不同的处理器之间传递请求

## 与其他模式的关系

1. 命令模式：责任链可以与命令模式结合使用
2. 装饰器模式：责任链模式是行为模式，装饰器是结构模式
3. 组合模式：责任链通常和组合模式一起使用

## 总结

责任链模式是一种非常实用的设计模式，它通过建立一条处理请求的链来降低请求的发送者和接收者之间的耦合关系。这种模式在处理复杂的请求流程时特别有用，如日志记录、请求过滤等场景。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Spring Security 文档
- Java Servlet Filter 机制

---

希望这篇文章能帮助您更好地理解Java中的责任链模式。如果您有任何问题，欢迎在评论区讨论！ 
abbrlink: '0'
---
 