# Redis 应用 —— Java 程序员视角的深度实战

---

## 阅读指南

本文按照 **"做什么 → 为什么 → 怎么做 → 踩什么坑 → 生产怎么用"** 的节奏，从浅入深讲解 Redis 的六大应用场景。每一节都从最简单的实现开始，逐步暴露问题，再到生产级方案。

---

## 1. 分布式锁

### 1.1 先理解问题：为什么 JVM 锁在分布式环境失效？

假设你写了一个扣库存的代码：

```java
// 单机部署时，synchronized 完美工作
public synchronized void deductStock(String productId) {
    int stock = getStock(productId);   // 查库存 → 3 件
    if (stock > 0) {
        updateStock(productId, stock - 1);  // 扣到 2 件
    }
}
```

**为什么这把锁在分布式环境下失效？**

```
架构图：3 个服务实例，各自有独立的 JVM

┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│   实例 A         │   │   实例 B         │   │   实例 C         │
│  synchronized   │   │  synchronized   │   │  synchronized   │
│  锁对象 A       │   │  锁对象 B       │   │  锁对象 C       │
│  (存在 JVM-A)   │   │  (存在 JVM-B)   │   │  (存在 JVM-C)   │
└────────┬────────┘   └────────┬────────┘   └────────┬────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                    都去查同一行数据库记录
                    同时读到 stock=3，同时扣到 2
                    超卖！
```

**关键洞察：** synchronized 锁住的是 **当前 JVM 内的某个对象**，管不了其他 JVM。分布式锁的本质是——所有实例去**同一个地方**抢同一把锁。

### 1.2 从零开始自己造锁——逐步演进

#### V1：SETNX —— 最简陋的锁

```java
/**
 * V1：最简单的"抢锁" —— SETNX
 * SETNX = SET if Not eXists（key 不存在才设置）
 * 原理：利用 Redis 单线程特性，谁先 SETNX 成功谁拿到锁
 */
public boolean tryLockV1(String lockKey, String lockValue) {
    // setnx 返回 1 → key 之前不存在，设置成功 → 拿到锁
    // setnx 返回 0 → key 已存在 → 别人拿着锁 → 获取失败
    Long result = jedis.setnx(lockKey, lockValue);
    return result == 1;
}

public void unlockV1(String lockKey) {
    // 释放锁就是删 key
    jedis.del(lockKey);
}
```

**致命问题：** 拿到锁后如果服务宕机，`unlock` 代码没机会执行 → key 永久存在 → **死锁**。:rocket::rocket:

#### V2：SETNX + EXPIRE —— 加个过期时间解决死锁？

```java
/**
 * V2：给锁加过期时间
 * 意图：30 秒后锁自动删除，即使服务崩溃也不会死锁
 */
public boolean tryLockV2(String lockKey, String lockValue) {
    Long result = jedis.setnx(lockKey, lockValue);
    if (result == 1) {
        jedis.expire(lockKey, 30);  // ← 如果这里宕机了呢？
    }
    return result == 1;
}
```

**致命问题：SETNX 和 EXPIRE 是两条独立命令，中间可能宕机。** 一个命令执行完，第二个命令还没发出去，进程被 kill -9 → 锁设置了但没有过期时间 → 死锁。:rocket:

#### V3：SET ... NX PX —— 原子操作

```java
/**
 * V3：用一条命令同时完成"加锁 + 设过期"
 *
 * SET lockKey lockValue NX PX 30000
 *   NX    = Not eXists，key 不存在才设置（等于 SETNX 的条件）
 *   PX 30000 = 过期时间 30000 毫秒（= 30 秒后自动释放）
 *
 * 一条命令 → Redis 单线程保证原子 → 中间不可能插入别的命令
 */
public boolean tryLockV3(String lockKey, String lockValue) {
    String result = jedis.set(lockKey, lockValue,
        SetParams.setParams()   // 构建 SET 命令的参数
            .nx()               // NX：不存在才设置
            .px(30000));        // PX 30000ms：30 秒过期
    return "OK".equals(result); // "OK"=成功，null=已存在被别人持有了
}
```

**新问题——误删别人的锁：**过期之后解锁，解的锁不是自己的锁:rocket::rocket:

```
时间线：
  t=0s   线程 A 拿到锁（lockValue = "uuid-A"），锁 30s 过期
  t=25s  线程 A 还在执行业务（慢了，比如调第三方接口超时）
  t=30s  锁自动过期了！
  t=31s  线程 B 拿到锁（lockValue = "uuid-B"）
  t=35s  线程 A 业务执行完 → 执行 unlock → jedis.del(lockKey)
         ↑ 这把锁是 B 的！A 把 B 的锁删了！
         线程 C 又可以拿到锁 → A 和 C 同时执行 → 并发安全崩了
```

**根本原因：释放锁时没有验证"这把锁是不是我的"。**

#### V4：释放时验证身份（Lua 原子脚本）:rocket::rocket:

```java
/**
 * V4：用 Lua 脚本原子地执行"验证 + 释放"
 *
 * 加锁时：lockValue = UUID + ":" + Thread.currentThread().getName()
 * 解锁时：用 Lua 判断 Redis 中的 value 是否等于我的 lockValue
 *         是 → 删（释放）  否 → 不删（别碰别人的锁）
 */
public class RedisLockV4 {
    // Lua 脚本：在 Redis 服务端原子执行
    // KEYS[1] = 锁的 key，如 "lock:product:1001"
    // ARGV[1] = 加锁时设置的 value，如 "abc123-def456:thread-1"
    private static final String UNLOCK_LUA =
        "if redis.call('get', KEYS[1]) == ARGV[1] then " +  // ① 比对 value
        "    return redis.call('del', KEYS[1]) " +           // ② 匹配 → 删除
        "else " +
        "    return 0 " +                                     // ③ 不匹配 → 不删
        "end";

    private Jedis jedis;

    // 加锁
    public boolean tryLock(String lockKey, int expireMs) {
        String lockValue = UUID.randomUUID() + ":" + Thread.currentThread().getName();
        String result = jedis.set(lockKey, lockValue,
            SetParams.setParams().nx().px(expireMs));
        return "OK".equals(result);
    }

    // 解锁
    public void unlock(String lockKey, String lockValue) {
        jedis.eval(UNLOCK_LUA,
            Collections.singletonList(lockKey),    // KEYS[1]
            Collections.singletonList(lockValue));  // ARGV[1]
    }
}
```

**为什么 lockValue 是 UUID:线程名？** 不同 JVM 实例可能生成相同的 UUID（极低概率），加上线程名确保**全局唯一**可辨识。

#### V1-V4 还没解决的问题：过期时间到底设多长？:rocket::rocket::rocket:

```
设短了（如 5 秒）→ 业务调第三方接口花了 6 秒 → 锁过期被别人拿走 → 并发
设长了（如 60 秒）→ 业务 1 秒就执行完 → 锁白白占了 59 秒 → 性能浪费

两难！因为你不知道业务会执行多久。
```

**解决方案——自动续期，也就是"看门狗"。**

### 1.3 Redisson —— 生产级分布式锁

Redisson 是 Redis 最广泛使用的 Java 客户端，它的分布式锁解决了 V1-V4 的所有问题，而且一行代码搞定。

#### 1.3.1 基本用法

```java
// 第一步：创建客户端（内部维护 Netty 连接池）
Config config = new Config();
config.useSingleServer().setAddress("redis://127.0.0.1:6379");
RedissonClient redisson = Redisson.create(config);

// 第二步：获取锁对象（这行只是创建 Java 对象，不发 Redis 命令）
RLock lock = redisson.getLock("lock:order:12345");

// 第三步：加锁并执行业务
// ===== 方式一：lock() —— 阻塞等待，会启动看门狗 =====
lock.lock();                     // 阻塞直到拿到锁；业务不结束，锁不释放
try {
    // 执行业务逻辑...
} finally {
    lock.unlock();               // 释放锁 + 关闭看门狗
}

// ===== 方式二：tryLock —— 带超时（推荐） =====
if (lock.tryLock(3, 30, TimeUnit.SECONDS)) {  // 最多等 3 秒，锁持有 30 秒
    try {
        // 执行业务...
    } finally {
        lock.unlock();
    }
} else {
    // 3 秒没拿到锁 → 做降级处理
}
```

**关键区别：** :rocket::rocket::rocket:`lock()` 启动看门狗，锁永不过期（只要你还在跑）。`tryLock(wait, leaseTime, unit)` 指定了 leaseTime，**不启动看门狗**，到期就释放。

#### 1.3.2 Redisson 用 Hash 存锁——为什么？

Redisson 不是用 `SET lockKey value NX PX` 存锁，而是用 **Hash**：

```
Redis 中的实际存储：
  Key:   lock:order:12345               ← Hash 类型
  Field: "abc123-def456:thread-1"       ← 客户端唯一标识
  Value: 1                              ← 可重入计数
```

| Redis 数据类型 | Key（锁名） | Field（谁持有锁） | Value（重入次数） |
| :------------- | :---------- | :---------------- | :---------------- |
| Hash           | `myLock`    | `UUID:threadId`   | 1（初始值）       |



:rocket::rocket::rocket::rocket::rocket::rocket::rocket:

`ARGV` 是一个**数组**

### “数组”其实来自 Lua 脚本的 `ARGV`

