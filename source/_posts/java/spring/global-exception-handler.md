---
title: 【Spring Boot】如何实现全局统一异常处理
tags:
  - Java
  - Spring Boot
  - 异常处理
  - 全局处理
categories:
  - Java
  - Spring
abbrlink: b55fa568
date: 2024-02-24 23:00:00
---

## 问题背景

在实际开发中，异常处理是一个非常重要的环节。良好的异常处理机制不仅能够提高系统的健壮性，还能为前端提供清晰的错误信息。Spring Boot 提供了 `@ControllerAdvice` 和 `@ExceptionHandler` 注解来实现全局异常处理。

## 实现方案

### 1. 定义统一响应结构

首先，我们需要定义一个统一的响应结构：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Result<T> {
    private Integer code;
    private String message;
    private T data;

    public static <T> Result<T> success(T data) {
        return new Result<>(200, "success", data);
    }

    public static <T> Result<T> error(Integer code, String message) {
        return new Result<>(code, message, null);
    }
}
```

### 2. 自定义业务异常

定义业务异常类，用于处理业务逻辑异常：

```java
@Getter
public class BusinessException extends RuntimeException {
    private final Integer code;
    private final String message;

    public BusinessException(String message) {
        this(500, message);
    }

    public BusinessException(Integer code, String message) {
        super(message);
        this.code = code;
        this.message = message;
    }
}
```

### 3. 实现全局异常处理器

创建全局异常处理类，处理各种类型的异常：

```java
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * 处理所有未知异常
     */
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error(500, "系统繁忙，请稍后重试");
    }

    /**
     * 处理业务异常
     */
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        log.warn("业务异常: {}", e.getMessage());
        return Result.error(e.getCode(), e.getMessage());
    }

    /**
     * 处理请求方法不支持异常
     */
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public Result<Void> handleHttpRequestMethodNotSupportedException(HttpRequestMethodNotSupportedException e) {
        log.warn("请求方法不支持: {}", e.getMessage());
        return Result.error(405, "请求方法不支持");
    }

    /**
     * 处理HTTP客户端异常
     */
    @ExceptionHandler(HttpClientErrorException.class)
    public Result<Void> handleHttpClientErrorException(HttpClientErrorException e) {
        log.error("HTTP客户端异常", e);
        return Result.error(e.getRawStatusCode(), e.getStatusText());
    }

    /**
     * 处理Feign调用异常
     */
    @ExceptionHandler(FeignException.class)
    public Result<Void> handleFeignException(FeignException e) {
        log.error("Feign调用异常", e);
        return Result.error(e.status(), "服务调用失败");
    }

    /**
     * 处理参数校验异常
     */
    @ExceptionHandler(ConstraintViolationException.class)
    public Result<Void> handleConstraintViolationException(ConstraintViolationException e) {
        log.warn("参数校验异常", e);
        return Result.error(400, e.getConstraintViolations().stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.joining("; ")));
    }

    /**
     * 处理数据验证异常
     */
    @ExceptionHandler(ValidationException.class)
    public Result<Void> handleValidationException(ValidationException e) {
        log.warn("数据验证异常", e);
        return Result.error(400, e.getMessage());
    }
}
```

### 4. 异常处理组件的特性

一个优秀的异常处理组件应具备以下特性：

1. **统一性**
   - 提供统一的异常处理方式
   - 统一的响应格式
   - 统一的错误码规范

2. **可扩展性**
   - 易于添加新的异常处理类型
   - 支持自定义异常处理逻辑

3. **安全性**
   - 避免敏感信息泄露
   - 生产环境中隐藏具体错误堆栈

4. **可追踪性**
   - 完善的日志记录
   - 唯一的错误追踪标识

5. **友好性**
   - 提供清晰的错误提示
   - 支持国际化

### 5. 使用示例

```java
@RestController
@RequestMapping("/api")
public class UserController {

    @GetMapping("/user/{id}")
    public Result<User> getUser(@PathVariable Long id) {
        if (id <= 0) {
            throw new BusinessException("用户ID不合法");
        }
        // 业务逻辑
        return Result.success(user);
    }

    @PostMapping("/user")
    public Result<User> createUser(@Validated @RequestBody UserDTO userDTO) {
        // 业务逻辑
        return Result.success(user);
    }
}
```

## 最佳实践建议

1. **错误码规范**
   - 建立统一的错误码体系
   - 不同类型的错误使用不同的错误码范围

2. **日志记录**
   - 区分不同级别的日志
   - 记录关键信息和上下文

3. **异常粒度**
   - 合理划分异常类型
   - 避免过细或过粗的异常处理

4. **性能考虑**
   - 避免在异常处理中进行重量级操作
   - 合理使用日志级别

## 常见问题

1. **如何处理嵌套异常？**
   ```java
   @ExceptionHandler(Exception.class)
   public Result<Void> handleException(Exception e) {
       Throwable root = getRootCause(e);
       log.error("系统异常", root);
       return Result.error(500, root.getMessage());
   }
   ```

2. **如何实现错误信息国际化？**
   ```java
   @Autowired
   private MessageSource messageSource;

   public String getLocalizedMessage(String code, Object[] args) {
       return messageSource.getMessage(code, args, LocaleContextHolder.getLocale());
   }
   ```

3. **如何处理异步方法的异常？**
   ```java
   @AsyncConfigurer
   public class AsyncConfig implements AsyncConfigurer {
       @Override
       public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
           return new SimpleAsyncUncaughtExceptionHandler();
       }
   }
   ```

## 总结

通过实现全局统一异常处理，我们可以：
1. 统一处理各类异常
2. 提供友好的错误提示
3. 简化异常处理流程
4. 提高代码的可维护性

## 参考资料

- [Spring Boot Exception Handling](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-error-handling)
- [Spring @ControllerAdvice](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-controller-advice)

---

希望这篇文章能帮助您更好地理解和实现全局统一异常处理。如果您有任何问题，欢迎在评论区讨论！
--- 
 