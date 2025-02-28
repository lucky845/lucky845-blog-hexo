---
title: 【Docker】MySQL部署与配置指南
tags:
  - Docker
  - MySQL
  - 容器化
  - 配置
  - 数据库
categories:
  - Linux
  - Docker
abbrlink: e55fa598
date: 2025-02-28 16:00:00
---

## 问题背景

MySQL是一个流行的关系型数据库管理系统，在Docker环境中部署MySQL可以简化安装过程，提高部署效率，并实现更好的资源隔离和管理。本文将详细介绍如何使用Docker部署和配置MySQL服务，包括单节点部署、Docker Compose配置以及性能优化等内容。

## 环境准备

在开始部署之前，请确保您的系统已经安装：

- Docker Engine (版本 20.10.0 或更高)
- Docker Compose (可选，用于多容器部署)

## 单节点部署

### 1. 拉取MySQL镜像

```bash
# 拉取最新版本的MySQL官方镜像
docker pull mysql:8.0
```

### 2. 创建数据持久化目录

```bash
# 创建本地目录用于数据持久化
mkdir -p /data/mysql/data
mkdir -p /data/mysql/conf
mkdir -p /data/mysql/logs
```

### 3. 创建自定义配置文件

创建 `/data/mysql/conf/my.cnf` 配置文件：

```cnf
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
default-time-zone=+8:00
max_connections=1000

# 日志配置
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=2

# InnoDB配置
innodb_buffer_pool_size=1G
innodb_log_file_size=256M
innodb_flush_log_at_trx_commit=2
```

### 4. 启动MySQL容器

```bash
# 基本启动命令
docker run -d \
  --name mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=your_password \
  -e MYSQL_DATABASE=your_db \
  -v /data/mysql/data:/var/lib/mysql \
  -v /data/mysql/conf/my.cnf:/etc/mysql/conf.d/my.cnf \
  -v /data/mysql/logs:/var/log/mysql \
  --restart=always \
  mysql:8.0
```

## Docker Compose配置

### 1. 创建docker-compose.yml文件

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=your_password
      - MYSQL_DATABASE=your_db
      - MYSQL_USER=your_user
      - MYSQL_PASSWORD=your_user_password
    volumes:
      - /data/mysql/data:/var/lib/mysql
      - /data/mysql/conf/my.cnf:/etc/mysql/conf.d/my.cnf
      - /data/mysql/logs:/var/log/mysql
    command: --default-authentication-plugin=mysql_native_password
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### 2. 启动服务

```bash
docker-compose up -d
```

## MySQL配置优化

### 1. 内存配置

在my.cnf中添加以下配置：

```cnf
# 缓冲池大小，建议设置为系统内存的50%-70%
innodb_buffer_pool_size=4G

# 查询缓存
query_cache_size=64M
query_cache_type=1

# 排序缓冲区
sort_buffer_size=4M
join_buffer_size=4M
```

### 2. 并发配置

```cnf
# 最大连接数
max_connections=1000

# 线程缓存
thread_cache_size=16

# 表缓存
table_open_cache=2000
```

### 3. 日志配置

```cnf
# 慢查询日志
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=2

# 错误日志
log_error=/var/log/mysql/error.log

# 二进制日志
log_bin=/var/log/mysql/mysql-bin.log
expire_logs_days=7
max_binlog_size=100M
```

## 数据备份与恢复

### 1. 备份数据

```bash
# 使用Docker执行备份命令
docker exec mysql sh -c 'exec mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --all-databases' > /path/to/backup.sql
```

### 2. 恢复数据

```bash
# 恢复数据到容器
cat /path/to/backup.sql | docker exec -i mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"'
```

## 多实例部署

对于需要部署MySQL主从复制的场景，可以使用以下Docker Compose配置：

```yaml
version: '3.8'

services:
  mysql-master:
    image: mysql:8.0
    container_name: mysql-master
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=master_password
    volumes:
      - /data/mysql-master/data:/var/lib/mysql
      - /data/mysql-master/conf/my.cnf:/etc/mysql/conf.d/my.cnf
    restart: always

  mysql-slave:
    image: mysql:8.0
    container_name: mysql-slave
    ports:
      - "3307:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=slave_password
    volumes:
      - /data/mysql-slave/data:/var/lib/mysql
      - /data/mysql-slave/conf/my.cnf:/etc/mysql/conf.d/my.cnf
    restart: always
    depends_on:
      - mysql-master
```

主服务器配置 `/data/mysql-master/conf/my.cnf`：

```cnf
[mysqld]
server-id=1
log_bin=mysql-bin
gtid_mode=ON
enforce_gtid_consistency=ON
binlog_format=ROW
```

从服务器配置 `/data/mysql-slave/conf/my.cnf`：

```cnf
[mysqld]
server-id=2
log_bin=mysql-bin
gtid_mode=ON
enforce_gtid_consistency=ON
read_only=ON
```

## 常见问题与解决方案

### 1. 容器无法启动

检查日志：

```bash
docker logs mysql
```

常见原因：
- 数据目录权限问题
- 配置文件格式错误
- 端口冲突

### 2. 连接被拒绝

确保：
- MySQL服务正在运行
- 端口映射正确
- 防火墙设置允许连接
- 用户有正确的访问权限

### 3. 性能问题

- 检查容器资源限制
- 优化MySQL配置参数
- 使用数据卷而非绑定挂载提高I/O性能

## 总结

通过Docker部署MySQL可以大大简化数据库的安装、配置和管理过程。本文介绍了单节点部署、Docker Compose配置、性能优化以及常见问题的解决方案。在生产环境中，建议根据实际需求调整配置参数，并确保数据的安全性和可靠性。

---

希望这篇文章能帮助您更好地理解和使用Docker部署MySQL。如果您有任何问题，欢迎在评论区讨论！