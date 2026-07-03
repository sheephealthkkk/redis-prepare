# Redis 性能优化 —— HotKey（热 Key）

---

## 第一章：什么是 HotKey？

### 1.1 定义

**HotKey = 在短时间内被大量请求访问的 key。** 比如一个爆款商品的详情缓存 key，每秒被访问几十万次。

```
正常 key：
  每秒 10 次 GET → 哪个 Redis 节点都轻松处理 → 无问题

热 key：
  每秒 10 万次 GET → 全部落到某个 Redis 节点
  → 该节点 CPU 打到 100%
  → 其他 key 的请求也被拖慢
  → 这个节点可能被打挂
```

### 1.2 HotKey 和 BigKey 的区别

| 维度 | BigKey | HotKey |
|------|--------|--------|
| **衡量** | key 值大/元素多 | 访问频率高 |
| **危害** | 操作阻塞、内存大、删除慢:rocket: | 单个节点过载、CPU 失衡:rocket: |
| **发现** | SCAN + MEMORY USAGE | 客户端统计 / Redis 命令统计 |
| **解决** | 拆分/UNLINK | 多级缓存/本地缓存/读写分离:rocket::rocket: |

一个 key 可以同时是 BigKey 也是 HotKey（最危险的情况）。:rocket::rocket；

---

## 第二章：HotKey 有什么危害？

### 2.1 流量集中→单节点过载

```
Cluster 模式下：

  ┌──────────────────────────────────────────────┐
  │              Redis Cluster (3 主 3 从)        │
  │                                              │
  │  节点1：CPU 30%  │  节点2：CPU 100%！│  节点3：CPU 25% │
  │  ↑ 正常          │  ↑ 热 key "爆款商品"   │  ↑ 正常          │
  │                  │    的 slot 正好在这  │                  │
  └──────────────────────────────────────────────┘

问题：一个节点被打满→整个集群吞吐受阻（因为请求可能路由到此节点）
```

### 2.2 可能导致缓存击穿

热 key 突然过期 → 瞬间大量请求 miss → 全部打到 DB → 击穿。

---

## 第三章：如何发现 HotKey？

### 3.1 客户端统计（最直接）:rocket::rocket::rocket::rocket::rocket:

```java
/**
 * 在客户端埋点统计 key 的访问频率
 * 用 ConcurrentHashMap + AtomicLong 本地累加，定时上报
 */
@Component
public class HotKeyDetector {
    // 本地计数器：key → 访问次数
    private final ConcurrentHashMap<String, LongAdder> counter = new ConcurrentHashMap<>();

    /**
     * 每次访问 Redis 前调用
     */
    public void recordAccess(String key) {
        counter.computeIfAbsent(key, k -> new LongAdder()).increment();
    }

    /**
     * 定时任务：每 10 秒统计一次 TopN 热 key
     */
    @Scheduled(fixedDelay = 10000)
    public void report() {
        // ① 取出当前快照，清空计数器
        ConcurrentHashMap<String, LongAdder> snapshot = 
            new ConcurrentHashMap<>(counter);
        counter.clear();

        // ② 按访问次数排序，取 Top 10
        List<Map.Entry<String, LongAdder>> topN = snapshot.entrySet().stream()
            .sorted((a, b) -> Long.compare(b.getValue().sum(), a.getValue().sum()))
            .limit(10)
            .collect(Collectors.toList());

        // ③ 上报（日志 / 监控 / 告警）
        for (Map.Entry<String, LongAdder> entry : topN) {
            log.info("HotKey: {} accessed {} times in 10s", 
                entry.getKey(), entry.getValue().sum());
        }
    }
}
```

### 3.2 Redis 命令统计（`INFO commandstats`）

```bash
# 看哪些命令被调用得最多（间接推断热 key 的存在场景）
127.0.0.1:6379> INFO commandstats
# cmdstat_get:calls=50000000,usec=2500000,usec_per_call=0.05
# ↑ GET 被调了 5000 万次 → 可能存在热 key

# 进一步（Redis 6.0+）
127.0.0.1:6379> INFO latencystats
# 查看延迟分布
```

### 3.3 redis-cli --hotkeys（Redis 4.0+）

```bash
# 内置热 key 扫描（基于 LFU 淘汰策略的计数器）
redis-cli --hotkeys

# 前提：maxmemory-policy 必须设为 allkeys-lfu 或 volatile-lfu
# 原理：LFU 模式下，每个 key 的 lru 字段存了访问频率
#       --hotkeys 扫描所有 key，输出频率最高的
```

### 3.4 网络层抓包分析

```bash
# tcpdump 抓取 Redis 流量 → 分析 key 访问频率
tcpdump -i eth0 port 6379 -w redis.pcap
# 然后用工具解析 RESP 协议提取 key，统计频率
```

---

## 第四章：如何解决 HotKey？

### 4.1 方案一：本地缓存（Caffeine）—— 最有效

```
思路：在 Redis 前面加一层本地缓存。热 key 第一次从 Redis 取，然后缓存在本地。
      后续请求直接读本地，不到 Redis。

架构：
  请求 → Caffeine(本地,纳秒级) → miss → Redis → miss → DB
         ↑ 热 key 命中在这里，不碰 Redis！
```

