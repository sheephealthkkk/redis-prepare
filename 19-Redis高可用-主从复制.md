# Redis 高可用 —— 主从复制深度详解

---

## 前言：从真实故障理解主从复制的必要性

假设你的电商平台上线了，所有商品详情页的缓存都在一台 Redis 上。这台 Redis 的 `maxmemory` 设为 16GB，存放了约 10GB 的热点商品数据。每天有 200 万用户访问商品详情，Redis 承担了 95% 的读流量，MySQL 只处理 5% 的穿透请求。

某天凌晨 2 点，运维在例行检查时发现 Redis 的内存使用率爬到了 99%。排查发现是上午上线的推荐系统 bug 导致大量推荐结果被缓存且没有设置 TTL。运维重启了 Redis 进程试图释放内存——结果灾难发生了：

```
Redis 重启的后果链条：

  ① Redis 进程被终止 → 10GB 数据瞬间消失
  ② 所有的商品详情请求 cache miss → 全部穿透到 MySQL
  ③ MySQL 平时只承接 5% 的流量（约 1000 QPS），现在承接 100%（约 20000 QPS）
  ④ MySQL CPU 瞬间飙到 100%，所有查询开始超时
  ⑤ 下单、支付、退款等核心业务也依赖 MySQL → 连锁反应 → 整个系统瘫了
  ⑥ Redis 重启完成 → 但缓存是空的 → 依然全部 miss → DB 依然被打
  ⑦ 需要从 MySQL 加载 10GB 数据到 Redis → 大约 5-10 分钟 → 这段时间用户看到的全是超时
```

这就是单机 Redis 最可怕的痛点：**Redis 重启或宕机后，缓存全部丢失，需要从零重建。** 重建期间，DB 承受全部流量——多数时候直接被压垮。

而主从复制解决的就是这个问题：**一台 Redis 不可用时，有一台或多台"热备份"随时可以顶上，缓存不丢，服务不断。**

---

## 第一章：主从复制能做什么？

### 1.1 三个核心价值

很多同学以为主从复制只是"多存一份数据"。实际上它的价值远不止于此：

**价值一：数据冗余——热备份**

主从复制最直接的价值是给 Master 配备一个"影子"——Slave。Master 上每一条写命令，Slave 都会收到并执行，保证 Slave 的数据和 Master 基本一致。Master 宕机时，Slave 已经有一份完整的数据副本，可以立即接管，不需要从零重建缓存。

这和 MySQL 的主从复制思路完全一样——但 Redis 的实现简单得多。MySQL 需要 binlog、redo log、undo log 三层日志才能完成复制和事务恢复。Redis 没有事务回滚，不需要那么复杂的日志体系——直接把"写命令"发给 Slave 就行。

**价值二：读写分离——分摊读压力**

大多数互联网业务是"读多写少"——商品详情页的读取量可能是库存写入量的 100 倍。如果所有请求都压在 Master 一台机器上，即使 Redis 能撑住，网络带宽也会成为瓶颈——一个 64GB 内存的 Redis Master，每秒处理 10 万次 GET 请求，每次返回 5KB，出流量就是 500MB/s——千兆网卡已经被打满。

读写分离让 Master 只处理写操作，读操作分摊到多个 Slave 上。如果部署了 3 个 Slave，每个 Slave 承担三分之一的读流量，每个 Slave 的出流量降到 167MB/s。同时 Master 的 CPU 也从处理大量 GET 中解放出来，专注于写操作和主从复制。

**价值三：为 Sentinel 和 Cluster 提供底层能力**

Sentinel（哨兵）的自动故障转移和 Cluster（集群）的分片，都依赖主从复制。Sentinel 只是在主从架构的基础上增加了"自动监测"和"自动切换"的能力。Cluster 是在多个主从对的基础上增加了"数据分片"和"跨分片路由"的能力。没有主从复制就没有高可用方案。

### 1.2 主从复制的基本拓扑

