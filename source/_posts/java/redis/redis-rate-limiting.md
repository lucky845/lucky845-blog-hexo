---
title: 【Spring Boot】使用 Redis 实现限流操作
tags:
  - Java
  - Spring Boot
  - Redis
  - 限流
  - AOP
categories:
  - Java
  - Spring
abbrlink: c5fa583
date: 2025-02-25 14:00:00
---

## 问题背景

在高并发的场景下，限流是一种有效的保护措施，可以防止系统过载。Redis 作为一个高性能的内存数据库，常被用作限流的存储方案。本文将介绍如何使用 Redis 实现限流操作，并结合 AOP 和注解来简化限流逻辑的实现。

## 1. 添加依赖

首先，确保在 `pom.xml` 中添加 Redis 和 AOP 的相关依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

## 2. 配置 Redis

在 `application.yml` 中配置 Redis 连接信息：

```yaml
spring:
  redis:
    host: localhost
    port: 6379
```

## 3. 创建限流注解

我们将创建一个自定义注解 `@RateLimit`，用于标记需要限流的方法。

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    String key() default ""; // 限流的 key
    int limit() default 5;   // 限流的次数
    int timeout() default 60; // 限流的时间窗口（秒）
}
```

## 4. 实现限流逻辑

接下来，我们将使用 AOP 来实现限流逻辑。创建一个切面类 `RateLimitAspect`，在其中实现限流的具体逻辑。

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Aspect
@Component
public class RateLimitAspect {

    @Autowired
    private RedisTemplate<String, Integer> redisTemplate;

    @Around("@annotation(rateLimit)")
    public Object limit(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
        String key = rateLimit.key();
        int limit = rateLimit.limit();
        int timeout = rateLimit.timeout();

        // 使用 Redis 计数器
        Integer count = redisTemplate.opsForValue().get(key);
        if (count == null) {
            // 如果没有记录，初始化为 1
            redisTemplate.opsForValue().set(key, 1, timeout, TimeUnit.SECONDS);
        } else if (count < limit) {
            // 如果计数未超过限制，增加计数
            redisTemplate.opsForValue().increment(key);
        } else {
            // 超过限制，抛出异常或返回错误信息
            throw new RuntimeException("请求过于频繁，请稍后再试。");
        }

        // 继续执行目标方法
        return joinPoint.proceed();
    }
}
```

## 5. 使用限流注解

现在，我们可以在需要限流的方法上使用 `@RateLimit` 注解。例如：

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TestController {

    @GetMapping("/test")
    @RateLimit(key = "testRateLimit", limit = 5, timeout = 60)
    public String test() {
        return "请求成功！";
    }
}
```

在这个例子中，`/test` 接口每分钟最多允许 5 次请求。

## 6. 测试限流功能

启动 Spring Boot 应用程序，并使用 Postman 或其他工具对 `/test` 接口进行多次请求。您将看到在超过限制后，返回的错误信息。

## 总结

通过使用 Redis 和 AOP，我们可以轻松实现限流操作。自定义注解和切面使得限流逻辑的实现变得简单而优雅。合理的限流策略可以有效保护系统，防止过载。

## 参考资料

- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
- [Redis Documentation](https://redis.io/documentation)

---

希望这篇文章能帮助您更好地理解和实现使用 Redis 的限流操作。如果您有任何问题，欢迎在评论区讨论！
--- 