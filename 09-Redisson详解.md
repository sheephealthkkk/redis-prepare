# Redisson 详解 —— Java 程序员的第一款 Redis 高级客户端

---

## 前言：Redisson 是什么？

市面上操作 Redis 的 Java 客户端有三个：

```
Jedis：
  ┌─────────────┐
  │ 直接发命令    │  jedis.set("k", "v")   → 直接映射为 SET k v
  │ 类似 JDBC    │  jedis.hset("k","f","v") → 直接映射为 HSET k f v
  └─────────────┘
  特点：最接近 Redis 原生命令，就像是 Redis 的 "JDBC"
  缺点：连接非线程安全（需连接池 JedisPool）、没有高级封装

Lettuce：
  ┌─────────────┐
  │ 异步 + 响应式 │  基于 Netty，天然非阻塞
  │ 线程安全     │  Spring Boot 2.x+ 默认 Redis 客户端
  └─────────────┘
  特点：连接线程安全、支持异步 + Reactor 响应式编程
  缺点：也没有高级封装（没有分布式锁这些，要自己实现）

Redisson：
  ┌─────────────────────────────┐
  │ 分布式对象 + 锁 + 集合 + ... │  不直接暴露 Redis 命令
  │ 就像 Java 的 JUC 分布式版    │  而是提供 Java 风格的 API
  └─────────────────────────────┘
  特点：把 Redis 封装成 Java 的 ConcurrentHashMap、ReentrantLock、CountDownLatch...
        你像是在用 Java 的并发工具包（JUC），只不过是分布式的
```

**一句话：** Jedis/Lettuce = Redis 的驱动；Redisson = Redis 的框架。

---

## 第一章：快速上手

### 1.1 Maven 依赖

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.23.5</version>  <!-- 截止 2024 最新 -->
</dependency>
```

无需额外引入 Jedis 或 Lettuce——Redisson 内置了 Netty 连接，直接和 Redis 通信。

### 1.2 创建 RedissonClient

```java
// ===== 单节点 =====
Config config = new Config();
config.useSingleServer()
    .setAddress("redis://127.0.0.1:6379")  // Redis 地址
    .setPassword("123456")                  // 密码（没有不设）
    .setDatabase(0)                         // 用 0 号库
    .setConnectionPoolSize(32)              // 连接池大小
    .setConnectionMinimumIdleSize(8)        // 最小空闲连接
    .setConnectTimeout(10000)               // 连接超时（ms）
    .setTimeout(3000);                      // 命令超时（ms）
    // .setClientName("my-app")             // 客户端名称（在 Redis CLIENT LIST 中可看到）

RedissonClient redisson = Redisson.create(config);

// ===== 主从模式 =====
config.useMasterSlaveServers()
    .setMasterAddress("redis://192.168.1.10:6379")
    .addSlaveAddress("redis://192.168.1.11:6379", "redis://192.168.1.12:6379");

// ===== 哨兵模式 =====
config.useSentinelServers()
    .setMasterName("mymaster")
    .addSentinelAddress("redis://192.168.1.10:26379",
                        "redis://192.168.1.11:26379");

// ===== Cluster 模式 =====
config.useClusterServers()
    .addNodeAddress("redis://192.168.1.10:6379",
                    "redis://192.168.1.11:6379",
                    "redis://192.168.1.12:6379");

// ===== 用完后关闭 =====
redisson.shutdown();  // 应用关闭时调用（或注册 ShutdownHook）
```

### 1.3 直接用 Java 风格操作 Redis

不同于 Jedis 的 `jedis.set("k", "v")`，Redisson 提供 Java 原生风格的 API：

```java
// ═══ Map（对应 Redis Hash）═══
RMap<String, String> map = redisson.getMap("user:1001");
map.put("name", "张三");         // HSET user:1001 name 张三
map.put("age", "25");            // HSET user:1001 age 25
String name = map.get("name");   // HGET user:1001 name

// ═══ Set（对应 Redis Set）═══
RSet<String> set = redisson.getSet("tags:article:123");
set.add("redis");                // SADD tags:article:123 redis
set.add("database");
boolean exists = set.contains("redis");  // SISMEMBER

// ═══ List（对应 Redis List）═══
RList<String> list = redisson.getList("queue:orders");
list.add("order-1");             // RPUSH queue:orders order-1
String first = list.get(0);      // LINDEX queue:orders 0

