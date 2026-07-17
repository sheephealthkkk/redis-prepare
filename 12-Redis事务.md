# Redis 事务 —— 深度详解

---

pipelinr,transation,lua

## 前言：Redis 事务和 MySQL 事务完全不是一回事

很多 Java 程序员对"事务"的第一反应是 MySQL 的 ACID：要么全成功，要么全回滚。如果你带着这个预期去看 Redis 事务，会非常困惑。

```
核心差异一句话：

  MySQL 事务：    要么全做，要么全不做（原子性 + 回滚）
  Redis 事务：    打包执行，但不支持回滚！
```

**Redis 事务的本质是：把一组命令排队，一次性按序执行，执行期间不被其他客户端的命令打断。**:rofl::rofl::rofl::rofl::rofl:

---

## 第一章：基本事务 —— MULTI / EXEC / DISCARD:o::o::o::o::o::o:

### 1.1 事务的三个命令

| 命令 | 作用 |
|------|------|
| `MULTI` | 开启事务（开始排队模式） |
| `EXEC` | 执行事务（把排队的所有命令一次性执行完） |
| `DISCARD` | 放弃事务（清空排队的所有命令，不执行） |

### 1.2 事务的基本流程

```
      Client                                Redis
        │                                     │
        │──── MULTI ────────────────────────▶ │  ① 开启事务模式
        │                                     │
        │──── SET k1 v1 ───────────────────▶ │  ② 命令不执行，只是加入「命令队列」
        │◄──────────────────────── QUEUED ─── │
        │                                     │
        │──── INCR counter ────────────────▶ │  ③ 命令不执行，继续排队
        │◄──────────────────────── QUEUED ─── │
        │                                     │
        │──── LPUSH list "a" ──────────────▶ │  ④ 命令不执行，继续排队
        │◄──────────────────────── QUEUED ─── │
        │                                     │
        │──── EXEC ────────────────────────▶ │  ⑤ 开始执行！
        │◄─────────────────── [OK, 1, 1] ── │     一次性执行所有排队的命令
        │                                     │     返回每条命令的执行结果
```

**关键理解：** MULTI 之后，每条命令 Redis 都**不执行**——只是返回 `QUEUED` 表示"已加入队列"。直到 EXEC 才真正执行，而且是一次性、不间断地执行完所有排队的命令。:rocket::rocket::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o:

### 1.3 Java 示例

```java
/**
 * Redis 基本事务示例
 * 所有写命令排队，EXEC 时一次性执行
 */
public void basicTransaction(Jedis jedis) {
    // ① 开启事务
    Transaction tx = jedis.multi();

    // ② 往队列里加命令（这些命令现在不执行，只是排队）
    tx.set("k1", "v1");          // 返回 "QUEUED"
    tx.incr("counter");          // 返回 "QUEUED"
    tx.lpush("list", "a");       // 返回 "QUEUED"

    // ③ 执行事务：一次性执行所有排队命令，返回每条命令的结果
    List<Object> results = tx.exec();
    // results = ["OK", 1, 1]
    //            SET    INCR  LPUSH
}
```

### 1.4 事务的原子性——不被其他命令打断

```
═══════════════════ 时间线 ═══════════════════

  Client-A                              Client-B
    │ MULTI                                │
    │ SET k1 v1 (排队)                     │ GET k1
    │ INCR counter (排队)                  │ → 等待！Client-A 正在 EXEC
    │ EXEC ────────┐                      │
    │              │ 执行 SET k1 v1       │
    │              │ 执行 INCR counter     │
    │              │ 中间不会被插入！       │
    │◄── ["OK", 1] │                      │
    │              └────────────────────── │ → 现在才能执行 GET k1
                                          │◄── "v1"

结论：EXEC 执行期间，其他客户端的命令不能插入。
     所有排队命令作为一个"不可分割的整体"执行。
```

### 1.5 事务中的错误处理——不！会！回！滚！:o::o:

这是 Redis 事务和 MySQL 事务最大的区别：:rocket:

