---
title: 【Docker】部署RocketMQ完全指南
tags:
  - Docker
  - RocketMQ
  - 消息队列
  - 容器化
  - 分布式系统
categories:
  - Linux
  - Docker
abbrlink: d55fa599
date: 2025-02-28 17:00:00
---

## 问题背景

RocketMQ 是阿里巴巴开源的分布式消息中间件，具有高吞吐量、低延迟、高可靠性等特点，广泛应用于大规模分布式系统中的消息传递和处理。传统部署 RocketMQ 需要处理复杂的依赖关系和配置，而使用 Docker 可以大幅简化这一过程。本文将详细介绍如何使用 Docker 部署 RocketMQ，包括单节点部署和集群部署方案。

## 1. 环境准备
### 1.1 系统要求

- Docker Engine 19.03+
- Docker Compose 1.25+（可选，用于多容器编排）
- 至少 4GB 可用内存（生产环境建议 8GB+）
- 足够的磁盘空间（建议 20GB+）

### 1.2 RocketMQ 组件介绍

RocketMQ 主要包含以下几个组件：

- **NameServer**：注册中心，存储 Broker 的路由信息
- **Broker**：消息存储和转发服务器
- **Producer**：消息生产者，负责发送消息
- **Consumer**：消息消费者，负责接收和处理消息
- **Dashboard**：可视化管理控制台（可选）

## 2. 单节点部署
### 2.1 使用 Docker 命令部署 RocketMQ

```bash
# 创建数据目录
mkdir -p /data/rocketmq/namesrv/logs
mkdir -p /data/rocketmq/namesrv/store
mkdir -p /data/rocketmq/broker/logs
mkdir -p /data/rocketmq/broker/store
mkdir -p /data/rocketmq/conf

# 创建配置文件
cat > /data/rocketmq/conf/broker.conf << EOF
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
# 设置为当前主机IP，不要使用localhost或127.0.0.1
brokerIP1 = 宿主机IP地址
EOF

# 运行 NameServer
docker run -d \
  --name rmqnamesrv \
  -p 9876:9876 \
  -v /data/rocketmq/namesrv/logs:/root/logs \
  -v /data/rocketmq/namesrv/store:/root/store \
  -e "MAX_HEAP_SIZE=256M" \
  -e "JAVA_OPT_EXT=-Xms128m -Xmx128m -Xmn64m" \
  apache/rocketmq:5.1.0 \
  sh mqnamesrv

# 运行 Broker
docker run -d \
  --name rmqbroker \
  -p 10909:10909 \
  -p 10911:10911 \
  -p 10912:10912 \
  -v /data/rocketmq/broker/logs:/root/logs \
  -v /data/rocketmq/broker/store:/root/store \
  -v /data/rocketmq/conf/broker.conf:/opt/rocketmq/conf/broker.conf \
  -e "NAMESRV_ADDR=rmqnamesrv:9876" \
  -e "MAX_HEAP_SIZE=512M" \
  -e "JAVA_OPT_EXT=-Xms256m -Xmx256m -Xmn128m" \
  --link rmqnamesrv:rmqnamesrv \
  apache/rocketmq:5.1.0 \
  sh mqbroker -c /opt/rocketmq/conf/broker.conf
```

参数说明：
- `-p 9876:9876`：NameServer 端口
- `-p 10909:10909`、`-p 10911:10911`、`-p 10912:10912`：Broker 端口
- `-v /data/rocketmq/namesrv/logs:/root/logs`：日志持久化
- `-v /data/rocketmq/namesrv/store:/root/store`：数据持久化
- `-e "MAX_HEAP_SIZE=256M"`：设置最大堆内存
- `-e "JAVA_OPT_EXT=-Xms128m -Xmx128m -Xmn64m"`：JVM 参数

### 2.2 验证部署

```bash
# 检查容器是否正常运行
docker ps | grep rocketmq

# 查看 NameServer 日志
docker logs rmqnamesrv

# 查看 Broker 日志
docker logs rmqbroker
```

## 3. 使用 Docker Compose 部署
### 3.1 创建 docker-compose.yml

