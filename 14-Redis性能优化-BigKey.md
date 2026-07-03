# Redis 性能优化 —— BigKey（大 Key）

---

## 第一章：什么是 BigKey？

### 1.1 定义——不是"占内存大"，是"操作开销不成比例"

Redis 官方说的 **bigkey**，不是指“这个 key 占用了 10GB 内存”，而是指**一个 key 对应的数据结构中包含的元素数量太多**

所以 bigkey 的核心特征是 **“元素多”** ，而不是单纯 **“内存大”**。

BigKey 不是 Redis 官方术语，是**业界对"操作开销与数据规模不成比例的 key"的俗称**：

元素多，自然占用的内存就大，这是必然的结果。但这只是**伴随现象**，不是导致操作慢的直接原因。

**真正的元凶是“时间复杂度”和“单线程阻塞”。**:rofl::eagle::call_me_hand:

```
BigKey 指的不是 Redis 内存使用率高，而是：
  某个 key 的操作（读/写/删/序列化）所需时间远超普通 key，
  因为它的 value 太大（String）或元素太多（List/Hash/Set/ZSet）

一个 10MB 的 String key 和一个 10 字节的 String key：
  GET 前者 = 10MB 网络传输 + 10MB 内存拷贝
  GET 后者 = 10B 网络传输 + 10B 内存拷贝
  → 前者可能慢 1000000 倍！
```

### 1.2 各类型的 BigKey 判定标准

| 类型 | BigKey 标准 | 数据规模 (Catch) | 极端案例 |
|------|-----------|-------|---------|
| **String** | > 10KB | 如果业务需要频繁 GET，超过 1KB 就要考虑 | 一个人把整个报表的 JSON 塞进一个 String（100MB） |
| **List** | > 10000 元素 | 队列入队速度 > 出队速度时容易堆积 | 消息队列堆积了几千万条积压消息 |
| **Hash** | > 5000 field | 把整个用户表存在一个 Hash 里 | Hash 里有 1000 万个 field |
| **Set** | > 10000 member | 粉丝列表、关注列表无限增长 | 某个明星的粉丝 Set 有 5000 万用户 |
| **ZSet** | > 10000 member | 实时排行榜的 key 无限扩大 | 全站所有用户的积分排行榜 1 个 ZSet |

### 1.3 每种类型的 BigKey 内部长什么样:rocket::rocket::rocket::rocket::rocket:

**String BigKey 内部：**

```
正常 String:
  ┌──────────────┐
  │ SDS (embstr) │  44 字节以内的紧凑分配
  │ "hello"      │
  └──────────────┘

String BigKey:
  ┌──────────────┐        ┌─────────────────────────────────┐
  │ RedisObject  │ ptr──▶ │  SDS (raw)                      │
  │ 16 字节       │         │  10MB 数据                      │
  └──────────────┘        │  独立 malloc，可能分散在多个内存页  │
                          └─────────────────────────────────┘
  → GET 时的开销：10MB 内存拷贝 + 10MB 输出缓冲区 + 10MB 网络传输
```

**Hash BigKey 内部：**

```
正常 Hash (< 512 field, 小 value):
  ┌──────────────────────────────────────┐
  │ listpack（连续内存，紧凑排列）         │
  │ [f1][v1][f2][v2]...[f100][v100]     │
  └──────────────────────────────────────┘
  HGET O(N) 但 N 小，依然很快

Hash BigKey (> 512 field 或大 value):
  编码从 listpack 升级为 dict（→ 不可逆！）
  ┌─────┐┌─────┐┌─────┐      ← 桶数组
  │ [0] ││ [1] ││ [2] │...
  └──┬──┘└──┬──┘└──┬──┘
     │      │      │
     ▼      ▼      ▼
  entry→entry  entry
  HGET O(1)，但 HGETALL O(N) 遍历所有 entry
  → N 大时 HGETALL 极其缓慢
```

**Set BigKey 内部：**

```
正常 Set (≤ 512 个纯整数):
  ┌─────────────────────────┐
  │ intset（有序整数数组）    │
  │ [1][2][3]...[512]       │  二分查找 O(logN)
  └─────────────────────────┘

Set BigKey (> 512 元素或有字符串):
  升级为 dict（→ 不可逆！）
  ┌─────┐┌─────┐┌─────┐
  │ [0] ││ [1] ││ [2] │...
  └──┬──┘└──┬──┘└──┬──┘
     │
     ▼
  entry(key=member, value=NULL) → entry → entry
  SISMEMBER O(1)，但 SMEMBERS O(N) 遍历所有 entry
```