```
═══════════════════════════════════════════════════════════════
                主从复制的几种拓扑结构
═══════════════════════════════════════════════════════════════

① 一主一从（最简单的部署）：
  ┌──────────┐  复制  ┌──────────┐
  │  Master  │──────▶│  Slave   │
  │  (读写)  │       │  (只读)   │
  └──────────┘       └──────────┘
  适用：数据量小、读压力不大、只需要热备份

② 一主多从（最常见的部署）：
                  复制
  ┌──────────┐───────▶ ┌──────────┐
  │  Master  │────────▶│  Slave-1 │
  │  (读写)  │────────▶│  Slave-2 │
  └──────────┘         │  Slave-3 │
                       └──────────┘
  适用：读压力大，需要多个 Slave 分摊

③ 链式复制（级联）：
  ┌──────────┐  复制  ┌──────────┐  复制  ┌──────────┐
  │  Master  │──────▶│  Slave-1 │──────▶│  Slave-2 │
  │  (读写)  │       │ (中转+读) │       │  (只读)   │
  └──────────┘       └──────────┘       └──────────┘
  适用：Slave 太多时，减轻 Master 的复制压力
  Slave-1 同时作为"从"（对 Master）和"主"（对 Slave-2）
  注意：链路过长会导致末端的 Slave 延迟累积

═══════════════════════════════════════════════════════════════
```

---

## 第二章：主从复制的配置与首次同步

### 2.1 最简配置

主从复制的配置出奇的简单——只有一行：

```bash
# Slave 的 redis.conf 中配置
replicaof <Master的IP> <Master的端口>

# 例如：
replicaof 192.168.1.100 6379
```

这行配置告诉 Redis："启动后自动连接 192.168.1.100:6379，把自己变成它的 Slave"。Slave 会主动发起 TCP 连接，向 Master 发送 `PSYNC` 命令来启动复制流程。

在 Redis 5.0 之前，这个配置叫 `slaveof`——含义一样，只是改了个名。你现在看到的大部分文档和面试题中可能还在用 `slaveof`，但实际上 Redis 源码和配置文件中都已经改为 `replicaof`。

除了静态配置，也可以动态调整——在 redis-cli 中执行 `REPLICAOF` 命令，不需要修改配置文件也不需要重启：

```bash
# 让当前 Redis 变成某台 Master 的 Slave
127.0.0.1:6380> REPLICAOF 192.168.1.100 6379

# 停止复制，把 Slave 升级为独立的 Master（Sentinel 故障转移时就用这个）
127.0.0.1:6380> REPLICAOF NO ONE
```

### 2.2 复制状态查看

```bash
# 查看当前实例的主从状态
127.0.0.1:6379> INFO replication

# === Master 侧的典型输出 ===
# role:master
# connected_slaves:2                    ← 当前有 2 个 Slave 连接
# slave0:ip=192.168.1.101,port=6379,state=online,offset=1234567,lag=0
# slave1:ip=192.168.1.102,port=6379,state=online,offset=1234567,lag=1
# master_replid:837...c4a              ← Master 的复制 ID
# master_repl_offset:1234567           ← 当前复制偏移量

# === Slave 侧的典型输出 ===
# role:slave
# master_host:192.168.1.100
# master_port:6379
# master_link_status:up                ← up=连接正常, down=断开
# master_last_io_seconds_ago:0         ← 上次收到 Master 数据距今多少秒
# master_sync_in_progress:0            ← 是否正在全量同步（1=是）
# slave_repl_offset:1234567            ← 当前已复制的偏移量
```

### 2.3 Slave 的读取特性与读写分离落地

在 Redis 的主从架构中，默认情况下 Slave 是**只读**的——由 `replica-read-only yes` 控制。这意味着如果你尝试在 Slave 上执行 `SET`、`DEL` 等写命令，Redis 会直接返回错误 `READONLY You can't write against a read only replica`。这么做的目的是防止数据在 Slave 上被意外修改，导致主从不一致。除非你有非常特殊的业务需求（比如在 Slave 上存一些仅本机使用的临时数据），否则永远不要关闭这个配置。

读写分离在 Java 代码中如何使用？主流 Redis 客户端（Redisson、Lettuce）已经内置了对主从架构的读写分离支持：

```java
/**
 * 使用 Redisson 的 Master/Slave 模式实现读写分离
 *
 * Redisson 会自动将读命令路由到 Slave，写命令路由到 Master。
 * 当 Master 宕机后，Redisson 会检测到连接断开。
 * 当 Slave 宕机后，Redisson 会把读操作路由到其他健康的 Slave 上。
 */
@Configuration
public class MasterSlaveConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useMasterSlaveServers()
            .setMasterAddress("redis://192.168.1.100:6379")
            .addSlaveAddress(
                "redis://192.168.1.101:6379",
                "redis://192.168.1.102:6379")
            // ReadMode 控制读操作的路由策略
            .setReadMode(ReadMode.SLAVE)          // 所有读都走 Slave
            // .setReadMode(ReadMode.MASTER)      // 所有操作都走 Master
            // .setReadMode(ReadMode.MASTER_SLAVE) // 读随机分配，均衡负载
            .setMasterConnectionPoolSize(32)      // Master 连接池
            .setSlaveConnectionPoolSize(64);      // Slave 连接池（读多，池大一些）
        return Redisson.create(config);
    }
}
```