```java
/**
 * 多级缓存——本地 Caffeine 扛热 key
 */
@Service
public class ProductService {
    // L1: Caffeine 本地缓存
    private final Cache<String, Product> localCache = Caffeine.newBuilder()
        .maximumSize(1000)                           // 只存 1000 条（最热的）
        .expireAfterWrite(5, TimeUnit.SECONDS)       // 5 秒过期 → 防止数据不一致太久
        .recordStats()                               // 开启统计（看命中率）
        .build();

    @Autowired private StringRedisTemplate redis;

    public Product getProduct(String productId) {
        String key = "product:" + productId;

        // ① 先查本地缓存（纳秒级，不经过网络）
        Product cached = localCache.getIfPresent(key);
        if (cached != null) {
            return cached;  // 命中本地缓存 → 零网络开销！
        }

        // ② 再查 Redis（毫秒级）
        String json = redis.opsForValue().get(key);
        if (json != null) {
            Product product = JSON.parseObject(json, Product.class);
            localCache.put(key, product);  // 回填本地
            return product;
        }

        // ③ 最后查 DB
        Product product = productMapper.selectById(productId);
        if (product != null) {
            redis.opsForValue().set(key, JSON.toJSONString(product), 1, TimeUnit.HOURS);
            localCache.put(key, product);
        }
        return product;
    }
}
```

### 4.2 方案二：Redis 读写分离

```
思路：热 key 的读压力分散到多个从节点。

  ┌─────────┐
  │  Master  │  ← 只处理写操作
  └────┬────┘
       │ 异步复制
  ┌────┼────┬────────────┐
  ▼    ▼    ▼            ▼
┌────┐┌────┐┌────┐  ┌──────────┐
│从1 ││从2 ││从3 │  │ 从节点... │  ← 多个从节点分摊读压力
└────┘└────┘└────┘  └──────────┘

客户端随机选一个从节点读 → 热 key 的读压力被 N 个从节点分摊
```

```java
// 配置读写分离
// 写走 Master，读随机选一个 Slave
// Redisson 支持 master/slave 模式自动路由
Config config = new Config();
config.useMasterSlaveServers()
    .setMasterAddress("redis://master:6379")
    .addSlaveAddress("redis://slave1:6379", "redis://slave2:6379")
    .setReadMode(ReadMode.SLAVE);  // 读从 Slave 读
```

### 4.3 方案三：热点数据复制（多份缓存）:rocket::rocket::rocket::rocket::rocket:

### 问题出在 **“所有请求都压在一个节点的一个 key 上”** ，那就把这个 key **复制多份**，分散到多个节点（甚至同节点的不同 key），让客户端读的时候**随机选择其中一个副本**。

```
思路：热 key 在 Redis 中存多份副本，请求随机选一份读取。
      key 不同 → hash slot 不同 → 落到不同 Redis 节点 → 压力分摊。

  product:detail:1001:0   → slot 1234 → 节点1
  product:detail:1001:1   → slot 5678 → 节点2
  product:detail:1001:2   → slot 9012 → 节点3
  ↑ 同一份数据，3 个副本分散在 3 个节点上
```

```java
/**
 * 热 key 多副本策略
 */
public Product getProductWithReplicas(String productId, int replicas) {
    // 随机选一个副本
    int replica = ThreadLocalRandom.current().nextInt(replicas);
    String key = "product:" + productId + ":" + replica;

    String json = redis.opsForValue().get(key);
    if (json != null) {
        return JSON.parseObject(json, Product.class);
    }

    // 所有副本都 miss → 查 DB + 回填所有副本
    Product product = productMapper.selectById(productId);
    if (product != null) {
        String value = JSON.toJSONString(product);
        for (int i = 0; i < replicas; i++) {
            redis.opsForValue()
                .set("product:" + productId + ":" + i, value, 1, TimeUnit.HOURS);
        }
    }
    return product;
}
```

预先在缓存里存多份：

```
hot_key:1
hot_key:2
hot_key:3
hot_key:4
```

客户端在访问时，先随机生成一个 1~4 的数字，拼出完整的 key 去读。例如：

```
副本号 = random(1, 4)
GET hot_key:{副本号}
```

### 4.4 方案四：将热 key 提前预热

```java
/**
 * 大促前/新闻推送前，提前把即将变热的 key 加载到缓存
 */
public void preheatKeys(List<String> expectedHotKeys) {
    for (String key : expectedHotKeys) {
        // 从 DB 加载数据
        Object data = loadFromDb(key);
        // 提前写入 Redis（不会等到用户请求才触发）
        redis.opsForValue().set(key, JSON.toJSONString(data));
    }
}
```

---

## ⭐️ 面试题

**Q1: HotKey 和 BigKey 有什么区别？怎么判断是哪一个？**

> BigKey 看大小（MEMORY USAGE / LLEN / HLEN）；HotKey 看频率（客户端统计 / redis-cli --hotkeys /INFO commandstats）。一个 key 可以同时是两者——比如一个爆款商品详情 JSON 既大又热。BigKey 伤害 Redis；HotKey 伤害集群均衡。

**Q2: 本地缓存 TTL 为什么设短（5-30 秒）？**

> 本地缓存不跨实例共享。如果 TTL 太长 → 数据更新后，各实例的本地缓存长时间不一致。TTL 短 → 不一致窗口短 → 偶尔 miss 走 Redis 也不贵。本地缓存的定位是"扛住热 key 的尖刺流量"，不是"替代 Redis"。

**Q3: 热 key 多副本方案有什么副作用？**

> ① 内存浪费 N 倍；② 更新困难——数据变更时要同时更新 N 个副本（否则部分副本是旧的）；③ 只适合"更新不频繁"的热 key。如果热 key 需要频繁更新，建议用本地缓存方案。

---

*Created: 2026-05-26 | Category: 15-Redis性能优化-HotKey*
