# Redis 高可用 —— 集群（Cluster）深度详解

---

## 前言：主从 + Sentinel 的局限性

主从复制 + Sentinel 解决了高可用的问题——Master 宕机后自动切换，几十秒内恢复服务。但有一个根本问题 Sentinel 解决不了：**如果数据量太大，一台机器装不下怎么办？**

```
现实场景：

电商平台运营了 3 年，Redis 缓存的数据量从最初的 2GB 涨到了 200GB。
一台物理机最多 256GB 内存，勉强还装得下——但你知道这只是时间问题。

更大的问题是 fork 阻塞：
  200GB 内存的 Redis → fork 需要拷贝约 400MB 的页表 → 阻塞主线程约 500ms 到 1 秒
  BGSAVE 每 15 分钟触发一次 → 每 15 分钟就卡 1 秒 → 延迟毛刺 → 用户体验恶化

还有 QPS：
  单机 Redis 极限约 10-15 万 QPS → 你的电商 QPS 在大促期间冲到 50 万 → 单机扛不住
  
这些都可以靠"加机器"来解决——但 Sentinel 不支持水平扩展。
Sentinel 的简单模型只能管一个 Master + 多个 Slave，每个 Slave 的数据和 Master 完全相同。
你不能把数据分到多个 Master 上——因为 Sentinel 不知道"哪个 key 在哪个 Master 上"。
```

Redis Cluster 就是为了解决这个问题的——**数据分片 + 高可用，一体搞定**。

---

## 第一章：Hash Slot —— 数据怎么分片？

### 1.1 分片的基本思路——从"一台管所有"到"每人管一部分"

分片的最基本思路是把数据分散到多台机器上，每台机器只存一部分数据。这跟数据库分库分表是一样的道理：当一张表太大，拆成多张表，每张表存一部分行。

但数据和机器之间的对应关系怎么建立？设你有 3 台 Master 节点，需要把一个 key 分配到其中一台。Redis 的做法是：

**① 先把整个键空间切成 16384 个"槽"**

不是直接把 key 映射到机器——那样很难做动态扩缩容。而是在中间加一层"槽（slot）"。16384 个槽号只是逻辑上的概念，和物理机器无关。

**② 每个 Master 分管一部分槽**

在集群初始化时，你需要把 16384 个槽分配给 3 个 Master。例如 Master-1 管 0~5460（5461 个槽），Master-2 管 5461~10922（5462 个槽），Master-3 管 10923~16383（5461 个槽）。

**③ key 通过 CRC16 哈希映射到槽**

当客户端要读写一个 key 时，用 `CRC16(key) % 16384` 算出它属于哪个槽。然后客户端根据本地的"槽→节点"映射表，把请求发给对应的 Master。

```
═══════════════════════════════════════════════════════════════
              key → slot → node 的路由示意
═══════════════════════════════════════════════════════════════

  客户端想执行 GET user:10086
      │
      │ 计算 slot = CRC16("user:10086") % 16384 = 12345
      │
      │ 查看本地路由表：slot 12345 → Master-3 (192.168.1.3:6379)
      │
      ▼
  发送 GET user:10086 ──────────▶  Master-3
                                   slot 10923~16383 都在我这里 ✓
                                   执行 GET → 返回 value
```

### 1.2 为什么是 16384 个槽？

这个数字不是随便选的。整个集群中，节点间的 Gossip 心跳消息包含一个"slots 位图"——用来告诉其他节点"我管哪些槽"。这个位图的大小 = 槽数 / 8 字节。

- 16384 个槽 → 位图 2KB
- 65536 个槽 → 位图 8KB

16384 的选择是"位图大小"和"分布均匀度"之间的折中点。16384 / 3 个 Master ≈ 每台 5461 个槽——分布已经很均匀了。实际部署中很少有人超过几百个节点，17684 完全够用。

### 1.3 Hash Tag——强制多个 key 在同一槽

在单机 Redis 中，MGET、MSET、事务、Lua 脚本可以操作任意多个 key。但在 Cluster 中，如果这些 key 分散在不同的槽（即不同的 Master 节点），一条命令无法跨节点执行。Redis 的解决方案是 Hash Tag：

```
普通 key 的 slot 计算：
  CRC16("user:1001")    → slot X
  CRC16("user:1002")    → slot Y  ← 不同！

使用 Hash Tag（只对 {} 中的部分做 CRC16）：
  CRC16("{user}:1001")  → 只算 "user" → slot X
  CRC16("{user}:1002")  → 只算 "user" → slot X  ← 相同！
```

Hash Tag 的原理是：如果 key 中包含 `{...}`，就只对 `{` 和第一个 `}` 之间的内容做 CRC16。`{user}:1001` 和 `{user}:1002` 的 Hash Tag 都是 `user`→同一 slot→同一节点→MGET/MSET/事务/Lua 都可以用了。