在这个配置下，当你调用 `redissonClient.getMap("product:1001").get("name")` 时，Redisson 自动将这个读命令路由到某个 Slave。调用 `redissonClient.getMap("product:1001").put("name", "iPhone")` 则自动路由到 Master。对业务代码来说，完全不需要区分"现在是读还是写"——Redisson 帮你做了这个决定。

---

## 第三章：主从复制的核心流程——深入版

### 3.1 从连接到同步的完整过程

当一个 Slave 第一次连接到 Master（或者断开太久后重连），会经历以下三个阶段。这不是简单的"复制粘贴"，而是一个精心设计的有序流程：

```
第一阶段：建立连接与身份确认
  Slave 向 Master 发起 TCP 连接 → Master 接受
  Slave 发送 PING → Master 回复 PONG（确认网络畅通）
  如果 Master 配置了密码：Slave 发送 AUTH → Master 验证
  Slave 发送 REPLCONF listening-port <port>（告诉 Master 自己的端口号）
  Slave 发送 REPLCONF capa eof capa psync2（声明自己支持的能力）

第二阶段：数据同步
  第一次连接 → 全量同步
  断线重连 → 先尝试增量同步，不行则全量

第三阶段：命令传播
  同步完成后，Master 每收到一条写命令，就会把它发给所有 Slave
  Slave 接收、执行、回复 ACK
```

### 3.2 全量同步——第一次同步的完整过程

全量同步是主从复制中开销最大的操作。每次全量同步都意味着 Master 要 fork 出一个子进程来生成 RDB 快照，然后把快照通过网络发给 Slave。这个过程会消耗 CPU、内存、磁盘 I/O 和网络带宽。

下面按时间线展开全量同步的每一步，解释每一步在做什么以及为什么这么设计：

**步骤一：Slave 发起同步请求**

Slave 启动或执行了 `REPLICAOF` 命令后，会主动向 Master 发起一个 TCP 连接。连接成功后，Slave 发送 `PSYNC ? -1` 命令。`?` 表示"我没有 Master 的复制 ID"（因为从来没有同步过），`-1` 表示"我的复制偏移量是 -1"（没有任何数据）。

**步骤二：Master 执行 BGSAVE**

Master 收到 `PSYNC ? -1` 后，判断这是第一次同步——因为 Slave 没有提供有效的 Replication ID。Master 回复 `FULLRESYNC <replid> <offset>`，告诉 Slave"我们要做全量同步，之后你的复制 ID 是 xxx，起始偏移量是 yyy"。

与此同时，Master 开始执行 BGSAVE。BGSAVE 会 fork 一个子进程，子进程拥有 Master 当前内存的一份快照（通过 COW 机制），负责生成 RDB 文件。

**步骤三：replication buffer 缓存增量命令**

在 BGSAVE 子进程写 RDB 的这几秒内，Master 的写操作不能停。如果不停，这些写操作修改的数据不会出现在 RDB 文件中（因为 RDB 是 BGSAVE 开始那一瞬间的快照）。所以 Master 在这段时间内，把所有新来的写命令都暂存在一个叫 **replication buffer** 的内存区域中。

这个 buffer 是每个 Slave 独占一份的（如果 Master 有 3 个 Slave，就有 3 个独立的 replication buffer）。它的大小不是固定的——它会根据 Slave 的处理速度动态变化。如果 Slave 处理慢，buffer 就会越来越大，直到超出 `client-output-buffer-limit` 限制——此时 Master 会断开这个 Slave 的连接。

**步骤四：Master 发送 RDB 给 Slave**

BGSAVE 完成后，Master 把生成的 RDB 文件（或直接网络流）发送给 Slave。有两种传输方式：

