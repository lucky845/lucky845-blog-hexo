---
title: 【Spring Boot】如何解决 Bean 装配过程中的循环依赖
tags:
  - Java
  - Spring Boot
  - Bean 装配
  - 循环依赖
categories:
  - Java
  - Spring
abbrlink: b55fa575
date: 2024-02-25 06:00:00
---

## 问题背景

在 Spring Boot 中，Bean 的装配是通过依赖注入实现的。然而，当两个或多个 Bean 互相依赖时，就会出现循环依赖的问题。这种情况会导致 Spring 容器无法正确创建 Bean，从而抛出 `BeanCurrentlyInCreationException` 异常。本文将介绍如何识别和解决 Spring Boot 中的循环依赖问题。

## 循环依赖的类型

在 Spring 中，循环依赖主要有两种类型：

1. **构造器循环依赖**：两个或多个 Bean 通过构造器相互依赖。
2. **属性循环依赖**：两个或多个 Bean 通过 setter 方法相互依赖。

## 示例代码

### 1. 构造器循环依赖示例

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class A {
    private final B b;

    @Autowired
    public A(B b) {
        this.b = b;
    }
}

@Component
public class B {
    private final A a;

    @Autowired
    public B(A a) {
        this.a = a;
    }
}
```

在这个例子中，`A` 和 `B` 互相依赖，导致构造器循环依赖。

### 2. 属性循环依赖示例

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class C {
    @Autowired
    private D d;
}

@Component
public class D {
    @Autowired
    private C c;
}
```

在这个例子中，`C` 和 `D` 通过属性注入互相依赖，导致属性循环依赖。

## 解决方案

### 1. 使用 `@Lazy` 注解

使用 `@Lazy` 注解可以延迟 Bean 的初始化，从而解决循环依赖问题。可以在构造器或属性上使用 `@Lazy` 注解。

#### 构造器循环依赖解决方案

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Component;

@Component
public class A {
    private final B b;

    @Autowired
    public A(@Lazy B b) {
        this.b = b;
    }
}

@Component
public class B {
    private final A a;

    @Autowired
    public B(@Lazy A a) {
        this.a = a;
    }
}
```

#### 属性循环依赖解决方案

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Component;

@Component
public class C {
    @Autowired
    @Lazy
    private D d;
}

@Component
public class D {
    @Autowired
    @Lazy
    private C c;
}
```

### 2. 使用 Setter 注入

将构造器注入改为 setter 注入，可以避免构造器循环依赖的问题。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class A {
    private B b;

    @Autowired
    public void setB(B b) {
        this.b = b;
    }
}

@Component
public class B {
    private A a;

    @Autowired
    public void setA(A a) {
        this.a = a;
    }
}
```

### 3. 重构代码

在某些情况下，循环依赖可能表明代码设计不合理。考虑重构代码以消除循环依赖。例如，可以将某些逻辑提取到新的服务类中，减少 Bean 之间的直接依赖。

## 最佳实践建议

1. **避免循环依赖**：在设计 Bean 时，尽量避免循环依赖的情况，保持 Bean 之间的清晰关系。

2. **使用接口**：通过接口来解耦 Bean 之间的依赖关系，减少直接依赖。

3. **使用 `@Lazy` 注解**：在确实需要的情况下，使用 `@Lazy` 注解来解决循环依赖。

4. **重构代码**：如果发现循环依赖，考虑重构代码以改善设计。

## 常见问题

1. **如何检测循环依赖？**
   - Spring 启动时会抛出 `BeanCurrentlyInCreationException` 异常，通常表示存在循环依赖。

2. **如何处理复杂的循环依赖？**
   - 对于复杂的循环依赖，考虑使用设计模式（如观察者模式）来解耦。

3. **使用 `@Lazy` 注解会影响性能吗？**
   - 使用 `@Lazy` 注解会导致 Bean 的初始化延迟，可能会影响性能，但在解决循环依赖时是一个有效的解决方案。

## 总结

通过以上方法，我们可以有效地解决 Spring Boot 中的 Bean 装配过程中的循环依赖问题。合理的设计和使用 Spring 提供的功能，可以提高应用程序的稳定性和可维护性。

## 参考资料

- [Spring Framework - Circular References](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-circular-references)
- [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)

---

希望这篇文章能帮助您更好地理解和解决 Spring Boot 中的循环依赖问题。如果您有任何问题，欢迎在评论区讨论！
--- 
 