**Hash Tag 的代价：** 如果把所有相关 key 都强塞到同一个 slot → 该 slot 所在的 Master 可能成为热点——大量数据集中在一个节点，失去了分片的意义。所以 Hash Tag 只应在那几个确实需要跨 key 原子操作的场景中使用。

---

## 第二章：Cluster 的请求路由——MOVED 和 ASK

### 2.1 客户端怎么知道"key 在哪个节点"

客户端第一次连接 Cluster 时，不知道任何 slot 信息。它会随机选一个节点连接，发送 `CLUSTER SLOTS` 命令，获取完整的 slot 分布图。返回结果类似：

```
CLUSTER SLOTS
1) 1) (integer) 0       ← slot 起始
   2) (integer) 5460    ← slot 结束
   3) 1) "192.168.1.1"  ← Master IP
      2) (integer) 6379 ← Master Port
   4) 1) "192.168.1.11" ← Slave IP
      2) (integer) 6379 ← Slave Port
2) 1) (integer) 5461
   2) (integer) 10922
   3) ... (第二个 Master 的信息)
...
```

客户端拿到这个结果后，在本地构建一个"槽→节点"映射表，缓存起来。之后的请求都根据这个路由表来发。

**缓存可能过时：** 如果集群发生了 slot 迁移或故障转移，客户端的路由表可能过时。这时客户端发请求到错误的节点→节点返回 MOVED 或 ASK→客户端更新路由表→重试——这个过程称为"重定向"。

### 2.2 MOVED 重定向——"key 不在我这儿，去那边"

MOVED 是"永久性"的重定向——slot 的归属已经变了，你需要更新本地路由表。

```
客户端向 Master-1 发送请求

  Client: GET user:10086
          │
          ▼
  Master-1 收到 → CRC16 → slot=12345
                  → 12345 不在我负责的 0~5460 内！
                  → 我知道 12345 在 Master-3 (192.168.1.3) 那里
          │
          ▼
  Client 收到: MOVED 12345 192.168.1.3:6379
          │
          │ ① 更新本地路由表：slot 12345 → Master-3
          │ ② 重新向 Master-3 发请求
          ▼
  Master-3: GET user:10086 → OK → 返回 value
```

### 2.3 ASK 重定向——"这个 key 正在搬家，临时去那边找"

ASK 和 MOVED 的区别很关键——ASK 是"临时性"的，发生在 slot 迁移的中途。

当一个 slot 正在从节点 A 迁移到节点 B 的途中，slot 中的一些 key 已经被迁移到 B，还有一些留在 A。客户端发请求到 A，如果这个 key 刚好已经被迁移到了 B，A 返回 ASK——表示"这个 key 已经不在这了，去 B 找"。

ASK 和 MOVED 的关键区别：收到 ASK 后，客户端应**执行 ASKING** 命令来请求 B 处理——但不更新本地路由表。因为 slot 的迁移还在进行中，B 还没有正式接管这个 slot——如果更新路由表，后续对这个 slot 的其他 key 的请求就全发给 B，而 B 可能还没有这些 key（还在 A 那里）。

```
ASK 重定向的完整流程：

  Client: GET key_in_migrating_slot
          │
          ▼
  Master-A (源): 这个 key 已经被 MIGRATE 到 Master-B 了
          │
          ▼
  Client 收到: ASK <slot> <Master-B的IP:Port>
          │
          │ ① 不更新路由表（因为 slot 迁移还在进行中）
          │ ② 向 Master-B 发 ASKING 命令（告诉 B"我收到了 ASK 重定向"）
          │ ③ 再向 Master-B 发原命令
          ▼
  Master-B: ASKING 后收到 GET key → 返回 value
```

---

## 第三章：Gossip 协议——Cluster 去中心化的通信机制

### 3.1 为什么不用中心化方案（像 Sentinel 那样）？

Sentinel 是去中心化的（多个 Sentinel 平等），但 Sentinel 只管理**一台 Master + 多个 Slave**——节点数量固定且很少（3~5 个）。Sentinel 只需要监控这少数几个节点。

Cluster 完全不同：一个生产 Cluster 可能有几十到几百个节点。如果像 Sentinel 一样通过一个 Pub/Sub 频道来交换信息——频道中的消息量会爆炸。而且 cluster 需要快速检测节点宕机——十几秒的延迟对于几十个节点的集群来说是不能接受的。

Redis Cluster 选择了**Gossip 协议**——一种去中心化、可扩展的节点通信机制。

### 3.2 Gossip 如何工作

Gossip 的核心思想是 **"你和邻居闲聊，邻居再和他们的邻居闲聊，最终所有人都知道了"**。在 Cluster 中，每个节点每秒随机选择几个其他节点，发 PING 消息。PING 中包含自己知道的集群元数据（哪些节点在线、哪些 slot 归我管）。收到 PING 的节点必须回复 PONG，PONG 中也包含自己的信息。

