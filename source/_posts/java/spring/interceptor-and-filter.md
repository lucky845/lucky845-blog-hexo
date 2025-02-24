---
title: 【Spring Boot】如何在所有 Web 请求的前后执行自定义代码
tags:
  - Java
  - Spring Boot
  - 拦截器
  - 过滤器
categories:
  - Java
  - Spring
abbrlink: b55fa576
date: 2024-02-25 07:00:00
---

## 问题背景

在开发 Web 应用时，常常需要在请求处理的前后执行一些自定义代码，例如记录日志、身份验证、性能监控等。Spring Boot 提供了两种主要机制来实现这一点：**拦截器**（Interceptor）和 **过滤器**（Filter）。本文将介绍如何使用这两种机制在所有 Web 请求的前后执行自定义代码。

## 1. 使用拦截器

### 1.1 创建拦截器

拦截器是 Spring MVC 提供的一种机制，可以在请求处理之前和之后执行自定义逻辑。首先，我们需要实现 `HandlerInterceptor` 接口。

```java
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class CustomInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("请求前处理: " + request.getRequestURI());
        // 返回 true 继续处理请求，返回 false 中断请求
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("请求后处理: " + request.getRequestURI());
    }
}
```

### 1.2 注册拦截器

接下来，我们需要在 Spring Boot 中注册这个拦截器。可以通过实现 `WebMvcConfigurer` 接口来完成。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private CustomInterceptor customInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(customInterceptor).addPathPatterns("/**"); // 拦截所有请求
    }
}
```

### 1.3 使用案例

在这个例子中，`CustomInterceptor` 会在每个请求处理之前打印请求的 URI，并在请求完成后打印相同的信息。可以根据需要在 `preHandle` 和 `afterCompletion` 方法中添加更多的自定义逻辑。

## 2. 使用过滤器

### 2.1 创建过滤器

过滤器是 Servlet 规范的一部分，可以在请求和响应的整个生命周期中执行自定义逻辑。首先，我们需要实现 `Filter` 接口。

```java
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import java.io.IOException;

public class CustomFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("请求前处理");
        chain.doFilter(request, response); // 继续处理请求
        System.out.println("请求后处理");
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // 初始化代码
    }

    @Override
    public void destroy() {
        // 清理代码
    }
}
```

### 2.2 注册过滤器

接下来，我们需要在 Spring Boot 中注册这个过滤器。可以通过实现 `FilterRegistrationBean` 来完成。

```java
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<CustomFilter> loggingFilter() {
        FilterRegistrationBean<CustomFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(new CustomFilter());
        registrationBean.addUrlPatterns("/*"); // 拦截所有请求
        return registrationBean;
    }
}
```

### 2.3 使用案例

在这个例子中，`CustomFilter` 会在每个请求处理之前和之后打印日志。与拦截器不同，过滤器可以处理请求和响应的整个生命周期，适合用于跨域处理、日志记录等场景。

## 3. 拦截器与过滤器的区别

| 特性         | 拦截器                     | 过滤器                     |
|--------------|----------------------------|----------------------------|
| 适用范围     | Spring MVC                 | Servlet                     |
| 处理请求     | 在 Controller 之前和之后  | 在请求和响应的整个生命周期 |
| 访问 Spring 上下文 | 可以访问                  | 不能直接访问               |
| 适用场景     | 业务逻辑处理、权限控制    | 日志记录、性能监控、跨域处理 |

## 总结

通过使用拦截器和过滤器，我们可以在 Spring Boot 应用中轻松地在所有 Web 请求的前后执行自定义代码。根据具体需求选择合适的机制，可以提高应用的灵活性和可维护性。

## 参考资料

- [Spring Boot - Interceptors](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-interceptors)
- [Servlet Filters](https://docs.oracle.com/javaee/7/api/javax/servlet/Filter.html)

---

希望这篇文章能帮助您更好地理解和实现 Spring Boot 中的请求前后处理。如果您有任何问题，欢迎在评论区讨论！
--- 
 