- **磁盘方式（`repl-diskless-sync no`，默认）**：Master 先把 RDB 写入磁盘，再从磁盘读出、发给 Slave。好处是不占用 Socket 缓冲区，坏处是磁盘 I/O 开销大。
- **无盘方式（`repl-diskless-sync yes`）**：Master 直接在内存中生成 RDB，一边生成一边通过 Socket 发给 Slave——RDB 文件不落地。好处是没有磁盘 I/O，坏处是占用 Socket 发送缓冲区。

**步骤五：Slave 加载 RDB**

Slave 收到 RDB 文件后，第一件事是**清空自己的所有旧数据**（相当于执行 FLUSHALL——但这是在 Slave 上，不影响 Master）。然后 Slave 把 RDB 文件加载到内存中。加载完成后，Slave 的数据和 Master 执行 BGSAVE 那一瞬间的数据完全一致。

**步骤六：Slave 回放 replication buffer 中的增量命令**

Master 把步骤三中缓存在 replication buffer 中的写命令全部发给 Slave。Slave 逐条执行这些命令。执行完后，Slave 的数据就和 Master 当前数据完全一致了——全量同步结束，进入命令传播阶段。

### 3.3 增量同步——断线重连后的"只补差异"

全量同步是这个过程中最昂贵的操作。每次全量同步 = Master 做一次 BGSAVE + 传输整个 RDB 文件。如果 Slave 只是短暂的网络抖动掉了线，重连后不应该再做全量同步——只发差异数据就够了。

Redis 实现增量同步，依赖三个关键的量：

**Replication ID（复制 ID）**

每个 Master 在启动时生成一个全局唯一的 40 字符的随机 ID（Replication ID）。这个 ID 就是 Master 的"身份标识"。Slave 同步后会记住这个 ID——下次重连时把它发给 Master："我上次同步的 Master 是 ID=xxx，你是我之前认识的 Master 吗？"

如果 Master 确认自己的 ID 和 Slave 提供的 ID 一致——说明 Slave 确实是从"我"这里同步过数据——那就有可能做增量同步。如果 ID 不一致——比如旧的 Master 宕机后 Slave 被提升为 Master，或者 Master 被重启了——那 ID 就变了→全量同步。

**复制偏移量（Replication Offset）**

Master 和 Slave 各自维护一个"复制偏移量"——一个从 0 开始递增的计数器。Master 每向 Slave 发送 N 个字节的复制命令，Master 的 offset 就加 N。Slave 每收到并执行 N 个字节的复制命令，Slave 的 offset 也加 N。

正常情况下，Master 的 offset 和 Slave 的 offset 应该一致（或 Slave 略微落后，因为命令在传输过程中）。如果 Slave 掉线后重连，可以发现自己的 offset 比 Master 的 offset 小——意味着 Slave 缺了一部分数据。

**repl-backlog-buffer（复制积压缓冲区）**

这是增量同步的"保险"。`repl-backlog-buffer` 是 Master 端的一个固定大小的**环形缓冲区**。Master 向 Slave 发送复制命令时，不仅发给 Slave，同时也写到这个缓冲区中。

这个缓冲区的设计思想是："Master 记住最近发送出去的数据"。当 Slave 掉线后重连，告诉 Master 自己的偏移量。Master 检查这个偏移量是否在 backlog buffer 的范围内——在的话，直接从那个偏移量处开始把缺失的部分发给 Slave（增量同步）；不在的话，只能做全量同步。

```
repl-backlog-buffer 的运作图：

  ┌─────────────────────────────────────────────────────────────┐
  │ backlog buffer（环形，大小 = repl-backlog-size）             │
  │                                                             │
  │ [off=100] [off=101] [off=102] ... [off=500] [off=501] ... │
  │     ↑                                  ↑         ↑         │
  │   已覆盖                          Slave断线前  Master最新    │
  │   (最早能提供                     的位置       offset       │
  │    的偏移量=100)                                           │
  │                                                             │
  │ Slave 断线前的 offset = 500，重连后的 offset = 500         │
  │ Master 最新 offset = 550                                    │
  │ 500 在 [100, 550] 范围内 → 增量同步！                       │
  │                                                             │
  │ 但如果 Slave 重连时偏移量 = 80（太老，已被缓冲区覆盖）       │
  │ → 只能全量同步！                                            │
  └─────────────────────────────────────────────────────────────┘
```

**`repl-backlog-size` 这个值怎么设定？**

