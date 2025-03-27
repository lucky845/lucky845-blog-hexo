---
title: 【Java】乐观锁与悲观锁详解
tags:
  - Java
  - 并发
  - 锁机制
  - 数据库
  - 分布式系统
categories:
  - Java
  - 基础知识
abbrlink: b55fa588
date: 2025-03-27 18:00:00
---

## 问题背景

在并发编程和数据库事务处理中，锁机制是保证数据一致性和完整性的关键技术。根据锁的实现策略，锁机制主要分为乐观锁（Optimistic Lock）和悲观锁（Pessimistic Lock）两种类型。本文将详细介绍这两种锁机制的原理、实现方式、适用场景以及各自的优缺点。

## 1. 锁的基本概念

在多线程或分布式环境下，多个操作可能会同时访问和修改同一资源，这可能导致数据不一致或资源竞争的问题。锁机制通过限制对共享资源的并发访问，确保在同一时间只有一个操作能够修改资源，从而保证数据的一致性。

## 2. 悲观锁（Pessimistic Lock）

### 2.1 基本原理

悲观锁的基本思想是：**假设最坏的情况，认为在数据处理过程中，数据随时会被其他线程或事务修改，因此在整个数据处理过程中需要将数据锁定，防止其他线程或事务进行修改。**

悲观锁的工作流程如下：

1. 当事务或线程需要操作数据时，首先尝试获取锁。
2. 如果获取锁成功，则可以对数据进行读取和修改；如果获取锁失败，则等待或放弃操作。
3. 操作完成后，释放锁，允许其他事务或线程获取锁并操作数据。

### 2.2 实现方式

#### 2.2.1 数据库中的悲观锁

在关系型数据库中，悲观锁通常通过以下方式实现：

**1. 排他锁（Exclusive Lock）**

在 MySQL 中，可以使用 `SELECT ... FOR UPDATE` 语句获取排他锁：

```sql
-- 开始事务
BEGIN;
-- 获取排他锁
SELECT * FROM users WHERE id = 1 FOR UPDATE;
-- 更新数据
UPDATE users SET balance = balance - 100 WHERE id = 1;
-- 提交事务
COMMIT;
```

**2. 共享锁（Shared Lock）**

在 MySQL 中，可以使用 `SELECT ... LOCK IN SHARE MODE` 语句获取共享锁：

```sql
-- 开始事务
BEGIN;
-- 获取共享锁
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;
-- 提交事务
COMMIT;
```

#### 2.2.2 Java 中的悲观锁

Java 中的悲观锁主要通过 `synchronized` 关键字和 `ReentrantLock` 类实现：

**1. synchronized 关键字**

```java
public class Account {
    private double balance;

    public synchronized void withdraw(double amount) {
        if (balance >= amount) {
            balance -= amount;
        }
    }

    public synchronized double getBalance() {
        return balance;
    }
}
```

**2. ReentrantLock 类**

```java
import java.util.concurrent.locks.ReentrantLock;

public class Account {
    private double balance;
    private final ReentrantLock lock = new ReentrantLock();

    public void withdraw(double amount) {
        lock.lock();
        try {
            if (balance >= amount) {
                balance -= amount;
            }
        } finally {
            lock.unlock();
        }
    }

    public double getBalance() {
        lock.lock();
        try {
            return balance;
        } finally {
            lock.unlock();
        }
    }
}
```

### 2.3 适用场景

悲观锁适用于以下场景：

1. **写操作频繁**：当系统中写操作比读操作更频繁时，使用悲观锁可以减少冲突。
2. **资源竞争激烈**：当多个线程或事务频繁地同时访问同一资源时，使用悲观锁可以避免频繁的重试和回滚。
3. **安全性要求高**：当系统对数据一致性的要求非常高，不能容忍任何数据不一致的情况时，悲观锁是更安全的选择。

### 2.4 优缺点

**优点：**

