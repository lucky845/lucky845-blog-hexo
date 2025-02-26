---
title: 【Spring Boot】解决跨域问题的几种方式
tags:
  - Java
  - Spring Boot
  - 跨域
  - CORS
categories:
  - Java
  - Spring Boot
abbrlink: b55fa605
date: 2025-02-26 12:00:00
---

## 问题背景

在现代 Web 开发中，跨域资源共享（CORS）是一个常见的问题。当前端应用程序尝试从不同的域名、协议或端口访问后端 API 时，浏览器会阻止这种请求以保护用户的安全。本文将介绍在 Spring Boot 中解决跨域问题的几种常见方式。

## 1. 使用 `@CrossOrigin` 注解

Spring Boot 提供了 `@CrossOrigin` 注解，可以方便地在控制器或方法上启用 CORS。

### 示例代码

```java
import org.springframework.web.bind.annotation.CrossOrigin;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@CrossOrigin(origins = "http://example.com") // 允许来自指定域的请求
public class MyController {

    @GetMapping("/api/data")
    public String getData() {
        return "Hello, World!";
    }
}
```

### 说明

- `@CrossOrigin` 注解可以放在类级别或方法级别。
- `origins` 属性指定允许的源，可以是单个域名或多个域名。

## 2. 全局配置 CORS

如果希望为整个应用程序配置 CORS，可以通过实现 `WebMvcConfigurer` 接口来进行全局配置。

### 示例代码

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**") // 允许所有路径
                .allowedOrigins("http://example.com") // 允许的源
                .allowedMethods("GET", "POST", "PUT", "DELETE") // 允许的请求方法
                .allowCredentials(true); // 允许携带凭证
    }
}
```

### 说明

- `addMapping("/**")` 表示允许所有路径的跨域请求。
- `allowedMethods` 可以指定允许的 HTTP 方法。

## 3. 使用 Filter 进行 CORS 处理

另一种方式是通过自定义 Filter 来处理 CORS 请求。

### 示例代码

```java
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

import org.springframework.stereotype.Component;

@Component
public class CorsFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        httpResponse.setHeader("Access-Control-Allow-Origin", "http://example.com");
        httpResponse.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
        httpResponse.setHeader("Access-Control-Allow-Headers", "Content-Type");
        chain.doFilter(request, response);
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {}

    @Override
    public void destroy() {}
}
```

### 说明

- 通过设置响应头来允许跨域请求。
- 这种方式适用于需要更复杂的 CORS 处理逻辑的场景。

## 4. 使用 Spring Security 配置 CORS

如果您的应用程序使用了 Spring Security，可以在安全配置中添加 CORS 支持。

### 示例代码

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.cors() // 启用 CORS
            .and()
            .csrf().disable(); // 根据需要禁用 CSRF
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(Arrays.asList("http://example.com"));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
        configuration.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### 说明

- 在 Spring Security 配置中启用 CORS。
- 通过 `CorsConfigurationSource` 定义允许的源和方法。

## 5. 使用 Nginx 处理 CORS

如果您的应用程序通过 Nginx 进行反向代理，可以在 Nginx 配置中添加 CORS 支持。

### 示例配置

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        add_header 'Access-Control-Allow-Origin' 'http://example.com';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'Content-Type';
        
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Max-Age' 86400;
            add_header 'Content-Length' 0;
            return 204;
        }

        proxy_pass http://localhost:8080; # 代理到 Spring Boot 应用
    }
}
```

### 说明

- 在 Nginx 配置中使用 `add_header` 指令添加 CORS 相关的响应头。
- 处理 `OPTIONS` 请求以支持预检请求。

## 6. 前端解决跨域问题

在某些情况下，前端可以通过 JSONP 或使用代理服务器来解决跨域问题。

### 示例代码（使用代理）

在开发环境中，可以使用 Webpack 的代理功能：

```javascript
// webpack.config.js
module.exports = {
    devServer: {
        proxy: {
            '/api': {
                target: 'http://localhost:8080',
                changeOrigin: true,
            },
        },
    },
};
```

### 说明

- 通过代理将请求转发到后端，避免跨域问题。

## 总结

在 Spring Boot 中解决跨域问题有多种方式，开发者可以根据具体需求选择合适的方法。无论是使用注解、全局配置、Filter、Spring Security、Nginx 还是前端代理，正确配置 CORS 都能确保前后端的顺利交互。

## 参考资料

- [Spring Boot CORS Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-cors)
- [MDN Web Docs - CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

---

希望这篇文章能帮助您更好地理解和解决 Spring Boot 中的跨域问题。如果您有任何问题，欢迎在评论区讨论！
--- 