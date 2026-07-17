# Redis 性能优化——批量操作与减少网络传输



---

## 前言:批量指令，pipeline，lua

## ：Redis 性能瓶颈到底在哪里？

Redis 单机 10w+ QPS 是操作内存的速度。但实际业务中，你往往达不到这个数字——瓶颈不在 Redis 本身，而在 **网络**。

### 先搞懂：什么是 RTT？

**RTT = Round-Trip Time = 网络往返时间。**

```
RTT 的生活类比：

  你打电话给朋友：
    你："喂，今天吃饭吗？"  ──▶  电话网络  ──▶  朋友
    你 ◀── "好啊，几点？"    ──  电话网络  ──  朋友

  你说一句话 + 等朋友回复 = 一次 RTT
  RTT 包括：你的话传到朋友耳朵的时间 + 朋友回话传到你耳朵的时间


RTT 在 Redis 中：

  Client 发送命令  ──▶  网络传输  ──▶  Redis 收到
                                          │
                                     Redis 执行命令
                                     (微秒级，极快)
                                          │
  Client 收到回复  ◀──  网络传输  ◀──  Redis 返回结果

  一次命令 + 一次回复 = 一次 RTT

RTT 的大致数值：
  同机房（局域网）：    0.1 ~ 0.5ms
  同城跨机房：          1 ~ 3ms
  跨城（如北京↔上海）：  10 ~ 30ms
  跨国（如中国↔美国）：  100 ~ 200ms

为什么 RTT 是 Redis 性能的关键？
  假设 RTT = 0.5ms，Redis 执行命令 = 0.001ms
  发 1 次 GET  = 0.5ms + 0.001ms ≈ 0.5ms（99.8% 是网络）
  发 10000 次 GET = 10000 × 0.5ms ≈ 5 秒（但 Redis 实际只忙了 10ms！）
  
  → Redis 很闲，但网络很忙 → 性能瓶颈在网络，不在 Redis
```

```
一次 Redis 请求的耗时拆解：

  ┌────────────────────────────────────────────────────────────┐
  │ Redis 命令执行（微秒级）             网络传输（毫秒级）      │
  │                                                          │
  │  Client ──── 网络 RTT ────▶ Redis 执行 ──── 网络 RTT ────▶ Client
  │           ~0.5ms                ~0.001ms      ~0.5ms       │
  │                                                          │
  │  一次请求总耗时 ≈ 1ms，其中 Redis 执行只占 0.1%           │
  │  99.9% 的时间花在了网络往返上！                           │
  └────────────────────────────────────────────────────────────┘

关键洞察：
  发 10000 次 GET → 10000 次网络 RTT → 约 5 秒
  发 1 次 MGET 10000 个 key → 1 次网络 RTT → 约 0.5ms
  → 差距 10000 倍！
```

**本章的核心目标：用一种方式把多次请求"打包"，用一次网络往返完成。**

---

## 第一章：原生批量操作命令

### 1.1 什么是原生批量命令？

Redis 内置了一批"一次操作多个 key"的命令，一条命令 = 一次网络往返 = 操作多个数据。

```
单次操作（1 次 RTT = 1 个数据）：
  GET user:1     → 1 次 RTT → 返回 1 条数据
  GET user:2     → 1 次 RTT → 返回 1 条数据
  GET user:3     → 1 次 RTT → 返回 1 条数据
  总耗时：3 次 RTT ≈ 3ms

批量操作（1 次 RTT = 3 个数据）：
  MGET user:1 user:2 user:3   → 1 次 RTT → 返回 3 条数据
  总耗时：1 次 RTT ≈ 1ms
```

### 1.2 所有原生批量命令一览:rocket::rocket:

| 类型 | 单次命令 | 批量版 | 说明 |
|------|---------|--------|------|
| **String** | GET / SET | MGET / MSET | 批量读写多个 String key |
| **Hash** | HGET / HSET | HMGET / HMSET(废弃) | HMGET 还在用；HSET 本身已支持多 field |
| **List** | LPUSH / RPUSH | 本身就支持多 value | `LPUSH key v1 v2 v3`——一条命令推多个 |
| **Set** | SADD / SREM | 本身就支持多 member | `SADD key m1 m2 m3`——一条命令加多个 |
| **ZSet** | ZADD / ZREM | 本身就支持多 member | `ZADD key s1 m1 s2 m2`——一条命令加多个 |

### 1.3 MGET / MSET 的使用与限制