在加锁/解锁的 Lua 脚本里，`ARGV` 是一个**脚本参数数组**，它是调用 `EVAL` 时传进去的**一系列字符串**。

lua

```
-- 解锁脚本
if redis.call('hexists', KEYS[1], ARGV[3]) == 0 then ... end
```



- `ARGV[1]`、`ARGV[2]`、`ARGV[3]` 是三个**独立的参数**，被 Lua 解释器放在一个叫 `ARGV` 的 table（类似数组）里。
- 它们分别是：解锁消息、过期时间、客户端标识。
- 每个参数只是一个**普通的字符串**，比如 `ARGV[3]` 就是 `"e7b1a3c2-...:1"`，绝不会被存成一个“多含义的数组元素”。

**Hash 的 value 没有存储这个数组，它只存重入计数。**hash的key，field，value都没有存它，它是存的脚本层面的内容。
ARGV 只是脚本执行时的临时输入，用完就没了。:rocket::rocket::rocket::rocket::rocket::rocket:











**为什么用 Hash？两个原因：**:rocket::rocket::rocket:

**原因一：可重入。** 同一个线程可以多次调用 `lock()`，count 累加；`unlock()` 时 count 递减，减到 0 才真正释放锁。

```
lock()   → HSET lock:order:12345 "uuid:t1" 1   → count=1
lock()   → HINCRBY lock:order:12345 "uuid:t1" 1   → count=2（重入）
unlock() → HINCRBY lock:order:12345 "uuid:t1" -1  → count=1（未释放）
unlock() → DEL lock:order:12345                    → count=0（真正释放）
```

**原因二：验证身份。** 解锁时 `hexists` 判断 "解锁者是否就是加锁者"，避免误删。

#### 1.3.3 加锁 Lua 脚本逐行解读

Redisson 加锁的本质是执行了下面这段 Lua 脚本（在 Redis 服务端原子执行）：

```lua
-- KEYS[1] = "lock:order:12345"  （锁的 key）
-- ARGV[1] = 30000                （锁过期时间 ms）
-- ARGV[2] = "abc123-def456:thread-1"  （客户端标识）

-- 情况 1：锁不存在 → 直接加锁
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);     -- HSET field=1
    redis.call('pexpire', KEYS[1], ARGV[1]);          -- 设 30s 过期
    return nil;  -- 加锁成功，返回 nil
end;

-- 情况 2：锁存在，且 field=我的标识 → 可重入
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);     -- count +1
    redis.call('pexpire', KEYS[1], ARGV[1]);          -- 重置 30s 过期
    return nil;  -- 可重入成功
end;

-- 情况 3：锁被其他人持有 → 返回剩余 TTL（让客户端知道还要等多久）
return redis.call('pttl', KEYS[1]);
```

**这段脚本解释了三件事：**:rocket::rocket:

1. 锁是 Hash，field 是客户端标识，value 是重入计数
2. 同线程重复 lock() 不会死锁（可重入）
3. 返回 TTL 让等待方知道 "大概还要等多少毫秒"，不用盲目轮询

#### 1.3.4 看门狗（Watchdog）—— 自动续期

看门狗是 Redisson 的杀手级特性。它解决的核心矛盾是：**"不设过期怕死锁，设了过期怕没执行完"**。:rocket::rocket::rocket:

```
工作流程：
  ① lock() → Lua 加锁，初始过期时间 = 30 秒
  ② 加锁成功后 → 启动看门狗定时任务
  ③ 每 10 秒（30s / 3）触发一次：
      检查 Hash 中 field=我 是否还存在
      存在 → PEXPIRE 重置过期时间为 30 秒
      不存在 → 停止续期（锁已被释放或过期）
  ④ unlock() → 取消看门狗定时任务
```

**看门狗源码（RedissonLock.java 简化）：**

```java
+// 全局 Map：存储每个锁对象的续期任务
// key = 锁的唯一标识, value = 续期任务信息
// 这样 unlock() 时可以根据锁标识找到对应的续期任务来取消它
private static final ConcurrentMap<String, ExpirationEntry> EXPIRATION_RENEWAL_MAP
    = new ConcurrentHashMap<>();

private void renewExpiration() {
    // ① 获取该锁的续期记录（一个锁对应一个续期任务）
    String entryName = getEntryName();  // 如 "lock:order:12345:abc123:thread-1"
    ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(entryName);
    if (ee == null) {
        return;  // 可能已经被 unlock 清理掉了，停止续期
    }

    // ② 创建延迟任务：等 10 秒后执行一次续期
    // internalLockLeaseTime = 30000ms，除以 3 = 10000ms = 10 秒
    Timeout task = commandExecutor.getConnectionManager()
        .newTimeout(timeout -> {

            // ③ 异步执行续期 Lua 脚本
            // if redis.call('+hexists', KEYS[1], ARGV[2]) == 1 then
            //     redis.call('pexpire', KEYS[1], ARGV[1])  ← 重置为 30 秒
            //     return 1
            // end
            // return 0  ← field 不存在，锁已消失
            RFuture<Boolean> future = renewExpirationAsync(threadId);

            // ④ 处理续期结果
            future.whenComplete((success, error) -> {
                if (success) {
                    // 续期成功 → 锁还在 → 10 秒后再续一次（递归循环）
                    renewExpiration();
                }
                // 续期失败（success=false）→ 锁没了 → 停止续期
            });

        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);  // 10 秒后触发

    // ⑤ 记录这个定时任务对象，unlock() 时需要 cancel 掉它
    ee.setTimeout(task);
}
```

**一句话总结：** 看门狗让 "只要业务在跑，锁就永远不会过期"；但如果程序崩溃，看门狗没了，锁 30 秒后还是自动过期——既防止了死锁，又保护了长业务。

#### 1.3.5 解锁全流程

```lua
-- 解锁 Lua 脚本
-- KEYS[1] = 锁 key
-- KEYS[2] = 解锁通知 channel（Pub/Sub）
-- ARGV[1] = 解锁消息（0）
-- ARGV[2] = 锁过期时间
-- ARGV[3] = 客户端标识

-- ① 锁里没有我的 field → 报错（不是我的锁，不能解）
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
    return nil;
end;

-- ② count - 1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);

-- ③ count > 0 → 还有重入层数未释放 → 续一下过期时间，但不释放
if (counter > 0) then
    redis.call('pexpire', KEYS[1], ARGV[2]);
    return 0;

-- ④ count == 0 → 所有重入都释放了 → 真正删锁
else
    redis.call('del', KEYS[1]);
    redis.call('publish', KEYS[2], ARGV[1]);  -- 发通知唤醒等待的线程
    return 1;
end;
```

#### 1.3.6 tryLock 等待机制——不是轮询

加锁失败后，Redisson 不是 `while(true) + sleep` 那种 CPU 空转，而是 **Pub/Sub + 信号量**：:rocket::rocket::rocket::rocket:

```
① tryLock(3秒) → Lua 加锁失败 → 返回剩余 TTL（如 25 秒）
② 订阅 Redis channel："redisson_lock__channel:{lock:order:12345}"
③ Java 线程在 Semaphore 上阻塞等待（无 CPU 消耗），超时时间 = min(waitTime, TTL)
④ 持有锁的客户端 unlock → publish 信号 → 当前客户端被唤醒 → 重新抢锁
⑤ 3 秒内抢到了 → 返回 true；超时 → 返回 false
```



### 1.4 理解"主从切换为什么导致锁失效"——红锁的前置知识

#### 1.4.1 Redis 主从复制的本质：异步

在你理解红锁之前，必须先理解 **Redis 主从之间是怎么同步数据的**。:rocket::rocket::rocket::rocket::rocket::rocket:

```
正常情况下的主从同步：

  Client ──SET lock NX PX──▶ Master ──写入成功，返回 OK──▶ Client
                              │
                              │ 异步复制（后台发）
                              ▼
                            Slave ──收到数据，写入本地

关键：Master 先回复 Client "OK"，然后才异步同步给 Slave。
      这意味着：在 Master 回复 OK 和 Slave 收到数据之间，
      存在一个短暂的时间窗口，数据只在 Master 上。
```

**这个窗口就是一切问题的根源。** 正常情况下这个窗口只有几毫秒，但万一 Master 恰好在这个窗口内宕机了呢？

#### 1.4.2 主从切换（Failover）下的锁失效场景:rocket::rocket::rocket::rocket::rocket::rocket:

```
场景图：

  时间轴：  t=0s                   t=0.1s              t=0.2s
           │                      │                   │
  Client-A─┤ SET lockA NX PX ✓    │                   │
           │ (Master 返回 OK)     │                   │
           │                      │                   │
  Master───┤ 写入 lockA           │ ⚡宕机！           │
           │ (还没来得及同步给Slave)│                   │
           │                      │                   │
  Slave────┤ (不知道 lockA)       │ 被哨兵提升为新Master │
           │                      │                   │
  Client-B─┤                      │                   │ SET lockB NX PX ✓ ！
           │                      │                   │ (新Master不知道lockA)
           
  结果：Client-A 以为锁还在自己手里，继续执行业务（查库存、扣库存）。
        Client-B 也拿到了锁，也开始执行业务。
        A 和 B 同时执行临界区代码 → 数据错乱！
```

**这就是 Redis 默认主从架构在极端情况下无法保证互斥的原因。**

### 1.5 红锁（RedLock）—— 用"多主投票"替代"主从同步"

#### 1.5.1 红锁的核心思想