1. **简单直观**：悲观锁的实现和使用相对简单，容易理解。
2. **安全性高**：悲观锁可以确保在锁定期间，数据不会被其他线程或事务修改，从而保证数据的一致性。

**缺点：**

1. **性能开销大**：悲观锁会锁定资源，导致其他线程或事务需要等待，降低系统的并发性能。
2. **可能导致死锁**：如果多个线程或事务以不同的顺序获取多个锁，可能会导致死锁。

## 3. 乐观锁（Optimistic Lock）

### 3.1 基本原理

乐观锁的基本思想是：**假设最好的情况，认为数据在大多数情况下不会被其他线程或事务修改，因此不需要在整个数据处理过程中锁定数据，而是在更新数据时检查数据是否被修改过。**

乐观锁的工作流程如下：

1. 当事务或线程需要操作数据时，首先读取数据及其版本号（或时间戳）。
2. 在更新数据时，检查数据的版本号是否与读取时的版本号一致。
3. 如果版本号一致，则更新数据并增加版本号；如果版本号不一致，则说明数据已被其他线程或事务修改，需要重试或放弃操作。

### 3.2 实现方式

#### 3.2.1 数据库中的乐观锁

在关系型数据库中，乐观锁通常通过以下方式实现：

**1. 版本号机制**

在数据表中添加一个版本号字段，每次更新数据时增加版本号：

```sql
-- 开始事务
BEGIN;
-- 读取数据及版本号
SELECT * FROM users WHERE id = 1;
-- 假设读取到的版本号为 1
-- 更新数据，同时检查版本号
UPDATE users SET balance = balance - 100, version = version + 1 WHERE id = 1 AND version = 1;
-- 提交事务
COMMIT;
```

**2. 时间戳机制**

在数据表中添加一个时间戳字段，每次更新数据时更新时间戳：

```sql
-- 开始事务
BEGIN;
-- 读取数据及时间戳
SELECT * FROM users WHERE id = 1;
-- 假设读取到的时间戳为 '2023-01-01 12:00:00'
-- 更新数据，同时检查时间戳
UPDATE users SET balance = balance - 100, update_time = NOW() WHERE id = 1 AND update_time = '2023-01-01 12:00:00';
-- 提交事务
COMMIT;
```

#### 3.2.2 Java 中的乐观锁

Java 中的乐观锁主要通过 CAS（Compare-And-Swap）操作实现：

**1. AtomicInteger 类**

```java
import java.util.concurrent.atomic.AtomicInteger;

public class Counter {
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        int oldValue, newValue;
        do {
            oldValue = count.get();
            newValue = oldValue + 1;
        } while (!count.compareAndSet(oldValue, newValue));
    }

    public int getCount() {
        return count.get();
    }
}
```

**2. 自定义乐观锁**

```java
public class OptimisticAccount {
    private double balance;
    private int version;

    public boolean withdraw(double amount, int expectedVersion) {
        synchronized (this) {
            if (version == expectedVersion) {
                if (balance >= amount) {
                    balance -= amount;
                    version++;
                    return true;
                }
            }
            return false;
        }
    }

    public synchronized AccountInfo getAccountInfo() {
        return new AccountInfo(balance, version);
    }

    public static class AccountInfo {
        private final double balance;
        private final int version;

        public AccountInfo(double balance, int version) {
            this.balance = balance;
            this.version = version;
        }

        public double getBalance() {
            return balance;
        }

        public int getVersion() {
            return version;
        }
    }
}
```

#### 3.2.3 Redis 中的乐观锁

Redis 提供了 `WATCH` 命令来实现乐观锁：

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Transaction;

