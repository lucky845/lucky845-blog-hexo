---
title: 【Docker】 Docker部署Nacos（单机/集群）完全指南
date: 2025-03-02 10:00:00
tags:
  - Docker
  - Nacos
categories:
  - Docker部署指南
keywords: 'Docker,Nacos,部署,单机,集群'
description: 本文详细介绍如何使用Docker部署Nacos的单机版和集群版，包括环境准备、配置说明、性能优化等内容。
abbrlink: e55fa612
---

## 前言

Nacos 是阿里巴巴开源的一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。本文将详细介绍如何使用 Docker 来部署 Nacos 的单机版和集群版。

## 环境要求

在开始部署之前，请确保您的系统满足以下要求：

- Docker Engine (版本 20.10.0 或更高)
- Docker Compose (可选，用于多容器部署)
- 至少 2GB 可用内存
- 可用存储空间 >= 5GB

## 单机部署

### 1. 创建必要目录

```bash
# 创建数据持久化目录
mkdir -p /data/nacos/standalone/logs
mkdir -p /data/nacos/standalone/data
mkdir -p /data/nacos/standalone/conf
```

### 2. 配置文件准备

创建 `/data/nacos/standalone/conf/custom.properties` 配置文件：

```properties
# 数据持久化配置
nacos.core.auth.enabled=true
nacos.core.auth.server.identity.key=serverIdentity
nacos.core.auth.server.identity.value=security

# JVM配置
JVM_XMS=512m
JVM_XMX=512m
JVM_XMN=256m

# 默认超时时间设置
nacos.core.protocol.raft.data.read_timeout_ms=5000
nacos.core.protocol.raft.data.write_timeout_ms=5000

# 日志配置
nacos.core.auth.plugin.nacos.token.secret.key=SecretKey012345678901234567890123456789012345678901234567890123456789
```

### 3. 启动单机版Nacos

```bash
docker run -d \
  --name nacos-standalone \
  --restart always \
  -p 8848:8848 \
  -p 9848:9848 \
  -v /data/nacos/standalone/conf/custom.properties:/home/nacos/conf/application.properties \
  -v /data/nacos/standalone/logs:/home/nacos/logs \
  -v /data/nacos/standalone/data:/home/nacos/data \
  -e MODE=standalone \
  -e SPRING_DATASOURCE_PLATFORM=embedded \
  -e NACOS_AUTH_ENABLE=true \
  nacos/nacos-server:v2.2.3
```

### 4. 验证部署

访问Nacos控制台：`http://localhost:8848/nacos`

默认账号密码：
- 用户名：nacos
- 密码：nacos

## 集群部署

### 1. 准备MySQL数据库

集群模式需要使用MySQL存储配置信息。首先创建数据库并导入初始化SQL：

```sql
CREATE DATABASE nacos_config DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_general_ci;
```