```
情况 1：命令语法错误（MULTI 之前就检测到）

  127.0.0.1:6379> MULTI
  OK
  127.0.0.1:6379> SET k1 v1
  QUEUED
  127.0.0.1:6379> WRONG_COMMAND       ← 这个命令根本不存在
  (error) ERR unknown command
  127.0.0.1:6379> EXEC
  (error) EXECABORT Transaction discarded because of previous errors.
  
  → 整个事务被丢弃，一条都没执行 ✓


情况 2：命令语法正确，但运行时出错（这是坑！）

  127.0.0.1:6379> MULTI
  OK
  127.0.0.1:6379> SET k1 "hello"      ← k1 是 String 类型
  QUEUED
  127.0.0.1:6379> INCR k1             ← 对 String 执行 INCR？语法没问题，运行时报错！
  QUEUED
  127.0.0.1:6379> SET k2 "world"
  QUEUED
  127.0.0.1:6379> EXEC
  1) OK                               ← SET k1 成功了！
  2) (error) ERR value is not an integer  ← INCR k1 失败了...
  3) OK                               ← SET k2 还是成功了！
  
  → INCR 失败了，但 SET k1 和 SET k2 照样执行了！
  → Redis 事务不支持回滚！不会因为中间一条失败而回退前面的！
```

### 1.6 为什么 Redis 不实现事务回滚？——深度分析

这是面试中的高频追问。antirez（Redis 作者）在官方文档中给出了明确的解释：

**原因一（官方）：Redis 命令失败 = 编程错误，应该在开发阶段暴露**:o::o::o::o::o::o::o::o::o::o::o:

```
MySQL 事务回滚的场景：
  → 插入重复主键、外键约束失败、死锁检测... 这些是"运行时可能发生的"
  → 需要回滚来保证数据一致性

Redis 事务失败的原因：
  → 对 String 执行 INCR、对 List 执行 HGET... 这是类型错误
  → 类型错误 = 程序员写错了代码！不是"运行时偶然事件"！
  → 应该在测试阶段发现，而不是在生产中依赖回滚

对比：
  MySQL 的回滚场景是"合理的运行时异常"
  Redis 的事务失败是"写出 bug 了"
```

**原因二（深度）：实现回滚的代价太大**

```
如果 Redis 实现回滚，需要为每条命令维护"逆操作"：

  SET key value     →  逆操作：记住旧 value，失败时 SET 回去（或者 DEL）
  INCR key          →  逆操作：记住旧 value，失败时 SET 回去
  LPUSH key v       →  逆操作：记住 list 长度，失败时 LPOP/LTRIM
  SADD key member   →  逆操作：SREM key member
  ZADD key score m  →  逆操作：ZREM key member（或记住旧 score）
  
  这些"回滚信息"要在 EXEC 之前一直保存在内存中
  → 占用额外内存
  → 增加代码复杂度（数千行额外代码）
  → 和 Redis "极简、极快"的设计哲学违背

antirez："Redis is not a toy, it's simple. Simplicity is a feature."
```

**原因三：现实中的替代方案已经足够**

```
① 开发阶段：用类型检查避免错误
   - 确保 INCR 只操作整数类型的 key
   - 通过代码规范 + 单元测试保证

② 生产阶段：用 Lua 脚本做前置校验
   - Lua 中可以先检查 key 类型再操作
   - 类型不对 → 不执行 → 返回错误

③ 架构层面：用 Redis Stream + 消费者做最终一致性
   - 库存扣减失败 → 消息留在 PEL → 重试
   - 不是依赖回滚，而是依赖重试 + 幂等
```

### 1.7 Redis 事务到底有多少"ACID"？

面试中如果能从 ACID 四个维度分析 Redis 事务，会展现很深的理解力：:rofl::rofl::rofl::rofl:

```
A（原子性 Atomicity）：
  ✅ EXEC 执行期间，所有排队命令作为一个整体执行，中间不被插入
  ❌ 但如果某条命令执行失败，前面的不回滚，后面的继续执行
  → "打包执行" 但 "不打包回滚"

C（一致性 Consistency）：
  ❌ Redis 没有外键、约束、触发器... 无"一致性"概念
  → 数据的一致性由程序员保证，Redis 不管

I（隔离性 Isolation）：
  ✅ EXEC 执行期间，其他客户端的命令不插入
  ✅ 单线程执行 → 天然 Serializable 隔离级别

D（持久性 Durability）：
  ❌ 纯内存模式 → 无持久性（重启全丢）
  ⚠️ 开启 AOF everysec → 最多丢 1 秒
  ⚠️ 开启 RDB → 可能丢几分钟

总结一句话：
  Redis 事务只提供了"隔离性"和"打包执行的原子性"，
  不提供"回滚"和原生的"持久性"。
```

