---
title: 【Spring】解决数据库 Long 类型雪花 ID 前端精度损失的问题
categories:
  - 技术笔记
  - Java
  - Spring
  - Spring Boot
tags:
  - 雪花算法
  - 精度损失
  - JSON序列化
  - FastJson2
abbrlink: 4977917d
date: 2025-02-24 14:30:00
---

# 问题背景

在现代分布式系统中，雪花算法（Snowflake）被广泛用于生成唯一的 ID。这些 ID 通常以 Long 类型存储在数据库中。然而，当这些 ID 被传递到前端时，可能会出现精度损失的问题。本文将探讨这一问题的原因，并提供几种解决方案。

## 问题描述

在 Java 中，雪花 ID 通常是一个 64 位的 Long 类型数字。然而，在 JavaScript 中，数字的精度限制为 53 位（即 `Number.MAX_SAFE_INTEGER`），这意味着当 Long 类型的雪花 ID 超过 53 位时，可能会出现精度损失。

例如，以下雪花 ID 在 JavaScript 中可能会被错误处理：

```java
// Java 中的雪花 ID
long snowflakeId = 1234567890123456789L; // 64 位 Long
```

在 JavaScript 中，这个 ID 可能会被处理为：

```javascript
// JavaScript 中的处理
let id = 1234567890123456789; // 精度损失
console.log(id); // 输出: 1234567890123456700
```

## 解决方案

### 1. 修改 Jackson 的序列化

如果您使用 Jackson 进行 JSON 序列化，可以通过自定义序列化器来处理 Long 类型的雪花 ID。以下是一个示例：

```java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.JsonSerializer;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;

@JsonSerialize(using = SnowflakeIdSerializer.class)
public class YourEntity {
    private Long snowflakeId;

    // getters and setters
}

public class SnowflakeIdSerializer extends JsonSerializer<Long> {
    @Override
    public void serialize(Long value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeString(value.toString()); // 将 Long 转为 String
    }
}
```

### 2. 使用 Fastjson2 替换默认的 Jackson

Fastjson2 是一个高性能的 JSON 处理库，支持更好的数字精度处理。您可以通过以下方式使用 Fastjson2：

```java
import com.alibaba.fastjson2.JSONReader;
import com.alibaba.fastjson2.JSONWriter;
import com.alibaba.fastjson2.support.config.FastJsonConfig;
import com.alibaba.fastjson2.support.spring6.http.converter.FastJsonHttpMessageConverter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;

import java.nio.charset.StandardCharsets;
import java.util.List;

@Slf4j
@AutoConfiguration
public class FastJson2Config {

    @Bean
    public HttpMessageConverter<Object> fastJsonHttpMessageConverter() {
        FastJsonHttpMessageConverter fastConverter = new FastJsonHttpMessageConverter();
        FastJsonConfig config = getFastJsonConfig();
        fastConverter.setFastJsonConfig(config);
        fastConverter.setDefaultCharset(StandardCharsets.UTF_8);
        fastConverter.setSupportedMediaTypes(List.of(MediaType.APPLICATION_JSON));
        return fastConverter;
    }

    private static FastJsonConfig getFastJsonConfig() {
        FastJsonConfig config = new FastJsonConfig();
        config.setCharset(StandardCharsets.UTF_8);
        config.setDateFormat("yyyy-MM-dd HH:mm:ss");
        config.setReaderFeatures(
            JSONReader.Feature.FieldBased,
            JSONReader.Feature.SupportArrayToBean,
            JSONReader.Feature.TrimString
        );
        config.setWriterFeatures(
            JSONWriter.Feature.WriteMapNullValue,
            JSONWriter.Feature.PrettyFormat,
            JSONWriter.Feature.BrowserCompatible,
            JSONWriter.Feature.WriteNonStringKeyAsString,
            JSONWriter.Feature.WriteNullStringAsEmpty,
            JSONWriter.Feature.WriteEnumUsingToString,
            JSONWriter.Feature.WriteBigDecimalAsPlain
        );
        return config;
    }
}
```

### 3. 在前端处理

如果您无法更改后端代码，可以在前端处理接收到的雪花 ID。将其作为字符串处理，以避免精度损失：

```javascript
// 假设从后端接收到的雪花 ID
let snowflakeId = "1234567890123456789"; // 作为字符串处理
console.log(snowflakeId); // 输出: 1234567890123456789
```

### 4. 使用 BigInt

在现代 JavaScript 中，您可以使用 `BigInt` 来处理大整数。这样可以避免精度损失：

```javascript
let snowflakeId = BigInt("1234567890123456789");
console.log(snowflakeId); // 输出: 1234567890123456789n
```

## 结论

在处理数据库存储的 Long 类型雪花 ID 时，前端的精度损失是一个常见问题。通过修改 Jackson 的序列化、使用 Fastjson2、在前端处理或使用 `BigInt`，可以有效解决这一问题。选择合适的解决方案可以确保系统的稳定性和数据的准确性。

---

希望这篇文章能帮助您更好地解决Long类型前端出现精度丢失的问题。如果您有任何问题，欢迎在评论区讨论！ 
---