在创建好数据库后，执行Nacos官方提供的[数据库初始化脚本](https://github.com/alibaba/nacos/blob/master/distribution/conf/mysql-schema.sql)。

### 2. 创建集群配置目录

```bash
# 为每个节点创建独立目录
mkdir -p /data/nacos/cluster/node{1,2,3}/{conf,logs,data}
```

### 3. 配置集群节点

创建 `/data/nacos/cluster/node1/conf/custom.properties` 配置文件（节点2和节点3类似）：

```properties
# 数据源配置
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://your-mysql-host:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=your_username
db.password.0=your_password

# 集群配置
nacos.core.auth.enabled=true
nacos.core.auth.server.identity.key=serverIdentity
nacos.core.auth.server.identity.value=security
nacos.core.auth.plugin.nacos.token.secret.key=SecretKey012345678901234567890123456789012345678901234567890123456789

# 集群节点列表
nacos.member.list=172.16.1.10:8848,172.16.1.11:8848,172.16.1.12:8848

# JVM配置
JVM_XMS=2g
JVM_XMX=2g
JVM_XMN=1g

# 性能调优
nacos.core.protocol.raft.data.read_timeout_ms=5000
nacos.core.protocol.raft.data.write_timeout_ms=5000
```

### 4. 使用Docker Compose启动集群

创建 `docker-compose.yml` 文件：

```yaml
version: '3'
services:
  nacos1:
    image: nacos/nacos-server:v2.2.3
    container_name: nacos-cluster-1
    networks:
      nacos_net:
        ipv4_address: 172.16.1.10
    volumes:
      - /data/nacos/cluster/node1/conf/custom.properties:/home/nacos/conf/application.properties
      - /data/nacos/cluster/node1/logs:/home/nacos/logs
      - /data/nacos/cluster/node1/data:/home/nacos/data
    ports:
      - "8848:8848"
      - "9848:9848"
    environment:
      - MODE=cluster
      - NACOS_AUTH_ENABLE=true
    restart: always

  nacos2:
    image: nacos/nacos-server:v2.2.3
    container_name: nacos-cluster-2
    networks:
      nacos_net:
        ipv4_address: 172.16.1.11
    volumes:
      - /data/nacos/cluster/node2/conf/custom.properties:/home/nacos/conf/application.properties
      - /data/nacos/cluster/node2/logs:/home/nacos/logs
      - /data/nacos/cluster/node2/data:/home/nacos/data
    ports:
      - "8849:8848"
      - "9849:9848"
    environment:
      - MODE=cluster
      - NACOS_AUTH_ENABLE=true
    restart: always

  nacos3:
    image: nacos/nacos-server:v2.2.3
    container_name: nacos-cluster-3
    networks:
      nacos_net:
        ipv4_address: 172.16.1.12
    volumes:
      - /data/nacos/cluster/node3/conf/custom.properties:/home/nacos/conf/application.properties
      - /data/nacos/cluster/node3/logs:/home/nacos/logs
      - /data/nacos/cluster/node3/data:/home/nacos/data
    ports:
      - "8850:8848"
      - "9850:9848"
    environment:
      - MODE=cluster
      - NACOS_AUTH_ENABLE=true
    restart: always

networks:
  nacos_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.16.1.0/24
```

启动集群：

```bash
docker-compose up -d
```

### 5. 配置负载均衡

为了实现高可用，建议在集群前面配置负载均衡。以下是使用Nginx的配置示例：

```nginx
upstream nacos-cluster {
    server 172.16.1.10:8848;
    server 172.16.1.11:8848;
    server 172.16.1.12:8848;
}

server {
    listen 80;
    server_name nacos.example.com;

    location / {
        proxy_pass http://nacos-cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## 性能优化建议

1. **JVM参数调整**
   - 根据服务器内存大小适当调整JVM参数
   - 建议开启G1垃圾收集器

2. **数据库优化**
   - 使用高性能SSD存储
   - 优化MySQL配置参数
   - 定期清理历史配置

3. **网络优化**
   - 确保集群节点间网络延迟低
   - 适当调整超时时间

## 常见问题

1. **启动失败**
   - 检查端口是否被占用
   - 查看日志文件排查错误
   - 确认配置文件格式正确

2. **节点无法加入集群**
   - 检查网络连接
   - 验证集群地址配置
   - 确认MySQL连接正常

3. **内存占用过高**
   - 调整JVM参数
   - 检查是否存在内存泄漏
   - 考虑增加节点数量

## 总结

通过Docker部署Nacos可以大大简化安装和维护过程。本文详细介绍了单机版和集群版的部署方法，包括必要的配置和优化建议。在生产环境中，建议使用集群模式部署，并根据实际需求调整配置参数。

## 参考资料

- [Nacos官方文档](https://nacos.io/zh-cn/docs/what-is-nacos.html)
- [Docker Hub - Nacos](https://hub.docker.com/r/nacos/nacos-server)
- [Nacos GitHub](https://github.com/alibaba/nacos)

---

希望这篇文章能帮助您更好地理解和部署Nacos。如果您有任何问题，欢迎在评论区讨论！