public class RedisOptimisticLock {
    public static void main(String[] args) {
        Jedis jedis = new Jedis("localhost");

        String key = "account:1";

        // 监视键
        jedis.watch(key);

        // 读取当前余额
        String balanceStr = jedis.get(key);
        double balance = Double.parseDouble(balanceStr);
        double amount = 100.0;

        if (balance >= amount) {
            // 开始事务
            Transaction transaction = jedis.multi();

            // 更新余额
            transaction.set(key, String.valueOf(balance - amount));

            // 执行事务
            if (transaction.exec() == null) {
                System.out.println("事务执行失败，键已被修改");
            } else {
                System.out.println("事务执行成功");
            }
        }

        // 关闭连接
        jedis.close();
    }
}
```

### 3.3 适用场景

乐观锁适用于以下场景：

1. **读操作频繁**：当系统中读操作比写操作更频繁时，使用乐观锁可以提高系统的并发性能。
2. **资源竞争不激烈**：当多个线程或事务同时访问同一资源的概率较低时，使用乐观锁可以减少不必要的锁定。
3. **性能要求高**：当系统对性能的要求高于对数据一致性的要求时，乐观锁是更好的选择。

### 3.4 优缺点

**优点：**

1. **并发性能高**：乐观锁不会锁定资源，允许多个线程或事务同时访问资源，提高系统的并发性能。
2. **不会导致死锁**：乐观锁不会导致死锁，因为它不会锁定资源。

**缺点：**

1. **实现复杂**：乐观锁的实现相对复杂，需要维护版本号或时间戳。
2. **可能导致活锁**：如果多个线程或事务频繁地同时修改同一资源，可能会导致活锁，即所有线程或事务都无法成功更新数据。

## 4. 乐观锁与悲观锁的对比

| 特性       | 乐观锁                     | 悲观锁                   |
| ---------- | -------------------------- | ------------------------ |
| 基本思想   | 假设数据不会被修改         | 假设数据会被修改         |
| 锁定方式   | 不锁定资源，在更新时检查   | 锁定资源，防止其他操作   |
| 并发性能   | 高                         | 低                       |
| 实现复杂度 | 复杂                       | 简单                     |
| 适用场景   | 读操作频繁，资源竞争不激烈 | 写操作频繁，资源竞争激烈 |
| 可能问题   | 活锁                       | 死锁                     |

## 5. 实际应用案例

### 5.1 电商系统中的库存管理

在电商系统中，库存管理是一个典型的并发问题。当多个用户同时下单购买同一商品时，需要确保库存数量的正确性。

**使用悲观锁的实现：**

```java
public class PessimisticInventoryService {
    public boolean reduceStock(Long productId, int quantity) {
        synchronized (this) {
            // 查询库存
            int stock = getStock(productId);

            // 检查库存是否充足
            if (stock >= quantity) {
                // 减少库存
                updateStock(productId, stock - quantity);
                return true;
            }

            return false;
        }
    }

    private int getStock(Long productId) {
        // 从数据库查询库存
        return 0; // 示例代码
    }

    private void updateStock(Long productId, int newStock) {
        // 更新数据库中的库存
    }
}
```

**使用乐观锁的实现：**

```java
public class OptimisticInventoryService {
    public boolean reduceStock(Long productId, int quantity) {
        int retryCount = 0;
        int maxRetryCount = 3;

        while (retryCount < maxRetryCount) {
            // 查询库存及版本号
            InventoryInfo info = getInventoryInfo(productId);
            int stock = info.getStock();
            int version = info.getVersion();

            // 检查库存是否充足
            if (stock >= quantity) {
                // 使用乐观锁更新库存
                boolean success = updateStock(productId, stock - quantity, version);

                if (success) {
                    return true;
                }
            } else {
                return false;
            }

            retryCount++;
        }

        return false;
    }

    private InventoryInfo getInventoryInfo(Long productId) {
        // 从数据库查询库存及版本号
        return new InventoryInfo(0, 0); // 示例代码
    }

    private boolean updateStock(Long productId, int newStock, int expectedVersion) {
        // 使用乐观锁更新数据库中的库存
        // UPDATE inventory SET stock = newStock, version = version + 1 WHERE product_id = productId AND version = expectedVersion
        return false; // 示例代码
    }