这是生产中非常容易踩的坑。默认的 `repl-backlog-size` 是 1MB——小得离谱。在写密集的场景下，Master 每秒产生几 MB 的复制数据，1MB 的缓冲区只能覆盖几百毫秒内的数据。这意味着 Slave 只要掉线超过几百毫秒，重连后就只能做全量同步。

生产建议：
- 中等写入量（几千 QPS）：设置为 64MB
- 高写入量（几万 QPS）：设置为 256MB
- 极端写入量：512MB 或更大

一个经验公式：`repl-backlog-size = 复制数据流速(MB/s) × 预期最长断线时间(秒)`

例如 Master 每秒产生 5MB 复制数据，你预期 Slave 最多断线 60 秒——`repl-backlog-size = 5 × 60 = 300MB`。

**如果 backlog 不够大，会导致什么？**

Slave 短暂断连 → 重连 → offset 不在 backlog 范围内 → 触发全量同步。每次全量同步 = Master fork + BGSAVE + 传输整个 RDB。如果频繁发生——比如机房网络不稳定导致 Slave 频繁抖动掉线——就是大量的全量同步 → CPU、网络、磁盘全面冲击 → 整个集群被拖垮。**这就是为什么 backlog size 太小会放大集群不稳定性。**

---

## 第四章：命令传播阶段——主从之间的持续数据流

完成数据同步后，主从进入"命令传播"阶段——这是复制链路的长常态。

### 4.1 命令传播的工作原理

Master 的写命令处理器中有这样一段逻辑：每成功执行一条写命令，就检查"是否有 Slave 连着"。有的话，就把这条命令（以 RESP 协议编码）写入每个 Slave 的复制缓冲区，同时把这条命令的数据追加到 `repl-backlog-buffer`，然后更新 Master 的 offset。这一切发生在执行完命令后、返回客户端之前——对 Master 的性能几乎没有影响。

Slave 收到复制命令后，和普通命令一样执行——解析 RESP、执行命令、修改自己的内存数据——唯一的区别是 Slave 处理的"伪客户端"不需要做权限校验（因为 Master 已经校验过了）。

### 4.2 心跳维持

在命令传播阶段，主从之间是"长连接"——一条 TCP 连接维持几个甚至几个月。为了确认连接还活着，双方定期发送心跳：

**Master → Slave：PING**

Master 默认每 10 秒向 Slave 发送一次 PING（由 `repl-ping-replica-period` 控制）。这不是为了检查 Slave 是否还活着（TCP 保活机制已经能检查），而是为了**维持 TCP 连接**——防止长时间无数据传输，中间的网络设备（如防火墙、负载均衡器）把空闲连接断开。同时如果 Slave 没有在规定时间内回复 PONG，Master 就知道 Slave 出现了问题。

**Slave → Master：REPLCONF ACK**

Slave 默认每 1 秒向 Master 发送一次 `REPLCONF ACK <当前offset>`。这个心跳消息有两个作用：① 告诉 Master"我还活着，我的 offset 是 xxx"；② 让 Master 知道 Slave 的复制进度。

Master 通过 ACK 中的 offset 判断 Slave 的延迟。如果某个 Slave 的 offset 和 Master 差距很大——说明 Slave 的处理速度跟不上 Master 的写入速度。如果连续检测到 Slave 延迟过大，可能需要排查 Slave 机器是否有性能问题。

### 4.3 复制偏移量的三个作用

复制偏移量不只是"数据一致性的度量"，它在 Redis 中有三个关键的用途：

**用途一：增量同步判断** —— Slave 重连时发送自己的 offset，Master 判断是否在 backlog 内。

**用途二：Sentinel 选主排序** —— Sentinel 故障转移时，多个 Slave 抢着当新 Master。谁的 offset 大（数据越新），谁当选的概率就越高。这保证了"数据最完整的 Slave 成为新 Master"。

**用途三：`min-replicas-to-write` 安全保障** —— Master 写数据时，检查"有多少 Slave 的 offset 和我的 offset 差距在可接受范围内"。达不到最少 Slave 数——Master 拒绝写入——用"牺牲可用性"换"数据安全性"。

---

## 第五章：主从延迟——读写分离的"隐形杀手"

### 5.1 主从延迟是怎么产生的？

即使是最理想的环境下（同机房、万兆网络、Slave 机器性能足够），主从延迟也是客观存在的。Master 写入数据到 Slave 收到并执行，中间的延迟包括：

