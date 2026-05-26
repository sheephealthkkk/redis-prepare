# Redis 三种特殊数据类型 —— 深度详解

---

## 前言：为什么叫"特殊"类型？

Redis 的五种基础类型（String/List/Hash/Set/ZSet）是通用型数据结构。而 **Bitmap、HyperLogLog、GEO** 是面向特定场景深度优化的——用极少的内存解决特定问题。它们底层都复用了已有的数据结构（Bitmap→String，HyperLogLog→String，GEO→ZSet），但对外暴露的命令完全不同。

```
三种特殊类型的定位：

  Bitmap      → 统计"是非"类数据       （今天签到没？今天活跃没？）
  HyperLogLog → 统计"去重计数"         （有多少不同的访客？）
  GEO         → 统计"地理位置"相关      （附近的人？离我多远？）
```

---

## 第一章：Bitmap（位图）

### 1.1 什么是 Bitmap？

**Bitmap 不是一种新的数据类型——它是对 String 的"位操作"。** String 最大 512MB = 2^32 位，可以表示 42 亿个不同的标志位。

```
Bitmap 的核心思想：

  一个 String 就是一个 byte[]（字节数组），每个字节 8 位（bit）。
  每一位可以表示一个"是/否"状态：1 = 签到，0 = 没签到。

  byte[0]                                        byte[1]
  ┌──┬──┬──┬──┬──┬──┬──┬──┐   ┌──┬──┬──┬──┬──┬──┬──┬──┐
  │0 │1 │2 │3 │4 │5 │6 │7 │   │8 │9 │10│11│12│13│14│15│  ← bit 序号
  └──┴──┴──┴──┴──┴──┴──┴──┘   └──┴──┴──┴──┴──┴──┴──┴──┘

  SETBIT key 5 1  →  把第 5 位设为 1
  SETBIT key 13 1 →  把第 13 位设为 1
  BITCOUNT key    →  2（有 2 个 1）
  GETBIT key 5    →  1（第 5 位是 1）
```

### 1.2 核心命令

```bash
# ① 设置/获取某个 bit 的值
SETBIT key offset value       # 把第 offset 位设为 value（0 或 1）
GETBIT key offset             # 获取第 offset 位的值

# ② 统计值为 1 的 bit 个数
BITCOUNT key [start end]      # 统计 [start, end] 字节范围内 1 的个数
                              # start/end 是字节索引，不是 bit 索引！

# ③ 多个 Bitmap 的位运算
BITOP AND dest key1 key2 ...  # 按位与（交集：两个 Bitmap 对应位都是 1 → 结果位为 1）
BITOP OR  dest key1 key2 ...  # 按位或（并集）
BITOP XOR dest key1 key2 ...  # 按位异或
BITOP NOT dest key            # 按位取反

# ④ 查找第一个 0 或 1 的位置
BITPOS key 0                  # 第一个值为 0 的 bit 位置
BITPOS key 1                  # 第一个值为 1 的 bit 位置
```

### 1.3 经典场景：日活用户（DAU）统计

**问题：** 每天有 1 亿用户，要统计哪些用户今天活跃了，以及连续 7 天活跃的有多少。MySQL `SELECT COUNT(DISTINCT user_id)` 在亿级数据上要几十秒；Redis Set 精确但要 5GB+。

**Bitmap 方案：** 一个 Bitmap 存一天的活跃状态，每个用户对应一个 bit。

```
┌─────────────────────────────────────────────────────────────┐
│                   Bitmap 日活统计模型                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  用户 bit 位:  user:0  user:1  user:2  ...  user:100000000  │
│              ┌───────────────────────────────────────┐      │
│  5月19日:    │ 0 1 1 1 0 0 0 1 ... 1   (12.5MB)     │      │
│  5月20日:    │ 0 1 1 0 1 0 1 1 ... 0   (12.5MB)     │      │
│  5月21日:    │ 1 1 0 0 1 1 0 1 ... 1   (12.5MB)     │      │
│  ...                                                   │      │
│  5月25日:    │ 0 1 1 1 0 1 0 1 ... 1   (12.5MB)     │      │
│              └───────────────────────────────────────┘      │
│                                                             │
│  7 天连续活跃（BITOP AND）:                                  │
│    只有 5月19日~5月25日 每天都是 1 的用户，结果才是 1         │
│    BITCOUNT dau:week  →  连续活跃用户数                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```java
@Component
public class DauBitmapService {
    @Autowired private StringRedisTemplate redis;