    private static class InventoryInfo {
        private final int stock;
        private final int version;

        public InventoryInfo(int stock, int version) {
            this.stock = stock;
            this.version = version;
        }

        public int getStock() {
            return stock;
        }

        public int getVersion() {
            return version;
        }
    }
}
```

### 5.2 分布式系统中的会话管理

在分布式系统中，多个服务实例可能需要同时访问和修改用户会话数据。

**使用悲观锁的实现（基于 Redis）：**

```java
import redis.clients.jedis.Jedis;

public class PessimisticSessionManager {
    private final Jedis jedis;

    public PessimisticSessionManager(Jedis jedis) {
        this.jedis = jedis;
    }

    public void updateSession(String sessionId, String key, String value) {
        String lockKey = "lock:session:" + sessionId;
        String lockValue = String.valueOf(System.currentTimeMillis());

        try {
            // 获取锁
            boolean acquired = acquireLock(lockKey, lockValue, 10);

            if (acquired) {
                // 更新会话数据
                String sessionKey = "session:" + sessionId;
                jedis.hset(sessionKey, key, value);
            }
        } finally {
            // 释放锁
            releaseLock(lockKey, lockValue);
        }
    }

    private boolean acquireLock(String lockKey, String lockValue, int expireSeconds) {
        String result = jedis.set(lockKey, lockValue, "NX", "EX", expireSeconds);
        return "OK".equals(result);
    }

    private void releaseLock(String lockKey, String lockValue) {
        String currentValue = jedis.get(lockKey);
        if (lockValue.equals(currentValue)) {
            jedis.del(lockKey);
        }
    }
}
```

**使用乐观锁的实现（基于 Redis）：**

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Transaction;

public class OptimisticSessionManager {
    private final Jedis jedis;

    public OptimisticSessionManager(Jedis jedis) {
        this.jedis = jedis;
    }

    public boolean updateSession(String sessionId, String key, String value) {
        String sessionKey = "session:" + sessionId;

        // 监视键
        jedis.watch(sessionKey);

        // 开始事务
        Transaction transaction = jedis.multi();

        // 更新会话数据
        transaction.hset(sessionKey, key, value);

        // 执行事务
        if (transaction.exec() == null) {
            return false; // 事务执行失败，键已被修改
        }

        return true; // 事务执行成功
    }
}
```

## 6. 注意事项

1. **选择合适的锁机制**：根据系统的特点和需求，选择合适的锁机制。如果系统中读操作远多于写操作，可以考虑使用乐观锁；如果写操作频繁或对数据一致性要求高，可以考虑使用悲观锁。

2. **避免死锁和活锁**：在使用悲观锁时，注意避免死锁；在使用乐观锁时，注意避免活锁。

3. **设置合理的超时时间**：在使用悲观锁时，设置合理的超时时间，避免长时间锁定资源。

4. **考虑锁的粒度**：锁的粒度越小，并发性能越高，但实现复杂度也越高。根据系统的需求，选择合适的锁粒度。

5. **考虑分布式环境**：在分布式环境中，需要使用分布式锁来确保在多个服务实例之间的数据一致性。

## 7. 总结

乐观锁和悲观锁是两种不同的锁机制，各有优缺点和适用场景。悲观锁通过锁定资源来防止其他操作，适用于写操作频繁、资源竞争激烈的场景；乐观锁通过版本检查来确保数据一致性，适用于读操作频繁、资源竞争不激烈的场景。在实际应用中，需要根据系统的特点和需求，选择合适的锁机制，以实现数据一致性和高并发性能的平衡。

## 参考资料

- [Java Concurrency in Practice](https://jcip.net/)
- [MySQL Documentation](https://dev.mysql.com/doc/)
- [Redis Documentation](https://redis.io/documentation)

---

希望这篇文章能帮助您更好地理解乐观锁和悲观锁的概念、实现方式和适用场景。如果您有任何问题，欢迎在评论区讨论！