```yaml
version: '3'

services:
  namesrv:
    image: apache/rocketmq:5.1.0
    container_name: rmqnamesrv
    ports:
      - 9876:9876
    volumes:
      - ./data/namesrv/logs:/root/logs
      - ./data/namesrv/store:/root/store
    environment:
      - MAX_HEAP_SIZE=256M
      - JAVA_OPT_EXT=-Xms128m -Xmx128m -Xmn64m
    command: sh mqnamesrv

  broker:
    image: apache/rocketmq:5.1.0
    container_name: rmqbroker
    ports:
      - 10909:10909
      - 10911:10911
      - 10912:10912
    volumes:
      - ./data/broker/logs:/root/logs
      - ./data/broker/store:/root/store
      - ./conf/broker.conf:/opt/rocketmq/conf/broker.conf
    environment:
      - NAMESRV_ADDR=namesrv:9876
      - MAX_HEAP_SIZE=512M
      - JAVA_OPT_EXT=-Xms256m -Xmx256m -Xmn128m
    command: sh mqbroker -c /opt/rocketmq/conf/broker.conf
    depends_on:
      - namesrv

  dashboard:
    image: apacherocketmq/rocketmq-dashboard:1.0.0
    container_name: rmqdashboard
    ports:
      - 8080:8080
    environment:
      - JAVA_OPTS=-Drocketmq.namesrv.addr=namesrv:9876
    depends_on:
      - namesrv
```

### 3.2 创建 broker.conf 配置文件

```bash
mkdir -p conf
cat > conf/broker.conf << EOF
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
# 设置为当前主机IP，不要使用localhost或127.0.0.1
brokerIP1 = 宿主机IP地址
EOF
```

### 3.3 启动服务

```bash
docker-compose up -d
```

### 3.4 访问控制台

在浏览器中打开 `http://localhost:8080`，即可访问 RocketMQ Dashboard。

## 4. RocketMQ 集群部署
### 4.1 创建集群的 docker-compose.yml

```yaml
version: '3'

services:
  namesrv1:
    image: apache/rocketmq:5.1.0
    container_name: rmqnamesrv1
    ports:
      - 9876:9876
    volumes:
      - ./data/namesrv1/logs:/root/logs
      - ./data/namesrv1/store:/root/store
    environment:
      - MAX_HEAP_SIZE=256M
      - JAVA_OPT_EXT=-Xms128m -Xmx128m -Xmn64m
    command: sh mqnamesrv

  namesrv2:
    image: apache/rocketmq:5.1.0
    container_name: rmqnamesrv2
    ports:
      - 9877:9876
    volumes:
      - ./data/namesrv2/logs:/root/logs
      - ./data/namesrv2/store:/root/store
    environment:
      - MAX_HEAP_SIZE=256M
      - JAVA_OPT_EXT=-Xms128m -Xmx128m -Xmn64m
    command: sh mqnamesrv

  broker-a-master:
    image: apache/rocketmq:5.1.0
    container_name: rmqbroker-a-master
    ports:
      - 10909:10909
      - 10911:10911
      - 10912:10912
    volumes:
      - ./data/broker-a-master/logs:/root/logs
      - ./data/broker-a-master/store:/root/store
      - ./conf/broker-a-master.conf:/opt/rocketmq/conf/broker.conf
    environment:
      - NAMESRV_ADDR=namesrv1:9876;namesrv2:9876
      - MAX_HEAP_SIZE=512M
      - JAVA_OPT_EXT=-Xms256m -Xmx256m -Xmn128m
    command: sh mqbroker -c /opt/rocketmq/conf/broker.conf
    depends_on:
      - namesrv1
      - namesrv2

  broker-a-slave:
    image: apache/rocketmq:5.1.0
    container_name: rmqbroker-a-slave
    ports:
      - 10919:10909
      - 10921:10911
      - 10922:10912
    volumes:
      - ./data/broker-a-slave/logs:/root/logs
      - ./data/broker-a-slave/store:/root/store
      - ./conf/broker-a-slave.conf:/opt/rocketmq/conf/broker.conf
    environment:
      - NAMESRV_ADDR=namesrv1:9876;namesrv2:9876
      - MAX_HEAP_SIZE=512M
      - JAVA_OPT_EXT=-Xms256m -Xmx256m -Xmn128m
    command: sh mqbroker -c /opt/rocketmq/conf/broker.conf
    depends_on:
      - namesrv1
      - namesrv2
      - broker-a-master

  broker-b-master:
    image: apache/rocketmq:5.1.0
    container_name: rmqbroker-b-master
    ports:
      - 10929:10909
      - 10931:10911
      - 10932:10912
    volumes:
      - ./data/broker-b-master/logs:/root/logs
      - ./data/broker-b-master/store:/root/store
      - ./conf/broker-b-master.conf:/opt/rocketmq/conf/broker.conf
    environment:
      - NAMESRV_ADDR=namesrv1:9876;namesrv2:9876
      - MAX_HEAP_SIZE=512M
      - JAVA_OPT_EXT=-Xms256m -Xmx256m -Xmn128m
    command: sh mqbroker -c /opt/rocketmq/conf/broker.conf
    depends_on:
      - namesrv1
      - namesrv2

  broker-b-slave:
    image: apache/rocketmq:5.1.0
    container_name: rmqbroker-b-slave
    ports:
      - 10939:10909
      - 10941:10911
      - 10942:10912
    volumes:
      - ./data/broker-b-slave/logs:/root/logs
      - ./data/broker-b-slave/store:/root/store
      - ./conf/broker-b-slave.conf:/opt/rocketmq/conf/broker.conf
    environment:
      - NAMESRV_ADDR=namesrv1:9876;namesrv2:9876
      - MAX_HEAP_SIZE=512M
      - JAVA_OPT_EXT=-Xms256m -Xmx256m -Xmn128m
    command: sh mqbroker -c /opt/rocketmq/conf/broker.conf
    depends_on:
      - namesrv1
      - namesrv2
      - broker-b-master

  dashboard:
    image: apacherocketmq/rocketmq-dashboard:1.0.0
    container_name: rmqdashboard
    ports:
      - 8080:8080
    environment:
      - JAVA_OPTS=-Drocketmq.namesrv.addr=namesrv1:9876;namesrv2:9876
    depends_on:
      - namesrv1
      - namesrv2
```