```
Gossip 通信的节奏：

  每个节点每秒做的事：
    ① 从所有已知节点中随机选 5 个（最少 1 个，最多 1/10 的集群大小）
    ② 如果其中某个节点上次 PONG 超过 cluster-node-timeout/2 → 优先 PING
    ③ 向选中的节点发送 PING
       PING 中携带：
         - 本节点的名称、IP、Port、flags（角色/状态）
         - 本节点管理的 slot 位图（2KB）
         - 本节点对随机几个其他节点的认知（用于传播）
    
  收到 PING 的节点：
    ① 检查 PING 中是否有自己不知道的新节点 → 有则加入本地节点列表
    ② 更新自己对各节点的认知（flags、slot 分布）
    ③ 回复 PONG（包含自己的信息）
```

### 3.3 故障检测——PFAIL 和 FAIL

Cluster 的故障检测用到了 Gossip 的两阶段特性：

**PFAIL（Probable Fail，可能下线）**

一个节点 A 向节点 B 发 PING → 超过 `cluster-node-timeout` 时间没有 PONG → A 在自己的本地视图里标记 B 为 PFAIL（可能下线）。这相当于 Sentinel 中的 SDOWN——只是 A 的个人判断。

**FAIL（确认下线）**

节点 A 的 PFAIL 标记通过 Gossip PING 传播给其他节点。当一个 Master 收到 PING 消息，发现消息中标记了某个节点是 PFAIL → 它会检查"有多少个 Master 同时标记这个节点是 PFAIL"（包括自己）。当达到半数以上（≥ N/2+1，N 是所有 Master 的数量）时——将该节点标记为 FAIL → 在自己的本地视图更新 → 在自己的下一个 PING 中广播这个 FAIL 标记。

被标记为 FAIL 后——如果 FAIL 的是 Slave → 仅标记，不做其他处理。如果 FAIL 的是 Master → 触发故障转移！该 Master 的所有 Slave 竞争上岗。

### 3.4 故障转移——Slave 竞选新 Master

```
═══════════════════════════════════════════════════════════════
          Cluster 故障转移的完整流程（Gossip 视角）
═══════════════════════════════════════════════════════════════

  ① Master-1 宕机
  ② 其他 Master 通过 PING 超时 → 标记 Master-1 为 PFAIL
  ③ Gossip 传播 PFAIL → 过半 Master 确认
  ④ Master-1 被标记为 FAIL → Gossip 广播 FAIL 消息
  
  ⑤ Master-1 的所有 Slave 收到 FAIL 消息
     Slave-1a 和 Slave-1b 都开始准备竞选

  ⑥ 竞选规则：
     - 谁的 offset 更大（数据更新的）→ 优先
     - 谁的 rank 更小（先发起竞选的）→ 优先

  ⑦ 胜出的 Slave 向其他 Master 发起投票请求（FAILOVER_AUTH_REQUEST）
     需要 ≥ N/2+1 个 Master 同意（N 是所有 Master 数量，包括宕机的 Master-1）

  ⑧ 获得足够票数 → Slave 执行故障转移：
     - 标记自己的 role 为 Master
     - 接管 Master-1 的所有 slot
     - 广播 PONG（携带最新的 slot 信息）通知所有节点

  ⑨ 客户端收到 MOVED 重定向 → 更新路由表 → 请求发送到新 Master

═══════════════════════════════════════════════════════════════
```

---

## 第四章：Slot 迁移——扩缩容的关键

### 4.1 为什么要迁移 slot？

当你往 Cluster 中添加新节点时，新节点本来是空白的（没有任何 slot 也没有任何数据）。你需要把一些 slot 从老节点迁移到新节点——这就是 `redis-cli --cluster reshard` 做的事情。

迁移本质上是"把某个 slot 的所有 key 从 Master-A 搬到 Master-B"。

### 4.2 迁移的底层流程——MIGRATE 逐 key 搬

这个过程的精妙之处在于——它不是"把整个 slot 原子地搬过去"——而是逐个 key、逐个 key 地搬。这保证了迁移过程中集群仍然可以正常接受请求：

