---
title: 【Docker】Sentinel部署指南
tags:
  - Docker
  - Sentinel
  - 微服务
  - 容器化
categories:
  - Linux
  - Docker
abbrlink: d7a9c123
date: 2025-03-02 11:00:00
---

# Docker部署Sentinel指南

## 1. 简介

Sentinel是阿里巴巴开源的面向分布式服务架构的流量控制组件，主要以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度来保护服务的稳定性。本文将详细介绍如何使用Docker部署Sentinel控制台，并进行相关配置。

## 2. 环境准备

在开始部署之前，请确保您的系统已经具备以下条件：

- Docker Engine (版本 20.10.0 或更高)
- Docker Compose (可选，用于多容器部署)
- 至少 1GB 可用内存
- 可用的 8080 端口（Sentinel 控制台默认端口）

## 3. 单机部署

### 3.1 拉取官方镜像

```bash
docker pull bladex/sentinel-dashboard:latest
```

### 3.2 创建配置文件

创建 `sentinel-config` 目录用于存储配置文件：

```bash
mkdir -p /data/sentinel/conf
```

### 3.3 启动容器

```bash
docker run -d \
    --name sentinel-dashboard \
    -p 8080:8080 \
    -v /data/sentinel/conf:/root/logs \
    --restart always \
    bladex/sentinel-dashboard:latest
```

参数说明：
- `-p 8080:8080`: 映射容器的8080端口到主机
- `-v /data/sentinel/conf:/root/logs`: 挂载日志目录
- `--restart always`: 容器自动重启

### 3.4 访问控制台

启动成功后，通过浏览器访问：`http://localhost:8080`

默认登录账号密码：
- 用户名：sentinel
- 密码：sentinel

## 4. 集群部署

### 4.1 创建docker-compose.yml

```yaml
version: '3'
services:
  sentinel-dashboard1:
    image: bladex/sentinel-dashboard:latest
    container_name: sentinel-dashboard1
    ports:
      - "8080:8080"
    volumes:
      - /data/sentinel/conf1:/root/logs
    environment:
      - JAVA_OPTS="-Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080"
    restart: always

  sentinel-dashboard2:
    image: bladex/sentinel-dashboard:latest
    container_name: sentinel-dashboard2
    ports:
      - "8081:8080"
    volumes:
      - /data/sentinel/conf2:/root/logs
    environment:
      - JAVA_OPTS="-Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8081"
    restart: always

  nginx:
    image: nginx:latest
    container_name: sentinel-nginx
    ports:
      - "80:80"
    volumes:
      - /data/sentinel/nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      - sentinel-dashboard1
      - sentinel-dashboard2
    restart: always
```

### 4.2 配置Nginx负载均衡

创建 `/data/sentinel/nginx/conf.d/sentinel.conf`：

```nginx
upstream sentinel {
    server sentinel-dashboard1:8080;
    server sentinel-dashboard2:8080;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://sentinel;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 4.3 启动集群

```bash
docker-compose up -d
```

## 5. 配置优化

### 5.1 JVM参数调整

可以通过环境变量 `JAVA_OPTS` 调整JVM参数：

```bash
docker run -d \
    --name sentinel-dashboard \
    -p 8080:8080 \
    -e JAVA_OPTS="-Xms256m -Xmx512m -Dserver.port=8080" \
    bladex/sentinel-dashboard:latest
```

### 5.2 持久化配置

添加以下环境变量开启持久化：

```bash
-e JAVA_OPTS="-Dsentinel.dashboard.auth.username=admin \
               -Dsentinel.dashboard.auth.password=admin123 \
               -Dserver.servlet.session.timeout=7200"
```

## 6. 常见问题

### 6.1 访问控制台失败

1. 检查端口映射是否正确
2. 确认防火墙设置
3. 验证容器运行状态：
```bash
docker logs sentinel-dashboard
```

### 6.2 客户端注册失败

1. 检查网络连通性
2. 确认客户端配置是否正确
3. 查看日志中的具体错误信息

## 总结

通过Docker部署Sentinel可以大大简化安装和维护过程。本文详细介绍了单机版和集群版的部署方法，包括必要的配置和优化建议。在生产环境中，建议使用集群模式部署，并根据实际需求调整配置参数。

---

希望这篇文章能帮助您更好地理解和使用Docker部署Sentinel。如果您有任何问题，欢迎在评论区讨论！