    /**
     * 用户活跃时调用：把该用户的 bit 设为 1
     */
    public void markActive(long userId) {
        String key = "dau:" + LocalDate.now();
        redis.opsForValue().setBit(key, userId, true);
        redis.expire(key, 32, TimeUnit.DAYS);  // 32天后自动删除
    }

    /**
     * 今日活用户数
     */
    public long todayDAU() {
        String key = "dau:" + LocalDate.now();
        return redis.execute(
            (RedisCallback<Long>) conn -> conn.bitCount(key.getBytes()));
    }

    /**
     * 连续 7 天活跃用户数
     */
    public long consecutiveDAU(int days) {
        // 收集最近 N 天的 key
        List<byte[]> dayKeys = new ArrayList<>();
        for (int i = 0; i < days; i++) {
            dayKeys.add(("dau:" + LocalDate.now().minusDays(i)).getBytes());
        }

        byte[] resultKey = ("dau:last" + days + "days").getBytes();

        // BITOP AND：所有天的 Bitmap 做位与 → 结果的每一位 = 每天都活跃
        redis.execute((RedisCallback<Object>) conn -> {
            conn.bitOp(BitOperation.AND, resultKey,
                dayKeys.toArray(new byte[0][]));
            return null;
        });

        return redis.execute(
            (RedisCallback<Long>) conn -> conn.bitCount(resultKey));
    }
}
```

**内存对比：**
- MySQL：user_id(8B) × 1亿 = 800MB（仅数据）
- Redis Set：1亿 × ~50B = 5GB
- **Redis Bitmap：1亿 bit = 12.5MB**

### 1.4 经典场景：用户签到

```java
/**
 * 用户签到——每个用户一个 Bitmap，365 位存全年签到状态
 */
public class CheckinService {

    /**
     * 签到
     * @param userId 用户 ID
     * @param dayOfYear 今天是今年的第几天（1~365）
     */
    public void checkin(long userId, int dayOfYear) {
        String key = "checkin:user:" + userId + ":" + Year.now().getValue();
        redis.opsForValue().setBit(key, dayOfYear - 1, true);
    }

    /** 是否签到 */
    public boolean hasCheckedIn(long userId, int dayOfYear) {
        String key = "checkin:user:" + userId + ":" + Year.now().getValue();
        return redis.opsForValue().getBit(key, dayOfYear - 1);
    }

    /** 今年签到天数 */
    public long checkinDays(long userId) {
        String key = "checkin:user:" + userId + ":" + Year.now().getValue();
        return redis.execute(
            (RedisCallback<Long>) conn -> conn.bitCount(key.getBytes()));
    }
}
// 每个用户 365 bit ≈ 46 字节！10 万用户 ≈ 4.6 MB
```

### 1.5 SETBIT 的底层实现

```
SETBIT key 10086 1 的底层做了什么？

  ① 定位到第几个字节：10086 / 8 = 1260（第 1260 个字节）
  ② 定位到字节内第几位：10086 % 8 = 6（该字节的第 6 位）
  ③ 读取 byte[1260]
  ④ 将该字节的第 6 位设成 1（|= 0x40）
  ⑤ 写回

整个操作 O(1)，纯位运算，极度高效。
```

---

## 第二章：HyperLogLog（基数统计）

### 2.1 什么是 HyperLogLog？

**HyperLogLog = 一种概率数据结构，用于统计"去重后的元素个数"（基数）。** 标准误差 0.81%，**固定 12KB 内存**可以统计 2^64 个不同的元素。

```
HyperLogLog 能做什么：
  统计 "今天有多少不同的用户访问了网站"（UV）

HyperLogLog 不能做什么：
  - 不能列出具体哪些用户（只给数字）
  - 不能精确计数（有 0.81% 误差）
  - 不能删除/去重某个元素