### 4.2 创建 Broker 配置文件

```bash
# broker-a-master.conf
cat > conf/broker-a-master.conf << EOF
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
# 设置为当前主机IP，不要使用localhost或127.0.0.1
brokerIP1 = 宿主机IP地址
EOF

# broker-a-slave.conf
cat > conf/broker-a-slave.conf << EOF
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 1
deleteWhen = 04
fileReservedTime = 48
brokerRole = SLAVE
flushDiskType = ASYNC_FLUSH
# 设置为当前主机IP，不要使用localhost或127.0.0.1
brokerIP1 = 宿主机IP地址
# 指定主节点地址
brokerMasterAddr = 宿主机IP地址:10911
EOF

# broker-b-master.conf
cat > conf/broker-b-master.conf << EOF
brokerClusterName = DefaultCluster
brokerName = broker-b
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
# 设置为当前主机IP，不要使用localhost或127.0.0.1
brokerIP1 = 宿主机IP地址
EOF

# broker-b-slave.conf
cat > conf/broker-b-slave.conf << EOF
brokerClusterName = DefaultCluster
brokerName = broker-b
brokerId = 1
deleteWhen = 04
fileReservedTime = 48
brokerRole = SLAVE
flushDiskType = ASYNC_FLUSH
# 设置为当前主机IP，不要使用localhost或127.0.0.1
brokerIP1 = 宿主机IP地址
# 指定主节点地址
brokerMasterAddr = 宿主机IP地址:10931
EOF
```

### 4.3 启动集群

```bash
# 创建必要的数据目录
mkdir -p data/namesrv1/logs data/namesrv1/store \
         data/namesrv2/logs data/namesrv2/store \
         data/broker-a-master/logs data/broker-a-master/store \
         data/broker-a-slave/logs data/broker-a-slave/store \
         data/broker-b-master/logs data/broker-b-master/store \
         data/broker-b-slave/logs data/broker-b-slave/store

# 启动集群
docker-compose up -d
```

### 4.4 验证集群部署

```bash
# 检查所有容器是否正常运行
docker ps | grep rmq

# 查看各节点日志
docker logs rmqnamesrv1
docker logs rmqbroker-a-master
docker logs rmqbroker-a-slave

# 使用Dashboard验证集群状态
# 访问 http://localhost:8080
```