---

## 第二章：BigKey 是怎么产生的？有什么危害？

### 2.1 产生原因:rocket::rocket::rofl::rocket:

```
① 设计不当——把 Redis 当数据库用
   - "把所有的用户资料放在一个 Hash 里，HSET users:all {userId} {json}"
   - 几百万用户 × 每个用户一条 field → 一个 Hash 数百万 field

② 业务逻辑漏洞——数据只增不减
   - 消息队列 List：生产者 LPUSH，消费者 RPOP 跟不上生产速度
   - 日志收集 List：每次请求 LPUSH 一条，从未清除
   - 粉丝列表 Set：只 SADD 新粉丝，不 SREM

③ 序列化过大——一个 String 存了整个对象图
   - Java 对象 → JSON → String
   - 这个对象包含子对象、列表、嵌套对象 → JSON 几十 KB

④ 缺乏 TTL ——key 永久存在
   - 没有 EXPIRE，数据无限累积
   - 典型：临时数据（验证码、token）忘了设 TTL

⑤ 编码升级——从紧凑到膨胀
   - listpack→dict 或 intset→dict 升级后内存膨胀约 5-10 倍
   - 且升级不可逆（即使元素删到阈值以下，也不降回紧凑编码）
```

### 2.2 危害详解——每个危害对应哪些命令

#### 危害 1：特定命令严重阻塞主线程:rocket:

```
每种类型的 BigKey 对应的"危险命令"：

  String BigKey：
    GET big:string:key       ← 10MB 内存拷贝 + 输出缓冲区分配
    DEL big:string:key       ← 释放 10MB SDS（一大块连续内存，free 可能慢）
    危险度：★★★

  List BigKey：
    LRANGE key 0 -1          ← 遍历所有 quicklistNode，返回所有元素
    LINDEX key 9999999       ← 沿链表走到第 1000 万个元素
    DEL key                  ← 遍历所有 quicklistNode 逐个 free
    危险度：★★★★

  Hash BigKey：
    HGETALL key              ← 遍历所有 dictEntry，分配输出缓冲区
    HKEYS / HVALS key        ← 同上
    DEL key                  ← 遍历所有 bucket + entry 逐个 free
    危险度：★★★★★

  Set BigKey：
    SMEMBERS key             ← 遍历所有 dictEntry，返回所有 member
    SUNION/SINTER/SDIFF      ← 如果 BigKey 参与集合运算 → O(N*M)
    DEL key                  ← 遍历所有 bucket + entry 逐个 free
    危险度：★★★★★

  ZSet BigKey：
    ZRANGE key 0 -1          ← 遍历 skiplist level 0 所有节点
    ZRANGEBYSCORE key -inf +inf ← 同上
    DEL key                  ← 释放 dict + skiplist + 所有节点
    危险度：★★★★★
```

#### 危害 2：网络带宽被打爆

```
场景：一个热门接口每次返回一个 5MB 的 String BigKey

  QPS 100 × 5MB = 500MB/s 出流量
  QPS 1000 × 5MB = 5GB/s 出流量 → 万兆网卡全部打满

  更坏的是：单个大 key 占用了极多带宽，
            小 key 的请求被"挤"得超时
```

#### 危害 3：删除时产生的延迟毛刺（latency spike）:rocket::rofl:

这是生产环境排查"Redis 偶尔响应慢"最容易被忽视的原因：

```
DEL 一个 1000 万成员的 Set：

  主线程：
    dictRelease() → 遍历所有 1000 万个 dictEntry → 逐个 free()
    → 可能耗时 200ms ~ 2s（取决于 CPU + 内存碎片程度）

  影响：
    这 200ms 期间，Redis 不处理任何请求
    如果监控看到 Redis 每秒钟都有几毫秒的延迟，
    而日志中恰好有定时任务在 DEL 大 key → 这就是根源
```

#### 危害 4：主从全量同步卡顿

```
主从全量同步流程：
  Slave 连上 Master
  → Master 执行 BGSAVE（fork + 子进程生成 RDB）
  → Master 把 RDB 文件发给 Slave
  → Slave 加载 RDB

BigKey 的影响：
  ① fork 阶段：内存大 → 页表大 → fork 阻塞主线程时间长
  ② COW 阶段：BigKey 导致更多 COW 复制（大内存对象更容易被修改）
  ③ RDB 传输：RDB 文件巨大 → 网络传输几十 GB → Slave 很久才能追上 Master
```