Redis 作者 antirez 提出了 RedLock 算法。核心思想很直接：**不要主从，用多个独立的主节点，过半成功才有效。**

```
架构对比：

  传统主从：                        红锁（RedLock）：
  ┌───────┐                        ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
  │Master │                        │Redis│ │Redis│ │Redis│ │Redis│ │Redis│
  └──┬────┘                        │  1  │ │  2  │ │  3  │ │  4  │ │  5  │
     │                             └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘
  ┌──┴──┐                             │       │       │       │       │
  │Slave│                             └───────┴───┬───┴───────┴───────┘
  └─────┘                                        │
                                          所有节点平等，都是 Master
  一个 Master 挂了 → 丢锁          一个节点挂了 → 还有其他 4 个
                                  过半（≥3）成功 → 锁才有效
```

**关键决策：** 不用主从复制，而是 5 个完全独立的 Redis 实例（不互相复制，各存各的）。锁同时写到这 5 个节点上，需要大多数（≥3）写成功，才算拿到锁。:rocket::rocket::rocket::rocket::rocket::rocket:

#### 1.5.2 红锁算法的完整执行步骤

```
获取锁（Acquire）：

  假设有 N=5 个 Redis 节点，锁的 TTL=30 秒

  ① 记录开始时间：startTime = 当前毫秒时间戳

  ② 依次向 5 个节点发送：SET lockName randomValue NX PX 30000
     每个节点有独立的超时（比如 50ms）—— 单节点超时不要死等

  ③ 统计结果：
     在多少个节点上 SET 成功了？
     总耗时 = 当前时间 - startTime

  ④ 判断是否成功：
     成功节点数 ≥ N/2+1（即 ≥3）       ← "过半"原则
     且  总耗时 < TTL（30秒）           ← 锁的实际有效时间 = TTL - 总耗时
     → 获取锁成功！

  ⑤ 如果失败：
     向所有节点发送 Lua DEL 脚本（不管有没有成功 SET 过，全部清理）
     避免部分节点写入了但部分没写入，留下残留锁


释放锁（Release）：

  向所有 5 个节点发送 Lua DEL 脚本（无条件全部删）


Redisson 用法：
  RLock lock = redisson.getRedLock(
      redisson.getLock("lock:node1"),
      redisson.getLock("lock:node2"),
      redisson.getLock("lock:node3"),
      redisson.getLock("lock:node4"),
      redisson.getLock("lock:node5")
  );
  lock.lock();
```

**为什么必须"过半"？** 这是分布式共识的基本法则。如果两个客户端同时抢锁，它们不可能同时获得大多数节点的成功——因为每个节点只能被一个客户端 SET NX 成功。这是靠 NX 条件保证的。:rocket::rocket::rocket:

**为什么总耗时必须 < TTL？** 因为从第一个节点 SET 成功到最后一个节点 SET 成功之间拖了时间，锁的实际剩余有效时间变短了。如果总耗时接近或超过 TTL，说明这条锁实际上已经没保护了。:rocket:

#### 1.5.3 红锁仍然不安全——一个经典的反驳

分布式系统大神 Martin Kleppmann（Kafka 作者之一、《Designing Data-Intensive Applications》作者）对 RedLock 提出了一个著名反驳：

**"依赖时钟的分布式锁是不可靠的。"**

```
假设 Client-A 在 3 个节点上拿到了锁：

  Node1: lockA 存入，TTL=30s
  Node2: lockA 存入，TTL=30s
  Node3: lockA 存入，TTL=30s
  Node4: 没拿到
  Node5: 没拿到

  Client-A 认为"我拿到了锁"（≥3 成功）

现在发生以下情况：
  Node1 上的 TTL 因为系统时间跳变（NTP 校时），提前到期了
  或者：Client-A 的 GC 暂停了 40 秒（Java 的 Full GC）

  等 Client-A 从 GC 中恢复过来，Node1 上的锁早就过期了
  但 Client-A 不知道 → 继续执行业务 → 可能和别的 Client 冲突
```

**Martin 的核心观点：**
- 锁的"过期时间"依赖系统时钟，但分布式系统中时钟不是完美的（时钟漂移、NTP 对时、GC 暂停等）:rocket:
- 无法保证"我以为锁还在"和"锁确实还在"之间的正确对应
- **正确的分布式锁应该用单调递增的版本号（fencing token），而不是依赖时钟**

**antirez 的回应：**
- RedLock 不需要精确时钟，只需要"时钟偏差 < TTL"这个宽松假设
- GC 暂停问题可以通过调整 TTL 来缓解（TTL 远大于 GC 暂停时间）:rocket:
- 完全脱离时钟的锁不存在——就算是 ZooKeeper 的 ephemeral 节点也需要心跳超时（本质上也是时间）

**这场争论没有绝对赢家，但它告诉我们一个道理：** 如果你的业务对一致性的要求到了"不容半点偏差"的地步，就不要用 Redis 锁——用 ZooKeeper 或 etcd。

### 1.6 背景知识：Raft 和 ZooKeeper 是怎么回事？

既然提到了 ZooKeeper 和 Raft，这里做零基础入门讲解——理解它们的基本原理，你才能明白"为什么 ZK 的锁比 Redis 的锁更安全"。

#### 1.6.1 分布式系统的核心难题：共识（Consensus）

**共识问题 = 多个节点如何对一个"决定"达成一致。**

```
场景：3 个节点，需要选一个"谁是锁的持有者"

  节点A 收到 Client1 的加锁请求
  节点B 收到 Client2 的加锁请求（几乎同时）
  节点C 暂时没收到任何请求

  问题：三个节点各有一个"世界状态"，怎么统一？
  谁来决定谁是锁的持有者？
```

这就是分布式共识要解决的问题。有两个经典算法：

#### 1.6.2 Raft 算法——用"领导说了算"解决问题

**Raft 的核心思想：** 先选出一个 Leader（领导），所有写操作必须经过 Leader，Leader 把决策同步给 Follower（追随者），过半确认才提交。

```
Raft 集群（3 节点）：

  ┌──────────┐
  │  Leader  │ ← 唯一的"领导"，所有写操作经过它
  └────┬─────┘
       │ ① Leader 收到写请求 → 先记本地日志
       │ ② 把日志复制给 Follower-A 和 Follower-B
       ▼
  ┌──────────┐  ┌──────────┐
  │Follower A│  │Follower B│
  └──────────┘  └──────────┘
       │              │
       └──────┬───────┘
              │ ③ 过半（≥2）确认 → Leader 提交 → 通知所有节点
              
  关键：任何决定都必须过半节点确认才生效。
       如果 Leader 宕机 → 剩余节点重新选举新 Leader（也是过半投票）
```

**Raft 的锁实现（etcd 用 Raft）：** 锁 = 一个 KV 条目。加锁 = 把这个 KV 条目通过 Raft 提交到半数以上节点。**Leader 宕机了不会丢锁**——新 Leader 包含所有已提交的条目（包括锁）。

#### 1.6.3 ZooKeeper——用 CP 语义实现分布式锁

ZooKeeper 使用 ZAB 协议（类似 Raft，也是过半提交）。ZK 有两个关键特性使它适合做锁：

**特性一：临时顺序节点**

```
加锁流程：
  ① 所有客户端在同一个 ZK 路径下创建"临时顺序节点"
     例如：/lock/order/000001, /lock/order/000002, /lock/order/000003
  
  ② 序号最小的节点 = 锁的持有者
  
  ③ 序号更大的节点 → 监听前一个节点的删除事件 → 形成"等待队列"

  ④ 锁释放 → 删除节点 → 下一个监听者被通知 → 自动获取锁

  ⑤ 如果客户端崩溃 → ZK Session 断开 → 临时节点自动删除 → 后面的人自动获取锁
     （不需要等超时！崩溃检测是即时的！）
```

**特性二：CP 一致性保证**

```
ZK 集群（3 节点或 5 节点）→ CP 系统（CAP 中的 C + P）
  - 写入必须过半节点确认（一致性强）
  - 如果 Leader 宕机 → 选举期间不对外服务（牺牲可用性保一致性）

vs Redis Sentinel：
  - 主从异步复制 → 可能丢数据 → AP 系统（保证可用性但弱一致性）
```

#### 1.6.4 Redis 锁 vs ZK 锁 vs etcd 锁——完整对比

| 维度 | Redis (单节点+Sentinel) | Redis 红锁 (RedLock) | ZooKeeper | etcd |
|------|------------------------|---------------------|-----------|------|
| **一致性模型** | AP（弱一致） | 多主投票，依赖时钟 | CP（强一致） | CP（强一致） |
| **共识算法** | 无（异步复制） | 无（客户端投票） | ZAB（类 Raft） | Raft |
| **锁安全性** | 主从切换可能丢锁 | 理论上更强（不保证绝对安全） | 极强 | 极强 |
| **崩溃响应** | 等 TTL 过期 | 等 TTL 过期 | ZK Session 断开 → 立即释放 | Lease 过期 → 释放 |
| **性能** | 极高（单点） | 高（N 并发请求） | 中等（写要过半） | 中等（写要过半） |
| **运维复杂度** | 低 | 中（5 个独立实例） | 高（ZK 集群 + JVM） | 中（etcd 集群） |
| **典型场景** | 99% 互联网业务 | 需要更高安全性的场景 | 严格互斥（金融、配置） | 严格互斥、服务发现 |