## 5. 常见问题及解决方案

### 5.1 Broker 无法连接到 NameServer

**症状**：Broker 容器启动后立即退出，或日志中显示无法连接到 NameServer。

**解决方案**：
1. 检查网络连接，确保容器间可以通信
2. 检查 `NAMESRV_ADDR` 环境变量是否正确设置
3. 确保 NameServer 已经成功启动

```bash
# 检查网络连接
docker network inspect bridge

# 手动测试连接
docker exec -it rmqbroker ping rmqnamesrv
```

### 5.2 brokerIP1 配置错误

**症状**：Producer 或 Consumer 无法连接到 Broker，或连接后无法正常发送/接收消息。

**解决方案**：
1. 确保 broker.conf 中的 brokerIP1 设置为宿主机的实际 IP 地址，而不是 127.0.0.1 或 localhost
2. 修改配置后需要重启 Broker 容器

```bash
# 获取宿主机IP地址
ifconfig | grep inet

# 更新配置并重启
docker restart rmqbroker
```

### 5.3 内存不足

**症状**：容器频繁重启，日志中出现 OutOfMemoryError。

**解决方案**：
1. 调整 JVM 内存参数
2. 增加宿主机可用内存

```bash
# 调整JVM参数示例
docker run -d \
  --name rmqbroker \
  -e "MAX_HEAP_SIZE=1G" \
  -e "JAVA_OPT_EXT=-Xms512m -Xmx512m -Xmn256m" \
  ... 其他参数 ...
```

## 6. 性能调优

### 6.1 JVM 参数优化

```bash
# 生产环境推荐JVM参数
-Xms4g -Xmx4g -Xmn2g -XX:+UseG1GC -XX:G1HeapRegionSize=16m \
-XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 \
-XX:SoftRefLRUPolicyMSPerMB=0 -XX:SurvivorRatio=8 \
-XX:MaxGCPauseMillis=200
```

### 6.2 系统参数优化

```bash
# 在宿主机上设置以下系统参数
sysctl -w vm.max_map_count=655360
sysctl -w vm.swappiness=10
```

### 6.3 Broker 参数优化

```
# broker.conf 性能优化参数
flushDiskType = ASYNC_FLUSH  # 异步刷盘提高性能
sendMessageThreadPoolNums = 16  # 发送消息线程池大小
pullMessageThreadPoolNums = 16  # 拉取消息线程池大小
processReplyMessageThreadPoolNums = 16  # 处理回复消息线程池大小
```

## 7. 生产环境最佳实践

### 7.1 高可用部署

- 至少部署 2 个 NameServer 节点
- 每个 Broker 至少有一主一从
- 跨机器部署主从节点，避免单点故障
- 使用 Docker Swarm 或 Kubernetes 进行容器编排

### 7.2 监控告警

- 集成 Prometheus 和 Grafana 监控 RocketMQ 指标
- 设置关键指标告警，如消息堆积、磁盘使用率等

```yaml
# Prometheus 配置示例
scrape_configs:
  - job_name: 'rocketmq'
    static_configs:
      - targets: ['rmqdashboard:8080']
```

### 7.3 数据持久化

- 使用高性能存储设备存储消息数据
- 定期备份重要数据
- 考虑使用数据卷或网络存储提高可靠性

```yaml
# 使用命名卷示例
volumes:
  - rocketmq-namesrv-data:/root/store
  - rocketmq-broker-data:/root/store

volumes:
  rocketmq-namesrv-data:
  rocketmq-broker-data:
```

### 7.4 安全配置

- 启用 ACL 访问控制
- 配置 TLS 加密通信
- 限制容器网络访问

```
# broker.conf 安全配置
aclEnable=true
sslEnable=true
```

## 8. 总结

本文详细介绍了如何使用 Docker 部署 RocketMQ，包括单节点部署和集群部署方案。通过容器化技术，我们可以快速搭建高可用的 RocketMQ 消息中间件服务，简化了传统部署的复杂性。在生产环境中，建议根据实际业务需求进行性能调优和安全配置，确保消息服务的稳定性和可靠性。

---

希望这篇文章能帮助您更好地理解和使用Docker部署RocketMQ。如果您有任何问题，欢迎在评论区讨论！