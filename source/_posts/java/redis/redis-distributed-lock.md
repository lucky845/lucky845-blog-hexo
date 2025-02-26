---
title: 【Spring Boot】使用 Redis 实现分布式锁
tags:
  - Java
  - Spring Boot
  - Redis
  - 分布式锁
categories:
  - Java
  - Spring
abbrlink: b55fa581
date: 2025-02-25 14:00:00
---

## 问题背景

在分布式系统中，多个服务实例可能会同时访问共享资源，这可能导致数据不一致或资源竞争的问题。为了解决这个问题，我们可以使用分布式锁来确保同一时间只有一个实例可以访问共享资源。Redis 是一个高性能的内存数据库，常被用作实现分布式锁的工具。本文将介绍如何使用 Redis 实现分布式锁。

## 1. 添加依赖

首先，确保在 `pom.xml` 中添加 Redis 的相关依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

## 2. Redis 分布式锁的基本原理

Redis 分布式锁的基本原理是使用 Redis 的 `SETNX` 命令（SET if Not eXists）来实现锁的获取。具体步骤如下：

1. 客户端尝试在 Redis 中设置一个锁的键（如 `lock:key`），并设置一个过期时间。
2. 如果设置成功，表示获取锁成功；如果设置失败，表示锁已被其他客户端持有。
3. 客户端在完成操作后，删除锁的键，释放锁。

## 3. 实现分布式锁

### 3.1 创建 RedisLock 类

我们可以创建一个 `RedisLock` 类来封装分布式锁的逻辑：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class RedisLock {

    @Autowired
    private StringRedisTemplate redisTemplate;

    // 获取锁
    public boolean tryLock(String key, String value, long timeout) {
        Boolean success = redisTemplate.opsForValue().setIfAbsent(key, value, timeout, TimeUnit.SECONDS);
        return success != null && success;
    }

    // 释放锁
    public void unlock(String key, String value) {
        String currentValue = redisTemplate.opsForValue().get(key);
        if (value.equals(currentValue)) {
            redisTemplate.delete(key);
        }
    }
}
```

### 3.2 使用分布式锁

在需要使用分布式锁的业务逻辑中，我们可以使用 `RedisLock` 类来获取和释放锁。例如：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class MyService {

    @Autowired
    private RedisLock redisLock;

    public void performTask() {
        String lockKey = "lock:myTask";
        String lockValue = String.valueOf(System.currentTimeMillis() + 10000); // 锁的过期时间
        long timeout = 10; // 锁的超时时间

        // 尝试获取锁
        if (redisLock.tryLock(lockKey, lockValue, timeout)) {
            try {
                // 执行需要加锁的业务逻辑
                System.out.println("执行任务...");
                // 模拟任务执行
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                // 释放锁
                redisLock.unlock(lockKey, lockValue);
            }
        } else {
            System.out.println("获取锁失败，任务正在被其他实例执行");
        }
    }
}
```

## 4. 注意事项

1. **锁的过期时间**：在获取锁时设置合理的过期时间，防止因业务逻辑执行时间过长而导致锁失效。
2. **锁的唯一性**：确保锁的键具有唯一性，以避免不同业务逻辑之间的锁冲突。
3. **异常处理**：在释放锁时，确保只有持有锁的客户端才能释放锁，避免误释放。
4. **可重入锁**：如果需要支持可重入锁，可以在 `RedisLock` 中维护锁的计数器。

## 5. 总结

通过使用 Redis，我们可以轻松实现分布式锁，确保在分布式系统中对共享资源的安全访问。合理的锁机制可以提高系统的稳定性和数据一致性。

## 参考资料

- [Redis Documentation](https://redis.io/documentation)
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)

---

希望这篇文章能帮助您更好地理解和实现 Redis 分布式锁。如果您有任何问题，欢迎在评论区讨论！
--- 