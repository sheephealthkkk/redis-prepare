# Redis 三种特殊数据类型 —— 深度详解

---

## 前言：为什么叫"特殊"类型？

Redis 的五种基础类型（String/List/Hash/Set/ZSet）是通用型数据结构。而 **Bitmap、HyperLogLog、GEO** 是面向特定场景深度优化的——用极少的内存解决特定问题。它们底层都复用了已有的数据结构:rocket::rocket::rocket::rofl::rocket::rocket::rocket:（Bitmap→String，HyperLogLog→String，GEO→ZSet），但对外暴露的命令完全不同。

```
三种特殊类型的定位：

  Bitmap      → 统计"是非"类数据       （今天签到没？今天活跃没？）
  HyperLogLog → 统计"去重计数"         （有多少不同的访客？）
  GEO         → 统计"地理位置"相关      （附近的人？离我多远？）
```

---

## 第一章：Bitmap（位图）

### 1.1 什么是 Bitmap？:rocket:

**Bitmap 不是一种新的数据类型——它是对 String 的"位操作"。** String 最大 512MB = 2^32 位，可以表示 42 亿:rocket:个不同的标志位。

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

### 1.2 核心命令:rocket::o::o::o::o::o::o::o::o:

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

因为 Redis Bitmap 底层是一个 String，可以支持长达 2^32 位（512MB），所以理论上能容纳 40 亿用户，但**实际使用时用户 ID 必须是非负整数，并且不能过大，否则会撑爆内存**。

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
核心思路：用 Bitmap 存储每日活跃用户
Key：格式为 dau:2026-06-26，代表某一天的活跃记录。

Offset：直接使用用户 ID（userId）作为位偏移量。

Value：1 表示该用户当天活跃，0 表示不活跃。
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

**如果用户 ID 是全局 ID**（比如雪花算法生成的 64 位长整型、或 UUID、或分布式自增 ID），直接拿它当 Bitmap 的 offset 就会踩坑：

---

### 1. 问题所在：Bitmap 的 offset 是按最大 offset 分配内存的

Redis 的 Bitmap 是字符串，**offset 有多大，字符串就得多长**。  
内存占用 = `(max_offset / 8)` 字节。  
如果你的用户 ID 是 10 亿（1000000000），光这一个 Bitmap 就要占 **125MB**，如果是雪花 ID，可能达到几十亿甚至更大，内存直接爆炸。

更糟的是：**全局 ID 往往不连续**，雪花算法的 ID 包含了时间戳、机器标识，虽然趋势递增但中间有大量空洞，那些从未用到的 offset 也会被分配内存并填 0，造成巨大浪费。

---

### 2. 解决方案：给用户 ID 做一个“映射”，变成紧凑的序号:o::o::o::o::o::o::o::o::o::o:

核心思路：**用一套内部自增的小整数 ID，替代全局 ID 作为 Bitmap 的 offset。**

#### 方案 A：用户注册时分配“活跃统计专用序号”:rocket::rocket:
- 用 Redis 的自增计数器 `INCR global:user:seq` 给每个用户生成一个连续的数字（1,2,3…）。
- 把这个序号和用户全局 ID 的双向映射存起来（Hash 或 String）。
- 打点活跃时，先根据全局 ID 找到序号，再用序号作为 Bitmap 的 offset。

**优点**：序号完全连续，Bitmap 极其紧凑，一位都不会浪费。  
**缺点**：多一次映射查询，需要维护映射表；用户量极大时映射表本身不小（但仍是 O(N) 的 key-value，约每用户几十字节）。

**代码示例**：
```java
public void markActive(String globalUserId) {
    // 获取或创建内部序号
    String seqKey = "user:seq:" + globalUserId;
    Long seq = redis.opsForValue().increment(seqKey); // 首次访问会创建
    // 但这样每个用户一个 key，不够优。更好的方式：
    // 用一个全局 Hash 存储映射： HSET user:seqmap globalUserId seq
    // 序号预先用 INCR 生成。
    ...
}
```
建议使用一个全局 Hash `user:seq`，field=globalUserId，value=内部序号。序号用 `INCR seq:next` 预分配。