#### 危害 5：内存碎片加剧

```
BigKey 经常被修改（如 List 的频繁 LPUSH + LPOP + LTRIM）：
  → quicklist 节点频繁分配和释放
  → jemalloc 产生碎片
  → mem_fragmentation_ratio 升高
  → 物理内存用超
```

---

## 第三章：如何全面发现 BigKey？

### 3.1 redis-cli --bigkeys（最常用）

```bash
redis-cli --bigkeys

# 完整输出示例（含每一阶段的进度）：
# Scanning the entire keyspace to find biggest keys.
# [00.00%] Biggest string found so far 'cache:page:home' with 5120 bytes
# [15.27%] Biggest string found so far 'report:2026-05' with 10485760 bytes
# [42.18%] Biggest list   found so far 'queue:logs' with 5000000 items
# [78.33%] Biggest hash   found so far 'users:all' with 1000000 fields
# 
# -------- summary -------
# Biggest string found: 'report:2026-05' has 10485760 bytes
# Biggest   list found: 'queue:logs' has 5000000 items
# Biggest   hash found: 'users:all' has 1000000 items
# Biggest    set found: 'followers:celebrity:001' has 50000000 items
# Biggest  zset found: 'rank:global' has 1000000 items
```

**使用注意事项：**
- 底层用 SCAN（非阻塞），不会卡 Redis
- **只输出每种类型的最大 key，不输出第二大、第三大**:rocket::rocket::rocket::rocket:——如果前 10 大 key 你都需要 → 自己写 SCAN 脚本
- 采样性——SCAN 遍历过程中元素可能变化，不是 100% 精确
- 不统计内存占用—— `MEMORY USAGE key` 配合使用

### 3.2 MEMORY USAGE（精确内存视角）

```bash
# 查看单个 key 的精确内存占用（包括 redisObject + 数据结构开销）
127.0.0.1:6379> MEMORY USAGE report:2026-05
(integer) 10486200   # 约 10MB（比 STRLEN 看到的字节数大 200 字节：
                     # 多的这 200 字节是 redisObject + sdshdr 开销）

# 查看采样统计（适用于所有 key）
127.0.0.1:6379> MEMORY USAGE report:2026-05 SAMPLES 0
# SAMPLES 0 = 不使用采样，精确计算（适合单个 key 调试）
```

### 3.3 MEMORY DOCTOR（Redis 4.0+ 内存诊断）

```bash
127.0.0.1:6379> MEMORY DOCTOR

# 输出示例：
# Hi Sam，I'm the Redis memory doctor. I'm here to help you.
# 
# 1) Peak memory usage is 12.5 GB (当前内存峰值)
# 2) High allocator fragmentation: mem_fragmentation_ratio = 1.85 > 1.5
#    → 碎片率高，可能由 BigKey 频繁创建/删除引起
# 3) Big keys: there are 12 keys with more than 1000000 bytes
#    → 发现 12 个大 key！
# 4) Big hash keys: users:all has 1000000 fields
#    → Hash BigKey 细节
```

### 3.4 INFO commandstats（间接告警）

```bash
127.0.0.1:6379> INFO commandstats
# cmdstat_del:calls=10,usec=5000000,usec_per_call=500000.00
#                ↑ 被调了10次   ↑ 总共5秒     ↑ 平均每次 500ms！
#
# 正常 DEL < 0.1ms，500ms 说明有 1000 多万元素的 BigKey 在被删除

# cmdstat_hgetall:calls=100,usec=2000000,usec_per_call=20000.00
# ↑ HGETALL 平均 20ms → 有 Big Hash
```

### 3.5 自定义 SCAN 扫描脚本（全面版）

