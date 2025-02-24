---
title: 【Spring Boot】如何优雅地停止服务
tags:
  - Java
  - Spring Boot
  - 服务停止
  - 优雅停机
categories:
  - Java
  - Spring
abbrlink: b55fa579
date: 2024-02-25 10:00:00
---

## 问题背景

在生产环境中，优雅地停止服务是确保系统稳定性和数据一致性的关键。Spring Boot 提供了多种方式来实现优雅停机，确保在关闭服务时能够完成正在进行的请求、释放资源并进行必要的清理。本文将介绍如何在 Spring Boot 中优雅地停止服务。

## 1. 使用 `@PreDestroy` 注解

在 Spring 中，可以使用 `@PreDestroy` 注解标记一个方法，该方法会在 Bean 被销毁之前执行。可以在这个方法中添加清理逻辑。

### 示例代码

```java
import org.springframework.stereotype.Component;

import javax.annotation.PreDestroy;

@Component
public class MyService {

    // 其他业务逻辑

    @PreDestroy
    public void cleanup() {
        System.out.println("服务正在停止，执行清理操作...");
        // 释放资源、保存状态等
    }
}
```

在这个例子中，当 Spring Boot 应用关闭时，`cleanup` 方法会被调用，执行必要的清理操作。

## 2. 使用 `ApplicationListener` 监听 `ContextClosedEvent`

可以通过实现 `ApplicationListener` 接口来监听 Spring 应用上下文关闭事件，从而执行自定义的停机逻辑。

### 示例代码

```java
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.stereotype.Component;

@Component
public class ShutdownListener implements ApplicationListener<ContextClosedEvent> {

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        System.out.println("应用上下文正在关闭，执行停机逻辑...");
        // 执行清理操作
    }
}
```

在这个例子中，当 Spring 应用上下文关闭时，`onApplicationEvent` 方法会被调用，执行自定义的停机逻辑。

## 3. 配置优雅停机

Spring Boot 2.3 及以上版本提供了内置的优雅停机支持。可以通过配置文件来启用优雅停机。

### 配置示例

在 `application.yml` 中添加以下配置：

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s # 设置每个停机阶段的超时时间
```

### 处理请求的超时

可以通过设置 `server.shutdown` 属性来控制服务的停机行为：

```yaml
server:
  shutdown: graceful
```

当设置为 `graceful` 时，Spring Boot 会在接收到关闭请求后，等待正在处理的请求完成。

## 4. 自定义停机逻辑

如果需要更复杂的停机逻辑，可以使用 `SmartLifecycle` 接口。实现该接口的 Bean 可以控制其启动和停止的顺序。

### 示例代码

```java
import org.springframework.context.SmartLifecycle;
import org.springframework.stereotype.Component;

@Component
public class MyLifecycleComponent implements SmartLifecycle {

    private boolean running = false;

    @Override
    public void start() {
        running = true;
        System.out.println("服务启动...");
    }

    @Override
    public void stop() {
        System.out.println("服务正在停止...");
        // 执行清理操作
        running = false;
    }

    @Override
    public boolean isRunning() {
        return running;
    }

    @Override
    public int getPhase() {
        return 0; // 返回停机的优先级
    }

    @Override
    public boolean isAutoStartup() {
        return true;
    }

    @Override
    public void stop(Runnable callback) {
        stop();
        callback.run(); // 停止后执行回调
    }
}
```

在这个例子中，`MyLifecycleComponent` 实现了 `SmartLifecycle` 接口，提供了自定义的启动和停止逻辑。

## 最佳实践建议

1. **确保清理资源**：在停机时，确保释放所有占用的资源，如数据库连接、线程池等。

2. **记录停机日志**：在停机过程中记录日志，以便后续排查问题。

3. **测试停机逻辑**：在开发和测试环境中充分测试停机逻辑，确保在生产环境中能够正常工作。

4. **监控停机状态**：使用监控工具监控服务的停机状态，确保服务能够优雅地停止。

## 总结

通过使用 `@PreDestroy` 注解、监听 `ContextClosedEvent`、配置优雅停机和自定义停机逻辑，我们可以在 Spring Boot 中优雅地停止服务，确保系统的稳定性和数据的一致性。

## 参考资料

- [Spring Boot - Graceful Shutdown](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-graceful-shutdown)
- [Spring Framework - Lifecycle](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-functionality-lifecycle)

---

希望这篇文章能帮助您更好地理解和实现 Spring Boot 中的优雅停机。如果您有任何问题，欢迎在评论区讨论！
--- 
 