#### 1.6.5 选型建议

```
你的业务属于哪种？

  ① "锁偶尔失效可以接受，下次请求自动纠正"
     → 单节点 Redis + Sentinel，性能高，运维简单
     → 绝大多数互联网业务都在这层

  ② "需要较高安全性的分布式锁"
     → Redis 红锁（5 个独立节点）
     → 或者用 Redisson 的单节点 + 看门狗（大多数情况够用）

  ③ "锁不容半点失效，这是金融/支付系统的资金操作"
     → 放弃 Redis，用 ZooKeeper 或 etcd
     → 这些系统的一致性模型是为"不能出错"设计的
```

### 1.7 分布式锁面试要点

- **为什么主从切换可能导致 Redis 锁失效？** → 主从异步复制：Master 回复 OK 后才同步 Slave，Master 宕机时未同步的锁永久丢失
- **红锁怎么解决这个问题？** → 5 个独立 Master，过半（≥3）成功才算获取
- **红锁的致命弱点是什么？（Martin 的批评）** → 依赖时钟（TTL），GC 暂停/时钟跳跃可能让它以为锁还在但实际已过期
- **Raft 共识的原理是什么？** → Leader + 过半提交 + 日志复制。Leader 宕机重新选举，新 Leader 包含所有已提交日志
- **ZooKeeper 锁为什么比 Redis 安全？** → 临时节点 + Session 心跳 → 崩溃即时检测（不用等 TTL）；ZAB 协议过半写入 → 强一致性
- **什么时候用 Redis 锁，什么时候用 ZK？** → Redis：性能高、运维简单、接受偶尔失效 → 99% 互联网业务。ZK/etcd：强一致性要求 → 金融/支付



## 2. 限流

### 2.1 先彻底搞懂：为什么需要分布式限流？

#### 2.1.1 限流的动机——保护后端

限流的本质目的：**后端服务的吞吐有上限，超过上限就会崩溃。**

```
一条请求链路的处理能力：

  用户请求 → Nginx → 业务服务 → Redis → MySQL
                              │
                              └── 这个业务服务最多处理 1000 QPS
                                  超过 1000 → CPU 100% → 响应超时
                                  → 用户疯狂刷新 → 请求更多 → 服务彻底崩
                                  → 雪崩
```

**限流就是在入口处拦掉"多余的请求"，让通过的数量不超过后端的处理能力。**

#### 2.1.2 为什么单机限流在分布式环境下不起作用

假设你的发短信接口 QPS 上限是 1000，部署了 3 台服务器：

```
场景 A：每台服务器限 1000 QPS（单机限流）

  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  实例 A   │  │  实例 B   │  │  实例 C   │
  │ 限 1000/s │  │ 限 1000/s │  │ 限 1000/s │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
       └──────────────┼──────────────┘
                      ▼
              ┌──────────────┐
              │  短信服务商    │ ← 承受 3000 QPS！
              │  上限: 1000   │    账号被封或额外收费
              └──────────────┘

  问题：每台服务器都觉得自己"只发了 1000"，但加起来发了 3000


场景 B：每台限 333 QPS（单机限流，手动除 3）

  问题 1：流量不均匀——A 可能被分配了 800 QPS 但只能处理 333（白白拒绝）
  问题 2：扩缩容要重新算——加了第 4 台 → 每台 250 → 要改配置重启
```

**核心矛盾：** 单机限流感知不到全局:rocket::rocket:。集群的总流量是各实例流量的总和，但单个实例不知道别的实例在做什么。

#### 2.1.3 解决方案——所有实例去同一个地方计数:rocket::rocket::rocket::rocket::rocket::rocket:

```
分布式限流：

  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  实例 A   │  │  实例 B   │  │  实例 C   │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
       └──────────────┼──────────────┘
                      │ 每个请求 → Redis INCR key
                      ▼
              ┌──────────────┐
              │    Redis     │  ← 全局计数器
              │  rate:sms   │  ← 单线程，INCR 天然原子
              │  count = N  │  ← 所有实例共享同一个 view
              └──────┬───────┘
                     │ N > 1000 ？→ 拒绝  → 返回"系统繁忙"
                     │ N ≤ 1000 ？→ 放行
                     ▼
              ┌──────────────┐
              │  短信服务商    │ ← 永远最多收 1000/s
              │  上限: 1000   │
              └──────────────┘
```

**为什么 Redis？** 因为 Redis 是单线程执行命令的，多台服务器同时 INCR 同一个 key → Redis 排队串行执行 → 计数精确、没有竞态。如果三台服务器各自在本地内存中做 `AtomicInteger.incrementAndGet()`，它们是三个不同的对象，无法加起来。

### 2.2 限流算法一：固定窗口——最简单也最危险

```java
/**
 * 最简单的固定窗口：INCR + EXPIRE
 * 规则：每个用户每 60 秒最多 10 次请求
 */
public boolean fixedWindow(String userId) {
    String key = "rate:" + userId;         // 如 rate:user:10086
    long count = jedis.incr(key);           // 自增计数（INCR 原子）
    if (count == 1) {                       // 第一次访问
        jedis.expire(key, 60);              // 设 60 秒过期（窗口长度）
    }
    return count <= 10;                     // ≤10 → 放行 ; >10 → 拒绝
}
```

**固定窗口的漏洞——临界问题：**:rocket::rocket:

```
窗口：每分钟最多 10 次

  0:00 ═══════════════ 0:59 ──── 1:00 ═══════════════ 1:59
  窗口1 [──最多 10 次──]       │ 窗口2 [──最多 10 次──]

攻击者在 0:58 ~ 1:02 这 4 秒内：
  0:58 → 发 10 次（窗口1 的配额）
  1:00 → 发 10 次（窗口2 的配额）
  结果：4 秒内 20 次 → 2 倍于限制的流量 → 后端承受不住
```

### 2.3 滑动窗口限流——ZSet 实现

**核心思路：** 不用死边界，用当前时间为终点，往回看 60 秒内的请求数。

```
固定窗口：┌────────────┬────────────┐  每次跨边界有机会绕过
         0:00      0:59        1:00

滑动窗口：              ┌────────────┐  窗口始终以"现在"为终点
                now-60s          now      每次统计都是平滑的
```

```java
/**
 * ZSet 滑动窗口限流
 *
 * 原理：
 *   - score = 请求时间戳（毫秒）
 *   - member = UUID（保证每条记录唯一）
 *   - 每次请求先把 "60 秒之前的旧数据" 全部删掉
 *   - 统计窗口内还剩多少条 → 判断是否放行
 *
 * 为什么用 Lua？
 *   "删旧数据 + 计数 + 判断 + 加新数据" 四步必须原子，
 *   否则多个请求可能同时通过判断，都认为自己没超限。
 */
/**
 * 滑动窗口限流（基于 Redis ZSET + Lua 原子操作）
 *
 * @param userId    用户标识
 * @param limit     窗口内允许的最大请求次数
 * @param windowSec 窗口大小（秒）
 * @return true=放行， false=限流拒绝
 */
public boolean slidingWindow(String userId, int limit, int windowSec) {
    // 1. 拼接 Redis key，每个用户一个独立的 ZSET
    //    例如: "rate:sliding:user_123"
    String key = "rate:sliding:" + userId;

    // 2. 获取当前时间戳（毫秒），作为本次请求的发生时间
    long now = System.currentTimeMillis();

    // 3. 计算滑动窗口的左边界（最早有效时间）
    //    超过这个时间戳的请求记录都要被删除
    long windowStart = now - windowSec * 1000L;

    // 4. 定义 Lua 脚本，保证“清理 + 统计 + 判断 + 记录”四步原子执行
    String lua =
        // ---------- 第 1 步：删除窗口外的旧数据 ----------
        // ZREMRANGEBYSCORE key min max
        // 删除 score 在 [0, windowStart] 之间的所有成员（即过期的请求记录）
        "redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1]) " +

        // ---------- 第 2 步：统计窗口内还剩多少条请求 ----------
        // ZCARD key  返回有序集合的成员数量
        "local count = redis.call('ZCARD', KEYS[1]) " +

        // ---------- 第 3 步：判断是否超过限制 ----------
        // ARGV[2] 是字符串类型的 limit，需要转成数字再比较
        "if count < tonumber(ARGV[2]) then " +

        // ---------- 第 4 步：未超限，放行，记录本次请求 ----------
        // ZADD key score member  将本次请求加入 ZSET
        // score = 当前时间戳（ARGV[3]），member = UUID（ARGV[4]）
        "    redis.call('ZADD', KEYS[1], ARGV[3], ARGV[4]) " +

        // ---------- 第 5 步：刷新 key 的过期时间 ----------
        // EXPIRE key seconds  设置 TTL，避免冷用户长期占用内存
        // 过期时间比窗口稍长（windowSec + 1），保证窗口数据完整
        "    redis.call('EXPIRE', KEYS[1], ARGV[5]) " +

        // 返回 1 表示放行
        "    return 1 " +

        "else " +
        // ---------- 第 6 步：超限，拒绝 ----------
        // 返回 0 表示限流
        "    return 0 " +
        "end";

    // 5. 生成本次请求的唯一标识（UUID），作为 ZSET 的 member
    //    这样同一毫秒内的多个请求也能分别记录，不会互相覆盖
    String memberId = UUID.randomUUID().toString();

    // 6. 执行 Lua 脚本，所有参数通过 KEYS / ARGV 传入
    Object result = jedis.eval(
        lua,
        // KEYS 列表，这里只有一个元素，对应 Lua 中的 KEYS[1]
        Collections.singletonList(key),
        // ARGV 列表，顺序和 Lua 脚本中 ARGV[1] ~ ARGV[5] 一一对应
        Arrays.asList(
            String.valueOf(windowStart),   // ARGV[1] 窗口左边界（毫秒）
            String.valueOf(limit),         // ARGV[2] 限流阈值
            String.valueOf(now),           // ARGV[3] 当前时间戳（score）
            memberId,                      // ARGV[4] UUID（member）
            String.valueOf(windowSec + 1)  // ARGV[5] key 的过期秒数
        )
    );

    // 7. 解析 Lua 返回值：1 表示放行，0 表示限流
    return "1".equals(result.toString());
}
```