```java
/**
 * 完整版 BigKey 扫描工具——同时统计元素数和内存占用
 */
@Component
public class BigKeyScanner {
    @Autowired private StringRedisTemplate redis;

    // 阈值可配置
    private static final long STRING_MAX_BYTES = 10 * 1024;      // 10KB
    private static final long LIST_MAX_ITEMS = 5000;
    private static final long HASH_MAX_FIELDS = 5000;
    private static final long SET_MAX_MEMBERS = 10000;
    private static final long ZSET_MAX_MEMBERS = 10000;

    public List<BigKeyInfo> scan() {
        List<BigKeyInfo> bigKeys = new ArrayList<>();
        String cursor = "0";

        do {
            // ① SCAN 分批
            ScanResult<String> result = redis.opsForValue().getOperations()
                .scan(cursor, ScanOptions.scanOptions().count(1000).build());
            cursor = result.getCursor();

            for (String key : result.getKeys()) {
                // ② 跳过非用户 key（如内部 key）
                if (key.startsWith("redisson_") || key.startsWith("spring:session")) {
                    continue;
                }

                // ③ 获取类型
                DataType type = redis.type(key);
                if (type == null) continue;

                long size = 0;
                long memoryBytes = 0;
                boolean isBig = false;

                switch (type) {
                    case STRING:
                        // STRLEN O(1)
                        size = redis.opsForValue().size(key);
                        if (size > STRING_MAX_BYTES) isBig = true;
                        break;
                    case LIST:
                        size = redis.opsForList().size(key);  // LLEN O(1)
                        if (size > LIST_MAX_ITEMS) isBig = true;
                        break;
                    case SET:
                        size = redis.opsForSet().size(key);  // SCARD O(1)
                        if (size > SET_MAX_MEMBERS) isBig = true;
                        break;
                    case ZSET:
                        size = redis.opsForZSet().size(key);  // ZCARD O(1)
                        if (size > ZSET_MAX_MEMBERS) isBig = true;
                        break;
                    case HASH:
                        size = redis.opsForHash().size(key);  // HLEN O(1)
                        if (size > HASH_MAX_FIELDS) isBig = true;
                        break;
                }

                if (isBig) {
                    // ④ 获取实际内存占用（可选，大量 key 时关闭以提高速度）
                    // memoryBytes = redis.execute(
                    //     (RedisCallback<Long>) conn ->
                    //         conn.execute("MEMORY", "USAGE", key.getBytes()));
                    
                    bigKeys.add(new BigKeyInfo(key, type.name(), size, memoryBytes));
                }
            }
        } while (!"0".equals(cursor));

        // ⑤ 按大小排序
        bigKeys.sort((a, b) -> Long.compare(b.size, a.size));
        return bigKeys;
    }

    @Data @AllArgsConstructor
    public static class BigKeyInfo {
        private String key;
        private String type;
        private long size;         // 字节数(String) 或 元素数(其他类型)
        private long memoryBytes;  // 实际内存占用
    }
}
```

### 3.6 运行时 BigKey 请求拦截（实时监控）

```java
/**
 * 在 Redis 客户端侧拦截对 BigKey 的读操作
 * 记录日志 + 告警，方便追踪是哪个业务代码在请求 BigKey
 */
@Aspect
@Component
public class BigKeyAccessAspect {

    // 已知的 BigKey 列表（可以从 BigKeyScanner 的结果中加载）
    private final Set<String> knownBigKeys = ConcurrentHashMap.newKeySet();

    @Around("execution(* org.springframework.data.redis.core.*Template.opsFor*(..))")
    public Object intercept(ProceedingJoinPoint pjp) throws Throwable {
        // 从参数中提取 key
        String key = extractKey(pjp.getArgs());
        if (key != null && knownBigKeys.contains(key)) {
            long start = System.currentTimeMillis();
            Object result = pjp.proceed();
            long elapsed = System.currentTimeMillis() - start;

            if (elapsed > 10) {  // 超过 10ms
                log.warn("BigKey 慢查询！key={}, 方法={}, 耗时={}ms, 调用栈={}",
                    key, pjp.getSignature(), elapsed,
                    Arrays.stream(Thread.currentThread().getStackTrace())
                        .limit(5).collect(Collectors.toList()));
            }
            return result;
        }
        return pjp.proceed();
    }
}
```

---

## 第四章：如何处理 BigKey？

### 4.1 处理决策树

```
发现 BigKey 后怎么办？

  ① 这个 key 还在被业务使用吗？
     ├─ 不用了 → UNLINK 立即删除（异步不阻塞）
     └─ 还在用 ↓

  ② 是 String 类型吗？
     ├─ 是 → GET → 缩小数据（服务端压缩 / CDN） → SET 回
     └─ 不是 ↓

  ③ 是 List 吗？
     ├─ 是 & 做队列 → 检查消费者速度 / 加消费实例
     ├─ 是 & 做时间线 → LTRIM 保留最近 N 条
     └─ 不是 ↓

  ④ 是 Hash / Set / ZSet 吗？
     ├─ 可以拆分 → 拆成多个小 key（按 ID 取模/按日期）
     ├─ 不能拆分 → 控制元素上限 + TTL
     └─ 数据不需要了 → UNLINK
```