```java
// ═══ MGET：一次取多个 key ═══
List<String> values = jedis.mget("user:1", "user:2", "user:3");
// → ["{user1的JSON}", "{user2的JSON}", "{user3的JSON}"]
// 一次 RTT！vs 三次 GET = 三次 RTT

// ═══ MSET：一次设多个 key ═══
jedis.mset("k1", "v1", "k2", "v2", "k3", "v3");
// MSET 是原子的！要么全成功，要么全失败
// vs 三次 SET = 三次 RTT 且不一定原子
```

**MGET/MSET 的限制：**
- MGET 返回的值顺序和 key 顺序一致，但 key 不存在返回 nil（不是空字符串）
- MSET 是原子的——所有 key 同时被设置，中间看不到部分更新的状态

### 1.4 Cluster 模式下 MGET/MSET 的"坑"——深度剖析:eagle::eagle::eagle::rofl::rofl:

#### 为什么跨 slot 会报错？

Redis Cluster 把数据分成了 **16384 个哈希槽（hash slot）**，每个 key 通过 CRC16 哈希算法计算它属于哪个槽：

```
Redis Cluster 的数据分布：

  Redis 集群 (3 主 3 从)
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   Master 1   │  │   Master 2   │  │   Master 3   │
  │ slot 0~5460  │  │slot 5461~10922│ │slot 10923~16383│
  └──────────────┘  └──────────────┘  └──────────────┘
        ↑                  ↑                  ↑
    keyA 在这里        keyB 在这里        keyC 在这里

每个 key 只存在于一个节点上——这就是"分片"的含义。
```

```
slot 是怎么算出来的？

  slot = CRC16(key) % 16384

  例如：
    key = "user:1001"  → CRC16("user:1001") % 16384 = 1234  → Master 1
    key = "order:5001" → CRC16("order:5001") % 16384 = 8900 → Master 2
    key = "product:99" → CRC16("product:99") % 16384 = 15000 → Master 3
```

**MGET "user:1001" "order:5001" 跨 slot 时会怎样？**

```
Client 发送：MGET user:1001 order:5001
                │
                ▼
    Client 将命令发给 Master 1（随机选的一个节点）
                │
                ▼
    Master 1 计算：
      user:1001 → slot 1234  ← 在我这里 ✓
      order:5001 → slot 8900 ← 在 Master 2 那里！✗
                │
                ▼
    Master 1 返回错误：
    (error) CROSSSLOT Keys in request don't hash to the same slot

原因：MGET 是一条命令，Redis 不会帮你把一条命令拆成两部分
      分别发给两个节点——一条命令只能在一个节点上执行。
```

#### 解决方案：Hash Tag（哈希标签）`{}`

**Redis 提供了 Hash Tag 机制：只对 key 中 `{}` 里面的部分计算哈希。**

好的，我们用最原生的 Redis 指令对比，彻底看清背后的机制。

---

### 1. 无 Hashtag 的情况

假设我们想存用户 1 的姓名和年龄，并一次取出：

```bash
> SET user:1:name "Alice"
OK
> SET user:1:age 25
OK
```

现在用 `CLUSTER KEYSLOT` 命令看它们各在哪个槽：
```bash
> CLUSTER KEYSLOT user:1:name
(integer) 5850
> CLUSTER KEYSLOT user:1:age
(integer) 11840
```
槽号不同，它们被分到了不同的节点（假设 5850 在节点 A，11840 在节点 B）。

执行 `MGET`：
```bash
> MGET user:1:name user:1:age
(error) CROSSSLOT Keys in request don't hash to the same slot
```
因为这两个 key 不在同一个槽，Redis Cluster 拒绝执行。

---

### 2. 加上 Hashtag 的情况

改造 key，用 `{}` 把用户 ID 包起来：

```bash
> SET user:{1}:name "Alice"
OK
> SET user:{1}:age 25
OK
```

再看它们的槽：
```bash
> CLUSTER KEYSLOT user:{1}:name
(integer) 5088
> CLUSTER KEYSLOT user:{1}:age
(integer) 5088
```
现在两个 key 的槽号完全一样，都是 5088，它们必然在同一个节点上。

执行 `MGET`：
```bash
> MGET user:{1}:name user:{1}:age
1) "Alice"
2) "25"
```
一切正常。

---

### 3. 机制揭秘

Redis 在计算 slot 时的内部逻辑（伪代码）：

```
计算 slot(key):
    hashtag = 提取第一个 { 和 } 之间的内容（如果存在且非空）
    if hashtag 不为空:
        用 hashtag 计算 CRC16 取模
    else:
        用整个 key 计算 CRC16 取模
```