**滑动窗口 VS 固定窗口：** 滑动窗口没有"窗口边界"，攻击者在任意 4 秒内的请求量都被平滑限制。代价：每次请求都要 ZREMRANGEBYSCORE（删旧数据）——但 ZSet 元素少时开销极小。

### 2.4 令牌桶——生产环境的工业标准（从零讲起）:rocket::rocket::rocket::rocket::rocket:

#### 2.4.1 滑动窗口有什么不足？

滑动窗口已经很好了——没有边界漏洞，统计平滑。但它有一个缺点：**不允许突发。**

```
滑动窗口：每 60 秒最多 10 次
  请求均匀分布在 60 秒内 → 完美
  但前 1 秒来了 10 个请求 → 窗口瞬间用满 → 后面 59 秒的请求全部被拒绝
  → 业务上可能不合理（比如刚发布了新内容，1 秒内需要密集推送消息）
```

**令牌桶解决的就是这个问题：允许短时突发，但平滑长期速率。**

#### 2.4.2 什么是令牌桶——用水桶类比来理解

想象一个水桶（令牌桶），一个人（令牌生成器）以固定速度往桶里放令牌：

```
令牌生成器：每 0.1 秒放 1 个令牌（= 每秒 10 个）

  ┌──────────┐
  │ 令牌生成器 │  速率：1 个/0.1 秒 = 10 个/秒
  └────┬─────┘
       │ 匀速放令牌
       ▼
  ┌──────────┐
  │          │
  │  令牌桶   │  容量：最多存 10 个令牌（容量的意义见下文）
  │  [][][][]│  ← 目前桶里有 4 个令牌
  │          │
  └────┬─────┘
       │
  ┌────▼─────┐
  │  请求来了  │  每个请求想要通过，必须从桶里拿 1 个令牌
  │ 拿 1 个令牌 │  桶里有 → 拿走，请求通过
  │          │  桶里没有 → 请求被限流
  └──────────┘

运转逻辑：
  空闲时：生成器持续放令牌 → 桶里攒满（最多 10 个）
  来请求时：拿走令牌 → 桶里令牌变少
  大量请求瞬间涌来：桶里攒的令牌被快速消耗 → 令牌耗光 → 后续请求限流
  请求过去后：生成器继续放 → 桶里慢慢恢复
```

**为什么桶容量 = rate（10）而不是更大？**

```
假设桶容量 = 1000（无限大）：
  系统空闲 100 秒 → 桶里攒了 1000 个令牌
  突然来了 1000 个请求 → 瞬间全部通过 → 后端承受 1000 QPS → 崩溃

假设桶容量 = 10（= rate）：
  系统空闲 100 秒 → 桶里最多只能攒 10 个令牌（多余的丢掉）
  突然来了 1000 个请求 → 前 10 个通过，第 11 个开始被限流
  → 后端最多承受 10 个的瞬时高峰 → 安全
```

**这是令牌桶的第一个精髓：桶容量 = rate，用于削峰，防止流量尖刺。**

### 内存占用和实现复杂度更低

| 维度           | 令牌桶                                                | 滑动日志窗口                               |
| :------------- | :---------------------------------------------------- | :----------------------------------------- |
| Redis 数据结构 | 1 个 key 存**令牌数 + 最后刷新时间**（Hash / String） | 1 个 ZSET，每个请求存一个 member           |
| 空间复杂度     | O(1)                                                  | O(窗口内请求数)，高并发时大量成员          |
| 每次请求操作   | 读取、计算、写回（几个 HINCRBY 或 Lua 原子操作）      | 删除过期 + 统计 + 插入新成员               |
| 清理机制       | 无需主动删除，计算时基于时间差补充令牌                | 每次都要 `ZREMRANGEBYSCORE` 删除窗口外数据 |

令牌桶只需记录“当前令牌数”和“上次生成令牌的时间”，每条请求消耗一个令牌，逻辑非常简单，不产生历史数据堆积。

#### 2.4.3 令牌桶 vs 漏桶——最容易混淆的两个概念

```
令牌桶（Token Bucket）：
  原理：匀速"生产令牌"，请求"消耗令牌"
  突发：允许——桶里攒了几个令牌，就可以瞬间过几个请求
  请求行为：主动拿令牌，拿不到就被限流
  适用场景：需要处理正常业务突发的场景（秒杀、大促、突发新闻推送）


漏桶（Leaky Bucket）：
  原理：请求以任意速度"涌入桶中"，桶以固定速率"漏水"（处理请求）
  突发：不允许——不管进来多少，必须以固定速率流出
  请求行为：被动等待被处理，超过桶容量要排队或丢弃
  适用场景：严格平滑流量（网络流量整形、ISP 限速）
```

**一句话区分：** 令牌桶 = "放东西进桶，请求拿走"；漏桶 = "请求进水桶，固定速度流出"。Redisson RRateLimiter 是令牌桶。

#### 2.4.4 用具体数字把令牌桶的每一秒走一遍

用真实的计算过程理解令牌桶。假设 rate=10 个/秒，interval=1000ms，桶容量=rate=10。

```
═══════════ 时刻 1：第一个请求（系统刚启动） ═══════════

Redis Hash 中的状态：
  rate      = 10
  interval  = 1000  (毫秒)
  value     = 0     (当前令牌数，初始为 0)
  timestamp = 0     (上次补充令牌的时间，初始为 0)

当前时间 now = 第 1000 毫秒（距启动过了 1 秒）

步骤 1：经过的时间 = now - timestamp = 1000 - 0 = 1000ms
步骤 2：这段时间应生成多少令牌？
        生成数 = floor(1000 / 1000 × 10) = 10 个
步骤 3：补充令牌 = min(0 + 10, 10) = 10（不能超过桶容量）
步骤 4：消耗 1 个 → value = 9
步骤 5：更新 timestamp = 1000

→ 请求通过！桶里还剩 9 个令牌


═══════════ 时刻 2：50 毫秒后又来一个请求 ═══════════

now = 1050ms

步骤 1：elapsed = 1050 - 1000 = 50ms
步骤 2：生成数 = floor(50 / 1000 × 10) = floor(0.5) = 0 个
        （不够 1 个令牌的生成时间，所以没生成）
步骤 3：value = min(9 + 0, 10) = 9
步骤 4：消耗 1 个 → value = 8

→ 请求通过！还剩 8 个


═══════════ 时刻 3：瞬间连续涌来 10 个请求 ═══════════

第 1 个请求: value=8 → 消耗 1 → 7  ✓
第 2 个请求: value=7 → 消耗 1 → 6  ✓
第 3 个请求: value=6 → 消耗 1 → 5  ✓
...
第 8 个请求: value=1 → 消耗 1 → 0  ✓  ← 最后一个令牌
第 9 个请求: value=0 → 不够！→ ✗ 限流
第 10 个请求: value=0 → 不够！→ ✗ 限流

8 个瞬间过，2 个被拒绝。这就是"允许短时突发"——桶里攒的令牌
可以一次性消耗完，但超过桶容量的瞬间流量会被限。


═══════════ 时刻 4：空闲 5 秒后又来一个请求 ═══════════

now = 6050ms

步骤 1：elapsed = 6050 - 1050 = 5000ms
步骤 2：生成数 = floor(5000/1000 × 10) = 50 个
步骤 3：value = min(0 + 50, 10) = 10
        ↑ 只补充到 10！多余的 40 个被丢了！
步骤 4：消耗 1 → value = 9

→ 请求通过！

关键发现：空闲时间再长，桶里最多只有 10 个令牌。
这就是桶容量=rate 防止空闲后"瞬间泄洪"的作用。
```

#### 2.4.5 Redisson RRateLimiter 的使用和底层

```java
// ═══════ 初始化限流器 ═══════
// "rate:sms" = Redis 中存储这个限流器状态的 key
RRateLimiter limiter = redisson.getRateLimiter("rate:sms");
// 每 1 秒生成 10 个令牌 → 等于 10 QPS
limiter.trySetRate(
    RateType.OVERALL,           // OVERALL=全局限流（所有实例共享）
                                // PER_CLIENT=每个实例单独计算
    10,                         // 速率
    1,                          // 间隔
    RateIntervalUnit.SECONDS);

// ═══════ 获取令牌 ═══════
// tryAcquire() → 立即返回
if (limiter.tryAcquire()) {
    sendSms();                                       // 拿到令牌了
} else {
    throw new RuntimeException("系统繁忙");          // 被限流
}

// tryAcquire(1, 3, SECONDS) → 等最多 3 秒
boolean ok = limiter.tryAcquire(1, 3, TimeUnit.SECONDS);
```