#### 方案 B：对全局 ID 取模，容忍少量碰撞:o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o::o:
- `offset = hash(globalUserId) % MAX_USERS`，比如取 1 亿。
- 这样 Bitmap 固定大小（约 12.5MB）。
- 但会有哈希碰撞，导致不同用户共用一个位，统计结果有误差（多算）。
- **适合允许少量误差的场景**，例如大概的 DAU 趋势监控。

#### 方案 C：不使用 Bitmap，改用 HyperLogLog
- 如果只需要统计“大概的独立用户数”（UV），不需要“某个用户是否活跃”“连续活跃”等细节，直接用 `PFADD dau:20260626 globalUserId`。
- 内存固定 12KB，不管用户量多大。
- 无法做 BITOP 计算连续活跃，只能做 `PFCOUNT` 和 `PFMERGE`。

#### 方案 D：在应用层使用 Roaring Bitmap
- Roaring Bitmap 是一种压缩位图，对稀疏大整数高效压缩。
- 可以将活跃的用户 ID 收集到应用内存，用 Roaring Bitmap 序列化后存到 Redis String。
- 操作都在应用端完成（如 AND/OR/COUNT），Redis 只当存储。
- 性能好，内存占用远低于原生 Bitmap，适合海量不连续 ID。





### 1.4 经典场景：用户签到:o::o:

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

其他场景

### 布隆过滤器（缓存穿透拦截）
- **用法**：用Bitmap实现一个概率型集合，存所有存在的ID的哈希位。查询时若任一位为0则一定不存在，全为1则可能存在。
- **命令**：多个哈希函数计算位偏移，用 `SETBIT` 插入，`GETBIT` 查询。
- **优势**：内存极小，能快速过滤掉绝大部分恶意不存在的请求，保护数据库。

###  权限控制 / 功能开关（位掩码）:o::o::o::o::o::o::o::o::o::o::o::o::o::o:
- **用法**：一个用户用一个Bitmap表示其拥有的权限集，第0位代表查看，第1位代表编辑，第2位代表删除等。
- **命令**：`SETBIT perm:userId 2 1` 表示赋予删除权限，`GETBIT` 判断权限。
- **优势**：一个key即可表示数十种权限，位运算判权极快，存储成本极低。

### 用户行为漏斗 / 留存分析
- **用法**：例如计算“某日活跃用户在次日是否还活跃”。将两天的BitMap做 `BITOP AND`，结果的 `BITCOUNT` 即留存数。
- **命令**：`BITOP AND result day1 day2` → `BITCOUNT result`
- **优势**：海量用户跨天分析仅需毫秒级运算，比数据库JOIN高效无数倍。

### 视频 / 内容播放去重（已读/未读）
- **用法**：用户ID作为offset，视频ID映射到某一位（可用哈希），记录是否播放过。
- **命令**：`SETBIT video:123  userIdOffset 1`，`GETBIT` 判断。
- **优势**：无需为每个视频维护巨大的Set，内存可控。

### 抽奖活动参与记录（去重）
- **用法**：活动ID为key，用户ID的映射序号为offset，参与即置1。
- **命令**：`SETBIT lottery:act001 userSeq 1`，`GETBIT` 判断是否已参加。
- **优势**：高并发下原子性保证（单条命令），且不会重复扣减库存。

### 黑白名单 / IP封禁:o::o::o:
- **用法**：将IP转为整数作为offset，存入封禁名单。
- **命令**：`SETBIT banlist ipInt 1`，查询时 `GETBIT`。
- **优势**：毫秒级判断，空间效率远超Hash/Set。

### 数据压缩存储（如 Roaring Bitmap）
- **用法**：对于稀疏大整数ID，在应用层用Roaring Bitmap压缩，再存入Redis的String。
- **命令**：`SET key compressedBytes`，操作在应用端完成。
- **优势**：比原生Bitmap更省内存，同时保留快速集合运算能力。

