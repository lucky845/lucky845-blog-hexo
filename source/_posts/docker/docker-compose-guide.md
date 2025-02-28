---
title: 【Docker】Compose使用与配置指南
tags:
  - Docker
  - Docker Compose
  - 容器化
  - 配置
  - 系统管理
categories:
  - Linux
  - Docker
abbrlink: d55fa596
date: 2025-02-28 12:00:00
---

## 问题背景

随着容器化应用的增多，单独管理多个容器变得越来越复杂。Docker Compose 是一个用于定义和运行多容器 Docker 应用程序的工具，通过一个 YAML 文件配置应用的服务，然后使用一个命令创建并启动所有服务。本文将详细介绍 Docker Compose 的使用方法与配置技巧。

## 1. Docker Compose 简介

### 1.1 什么是 Docker Compose

Docker Compose 是 Docker 官方提供的开源项目，用于定义和运行由多个容器组成的应用。使用 Compose，您可以通过一个 YAML 文件定义一个多容器的应用，然后使用一个命令完成应用的创建和启动。

### 1.2 Docker Compose 的主要优势

- **简化配置**：通过 YAML 文件声明式地定义应用服务
- **单一命令管理**：使用一条命令完成环境的创建和启动
- **环境隔离**：每个项目可以有独立的环境
- **保持容器数据卷**：Compose 可以保存数据卷中的数据
- **仅重新创建已更改的容器**：Compose 会检测配置变化，只重新创建变更的容器
- **变量支持**：支持环境变量和配置文件中的变量替换

## 2. 安装 Docker Compose

### 2.1 在 Linux 上安装

```bash
# 下载 Docker Compose 二进制文件
sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 添加可执行权限
sudo chmod +x /usr/local/bin/docker-compose

# 验证安装
docker-compose --version
```

### 2.2 在 macOS 上安装

如果已经安装了 Docker Desktop for Mac，则 Docker Compose 已经包含在其中。

### 2.3 在 Windows 上安装

如果已经安装了 Docker Desktop for Windows，则 Docker Compose 已经包含在其中。

## 3. Docker Compose 文件结构

### 3.1 docker-compose.yml 基本结构

Docker Compose 使用 YAML 文件（通常命名为 `docker-compose.yml`）来定义服务配置：

```yaml
version: '3'  # Compose 文件版本

services:      # 定义服务
  web:         # 服务名称
    image: nginx:latest  # 使用的镜像
    ports:     # 端口映射
      - "80:80"
    volumes:   # 挂载卷
      - ./html:/usr/share/nginx/html
    networks:  # 网络配置
      - frontend

  db:          # 另一个服务
    image: mysql:5.7
    environment: # 环境变量
      MYSQL_ROOT_PASSWORD: example
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - backend

volumes:       # 定义卷
  db_data:     # 卷名称

networks:      # 定义网络
  frontend:    # 网络名称
  backend:
```

### 3.2 主要配置项说明

1. **version**：指定 Compose 文件格式版本
2. **services**：定义应用的各个服务
3. **volumes**：定义数据卷
4. **networks**：定义网络

## 4. Docker Compose 核心配置项

### 4.1 服务配置

```yaml
services:
  webapp:
    image: nginx:latest  # 使用现有镜像
    # 或者使用 Dockerfile 构建
    build: 
      context: ./dir  # 构建上下文路径
      dockerfile: Dockerfile-alternate  # 指定 Dockerfile
    container_name: my-web-container  # 容器名称
    restart: always  # 重启策略
    ports:  # 端口映射
      - "80:80"
      - "443:443"
    environment:  # 环境变量
      - NODE_ENV=production
    env_file:  # 从文件加载环境变量
      - ./common.env
    depends_on:  # 依赖关系
      - db
      - redis
    volumes:  # 数据卷
      - ./data:/data
      - logs:/var/log
    networks:  # 网络
      - frontend
    deploy:  # 部署配置（Swarm 模式）
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 50M
```