// ═══ AtomicLong（对应 Redis String + INCR）═══
RAtomicLong counter = redisson.getAtomicLong("counter:views");
counter.incrementAndGet();       // INCR counter:views
long val = counter.get();        // GET counter:views
```

### 1.4 Redisson 和 Spring Boot 集成

```java
@Configuration
public class RedissonConfig {

    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port}")
    private int redisPort;

    @Bean(destroyMethod = "shutdown")
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
            .setAddress("redis://" + redisHost + ":" + redisPort);
        return Redisson.create(config);
    }

    // 之后直接 @Autowired 注入
    // @Autowired private RedissonClient redissonClient;
}
```

---

## 第二章：七大锁类型的完整讲解

Redisson 的核心价值在于**分布式锁**。它提供了 Java JUC（`java.util.concurrent`）的分布式版本。如果你熟悉 Java 的 `ReentrantLock`、`ReadWriteLock`、`Semaphore`、`CountDownLatch`，那么 Redisson 的锁就是这些类的分布式版。

### 2.1 RLock —— 可重入锁（最常用）

**对应的 Java 概念：** `ReentrantLock`

```java
RLock lock = redisson.getLock("lock:order:12345");

// ═══ 方式 1：阻塞获取（会启动看门狗） ═══
lock.lock();            // 一直等到拿锁为止，拿到锁前线程阻塞
try {
    // 执行业务
} finally {
    lock.unlock();      // 释放锁 + 关闭看门狗
}

// ═══ 方式 2：尝试获取（推荐） ═══
if (lock.tryLock(3, 30, TimeUnit.SECONDS)) {  // 等 3s，锁持有 30s
    try {
        // 执行业务
    } finally {
        lock.unlock();
    }
}

// ═══ 方式 3：非阻塞尝试 ═══
if (lock.tryLock()) {  // 立即返回 true/false，不等待
    try {
        // 执行业务
    } finally {
        lock.unlock();
    }
}

// ═══ 方式 4：带租约（不启动看门狗） ═══
lock.lock(10, TimeUnit.SECONDS);  // 10 秒后自动过期，不看门狗
```

**可重入验证：**

```java
RLock lock = redisson.getLock("myLock");

public void outerMethod() {
    lock.lock();
    try {
        System.out.println("外层获取锁, count=1");
        innerMethod();  // 同一个线程再次 lock
    } finally {
        lock.unlock();  // count: 2 → 1
    }
}

public void innerMethod() {
    lock.lock();       // ✅ 同一个线程，count++ = 2（可重入！）
    try {
        System.out.println("内层获取锁, count=2");
    } finally {
        lock.unlock();  // count: 2 → 1（不释放）
    }
}
// 最终 count=0 → 锁真正释放
```

> 可重入锁的详细原理（Hash结构、看门狗、Lua脚本、Pub/Sub等待）见 `04-Redis应用.md` 第 1.3 节。

### 2.2 RReadWriteLock —— 读写锁

**对应的 Java 概念：** `ReentrantReadWriteLock`

**解决什么问题？** 有些场景中，"读操作"可以并发，"写操作"必须互斥。普通锁读读也互斥，浪费性能。

```
RReadWriteLock 的规则（跟 Java 的 ReentrantReadWriteLock 完全一样）：

  读锁 + 读锁 = 可以并发（多个线程同时持有读锁）
  读锁 + 写锁 = 互斥（有人在读时，写锁等待）
  写锁 + 读锁 = 互斥（有人在写时，读锁等待）
  写锁 + 写锁 = 互斥（有人在写时，其他写锁等待）
```

```java
RReadWriteLock rwLock = redisson.getReadWriteLock("lock:product:1001");

RLock readLock = rwLock.readLock();    // 读锁
RLock writeLock = rwLock.writeLock();  // 写锁

// ═══ 读操作（多个线程可以并发读） ═══
public Product readProduct(Long productId) {
    readLock.lock();
    try {
        // 查缓存 + 查 DB → 纯读操作
        return queryProduct(productId);
    } finally {
        readLock.unlock();
    }
}

// ═══ 写操作（写时互斥，别人不能读也不能写） ═══
public void updateProduct(Long productId, Product newData) {
    writeLock.lock();
    try {
        // 更新缓存 + 更新 DB → 读写都要互斥
        updateProductInDb(productId, newData);
        redisTemplate.delete("product:" + productId);  // 删缓存
    } finally {
        writeLock.unlock();
    }
}
```

**使用场景：** 缓存更新——多数时候是读缓存，偶尔更新。读读并发不互斥，性能比普通互斥锁高得多。

### 2.3 RSemaphore —— 信号量

**对应的 Java 概念：** `Semaphore`

**解决什么问题？** 限制**同时访问**某个资源的线程数。和 RLock 不同——RLock 只允许 1 个线程，RSemaphore 允许 N 个线程同时。

```
生活中的类比：
  RLock (Permits=1)：厕所——每次只能进一个人
  RSemaphore (Permits=3)：停车场的 3 个车位——最多 3 辆车同时停
