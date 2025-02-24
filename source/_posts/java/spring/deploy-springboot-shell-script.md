---
title: 【Linux】如何编写一个部署 Spring Boot 程序的 Shell 脚本
tags:
  - Java
  - Spring Boot
  - Shell 脚本
  - 部署
categories:
  - Java
  - Spring
abbrlink: b55fa582
date: 2024-02-25 13:00:00
---

## 问题背景

在 Linux 环境中，部署 Spring Boot 应用程序时，通常需要一个 Shell 脚本来简化部署、启动、停止和重启等操作。本文将介绍如何编写一个满足以下特性的 Shell 脚本：

- 关闭终端后，Java 进程不会退出。
- 提供部署、启动、停止、重启三种常见操作。
- 每次部署时，均备份当前的程序包（可通过参数关闭备份）。

## 1. 脚本结构

以下是一个简单的 Shell 脚本示例，包含了上述特性。您可以根据需要进行修改。

### 1.1 创建脚本文件

在您的 Linux 服务器上，创建一个名为 `deploy.sh` 的文件：

```bash
touch deploy.sh
chmod +x deploy.sh
```

### 1.2 编写脚本内容

将以下内容复制到 `deploy.sh` 文件中：

```bash
#!/bin/bash

# 配置参数
APP_NAME="your-spring-boot-app"  # 应用名称
JAR_PATH="/path/to/your/app.jar"   # JAR 文件路径
BACKUP_DIR="/path/to/backup"       # 备份目录
PID_FILE="/tmp/${APP_NAME}.pid"    # PID 文件路径

# 函数：备份当前程序包
backup() {
    if [ ! -d "$BACKUP_DIR" ]; then
        mkdir -p "$BACKUP_DIR"
    fi
    cp "$JAR_PATH" "$BACKUP_DIR/${APP_NAME}-$(date +%Y%m%d%H%M%S).jar"
    echo "备份完成: $BACKUP_DIR"
}

# 函数：启动应用
start() {
    if [ -f "$PID_FILE" ]; then
        echo "$APP_NAME 已在运行 (PID: $(cat $PID_FILE))"
        return
    fi
    nohup java -jar "$JAR_PATH" > /dev/null 2>&1 &
    echo $! > "$PID_FILE"
    echo "$APP_NAME 启动成功 (PID: $(cat $PID_FILE))"
}

# 函数：停止应用
stop() {
    if [ ! -f "$PID_FILE" ]; then
        echo "$APP_NAME 未在运行"
        return
    fi
    kill $(cat "$PID_FILE")
    rm "$PID_FILE"
    echo "$APP_NAME 停止成功"
}

# 函数：重启应用
restart() {
    stop
    start
}

# 函数：显示帮助信息
help() {
    echo "用法: $0 [deploy|start|stop|restart] [--no-backup]"
    echo "  deploy: 部署应用并备份当前程序包"
    echo "  start: 启动应用"
    echo "  stop: 停止应用"
    echo "  restart: 重启应用"
    echo "  --no-backup: 部署时不备份当前程序包"
}

# 解析参数
case "$1" in
    deploy)
        if [[ "$2" != "--no-backup" ]]; then
            backup
        fi
        start
        ;;
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        restart
        ;;
    *)
        help
        ;;
esac
```

## 2. 脚本功能说明

- **备份功能**：在部署时，脚本会将当前的 JAR 文件备份到指定的备份目录。可以通过 `--no-backup` 参数来关闭备份功能。
- **启动功能**：使用 `nohup` 命令启动 Java 进程，确保即使关闭终端，进程仍然在后台运行。进程的 PID 会被写入到指定的 PID 文件中。
- **停止功能**：通过读取 PID 文件来停止正在运行的 Java 进程，并删除 PID 文件。
- **重启功能**：先停止应用，再启动应用。
- **帮助信息**：提供使用说明，帮助用户了解如何使用脚本。

## 3. 使用示例

- 部署并备份当前程序包：

```bash
./deploy.sh deploy
```

- 部署但不备份当前程序包：

```bash
./deploy.sh deploy --no-backup
```

- 启动应用：

```bash
./deploy.sh start
```

- 停止应用：

```bash
./deploy.sh stop
```

- 重启应用：

```bash
./deploy.sh restart
```

## 总结

通过编写一个简单的 Shell 脚本，您可以轻松地在 Linux 环境中部署和管理 Spring Boot 应用程序。该脚本提供了优雅的启动、停止和重启功能，并支持备份当前程序包，帮助您更好地管理应用的生命周期。

## 参考资料

- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
- [Bash Scripting Guide](https://tldp.org/LDP/Bash-Beginners-Guide/html/)

---

希望这篇文章能帮助您更好地理解和实现 Linux 环境下的 Spring Boot 应用部署。如果您有任何问题，欢迎在评论区讨论！
--- 
 