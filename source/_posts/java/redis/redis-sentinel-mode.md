---
title: 【Redis】哨兵模式详解与实践
tags:
  - Redis
  - 哨兵模式
  - 高可用
  - 数据库
  - 分布式系统
categories:
  - 数据库
  - Redis
abbrlink: c55fa584
date: 2025-03-01 14:00:00
---

## 问题背景

Redis 是一个高性能的内存数据库，在生产环境中通常需要保证其高可用性。当使用主从复制时，如果主节点发生故障，需要手动将从节点提升为主节点，这个过程繁琐且容易出错。Redis 哨兵（Sentinel）模式提供了自动故障检测和故障转移功能，能够在主节点故障时自动选举新的主节点，保证系统的高可用性。本文将详细介绍 Redis 哨兵模式的工作原理、配置方法及最佳实践。

## 1. Redis 哨兵模式基本概念

Redis 哨兵是 Redis 高可用性解决方案的核心组件，它是一个独立运行的进程，负责监控 Redis 主从实例的健康状态，并在主节点故障时自动进行故障转移。哨兵模式的主要特点包括：

- **自动故障检测**：哨兵会定期检查主节点和从节点的健康状态。
- **自动故障转移**：当主节点不可用时，哨兵会自动选举一个从节点作为新的主节点。
- **客户端通知**：哨兵会通知客户端新主节点的地址，使客户端能够连接到新的主节点。
- **配置提供者**：哨兵可以为客户端提供服务发现功能，帮助客户端找到当前的主节点。

## 2. 哨兵模式的工作原理

### 2.1 监控（Monitoring）

哨兵会定期向主节点和从节点发送 PING 命令，检查它们是否正常运行。如果在指定的时间内没有收到有效回复，哨兵会将该实例标记为「主观下线」（Subjectively Down，简称 SDOWN）。

### 2.2 通知（Notification）

当哨兵发现主节点或从节点处于「主观下线」状态时，会通过发布/订阅机制通知其他哨兵。如果多个哨兵都认为主节点已经下线，那么主节点会被标记为「客观下线」（Objectively Down，简称 ODOWN）。

### 2.3 故障转移（Failover）

当主节点被标记为「客观下线」后，哨兵会进行以下操作：

1. **选举领导者**：哨兵集群会选举一个领导者来执行故障转移操作。
2. **选择新的主节点**：领导者会从从节点中选择一个最适合的节点作为新的主节点。
3. **配置从节点**：将其他从节点重新配置为新主节点的从节点。
4. **通知客户端**：通知客户端新主节点的地址。

## 3. 哨兵模式的配置

### 3.1 环境准备

在配置哨兵模式之前，您需要准备以下环境：

- 一个 Redis 主节点
- 至少一个 Redis 从节点（推荐至少两个）
- 至少三个哨兵实例（推荐奇数个）

### 3.2 配置主从复制

首先，需要配置 Redis 的主从复制。在从节点的配置文件中添加以下配置：

```conf
# 配置主节点的地址和端口
replicaof 192.168.1.100 6379

# 如果主节点设置了密码，需要配置密码
masterauth your_master_password
```

### 3.3 配置哨兵

创建一个 `sentinel.conf` 配置文件，添加以下配置：

```conf
# 监听端口
port 26379

# 指定工作目录
dir /tmp

# 监控的主节点
# sentinel monitor <master-name> <ip> <port> <quorum>
sentinel monitor mymaster 192.168.1.100 6379 2

# 主节点或从节点多久无响应视为下线（毫秒）
sentinel down-after-milliseconds mymaster 30000

# 故障转移超时时间（毫秒）
sentinel failover-timeout mymaster 180000

# 同时进行故障转移的从节点数量
sentinel parallel-syncs mymaster 1

# 如果主节点有密码，需要配置密码
sentinel auth-pass mymaster your_master_password
```

配置说明：

- `sentinel monitor mymaster 192.168.1.100 6379 2`：监控名为 mymaster 的主节点，IP 为 192.168.1.100，端口为 6379，当至少有 2 个哨兵认为主节点不可用时，才会进行故障转移。
- `sentinel down-after-milliseconds mymaster 30000`：如果主节点在 30 秒内没有响应，则认为主节点已下线。
- `sentinel failover-timeout mymaster 180000`：故障转移的超时时间为 180 秒。
- `sentinel parallel-syncs mymaster 1`：在故障转移期间，每次只能有 1 个从节点进行同步。

### 3.4 启动哨兵

使用以下命令启动哨兵：

```bash
redis-sentinel /path/to/sentinel.conf
```