---

### 总结：为什么用Bitmap？
- **空间效率**：存储上亿个布尔值仅需十几MB。
- **CPU效率**：单个位操作为O(1)；多个key聚合用 `BITOP` 在服务端完成，免去数据拉取。
- **原子性**：`SETBIT`、`BITOP` 等命令天然原子，并发安全。
- **简单易用**：无需复杂的数据模型，直接映射offset即可。

只要涉及:o::o::o::o::o::o::o::o:**海量、稀疏、布尔标志**:o::o::o::o:的集合，Bitmap几乎都是最优解。

### 1.5 SETBIT 的底层实现:rocket::rocket::rocket:

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

### 核心原理：假设你开了一家奶茶店

你想知道**今天大概来了多少个不同的客人**，但你不想一个一个记名字（太累，脑子记不住）。

你想到一个**很懒的办法**：

你只在纸上记一个数字：**“今天客人报的生日日期里，最大的那个日期”**。

---

### 规则是这样的

每个客人进门时，你让他随便说一个自己的生日日期（比如3号、15号、28号）。

你只看他说的数字，然后和你纸上已经记的数字比较：
- 如果他说的是 **28号**，你纸上记的最大是 **15号** → 你把纸上的数字改成 **28**
- 如果他说的是 **5号**，你纸上记的最大是 **28号** → 纸上还是 **28**，不用改

晚上关门时，你纸上记的是 **28号**。

---

### 用这个数字猜总人数

你根据经验知道一个规律：
- 如果客人很少（比如只来了2个人），他们报的生日日期里，最大值一般很小（比如最多到5号）
- 如果客人很多（比如来了100个人），几乎一定会有人报出28号、29号这种大日期

所以你看到纸上写的 **28号**，你推断：  
**今天大概来了 28 个左右的客人。**

当然这个推断可能有点误差（可能来了30个人也只报到25号，或碰巧来了一个人就报了28号），但**大方向是对的**：
- 纸上数字越大 → 来的人越多
- 纸上数字越小 → 来的人越少

---

### 这就是 HyperLogLog 的核心思想

| 奶茶店例子         | HyperLogLog                               |
| ------------------ | ----------------------------------------- |
| 客人               | 每个访问用户                              |
| 报生日日期         | 把用户 ID 转换成一个随机数:o::o::o::o::o: |
| 纸上只记最大日期   | Redis 只记所有随机数里最大的那个          |
| 用最大日期猜总人数 | 用数学公式（2^最大随机数）估算 UV         |
| 纸永远不会写满     | 内存固定 12KB                             |

：报生日日期就是报id转成随机数，那如果是随机生成了一个很大的数呢，那靠这个分桶取平均:rocket::rocket::rocket::rocket::rocket::rocket::rocket::rocket::rocket:

**核心就是：不记具体是谁，只记一个“最大值”，然后用这个值反推大概有多少人。**

---

### 但只记一个数字不太准，所以 Redis 记了 16384 个

Redis 不是只记一个最大值，而是准备了 **16384 张小纸条**（桶）。

每个客人进来时，Redis 根据客人 ID 随机分配一张纸条，然后在那张纸条上更新最大值。

关门时，16384 张纸条每张都能算出一个“人数”，最后取平均值，结果就非常稳定了。



“最大连续 0 的个数”这玩意儿，乍一看确实很抽象，不知道它能干嘛。但它的作用，用一句大白话说就是：

**“连续 0 的个数越大，说明这件事越罕见；这件事越罕见，说明你试过的次数一定越多。”**

HyperLogLog 就是靠“你遇见的最罕见的事儿有多罕见”来反推你总共试了多少次（也就是来了多少不同用户）。

### 一步一步拆开

#### 1. 哈希值 = 随机抛硬币
每个用户 ID 被哈希成一个 64 位的二进制数，每位是 0 或 1，随机且独立。

