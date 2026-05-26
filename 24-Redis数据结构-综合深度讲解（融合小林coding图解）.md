# Redis 数据结构 —— 综合小林coding图解 + Java程序员视角深度剖解

---

## 阅读指南

本文融合了两个来源：

- **小林coding《Redis 数据结构》图解** —— 9 种底层数据结构的完整图示，展示从 Redis 3.0 到最新版本的演进
- **Java 程序员视角的源码级解释** —— C 语言概念用 Java 类比讲解，数据结构的内存布局和算法细节

读完你会理解：Redis 的每个数据类型在不同的数据规模和版本下，到底是怎么选择底层存储的。

---

## 前置概念：Redis 数据类型 vs 底层数据结构

很多面试者把"Redis 数据类型"和"底层数据结构"混为一谈。这是两个层面：

```
Redis 数据类型（对外暴露的）：     底层数据结构（内部实现的）：
  String                          ├── SDS（Simple Dynamic String）
  List                            ├── 双向链表
  Hash                            ├── 压缩列表（ziplist，已废弃）
  Set                             ├── 哈希表（dict）
  ZSet                            ├── 跳表（skiplist）
                                  ├── 整数集合（intset）
                                  ├── quicklist
                                  └── listpack
```

**同一个数据类型在不同数据规模下使用的底层结构不同。** 比如 Hash，小数据用 listpack（Redis 7.0）/ ziplist（旧版），大数据用 dict（哈希表）。这是 Redis 最重要的设计策略之一——"一种数据类型，多种底层编码，按数据规模自动切换"。

下面的两幅图来自小林coding的图解，第一幅是 Redis 3.0 版本的对应关系（稍有过时），第二幅是 GitHub 最新代码的对应关系：

![Redis 3.0 数据类型与底层结构对应](./11111_md_output/images/illustration_p01_001.png)

![Redis 最新版数据类型与底层结构对应](./11111_md_output/images/illustration_p01_003.png)

从图中可以看到核心变化：新版本中 **ziplist 被 listpack 全面取代**，而 Set 在大数据时的编码也从 hashtable 加上了 listpack 选项。

---

## 第一章：RedisObject——所有数据的统一包装

### 1.1 为什么需要一个"包装盒"

Redis 用 C 语言编写，C 没有 Java 的多态——没有 `Object obj = "hello"; obj.getClass()` 这种机制。但 Redis 需要在一个统一的键空间（全局 dict）中存储 String、List、Hash、Set、ZSet 五种不同类型的数据。为了让"同一个 dict 能存任何一种类型的 value"，Redis 给每个 value 包了一层统一的头部——`redisObject`。

![RedisObject 结构](./11111_md_output/images/illustration_p25_001.png)

### 1.2 redisObject 的 16 字节

```
redisObject（16 字节 = 4 个字段）：

┌──────────────┬──────────────┬────────────────────┬────────────┬──────────────┐
│ type         │ encoding     │ lru (24 bit)       │ refcount   │ *ptr         │
│ 4 bit        │ 4 bit        │ LRU/LFU 淘汰用     │ 4 字节     │ 8 字节(指针)  │
│ 对外的逻辑类型 │ 内部编码方式  │                    │ 引用计数    │ 指向实际数据   │
└──────────────┴──────────────┴────────────────────┴────────────┴──────────────┘
     ← 4 字节(bit字段)→  ← 4 字节(lru+refcount) →        ← 8 字节(指针) →
                                                                 共 16 字节
```

| 字段 | 大小 | 含义 | Java 类比 |
|------|------|------|----------|
| `type` | 4 bit | 对外的逻辑类型（STRING/LIST/HASH/SET/ZSET） | `obj instanceof String` |
| `encoding` | 4 bit | 内部用什么数据结构存的（INT/EMBSTR/RAW/HT/SKIPLIST...） | 底层实现标记（ArrayList vs LinkedList） |
| `lru` | 24 bit | 存 LRU 时间戳或 LFU 计数器 | Guava Cache 的 accessTime |
| `refcount` | 4 字节 | 引用计数（共享对象用，如 0~9999 整数） | 简化版 GC |
| `*ptr` | 8 字节 | 指向实际数据的指针 | **Java 引用！存的是内存地址** |