关键就是：**一旦 key 中包含 `{xxx}`，就只用 `xxx` 计算槽，其他部分完全忽略。**

所以：
- `user:{1}:name` 提取出 `1` → CRC16("1") % 16384 = 5088
- `user:{1}:age` 提取出 `1` → 同样结果

只要 `{}` 内的字符串相同，算出来的槽就 100% 相同。

---

### 4. 总结指令对比

| 场景       | Key 形式                        | Slot 计算依据    | MGET 结果            |
| ---------- | ------------------------------- | ---------------- | -------------------- |
| 无 hashtag | `user:1:name`, `user:1:age`     | 整个 key         | 失败，CROSSSLOT 错误 |
| 有 hashtag | `user:{1}:name`, `user:{1}:age` | 仅 `{}` 内的 `1` | 成功，返回两条数据   |

Hashtag 的本质就是**人为干预哈希分区**，让本来会分散的 key 强制聚集在同一个槽，从而让多键命令成为可能。

**Hash Tag 的注意事项：**

```
① 不要滥用 Hash Tag——把所有 key 都放同一个 slot 会导致：
   → 所有数据集中在一个节点 → 失去了分片的意义 → 该节点内存/CPU 过载

② Hash Tag 只在需要 MGET/MSET/事务/Lua 多 key 时才用

③ 第一个完整的 {xxx} 会被识别为 tag
   key = "order:{user:1001}:detail"
   实际的 hash tag = "{user:1001}"（第一个完整的 {}）
   
④ 如果没有完整的 {}，按整个 key 计算 slot
   key = "user:1001"  → CRC16("user:1001") % 16384
```

#### Cluster 下 MGET/MSET 的替代方案

```java
// 方案 1：自己按 slot 分组，分批发送 MGET（最常用）
public Map<String, String> clusterMget(JedisCluster jedis, List<String> keys) {
    Map<String, String> result = new HashMap<>();
    
    // 按 slot 分组
    Map<Integer, List<String>> slotGroups = new HashMap<>();
    for (String key : keys) {
        int slot = JedisClusterCRC16.getSlot(key);  // 计算 slot
        slotGroups.computeIfAbsent(slot, k -> new ArrayList<>()).add(key);
    }
    
    // 对每个 slot 分别发 MGET（同 slot 的 key 一批完成）
    for (List<String> group : slotGroups.values()) {
        List<String> vals = jedis.mget(group.toArray(new String[0]));
        for (int i = 0; i < group.size(); i++) {
            result.put(group.get(i), vals.get(i));
        }
    }
    return result;
}
// 如果所有 key 刚好在同一 slot → 1 次 MGET 完成
// 如果分布在 N 个 slot → N 次 MGET（仍远少于逐个 GET 的 N×RTT）


// 方案 2：用 Pipeline（自动处理路由）
Map<String, Response<String>> responses = new LinkedHashMap<>();
Pipeline pipeline = jedis.pipelined();
for (String key : keys) {
    responses.put(key, pipeline.get(key));  // 客户端自动路由到正确节点
}
pipeline.sync();
```

### 1.4 HMGET 和 HSET 的批量用法

```java
// ═══ HMGET：一次取 Hash 中的多个 field ═══
List<String> fields = jedis.hmget("user:1001", "name", "age", "city");
// → ["张三", "25", "北京"]    ← 一次 RTT

// ═══ HSET 也可以批量设多个 field ═══
Map<String, String> map = new HashMap<>();
map.put("name", "张三");
map.put("age", "25");
map.put("city", "北京");
jedis.hset("user:1001", map);  // 一次 RTT 设 3 个 field
```

### 1.5 原生批量命令的优缺点

| 优点 | 缺点 |
|------|------|
| 一条命令，最快 | 只能操作同一种数据结构的 key |
| 原子执行:rofl: | Cluster 跨 slot 问题 |
| 代码极简 | 不支持"条件判断"（如 key 不存在才设） |
| 服务端一次处理 | 不支持"把上一个命令的结果作为下一个命令的参数" |

---

pipeline是非原子的

## 第二章：Pipeline（管道）

### 2.1 什么是 Pipeline？

**Pipeline 是把多条命令打包到一次网络传输中发送，Redis 按顺序执行并返回所有结果。**