我们只看这个数的“结尾连续 0 的个数”：
- `...0001` → 连续 0 个（最后一位是 1）
- `...0010` → 连续 1 个 0
- `...1000` → 连续 3 个 0
- `...0000000001` → 连续 9 个 0

#### 2. “连续 k 个 0”这件事有多罕见？
每一位出现 0 的概率是 1/2，连续出现 k 个 0 再碰到一个 1，概率是：
```
(1/2)^k * (1/2) = 1 / 2^(k+1)
```
- k=0（即最后一位是1）：概率 1/2，很常见
- k=3（`...1000`）：概率 1/16，有点少见
- k=9（`...0000000001`）：概率 1/1024，很罕见
- k=20：概率约百万分之一，极其罕见

#### 3. 从“最罕见的事件”反推总次数
如果你抛硬币抛了 10 次，你就别想看到连续 20 个反面——那需要的次数远远超过 10 次。  
相反，如果你告诉我你抛硬币时，最高纪录是连续 9 个反面，那我猜你抛了大约 2^10 = 1024 次，才能碰上一次这种罕见情况。

> **核心逻辑：最罕见事件的稀有程度，和总尝试次数正相关。**

所以 HyperLogLog 把所有用户 ID 的哈希值看一遍，找到最大的“连续 0 个数”（记作 K_max），然后用 2^(K_max+1) 估算总共有多少不同用户。

---

### 4. 为什么用“最大连续 0”而不是别的？
因为**最大值对极端事件最敏感**。  
我们只想找一个统计量，能随着人数增多而稳定增大，这样反推才有效。

- 如果你用“平均值”，那增加再多用户，平均连续 0 的个数也就 1~2 之间，变化太小，无法估计。
- 用“最大值”则不同：100 个人时，K_max 大约 5~6；1000 个人时，K_max 大约 8~9；100 万个人时，K_max 大约 18~19。它随基数对数增长，非常适合做估计。

但用单个最大值很容易被偶然的极端值带偏（例如才 10 个人就碰巧有个连续 7 个零），所以才加入 **分桶 + 调和平均** 来平滑。

---

### 5. 举个完整的例子

假设今天实际 UV = 10 人。

哈希后，每个用户的“连续 0 个数”分别是：
```
用户1: 0
用户2: 2
用户3: 1
用户4: 3   ← 最大
用户5: 0
用户6: 2
用户7: 1
用户8: 4   ← 最终最大值
用户9: 0
用户10: 1
```

K_max = 4。  
粗估：2^(4+1) = 32 人，和真实的 10 人偏差有点大。  
这就是只用单个最大值的弱点。

现在 Redis 把这 10 个人分到 16384 个桶里，每个桶独立记录自己的 K_max，最后取调和平均，就能把这种波动熨平，估计值会非常接近 10。

---

### 总结一句话

> **“最大连续 0 的个数”是用来衡量“最倒霉（最罕见）事件”有多倒霉。越倒霉，说明你试过的次数（不同用户数）一定越多。HyperLogLog 就是靠这个来推断 UV 的。**















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

---

### GEO 的底层：ZSet + GeoHash

Redis 的 GEO 并没有发明新的底层结构，而是直接**复用了 ZSet**。

当你执行：
```bash
GEOADD locations 116.404 39.915 "Beijing"
```
Redis 内部做了两件事：
1. 把经纬度 `(116.404, 39.915)` 编码成一个 52 位的整数（GeoHash 的变种）。
2. 将这个整数作为 **ZSet 的 score**，把 `"Beijing"` 作为 member，放入一个 ZSet 中。

所以，`locations` 实际上是一个 ZSet，member 是地点名，score 是经纬度编码后的整数。

---

### GeoHash 原理：把地球“切成小格子”