### 1.3 type 和 encoding 的关系——解耦的智慧

`type` = 用户视角（`TYPE key` 查看），`encoding` = 内部视角（`OBJECT ENCODING key` 查看）。两者解耦——同一个 type 可以有多种 encoding，在不同数据规模下自动切换。这就像 Java 的 `List<String> list = new ArrayList<>()` vs `new LinkedList<>()`——你用 List 接口，不关心底层。

![type 和 encoding 的对应关系](./11111_md_output/images/illustration_p27_001.png)

---

## 第二章：SDS（Simple Dynamic String）

### 2.1 C 语言的 `char*` 为什么不能用

C 语言的"字符串"本质是一个 `char*`指针，指向连续的内存区域，以 `\0` 结尾。这种设计有三个致命问题：

1. **获取长度 O(N)：** 必须遍历到 `\0`才能知道长度
2. **缓冲区溢出：** `strcat`拼接字符串时不检查空间，写越界直接覆盖后面的内存→安全漏洞
3. **二进制不安全：** 数据中包含 `\0`会被当作结束符，无法存图片、Protobuf

Java 的 String 没有这些问题——底层是 `byte[]`，长度由数组 `.length` 决定。SDS 就是 C 语言版的"Java String 替代品"。

### 2.2 SDS 的结构设计

![SDS 结构图](./11111_md_output/images/illustration_p05_001.png)

Redis 3.2+ 把 SDS 分成了 5 种版本，头部从 1 字节到 17 字节：

```
SDS 版本     │ 头大小│ 适用数据        │ Redis 内部名
────────────┼──────┼───────────────┼────────────
sdshdr5(废弃)│ 1B   │ < 32 字节      │ TYPE_5
sdshdr8     │ 3B   │ < 256 字节     │ TYPE_8  ← embstr 用
sdshdr16    │ 5B   │ < 65536 字节   │ TYPE_16
sdshdr32    │ 9B   │ < 4GB          │ TYPE_32
sdshdr64    │ 17B  │ ≥ 4GB          │ TYPE_64
```

**以 sdshdr8 为例（3 字节头部）：**

```
┌─────────────┬──────────────┬──────────────┬────────────────────┐
│ len(1B)     │ alloc(1B)    │ flags(1B)    │ buf[] (柔性数组)    │
│ 已用长度     │ 总容量(-1)    │ SDS类型标记   │ 实际数据 + '\0'    │
└─────────────┴──────────────┴──────────────┴────────────────────┘
          ← 固定 3 字节 = sizeof(sdshdr8) →
```

**SDS 扩容策略（类比 ArrayList 的 grow）：**

![SDS 扩容过程](./11111_md_output/images/illustration_p06_001.png)

```
增长后新长度 <  1MB → 扩容为 2 × 新长度（翻倍）
增长后新长度 ≥  1MB → 扩容为 新长度 + 1MB（固定增量）
```

Java 的 ArrayList 是 1.5 倍，SDS 更激进（翻倍）——因为 Redis 是内存数据库，内存充足，时间比空间更贵。

### 2.3 SDS 和 C 字符串的对比

| 特性 | C 字符串 | SDS |
|------|---------|-----|
| 获取长度 | O(N) 遍历 | O(1) 读 len 字段 |
| 缓冲区溢出 | 容易 | 自动扩容，杜绝 |
| 内存分配次数 | 每次修改都分配 | 空间预分配 + 惰性释放 |
| 二进制安全 | 遇 `\0` 截断 | len 标记长度，存任意字节 |
| 兼容 C 函数 | — | 结尾自动加 `\0` |