```

```java
RSemaphore semaphore = redisson.getSemaphore("sem:db-connections");

// ===== 初始化：最多 3 个许可（最多 3 个线程同时操作）=====
semaphore.trySetPermits(3);

// ===== 使用 =====
// ① 获取 1 个许可（阻塞）
semaphore.acquire();           // 没有许可时阻塞等待
try {
    operateDatabase();         // 同时最多 3 个线程执行到这里
} finally {
    semaphore.release();       // 释放许可
}

// ② 尝试获取（非阻塞）
if (semaphore.tryAcquire()) {
    try {
        operateDatabase();
    } finally {
        semaphore.release();
    }
} else {
    // 没有许可 → 降级或等待
}

// ③ 尝试获取（最多等 3 秒）
boolean ok = semaphore.tryAcquire(3, TimeUnit.SECONDS);

// ④ 增加/减少许可总数
semaphore.addPermits(2);       // 从 3 加到 5
semaphore.reducePermits(1);    // 从 5 减到 4
```

**适用场景：**
- 限制同时访问数据库的连接数（如上例）
- 限制同时调用某个第三方 API 的并发数
- 限流场景：`permits = 100` 表示最多 100 个并发（信号量限流）

### 2.4 RPermitExpirableSemaphore —— 可过期的信号量

**对应的 Java 概念：** `Semaphore` 但每个许可有 TTL 限制

**解决什么问题？** 普通 RSemaphore 的许可是永久的——你 `acquire()` 后如果不 `release()`，许可永久不归还。如果线程死了，许可泄露。RPermitExpirableSemaphore 的每个许可有**过期时间**——拿到许可超过 N 秒，许可自动归还。

```java
RPermitExpirableSemaphore semaphore =
    redisson.getPermitExpirableSemaphore("sem:api-calls");

// ===== 初始化：最多 5 个许可 =====
semaphore.trySetPermits(5);

// ===== 获取许可（许可 3 秒后自动归还，即使没有 release）=====
String permitId = semaphore.acquire(3, TimeUnit.SECONDS);
// permitId 是当前许可的唯一 ID（可用于手动释放指定许可）

try {
    callExternalApi();  // 调用第三方接口
} finally {
    // 正常释放（也可以不释放，3 秒后自动归还）
    semaphore.release(permitId);
}
```

**对比 RSemaphore：**

| 维度 | RSemaphore | RPermitExpirableSemaphore |
|------|-----------|--------------------------|
| 许可有 TTL | 无，永久有效 | 有，超时自动归还 |
| 死锁风险 | 线程死了许可泄露 | 超时自动归还，无泄露 |
| 手动释放指定许可 | 无法（只能 `release()`） | 可以 `release(permitId)` |
| 适用场景 | 长时间持有的许可 | 短时间许可，超时自动释放 |

### 2.5 RCountDownLatch —— 倒计数门闩

**对应的 Java 概念：** `CountDownLatch`

**解决什么问题？** 一个任务需要等待**多个并行任务**全部完成才能继续。

```
生活类比：
  团建出发——"等所有人都上车（countDown 到 0），大巴才发车（await 通过）"
```

```java
RCountDownLatch latch = redisson.getCountDownLatch("latch:data-import");

// ===== 主线程：初始化计数为 3 =====
latch.trySetCount(3);

// ===== 启动 3 个子任务（在 3 台机器上分别执行） =====
// 任务 1（机器 A）
new Thread(() -> {
    importUserData();            // 导入用户数据（可能 5 分钟）
    latch.countDown();           // 计数 -1（= 2）
}).start();

// 任务 2（机器 B）
new Thread(() -> {
    importOrderData();           // 导入订单数据（可能 10 分钟）
    latch.countDown();           // 计数 -1（= 1）
}).start();

// 任务 3（机器 C）
new Thread(() -> {
    importProductData();         // 导入商品数据（可能 3 分钟）
    latch.countDown();           // 计数 -1（= 0）
}).start();

// ===== 主线程：等着三个任务全部完成 =====
latch.await();  // 阻塞，直到计数归零
System.out.println("三个数据全部导入完成，可以开放系统了！");

