---
title: 【Spring】如何实现动态数据源切换
tags:
  - Java
  - Spring
  - 数据库
  - 数据源
  - 动态数据源
categories:
  - Java
  - Spring
abbrlink: b55fa560
date: 2024-03-21 16:30:00
---

## 问题背景

在[上一篇文章](/archives/b55fa55f.html)中，我们讨论了如何在Spring中配置多个数据源。但在某些场景下，我们可能需要在运行时动态切换数据源，比如：

1. 读写分离场景
2. 多租户系统
3. 分库分表
4. 数据源故障切换

本文将介绍如何实现动态数据源切换。

## 实现方案

### 1. 创建动态数据源类

首先，我们需要继承 `AbstractRoutingDataSource` 类来实现动态数据源：

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.getDataSourceType();
    }
}
```

### 2. 创建数据源上下文持有者

```java
public class DataSourceContextHolder {
    
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();
    
    public static void setDataSourceType(String dataSourceType) {
        contextHolder.set(dataSourceType);
    }
    
    public static String getDataSourceType() {
        return contextHolder.get();
    }
    
    public static void clearDataSourceType() {
        contextHolder.remove();
    }
}
```

### 3. 创建数据源切换注解

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DataSource {
    String value() default "primary";
}
```

### 4. 实现AOP切面

```java
@Aspect
@Component
@Order(1) // 保证该AOP在@Transactional之前执行
public class DataSourceAspect {
    
    @Pointcut("@annotation(com.example.annotation.DataSource)")
    public void dataSourcePointCut() {}
    
    @Around("dataSourcePointCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        MethodSignature signature = (MethodSignature) point.getSignature();
        Method method = signature.getMethod();
        
        DataSource ds = method.getAnnotation(DataSource.class);
        if (ds == null) {
            ds = point.getTarget().getClass().getAnnotation(DataSource.class);
        }
        
        if (ds != null) {
            DataSourceContextHolder.setDataSourceType(ds.value());
        }
        
        try {
            return point.proceed();
        } finally {
            DataSourceContextHolder.clearDataSourceType();
        }
    }
}
```

### 5. 配置动态数据源

```java
@Configuration
public class DynamicDataSourceConfig {
    
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @Primary
    public DynamicDataSource dataSource(
            DataSource primaryDataSource,
            DataSource secondaryDataSource) {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("primary", primaryDataSource);
        targetDataSources.put("secondary", secondaryDataSource);
        
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        dynamicDataSource.setDefaultTargetDataSource(primaryDataSource);
        dynamicDataSource.setTargetDataSources(targetDataSources);
        
        return dynamicDataSource;
    }
}
```

## 使用示例

### 1. 在Service层使用

```java
@Service
public class UserService {
    
    @Autowired
    private UserMapper userMapper;
    
    @DataSource("primary")
    public User getPrimaryUser(Long id) {
        return userMapper.selectById(id);
    }
    
    @DataSource("secondary")
    public User getSecondaryUser(Long id) {
        return userMapper.selectById(id);
    }
    
    // 在类上使用注解，对所有方法生效
    @DataSource("primary")
    @Service
    public class OrderService {
        // 所有方法默认使用primary数据源
    }
}
```

### 2. 实现读写分离

```java
@Aspect
@Component
public class ReadWriteDataSourceAspect {
    
    @Pointcut("execution(* com.example.service..*.select*(..))" +
            "|| execution(* com.example.service..*.get*(..))" +
            "|| execution(* com.example.service..*.find*(..))")
    public void readPointcut() {}
    
    @Pointcut("execution(* com.example.service..*.insert*(..))" +
            "|| execution(* com.example.service..*.add*(..))" +
            "|| execution(* com.example.service..*.update*(..))" +
            "|| execution(* com.example.service..*.delete*(..))")
    public void writePointcut() {}
    
    @Before("readPointcut()")
    public void setReadDataSource() {
        DataSourceContextHolder.setDataSourceType("read");
    }
    
    @Before("writePointcut()")
    public void setWriteDataSource() {
        DataSourceContextHolder.setDataSourceType("write");
    }
    
    @After("readPointcut() || writePointcut()")
    public void clearDataSource() {
        DataSourceContextHolder.clearDataSourceType();
    }
}
```

## 最佳实践建议

1. **合理使用注解**：优先使用注解方式切换数据源，使代码更清晰。

2. **注意事务**：确保数据源切换在事务开启之前完成。

3. **异常处理**：在切换数据源时做好异常处理，确保 ThreadLocal 能够正确清理。

4. **性能考虑**：
   - 避免频繁切换数据源
   - 使用数据源连接池
   - 合理设置连接池参数

## 常见问题

1. **事务失效问题**
   ```java
   // 错误示例
   @Transactional
   @DataSource("secondary")
   public void wrongMethod() {} // 事务可能失效
   
   // 正确示例
   @DataSource("secondary")
   @Transactional
   public void correctMethod() {} // 确保数据源切换在事务开启前
   ```

2. **线程安全问题**
   - 使用 ThreadLocal 确保线程安全
   - 注意在使用完后清理 ThreadLocal

3. **数据源切换失效**
   - 检查 AOP 切面优先级
   - 确保在事务开启前切换数据源

## 总结

动态数据源为我们提供了更灵活的数据源管理方案，主要应用场景包括：

1. 读写分离
2. 多租户系统
3. 分库分表
4. 故障转移

通过合理使用动态数据源，我们可以实现更灵活的数据库访问策略。

## 参考资料

- [Spring AbstractRoutingDataSource Documentation](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/lookup/AbstractRoutingDataSource.html)
- [Spring AOP Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop)

---

希望这篇文章能帮助您更好地理解和实现动态数据源。如果您有任何问题，欢迎在评论区讨论！
--- 
 