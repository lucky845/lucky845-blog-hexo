---
title: 【Spring】如何静态获取 HttpServletRequest 和 Response
tags:
  - Java
  - Spring
  - Servlet
  - Web
categories:
  - Java
  - Spring
abbrlink: b55fa566
date: 2025-02-24 21:00:00
---

## 问题背景

在 Spring Web 应用中，我们经常需要在非 Controller 层获取当前请求的 HttpServletRequest 和 HttpServletResponse 对象。虽然在 Controller 中可以直接通过参数注入获取这些对象，但在工具类或其他静态方法中获取它们则需要一些特殊处理。

## 解决方案

### 1. 创建 RequestContextHolder 工具类

首先，创建一个工具类来持有和获取 Request 和 Response 对象：

```java
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class ServletUtils {
    
    /**
     * 获取 HttpServletRequest 对象
     */
    public static HttpServletRequest getRequest() {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        return attributes != null ? attributes.getRequest() : null;
    }
    
    /**
     * 获取 HttpServletResponse 对象
     */
    public static HttpServletResponse getResponse() {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        return attributes != null ? attributes.getResponse() : null;
    }
}
```

### 2. IP 地址工具类

使用获取到的 Request 对象，我们可以编写一个获取请求者真实 IP 地址的工具类：

```java
public class IpUtils {
    
    private static final String[] IP_HEADER_NAMES = {
            "X-Forwarded-For",
            "Proxy-Client-IP",
            "WL-Proxy-Client-IP",
            "HTTP_X_FORWARDED_FOR",
            "HTTP_X_FORWARDED",
            "HTTP_X_CLUSTER_CLIENT_IP",
            "HTTP_CLIENT_IP",
            "HTTP_FORWARDED_FOR",
            "HTTP_FORWARDED",
            "HTTP_VIA",
            "REMOTE_ADDR"
    };
    
    /**
     * 获取请求者的真实IP地址
     */
    public static String getClientIp() {
        HttpServletRequest request = ServletUtils.getRequest();
        if (request == null) {
            return "";
        }
        
        String ip = null;
        for (String header : IP_HEADER_NAMES) {
            ip = request.getHeader(header);
            if (!isUnknown(ip)) {
                break;
            }
        }
        
        if (isUnknown(ip)) {
            ip = request.getRemoteAddr();
        }
        
        // 多个代理的情况，第一个IP为客户端真实IP
        if (ip != null && ip.contains(",")) {
            ip = ip.split(",")[0].trim();
        }
        
        return "0:0:0:0:0:0:0:1".equals(ip) ? "127.0.0.1" : ip;
    }
    
    private static boolean isUnknown(String ip) {
        return ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip);
    }
}
```

### 3. Request 对象的常用方法

使用 Request 对象可以获取很多有用的信息：

```java
public class RequestUtils {
    
    /**
     * 获取请求的URL
     */
    public static String getRequestUrl() {
        HttpServletRequest request = ServletUtils.getRequest();
        return request != null ? request.getRequestURL().toString() : "";
    }
    
    /**
     * 获取请求方法
     */
    public static String getMethod() {
        HttpServletRequest request = ServletUtils.getRequest();
        return request != null ? request.getMethod() : "";
    }
    
    /**
     * 获取请求头信息
     */
    public static String getHeader(String name) {
        HttpServletRequest request = ServletUtils.getRequest();
        return request != null ? request.getHeader(name) : "";
    }
    
    /**
     * 获取请求参数
     */
    public static String getParameter(String name) {
        HttpServletRequest request = ServletUtils.getRequest();
        return request != null ? request.getParameter(name) : "";
    }
    
    /**
     * 判断是否是Ajax请求
     */
    public static boolean isAjaxRequest() {
        HttpServletRequest request = ServletUtils.getRequest();
        if (request == null) {
            return false;
        }
        String requestedWith = request.getHeader("X-Requested-With");
        return "XMLHttpRequest".equals(requestedWith);
    }
}
```

### 4. Response 对象的常用方法

Response 对象主要用于处理响应：

```java
public class ResponseUtils {
    
    /**
     * 输出JSON数据
     */
    public static void renderJson(Object data) {
        HttpServletResponse response = ServletUtils.getResponse();
        if (response != null) {
            response.setContentType("application/json;charset=UTF-8");
            try {
                response.getWriter().write(new ObjectMapper().writeValueAsString(data));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    /**
     * 设置响应头
     */
    public static void setHeader(String name, String value) {
        HttpServletResponse response = ServletUtils.getResponse();
        if (response != null) {
            response.setHeader(name, value);
        }
    }
    
    /**
     * 文件下载
     */
    public static void download(String fileName, byte[] data) {
        HttpServletResponse response = ServletUtils.getResponse();
        if (response != null) {
            response.setContentType("application/octet-stream");
            response.setHeader("Content-Disposition", 
                    "attachment;filename=" + URLEncoder.encode(fileName, StandardCharsets.UTF_8));
            try {
                response.getOutputStream().write(data);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

## 使用示例

### 1. 获取客户端IP

```java
@RestController
public class DemoController {
    
    @GetMapping("/ip")
    public String getClientIp() {
        return "Your IP is: " + IpUtils.getClientIp();
    }
}
```

### 2. 判断请求类型

```java
@Service
public class UserService {
    
    public void processRequest() {
        if (RequestUtils.isAjaxRequest()) {
            // 处理Ajax请求
        } else {
            // 处理普通请求
        }
    }
}
```

## 最佳实践建议

1. **异常处理**：在获取 Request 和 Response 对象时要注意空指针检查。

2. **线程安全**：RequestContextHolder 是线程安全的，可以放心使用。

3. **性能考虑**：避免频繁调用获取 Request 和 Response 的方法，可以在需要时将对象保存在局部变量中。

4. **代理环境**：在获取客户端 IP 时要考虑代理服务器的情况。

## 常见问题

1. **为什么获取到的是 null？**
   - 确保在 Web 请求上下文中调用这些方法
   - 检查是否正确配置了 Spring 的 RequestContextListener

2. **如何处理跨域请求？**
   - 可以通过 Response 设置相应的 CORS 头信息

3. **如何处理大文件下载？**
   - 使用流式处理，避免一次性加载整个文件到内存

## 总结

通过静态方法获取 HttpServletRequest 和 HttpServletResponse 对象，我们可以在任何地方方便地访问请求和响应信息，实现更灵活的功能。

## 参考资料

- [Spring Web Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html)
- [Servlet Documentation](https://javaee.github.io/servlet-spec/)

---

希望这篇文章能帮助您更好地理解和使用 HttpServletRequest 和 HttpServletResponse。如果您有任何问题，欢迎在评论区讨论！
--- 
 