**底层在 Redis 中存的 Hash：**

```
KEY: "rate:sms" (Hash 类型)

  rate      = "10"               ← 令牌生成速率
  interval  = "1000"             ← 时间间隔（毫秒）
  type      = "0"                ← 0=OVERALL, 1=PER_CLIENT
  value     = "5"                ← 当前桶里还剩多少令牌
  timestamp = "1685012345678"    ← 上次补充令牌的时间
```

**每次 tryAcquire 时，发到 Redis 执行的 Lua 脚本（对应上面的算数过程）：**

```lua
-- KEYS[1] = "rate:sms"
-- ARGV[1] = 当前毫秒时间戳
-- ARGV[2] = 请求消耗的令牌数（通常为 1）

-- ① 从 Hash 里读状态
local rate = redis.call('hget', KEYS[1], 'rate')
local interval = redis.call('hget', KEYS[1], 'interval')
local value = redis.call('hget', KEYS[1], 'value')
local timestamp = redis.call('hget', KEYS[1], 'timestamp')

-- ② 算经过的时间，生成应得的令牌
local elapsed = ARGV[1] - timestamp
local generated = math.floor(elapsed / interval * rate)

-- ③ 补充令牌到桶里，不能超过上限 rate
value = math.min(value + generated, rate)

-- ④ 判断令牌是否够用
if value >= tonumber(ARGV[2]) then
    value = value - ARGV[2]               -- 消耗
    redis.call('hmset', KEYS[1],          -- 写回
        'value', value,
        'timestamp', ARGV[1])
    return 1  -- 放行
else
    redis.call('hset', KEYS[1], 'value', value)
    redis.call('hset', KEYS[1], 'timestamp', ARGV[1])
    return 0  -- 限流
end
```

### 2.5 限流方案完整选型

| 方案 | 原理 | 是否允许突发 | 精度 | 复杂度 | 适用场景 |
|------|------|------------|------|--------|---------|
| 固定窗口 INCR | 每分钟重置计数器 | 临界漏洞（边界翻倍）| 粗 | 极低 | 对精度不敏感 |
| ZSet 滑动窗口 | 时间戳排序，动态窗口 | 无 | 毫秒级 | 中 | 需要严格精确限流 |
| 令牌桶（Redisson）| 匀速生成令牌，桶内可攒 | 允许（上限=rate）| 毫秒级 | 低（封装完）| **生产推荐** |
| Guava RateLimiter | 本地内存令牌桶 | 允许 | 微秒级 | 极低 | 仅单机限流 |

---

## 3. 消息队列

### 3.1 最简单的消息队列——List

```java
// ===== 生产者：3 行代码 =====
// LPUSH 往列表左边推入消息
jedis.lpush("queue:order", orderJson);  // "queue:order" 是队列名

// ===== 消费者：循环 BRPOP =====
while (true) {
    // BRPOP：阻塞弹出（Blocking RPOP）
    // 参数：key=队列名, timeout=0（0=无限阻塞，有数据才返回）
    // 返回：List<String> → [0]=key名, [1]=消息内容
    List<String> result = jedis.brpop(0, "queue:order");
    if (result != null) {
        String body = result.get(1);  // result[0] = "queue:order", result[1] = 实际消息
        processOrder(body);           // 处理订单
    }
}
```

**这个方案能跑，但它有 5 个致命问题：**

```
问题 1：无 ACK —— BRPOP 弹出即删除，消费者处理到一半崩了 → 消息永久丢失
问题 2：无持久化 —— Redis 重启 → 所有消息丢失
问题 3：不能一对多 —— 一条消息只能被一个消费者拿走（不能广播）
问题 4：无消费组 —— 没法让多个消费者自动分摊消息
问题 5：无消息回溯 —— 消息"弹出去就没了"，不能回头重新读
```

### 3.2 Redis Stream——带 ACK 的轻量级消息队列

Redis 5.0 引入 Stream，对标 Kafka 的消费模型。它的核心概念：

```
Stream（流）= 消息的 append-only 日志          ← 类似 Kafka Topic
消息 ID     = <毫秒时间戳>-<序号>               ← 如 1685012345678-0
消费组      = 一组消费者共享消费进度             ← 类似 Kafka Consumer Group
PEL        = Pending Entries List，已投递但未 ACK 的消息
```

```
Stream: "mystream"
┌─────────────┬─────────────┬─────────────┬─────────────┐
│   msg 0-1   │   msg 0-2   │   msg 0-3   │   msg 0-4   │  ← 消息有序追加
│ {name:"zs"} │ {name:"ls"} │ {name:"ww"} │ {name:"zl"} │
└──────┬──────┴──────┬──────┴──────┬──────┴─────────────┘
       │             │             │
   ┌───▼──┐      ┌───▼──┐      ┌───▼──┐
   │消费组A│      │消费组B│      │消费组C│       ← 每个消费组独立消费全量消息
   │A1 A2 │      │B1    │      │C1 C2 │       ← 同组内消费者分摊消费
   └──────┘      └──────┘      └──────┘
```

#### Java 实现

```java
@Autowired
private StringRedisTemplate redisTemplate;  // Spring Boot 自动配置

// ════════════════════ 生产者 ════════════════════
public String sendMsg(String streamKey, Map<String, String> message) {
    // ① 把 Map 包装成 Stream 记录
    StringRecord record = StreamRecords.string(message)
        .withStreamKey(streamKey);              // 指定写入哪个 Stream

    // ② XADD streamKey * key1 val1 key2 val2 ...
    // * = 让 Redis 自动生成消息 ID
    RecordId id = redisTemplate.opsForStream().add(record);

    // ③ 返回消息 ID（如 "1685012345678-0"）
    return id.getValue();
}

// ════════════════════ 消费者组初始化（只执行一次）════════════════════
// XGROUP CREATE streamKey groupName $ MKSTREAM
// $ = 仅消费创建后的新消息（不消费旧消息）
// MKSTREAM = stream 不存在则自动创建
redisTemplate.opsForStream().createGroup(streamKey, groupName);

// ════════════════════ 消费者循环 ════════════════════
String groupName = "order-consumers";     // 消费组名
String consumerName = "instance-1";       // 本消费者名（同组内唯一）

while (running) {
    // ① XREADGROUP GROUP order-consumers instance-1
    //          COUNT 10 BLOCK 2000 STREAMS mystream >
    //    ">" = 拉取本消费组"未消费过"的新消息
    //    BLOCK 2000 = 没有消息时阻塞 2 秒（长轮询，不空转 CPU）
    //    COUNT 10 = 一次最多取 10 条
    List<MapRecord<String, Object, Object>> messages =
        redisTemplate.opsForStream().read(
            Consumer.from(groupName, consumerName),
            StreamReadOptions.empty()
                .count(10)                                // 每次最多 10 条
                .block(Duration.ofSeconds(2)),            // 长轮询 2 秒
            StreamOffset.create(streamKey, ReadOffset.from(">"))
        );

    // ② 逐条处理
    for (MapRecord<String, Object, Object> msg : messages) {
        try {
            process(msg.getValue());                        // 处理业务

            // ③ 处理成功 → XACK 确认
            // 对应 Redis：XACK mystream order-consumers msgId
            // 告诉 Redis："这条消息处理完了，从我 PEL 中移除"
            redisTemplate.opsForStream()
                .acknowledge(streamKey, groupName, msg.getId());

        } catch (Exception e) {
            // ④ 处理失败 → 不 ACK
            // 消息留在 PEL 中 → 其他消费者可通过 XCLAIM 认领
        }
    }
}
```

### 3.3 PEL——消息可靠性的核心

消费者拿到消息后，消息进入 PEL（Pending Entries List）。消息可靠性的三种状态：

```
状态 1：正常消费
  拿消息 → 处理 → ACK → 从 PEL 移除 ✓

状态 2：消费者崩溃
  拿消息 → 崩溃（没 ACK） → 消息留在 PEL，idle 时间持续增长
  → 其他消费者 XAUTOCLAIM 认领 → 重新投递 ✓

状态 3：处理失败
  拿消息 → 抛异常 → 不 ACK → 留在 PEL → 重试/人工处理
```

```java
// XAUTOCLAIM：自动认领其他消费者的死消息
// idle 超过 30 秒 → 视为原消费者已死 → 新消费者认领
redisTemplate.opsForStream().claim(
    streamKey, groupName, consumerName,
    Duration.ofSeconds(30),     // idle 超过 30 秒才认领
    msgIds);
```

### 3.4 Stream vs List vs Kafka 选型

| 需求 | 推荐 |
|------|------|
| 丢了无所谓、一个消费者、简单队列 | List（BRPOP） |
| 需要 ACK、多个消费者、不想部署新中间件 | Redis Stream |
| 需要严格持久化、TB 级数据、Connector 生态 | Kafka |

---

## 4. 延时队列

### 4.1 问题：什么叫延时队列？:rocket::rocket::rocket::rocket::rocket::rocket:

```
不是"来了就处理"，而是"来了等 N 秒再处理"。

场景：
  - 下订单后 30 分钟未支付 → 取消订单
  - 发送短信验证码 5 分钟后 → 使验证码失效
  - 评论发布后 3 秒 → 通知审核系统
```

### 4.2 ZSet 实现——score 就是执行时间