### 4.2 方案一：异步删除 UNLINK（最优先使用）

```bash
# ❌ DEL：阻塞（主线程遍历所有元素逐个 free）
DEL big:set

# ✅ UNLINK：非阻塞（Redis 4.0+）
UNLINK big:set

# 内部实现：
#   主线程：把 big:set 从键空间摘掉（微秒级）→ 返回 OK
#   BIO-3 后台线程：遍历 dictEntry 逐个 free（异步，不影响主线程）
```

### 4.3 方案二：分批渐进式删除

```java
/**
 * 分批删除大 Hash（500 万 field）——不阻塞主线程
 * 每次 HDEL 1000 个 field，间歇 10ms 让主线程服务其他请求
 */
public void batchDeleteHash(Jedis jedis, String key) {
    String cursor = "0";
    do {
        ScanResult<Map.Entry<String, String>> result =
            jedis.hscan(key, cursor, new ScanParams().count(1000));
        cursor = result.getCursor();

        List<String> fields = result.getResult().stream()
            .map(Map.Entry::getKey).collect(Collectors.toList());

        if (!fields.isEmpty()) {
            jedis.hdel(key, fields.toArray(new String[0]));
        }

        // 关键：每批之间 sleep 10-50ms
        // 不 sleep → 虽然分批了，但 CPU 仍被占满，主线程没时间服务其他请求
        Thread.sleep(10);

    } while (!"0".equals(cursor));
}

/**
 * 分批删除大 List（不用 SCAN，用 LTRIM 逐步裁切）
 */
public void batchDeleteList(Jedis jedis, String key) {
    long len = jedis.llen(key);
    while (len > 0) {
        // 每次从右侧裁剪掉 100 个元素
        // LTRIM key 0 -101 = 保留前 len-100 个
        jedis.ltrim(key, 0, -(Math.min(100, len) + 1));
        len = jedis.llen(key);
        Thread.sleep(10);
    }
}

/**
 * 分批删除大 Set（SSCAN + SREM）
 */
public void batchDeleteSet(Jedis jedis, String key) {
    String cursor = "0";
    do {
        ScanResult<String> result = jedis.sscan(key, cursor, new ScanParams().count(1000));
        cursor = result.getCursor();

        if (!result.getResult().isEmpty()) {
            jedis.srem(key, result.getResult().toArray(new String[0]));
        }
        Thread.sleep(10);
    } while (!"0".equals(cursor));
}

/**
 * 分批删除大 ZSet（ZSCAN + ZREM）
 */
public void batchDeleteZSet(Jedis jedis, String key) {
    String cursor = "0";
    do {
        ScanResult<Tuple> result = jedis.zscan(key, cursor, new ScanParams().count(1000));
        cursor = result.getCursor();

        List<String> members = result.getResult().stream()
            .map(Tuple::getElement).collect(Collectors.toList());

        if (!members.isEmpty()) {
            jedis.zrem(key, members.toArray(new String[0]));
        }
        Thread.sleep(10);
    } while (!"0".equals(cursor));
}
```

### 4.4 方案三：从源头拆分 BigKey

```
═══════════════════════════════════════════════════════════════
                    拆分策略对照
═══════════════════════════════════════════════════════════════

类型│ 拆分前 (BigKey)             │ 拆分后
───┼─────────────────────────────┼──────────────────────
Str │ SET report:2026-05 ［100MB  │ 拆分为 report:2026-05:page1
    │ 整个报表的JSON］             │ page2 ...每页 10KB
    │                             │
Hash│ HSET users:all ［1000万field］│ HSET users:{hash}:0 ［1万field］
    │                             │ HSET users:{hash}:1 ［1万field］
    │                             │ ... 共 100 个 Hash
    │                             │ hash = CRC32(userId) % 100
    │
Set │ SADD followers:star:001     │ SADD followers:star:001:0
    │ ［5000万member］              │ followers:star:001:1
    │                             │ ... 按 userID 哈希分桶
    │
ZSet│ ZADD rank:global ［1亿分］   │ ZADD rank:daily:2026-05-26
    │                             │ ZADD rank:hourly:2026-05-26:15
    │                             │ 只保留活跃期 → 自动过期
```