---

## 第二章：乐观锁 —— WATCH

## 第二章：乐观锁 —— WATCH

### 2.1 问题：事务之间怎么防止冲突？

Redis 事务 EXEC 期间是原子的（不会被插入），但 MULTI 到 EXEC 这段"排队时间"内，**其他客户端是可以执行命令的**。:rocket::rofl::rofl:

```
Client-A                          Client-B
  │                                  │
  │ GET counter → "10"              │
  │                                  │ GET counter → "10"
  │ MULTI                            │ MULTI
  │ INCRBY counter 5                 │ INCRBY counter 3
  │ EXEC → counter = 15              │ EXEC → counter = 13
  │                                  │
  ▼                                  ▼
  期望结果：counter = 18（10 + 5 + 3）
  实际结果：counter = 13（Client-B 覆盖了 A 的修改！）
```

### 2.2 WATCH —— Redis 的乐观锁:rocket::rocket:

**WATCH 是一种乐观锁机制：监视某个 key，如果在 EXEC 之前这个 key 被其他客户端改过，EXEC 就拒绝执行。**

```
WATCH 的工作流程：

  ① WATCH counter        ← 我开始"监视"counter
  ② GET counter → "10"   ← 读当前值
  ③ 基于读到的"10"做计算：准备 SET counter 15
  ④ MULTI
  ⑤ SET counter 15       ← 排队
  ⑥ EXEC  ← 如果 step②~⑥ 之间 counter 被其他客户端改过
              → EXEC 返回 null（拒绝执行！）
              如果 counter 没被改过
              → EXEC 正常执行


完整的"乐观锁扣库存"例子：

  WATCH stock:1001              ← 监视库存
  GET stock:1001 → "10"        ← 读库存
  (计算: 10 - 1 = 9)
  MULTI
  SET stock:1001 9              ← 排队设置新库存
  EXEC                          ← 如果 nobody 改过 stock → 执行
                                ← 有人改过 → 返回 null，重试
```

### 2.3 Java 实现

```java
/**
 * 使用 WATCH 实现乐观锁扣库存
 */
public boolean deductStock(Jedis jedis, String productId) {
    String stockKey = "stock:" + productId;

    // 重试循环——EXEC 失败就重来
    while (true) {
        // ① WATCH：开始监视库存 key
        jedis.watch(stockKey);

        // ② 读取当前库存
        String stockStr = jedis.get(stockKey);
        int stock = Integer.parseInt(stockStr != null ? stockStr : "0");
        if (stock <= 0) {
            jedis.unwatch();  // 库存不足，取消监视
            return false;
        }

        // ③ MULTI + SET + EXEC
        Transaction tx = jedis.multi();
        tx.set(stockKey, String.valueOf(stock - 1));  // 扣库存

        List<Object> results = tx.exec();
        // exec() 返回 null → 有人改过 stockKey → WATCH 触发 → 拒绝执行
        // exec() 返回非 null → 执行成功！

        if (results != null) {
            return true;  // 扣库存成功
        }
        // results == null → 被其他客户端改过 → 回到 while 循环重试
    }
}
```

**效果对比：**

```
WATCH 扣库存：

  Client-A                              Client-B
    │ WATCH stock:1001                    │
    │ GET stock:1001 → 10                 │
    │                                      │ WATCH stock:1001
    │                                      │ GET stock:1001 → 10
    │ MULTI                                │ MULTI
    │ SET stock:1001 9                     │ SET stock:1001 9
    │ EXEC → ["OK"]  ← A 先到，成功！       │
    │                                      │ EXEC → null  ← WATCH 检测到 A 改过 stock
    │                                      │ 重试：
    │                                      │   WATCH → GET=9 → MULTI → SET 8 → EXEC ✓

结果：counter 最终 = 8（正确！10 - 1 - 1 = 8）
```

### 2.4 WATCH 的注意事项

- WATCH 在 EXEC 执行后**自动解除**（不管成功还是失败）
- 也可以手动 `UNWATCH` 取消监视
- WATCH 监视的是 key 是否被**修改**（值变了或 key 被删都算），读操作不触发
- **WATCH 不能用于 Cluster 模式**（如果 key 在多个 slot 上，WATCH 只在单个节点有效）