---

## 第三章：双向链表——但已经被淘汰了

### 3.1 链表节点和链表结构

![链表节点结构](./11111_md_output/images/illustration_p07_001.png)

![链表结构](./11111_md_output/images/illustration_p07_002.png)

Redis 3.0 时代的 List 在大数据时使用双向链表。每个节点（listNode）有三个字段：`prev`（前驱指针，8 字节）、`next`（后继指针，8 字节）、`value`（值指针，8 字节）。一个节点光是三个指针就 24 字节——如果数据本身才 3 字节，结构开销是数据的 8 倍。

### 3.2 链表的致命缺陷

- **内存不连续**——无法利用 CPU 缓存
- **指针开销大**——每个节点的前后指针 16 字节固定开销
- **内存碎片**——链表节点分散在堆各处

Redis 3.2 引入 quicklist 替代了纯链表。Redis 7.0 进一步用 listpack 替代 ziplist。到现在的 Redis 版本中，**纯双向链表已经基本退出了数据存储的舞台**。

---

## 第四章：压缩列表（ziplist）——已被 listpack 取代

### 4.1 ziplist 的设计初衷

ziplist 是 Redis 为了节省内存而设计的连续内存结构。它把所有元素紧凑地排在一起，用**相对偏移量**代替指针，省去了所有的 8 字节指针开销。

![ziplist 整体结构](./11111_md_output/images/illustration_p08_001.png)

```
ziplist 的整体结构（一整块连续内存）：

┌──────────┬──────────┬──────────┬─────────┬─────────┬─────┬─────────┬────────┐
│ zlbytes  │ zltail   │ zllen    │ entry1  │ entry2  │ ... │ entryN  │ zlend  │
│ 4B(总字节)│ 4B(尾偏移)│ 2B(元素数)│ 变长    │ 变长     │     │ 变长    │ 0xFF   │
└──────────┴──────────┴──────────┴─────────┴─────────┴─────┴─────────┴────────┘
     ←──────────── 固定 10 字节 header ────────────→
```

### 4.2 ziplist entry 的结构与致命问题

![ziplist entry 结构](./11111_md_output/images/illustration_p09_001.png)

每个 entry 由三部分组成：`prevlen`（前驱长度）、`encoding`（编码+长度）、`data`（实际数据）。**`prevlen` 就是一切问题的根源。**

![连锁更新过程](./11111_md_output/images/illustration_p10_001.png)

![连锁更新详细图解](./11111_md_output/images/illustration_p10_002.png)

**连锁更新——ziplist 最大的缺陷：**

```
前驱长度 < 254 字节 → prevlen 占 1 字节
前驱长度 ≥ 254 字节 → prevlen 占 5 字节（1B 标记 0xFE + 4B 存真实值）

连锁触发场景：所有 entry 长度刚好在 253 字节
  在头部插入一个 254 字节的新 entry →
  entry2 的前驱从 253→254 → prevlen 从 1B 变 5B → entry2 从 254B 变 258B →
  entry3 的前驱从 253→258 → prevlen 从 1B 变 5B → ... 
  一直传导到末尾！每个 entry 都扩容 → 每次扩容要 memmove 后续数据 → O(N²)
```

![连锁更新图解 2](./11111_md_output/images/illustration_p11_001.png)

现实中极少触发（所有 entry 整齐落在 253/254 边界需要巧合），但理论上存在。Redis 7.0 用 listpack 彻底根治。

### 4.3 listpack —— ziplist 的继承与改良

![listpack vs ziplist](./11111_md_output/images/illustration_p14_001.png)

![listpack 结构](./11111_md_output/images/illustration_p14_002.png)

![listpack vs ziplist 对比 2](./11111_md_output/images/illustration_p14_003.png)

![listpack vs ziplist 对比 3](./11111_md_output/images/illustration_p14_004.png)

