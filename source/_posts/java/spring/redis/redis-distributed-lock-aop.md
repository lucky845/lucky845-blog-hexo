---
title: 【Spring Boot】使用 AOP 实现分布式锁注解
tags:
  - Java
  - Spring Boot
  - Redis
  - 分布式锁
  - AOP
categories:
  - Java
  - Spring
abbrlink: b55fa582
date: 2025-02-25 15:00:00
---

## 问题背景

在分布式系统中，使用分布式锁可以有效地防止多个服务实例同时访问共享资源。为了简化分布式锁的使用，我们可以通过自定义注解和 AOP（面向切面编程）来实现分布式锁的功能。本文将介绍如何使用 AOP 实现分布式锁注解。

## 1. 添加依赖

确保在 `pom.xml` 中添加 Redis 和 AOP 的相关依赖：

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

## 2. 创建分布式锁注解

首先，我们需要定义一个自定义注解 `@RedisLock`，用于标记需要加锁的方法。

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RedisLock {
    String key(); // 锁的键
    long timeout() default 10; // 锁的超时时间
}
```

## 3. 实现 AOP 切面

接下来，我们需要创建一个 AOP 切面，用于处理带有 `@RedisLock` 注解的方法。

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class RedisLockAspect {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Around("@annotation(redisLock)")
    public Object around(ProceedingJoinPoint joinPoint, RedisLock redisLock) throws Throwable {
        String key = redisLock.key();
        String value = String.valueOf(System.currentTimeMillis() + 10000); // 锁的过期时间
        long timeout = redisLock.timeout();

        // 尝试获取锁
        Boolean success = redisTemplate.opsForValue().setIfAbsent(key, value, timeout, TimeUnit.SECONDS);
        if (success != null && success) {
            try {
                // 执行目标方法
                return joinPoint.proceed();
            } finally {
                // 释放锁
                String currentValue = redisTemplate.opsForValue().get(key);
                if (value.equals(currentValue)) {
                    redisTemplate.delete(key);
                }
            }
        } else {
            throw new RuntimeException("获取锁失败，任务正在被其他实例执行");
        }
    }
}
```

## 4. 使用分布式锁注解

现在，我们可以在需要加锁的方法上使用 `@RedisLock` 注解。例如：

```java
import org.springframework.stereotype.Service;

@Service
public class MyService {

    @RedisLock(key = "lock:myTask", timeout = 10)
    public void performTask() {
        // 执行需要加锁的业务逻辑
        System.out.println("执行任务...");
        // 模拟任务执行
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 5. 注意事项

1. **锁的过期时间**：在获取锁时设置合理的过期时间，防止因业务逻辑执行时间过长而导致锁失效。
2. **锁的唯一性**：确保锁的键具有唯一性，以避免不同业务逻辑之间的锁冲突。
3. **异常处理**：在释放锁时，确保只有持有锁的客户端才能释放锁，避免误释放。
4. **可重入锁**：如果需要支持可重入锁，可以在 `RedisLockAspect` 中维护锁的计数器。

## 6. 总结

通过使用 AOP 和自定义注解，我们可以轻松实现分布式锁的功能，简化锁的使用。合理的锁机制可以提高系统的稳定性和数据一致性。

## 参考资料

- [Redis Documentation](https://redis.io/documentation)
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
- [Spring AOP Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop)

---

希望这篇文章能帮助您更好地理解和实现使用 AOP 的分布式锁注解。如果您有任何问题，欢迎在评论区讨论！
--- 