---
title: 【Spring】如何同时启用多个数据源
tags:
  - Java
  - Spring
  - 数据库
  - 数据源
categories:
  - Java
  - Spring
abbrlink: b55fa55f
date: 2024-03-21 16:00:00
---

## 问题背景

在实际应用中，可能需要连接多个数据库，例如一个用于主数据存储，另一个用于日志或分析数据。Spring 提供了灵活的方式来配置多个数据源，以便在同一个应用中使用。

## 解决方案

### 1. 添加依赖

首先，确保在 `pom.xml` 中添加了 Spring Data JPA 和数据库驱动的依赖。例如，如果使用 MySQL，可以添加如下依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

### 2. 配置数据源

在 `application.yml` 或 `application.properties` 中配置多个数据源。例如：

```yaml
spring:
  datasource:
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

### 3. 创建数据源配置类

为每个数据源创建配置类，并使用 `@Primary` 注解标记主数据源。

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.orm.jpa.HibernatePropertiesCustomizer;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.orm.jpa.EntityManagerFactoryBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.transaction.PlatformTransactionManager;

import javax.persistence.EntityManagerFactory;

@Configuration
@EnableJpaRepositories(
        basePackages = "com.example.repository.primary",
        entityManagerFactoryRef = "primaryEntityManagerFactory",
        transactionManagerRef = "primaryTransactionManager"
)
public class PrimaryDataSourceConfig {

    @Primary
    @Bean(name = "primaryDataSource")
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("primaryDataSource") DataSource dataSource) {
        return builder
                .dataSource(dataSource)
                .packages("com.example.model.primary")
                .persistenceUnit("primary")
                .build();
    }

    @Primary
    @Bean(name = "primaryTransactionManager")
    public PlatformTransactionManager primaryTransactionManager(
            @Qualifier("primaryEntityManagerFactory") EntityManagerFactory primaryEntityManagerFactory) {
        return new JpaTransactionManager(primaryEntityManagerFactory);
    }
}
```

### 4. 创建第二个数据源配置类

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.orm.jpa.HibernatePropertiesCustomizer;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.orm.jpa.EntityManagerFactoryBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.transaction.PlatformTransactionManager;

import javax.persistence.EntityManagerFactory;

@Configuration
@EnableJpaRepositories(
        basePackages = "com.example.repository.secondary",
        entityManagerFactoryRef = "secondaryEntityManagerFactory",
        transactionManagerRef = "secondaryTransactionManager"
)
public class SecondaryDataSourceConfig {

    @Bean(name = "secondaryDataSource")
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "secondaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean secondaryEntityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("secondaryDataSource") DataSource dataSource) {
        return builder
                .dataSource(dataSource)
                .packages("com.example.model.secondary")
                .persistenceUnit("secondary")
                .build();
    }

    @Bean(name = "secondaryTransactionManager")
    public PlatformTransactionManager secondaryTransactionManager(
            @Qualifier("secondaryEntityManagerFactory") EntityManagerFactory secondaryEntityManagerFactory) {
        return new JpaTransactionManager(secondaryEntityManagerFactory);
    }
}
```

### 5. 创建实体和仓库

为每个数据源创建相应的实体类和仓库接口。例如：

```java
// Primary Data Source Entity
@Entity
public class PrimaryEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    // getters and setters
}

// Secondary Data Source Entity
@Entity
public class SecondaryEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String description;
    // getters and setters
}
```

```java
// Primary Repository
public interface PrimaryRepository extends JpaRepository<PrimaryEntity, Long> {
}

// Secondary Repository
public interface SecondaryRepository extends JpaRepository<SecondaryEntity, Long> {
}
```

## 实际应用示例

### 1. 使用服务层

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class MyService {

    @Autowired
    private PrimaryRepository primaryRepository;

    @Autowired
    private SecondaryRepository secondaryRepository;

    public void saveData() {
        PrimaryEntity primaryEntity = new PrimaryEntity();
        primaryEntity.setName("Primary Data");
        primaryRepository.save(primaryEntity);

        SecondaryEntity secondaryEntity = new SecondaryEntity();
        secondaryEntity.setDescription("Secondary Data");
        secondaryRepository.save(secondaryEntity);
    }
}
```

### 2. 测试验证

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public class MyServiceTest {

    @Autowired
    private MyService myService;

    @Test
    public void testSaveData() {
        myService.saveData();
        // 验证数据是否保存成功
    }
}
```

## 最佳实践建议

1. **使用配置文件管理数据源**：将数据源的配置放在 `application.yml` 或 `application.properties` 中，便于管理和修改。
2. **分离数据源逻辑**：将每个数据源的逻辑分开，避免混淆。
3. **使用事务管理**：确保在操作多个数据源时，使用适当的事务管理策略。

## 常见问题

1. **如何处理事务？**
   - 可以使用 Spring 的事务管理器，确保在多个数据源之间的操作是原子的。

2. **如何动态切换数据源？**
   - 可以使用 AOP 或者自定义注解来动态切换数据源。

3. **如何处理数据源连接池？**
   - 可以使用 HikariCP 或其他连接池来管理数据源的连接。

## 总结

通过以上步骤，我们可以在 Spring 中同时启用多个数据源，增强了应用的灵活性和可扩展性。无论是通过配置文件还是代码配置，都能有效地管理多个数据源。

## 参考资料

- [Spring Data JPA Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#reference)
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-sql) 
---

希望这篇文章能帮助您更好地解决Spring配置多数据源的问题。如果您有任何问题，欢迎在评论区讨论！
---