listpack 的解决方案简单而优雅——**不存前驱长度，存自身长度**：

```
ziplist entry：
  [prevlen][encoding+len][data]  ← prevlen 存前驱长度 → 受前驱牵连

listpack entry：
  [encoding+len][data][backlen]  ← backlen 存自身长度 → 不受任何其他 entry 影响

反向遍历：读当前 entry 的 backlen → 知道当前 entry 总长 → 跳到前一个 entry
```

![listpack 演进](./11111_md_output/images/illustration_p24_001.png)

---

## 第五章：哈希表（dict）——Redis 的核心引擎

### 5.1 dict 的三层结构

dict 是 Redis 中最核心的数据结构——全局键空间、Hash 大数据编码、Set 大数据编码、ZSet 的 member→score 映射都用 dict。

![哈希表结构](./11111_md_output/images/illustration_p15_001.png)

```
dict 三层嵌套结构（外层到内层）：

dict（字典总控）
 ├── dictht[0]（哈希表 0，日常使用）
 ├── dictht[1]（哈希表 1，rehash 目标，平时为空）
 └── rehashidx（-1 = 不在 rehash，≥0 = 正在搬迁）

dictht（哈希表）
 ├── **table → 桶数组（dictEntry* 数组）
 ├── size：桶数量（总是 2^n）
 ├── sizemask：size - 1（位运算取模，hash & sizemask）
 └── used：已有 entry 数量

dictEntry（哈希表节点）
 ├── *key：指向 key（SDS 字符串）
 ├── v：union{*val, u64, s64, double}（8 字节，多身份）
 └── *next：链表下一个（链地址法解决冲突）
```

每个 dictEntry 共 24 字节：3 个 8 字节字段。v 使用 union（联合体）——同一块 8 字节空间，有时存指针、有时存 double、有时存整数——避免了为不同 value 类型分配不同大小的 entry。

### 5.2 渐进式 Rehash——Redis 避免阻塞的关键设计

Java 的 HashMap 扩容是一次性把所有元素搬到新表——百万级元素可能耗时几十毫秒。Redis 单线程下这样做会导致明显卡顿。

**渐进式 rehash 的核心思想：把"大搬迁"拆成"无数次小搬迁"。**

![rehash 过程](./11111_md_output/images/illustration_p16_001.png)

```
rehash 的步骤：

启动：
  ① ht[1].size = ≥ ht[0].used × 2 的最小 2^n
  ② rehashidx = 0（从 ht[0] 的第 0 个桶开始）

每次搬迁（触发时机：dict 被访问时——增/删/改/查）：
  ③ 把 ht[0].table[rehashidx] 整个桶的链表搬迁到 ht[1]
  ④ rehashidx++

完成：
  ⑤ rehashidx == ht[0].size（所有桶搬完）
  ⑥ 释放 ht[0]，ht[1] 升格为新的 ht[0]，rehashidx = -1
```

![渐进式 rehash 图解](./11111_md_output/images/illustration_p17_001.png)

**rehash 期间的查找：** 先去 ht[0] 找 → 没找到且 rehash 进行中 → 再去 ht[1] 找。新增只写 ht[1]。

**如果没有任何请求怎么办？** Redis 的定时任务 `serverCron` 每 100ms 运行一次，每次用 1ms 时间片做 rehash 搬迁。保证空实例的 rehash 也持续推进。

**触发条件（负载因子）：**
- 负载因子（used/size）≥ 1 ——正常触发扩容
- 负载因子 ≥ 5 ——**强制扩容**（不管是否在进行 BGSAVE）

---

## 第六章：整数集合（intset）——纯整数的极致压缩

### 6.1 intset 的结构

![intset 结构](./11111_md_output/images/illustration_p18_001.png)

intset 是有序整数数组，用定长存储，不带任何指针。header 固定 8 字节。