### 4.2 网络配置

```yaml
networks:
  frontend:
    driver: bridge  # 网络驱动
    driver_opts:  # 驱动选项
      com.docker.network.bridge.name: frontend
    ipam:  # IP 地址管理
      driver: default
      config:
        - subnet: 172.28.0.0/16
  backend:
    external: true  # 使用已存在的外部网络
```

### 4.3 数据卷配置

```yaml
volumes:
  db_data:  # 普通卷
  cached_data:
    driver: local  # 卷驱动
    driver_opts:  # 驱动选项
      type: none
      device: /path/on/host
      o: bind
  external_data:
    external: true  # 使用已存在的外部卷
```

## 5. Docker Compose 常用命令

### 5.1 启动和停止服务

```bash
# 创建并启动所有服务
docker-compose up

# 后台运行服务
docker-compose up -d

# 停止并移除所有服务
docker-compose down

# 停止服务但不移除容器、网络等
docker-compose stop

# 启动已停止的服务
docker-compose start
```

### 5.2 服务管理

```bash
# 查看服务状态
docker-compose ps

# 查看服务日志
docker-compose logs

# 实时查看特定服务的日志
docker-compose logs -f service_name

# 在指定服务中执行命令
docker-compose exec service_name command

# 重新构建服务
docker-compose build service_name
```

### 5.3 扩展服务实例

```bash
# 将 worker 服务扩展到 3 个实例
docker-compose up -d --scale worker=3
```

## 6. 实际应用案例

### 6.1 Web 应用 + 数据库 + Redis 缓存

```yaml
version: '3'

services:
  web:
    build: ./web
    ports:
      - "8000:8000"
    depends_on:
      - db
      - redis
    environment:
      - DATABASE_URL=postgres://postgres:postgres@db:5432/app
      - REDIS_URL=redis://redis:6379/0
    restart: always

  db:
    image: postgres:13
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=app

  redis:
    image: redis:6
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 6.2 WordPress + MySQL

```yaml
version: '3'

services:
  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db_data:/var/lib/mysql

volumes:
  wordpress_data:
  db_data:
```

## 7. Docker Compose 最佳实践

1. **使用版本控制**：将 docker-compose.yml 文件纳入版本控制系统
2. **环境变量分离**：使用 .env 文件或环境变量管理敏感信息
3. **合理组织服务**：相关服务放在同一个 Compose 文件中，不相关的分开
4. **使用健康检查**：为服务配置健康检查，确保依赖服务正常运行
5. **资源限制**：为服务设置资源限制，避免单个服务消耗过多资源
6. **日志管理**：配置适当的日志驱动和轮转策略
7. **网络隔离**：使用多个网络隔离不同服务组
8. **数据持久化**：使用命名卷而非绑定挂载来持久化数据

## 8. 常见问题与解决方案

### 8.1 服务启动顺序问题

虽然 `depends_on` 可以控制服务启动顺序，但它不会等待服务就绪。解决方案：

1. 使用健康检查：

```yaml
services:
  web:
    depends_on:
      db:
        condition: service_healthy
  db:
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

2. 使用等待脚本：

```yaml
services:
  web:
    command: ["./wait-for-it.sh", "db:5432", "--", "python", "app.py"]
```

### 8.2 网络连接问题

服务间通信使用服务名作为主机名，确保在同一网络中。

### 8.3 数据持久化问题

使用命名卷而非匿名卷，并在需要时使用外部卷。

## 总结

Docker Compose 是管理多容器应用的强大工具，通过简单的 YAML 配置文件和命令行工具，可以大大简化容器化应用的部署和管理。掌握 Docker Compose 的使用与配置，能够帮助开发者更高效地构建、测试和部署复杂的应用系统。

---

希望这篇文章能帮助您更好地理解和使用 Docker Compose。如果您有任何问题，欢迎在评论区讨论！
---