// 或者带超时的等待：最多等 15 分钟
boolean completed = latch.await(15, TimeUnit.MINUTES);
if (!completed) {
    System.out.println("有任务超时，手动处理");
}
```

**和 JUC CountDownLatch 的区别：**
- JUC：同一 JVM 内的线程协调，计数器存在内存中
- Redisson：跨 JVM 的分布式协调，计数器存在 Redis 中

### 2.6 RedLock —— 红锁（多节点锁）

**解决什么问题？** Redis 主从异步复制导致锁失效的问题。

普通 RLock 在 Redis 主从架构下存在风险：Master 刚写入锁还没同步给 Slave，Master 宕机，Slave 被提升为新 Master——锁丢了。

RedLock 用 5 个独立的 Master 节点，过半成功才有效：

```java
// ===== 5 个独立的 Redis 节点（不是主从，各管各）=====
RLock lock1 = redisson1.getLock("lock:order");
RLock lock2 = redisson2.getLock("lock:order");
RLock lock3 = redisson3.getLock("lock:order");
RLock lock4 = redisson4.getLock("lock:order");
RLock lock5 = redisson5.getLock("lock:order");

RLock redLock = redisson.getRedLock(lock1, lock2, lock3, lock4, lock5);

// 使用方式和普通 RLock 完全一样
redLock.lock();
try {
    // 业务
} finally {
    redLock.unlock();
}
```

> 红锁的详细原理（过半原则、时钟依赖、Martin vs antirez 争议、Raft/ZK 对比）见 `04-Redis应用.md` 第 1.4-1.6 节。

### 2.7 RFencedLock —— 栅栏锁（Redis 7.0+ 新特性）

**解决什么问题？** 客户端 GC 暂停导致锁过期后，仍然认为自己在持有锁。

```
GC 暂停问题：

  ① Client-A 拿到锁，开始执行业务
  ② Client-A 发生 Full GC，暂停 40 秒
  ③ 锁在 30 秒后过期 → Client-B 拿到了锁
  ④ Client-A GC 结束，继续执行业务（不知道锁已经过期了）
  → A 和 B 同时执行临界区！

RFencedLock 的解法：
  每次获取锁时，Redis 返回一个"栅栏令牌"（fencing token）——一个单调递增的数字。
  Client 在写回数据时需要带上这个令牌，存储层接受令牌 >= 上次记录的值，
  令牌小于上次的值 → 操作被拒绝。
```

```java
RFencedLock lock = redisson.getFencedLock("lock:order");

// ===== 获取锁 =====
Long token = lock.lockAndGetToken();  // 返回栅栏令牌（如 3）
try {
    // 执行业务...
    // 写回数据时带上 token
    writeBackData(token);  // DB 也要校验 token
} finally {
    lock.unlock();
}
```

**注意：** RFencedLock 需要在**存储层**做令牌校验——Redis 只负责返回一个递增的数字，你可以把这个数字写到 MySQL（如 `version` 字段）做乐观锁校验。Redis 不帮你在存储层做校验——那需要你的业务代码配合。

### 2.8 RLock 内部的 tryLock 等待机制——再次强调

很多同学以为 tryLock 等不到锁时是 `while(true) + sleep` 那种轮询，实际上 Redisson 用了更优雅的方式：

```
tryLock(waitTime=3s) 等待过程：

  ① 尝试加锁 → 失败（返回剩余 TTL，如 25 秒）
  ② 订阅 Redis Channel: "redisson_lock__channel:{lockKey}"
     这是 Redis 的 Pub/Sub——不是轮询！
  ③ Java 线程在 Semaphore 上阻塞（park），不消耗 CPU
  ④ 锁被释放 → Redis 自动 publish 信号 → 当前客户端被唤醒 → 重新抢锁
  ⑤ 如果在 waitTime (3s) 内抢到了 → 返回 true
  ⑥ 超时 → 返回 false
