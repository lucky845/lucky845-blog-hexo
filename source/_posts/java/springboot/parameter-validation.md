---
title: 【Spring Boot】如何进行 Body、Query、Path Variable 类型的参数校验
tags:
  - Java
  - Spring Boot
  - 参数校验
  - 数据验证
categories:
  - Java
  - Spring Boot
abbrlink: b55fa572
date: 2025-02-25 03:00:00
---

## 问题背景

在开发 RESTful API 时，参数校验是确保数据有效性和安全性的重要环节。Spring Boot 提供了强大的参数校验功能，可以对请求中的 Body、Query 和 Path Variable 类型的参数进行校验。本文将介绍如何在 Spring Boot 中实现这些参数的校验。

## 依赖配置

首先，确保在 `pom.xml` 中添加了 `spring-boot-starter-validation` 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

## 1. Body 参数校验

对于请求体中的参数校验，我们可以使用 `@Valid` 注解结合 Java Bean Validation API（如 Hibernate Validator）来实现。

### 示例代码

```java
import javax.validation.constraints.Email;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Size;

public class UserDTO {
    
    @NotBlank(message = "用户名不能为空")
    private String username;

    @Email(message = "邮箱格式不正确")
    private String email;

    @Size(min = 6, message = "密码长度不能少于6位")
    private String password;

    // getters and setters
}
```

### 控制器示例

```java
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<String> createUser(@Valid @RequestBody UserDTO userDTO) {
        // 处理用户创建逻辑
        return ResponseEntity.ok("用户创建成功");
    }
}
```

## 2. Query 参数校验

对于查询参数的校验，可以直接在控制器方法的参数上使用校验注解。

### 示例代码

```java
import javax.validation.constraints.Min;
import javax.validation.constraints.NotNull;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping
    public ResponseEntity<List<Product>> getProducts(
            @RequestParam @NotNull(message = "页码不能为空") @Min(value = 1, message = "页码必须大于0") Integer page,
            @RequestParam @NotNull(message = "每页数量不能为空") @Min(value = 1, message = "每页数量必须大于0") Integer size) {
        // 处理获取产品逻辑
        return ResponseEntity.ok(productService.getProducts(page, size));
    }
}
```

## 3. Path Variable 校验

对于路径变量的校验，方法与查询参数类似，可以在路径变量上使用校验注解。

### 示例代码

```java
import javax.validation.constraints.Min;
import javax.validation.constraints.NotNull;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(
            @PathVariable @NotNull(message = "订单ID不能为空") @Min(value = 1, message = "订单ID必须大于0") Long id) {
        // 处理获取订单逻辑
        return ResponseEntity.ok(orderService.getOrderById(id));
    }
}
```

## 4. 全局异常处理

为了处理校验失败时的异常，我们可以使用全局异常处理器。

### 示例代码

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

import java.util.HashMap;
import java.util.Map;

@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> 
            errors.put(error.getField(), error.getDefaultMessage())
        );
        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }
}
```

## 最佳实践建议

1. **使用注解进行校验**：利用 Java Bean Validation 提供的注解进行参数校验，保持代码简洁。

2. **全局异常处理**：使用 `@ControllerAdvice` 进行全局异常处理，集中管理异常响应。

3. **清晰的错误信息**：提供清晰的错误信息，帮助前端开发人员快速定位问题。

4. **文档记录**：在 API 文档中记录参数校验规则，确保前后端一致。

## 常见问题

1. **如何自定义校验注解？**
   - 可以通过实现 `ConstraintValidator` 接口来自定义校验逻辑。

2. **如何处理复杂的校验逻辑？**
   - 可以使用组合注解或自定义注解来处理复杂的校验需求。

3. **如何进行国际化的错误信息？**
   - 可以使用 `MessageSource` 来实现国际化的错误信息。

## 总结

通过以上方法，我们可以在 Spring Boot 中有效地进行 Body、Query 和 Path Variable 类型的参数校验，确保 API 的数据有效性和安全性。

## 参考资料

- [Spring Boot Validation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-validation)
- [Java Bean Validation](https://beanvalidation.org/)

---

希望这篇文章能帮助您更好地理解和实现 Spring Boot 中的参数校验。如果您有任何问题，欢迎在评论区讨论！
--- 
 