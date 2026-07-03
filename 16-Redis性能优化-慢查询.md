# Redis 性能优化 —— 慢查询命令

---

## 第一章：为什么会有慢查询命令？

### 1.1 根本原因——Redis 是单线程执行命令的

Redis 主线程一次只能执行一条命令。大多数命令是微秒级（GET/SET/INCR 等），但如果某条命令的执行时间达到了毫秒级甚至秒级，后续所有请求都要排队：

```
正常（GET 耗时 0.001ms）：
  GET k1 █ GET k2 █ GET k3 █ ... 每个请求微秒级 → 无延迟感知

慢查询（KEYS * 耗时 500ms）：
  GET k1 █ KEYS * █████████████████████████...████████ █ GET k2 █ GET k3
          ↑                                                ↑
  0.001ms  └─────── 500ms ──────────┘              这些请求等了 500ms
          期间所有其他请求排队！
```

### 1.2 哪些命令是"慢"的？

**核心规律：** 时间复杂度不是 O(1) / 操作元素数过多的命令。

```
致命级（生产环境禁止）：
  KEYS *           O(N) 遍历所有 key！N=几百万时阻塞秒级
  FLUSHALL/FLUSHDB O(N) 清空所有 key！(Redis 4.0+ 用 FLUSHALL ASYNC 异步)
  SAVE             O(N) 主线程做快照！阻塞秒到几十秒

危险级（数据量大时慎用）：
  SMEMBERS key     O(N) 返回整个 Set——大 Set 会阻塞
  HGETALL key      O(N) 返回整个 Hash——大 Hash 会阻塞
  LRANGE key 0 -1  O(N) 返回整个 List——大 List 会阻塞
  ZRANGE key 0 -1  O(logN+M) 返回整个 ZSet
  SUNION/SINTER/SDIFF  O(N*M) 大集合运算
  SORT key         O(N*logN) 排序

增大 N 明显变慢的：
  LINDEX key 10000000   O(N) 走到第 1000 万个元素
  ZRANGE key 100000 100010  O(logN + offset) 大 offset 遍历 span
```

### 1.3 不要 KEYS *——用 SCAN:rocket::rocket::rocket::rocket::rocket::rocket::rocket:

| 命令           | 执行方式                                            | 影响                                                         |
| :------------- | :-------------------------------------------------- | :----------------------------------------------------------- |
| `KEYS pattern` | **一次性全量扫描**，将所有匹配的 key 返回           | 如果 Redis 里有几百万个 key，这个命令可能执行几百毫秒甚至更久。由于 Redis 是单线程，这期间无法处理任何其他请求，造成**严重阻塞** |
| `SCAN cursor`  | **渐进式遍历**，每次只返回一小部分 key 和下一个游标 | 每次执行时间很短，可以在不阻塞其他请求的情况下，分多次完成全量遍历 |

> **核心思想：化整为零，分摊耗时。**

```bash
# ❌ KEYS * ——遍历所有 key，阻塞
KEYS user:*

# ✅ SCAN ——分批迭代，不阻塞
SCAN 0 MATCH user:* COUNT 1000
```

```java
// SCAN 的正确遍历方式
String cursor = "0";
do {
    ScanResult<String> result = jedis.scan(cursor, 
        new ScanParams().match("user:*").count(1000));
    for (String key : result.getResult()) {
        // 处理 key
    }
    cursor = result.getCursor();
} while (!"0".equals(cursor));
```

---

## 第二章：如何找到慢查询命令？

### 2.1 Slowlog（慢查询日志）——最核心工具

Redis 内置了慢查询日志，记录执行时间超过阈值的命令：

```bash
# ① 配置慢查询阈值（微秒）
CONFIG SET slowlog-log-slower-than 10000   # 10 毫秒 = 10000 微秒
                                           # 0 = 记录所有命令
                                           # -1 = 关闭慢查询

# ② 配置慢查询日志最大条数
CONFIG SET slowlog-max-len 128             # 存最近 128 条

# ③ 查看慢查询日志
SLOWLOG GET 10                             # 最近 10 条慢查询

# 输出示例：
# 1) 1) (integer) 42               ← 日志 ID
#    2) (integer) 1715000000       ← 发生时间戳
#    3) (integer) 150000           ← 执行耗时（微秒）= 150ms！
#    4) 1) "KEYS"                   ← 命令
#       2) "user:*"                ← 参数
#    5) "127.0.0.1:12345"          ← 客户端 IP
#    6) ""                         ← 客户端名称

# ④ 查看当前慢查询条数
SLOWLOG LEN

# ⑤ 清空慢查询日志
SLOWLOG RESET
```

**配置建议：**

```
slowlog-log-slower-than：生产建议设 10000（10ms）
  - 大多数 Redis 命令 < 0.1ms → 超过 10ms 的绝对值得关注
  - 如果你的业务对延迟极度敏感 → 设 1000 (1ms)

slowlog-max-len：生产建议设 128~256
  - 每条慢查询记录占几十字节 → 128 条约几 KB → 可忽略
  - 太小会丢失历史信息
```

### 2.2 从 Java 代码中查看慢查询