- **Master 处理时间**：写命令执行完成到写入复制缓冲区的耗时（微秒级，可忽略）
- **网络传输时间**：从 Master 发送数据到 Slave 接收数据（同机房 0.1-0.5ms，跨机房 1-5ms）
- **Slave 处理时间**：Slave 收到命令、解析、执行的耗时（微秒级，和 Master 执行命令差不多）

正常情况下的主从延迟大约 0.5-2ms。但在以下场景中，延迟可能飙升至秒级甚至几十秒：

- **Master 瞬时大量写入**：比如 1 秒内 5 万次 SET → Master 要发 5 万条命令给 Slave → Slave 处理不过来 → 积压
- **Slave 处理能力不足**：Slave 的 CPU/内存不如 Master → 执行跟不上发送速度
- **网络带宽瓶颈**：复制流量占满网卡 → 新命令发不过去 → 延迟飙升
- **BigKey 写入**：Master 执行 `SET bigkey 10MB数据` → Master 正常写入 → Slave 执行时 10MB 的 SDS 分配更慢（因为 Slave 机器配置可能更低）

### 5.2 主从延迟会引发的具体问题

**问题一：刚写入的数据读不到**

这是读写分离最常见的投诉。用户提交了一个订单——`SET order:12345 ...` 写入 Master → 页面自动跳转到订单详情页 → 代码从 Slave 读取 `GET order:12345` → Slave 还没收到这个 key → 返回 nil → 页面显示"订单不存在"。用户刷新一下，请求路由到另一个 Slave（这个 Slave 已经收到了）→ 订单出现了。用户：你们的系统不稳定吧？

**问题二：分布式锁在主从切换时丢失**

这是 Redis 分布式锁最大的"坑"。Client-A 在 Master 上拿到了锁 `SET lock:order NX PX 30000` → Master 还没来得及把这条命令同步给 Slave → Master 宕机 → Sentinel 提升 Slave 为新 Master → Client-B 在新 Master 上拿锁 `SET lock:order NX PX 30000` → 锁根本不存在（因为旧 Master 的锁没同步过来！）→ Client-B 也拿到了锁 → A 和 B 同时执行业务 → 并发安全崩了。这就是为什么 RedLock 和 ZooKeeper 存在的原因——普通 Redis 主从复制的锁不是绝对安全的。

### 5.3 业务层面如何应对主从延迟

对于问题一（刚写读不到），业务代码可以标记"哪些请求刚执行了写操作"，对于这些请求，强制从 Master 读取，避免延迟问题：

```java
/**
 * 读写分离 + 写后一致性策略
 * 
 * 设计思路：
 *   1. 维护一个 ThreadLocal 标记"当前请求是否刚刚执行了写操作"
 *   2. 写操作执行后设置标记为 true
 *   3. 读操作读之前检查标记 → true 则走 Master → false 则走 Slave（正常分流）
 *   4. 请求处理完成后清除标记
 * 
 * 这种策略覆盖了绝大部分"写入 → 马上查询"的业务场景：
 *   下订单 → 查看订单详情
 *   修改个人信息 → 刷新个人主页
 *   发布内容 → 查看发布的内容
 */
@Service
public class ReadAfterWriteService {

    private final ThreadLocal<Boolean> wroteInCurrentRequest = 
        ThreadLocal.withInitial(() -> false);

    /**
     * 写操作——标记"刚刚写过"
     */
    public void saveProduct(String productId, Map<String, String> data) {
        // 写 Master（正常）
        redissonClient.getMap("product:" + productId).putAll(data);
        // 标记：当前请求刚写过数据
        wroteInCurrentRequest.set(true);
    }

    /**
     * 读操作——如果刚写过，强制读 Master
     */
    public Map<String, String> getProduct(String productId) {
        RMap<String, String> product;

        if (wroteInCurrentRequest.get()) {
            // 当前请求刚写过数据 → 强制从 Master 读
            // 避免主从延迟导致"读不到刚写的数据"
            product = redissonClient.getMap("product:" + productId);
            // 读取完成后重置标记
            wroteInCurrentRequest.remove();
        } else {
            // 正常请求 → 走 Slave（由 Redisson 自动路由）
            product = redissonClient.getMap("product:" + productId);
        }
        return product.readAllMap();
    }
}
```

### 5.4 Master 端的数据安全保障