```
┌───────────────┬───────────────┬───────────────────────┐
│ encoding(4B)  │ length(4B)    │ contents[]            │
│ 编码类型       │ 元素个数       │ 按 encoding 定长存储   │
└───────────────┴───────────────┴───────────────────────┘
    固定 8 字节头

encoding 三种取值：
  INTSET_ENC_INT16 = 2  → 每个元素 2 字节
  INTSET_ENC_INT32 = 4  → 每个元素 4 字节
  INTSET_ENC_INT64 = 8  → 每个元素 8 字节
```

### 6.2 编码升级——只升不降

当插入一个超出当前范围的新整数时，intset 会升级编码。比如现有 encoding=INT16 的 set 存 `[1,2,3]`，插入 `65536`（超出 INT16）→ 编码升级到 INT32 → 从后往前逐位搬迁 → 插入新值。

![intset 升级过程](./11111_md_output/images/illustration_p18_002.png)

关键：升级**不可逆**——删掉 65536 后 encoding 仍是 INT32，不会降回去。

---

## 第七章：跳表（skiplist）—— ZSet 的排序引擎

### 7.1 为什么不用红黑树？

Java 的 TreeMap 用红黑树。跳表的优势在于**简单**——Redis 跳表约 200 行 C 代码，红黑树约 2500 行。简单 = 容易实现 = 不容易有 bug = 维护成本低。

### 7.2 跳表的直觉——给链表加"快速通道"

![跳表结构图解](./11111_md_output/images/illustration_p19_001.png)

```
普通有序链表查找 30：
  HEAD → 10 → 20 → 30 → 40 → 50 → 从头往后找，O(N)

加一层"高速公路"：
  level 1: HEAD ────────▶ 30 ────────▶ 50   ← 快速层
  level 0: HEAD → 10 → 20 → 30 → 40 → 50   ← 全量层

查找 30：从高层开始 → HEAD → 30（一步！）→ O(logN)
```

### 7.3 Redis 跳表的完整结构

![跳表节点结构](./11111_md_output/images/illustration_p20_001.png)

![跳表插入过程](./11111_md_output/images/illustration_p20_002.png)

![跳表层高与跨度](./11111_md_output/images/illustration_p20_003.png)

每个节点的高度是随机的——每次插入时以 25%（1/4）概率增加一层，最多 64 层。

**为什么用 0.25 而不是经典跳表的 0.5？** 0.5 会导致更多高层节点（期望 2 层），插入时需更新的指针更多。0.25 让跳表更"矮胖"（期望 1.33 层），插入更快，少量的查找效率损失微乎其微。

### 7.4 span——ZRANK O(logN) 的秘密

span 记录了每一层从当前节点到下一个节点"中间跳过了几个节点"。ZRANK 执行时，不需要从 level 0 一路数——而是在搜索路径上顺手累加 span 值。这使得计算排名也是 O(logN) 而不是 O(N)。

### 7.5 ZSet 的 dict + skiplist 双结构

ZSet 同时维护 dict（member → score，O(1) 查分）和 skiplist（score → member，O(logN) 排序）。两者共享 member 对象（同一份 SDS，不额外内存）。

---

## 第八章：quicklist——链表和压缩列表的混合体

![quicklist 结构](./11111_md_output/images/illustration_p22_001.png)

### 8.1 quicklist 的设计理念

纯 ziplist 省内存但插入慢（需要 memmove），纯 linkedlist 插入快但浪费内存（大量指针）。**quicklist 取两者之长——用双向链表做"骨架"，每个节点内嵌一个小的 ziplist/listpack。**

在 Redis 3.2 ~ 6.2 中，每个 quicklistNode 内嵌的是 ziplist。Redis 7.0+ 替换为 listpack。

### 8.2 配置参数

```bash
list-max-ziplist-size -2  # -2 = 8KB，每个节点最多 8KB
list-compress-depth 1     # 两端各 1 个不压缩，中间 LZF 压缩
```