或者：

```bash
redis-server /path/to/sentinel.conf --sentinel
```

## 4. 哨兵模式的实际部署

### 4.1 Docker 环境下的部署

在 Docker 环境中部署 Redis 哨兵模式，可以使用 Docker Compose 来简化配置。以下是一个示例的 `docker-compose.yml` 文件：

```yaml
version: '3'

services:
  redis-master:
    image: redis:latest
    container_name: redis-master
    ports:
      - "6379:6379"
    volumes:
      - ./redis-master.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-slave-1:
    image: redis:latest
    container_name: redis-slave-1
    ports:
      - "6380:6379"
    volumes:
      - ./redis-slave-1.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    depends_on:
      - redis-master

  redis-slave-2:
    image: redis:latest
    container_name: redis-slave-2
    ports:
      - "6381:6379"
    volumes:
      - ./redis-slave-2.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    depends_on:
      - redis-master

  redis-sentinel-1:
    image: redis:latest
    container_name: redis-sentinel-1
    ports:
      - "26379:26379"
    volumes:
      - ./sentinel-1.conf:/usr/local/etc/redis/sentinel.conf
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    depends_on:
      - redis-master
      - redis-slave-1
      - redis-slave-2

  redis-sentinel-2:
    image: redis:latest
    container_name: redis-sentinel-2
    ports:
      - "26380:26379"
    volumes:
      - ./sentinel-2.conf:/usr/local/etc/redis/sentinel.conf
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    depends_on:
      - redis-master
      - redis-slave-1
      - redis-slave-2

  redis-sentinel-3:
    image: redis:latest
    container_name: redis-sentinel-3
    ports:
      - "26381:26379"
    volumes:
      - ./sentinel-3.conf:/usr/local/etc/redis/sentinel.conf
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    depends_on:
      - redis-master
      - redis-slave-1
      - redis-slave-2
```

### 4.2 客户端连接

在 Java 应用中使用 Jedis 连接 Redis 哨兵模式的示例代码：

```java
import redis.clients.jedis.JedisSentinelPool;
import redis.clients.jedis.Jedis;
import java.util.HashSet;
import java.util.Set;

public class RedisSentinelExample {
    public static void main(String[] args) {
        // 配置哨兵信息
        Set<String> sentinels = new HashSet<>();
        sentinels.add("192.168.1.100:26379");
        sentinels.add("192.168.1.101:26379");
        sentinels.add("192.168.1.102:26379");
        
        // 创建哨兵连接池
        JedisSentinelPool pool = new JedisSentinelPool("mymaster", sentinels);
        
        // 获取 Jedis 实例
        try (Jedis jedis = pool.getResource()) {
            // 执行操作
            jedis.set("key", "value");
            String value = jedis.get("key");
            System.out.println(value);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 关闭连接池
            pool.close();
        }
    }
}
```

## 5. 哨兵模式的最佳实践

### 5.1 哨兵数量

建议至少部署 3 个哨兵实例，并且使用奇数个哨兵，以便在选举过程中避免平票情况。哨兵实例应该部署在不同的物理机器上，以提高可用性。

### 5.2 配置优化

- **合理设置 down-after-milliseconds**：根据网络环境和业务需求，设置合适的主观下线时间。
- **调整 parallel-syncs**：如果从节点较多，可以适当增加 parallel-syncs 的值，加快故障转移速度。
- **配置 notification-script**：可以配置通知脚本，在故障转移时发送通知。

```conf
sentinel notification-script mymaster /path/to/notification.sh
```

### 5.3 监控与维护

- 定期检查哨兵日志，了解哨兵的运行状态。
- 使用 Redis 的 INFO 命令监控主从复制的状态。
- 定期进行故障演练，确保哨兵能够正常进行故障转移。

## 6. 哨兵模式与集群模式的比较

| 特性 | 哨兵模式 | 集群模式 |
| --- | --- | --- |
| 数据分片 | 不支持 | 支持 |
| 自动故障转移 | 支持 | 支持 |
| 扩展性 | 有限 | 高 |
| 配置复杂度 | 中等 | 高 |
| 适用场景 | 数据量较小，主要需要高可用性 | 数据量大，需要水平扩展 |

## 参考资料

- [Redis Sentinel Documentation](https://redis.io/topics/sentinel)
- [Redis 官方文档](https://redis.io/documentation)
- [Redis 高可用解决方案](https://redis.io/topics/high-availability)

---

希望这篇文章能帮助您更好地理解和配置 Redis 哨兵模式。如果您有任何问题，欢迎在评论区讨论！