**原理：** `score = 执行时间戳`（当前时间 + 延迟）。消费者定时轮询"当前时间 ≥ score"的记录。

```java
// ═════════ 生产者：投递延时任务 ═════════
public void addDelayJob(String queueKey, String jobData, long delayMs) {
    // 执行时间 = 当前时刻 + 延迟
    long executeTime = System.currentTimeMillis() + delayMs;

    // ZADD delay:queue {executeTime} {jobData}
    // score = 执行时间戳（未来某个时刻），member = 任务数据
    jedis.zadd(queueKey, executeTime, jobData);
}

// ═════════ 消费者：轮询到期任务 ═════════
public void consumeDelayJob(String queueKey) {
    while (true) {
        long now = System.currentTimeMillis();

        // ZRANGEBYSCORE delay:queue 0 {now} LIMIT 0 10
        // 取出 score 在 [0, 当前时间] 区间内的元素
        // score ≤ now = "该执行了"
        Set<String> jobs = jedis.zrangeByScore(queueKey, 0, now, 0, 10);

        for (String job : jobs) {
            // ZREM：从 ZSet 中真正删除
            // 返回 1 → 当前消费者成功认领
            // 返回 0 → 已被其他消费者认领（并发消费的防护）
            if (jedis.zrem(queueKey, job) > 0) {
                process(job);
            }
        }

        Thread.sleep(500);  // 轮询间隔（别 < 100ms，避免高频查 Redis）
    }
}
```

**ZSet 方案的优缺点：**
- 优点：零依赖，毫秒精度，代码简洁
- 缺点：需要轮询（有 CPU 开销），无 ACK 机制，大量到期消息时 ZRANGEBYSCORE 可能慢

### 4.3 Redisson RDelayedQueue——封装好的延时队列

Redisson 帮你把轮询逻辑封装了：

```java
// ═════════ 初始化 ═════════
// RBlockingQueue = 目标队列（消费者从这取）
// RDelayedQueue  = 延时队列（生产者往这投，到期后自动转移到目标队列）
RBlockingQueue<String> targetQueue = redisson.getBlockingQueue("delay:queue");
RDelayedQueue<String> delayedQueue = redisson.getDelayedQueue(targetQueue);

// ═════════ 生产者：投递延时任务 ═════════
delayedQueue.offer("job-123", 10, TimeUnit.SECONDS);   // 10 秒后消费者才能 take
delayedQueue.offer("job-456", 1, TimeUnit.HOURS);      // 1 小时后消费者才能 take

// ═════════ 消费者：阻塞等待 ═════════
while (true) {
    String job = targetQueue.take();  // 阻塞等待（底层是 BRPOP）
    process(job);
}
```

**Redisson 内部做了什么：**

```
生产时：ZADD redisson_delay_queue_{key} <执行时间戳> <序列化数据>

后台任务（每 100ms）：
  ① ZRANGEBYSCORE ... 0 NOW() LIMIT 100  ← 找出到期数据
  ② RPUSH redisson_delay_queue_channel_{key} <data>  ← 转移到 List
  ③ ZREM  ← 从 ZSet 删除

消费时：targetQueue.take() → BRPOP channel → 拿到数据
```

**局限：** 后台扫描间隔 100ms，所以精确度在百毫秒级（达不到毫秒级精确）。

### 4.4 生产级方案：ZSet + MQ 混搭:rocket::rocket::rocket::rocket::rocket:

```
Redis ZSet（延时存储和排序）
      │
      │ 定时任务扫描到期 → 投递到 MQ
      ▼
RocketMQ / Kafka（可靠投递 + ACK + 消费组）
      │
      ▼
消费者处理
```

**为什么这样设计？** ZSet 擅长"按时间排序暂存"（score 天然是时间索引），MQ 擅长"可靠投递 + ACK + 消费组"。两者各取所长。

---

## 5. 分布式 Session

### 5.1 问题：Tomcat 的 Session 在分布式下失效

```
传统 Session（Tomcat 内置）：
  请求-1 → Nginx → 服务器A → Session 存在 A 的 JVM 内存
  请求-2 → Nginx → 服务器B → "你是谁？"（B 没有你的 Session）
  → 用户被踢出登录 → 体验极差
```

三种解决方案：

| 方案 | 做法 | 问题 |
|------|------|------|
| IP Hash:rocket::rocket::rocket::rocket: | Nginx 把同一 IP 永远路由到同一台服务器 | 扩容缩容破坏粘性；服务器宕机 = 该 IP 的所有用户全掉 |
| Session 复制 | Tomcat 集群间互相同步 Session | 网络开销大；内存冗余（每个节点存全量 Session） |
| **Redis 集中存储** | **Session 存 Redis，所有服务器共享** | **多一次网络调用；但架构干净** |

**IP Hash**:rocket::rocket::rocket::rocket::rocket:

### 基本原理

1. **获取客户端 IP**
   比如 `192.168.1.100`。
2. **计算哈希值**
   用某种哈希算法（CRC32、MD5、FNV 等）把 IP 字符串转换为一个数字。
3. **取模映射到节点**
   假设有 N 个服务器，用 `hash % N` 得到 0 ~ N-1 的编号，决定路由到哪台服务器。

> **公式：** `target_server = hash(客户端IP) % 节点总数`

这样，同一个 IP 的请求会被**始终路由到同一台后端服务器**，直到节点数量发生变化。

###  IP Hash 的缺点和问题

#### 4.1 负载不均衡

哈希函数并不能保证绝对均匀。如果少数 IP 发送大量请求（如公司出口网关、代理服务器），会造成**单点过热**，某台服务器压力巨大而其他服务器空闲。

#### 4.2 后端节点变化导致重映射

当添加或移除服务器时，`hash % N` 的 `N` 改变，会导致**大量用户重新分配到不同服务器**，所有 Session 全部丢失（发生“缓存雪崩”）。

> 例如原本 3 台服务器，`hash(ip) % 3 = 1` 路由到 server2。当增加第 4 台时，取模变为 `% 4`，结果很可能变成另外一台，导致用户 Session 失效。

#### 4.3 NAT 与代理问题

- 很多用户通过同一个出口 IP 上网（如学校、公司、运营商 NAT），这些用户会被误认为同一个 IP，全部路由到同一台服务器，加剧负载不均。
- 使用正向代理或 CDN 时，后端看到的 IP 可能是代理的 IP，而不是真实客户端 IP。此时需要正确获取 `X-Forwarded-For` 或 `X-Real-IP` 头。
- 演进：一致性hash

###  一致性哈希的核心思想:rocket::rocket::rocket::rocket::rocket:

不再将 IP 和服务器做**线性取模**，而是把**服务器节点**和**数据 key（如 IP）**都哈希到一个**环形空间**（哈希环）上，数据按照顺时针方向找到离自己最近的节点。

#### 哈希环（Hash Ring）

- 选择一个固定的整数范围，比如 `0` ~ `2^32 - 1`（0 到大约 43 亿），首尾相连形成一个圆。
- 每个服务器节点通过相同的哈希算法（如 MD5、FNV）计算出一个哈希值，映射到环上的某个点。
- 每个请求的 key 也计算出哈希值，落在环上。
- **路由规则**：从 key 的位置出发，顺时针遇到的第一个节点就是负责处理该 key 的服务器。

#### 示例

text

```
环范围：[0, 2^32 - 1]  (顺时针方向)
节点：
  Server A 哈希在 1000 位置
  Server B 哈希在 5000 位置
  Server C 哈希在 9000 位置

请求 key1 哈希在 4000 → 顺时针遇到 B → 分配给 Server B
请求 key2 哈希在 9500 → 顺时针越过最大值回到 0，遇到 A → 分配给 Server A
```



------

### 动态增减节点：只影响局部数据

#### 增加节点

假设在 `Server A (1000)` 和 `Server B (5000)` 之间新增 `Server D`，哈希在 `3000`：

- 原本落在 `(1000, 5000]` 区间内、被分配给 B 的请求，现在 `(1000, 3000]` 范围内的请求会重新分配给 D。
- 其他区间（如 `(5000, 9000]` 和 `(9000, 1000]`）的数据映射完全不变。
- **仅影响相邻节点之间的一小段**。

#### 移除节点

`Server B (5000)` 宕机：

- 原本分配给 B 的请求（落在 `(1000, 5000]`）将自动顺时针移动到下一个节点 `C (9000)`。
- 其他节点数据无影响。

这就把节点变更的影响降到了最低，是分布式缓存、负载均衡等场景的基石。



redis集中存储

### 5.2 Spring Session + Redis 实现

```java
// ===== 第一步：依赖 =====
// pom.xml：
//   spring-session-data-redis
//   spring-boot-starter-data-redis

// ===== 第二步：配置 =====
// application.yml
spring:
  session:
    store-type: redis                  # 告诉 Spring：Session 存 Redis
    redis:
      namespace: spring:session       # Redis key 前缀

// ===== 第三步：启用 =====
@Configuration
// maxInactiveIntervalInSeconds=1800 → 30 分钟无操作自动过期
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        // 用 JSON 序列化存 Redis（可读性好）
        return new GenericJackson2JsonRedisSerializer();
    }
}

// ===== 第四步：Controller 和普通 HttpSession 写法完全一样 =====
@RestController
public class UserController {

    @PostMapping("/login")
    public String login(@RequestParam String username, HttpSession session) {
        // session.setAttribute → Spring Session 拦截 → 存 Redis
        // Controller 完全不知道底层是 Redis 还是 Tomcat 内存
        session.setAttribute("user", username);
        return "ok";
    }

    @GetMapping("/user")
    public String currentUser(HttpSession session) {
        // session.getAttribute → Spring Session 拦截 → 从 Redis 读
        // 不管 Nginx 把请求路由到哪台机器，都能拿到 Session
        return (String) session.getAttribute("user");
    }
}
```

