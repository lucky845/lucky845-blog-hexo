---
title: 【Redis】过期删除与内存淘汰策略详解
tags:
  - Redis
  - 缓存
  - 内存管理
  - 数据库
categories:
  - 数据库
  - Redis
abbrlink: b55fa587
date: 2025-03-01 16:00:00
---

## 问题背景

Redis 作为一个内存数据库，其数据都存储在内存中，这就带来了两个关键问题：一是如何处理设置了过期时间的键，二是当内存不足时如何进行数据淘汰。本文将详细介绍 Redis 的过期删除策略和内存淘汰机制，帮助您更好地管理 Redis 内存资源。

## 1. Redis 的过期删除策略

Redis 中可以通过 `EXPIRE`、`EXPIREAT`、`PEXPIRE`、`PEXPIREAT` 等命令为键设置过期时间。当键过期后，Redis 需要将其从内存中删除。Redis 采用了三种过期删除策略的组合来处理过期键。

### 1.1 定时删除（Time Based Deletion）

定时删除是指在设置键的过期时间时，创建一个定时器，当时间到达时，立即删除该键。

#### 优点

- **内存友好**：过期键会被立即删除，释放内存空间。

#### 缺点

- **CPU 开销大**：需要维护大量的定时器，对 CPU 资源消耗较大。
- **实现复杂**：需要一个独立的定时器，且难以高效实现。

由于上述缺点，Redis 实际上并未完全采用定时删除策略。

### 1.2 惰性删除（Lazy Deletion）

惰性删除是指只有当客户端尝试访问一个键时，才会检查该键是否过期，如果过期则删除。

#### 优点

- **CPU 友好**：只在必要时才检查和删除过期键，减少了 CPU 开销。

#### 缺点

- **内存不友好**：如果过期键长时间未被访问，会一直占用内存空间。

#### 实现方式

在 Redis 中，惰性删除策略通过在读写操作前检查键的过期时间来实现：

```c
// Redis 源码中的伪代码
if (key.expired()) {
    delete(key);
    return null;
}
return key.value;
```

### 1.3 定期删除（Periodic Deletion）

定期删除是指 Redis 定期随机抽样一部分键，检查其是否过期，如果过期则删除。

#### 优点

- **平衡 CPU 和内存**：通过控制删除操作的频率和执行时间，在 CPU 和内存之间取得平衡。

#### 缺点

- **不能保证及时删除所有过期键**：由于是随机抽样，可能会有部分过期键未被检查到。

#### 实现方式

Redis 的定期删除策略在 `serverCron()` 函数中实现，默认每 100ms 执行一次，每次最多删除 20 个过期键。

```conf
# redis.conf 中的相关配置
hz 10  # 默认值为 10，表示每秒执行 10 次 serverCron()
```

## 2. Redis 的内存淘汰策略

当 Redis 的内存使用达到上限时，需要通过内存淘汰策略来释放空间。Redis 提供了多种内存淘汰策略，可以通过 `maxmemory-policy` 配置项来设置。

### 2.1 不淘汰（noeviction）

当内存不足时，新写入操作会报错，不会删除任何现有数据。

```conf
maxmemory-policy noeviction
```

适用场景：对数据完整性要求极高，不允许丢失任何数据的场景。

### 2.2 淘汰所有键中的最近最少使用（allkeys-lru）

当内存不足时，从所有键中选择最近最少使用的键进行淘汰。

```conf
maxmemory-policy allkeys-lru
```

适用场景：缓存场景，希望保留热点数据。

### 2.3 淘汰所有键中的最少使用（allkeys-lfu）

当内存不足时，从所有键中选择使用频率最低的键进行淘汰。

```conf
maxmemory-policy allkeys-lfu
```

适用场景：访问频率差异明显的缓存场景。

### 2.4 淘汰所有键中的随机键（allkeys-random）

当内存不足时，随机选择键进行淘汰。

```conf
maxmemory-policy allkeys-random
```

适用场景：所有键访问概率相同的场景。

### 2.5 淘汰设置了过期时间的键中的最近最少使用（volatile-lru）

当内存不足时，从设置了过期时间的键中选择最近最少使用的键进行淘汰。

```conf
maxmemory-policy volatile-lru
```

适用场景：希望只淘汰设置了过期时间的键，且希望保留热点数据。

### 2.6 淘汰设置了过期时间的键中的最少使用（volatile-lfu）

当内存不足时，从设置了过期时间的键中选择使用频率最低的键进行淘汰。

```conf
maxmemory-policy volatile-lfu
```

适用场景：希望只淘汰设置了过期时间的键，且访问频率差异明显。

### 2.7 淘汰设置了过期时间的键中的随机键（volatile-random）

当内存不足时，从设置了过期时间的键中随机选择键进行淘汰。

```conf
maxmemory-policy volatile-random
```

适用场景：希望只淘汰设置了过期时间的键，且所有键访问概率相同。

### 2.8 淘汰快要过期的键（volatile-ttl）

当内存不足时，从设置了过期时间的键中选择剩余生存时间（TTL）最短的键进行淘汰。

```conf
maxmemory-policy volatile-ttl
```

适用场景：希望优先保留生存时间较长的键。

## 3. 内存管理最佳实践

### 3.1 设置合理的内存上限

根据服务器内存情况，为 Redis 设置合理的内存上限，避免 Redis 占用过多系统资源。

```conf
# 设置 Redis 最大内存为 2GB
maxmemory 2gb
```

### 3.2 选择合适的内存淘汰策略

根据业务需求选择合适的内存淘汰策略：

- 对于纯缓存场景，推荐使用 `allkeys-lru` 或 `allkeys-lfu`。
- 对于既有缓存又有持久化数据的场景，推荐使用 `volatile-lru` 或 `volatile-lfu`。
- 对于数据完整性要求高的场景，可以使用 `noeviction`，但需要确保有足够的内存。

### 3.3 合理设置键的过期时间

为键设置合理的过期时间，避免无用数据长期占用内存：

```redis
SET key value EX 3600  # 设置键的过期时间为 1 小时
```

### 3.4 监控内存使用情况

定期监控 Redis 的内存使用情况，及时调整配置：

```bash
# 使用 Redis CLI 查看内存使用情况
redis-cli info memory
```

### 3.5 使用 Redis 内存分析工具

使用 Redis 提供的内存分析工具，如 `MEMORY USAGE` 命令和 `redis-memory-analyzer`，分析内存使用情况：

```bash
# 查看某个键的内存使用情况
redis-cli memory usage key
```

## 4. 总结

Redis 通过过期删除策略（惰性删除和定期删除的组合）和内存淘汰策略来管理内存资源。合理配置这些策略，可以在保证数据可用性的同时，有效利用内存资源。在实际应用中，应根据业务需求选择合适的策略，并定期监控内存使用情况，及时调整配置。

## 参考资料

- [Redis 官方文档](https://redis.io/documentation)
- [Redis 内存优化](https://redis.io/topics/memory-optimization)
- [Redis 配置](https://redis.io/topics/config)

---

希望这篇文章能帮助您更好地理解 Redis 的过期删除与内存淘汰机制。如果您有任何问题，欢迎在评论区讨论！