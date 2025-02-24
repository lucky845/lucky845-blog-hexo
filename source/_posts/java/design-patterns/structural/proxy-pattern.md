---
title: Java设计模式之代理模式详解
date: 2025-02-23 20:50:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 3f4e205f
---

## 什么是代理模式？

代理模式（Proxy Pattern）是一种结构型设计模式，它允许你提供一个代理来控制对其他对象的访问。代理对象可以在客户端和目标对象之间起到中介的作用，并且可以添加额外的功能。

## 为什么使用代理模式？

1. 控制对对象的访问
2. 在访问对象时添加额外的功能
3. 延迟加载
4. 权限控制

## 代理模式的类型

1. 静态代理
2. 动态代理（JDK动态代理）
3. CGLIB代理

## 实现示例

### 1. 静态代理

```java
// 共同的接口
public interface Subject {
    void request();
}

// 真实主题
public class RealSubject implements Subject {
    @Override
    public void request() {
        System.out.println("RealSubject处理请求");
    }
}

// 代理类
public class Proxy implements Subject {
    private RealSubject realSubject;
    
    public Proxy() {
        this.realSubject = new RealSubject();
    }
    
    @Override
    public void request() {
        System.out.println("代理开始");
        realSubject.request();
        System.out.println("代理结束");
    }
}
```

### 2. JDK动态代理

```java
// 接口
public interface UserService {
    void addUser(String userId, String userName);
    void deleteUser(String userId);
}

// 实现类
public class UserServiceImpl implements UserService {
    @Override
    public void addUser(String userId, String userName) {
        System.out.println("添加用户: " + userName);
    }
    
    @Override
    public void deleteUser(String userId) {
        System.out.println("删除用户: " + userId);
    }
}

// 动态代理处理器
public class LogHandler implements InvocationHandler {
    private Object target;
    
    public LogHandler(Object target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("开始执行方法: " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("方法执行结束");
        return result;
    }
    
    public static Object createProxy(Object target) {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new LogHandler(target)
        );
    }
}
```

### 3. CGLIB代理

```java
// 目标类
public class UserDao {
    public void save(User user) {
        System.out.println("保存用户: " + user.getName());
    }
    
    public void delete(String userId) {
        System.out.println("删除用户: " + userId);
    }
}

// CGLIB代理
public class CglibProxy implements MethodInterceptor {
    private Object target;
    
    public CglibProxy(Object target) {
        this.target = target;
    }
    
    public Object getProxyInstance() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }
    
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) 
            throws Throwable {
        System.out.println("开始执行方法: " + method.getName());
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("方法执行结束");
        return result;
    }
}
```

### 4. 实际应用示例：远程代理

```java
// 远程服务接口
public interface RemoteService {
    String getData(String key);
}

// 远程服务实现
public class RemoteServiceImpl implements RemoteService {
    @Override
    public String getData(String key) {
        // 模拟远程调用
        return "Data for key: " + key;
    }
}

// 远程代理
public class RemoteProxy implements RemoteService {
    private RemoteService remoteService;
    
    public RemoteProxy() {
        // 模拟RPC获取远程服务实例
        this.remoteService = new RemoteServiceImpl();
    }
    
    @Override
    public String getData(String key) {
        try {
            System.out.println("准备远程调用");
            String result = remoteService.getData(key);
            System.out.println("远程调用成功");
            return result;
        } catch (Exception e) {
            System.out.println("远程调用失败: " + e.getMessage());
            return null;
        }
    }
}
```

### 5. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 静态代理
        Subject proxy = new Proxy();
        proxy.request();
        
        // JDK动态代理
        UserService userService = new UserServiceImpl();
        UserService proxyService = (UserService) LogHandler.createProxy(userService);
        proxyService.addUser("1", "张三");
        
        // CGLIB代理
        UserDao userDao = new UserDao();
        UserDao proxyDao = (UserDao) new CglibProxy(userDao).getProxyInstance();
        proxyDao.save(new User("李四"));
        
        // 远程代理
        RemoteService remoteProxy = new RemoteProxy();
        String data = remoteProxy.getData("test");
        System.out.println(data);
    }
}
```

## 代理模式的优点

1. 职责清晰
2. 高扩展性
3. 智能化
4. 保护目标对象

## 代理模式的缺点

1. 在客户端和目标对象之间增加代理对象，可能会降低系统性能
2. 实现代理模式需要额外的工作
3. 有些代理模式的实现非常复杂

## 适用场景

1. 远程代理：为远程对象提供代理
2. 虚拟代理：延迟加载
3. 保护代理：控制对对象的访问
4. 智能引用：在访问对象时添加额外的操作

## 与其他模式的关系

1. 装饰器模式：装饰器为对象添加功能，代理控制对对象的访问
2. 适配器模式：适配器提供不同接口，代理提供相同接口
3. 外观模式：外观模式简化接口，代理模式控制访问

## 总结

代理模式是一种非常实用的设计模式，它可以在不修改原有代码的情况下，通过代理对象来控制对目标对象的访问。Java中提供了多种代理实现方式，包括静态代理、JDK动态代理和CGLIB代理，可以根据具体需求选择合适的实现方式。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Spring AOP 文档
- Java动态代理机制分析

---

希望这篇文章能帮助您更好地理解Java中的代理模式。如果您有任何问题，欢迎在评论区讨论！ 
---
 