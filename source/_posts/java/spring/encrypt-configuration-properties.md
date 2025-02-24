---
title: 【Spring Boot】如何统一给配置项属性值加密
tags:
  - Java
  - Spring Boot
  - 配置加密
  - 属性安全
categories:
  - Java
  - Spring
abbrlink: b55fa577
date: 2024-02-25 08:00:00
---

## 问题背景

在开发应用程序时，保护敏感信息（如数据库密码、API 密钥等）是非常重要的。Spring Boot 提供了多种方式来加密和解密配置项属性值，以确保这些敏感信息不会被泄露。本文将介绍如何在 Spring Boot 中统一给配置项属性值加密。

## 1. 使用 Spring Cloud Config

Spring Cloud Config 提供了集中式的配置管理功能，可以将配置项存储在 Git、文件系统或其他存储中，并支持加密和解密。以下是使用 Spring Cloud Config 进行配置加密的步骤。

### 1.1 添加依赖

在 `pom.xml` 中添加 Spring Cloud Config 相关依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

### 1.2 配置加密密钥

在 `application.yml` 或 `application.properties` 中配置加密密钥：

```yaml
spring:
  cloud:
    config:
      server:
        encrypt:
          key: your-encryption-key
```

### 1.3 加密配置项

使用 Spring Cloud Config 提供的加密工具加密配置项。例如，使用以下命令加密数据库密码：

```bash
curl -X POST http://localhost:8888/encrypt -d 'your-database-password'
```

返回的加密字符串可以放入配置文件中：

```yaml
database:
  password: '{cipher}encrypted-password'
```

### 1.4 解密配置项

在应用程序中，Spring Cloud Config 会自动解密配置项。只需使用 `@Value` 注解获取配置项即可：

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class DatabaseConfig {

    @Value("${database.password}")
    private String password;

    // ...
}
```

## 2. 使用 Jasypt 加密

如果不使用 Spring Cloud Config，可以使用 Jasypt（Java Simplified Encryption）库来加密和解密配置项。

### 2.1 添加依赖

在 `pom.xml` 中添加 Jasypt 依赖：

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.4</version>
</dependency>
```

### 2.2 配置加密密钥

在 `application.yml` 或 `application.properties` 中配置 Jasypt 密钥：

```yaml
jasypt:
  encryptor:
    password: your-encryption-key
```

### 2.3 加密配置项

使用 Jasypt 提供的命令行工具加密配置项。例如，使用以下命令加密数据库密码：

```bash
java -cp jasypt-1.9.3.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI input="your-database-password" password="your-encryption-key"
```

返回的加密字符串可以放入配置文件中：

```yaml
database:
  password: ENC(encrypted-password)
```

### 2.4 解密配置项

在应用程序中，Jasypt 会自动解密配置项。只需使用 `@Value` 注解获取配置项即可：

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class DatabaseConfig {

    @Value("${database.password}")
    private String password;

    // ...
}
```

## 3. 自定义加密逻辑

如果需要更复杂的加密逻辑，可以实现自定义的 `PropertySource` 或使用 `@ConfigurationProperties` 结合自定义的加密和解密方法。

### 示例代码

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "secure")
public class SecureProperties {

    private String password;

    public String getPassword() {
        return decrypt(password);
    }

    public void setPassword(String password) {
        this.password = password;
    }

    private String decrypt(String encrypted) {
        // 自定义解密逻辑
        return decryptedValue;
    }
}
```

## 最佳实践建议

1. **使用环境变量**：将加密密钥存储在环境变量中，而不是硬编码在配置文件中。

2. **定期更换密钥**：定期更换加密密钥，以提高安全性。

3. **审计和监控**：对敏感信息的访问进行审计和监控，确保没有未授权的访问。

4. **使用安全的加密算法**：选择安全的加密算法和密钥长度，确保数据的安全性。

## 总结

通过使用 Spring Cloud Config 或 Jasypt，我们可以在 Spring Boot 中统一给配置项属性值加密，确保敏感信息的安全性。合理的设计和使用加密机制，可以提高应用程序的安全性和可维护性。

## 参考资料

- [Spring Cloud Config](https://cloud.spring.io/spring-cloud-config/)
- [Jasypt](http://www.jasypt.org/)

---

希望这篇文章能帮助您更好地理解和实现 Spring Boot 中的配置项加密。如果您有任何问题，欢迎在评论区讨论！
--- 
 