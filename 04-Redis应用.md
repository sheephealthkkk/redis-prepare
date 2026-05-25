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

**致命问题：** 拿到锁后如果服务宕机，`unlock` 代码没机会执行 → key 永久存在 → **死锁**。

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

**致命问题：SETNX 和 EXPIRE 是两条独立命令，中间可能宕机。** 一个命令执行完，第二个命令还没发出去，进程被 kill -9 → 锁设置了但没有过期时间 → 死锁。

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

**新问题——误删别人的锁：**

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

#### V4：释放时验证身份（Lua 原子脚本）

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

#### V1-V4 还没解决的问题：过期时间到底设多长？

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

**关键区别：** `lock()` 启动看门狗，锁永不过期（只要你还在跑）。`tryLock(wait, leaseTime, unit)` 指定了 leaseTime，**不启动看门狗**，到期就释放。

#### 1.3.2 Redisson 用 Hash 存锁——为什么？

Redisson 不是用 `SET lockKey value NX PX` 存锁，而是用 **Hash**：

```
Redis 中的实际存储：
  Key:   lock:order:12345               ← Hash 类型
  Field: "abc123-def456:thread-1"       ← 客户端唯一标识
  Value: 1                              ← 可重入计数
```

**为什么用 Hash？两个原因：**

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

**这段脚本解释了三件事：**
1. 锁是 Hash，field 是客户端标识，value 是重入计数
2. 同线程重复 lock() 不会死锁（可重入）
3. 返回 TTL 让等待方知道 "大概还要等多少毫秒"，不用盲目轮询

#### 1.3.4 看门狗（Watchdog）—— 自动续期

看门狗是 Redisson 的杀手级特性。它解决的核心矛盾是：**"不设过期怕死锁，设了过期怕没执行完"**。

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
// 全局 Map：存储每个锁对象的续期任务
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
            // if redis.call('hexists', KEYS[1], ARGV[2]) == 1 then
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

加锁失败后，Redisson 不是 `while(true) + sleep` 那种 CPU 空转，而是 **Pub/Sub + 信号量**：

```
① tryLock(3秒) → Lua 加锁失败 → 返回剩余 TTL（如 25 秒）
② 订阅 Redis channel："redisson_lock__channel:{lock:order:12345}"
③ Java 线程在 Semaphore 上阻塞等待（无 CPU 消耗），超时时间 = min(waitTime, TTL)
④ 持有锁的客户端 unlock → publish 信号 → 当前客户端被唤醒 → 重新抢锁
⑤ 3 秒内抢到了 → 返回 true；超时 → 返回 false
```

### 1.4 红锁（RedLock）—— 主从切换的隐患

**主从异步复制带来的问题：**

```
① 客户端 A → Master：SET lock NX PX ✓ （拿到锁）
② Master 还没来得及把这条数据同步给 Slave → Master 宕机
③ Sentinel 把 Slave 提升为新 Master
④ 客户端 B → 新 Master：SET lock NX PX ✓ （也拿到锁了！）
   → A 和 B 都以为自己持有锁 → 并发安全问题！
```

**RedLock 算法（Redis 官方方案）：**

部署 5 个独立的 Redis 节点（不是主从，各管各）。获取锁时需要向大多数节点（≥3）成功获取才算成功：

```java
RLock lock = redisson.getRedLock(
    redisson.getLock("lock1"),  // 节点 1
    redisson.getLock("lock2"),  // 节点 2
    redisson.getLock("lock3"),  // 节点 3
    redisson.getLock("lock4"),  // 节点 4
    redisson.getLock("lock5")   // 节点 5
);
lock.lock();  // 内部分别向 5 个节点发 SET NX PX，≥3 成功即返回
```

**业界实际态度：** 绝大多数业务用单节点 + Sentinel 足够。真需要强一致性锁的极端场景（如金融资金），用 ZooKeeper（CP 系统）或 etcd（Raft 协议）。

### 1.5 分布式锁面试要点

- **为什么用 Hash 不用 String？** → 可重入计数 + 身份验证
- **看门狗解决什么问题？** → 不设过期死锁 vs 设过期不够长的矛盾
- **tryLock 的等待是轮询吗？** → 不是，Pub/Sub 事件通知 + Semaphore
- **红锁解决什么问题？** → 主从异步复制的脑裂问题

---

## 2. 限流

### 2.1 先理解问题：为什么需要分布式限流？

假设短信接口 QPS 上限 1000，部署了 3 个实例：

```
单机限流（Guava RateLimiter）的问题：
  实例 A → 限 1000/s → 实际 3 实例 = 3000/s → 后端被打爆
  实例 A → 限 333/s → 单个实例的能力又没发挥出来

分布式限流：
  所有实例去 Redis 同一个 key 做计数 → 真正的全局 1000/s
```

### 2.2 固定窗口限流——最简单也最危险

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

