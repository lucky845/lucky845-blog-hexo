---
title: 【Spring】使用动态数据源框架实现动态数据源切换
tags:
  - Java
  - Spring
  - 数据库
  - 数据源
  - 动态数据源
  - 开源框架
categories:
  - Java
  - Spring
abbrlink: b55fa561
date: 2025-02-24 17:00:00
---

## 问题背景

在[上一篇文章](/archives/b55fa560.html)中，我们讨论了如何在Spring中实现动态数据源切换。虽然手动实现动态数据源切换是可行的，但使用开源的第三方库可以大大简化开发过程，提高代码的可维护性和可读性。

本文将介绍如何使用开源的 `dynamic-datasource` 框架来实现动态数据源切换。

## 1. 添加依赖

首先，在 `pom.xml` 中添加 `dynamic-datasource` 的依赖：

```xml
<dependency>
    <groupId>com.github.dynamic-datasource</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.1.0</version> <!-- 请根据最新版本进行调整 -->
</dependency>
```

## 2. 配置数据源

在 `application.yml` 或 `application.properties` 中配置多个数据源。例如：

```yaml
spring:
  datasource:
    dynamic:
      primary:
        url: jdbc:mysql://localhost:3306/primary_db
        username: root
        password: password
        driver-class-name: com.mysql.cj.jdbc.Driver
      secondary:
        url: jdbc:mysql://localhost:3306/secondary_db
        username: root
        password: password
        driver-class-name: com.mysql.cj.jdbc.Driver
```

## 3. 使用注解切换数据源

动态数据源的切换有两种方式:注解方式和硬编码方式。

### 3.1 注解方式切换

```java
import com.baomidou.dynamic.datasource.annotation.DS;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @DS("primary") // 切换到主数据源
    public User getPrimaryUser(Long id) {
        // 从主数据源获取用户
    }

    @DS("secondary") // 切换到次数据源
    public User getSecondaryUser(Long id) {
        // 从次数据源获取用户
    }
}
```

### 3.2 硬编码方式切换

使用 `DynamicDataSourceContextHolder` 类进行手动切换:

```java
import com.baomidou.dynamic.datasource.toolkit.DynamicDataSourceContextHolder;

@Service
public class UserService {

    public User getUser(Long id, String dataSource) {
        // 切换数据源
        DynamicDataSourceContextHolder.push(dataSource);
        try {
            // 执行业务操作
            return userMapper.selectById(id);
        } finally {
            // 清除数据源
            DynamicDataSourceContextHolder.poll();
        }
    }

    public void complexOperation() {
        // 查询主库
        DynamicDataSourceContextHolder.push("primary");
        User primaryUser = userMapper.selectById(1L);
        DynamicDataSourceContextHolder.poll();

        // 查询从库
        DynamicDataSourceContextHolder.push("secondary");
        User secondaryUser = userMapper.selectById(2L);
        DynamicDataSourceContextHolder.poll();

        // 更多复杂操作...
    }
}
```

硬编码方式的优点是更加灵活，可以在方法内部根据不同的业务逻辑动态切换数据源。但需要注意以下几点：

1. 必须手动清理数据源，建议使用 try-finally 块。
2. 嵌套切换时要注意顺序，先进后出。
3. 代码侵入性较强，不如注解方式优雅。

## 4. 读写分离示例

如果需要实现读写分离，可以使用 `@DS` 注解结合 AOP 来实现。例如：

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
        DataSourceContextHolder.setDataSourceType("secondary");
    }

    @Before("writePointcut()")
    public void setWriteDataSource() {
        DataSourceContextHolder.setDataSourceType("primary");
    }

    @After("readPointcut() || writePointcut()")
    public void clearDataSource() {
        DataSourceContextHolder.clearDataSourceType();
    }
}
```

## 5. 使用示例

### 1. 在Service层使用

使用 `@DSTransactional` 注解来确保事务在切换数据源时能够正常工作：

```java
import com.baomidou.dynamic.datasource.annotation.DSTransactional;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @DS("primary")
    @DSTransactional // 使用动态数据源的事务注解
    public User getPrimaryUser(Long id) {
        return userMapper.selectById(id);
    }

    @DS("secondary")
    @DSTransactional // 使用动态数据源的事务注解
    public User getSecondaryUser(Long id) {
        return userMapper.selectById(id);
    }
}
```

### 2. 测试验证

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    public void testGetUsers() {
        User primaryUser = userService.getPrimaryUser(1L);
        User secondaryUser = userService.getSecondaryUser(2L);
        // 验证用户是否正确
    }
}
```

## 最佳实践建议

1. **使用开源框架**：使用 `dynamic-datasource` 框架可以减少手动实现的复杂性，提高开发效率。

2. **合理配置数据源**：在配置文件中集中管理数据源的配置，便于维护。

3. **注意事务管理**：确保在切换数据源时，事务能够正确处理，使用 `@DSTransactional` 注解。

4. **性能考虑**：使用连接池来管理数据源的连接，确保性能。

## 常见问题

1. **如何处理事务？**
   - 使用 `@DSTransactional` 注解确保事务在切换数据源时能够正常工作。

2. **如何动态切换数据源？**
   - 使用 `@DS` 注解可以方便地在方法级别切换数据源。

3. **如何处理数据源连接池？**
   - `dynamic-datasource` 框架支持多种连接池，可以根据需要进行配置。

## 总结

通过使用 `dynamic-datasource` 框架，我们可以轻松实现动态数据源切换，增强了应用的灵活性和可维护性。无论是读写分离还是多租户系统，开源框架都能提供强大的支持。

## 参考资料

- [dynamic-datasource GitHub](https://github.com/baomidou/dynamic-datasource-spring-boot-starter)

---

希望这篇文章能帮助您更好地理解和实现动态数据源。如果您有任何问题，欢迎在评论区讨论！
--- 
 