---

## 第九章：listpack——Redis 7.0 的答案

![listpack 演进](./11111_md_output/images/illustration_p24_001.png)

listpack 和 ziplist 一样是连续内存结构——但没有 prevlen 字段。listpack entry 的结构是 `[encoding][data][backlen]`——backlen 记录自身长度（用变长编码，从右往左解析），数据变化时只影响自己的 backlen，不影响相邻 entry。连锁更新的根源被彻底切除。

listpack 的整数编码也比 ziplist 更灵活——13 种变长编码 vs ziplist 的 6 种。

---

## 第十章：五种数据类型与底层结构的完整映射

看完上面的 9 种底层结构后，回到最初的五大数据类型——它们在 Redis 7.0 中到底用哪些底层结构：

| 数据类型 | 小数据编码 | 大数据编码 | 切换条件（默认） |
|---------|-----------|-----------|---------------|
| **String** | int / embstr | raw | 非整数 or >44字节 |
| **List** | quicklist（统一实现） | — | fill=8KB/节点 |
| **Hash** | listpack | dict | field>512 or value>64B |
| **Set** | intset / listpack | dict | element>512 or 非整数 |
| **ZSet** | listpack | dict + skiplist | element>128 or value>64B |

**核心哲学：** 小数据用紧凑结构（listpack/intset/embstr）——连续内存、无指针、CPU缓存友好。大数据用高效结构（dict/skiplist）——保证操作复杂度可控。**编码升级是单向不可逆的。**

---

## ⭐️ 面试题汇总

**Q1: RedisObject 有哪几个字段？有什么作用？**

> type（4bit，用户看到的类型 STRING/LIST/HASH/SET/ZSET）、encoding（4bit，内部实现 INT/EMBSTR/RAW/HT/SKIPLIST 等）、lru（24bit，淘汰用的访问时间/频率）、refcount（引用计数，共享对象用）、*ptr（指向实际数据）。总共 16 字节。

**Q2: SDS 和 C 字符串相比有什么优势？**

> O(1) 获取长度（读 len 字段而非遍历 `\0`）、自动扩容防溢出、空间预分配减少 malloc 次数、惰性释放不频繁回收、二进制安全（len 标长度，不依赖 `\0`）。

**Q3: 压缩列表（ziplist）为什么被 listpack 取代？**

> ziplist 的 entry 存了前驱长度 prevlen——前驱变大时 prevlen 从 1B 扩展为 5B→当前 entry 也变大→下一个 entry 的 prevlen 也要扩展→连锁更新到末尾，最坏 O(N²)。listpack 存自身长度 backlen——不受前驱影响→根除连锁更新。

**Q4: 渐进式 rehash 解决什么问题？**

> 避免 Redis 单线程在一次性 rehash 时卡顿（百万 key 一次性搬迁可能阻塞几十毫秒到几秒）。把搬迁工作分散到每次 dict 访问中——每次访问搬一个桶。通过 serverCron 定时任务兜底——即使空闲实例也在推进 rehash。

**Q5: 跳表为什么 span 字段是 O(logN) 查排名的关键？**

> span 在跳表的每一层记录了"跳过了几个节点"。ZRANK 搜索目标节点时，在搜索路径上累加 span→沿着高层跳跃绕过绝大多数节点→不需要遍历 level 0 来数→O(logN) 而非 O(N)。

**Q6: quicklist 解决了什么设计矛盾？**

> 纯 ziplist 内存省但插入慢（memmove），纯 linkedlist 插入快但内存废（每个节点 16B 指针）。quicklist = 链表骨架 + 小 ziplist/listpack 做填充→比纯 ziplist 快（节点少搬移少），比纯 linkedlist 省（节点少指针少）。

---

*Created: 2026-05-26 | 融合小林coding图解 + Java 程序员视角 | Category: 24-Redis数据结构-综合讲解*