**拆分原则：**
- Hash / Set / ZSet：按 ID 取模 / CRC32 哈希分桶:rocket::rocket:
- 时间序列数据：按日期/小时拆分 → 旧 key 自然过期清理
- String：按业务维度拆（如分页、分段）:rocket:

### 4.5 方案四：控制不升级编码

```bash
# 让更多数据停留在 listpack/intset（紧凑编码，不升级为 dict/skiplist）
hash-max-listpack-entries 1024      # 从 512 → 1024
hash-max-listpack-value 128         # 从 64 → 128
set-max-intset-entries 1024         # 从 512 → 1024
zset-max-listpack-entries 1024      # 从 128 → 1024

# 代价：紧凑编码下操作 O(N)，N 大到超出阈值时可能慢
# 所以只适合"数据确实不大不小"的场景
```

### 4.6 方案五：定期清理 + TTL 倒逼

```java
/**
 * 定时任务：清理超时的临时 key
 */
@Component
public class KeyCleanupTask {

    /**
     * 每天凌晨 3 点：清理所有"不应该活超过 48 小时"的临时 key
     */
    @Scheduled(cron = "0 0 3 * * ?")
    public void cleanupStaleKeys() {
        String cursor = "0";
        do {
            ScanResult<String> result = jedis.scan(cursor, 
                new ScanParams().match("tmp:*").count(1000));
            cursor = result.getCursor();

            for (String key : result.getResult()) {
                // 检查 TTL，只剩 < 1 小时的话直接清
                Long ttl = jedis.ttl(key);
                if (ttl != null && ttl >= 0 && ttl < 3600) {
                    jedis.unlink(key);  // UNLINK 异步删除
                }
            }
        } while (!"0".equals(cursor));
    }
}
```

---

## ⭐️ 面试题汇总

**Q1: BigKey 的判定标准是什么？**

> String > 10KB；List / Hash / Set / ZSet > 5000 ~ 10000 元素。对 String 要看实际内存占用（MEMORY USAGE），对集合类型还要看操作频率和命令类型（HGETALL 的 BigKey 不能只靠元素数判断）。核心判据是"操作耗时是否可接受"。

**Q2: DEL 一个 1000 万成员的 Set 会发生什么？为什么 UNLINK 更好？**

> DEL 在主线程遍历所有 dictEntry 逐个 free——1000 万个 entry 可能阻塞几百毫秒。此期间其他请求排队→延迟飙升。UNLINK 只把 key 从键空间摘掉（微秒级），真正释放内存交给后台 BIO-3 线程异步做（不阻塞主线程）。

**Q3: --bigkeys 扫描过程中会阻塞 Redis 吗？有什么局限？**

> 底层用 SCAN（分批遍历），不阻塞。局限：①只输出每种类型的最大 key，不输出第二大/第三大；②只统计元素数，不统计内存；③ SCAN 遍历期间数据可能变化，不是瞬时快照。需要更精确的结果→自己写 SCAN 脚本。

**Q4: 为什么 List BigKey 不能一次性 DEL？用什么替代？**

> DEL 需要释放所有 quicklistNode→遍历整个链表→阻塞。替代：`LTRIM key 0 -101` 逐步裁剪（每次删除 100 个元素），每步之间 sleep 10ms 给主线程喘息。或直接用 UNLINK。

**Q5: 如何从设计层面预防 BigKey 产生？**

> ① 给所有缓存 key 设 TTL；② Hash/Set/ZSet 按 ID 取模或按日期拆分；③ List 用 LTRIM 限制最大长度；④ 消息队列的消费速度监控→消费者堆积告警；⑤ 新 feature 上线前用 redis-cli --bigkeys 扫描一遍。

**Q6: INFO commandstats 中 usec_per_call 很高，这说明什么？**

> 表示某个命令的平均执行时间很长。例如 `cmdstat_hgetall:usec_per_call=50000`→平均每次 HGETALL 耗时 50ms→肯定有 Big Hash。`cmdstat_del:usec_per_call=200000`→有在上千万元素的 BigKey 被 DEL。

---

*Created: 2026-05-26 | Updated: 全面优化 | Category: 14-Redis性能优化-BigKey*