---

## 第三章：Pipeline、Transaction、Lua —— 如何选？

### 3.1 Pipeline —— 纯批量，不原子:rocket:我还以为是原子呢:rocket::rocket::rofl::rofl::o::o::o::o:

```
Pipeline：
  客户端一次性发送多条命令（不等单个回复），
  Redis 按顺序执行，回复按顺序返回。

  特点：减少网络 RTT，但不保证原子——中途可能插入其他客户端的命令。
```

```java
// Pipeline：5 个命令一次发送，5 个回复一次接收
Pipeline pipeline = jedis.pipelined();
pipeline.set("k1", "v1");
pipeline.incr("counter");
pipeline.lpush("list", "a");
pipeline.sync();  // 发送所有命令，接收所有回复
```

- **适用场景举例**
  - 批量设置 10 万个用户的在线状态（`SET user:1:online 1 ...`），无需顺序性。
  - 一次读取多个 hash 的字段，减少查库延迟。
  - 预热缓存时批量写入大量数据。

### 3.2 Transaction —— 打包原子，排队执行

```
Transaction（MULTI/EXEC）：
  命令先排队，EXEC 时原子执行，中途不被插入。

  特点：原子、串行、不支持回滚、不能根据中间结果做条件判断。
```

- **适用场景举例**
  - 转账操作：通过 `WATCH` 监视账户余额，若余额未被改动则执行扣款和加款，否则重试（乐观锁）。
  - 批量更新多个键，只要求中间不被插队，不要求出错回滚。
  - 需要简单原子性但无需返回值判断的操作。

### 3.3 Lua —— 灵活脚本，服务端原子

```
Lua 脚本：
  一整段脚本在 Redis 服务端编译、原子执行。
  可以用任何 Redis 命令，可以做 if/else/for/while 条件判断，
  可以读取中间结果做后续决策。

  特点：原子、灵活、可条件判断、是生产环境最推荐的原子操作方式。
```

```java
// Lua：扣库存 + 判断结果 + 条件操作，全部原子
String lua =
    "local stock = tonumber(redis.call('get', KEYS[1]) or 0) " +
    "if stock <= 0 then return 0 end " +      // 库存不足 → 返回失败
    "redis.call('decr', KEYS[1]) " +           // 扣库存
    "return 1";                                 // 返回成功

Long result = (Long) jedis.eval(lua,
    Collections.singletonList("stock:1001"),
    Collections.emptyList());
// result = 1（成功） 或 0（库存不足）
```

- **适用场景举例**
  - **分布式锁释放**：先判断锁的值是否匹配，再决定是否删除（避免误删他人锁）。
  - **限流算法**：如滑动窗口、令牌桶等，需要原子地统计计数、比较、更新。
  - **库存扣减**：先查询库存，若足够则扣减，保证不超卖。
  - **复合缓存更新**：例如同时更新缓存和数据库的标记，保持一致性。

### 3.4 三种方案对比

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│              │  Pipeline    │ Transaction  │  Lua 脚本     │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ 是否原子      │ ❌ 不原子     │ ✅ 原子       │ ✅ 原子       │
│ 条件判断      │ ❌ 客户端判断  │ ❌ 不能       │ ✅ 可以       │
│ 读中间结果    │ ❌ 不行       │ ❌ 不行       │ ✅ 可以       │
│ 减少网络RTT   │ ✅           │ ✅           │ ✅           │
│ 回滚支持      │ ❌           │ ❌           │ ❌           │
│ 适用场景      │ 批量写入      │ 简单原子操作   │ 复杂原子逻辑   │
│ 例子          │ 批量SET缓存   │ 扣库存+创建订单│ 限流+扣库存    │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

---

## 第四章：常见使用场景

### 4.1 场景一：秒杀扣库存（已淘汰——用 Lua 更好）