```java
/**
 * 定时拉取 Redis Slowlog，超过阈值打印告警
 */
@Component
public class SlowlogMonitor {
    @Autowired private StringRedisTemplate redisTemplate;

    @Scheduled(fixedDelay = 30000)  // 每 30 秒检查一次
    public void checkSlowlog() {
        // 执行 SLOWLOG GET 10
        List<Object> slowlogs = redisTemplate.execute(
            (RedisCallback<List<Object>>) conn ->
                conn.execute("SLOWLOG", "GET", "10".getBytes())
        );

        if (slowlogs == null) return;

        for (Object entry : slowlogs) {
            List<Object> log = (List<Object>) entry;
            long execTimeUs = (Long) log.get(2);         // 执行耗时（微秒）
            List<byte[]> args = (List<byte[]>) log.get(3); // 命令 + 参数
            String command = new String(args.get(0));

            // 超过 10ms → 告警
            if (execTimeUs > 10000) {
                log.warn("Redis慢查询！耗时={}ms, 命令={}", 
                    execTimeUs / 1000.0, 
                    command + " " + argsToString(args));
            }
        }
    }
}
```

### 2.3 其他排查手段

```bash
# ① 延迟监控（Redis 6.0+）
127.0.0.1:6379> INFO latencystats
# 返回 p50/p99/p100 延迟分布

# ② 命令统计
127.0.0.1:6379> INFO commandstats
# cmdstat_keys:calls=10,usec=5000000,usec_per_call=500000.00
# ↑ KEYS 被调了 10 次，平均每次耗时 500ms！

# ③ 实时监控
redis-cli --latency        # 持续采样 Redis 响应时间
redis-cli --latency-history # 每 15 秒输出一次延迟
```

---

你这个问题非常敏锐，触及到了计算机操作系统的内存管理核心。:rocket::radio::rocket::rocket::rocket::rocket::rocket::rocket:

简单说：**“删除 key”只是把钥匙扔了，但仓库里的货物（内存）还占着地方，必须有人去清理才能腾出空间放新东西。**

---

### 1. 你删除一个 key，Redis 到底做了什么？

当你执行 `DEL user:1:name`：
1. **逻辑删除**：Redis 从字典里把这个 key 的名字和对应的 value 指针去掉，你再也访问不到它了。
2. **物理回收**：Redis 需要把 value 占用的那块内存真正标记为“可用”，还给操作系统的内存分配器（例如 jemalloc），方便以后存新数据。

第 2 步就是“释放内存”。如果不做，那块内存就永远被标记为“已占用”，但没有任何 key 指向它，成了**内存泄漏**（浪费掉的内存）。

---

### 2. 为什么释放内存需要花时间？

释放不是一瞬间把一大块内存清零，而是：
- 对于复杂类型（List、Set、Hash、ZSet），它们内部由很多小节点（链表节点、哈希表节点、跳表节点）组成。
- 释放时要**逐个遍历这些节点**，调用 free 函数，把每个小块内存归还给分配器。
- 如果 key 有几百万元素，就要执行几百万次 free 操作，主线程会卡在这几百万次循环里。

`UNLINK` 就是让主线程只做第 1 步（丢掉 key 的引用），把耗时的第 2 步交给后台线程慢慢做。

---

### UNLINK 是做什么的？

`UNLINK key [key ...]` 是 **异步删除** 命令，从 Redis 4.0 开始引入。

它的作用和 `DEL` 一样，都是把 key 彻底删掉，立刻就不能再访问。
**唯一区别是**：`DEL` 会**同步释放这个 key 占用的内存**，而 `UNLINK` 把这个“释放内存”的耗时操作交给后台线程去做，主线程**立即返回**。

### UNLINK 与 SCAN 的配合

在处理“批量删除特定前缀的 key”（比如临时数据、过期的缓存）时，最佳实践就是 `SCAN` + `UNLINK`：

bash

```
# 伪代码
cursor = 0
do {
    result = SCAN cursor MATCH user:temp:* COUNT 1000
    cursor = result.next_cursor
    if result.keys 非空:
        UNLINK result.keys  # 一次性异步删除这一批 key
} while cursor != 0
```

## 第三章：慢查询的优化策略

| 慢操作 | 替代方案 |
|--------|---------|
| KEYS * | SCAN 分批迭代 |
| DEL bigKey | UNLINK 异步删除 |
| SAVE | BGSAVE（fork 子进程） |
| SMEMBERS / HGETALL 大集合 | SSCAN / HSCAN 分批 |
| LRANGE key 0 -1 | LRANGE key 0 99 分页 |
| ZRANGE 大 offset | 按 score 范围查替代 offset |
| SINTER 大集合 | SINTERSTORE 结果存 key + TTL + 定时更新 |

---

## ⭐️ 面试题

**Q1: KEYS * 为什么危险？替代方案是什么？**

> KEYS 遍历所有 key，O(N)，N=几百万时阻塞秒级——期间 Redis 无法处理任何请求。替代：SCAN 命令，每次只返回少量 key，用 cursor 分批迭代，不阻塞。

**Q2: 怎么发现 Redis 实例中存在慢查询？**

> SLOWLOG GET（Redis 内置慢查询日志），配置阈值（slowlog-log-slower-than=10000 即 10ms），保留最近 128 条。也可以看 INFO commandstats（每个命令的调用次数和总耗时）、redis-cli --latency 实时测延迟。

**Q3: 生产中 HGETALL 偶尔导致延迟飙升，怎么排查？**

> SLOWLOG GET 看是否有 HGETALL 耗时记录。如果有 → 用 HLEN 确认是否是大 Hash → 大 Hash 用 HSCAN 替代 HGETALL → 或设计层面把大 Hash 拆分成多个小 Hash。

---

*Created: 2026-05-26 | Category: 16-Redis性能优化-慢查询*