**Redis 中实际存的 key：**

```
spring:session:sessions:abc-123-def   ← Hash（存的属性、创建时间、过期时间）
spring:session:sessions:expires:abc.. ← String（过期标记）
```

### 5.3 安全提醒

```java
// ❌ 不要存敏感数据
session.setAttribute("password", rawPassword);  // Redis 明文暴露

// ✅ 只存用户标识
session.setAttribute("userId", userId);

// ✅ 如果 Redis 宕机——降级方案
// 1. Redis 做高可用（主从 + Sentinel / Cluster）
// 2. 应用层用 JWT 做无状态认证（不依赖 Session）
```

---

## 6. 复杂业务场景

### 6.1 Bitmap —— 日活用户统计（DAU）

**问题：** 统计每天有多少不同用户登录了。MySQL `COUNT(DISTINCT user_id)` 在亿级数据上要几秒；Redis Set 精确但要几 GB 内存。

**Bitmap 怎么做：** 一个 String = 512MB 最大 = 2³² bit = 可以标记 42 亿不同用户。每一位代表一个用户：1 = 活跃，0 = 不活跃。

```
用户 userId=10086 今天活跃：
  SETBIT dau:2026-05-25 10086 1     ← 第 10086 位设为 1
用户 userId=20480 今天活跃：
  SETBIT dau:2026-05-25 20480 1

今天活跃总人数：
  BITCOUNT dau:2026-05-25            ← 统计所有 1 的位数

7 天连续活跃（BITOP AND 交集）：
  BITOP AND dau:week dau:05-19 dau:05-20 ... dau:05-25
  BITCOUNT dau:week                  ← 7 天每天都活跃的用户数
```

**内存对比（1 亿用户）：**
- MySQL：user_id(8B) × 1亿 = 800MB（仅数据，不算索引）
- Redis Set：1亿 × ~50B（redisObject+SDS+dictEntry）≈ 5GB
- **Redis Bitmap：1亿 bit = 12.5MB**

```java
@Component
public class DauService {
    @Autowired
    private StringRedisTemplate redisTemplate;

    /** 标记用户今天活跃 */
    public void markActive(long userId) {
        String key = "dau:" + LocalDate.now();               // dau:2026-05-25
        redisTemplate.opsForValue().setBit(key, userId, true);// 第 userId 位=1
        redisTemplate.expire(key, 32, TimeUnit.DAYS);         // 32天自动清
    }

    /** 今日活跃用户数 */
    public long todayDAU() {
        String key = "dau:" + LocalDate.now();
        // 执行 BITCOUNT（Spring Data Redis 没有直接包装，用底层连接）
        return redisTemplate.execute(
            (RedisCallback<Long>) conn -> conn.bitCount(key.getBytes()));
    }
}
```

### 6.2 HyperLogLog —— 网站 UV 统计

**问题：** UV 需要去重——同一个用户访问 100 次只算 1 个 UV。Set 精确但内存随 UV 线性增长。

**HyperLogLog：** 概率统计，误差 0.81%，**固定 12KB 内存**可以统计 2⁶⁴ 个不同元素。

```
内存对比（1 亿 UV）：
  Set：约 6-8GB
  HyperLogLog：固定 12KB  ← 1/500,000

但代价是：
  PFADD → 只返回 1 或 0（是否有"新"元素，但不是精确判断）
  PFCOUNT → 约 0.81% 误差
  不能列出具体是哪些人（只给数字）
```

**为什么 12KB 就够了？** 16384 个桶，每个桶 6 bit = 16384×6/8 = 12KB。

```
一个元素 → 64-bit hash → 前 14 位 = 桶号(0~16383) → 后 50 位中"最大连续0个数"存入桶
16384 个桶取调和平均数 → × 常数校准 → 估算总数
误差 = 1.04 / √16384 ≈ 0.81%
```

```java
// ═══════ 记录 UV ═══════
// PFADD uv:page:123:2026-05-25 {userId}
redisTemplate.opsForHyperLogLog().add("uv:page:123:" + LocalDate.now(),
    String.valueOf(userId));

// ═══════ 查询 UV ═══════
// PFCOUNT uv:page:123:2026-05-25 → 估算的去重访问人数
long uv = redisTemplate.opsForHyperLogLog()
    .size("uv:page:123:" + LocalDate.now());

// ═══════ 合并两个页面的去重 UV ═══════
// PFMERGE dest key1 key2
redisTemplate.opsForHyperLogLog().union("uv:merged",
    "uv:page:123", "uv:page:456");
```

**选型：** 需要精确 → Set；允许 0.81% 误差 → HyperLogLog；需要知道"哪些人" → Set 或 DB。

### 6.3 ZSet —— 实时排行榜

**ZSet 是排行榜的自然选择——score 天然就是排名依据。**

```java
// ═══════ 加分 ═══════
// ZINCRBY rank:game:daily {score} user:{userId}
redisTemplate.opsForZSet()
    .incrementScore("rank:game:daily", "user:" + userId, 200);

// ═══════ Top 10 ═══════
// ZREVRANGE rank:game:daily 0 9 WITHSCORES
Set<ZSetOperations.TypedTuple<String>> top10 =
    redisTemplate.opsForZSet()
        .reverseRangeWithScores("rank:game:daily", 0, 9);

// ═══════ 查排名 ═══════
// ZREVRANK rank:game:daily user:1001（从高到低排，0-based → +1 展示用）
Long rank = redisTemplate.opsForZSet()
    .reverseRank("rank:game:daily", "user:1001");
long displayRank = rank == null ? 0 : rank + 1;  // 第 3 名

// ═══════ 查分数 ═══════
// ZSCORE rank:game:daily user:1001（O(1)，查 dict）
Double score = redisTemplate.opsForZSet()
    .score("rank:game:daily", "user:1001");
```

**多维度综合排行榜（阅读 0.3 + 点赞 0.5 + 评论 0.2）：**

```java
// ZUNIONSTORE rank:hot:daily 3
//   rank:reads rank:likes rank:comments
//   WEIGHTS 0.3 0.5 0.2 AGGREGATE SUM
String lua =
    "redis.call('ZUNIONSTORE', KEYS[1], 3, " +
        "KEYS[2], KEYS[3], KEYS[4], " +
        "'WEIGHTS', 0.3, 0.5, 0.2, 'AGGREGATE', 'SUM') " +
    "redis.call('EXPIRE', KEYS[1], 86400) " +    // 24h 过期
    "return redis.call('ZREVRANGE', KEYS[1], 0, 99, 'WITHSCORES')";

jedis.eval(lua,
    Arrays.asList("rank:hot:daily", "rank:reads", "rank:likes", "rank:comments"),
    Collections.emptyList());
```

**性能：** ZADD/ZINCRBY O(logN)，百万用户榜单 Top 100 查询 O(logN+100) ≈ 微秒级。不管多少用户，Top 10 永远瞬间返回。

---

## ⭐️ 综合面试题

**Q1: Redisson 看门狗解决什么问题？**
> lock() 不找 leaseTime 时启动看门狗——每 10 秒重置 TTL 为 30 秒。只要业务在跑锁永不过期，避免"业务没跑完锁过期"的并发风险；如果程序崩溃看门狗没了，锁 30 秒后自动过期，防止死锁。指定了 leaseTime（如 tryLock）则不开看门狗。

**Q2: 令牌桶和漏桶的区别？**
> 漏桶：固定速率流出，超过桶容量丢弃 → 严格平滑，不能处理突发。令牌桶：固定速率生成令牌，桶可以攒令牌（上限=rate）→ 允许短时突发 ≤rate。Redisson RRateLimiter 是令牌桶。

**Q3: List 做消息队列有什么问题？**
> 无 ACK（消费者崩溃 → 消息丢失）、无消费组、不能一对多、消息无法回溯、无持久化保证。Stream 解决了大部分（ACK+PEL+消费组），严肃场景上 Kafka/RocketMQ。

**Q4: 1 亿 DAU，Bitmap vs Set vs MySQL？**
> Bitmap 12.5MB → Set 5GB → MySQL 800MB+。Bitmap 省内存近 1000 倍，但不存"谁是谁"的详细信息（仅 0/1）。Bit 操作 O(1)，BITOP AND 做连续活跃极其高效。

**Q5: HyperLogLog 为什么误差 0.81%？**
> 标准误差 = 1.04/√m，m=16384 桶。1.04/√16384 = 0.81%。桶越多误差越小但内存越大——12KB 是 Redis 选择的折中点。需要精确用 Set，需要省内存用 HLL。

**Q6: ZSet 排行榜在百万用户时性能如何？**
> ZADD O(logN) ≈ 20 次跳表比较 → 微秒级。ZREVRANGE 0 9 O(logN+10) → 微秒级。ZSCORE O(1) → 纳秒级。百万用户毫无压力，瓶颈在网络带宽而非 Redis。

---

*Created: 2026-05-25 · Updated: 从浅入深全量重写 · Category: 04-Redis应用*