```bash
# redis.conf（Master 端）——用"宁可不可用，不能丢数据"的策略
# 
# 当以下两个条件同时满足时，Master 才能执行写操作：
#   ① 可正常连接的 Slave 数量 ≥ 1
#   ② 所有 Slave 的复制延迟 ≤ 10 秒
# 
# 如果条件不满足 → Master 拒绝写 → 客户端收到报错 → 触发告警
# 
# 为什么这么设计？
#   与其让 Master 在"没有 Slave 兜底"的情况下继续写入数据，
#   不如直接拒绝写入 → 触发业务告警 → 人工介入
#   防止"Master 写入了一堆数据，但所有 Slave 都跟丢了"
#   最后 Master 宕机 → 这些数据永久丢失

min-replicas-to-write 1           # 至少 1 个 Slave 连接正常
min-replicas-max-lag 10           # 所有 Slave 的延迟 ≤ 10 秒
```

---

## 第六章：replication buffer 和 repl-backlog-buffer —— 不要搞混

这是面试中容易被混淆的两个概念。它们的目标完全不同：

| 维度 | replication buffer | repl-backlog-buffer |
|------|-------------------|-------------------|
| **归属** | 每个 Slave 独占一份 | Master 全局只有一份 |
| **生命周期** | Slave 连接期间存在 | Master 启动后就存在 |
| **内容** | 从全量同步开始到当前的所有增量 | 最近发出的复制数据 |
| **作用** | 全量同步期间缓存增量命令 | 增量同步时提供差异数据 |
| **大小** | 动态变化（无固定大小，受 client-output-buffer-limit 限制） | 固定大小（repl-backlog-size） |
| **满了** | 超过限制 → Master 断开 Slave | 环形覆盖老数据 |

**为什么需要两个不同的缓冲区？**

这个问题会暴露你对主从复制的理解深度。replication buffer 是为"全量同步期间"设计的——在 RDB 生成的几秒内暂存增量命令，发给 Slave 后就被清空。repl-backlog-buffer 是为"日常命令传播"设计的——持续记录 Master 发出的数据，以备未来 Slave 断线重连时做增量同步。

---

## ⭐️ 面试题汇总

**Q1: Redis 主从复制的完整流程是怎样的？分几个阶段？**

> 三个阶段。① 建立连接（Slave REPLICAOF→Master 接受，PING/PONG 确认网络通畅，AUTH 认证）。② 数据同步（首次连接→全量同步=Master BGSAVE RDB 发给 Slave→Slave 清空旧数据+加载 RDB+回放 replication buffer 中的增量命令；断线重连→先尝试增量同步=PSYNC+offset 判断，在 backlog 内就发差异，不在就全量同步）。③ 命令传播（Master 每条写命令→发给所有 Slave→Slave 执行→回 ACK offset）。核心：两个缓冲区（replication buffer 为全量同步存增量，backlog buffer 为增量同步存历史）。

**Q2: 全量同步和增量同步的区别？增量同步的条件是什么？**

> 全量同步需要 Master fork BGSAVE 生成 RDB→发送给 Slave→Slave 清空所有旧数据→加载 RDB，开销大。增量同步只需要把 Slave 缺失的部分数据发送过去，开销小。增量同步条件：① Slave 的 Replication ID == Master 自己的 ID；② Slave 的复制偏移量仍在 Master 的 repl-backlog-buffer 所覆盖的范围内。缺一个条件→全量同步。

**Q3: repl-backlog-buffer 设得太小会导致什么问题？**

> Slave 短暂断连后重连→缺失的数据已被 backlog 覆盖→无法增量同步→触发全量同步。每次全量同步=Master fork+BGSAVE+传输整个 RDB→CPU/网络/磁盘冲击。如果频繁发生（网络不稳定导致 Slave 频繁断连）→连续的全量同步几乎能把集群拖垮。生产建议至少 64-256MB。

**Q4: 主从延迟会导致什么问题？业务怎么应对？**

> ① 读写分离下"刚写的数据读不到"——用户下订单后跳到订单详情页，页面显示"订单不存在"。解决：用 ThreadLocal 标记"当前请求刚写过数据"，后续读操作强制走 Master。② 分布式锁在主从切换时丢失——Master 的锁还没来得及同步给 Slave→Master 宕机→Slave 提升后没有锁信息→锁丢失。解决：RedLock/ZooKeeper/etcd。③ 配置 `min-replicas-to-write`+`min-replicas-max-lag` 保障数据安全。

---

*Created: 2026-05-26 | Category: 19-Redis高可用-主从复制*
