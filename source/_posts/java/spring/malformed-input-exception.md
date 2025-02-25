---
title: 【Spring Boot】如何处理 YAML 或 Properties 的解析异常 MalformedInputException
tags:
  - Java
  - Spring Boot
  - 配置解析
  - 异常处理
categories:
  - Java
  - Spring
abbrlink: b55fa580
date: 2025-02-25 11:00:00
---

## 问题背景

在 Spring Boot 应用中，配置文件通常使用 YAML 或 Properties 格式来定义应用的各种参数。然而，在解析这些配置文件时，可能会遇到 `MalformedInputException` 异常。这种异常通常是由于文件编码不正确或文件内容格式不符合预期导致的。本文将介绍如何识别和解决 YAML 或 Properties 的解析异常 `MalformedInputException`。

## 1. 异常原因

### 1.1 文件编码问题

`MalformedInputException` 通常是由于文件的编码格式与 Spring Boot 期望的编码格式不匹配。Spring Boot 默认使用 UTF-8 编码来解析配置文件。如果配置文件使用了其他编码（如 ISO-8859-1），则可能导致解析失败。

### 1.2 格式错误

YAML 和 Properties 文件的格式要求严格，任何格式错误（如缺少冒号、缩进不正确等）都可能导致解析异常。

## 2. 解决方案

### 2.1 确保文件编码为 UTF-8

确保您的 YAML 或 Properties 文件使用 UTF-8 编码。可以使用文本编辑器（如 VSCode、Notepad++）检查和更改文件编码。

#### 在 VSCode 中更改编码

1. 打开文件。
2. 点击右下角的编码格式（如 "UTF-8"）。
3. 选择 "Reopen with Encoding" 并选择 "UTF-8"。

#### 在 Notepad++ 中更改编码

1. 打开文件。
2. 点击菜单 "编码"。
3. 选择 "转换为 UTF-8 编码"。

### 2.2 检查文件格式

仔细检查 YAML 或 Properties 文件的格式，确保符合规范。以下是一些常见的格式问题：

#### YAML 格式示例

```yaml
# 正确的 YAML 格式
server:
  port: 8080
  address: 127.0.0.1

# 错误的 YAML 格式（缺少冒号）
server
  port 8080
```

#### Properties 格式示例

```properties
# 正确的 Properties 格式
server.port=8080
server.address=127.0.0.1

# 错误的 Properties 格式（缺少等号）
server.port 8080
```

### 2.3 使用 Spring Boot 的配置验证

Spring Boot 提供了配置验证的功能，可以在启动时检查配置文件的有效性。可以通过添加 `spring-boot-starter-validation` 依赖来启用此功能。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

然后在配置类中使用 `@Validated` 注解进行验证。

### 2.4 捕获异常并提供友好的错误信息

在应用启动时，可以捕获 `MalformedInputException` 异常，并提供友好的错误信息，帮助开发者快速定位问题。

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.core.io.support.ResourcePropertySource;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        try {
            ApplicationContext context = SpringApplication.run(MyApplication.class, args);
        } catch (MalformedInputException e) {
            System.err.println("配置文件解析错误: " + e.getMessage());
            // 其他处理逻辑
        }
    }
}
```

## 3. 最佳实践建议

1. **使用统一的编码格式**：确保所有配置文件使用 UTF-8 编码，避免编码不一致导致的问题。

2. **定期审查配置文件**：定期检查和更新配置文件，确保格式正确且符合规范。

3. **使用 IDE 的语法检查**：使用 IDE 的语法检查功能，及时发现配置文件中的格式错误。

4. **编写单元测试**：为配置文件编写单元测试，确保在修改配置时不会引入新的问题。

## 总结

通过确保文件编码为 UTF-8、检查文件格式、使用 Spring Boot 的配置验证以及捕获异常，我们可以有效地处理 YAML 或 Properties 的解析异常 `MalformedInputException`。合理的配置管理和异常处理可以提高应用程序的稳定性和可维护性。

## 参考资料

- [Spring Boot - External Configuration](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-external-config)
- [YAML Syntax](https://yaml.org/spec/1.2/spec.html)

---

希望这篇文章能帮助您更好地理解和解决 Spring Boot 中的配置解析异常。如果您有任何问题，欢迎在评论区讨论！
--- 
 