```
没有 Pipeline（一问一答）：

  Client: GET k1 ────────▶ Redis
  Client: ◀──────────── "v1"
  Client: GET k2 ────────▶ Redis
  Client: ◀──────────── "v2"
  Client: GET k3 ────────▶ Redis
  Client: ◀──────────── "v3"
  耗时：3 × RTT


有 Pipeline（打包发送）：

  Client: GET k1\nGET k2\nGET k3 ──▶ Redis  ← 一次发完
  Client: ◀────────── "v1"\n"v2"\n"v3"       ← 一次收完
  耗时：1 × RTT
```

### 2.2 Pipeline 的本质——TCP 层面的批量:rocket::rocket::rocket::rofl::rofl::rofl::rofl:

Pipeline 不是 Redis 的"功能"，而是**利用 TCP 连接全双工特性的客户端技巧**:o::o::o:

```
Pipeline 的工作方式：

  ① 客户端把多条 RESP 命令拼接在一起（字节拼接）
  ② 通过 Socket 一次性写入（一次 write() 系统调用）
  ③ Redis 从 Socket 逐条读取 → 执行 → 响应 → 写入发送缓冲区
  ④ 客户端通过 Socket 一次性读取 Redis 发送缓冲区中的所有响应

关键：Redis 不知道这是"Pipeline"还是"普通请求"——它只是按 TCP 顺序读命令
```

### 2.3 Java 实现

```java
// ═══════════ 方式一：Jedis Pipeline ═══════════
public Map<String, String> batchGetWithPipeline(Jedis jedis, List<String> keys) {
    // ① 创建 Pipeline（获取一个可以连续发命令的管道对象）
    Pipeline pipeline = jedis.pipelined();

    // ② 把命令塞进管道（这些命令不会立即发出去！）
    // 每个 Response 对象是未来结果的"占位符"
    List<Response<String>> responses = new ArrayList<>();
    for (String key : keys) {
        responses.add(pipeline.get(key));  // 此时只是记录了"要发 GET key"
    }

    // ③ sync()：一次性把所有命令通过 Socket 发送出去
    //          然后等待 Redis 返回所有响应
    pipeline.sync();

    // ④ 从 Response 占位符中取出实际结果
    Map<String, String> result = new LinkedHashMap<>();
    for (int i = 0; i < keys.size(); i++) {
        result.put(keys.get(i), responses.get(i).get());
    }
    return result;
}


// ═══════════ 方式二：Spring Data Redis executePipelined ═══════════
public List<Object> batchGetWithSpring(StringRedisTemplate redisTemplate,
                                        List<String> keys) {
    List<Object> results = redisTemplate.executePipelined(
        new RedisCallback<Object>() {
            @Override
            public Object doInRedis(RedisConnection connection) {
                // 所有在这个 callback 里的操作都会自动打包
                StringRedisConnection strConn = (StringRedisConnection) connection;
                for (String key : keys) {
                    strConn.get(key);  // 返回 null（因为是 pipeline 模式）
                }
                return null;  // 必须返回 null
            }
        }
    );
    // executePipelined 返回所有结果（按操作顺序排列）
    return results;
}


// ═══════════ 方式三：Lettuce 异步 pipeline ═══════════
public void batchGetWithLettuce(LettuceConnection connection, List<String> keys) {
    connection.setAutoFlushCommands(false);  // 关闭自动 flush（= 开启批量模式）

    List<RedisFuture<String>> futures = new ArrayList<>();
    for (String key : keys) {
        futures.add(connection.async().get(key));  // 命令暂存在缓冲区
    }

    connection.flushCommands();  // 一次性全部发出！
    
    // 等待所有结果
    for (RedisFuture<String> future : futures) {
        try {
            String value = future.get();
        } catch (Exception e) { /* ... */ }
    }

    connection.setAutoFlushCommands(true);  // 恢复自动 flush
}
```

### 2.4 Pipeline 的注意事项

```
① Pipeline 不保证原子性
   Pipeline 发出的命令之间，其他客户端的命令可以插入
   如果需要原子 → 用 Lua 或 MULTI/EXEC

② Pipeline 中的命令不能依赖前一条命令的结果
   因为 Pipeline 先把所有命令发出去，再统一收结果
   你不能 "GET k1 → 判断值 → 决定 SET k2 的值"——Pipeline 不知道中间结果

③ Pipeline 一次发太多会撑爆内存
   Pipeline 的原理是把所有 Response 存在客户端内存
   如果 10 万条命令的 Pipeline → 客户端内存要存 10 万个 Response
   → 建议每批 ≤ 1000 条

④ Pipeline 中某条命令失败不影响其他命令
   不像 MULTI/EXEC（CAS 语法错误会丢弃整个事务）
```

### 2.5 Pipeline vs 原生批量命令