```java
// 用 WATCH + MULTI 实现（不太推荐的旧方式）
public boolean seckillOld(Jedis jedis, String productId) {
    String key = "stock:" + productId;
    jedis.watch(key);
    int stock = Integer.parseInt(jedis.get(key));
    if (stock <= 0) { jedis.unwatch(); return false; }

    Transaction tx = jedis.multi();
    tx.decr(key);
    List<Object> result = tx.exec();  // null = 冲突了，重试
    return result != null;
}

// 用 Lua 实现（推荐的生产方式）
public boolean seckillNew(Jedis jedis, String productId) {
    String lua =
        "local s = tonumber(redis.call('get', KEYS[1]) or 0) " +
        "if s <= 0 then return 0 end " +
        "redis.call('decr', KEYS[1]) " +
        "return 1";
    return ((Long) jedis.eval(lua,
        Collections.singletonList("stock:" + productId),
        Collections.emptyList())) == 1L;
}
```

### 4.2 场景二：计数器批量自增后一起返回

```java
// 对同一组 key 批量 INCR，EXEC 后拿到所有新值
public List<Long> batchIncr(Jedis jedis, String... keys) {
    Transaction tx = jedis.multi();
    for (String key : keys) {
        tx.incr(key);          // 排队：INCR k1, INCR k2, ...
    }
    return (List<Long>) (List<?>) tx.exec();
    // 返回 [新值1, 新值2, ...]
}
```

### 4.3 场景三：转账（A 扣钱 + B 加钱必须同时成功）

```java
/**
 * 转账：A -100, B +100，必须同时成功
 * 用 Lua 保证原子——避免 A 扣了 B 没加上
 */
public boolean transfer(Jedis jedis, String from, String to, int amount) {
    String lua =
        "local fromBalance = tonumber(redis.call('get', KEYS[1]) or 0) " +
        "if fromBalance < tonumber(ARGV[1]) then return 0 end " +  // 余额不够
        "redis.call('decrby', KEYS[1], ARGV[1]) " +                 // A 扣钱
        "redis.call('incrby', KEYS[2], ARGV[1]) " +                 // B 加钱
        "return 1";

    return ((Long) jedis.eval(lua,
        Arrays.asList(from, to),
        Collections.singletonList(String.valueOf(amount)))) == 1L;
}
```

---

## ⭐️ 面试题汇总

**Q1: Redis 事务和 MySQL 事务的本质区别是什么？**

> MySQL 事务支持回滚——某条语句失败，整个事务回退。Redis 事务不支持回滚——EXEC 中的某条命令失败了（如类型不匹配），它之前的命令不会撤销，它之后的命令继续执行。Redis 事务的"原子"指的是执行期间不被插入，不是"全成功或全失败"。

**Q2: MULTI 之后 SET 命令为什么不立即执行？**

> MULTI 进入的是"排队模式"——所有命令先加入一个命令队列（内存 list），各自返回 QUEUED。只有 EXEC 时才一次性从头到尾执行队列，且执行期间不被其他客户端打断——保证了事务的隔离性。

**Q3: WATCH 是什么？解决了什么问题？**

> WATCH 是 Redis 的乐观锁——监视一个或多个 key。在 EXEC 时如果被监视的 key 被其他客户端修改过，EXEC 返回 nil（拒绝执行）。这解决了"事务排队期间 key 被其他客户端修改"的竞态问题。通常在 WATCH + GET + MULTI + SET + EXEC 这个 CAS 模式中使用。

**Q4: WATCH 用在 Redis Cluster 中有什么问题？**

> WATCH 只能监视当前连接所在节点的 key。如果被 WATCH 的 key 分布在多个 Cluster 节点上，无法在一个连接中同时 WATCH 它们。Cluster 模式下推荐用 Lua 替代 WATCH（前提是 Lua 内操作的 key 在同一个 hash slot）。

**Q5: Pipeline、Transaction、Lua 怎么选？**

> - Pipeline：不要求原子，只为了减少网络 RTT → 批量 SET、批量 GET
> - Transaction：要求原子但逻辑简单（没有条件判断）→ 批量自增
> - Lua：要求原子且有条件判断、循环、读中间结果 → 扣库存、限流、转账、秒杀

**Q6: 为什么 Redis 不实现事务回滚？**

> antirez 认为：Redis 命令失败通常是编程错误（类型不匹配等），应该在开发阶段发现。实现回滚需要为每个命令维护"逆操作"→ 极大增加复杂度 → 与 Redis 简单高效的设计哲学相悖。如果需要回滚语义 → 用 Lua 做前置校验，或者业务层做补偿逻辑。

---

*Created: 2026-05-26 | Category: 12-Redis事务*