**固定窗口的漏洞——临界问题：**

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
public boolean slidingWindow(String userId, int limit, int windowSec) {
    String key = "rate:sliding:" + userId;     // Redis key
    long now = System.currentTimeMillis();      // 当前时间戳（毫秒）
    long windowStart = now - windowSec * 1000L; // 窗口起点 = 60 秒前

    String lua =
        // ① 删除窗口外的旧数据（score 在 0 ~ windowStart 之间的全部删掉）
        "redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1]) " +

        // ② 统计窗口内还剩几条请求
        "local count = redis.call('ZCARD', KEYS[1]) " +

        // ③ 判断：窗口内请求数 < 限制？
        "if count < tonumber(ARGV[2]) then " +
        "    redis.call('ZADD', KEYS[1], ARGV[3], ARGV[4]) " +  // ④ 是 → 记录本次请求
        "    redis.call('EXPIRE', KEYS[1], ARGV[5]) " +          // ⑤ 设 key 过期
        "    return 1 " +                                         // ⑥ 放行
        "else " +
        "    return 0 " +                                         // ⑦ 拒绝
        "end";

    String memberId = UUID.randomUUID().toString();  // 本条请求的唯一标识
    Object result = jedis.eval(lua,
        Collections.singletonList(key),        // KEYS[1]
        Arrays.asList(
            String.valueOf(windowStart),       // ARGV[1] = 窗口起点
            String.valueOf(limit),             // ARGV[2] = 10
            String.valueOf(now),               // ARGV[3] = 当前时间戳（score）
            memberId,                          // ARGV[4] = UUID（member）
            String.valueOf(windowSec + 1))     // ARGV[5] = TTL 61 秒
    );
    return "1".equals(result.toString());
}
```

**滑动窗口 VS 固定窗口：** 滑动窗口没有"窗口边界"，攻击者在任意 4 秒内的请求量都被平滑限制。代价：每次请求都要 ZREMRANGEBYSCORE（删旧数据）——但 ZSet 元素少时开销极小。

### 2.4 令牌桶——工业标准

令牌桶（Token Bucket）比滑动窗口更优雅——允许短时突发，但平滑长期速率。

```
令牌桶的直观理解：

  ┌─────────────┐
  │  令牌生成器   │ → 每秒生成 rate 个令牌放入桶
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐     ① 拿令牌
  │   令牌桶     │◄────────── 请求
  │  (容量=rate) │──────────▶
  └─────────────┘     ② 有令牌 → 放行
                      ③ 无令牌 → 拒绝

  桶容量 = rate（最多攒 1 秒的令牌）→ 允许瞬间来 rate 个请求（突发）
  但超过桶容量的令牌被丢弃 → 长期速率 ≤ rate
```

#### Redisson RRateLimiter

```java
// ========== 初始化：每秒 10 个令牌 ==========
RRateLimiter limiter = redisson.getRateLimiter("rate:sms");
limiter.trySetRate(
    RateType.OVERALL,           // OVERALL=全局限流，PER_CLIENT=单客户端限流
    10,                         // 速率：10 个
    1,                          // 间隔：1 秒
    RateIntervalUnit.SECONDS);  // → 10 个/秒

// ========== 使用 ==========
// tryAcquire()：尝试拿 1 个令牌，拿不到立即返回 false
if (limiter.tryAcquire()) {
    sendSms();          // 拿到令牌，发短信
} else {
    throw new RuntimeException("系统繁忙");  // 被限流
}

// tryAcquire(1, 3, SECONDS)：等最多 3 秒
boolean ok = limiter.tryAcquire(1, 3, TimeUnit.SECONDS);
```

**令牌桶的关键细节：**
- 桶容量 = rate（如 10），不是无限——防止空闲时攒下海量令牌，恢复流量瞬间打崩后端
- 底层存 Redis Hash：`rate`、`interval`、`value`（当前令牌数）、`timestamp`（上次补充时间）
- 每次 `tryAcquire` → Lua 脚本：计算经过时间 → 生成新令牌 → 判断够不够 → 消耗

### 2.5 限流方案选型

| 方案 | 适用 | 优点 | 缺点 |
|------|------|------|------|
| 固定窗口 INCR | 粗粒度限流 | 实现简单 | **临界问题**，可能被绕 |
| ZSet 滑动窗口 | 精确限流 | 无临界问题 | 每次请求要 ZREMRANGEBYSCORE |
| Redisson RRateLimiter | 生产推荐 | 令牌桶语义，允许突发 | Redisson 依赖 |
| Guava RateLimiter | 单机限流 | 纳秒级延迟 | 不管集群 |

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

### 4.1 问题：什么叫延时队列？

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

### 4.4 生产级方案：ZSet + MQ 混搭

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
| IP Hash | Nginx 把同一 IP 永远路由到同一台服务器 | 扩容缩容破坏粘性；服务器宕机 = 该 IP 的所有用户全掉 |
| Session 复制 | Tomcat 集群间互相同步 Session | 网络开销大；内存冗余（每个节点存全量 Session） |
| **Redis 集中存储** | **Session 存 Redis，所有服务器共享** | **多一次网络调用；但架构干净** |

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
