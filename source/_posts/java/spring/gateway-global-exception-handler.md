---
title: 【Spring Cloud Gateway】如何配置全局异常处理
tags:
  - Java
  - Spring Cloud
  - Gateway
  - 异常处理
categories:
  - Java
  - Spring
abbrlink: b55fa569
date: 2025-02-25 00:00:00
---

## 问题背景

在使用 Spring Cloud Gateway 作为 API 网关时，处理请求和响应的异常是非常重要的。良好的异常处理机制可以提高系统的健壮性，并为前端提供清晰的错误信息。与 Spring Boot 中的全局异常处理类似，Spring Cloud Gateway 也提供了全局异常处理的能力。

## 关联内容

在之前的文章中，我们讨论了如何在 Spring Boot 中实现全局统一异常处理，使用 `@ControllerAdvice` 和 `@ExceptionHandler` 注解来处理各种类型的异常。本文将基于这些概念，介绍如何在 Spring Cloud Gateway 中实现类似的功能。

## 实现方案

### 1. 定义统一响应结构

首先，我们需要定义一个统一的响应结构，类似于之前的实现：

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

### 2. 创建全局异常处理器

在 Spring Cloud Gateway 中，我们可以通过实现 `GlobalFilter` 接口来处理全局异常。以下是一个示例：

```java
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class GlobalExceptionHandlerFilter extends AbstractGatewayFilterFactory<GlobalExceptionHandlerFilter.Config> {

    public GlobalExceptionHandlerFilter() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            return chain.filter(exchange).onErrorResume(throwable -> {
                // 处理异常
                return handleException(exchange, throwable);
            });
        };
    }

    private Mono<Void> handleException(ServerWebExchange exchange, Throwable throwable) {
        // 根据异常类型返回不同的响应
        HttpStatus status = HttpStatus.INTERNAL_SERVER_ERROR;
        String message = "系统繁忙，请稍后重试";

        if (throwable instanceof BusinessException) {
            status = HttpStatus.BAD_REQUEST;
            message = throwable.getMessage();
        } else if (throwable instanceof HttpRequestMethodNotSupportedException) {
            status = HttpStatus.METHOD_NOT_ALLOWED;
            message = "请求方法不支持";
        }

        // 构建响应
        exchange.getResponse().setStatusCode(status);
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
        return exchange.getResponse().writeWith(Mono.just(exchange.getResponse().bufferFactory().wrap(
                new ObjectMapper().writeValueAsBytes(Result.error(status.value(), message))
        )));
    }

    public static class Config {
        // 配置类
    }
}
```

### 3. 注册全局过滤器

在 Spring Cloud Gateway 中，您需要将全局过滤器注册到应用程序中。可以在主应用类中使用 `@Bean` 注解进行注册：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

    @Bean
    public GlobalExceptionHandlerFilter globalExceptionHandlerFilter() {
        return new GlobalExceptionHandlerFilter();
    }
}
```

### 4. 使用示例

在 Gateway 中配置路由时，可以直接使用全局异常处理器：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/user/**
          filters:
            - name: GlobalExceptionHandlerFilter
```

## 最佳实践建议

1. **统一错误码规范**
   - 建立统一的错误码体系，确保前后端一致。

2. **日志记录**
   - 记录异常信息，便于后续排查。

3. **性能考虑**
   - 在异常处理过程中，避免进行重量级操作。

4. **友好性**
   - 提供清晰的错误提示，支持国际化。

## 常见问题

1. **如何处理嵌套异常？**
   - 可以在 `handleException` 方法中对异常进行递归处理，获取根异常。

2. **如何实现错误信息国际化？**
   - 可以使用 `MessageSource` 来获取国际化的错误信息。

3. **如何处理异步请求的异常？**
   - 在异步请求中，确保异常处理逻辑能够捕获到异常。

## 总结

通过实现全局统一异常处理，我们可以在 Spring Cloud Gateway 中有效地处理各种异常，提供友好的错误提示，并提高系统的健壮性。

## 参考资料

- [Spring Cloud Gateway Documentation](https://spring.io/projects/spring-cloud-gateway)
- [Spring Boot Exception Handling](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-error-handling)

---

希望这篇文章能帮助您更好地理解和实现 Spring Cloud Gateway 的全局异常处理。如果您有任何问题，欢迎在评论区讨论！
--- 
 