```
slot 1000 从 Master-1 迁移到 Master-2 的完整流程：

第一步：设置迁移标志
  Master-2: CLUSTER SETSLOT 1000 IMPORTING <Master-1的NodeID>
            → "我准备接手 slot 1000，数据从 Master-1 来"
  
  Master-1: CLUSTER SETSLOT 1000 MIGRATING <Master-2的NodeID>
            → "我正在把 slot 1000 搬到 Master-2 去"

第二步：逐 key 迁移
  Master-1 获取 slot 1000 中的所有 key
  对每个 key：
    MIGRATE Master-2的IP Master-2的Port <key> 0 <timeout>
           │               │              │  │     │
           │               │              │  │     └── 迁移超时
           │               │              │  └── 单 key 模式
           │               │              └── key 名
           │               └── 目标节点端口
           └── 目标节点 IP
    
  这条命令在 Master-1 上原子执行：
    ① 把 key 的序列化数据发送到 Master-2
    ② Master-2 接收 → 存入自己的数据库
    ③ Master-1 确认 Master-2 成功存入后 → 删除自己的这个 key
    ④ 整个过程中这个 key 在 Master-1 上被"锁住"（其他请求等待）

第三步：完成标志
  全部 key 迁移完毕后：
  Master-1 和 Master-2 都执行：
    CLUSTER SETSLOT 1000 NODE <Master-2的NodeID>
    → "slot 1000 正式归 Master-2 管理"
  
  → 这个信息通过 Gossip 传播给所有节点
```

### 4.3 业务场景——双十一前的集群扩容

```
═══════════════════════════════════════════════════════════════
              某电商双十一扩容方案
═══════════════════════════════════════════════════════════════

  平日：
    Cluster: 3 主 3 从 = 6 台机器
    每台 Master ~30GB 数据，QPS ~5万/秒
    
  双十一前 2 周：
    预估大促峰值 QPS ~20万/秒，数据量可能增长到 ~150GB
    
    扩容到 5 主 5 从 = 10 台机器：
      ① 准备 2 台高配服务器（作为新 Master）
      ② 准备 2 台普通服务器（作为他们的 Slave）
      ③ redis-cli --cluster add-node：加入新 Master
      ④ redis-cli --cluster add-node --cluster-slave：加入新 Slave
      ⑤ redis-cli --cluster reshard：
          从 3 个老 Master 各自移出 ~2000 个 slot 给新 Master
          迁移时间约 5-10 分钟（取决于数据量）
          迁移期间对线上无影响（只有被迁移的 key 在瞬间不可用）
      ⑥ 验证 slot 分布均匀

  双十一后：
    缩容回 3 主 3 从：
      ① reshard：把新 Master 的 slot 迁回老 Master
      ② del-node：移除新 Master 和 Slave
```

---

## ⭐️ 面试题汇总

**Q1: Redis Cluster 的 slot 是怎么计算的？为什么是 16384？**

> `slot = CRC16(key) % 16384`。16384 的选择是 Gossip 心跳消息中位图大小的折中——16384 个 slot = 2KB 位图；如果选 65536 = 8KB 位图→PING/PONG 消息太大。2KB 的心跳消息和分布均匀度达到平衡。

**Q2: MOVED 和 ASK 有什么区别？**

> MOVED = slot 永久迁移完毕→客户端应更新本地路由表。ASK = slot 正在迁移中→这个 key 暂时在另一个节点→客户端不应更新路由表（因为迁移还没完成）。收到 ASK 后需要先发 ASKING 命令再重发原命令。

**Q3: Cluster 的故障检测和 Sentinel 有什么不同？**

> Sentinel 有独立的监控进程（互相投票、协商一致）。Cluster 通过 Gossip 协议在节点间直接通信——PING/PONG 心跳→PFAIL→Gossip 传播→过半 Master 确认→FAIL→Slave 竞选。Cluster 的故障检测是"节点自治"的，不需要额外的 Sentinel 进程。

**Q4: Cluster 下为什么不能跨 slot 执行事务或 Lua 脚本？**

> 事务和 Lua 脚本需要在单个 Redis 实例上原子执行。如果 key 分布在多个 Master→无法单节点完成→Redis 不实现分布式事务（不搞 2PC/XA）→直接拒绝。解决方案：Hash Tag（{ }，让需要原子操作的 key 强制同一 slot。

**Q5: slot 迁移期间，集群怎么处理请求？对这个 slot 的读写有什么影响？**

> 迁移过程中，slot 中的 key 逐个被 MIGRATE。正在被 MIGRATE 的 key 在源节点上有短暂锁（通常 < 1ms）。其他 key 正常访问。如果源节点收到对"已迁移 key"的请求→返回 ASK→Client 去新节点找。迁移期间单个 key 的不可用窗口极短（毫秒级），对大部分业务无感知。

**Q6: Cluster 部署至少需要几个节点？**

> 最小高可用部署 = 3 主 3 从 = 6 个进程 = 6 台机器（如果混合部署可以用 3 台，每台 2 个进程）。每个 Master 负责 ~5461 个 slot，每个 Master 有一个 Slave 作为故障转移的后备。3 台 Master 保证了 Gossip 心跳中过半投票的基础。

---

*Created: 2026-05-26 | Category: 21-Redis高可用-集群Cluster*
