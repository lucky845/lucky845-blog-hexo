---
title: 【Docker】高级配置指南
tags:
  - Docker
  - 容器化
  - 配置
  - 系统管理
categories:
  - Linux
  - Docker
abbrlink: c55fa595
date: 2025-02-28 11:00:00
---

# Docker 高级配置指南

## Docker Daemon 配置

### daemon.json 配置文件

Docker daemon 的配置文件通常位于 `/etc/docker/daemon.json`，用于配置 Docker 引擎的行为：

```json
{
  "registry-mirrors": ["https://mirror.ccs.tencentyun.com"],
  "data-root": "/var/lib/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  }
}
```

### 主要配置项说明

1. **registry-mirrors**: 配置 Docker 镜像加速器
2. **data-root**: 指定 Docker 数据存储位置
3. **log-driver**: 设置容器日志驱动
4. **log-opts**: 配置日志相关选项
5. **default-ulimits**: 设置容器的系统资源限制

## 容器运行时配置

### 资源限制配置

使用 `docker run` 命令时可以配置容器的资源限制：

```bash
# 限制 CPU 和内存使用
docker run -d \
  --cpus=2 \
  --memory=2g \
  --memory-swap=4g \
  --name myapp \
  nginx
```

### 重启策略配置

配置容器的自动重启策略：

```bash
# 设置容器总是自动重启
docker run -d \
  --restart=always \
  --name myapp \
  nginx
```

## 网络配置

### 自定义网络

创建和使用自定义网络：

```bash
# 创建自定义网络
docker network create --driver bridge mynetwork

# 将容器连接到自定义网络
docker run -d \
  --network mynetwork \
  --name myapp \
  nginx
```

### 端口映射

配置容器端口映射：

```bash
# 映射多个端口
docker run -d \
  -p 80:80 \
  -p 443:443 \
  --name myapp \
  nginx
```

## 存储配置

### 数据卷配置

创建和使用数据卷：

```bash
# 创建数据卷
docker volume create mydata

# 使用数据卷
docker run -d \
  -v mydata:/data \
  --name myapp \
  nginx
```

### 绑定挂载

将主机目录挂载到容器：

```bash
# 挂载主机目录
docker run -d \
  -v /host/path:/container/path \
  --name myapp \
  nginx
```

## 安全配置

### 用户命名空间

启用用户命名空间映射：

```json
{
  "userns-remap": "default"
}
```

### 安全选项

配置容器安全选项：

```bash
# 以非特权模式运行容器
docker run -d \
  --security-opt=no-new-privileges \
  --cap-drop=ALL \
  --name myapp \
  nginx
```

## 日志配置

### 日志驱动配置

配置容器的日志驱动：

```bash
# 使用 syslog 日志驱动
docker run -d \
  --log-driver=syslog \
  --log-opt syslog-address=udp://1.2.3.4:1111 \
  --name myapp \
  nginx
```

## 最佳实践

1. **资源限制**: 始终为生产环境的容器设置资源限制
2. **日志轮转**: 配置适当的日志轮转策略避免磁盘空间耗尽
3. **网络隔离**: 使用自定义网络实现容器间的网络隔离
4. **数据持久化**: 使用数据卷而不是绑定挂载来持久化数据
5. **安全加固**: 遵循最小权限原则配置容器

## 总结

Docker 的高级配置涉及多个方面，包括 daemon 配置、容器运行时配置、网络配置、存储配置等。通过合理配置这些选项，可以构建更安全、更高效、更可靠的容器化环境。掌握这些配置选项对于在生产环境中部署和管理 Docker 容器至关重要。

---

希望这篇文章能帮助您更好地理解 Docker 的高级配置。如果您有任何问题，欢迎在评论区讨论！
---