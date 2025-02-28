---
title: 【Docker】部署RabbitMQ及配置延时队列完全指南
tags:
  - Docker
  - RabbitMQ
  - 消息队列
  - 容器化
  - 延时队列
categories:
  - Linux
  - Docker
abbrlink: d55fa598
date: 2025-02-28 15:00:00
---

## 问题背景

RabbitMQ 是一个开源的消息代理软件，它实现了高级消息队列协议(AMQP)，提供可靠的消息传递机制。在微服务架构和分布式系统中，RabbitMQ 扮演着重要角色。传统部署 RabbitMQ 需要处理复杂的依赖关系和配置，而使用 Docker 可以大幅简化这一过程。本文将详细介绍如何使用 Docker 部署 RabbitMQ，以及如何配置和使用延时队列功能。

## 1. 环境准备

### 1.1 系统要求

- Docker Engine 19.03+
- Docker Compose 1.25+（可选，用于多容器编排）
- 至少 2GB 可用内存（生产环境建议 4GB+）
- 足够的磁盘空间（建议 10GB+）

### 1.2 镜像选择

RabbitMQ 官方提供了多种 Docker 镜像，最常用的有：

- `rabbitmq:3.12-management`：包含管理界面的 RabbitMQ 3.12 版本
- `rabbitmq:3.12`：不包含管理界面的基础版本

本文主要使用带管理界面的版本，便于可视化操作和监控。

## 2. 单节点部署

### 2.1 使用 Docker 命令部署 RabbitMQ

```bash
# 创建数据和日志目录
mkdir -p /data/rabbitmq/data
mkdir -p /data/rabbitmq/log

# 运行 RabbitMQ 容器
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin123 \
  -v /data/rabbitmq/data:/var/lib/rabbitmq \
  -v /data/rabbitmq/log:/var/log/rabbitmq \
  --hostname my-rabbit \
  rabbitmq:3.12-management
```

参数说明：
- `-p 5672:5672`：AMQP 协议端口
- `-p 15672:15672`：管理界面端口
- `-e RABBITMQ_DEFAULT_USER=admin`：设置默认用户名
- `-e RABBITMQ_DEFAULT_PASS=admin123`：设置默认密码
- `-v /data/rabbitmq/data:/var/lib/rabbitmq`：数据持久化
- `-v /data/rabbitmq/log:/var/log/rabbitmq`：日志持久化
- `--hostname my-rabbit`：设置容器主机名（对集群很重要）

### 2.2 验证部署

```bash
# 检查容器是否正常运行
docker ps | grep rabbitmq

# 查看日志
docker logs rabbitmq
```

访问管理界面：在浏览器中打开 `http://localhost:15672`，使用设置的用户名和密码登录。

## 3. 使用 Docker Compose 部署

### 3.1 创建 docker-compose.yml 文件

```yaml
version: '3'

services:
  rabbitmq:
    image: rabbitmq:3.12-management
    container_name: rabbitmq
    hostname: my-rabbit
    ports:
      - "5672:5672"   # AMQP 端口
      - "15672:15672" # 管理界面端口
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    volumes:
      - ./data:/var/lib/rabbitmq
      - ./logs:/var/log/rabbitmq
    restart: always
```

### 3.2 启动服务

```bash
docker-compose up -d
```

## 4. RabbitMQ 集群部署

### 4.1 创建集群的 docker-compose.yml

```yaml
version: '3'

services:
  rabbitmq1:
    image: rabbitmq:3.12-management
    hostname: rabbitmq1
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    volumes:
      - ./rabbitmq1/data:/var/lib/rabbitmq
      - ./rabbitmq1/log:/var/log/rabbitmq

  rabbitmq2:
    image: rabbitmq:3.12-management
    hostname: rabbitmq2
    ports:
      - "5673:5672"
      - "15673:15672"
    environment:
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    volumes:
      - ./rabbitmq2/data:/var/lib/rabbitmq
      - ./rabbitmq2/log:/var/log/rabbitmq
    depends_on:
      - rabbitmq1

  rabbitmq3:
    image: rabbitmq:3.12-management
    hostname: rabbitmq3
    ports:
      - "5674:5672"
      - "15674:15672"
    environment:
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    volumes:
      - ./rabbitmq3/data:/var/lib/rabbitmq
      - ./rabbitmq3/log:/var/log/rabbitmq
    depends_on:
      - rabbitmq1
```