```

### 2.2 核心命令

```bash
PFADD key element [element ...]    # 添加元素到 HLL（Power of the First）
PFCOUNT key [key ...]              # 统计去重元素个数
PFMERGE dest key1 key2 [key3 ...]  # 合并多个 HLL 到 dest
```

命令只有 3 个——PF=Philippe Flajolet（HyperLogLog 的发明者）。

### 2.3 经典场景：网站 UV 统计

```java
@Component
public class UvService {
    @Autowired private StringRedisTemplate redis;

    /**
     * 记录一次访问（UV）
     * 内部：hash(userId) → 找对应桶 → 更新"最大连续 0 的个数"
     */
    public void recordVisit(long userId, long pageId) {
        String key = "uv:page:" + pageId + ":" + LocalDate.now();
        redis.opsForHyperLogLog().add(key, String.valueOf(userId));
        redis.expire(key, 32, TimeUnit.DAYS);
    }

    /**
     * 查询当日 UV（0.81% 误差的去重访问人数）
     */
    public long todayUV(long pageId) {
        String key = "uv:page:" + pageId + ":" + LocalDate.now();
        return redis.opsForHyperLogLog().size(key);
    }

    /**
     * 多页面的合并 UV（去重交叉用户）
     */
    public long mergedUV(long page1, long page2) {
        String mergedKey = "uv:merged:" + page1 + "_" + page2;
        redis.opsForHyperLogLog().union(mergedKey,
            "uv:page:" + page1 + ":" + LocalDate.now(),
            "uv:page:" + page2 + ":" + LocalDate.now());
        return redis.opsForHyperLogLog().size(mergedKey);
    }
}
```

**内存对比：**
- 1 亿 UV 精确去重（Set）：约 6-8 GB
- **HyperLogLog：固定 12 KB**（1/500,000）

### 2.4 HyperLogLog 的原理——伯努利试验 + 调和平均数

```
步骤 1：每个元素的 hash

  对 "user:10086" 做 64-bit 哈希：
    hash = 0b...101100111000000  ← 64 位

步骤 2：用前 14 位定位到 16384 个桶之一

  前 14 位 = 桶号（0~16383）
  假设 = 0b 0000 0000 0001 01 = 桶 5

步骤 3：在剩余 50 位中统计"从右向左连续 0 的个数"

  后 50 位 = ...00101100111000000
  连续 0 的个数 = 6
  
  如果 6 > 桶[5] 当前的值 → 桶[5] = 6

步骤 4：计算基数

  基数 ≈ 常数 × 16384² / Σ(1/2^桶值)
  常数 ≈ 0.7213 × 16384²（已校准）

  这就是"调和平均数"——对大值不敏感，对极端值有容忍度

为什么误差是 0.81%？

  标准误差 = 1.04 / √桶数
            = 1.04 / √16384
            = 1.04 / 128
            ≈ 0.008125 = 0.81%
```

### 2.5 PFMERGE 的合并原理

```
PFMERGE 不是简单的"求和"——而是取每个桶的"最大值"：

  桶 0：HLL_A=3, HLL_B=5  → 取 5
  桶 1：HLL_A=7, HLL_B=4  → 取 7
  ...
  桶 16383：HLL_A=2, HLL_B=8 → 取 8

  为什么取 max 而不是 sum？
    → 因为桶的值 = 看到过的最长 0 连续数
    → 合并两个 HLL = 两个视角看同一个集合 = 取各个视角的最大值
```

### 2.6 使用建议

| 场景 | 推荐 |
|------|------|
| 统计 UV、独立 IP 数 | ✅ HyperLogLog（0.81% 误差可接受） |
| 需要精确数字（如金融记账） | ❌ 用 Set 或 DB |
| 需要知道"具体是谁" | ❌ HLL 不存具体元素 |
| 多个维度的去重合并分析 | ✅ PFMERGE 快速合并 |

---

## 第三章：GEO（地理位置）

### 3.1 什么是 GEO？

**GEO = 存储经纬度坐标，支持距离计算、范围查询、附近的人等地理位置操作。** 底层是用 ZSet 实现的——经纬度编码成 52-bit 的 GeoHash 值，作为 ZSet 的 score。

```
GEO 和 ZSet 的关系：

  ZSet 的操作：
    ZADD key score member    →  GEOADD key longitude latitude member
    ZRANGE key 0 -1          →  GEOPOS key member
    ZRANGEBYSCORE key min max → GEORADIUS key ...（底层转 score 范围查询）

  GEO 把经纬度编码成一个 64 位数字（GeoHash），存入 ZSet 的 score。
  靠近的点 → GeoHash 相近 → ZSet 的 ZRANGEBYSCORE 就能查附近。
