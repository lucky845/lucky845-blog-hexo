---
title: 【Docker】基础命令介绍
tags:
  - Docker
  - 容器化
  - 基础
  - 系统管理
categories:
  - Linux
  - Docker
abbrlink: b55fa594
date: 2025-02-28 11:00:00
---

## 问题背景

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 或 Windows 机器上。掌握 Docker 的基础命令是使用和管理容器化应用的关键。本文将介绍一些常用的 Docker 基础命令及其用法。

## 1. Docker 环境管理命令

### 1.1 `docker version`

显示 Docker 版本信息。

```bash
docker version
```

### 1.2 `docker info`

显示 Docker 系统信息，包括镜像和容器数量。

```bash
docker info
```

### 1.3 `docker login`

登录到 Docker 仓库。

```bash
docker login [OPTIONS] [SERVER]
```

## 2. 镜像管理命令

### 2.1 `docker images`

列出本地镜像。

```bash
docker images          # 列出所有镜像
docker images -a       # 列出所有镜像（包括中间层镜像）
docker images --digests # 显示摘要信息
```

### 2.2 `docker pull`

从镜像仓库拉取镜像。

```bash
docker pull ubuntu:20.04  # 拉取指定标签的镜像
docker pull ubuntu        # 拉取最新版本的镜像
```

### 2.3 `docker rmi`

删除本地镜像。

```bash
docker rmi image_id          # 删除指定 ID 的镜像
docker rmi ubuntu:20.04      # 删除指定标签的镜像
docker rmi $(docker images -q) # 删除所有镜像
```

### 2.4 `docker build`

从 Dockerfile 构建镜像。

```bash
docker build -t myimage:tag .  # 在当前目录构建镜像
docker build -f Dockerfile.dev -t myimage:dev .  # 使用指定的 Dockerfile
```

### 2.5 `docker tag`

为镜像添加标签。

```bash
docker tag source_image:tag target_image:tag
```

### 2.6 `docker save`

将镜像保存为 tar 归档文件。

```bash
docker save -o myimage.tar myimage:latest
```

### 2.7 `docker load`

从 tar 归档文件加载镜像。

```bash
docker load -i myimage.tar
```

## 3. 容器管理命令

### 3.1 `docker run`

创建并启动一个新容器。

```bash
docker run -it ubuntu:20.04 /bin/bash  # 交互式启动容器
docker run -d nginx                    # 后台运行容器
docker run -p 8080:80 nginx            # 端口映射
docker run -v /host/path:/container/path nginx  # 挂载卷
```

### 3.2 `docker ps`

列出运行中的容器。

```bash
docker ps          # 列出运行中的容器
docker ps -a       # 列出所有容器（包括已停止的）
docker ps -q       # 只显示容器 ID
```

### 3.3 `docker start/stop/restart`

启动、停止或重启容器。

```bash
docker start container_id    # 启动容器
docker stop container_id     # 停止容器
docker restart container_id  # 重启容器
```

### 3.4 `docker exec`

在运行中的容器中执行命令。

```bash
docker exec -it container_id /bin/bash  # 进入容器交互式终端
docker exec container_id ls /app        # 在容器中执行命令
```

### 3.5 `docker rm`

删除容器。

```bash
docker rm container_id          # 删除指定容器
docker rm $(docker ps -aq)      # 删除所有容器
docker rm -f container_id       # 强制删除运行中的容器
```

### 3.6 `docker logs`

查看容器日志。

```bash
docker logs container_id        # 查看容器日志
docker logs -f container_id     # 实时查看日志
docker logs --tail 100 container_id  # 查看最后 100 行日志
```

### 3.7 `docker inspect`

查看容器或镜像的详细信息。

```bash
docker inspect container_id     # 查看容器详情
docker inspect image_id         # 查看镜像详情
```

### 3.8 `docker stats`

显示容器资源使用统计信息。

```bash
docker stats                    # 显示所有容器的统计信息
docker stats container_id       # 显示指定容器的统计信息
```

## 4. 网络管理命令

### 4.1 `docker network ls`

列出 Docker 网络。

```bash
docker network ls
```

### 4.2 `docker network create`

创建 Docker 网络。

```bash
docker network create mynetwork
docker network create --driver bridge mynetwork
```

### 4.3 `docker network connect/disconnect`

将容器连接到网络或断开连接。

```bash
docker network connect mynetwork container_id
docker network disconnect mynetwork container_id
```

## 5. 数据卷管理命令

### 5.1 `docker volume ls`

列出 Docker 数据卷。

```bash
docker volume ls
```

### 5.2 `docker volume create`

创建 Docker 数据卷。

```bash
docker volume create myvolume
```

### 5.3 `docker volume rm`

删除 Docker 数据卷。

```bash
docker volume rm myvolume
docker volume rm $(docker volume ls -q)  # 删除所有未使用的数据卷
```

## 6. Docker Compose 命令

### 6.1 `docker-compose up`

创建并启动 Docker Compose 定义的服务。

```bash
docker-compose up        # 启动服务
docker-compose up -d     # 后台启动服务
```

### 6.2 `docker-compose down`

停止并删除 Docker Compose 定义的服务。

```bash
docker-compose down      # 停止并删除服务
docker-compose down -v   # 同时删除卷
```

### 6.3 `docker-compose ps`

列出 Docker Compose 定义的服务状态。

```bash
docker-compose ps
```

### 6.4 `docker-compose logs`

查看 Docker Compose 服务的日志。

```bash
docker-compose logs
docker-compose logs -f service_name  # 实时查看指定服务的日志
```

## 7. Docker 系统清理命令

### 7.1 `docker system prune`

清理未使用的 Docker 对象（容器、镜像、网络和卷）。

```bash
docker system prune       # 清理未使用的对象
docker system prune -a    # 清理所有未使用的对象（包括未标记的镜像）
```

### 7.2 `docker container prune`

清理所有已停止的容器。

```bash
docker container prune
```

### 7.3 `docker image prune`

清理未使用的镜像。

```bash
docker image prune        # 清理悬空镜像
docker image prune -a     # 清理所有未使用的镜像
```

## 8. 总结

掌握 Docker 的基础命令是使用和管理容器化应用的基础。通过熟悉这些命令，您可以更高效地进行镜像管理、容器操作、网络配置和数据卷管理。Docker 的命令行工具提供了丰富的功能，可以满足从开发到部署的各种需求。

## 参考资料

- [Docker 官方文档](https://docs.docker.com/)
- [Docker 命令行参考](https://docs.docker.com/engine/reference/commandline/cli/)

---

希望这篇文章能帮助您更好地理解 Docker 的基础命令。如果您有任何问题，欢迎在评论区讨论！
---