### 4.2 配置集群

启动容器后，需要将节点加入集群：

```bash
# 进入第二个节点
docker exec -it rabbitmq2 bash

# 停止应用
rabbitmqctl stop_app

# 重置节点
rabbitmqctl reset

# 加入集群
rabbitmqctl join_cluster rabbit@rabbitmq1

# 启动应用
rabbitmqctl start_app

# 退出容器
exit

# 对第三个节点重复上述操作
docker exec -it rabbitmq3 bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@rabbitmq1
rabbitmqctl start_app
exit
```

## 5. 延时队列配置

### 5.1 延时队列概念

延时队列是指消息在发送到队列后，不会立即被消费，而是在指定的时间后才能被消费。RabbitMQ 本身并没有直接提供延时队列功能，但可以通过以下两种方式实现：

1. 使用 Dead Letter Exchange (DLX) 和消息 TTL
2. 使用 rabbitmq_delayed_message_exchange 插件

### 5.2 使用 Dead Letter Exchange 实现延时队列

#### 5.2.1 原理

当消息在队列中的存活时间超过设定的 TTL（Time To Live）值时，或者队列长度超过最大长度时，消息会被发送到与队列绑定的死信交换机（Dead Letter Exchange），然后路由到另一个队列，这个队列就是我们真正消费消息的队列。

#### 5.2.2 配置步骤

1. 进入 RabbitMQ 容器：

```bash
docker exec -it rabbitmq bash
```

2. 使用 rabbitmqadmin 创建所需的交换机和队列：

```bash
# 创建死信交换机
rabbitmqadmin declare exchange name=dlx_exchange type=direct

# 创建实际消费队列
rabbitmqadmin declare queue name=actual_queue

# 将实际消费队列绑定到死信交换机
rabbitmqadmin declare binding source=dlx_exchange destination=actual_queue routing_key=dlx_routing_key

# 创建延时队列，设置消息 TTL 和死信交换机
rabbitmqadmin declare queue name=delay_queue arguments='{"x-dead-letter-exchange":"dlx_exchange","x-dead-letter-routing-key":"dlx_routing_key","x-message-ttl":10000}'

# 创建普通交换机
rabbitmqadmin declare exchange name=normal_exchange type=direct

# 将延时队列绑定到普通交换机
rabbitmqadmin declare binding source=normal_exchange destination=delay_queue routing_key=delay_routing_key
```

上述配置创建了一个延时为 10 秒（10000 毫秒）的队列。

### 5.3 使用 rabbitmq_delayed_message_exchange 插件

#### 5.3.1 安装插件

```bash
# 进入容器
docker exec -it rabbitmq bash

# 启用插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# 退出容器
exit

# 重启容器使插件生效
docker restart rabbitmq
```

#### 5.3.2 配置延时交换机

通过管理界面或命令行创建延时交换机：

```bash
# 进入容器
docker exec -it rabbitmq bash

# 创建延时交换机
rabbitmqadmin declare exchange name=delayed_exchange type=x-delayed-message arguments='{"x-delayed-type":"direct"}'

# 创建队列
rabbitmqadmin declare queue name=delayed_queue

# 绑定队列到延时交换机
rabbitmqadmin declare binding source=delayed_exchange destination=delayed_queue routing_key=delayed_routing_key
```

## 6. 延时队列的使用示例

### 6.1 使用 Dead Letter Exchange 方式

#### Java 代码示例

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class DelayQueueProducer {
    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin123");
        
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {
            
            String message = "Hello, delayed message!";
            
            // 发送消息到普通交换机，路由到延时队列
            channel.basicPublish("normal_exchange", "delay_routing_key", null, message.getBytes());
            System.out.println("Sent message: '" + message + "', will be delivered in 10 seconds");
        }
    }
}
```

#### 消费者代码

```java
import com.rabbitmq.client.*;

public class DelayQueueConsumer {
    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin123");
        
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        
        // 消费实际队列中的消息
        channel.basicConsume("actual_queue", true, (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println("Received delayed message: '" + message + "'");
        }, consumerTag -> {});
        
        System.out.println("Waiting for delayed messages...");
    }
}
```

### 6.2 使用 rabbitmq_delayed_message_exchange 插件

#### Java 代码示例

```java
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.util.HashMap;
import java.util.Map;

public class DelayedPluginProducer {
    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin123");