#### 2.1 核心思路
- 地球上的任意一个点，可以通过**不断二分区间**的方式，变成一串二进制编码。
- 经度范围 [-180, 180]，纬度范围 [-85.05, 85.05]（因为极地不用）。
- 二分区间：
  - 经度首次二分：[-180, 0) 为 0，[0, 180] 为 1。116.404 落在 [0, 180] → 第一位为 1。
  - 然后继续二分这个子区间，得到更细的位。
- 经度和纬度的位**交替组合**（经度占偶数位，纬度占奇数位），形成最终的二进制串。
- 这个二进制串每 5 位转成一个 base32 字符，就是常见的字符串 GeoHash（如 `wx4g0b`）。但 Redis 不使用字符串，而是直接把整个 52 位二进制当作一个整数。

#### 2.2 精度与长度
- Redis 的 52 位整数能表达约 **0.6 米** 的精度，足够绝大多数 LBS 场景。
- 位数越多，格子越小，精度越高。
- 位数越少，格子越大，范围搜索时就能用 score 范围快速框定候选集。

---

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

###  如何实现“附近的人”？

利用 ZSet 的范围查询能力：**score 相近的点，地理位置也相近**（这是 GeoHash 的最关键特性）。

假设你想找 `(116.404, 39.915)` 周围 5 公里内的地点：

1. **计算目标点的 GeoHash 整数**：`target_hash`。
2. **估算半径对应的 GeoHash 差值**：因为经纬度每变化一定度数对应固定距离，Redis 可以算出需要查询的 score 范围 `[target_hash - delta, target_hash + delta]`。
3. **用 ZSet 命令取区间内的 member**：`ZRANGEBYSCORE locations min max`。
4. **精确过滤**：因为 GeoHash 的矩形区间可能会多包含一些边缘的点（不在圆形半径内），Redis 会计算这些候选点的实际距离，剔除超出半径的点。
5. **排序返回**。

Redis 的命令如 `GEORADIUS`（6.2 后推荐 `GEOSEARCH`）就封装了上述全部步骤。

---

### 4. 命令速览

| 命令                                                 | 作用                              |
| ---------------------------------------------------- | --------------------------------- |
| `GEOADD key lon lat member`                          | 添加地点                          |
| `GEOPOS key member`                                  | 获取地点经纬度                    |
| `GEODIST key m1 m2 [m|km|mi|ft]`                     | 计算两点距离                      |
| `GEORADIUS key lon lat radius unit [WITHDIST] [ASC]` | 查找指定点周围的地点              |
| `GEORADIUSBYMEMBER key member radius unit`           | 查找指定 member 周围的地点        |
| `GEOHASH key member`                                 | 获取 GeoHash 字符串               |
| `GEOSEARCH key ...` (Redis 6.2+)                     | 更灵活的附近搜索（支持矩形/圆形） |

**示例**：
```bash
GEOADD cities 116.404 39.915 beijing 121.473 31.230 shanghai
GEORADIUS cities 120.0 35.0 1000 km WITHDIST ASC
```

---

### 5. 为什么 GeoHash 能保证“附近”对应“score 相近”？

- 二分编码保证了：**两点共享的前缀越长，它们离得越近**。
- 在 ZSet 的 score 排序中，前缀相同的值会连续排列，所以范围查询能快速截取候选集。
- 但注意：GeoHash 的“相邻”是沿着 Z 型曲线填充的，有些边界情况（如赤道两侧的点）可能离得很近但 score 差很多。Redis 会通过扩大搜索范围+精确过滤来解决。

---

### 6. 注意事项和局限

- **数据存储在同一个 ZSet**：大量地点时，ZSet 操作仍可能阻塞，需注意 member 数量。
- **精确度够用但不绝对**：约 0.6 米误差，对于民用足够了。
- **不适合极端高并发大量写**：每次 `GEOADD` 都是一次 ZSet 的插入，时间复杂度 O(log N)。
- **不能直接存储多边形**：GEO 只支持点，如果要实现电子围栏（多边形内），需要额外处理（如射线法在应用层计算）。

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
