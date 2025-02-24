---
title: 【Spring】解决动态数据源在多线程环境下的传递问题
tags:
  - Java
  - Spring
  - 数据库
  - 数据源
  - 动态数据源
  - 线程池
categories:
  - Java
  - Spring
abbrlink: b55fa562
date: 2024-03-21 17:30:00
---

## 问题背景

在[上一篇文章](/archives/b55fa561.html)中，我们讨论了如何使用 `dynamic-datasource` 框架实现动态数据源切换。但在多线程环境下，特别是使用 `CompletableFuture` 时，会遇到一个常见问题：子线程无法获取父线程的数据源信息。

这是因为 `dynamic-datasource` 框架使用 `ThreadLocal` 来存储当前线程的数据源信息，而子线程无法自动继承父线程的 `ThreadLocal` 值。

解决这个问题有两种主要方案：
1. 自定义线程池工厂（如下文所述）
2. 使用阿里的 TransmittableThreadLocal 库

## 解决方案

### 方案一：使用 TransmittableThreadLocal

首先，添加 TransmittableThreadLocal 依赖：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.14.2</version>
</dependency>
```

然后，修改数据源上下文持有者：

```java
import com.alibaba.ttl.TransmittableThreadLocal;

public class DataSourceContextHolder {
    private static final TransmittableThreadLocal<String> CONTEXT_HOLDER = new TransmittableThreadLocal<>();
    
    public static void setDataSourceType(String dataSourceType) {
        CONTEXT_HOLDER.set(dataSourceType);
    }
    
    public static String getDataSourceType() {
        return CONTEXT_HOLDER.get();
    }
    
    public static void clearDataSourceType() {
        CONTEXT_HOLDER.remove();
    }
}
```

使用 TtlExecutors 装饰线程池：

```java
import com.alibaba.ttl.threadpool.TtlExecutors;

@Configuration
public class ThreadPoolConfig {
    
    @Bean
    public ExecutorService dataSourceAwareExecutor() {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            10,
            20,
            60L,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
        
        // 使用TTL装饰线程池
        return TtlExecutors.getTtlExecutorService(executor);
    }
}
```

使用示例：

```java
@Service
public class UserService {
    
    @Autowired
    private ExecutorService dataSourceAwareExecutor;
    
    @DS("primary")
    public void asyncOperation() {
        // 使用装饰后的线程池，TransmittableThreadLocal 值会自动传递
        CompletableFuture.supplyAsync(() -> {
            return userMapper.selectById(1L);
        }, dataSourceAwareExecutor);
    }
}
```

### 方案二：自定义线程池工厂

首先，创建一个自定义的线程工厂，用于传递数据源信息：

```java
import com.baomidou.dynamic.datasource.toolkit.DynamicDataSourceContextHolder;
import java.util.concurrent.ThreadFactory;

public class DataSourceThreadFactory implements ThreadFactory {
    
    private final ThreadFactory delegate;
    
    public DataSourceThreadFactory(ThreadFactory delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public Thread newThread(Runnable r) {
        // 获取当前线程的数据源
        String dataSource = DynamicDataSourceContextHolder.peek();
        
        // 包装Runnable，确保子线程可以获取父线程的数据源
        Runnable wrapper = () -> {
            try {
                // 设置子线程的数据源
                if (dataSource != null) {
                    DynamicDataSourceContextHolder.push(dataSource);
                }
                // 执行原始任务
                r.run();
            } finally {
                // 清理数据源
                DynamicDataSourceContextHolder.poll();
            }
        };
        
        return delegate.newThread(wrapper);
    }
}
```

## 最佳实践建议

1. **统一使用自定义线程池**：所有需要数据源传递的异步操作都应该使用自定义的线程池。

2. **正确清理数据源**：确保在任务完成后清理数据源信息，避免内存泄漏。

3. **合理配置线程池参数**：根据实际业务需求配置线程池的核心参数。

4. **异常处理**：在异步操作中添加适当的异常处理机制。

5. **选择合适的方案**：
   - 如果项目中已经使用了阿里的 TTL 库，推荐使用 TransmittableThreadLocal 方案
   - 如果想要更轻量级的解决方案，可以使用自定义线程池工厂方案

## 常见问题

1. **数据源信息丢失**
   ```java
   // 错误示例
   CompletableFuture.supplyAsync(() -> {
       return userMapper.selectById(1L); // 数据源信息丢失
   });
   
   // 正确示例
   CompletableFuture.supplyAsync(() -> {
       return userMapper.selectById(1L);
   }, dataSourceAwareExecutor);
   ```

2. **内存泄漏**
   ```java
   // 错误示例
   DynamicDataSourceContextHolder.push("primary");
   // 忘记清理数据源
   
   // 正确示例
   try {
       DynamicDataSourceContextHolder.push("primary");
       // 业务操作
   } finally {
       DynamicDataSourceContextHolder.poll();
   }
   ```

3. **线程池配置不当**
   - 核心线程数过小可能导致性能问题
   - 队列容量过大可能导致内存问题
   - 需要根据实际业务场景调整参数

## 总结

通过自定义线程池和线程工厂，我们可以优雅地解决动态数据源在多线程环境下的传递问题。这种方案具有以下优点：

1. 对业务代码侵入性小
2. 统一管理数据源传递
3. 避免手动管理数据源切换
4. 提高代码可维护性

## 参考资料

- [CompletableFuture Documentation](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)
- [ThreadPoolExecutor Documentation](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html)
- [dynamic-datasource GitHub](https://github.com/baomidou/dynamic-datasource-spring-boot-starter)
- [TransmittableThreadLocal GitHub](https://github.com/alibaba/transmittable-thread-local)

---

希望这篇文章能帮助您解决动态数据源在多线程环境下的传递问题。如果您有任何问题，欢迎在评论区讨论！
--- 
 