```

### 3.2 核心命令

```bash
# ① 添加位置
GEOADD key longitude latitude member [longitude latitude member ...]
# 例如：GEOADD cities 116.404 39.915 beijing 121.473 31.230 shanghai

# ② 获取位置
GEOPOS key member [member ...]
# → [["116.404","39.915"], ["121.473","31.230"]]

# ③ 计算两点距离
GEODIST key member1 member2 [m|km|ft|mi]
# GEODIST cities beijing shanghai km → 约 1068 km

# ④ 获取 GeoHash 编码
GEOHASH key member [member ...]

# ⑤ 半径查询（Redis 6.2 前）
GEORADIUS key longitude latitude radius m|km|ft|mi [WITHDIST] [WITHCOORD] [ASC|DESC] [COUNT n]
# 查询指定点周围 radius 范围内的所有元素

# ⑥ 半径查询——按已有 member（Redis 6.2 前）
GEORADIUSBYMEMBER key member radius m|km|ft|mi

# ⑦ Redis 6.2+ 统一搜索（替代 GEORADIUS/GEORADIUSBYMEMBER）
GEOSEARCH key FROMMEMBER member | FROMLONLAT lon lat
            BYRADIUS radius unit | BYBOX width height unit
            [ASC|DESC] [COUNT n] [WITHDIST]
```

### 3.3 经典场景：附近的人/附近的店

```java
@Component
public class NearbyService {
    @Autowired private StringRedisTemplate redis;

    private static final String GEO_KEY = "geo:shops";

    /**
     * 添加商家位置
     */
    public void addShop(long shopId, double longitude, double latitude) {
        // GEOADD geo:shops {longitude} {latitude} {shopId}
        redis.opsForGeo().add(GEO_KEY,
            new Point(longitude, latitude), String.valueOf(shopId));
    }

    /**
     * 查询我附近的商家（5 公里内，按距离排序，返回最近 10 个）
     */
    public List<NearbyShop> nearby(double myLng, double myLat, double radiusKm) {
        // GEOSEARCH geo:shops FROMLONLAT {lng} {lat} BYRADIUS {radius} km
        //   WITHDIST ASC COUNT 10
        GeoResults<RedisGeoCommands.GeoLocation<String>> results =
            redis.opsForGeo().search(GEO_KEY,
                new GeoReference.GeoCoordinateReference(myLng, myLat),
                new Distance(radiusKm, Metrics.KILOMETERS),
                RedisGeoCommands.GeoSearchCommandArgs.newGeoSearchArgs()
                    .includeDistance()
                    .sort(SortOrder.ASC)
                    .limit(10));

        List<NearbyShop> shops = new ArrayList<>();
        for (GeoResult<RedisGeoCommands.GeoLocation<String>> r : results) {
            String shopId = r.getContent().getName();
            double distanceKm = r.getDistance().getValue();
            shops.add(new NearbyShop(Long.parseLong(shopId), distanceKm));
        }
        return shops;
    }

    @Data @AllArgsConstructor
    public static class NearbyShop {
        private long shopId;
        private double distanceKm;  // 离我的距离
    }
}
```

### 3.4 GeoHash 原理（面试加分）

```
GeoHash 的核心思想：把二维坐标编码成一维字符串。

地球经纬度范围：
  经度：[-180, 180]
  纬度：[-90, 90]