```

---

## 第三章：七种锁的选型

| 锁类型 | 并发度 | 可重入 | 有 TTL | 典型场景 |
|--------|-------|--------|--------|---------|
| **RLock** | 1 | ✅ | 看门狗 | 通用互斥：扣库存、更新账户 |
| **RReadWriteLock** | 读并发/写互斥 | ✅ | 看门狗 | 读多写少：缓存更新、配置热加载 |
| **RSemaphore** | 1~N | ❌ | 无 | 并发限流：DB 连接数、API 调用数 |
| **RPermitExpirableSemaphore** | 1~N | ❌ | ✅ 每个许可有 TTL | 限流 + 防死锁：第三方接口调用 |
| **RCountDownLatch** | 协调用 | ❌ | 无 | 分布式任务协调：批量数据导入后统一切换 |
| **RedLock** | 1 | ✅ | 无 | 高安全锁：金融/支付，不能容忍主从切换丢锁 |
| **RFencedLock** | 1 | ✅ | 看门狗 | GC 暂停场景：写回数据要防"过期后还写" |

---

## 第四章：配置与调优

### 4.1 关键配置项

```java
Config config = new Config();
config.useSingleServer()
    // ═══ 连接相关 ═══
    .setAddress("redis://127.0.0.1:6379")
    .setConnectionPoolSize(64)          // 连接池大小（核心调优参数）
    .setConnectionMinimumIdleSize(24)   // 最少空闲连接（防止突发创建连接到超时）
    .setConnectTimeout(10000)           // 建连超时（ms）
    .setTimeout(3000)                   // 命令响应超时（ms）
    .setRetryAttempts(3)               // 命令失败重试次数
    .setRetryInterval(1500)            // 重试间隔（ms）

    // ═══ 看门狗调优（分布式锁相关） ═══
    .setLockWatchdogTimeout(30000)     // 看门狗续期间隔（默认 30 秒）
    // 锁的初始 TTL = 30 秒，续期间隔 = 30/3 = 10 秒

    // ═══ Redis 发布/订阅 ═══
    .setSubscriptionsPerConnection(5)  // 每连接可订阅的频道数
    .setSubscriptionConnectionPoolSize(8); // 订阅连接池大小
```

### 4.2 性能调优建议

```
① 连接池大小 = 不需要太大（Redisson 的连接是复用的）
   建议：connectionPoolSize = 并发请求数的 30-50%
   如果 QPS=10000，每次请求 1ms，需要 10 个并发连接 → 连接池设 32 足够

② timeout 不要设太短
   Lua 脚本、INCR 等操作通常 < 1ms，但 Redis 执行 BGSAVE/fork 时可能变慢
   建议 3000ms（给 Redis 一点抖动容忍）

③ 多 Redisson 实例
   一个 RedissonClient 通常足够（内置连接池）
   除非你连多个不同的 Redis 实例，才创建多个 Client

④ 编码方式
   Redisson 默认使用 Kryo 序列化（比 Java 自带快 10 倍）
   涉及 Java 对象的分布式 Map/Set 时明显更高效
```

### 4.3 YAML 配置方式

```yaml
# redisson.yaml
singleServerConfig:
  address: "redis://127.0.0.1:6379"
  connectionPoolSize: 64
  connectionMinimumIdleSize: 24
  connectTimeout: 10000
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
  lockWatchdogTimeout: 30000
```

```java
// 加载 YAML 配置
Config config = Config.fromYAML(new File("redisson.yaml"));
RedissonClient redisson = Redisson.create(config);
```

---

## ⭐️ 面试题汇总

**Q1: Redisson 和 Jedis/Lettuce 的区别？**

> Jedis/Lettuce 是 Redis 的"驱动"——提供最基础的命令执行能力。Redisson 是"框架"——把 Redis 封装成 Java 原生的并发工具（ReentrantLock、Semaphore、CountDownLatch），你不需要自己写 Lua 脚本和分布式锁逻辑。

**Q2: RLock 和 RSemaphore 的使用场景有什么区别？**

> RLock 用于互斥（一次只允许一个线程），如扣库存、更新余额。RSemaphore 用于限流（一次允许 N 个线程），如限制 DB 连接池大小、限制第三方 API 并发调用数。RLock 许可数=1，Semaphore 许可数=N。

**Q3: RSemaphore 和 RPermitExpirableSemaphore 的区别？**

> RSemaphore 的许可是永久的——线程死了没 release 会导致许可泄露。RPermitExpirableSemaphore 的每个许可有 TTL——超时自动归还，不会泄露。后者适合调用可能超时的外部 API。

**Q4: RReadWriteLock 的读写规则是什么？**

> 读-读：并发，不互斥（共享锁）；读-写：互斥；写-读：互斥；写-写：互斥（排他锁）。用于读多写少的场景——如配置更新（偶尔写）时，读操作仍然可以并发获取数据。

**Q5: RCountDownLatch 可以用在什么场景？**

> 分布式任务协调——N 个子任务分别在不同的机器上执行，主任务等待全部完成后进行下一步操作。比如批量数据导入后统一切换开关、多服务启动后注册到网关。

**Q6: 什么时候用 RedLock 而不是普通 RLock？**

> 对锁的安全性要求极高——不能容忍主从切换丢锁。普通 RLock 在 Master 宕机后锁可能不保（异步复制窗口）。RedLock 用 5 台独立 Master + 过半投票来防范这个问题。大多数互联网业务 RLock 够用。

---

*Created: 2026-05-26 | Category: 09-Redisson详解*