| 维度 | MGET/MSET | Pipeline |
|------|-----------|----------|
| **命令类型** | 只能同一种（MGET 只能 GET） | 可以混合（GET + SET + INCR...） |
| **原子性** | MSET 是原子的 | 不保证原子:o::o::o::o: |
| **Cluster** | 只能同 slot | 同 slot 跨 slot 都可以（客户端路由）:rofl::rofl: |
| **速度** | 略快（服务端一次处理） | 略慢（服务端逐条处理） |
| **灵活性** | 低 | **高** |

---

## 第三章：Lua 脚本 —— 最灵活的打包方式

### 3.1 Lua 脚本的独特价值

Lua 脚本是三种批量方案中**唯一可以在服务端做条件判断和变量赋值**的。:rocket::rocket::rocket::rocket::rocket::rocket::rocket:

```
原生批量命令：
  ❌ 不能做：if 条件判断
  ❌ 不能做：拿上一个命令的结果计算下一个命令的参数

Pipeline：
  ❌ 不能做：if 条件判断
  ❌ 不能做：拿上一个命令的结果做下一步决策（因为它先全发出去再收结果）

Lua：
  ✅ 可以做：if/else/for/while 各种逻辑
  ✅ 可以做：local x = redis.call('GET', KEYS[1]); redis.call('SET', KEYS[2], x+1);
  ✅ 可以做到：复杂的"读-判断-写"逻辑在 Redis 服务端原子完成
```

### 3.2 什么场景只能用 Lua？

```
场景 1：限流——判断计数 + 扣减（原子）
  ① 读计数器 → ② 判断是否超限 → ③ 不超限则 +1
  → 三步必须在 Redis 服务端原子完成（Pipeline 不行，它不能读中间结果）

场景 2：扣库存——判断 + 扣减（原子）
  ① 读库存 → ② 判断 > 0 → ③ 扣库存
  → 同限流，三步必须原子

场景 3：转账——A 扣钱 + B 加钱（原子）
  ① 读 A 余额 → ② 判断 ≥ 转账金额 → ③ A 扣 + B 加
  → 两步写操作必须原子

场景 4：复杂的封装——一次网络调用完成一个业务操作
  比如"创建订单"需要同时：INCR seq + SET order + SADD user_orders
  → Lua 把三个操作打包成一次网络 RTT
```

### 3.3 Lua vs Pipeline 的性能对比

```
同一个场景：100 个 key 做 GET

  100 次 GET：            100 × RTT ≈ 100ms
  MGET 100 个 key：       1 × RTT  ≈ 1ms    ← 最快，但不能混合命令
  Pipeline 100 次 GET：   1 × RTT  ≈ 1ms    ← 也很接近
  Lua 循环 100 次 GET：   1 × RTT  ≈ 1ms    ← 可以，但不如 MGET 快

结论：
  - 纯同类型操作：MGET/MSET > Pipeline > Lua
  - 需要混合命令：Pipeline ≈ Lua
  - 需要条件判断：只有 Lua
```

---

## ⭐️ 面试题汇总

**Q1: MGET 和 Pipeline 本质区别是什么？**

> MGET 是服务端命令——Redis 内部一次性处理，返回结果。Pipeline 是客户端技巧——多条命令一次性发过去，但 Redis 逐条处理逐条返回。MGET 略快（无逐条解析开销），但只能操作同类型；Pipeline 灵活（可混合命令），但不保证原子。

**Q2: Pipeline 在 Cluster 下怎么处理跨 slot 命令？**

> Pipeline 内的每条命令独立路由。如果 key 在不同的 slot → 客户端会分别发给不同的节点，Pipeline 仍然是有效的（各节点收到的仍然是"批量"），只是不能保证"多节点间的原子性"。通常用 `{}` 哈希标签把相关 key 强制放一个 slot 解决。

**Q3: 什么场景下 Pipeline 不行，必须用 Lua？**

> 当需要"根据前一个命令的结果决定下一个命令"时——Lua 可以在服务端做判断和变量操作。Pipeline 是"先发完所有命令再收结果"，不能读中间结果做决策。

**Q4: Pipeline 一次发多少条合适？**

> 没有绝对数字，看内存和 RTT。建议每批 ≤ 1000 条——单批 Pipeline 占用客户端内存约 `1000 × 平均回复大小`，响应延迟控制在几毫秒内。业务实时性要求高（<5ms）→ 每批 ≤ 100 条。

---

*Created: 2026-05-26 | Category: 13-Redis性能优化-批量操作*