GeoHash 二分编码过程（以北京 116.404, 39.915 为例）：

  第 1 步：二分经度 [-180, 180]
    116.404 → 在 [0, 180] 区间 → bit = 1
    下一轮区间 = [0, 180]

  第 2 步：二分纬度 [-90, 90]
    39.915 → 在 [0, 90] 区间 → bit = 1
    下一轮区间 = [0, 90]

  第 3 步：继续二分经度 [0, 180]
    116.404 → 在 [90, 180] 区间 → bit = 1
    
  ... 经度纬度交替二分，每次产生 1 bit
  总共 52 bit（经度 26 bit + 纬度 26 bit）

  这 52 bit 组合成一个 long → 作为 ZSet 的 score

特点：
  相近的地理位置 → GeoHash 分值相近 → ZSet 的 ZRANGEBYSCORE 能高效查
  但 GeoHash 边界处有突变（> 隔壁格子的值可能差很大）→ 需要查周围 8 个格子
```

### 3.5 GEO 底层——ZSet

```bash
# GEO 实际上是用 ZSet 实现的
# 可以验证：直接用 ZSet 命令操作 GEO 的 key

127.0.0.1:6379> TYPE geo:shops
"zset"

127.0.0.1:6379> ZRANGE geo:shops 0 -1 WITHSCORES
1) "1"            # shopId=1
2) "4069885363773130"  # GeoHash 编码后的 score
3) "2"
4) "4054986373849128"
```

---

## ⭐️ 三种特殊类型对比

| 维度 | Bitmap | HyperLogLog | GEO |
|------|--------|-------------|-----|
| **底层类型** | String（位操作） | String（16384 个 6-bit 桶） | ZSet（GeoHash 编码） |
| **内存占用** | 按 bit 数（1亿=12.5MB） | 固定 12KB | 按元素数（同 ZSet） |
| **精确度** | 精确（0/1） | 概率（0.81% 误差） | 精确（经纬度微小误差） |
| **典型场景** | 签到、DAU、布隆过滤 | UV、独立IP统计 | 附近的人、附近的店 |
| **能做什么** | 统计是/否 | 统计去重计数 | 距离计算、范围查询 |
| **不能做什么** | 不能存复杂数据 | 不能列具体元素 | 不能做地理面积计算 |

---

## ⭐️ 面试题汇总

**Q1: 为什么 1 亿 DAU 用 Bitmap 只需要 12.5MB？**

> 1 亿 bit / 8 = 12.5MB。每个 bit 表示一个用户的活跃状态（1=活跃，0=不活跃）。相对于 MySQL（800MB+索引）或 Redis Set（~5GB），Bitmap 对"是/否"这种二值信息极其高效，且 BITOP AND 做连续活跃等位运算也极快。

**Q2: PFCOUNT 统计 UV 误差 0.81% 怎么来的？1 亿 UV 比 Set 省多少内存？**

> 误差公式：1.04 / √16384 ≈ 0.81%。内存：HLL 固定 12KB，Set 要 ~5-8GB。省了约 40-60 万倍。代价是不能列出具体哪些用户、有 0.81% 统计误差。

**Q3: GEO 底层是什么？为什么用 ZSet？**

> GEO 底层是 ZSet。经纬度通过 GeoHash 编码成 52-bit 整数作为 ZSet 的 score，member 是地点名。ZSet 的 ZRANGEBYSCORE 天然支持"按 score 范围查询"→"附近的地点查询"。GEORADIUS 本质上就是 ZRANGEBYSCORE 的封装。

**Q4: PFMERGE 合并两个 HLL 时是求和吗？为什么不是？**

> 不是求和，是取每个桶的最大值。因为桶的值代表"见过的最长连续 0 的个数"。合并两个统计数据=取各自视角的最大值。如果求和会把相同的用户重复计入，造成严重高估。

**Q5: Bitmap 和 HyperLogLog 都用于统计，怎么选？**

> Bitmap 用于"已知用户集的二值状态"（如当日活跃/签到），1 亿用户=12.5MB，精确。HyperLogLog 用于"不关心具体用户是谁，只关心去重总数"（如 UV），12KB 固定，0.81% 误差。如果要知道具体是哪 100 万个人活跃 → 用 Set 或 Bitmap（用 bit 位标用户 ID）。

---

*Created: 2026-05-26 | Category: 18-Redis特殊数据类型*
