# Redis 高可用 —— 哨兵（Sentinel）深度详解

---

## 前言：主从复制缺少了什么？

主从复制解决了"数据冗余"和"读写分离"，但当你把主从架构部署到生产环境后，很快就会发现一个尴尬的问题：

Master 宕机了，谁来处理？

```
凌晨 3 点，你的 Redis Master 进程因为内存碎片触发了 OOM，被 Linux 的 OOM Killer 杀掉了。

此时：
  两个 Slave 还在运行——但它们是只读的，不能接收写操作
  所有依赖 Redis 写入的业务（下单、扣库存、记录积分）全部报错
  Slave 只是被动接收 Master 的数据，没有任何"自救"能力

你被报警电话叫醒，登进服务器：
  → 找到一个 Slave（数据最新、offset 最大）
  → 手动执行 REPLICAOF NO ONE，把它提升为 Master
  → 另一个 Slave 执行 REPLICAOF 新 Master
  → 修改所有应用配置，把 Redis 地址指向新 Master
  → 重启应用
  → 恢复服务：整个过程用了 5-10 分钟

在这 5-10 分钟内，你的系统对用户来说是完全不可用的。
```

Redis Sentinel 就是为这个场景设计的——**当 Master 宕机时，自动完成上面的所有手动操作，把恢复时间从"分钟级"降到"秒级"。**

---

## 第一章：Sentinel 是什么？

### 1.1 Sentinel 的本质

Sentinel 不是监控面板、不是高可用方案的一个"组件"——它是一个**运行在特殊模式下的 Redis 进程**。和普通 Redis 进程的区别是：Sentinel 不存储任何业务数据，它只运行一套监控、判断和通知逻辑。

你启动一个 Sentinel 的方式是：

```bash
redis-sentinel /path/to/sentinel.conf
# 或者
redis-server /path/to/sentinel.conf --sentinel
```

Sentinel 占用极少的内存和 CPU——通常一个 Sentinel 进程只用几 MB 内存。所以业内惯例是**把 Sentinel 和 Redis 部署在同一台机器上**——不额外浪费服务器资源。

### 1.2 为什么必须有多个 Sentinel？

一个 Sentinel 是不够的——因为 Sentinel 和任何进程一样，可能会宕机。如果只有 1 个 Sentinel 进程在监控，它挂了之后整个监控就停了——Master 宕机后再也没人切换。

但这不是唯一原因。即使 Sentinel 本身没挂，单 Sentinel 也无法克服**网络抖动**——Sentinel 和 Master 之间的网络可能出问题（而 Master 本身是好的），这时单 Sentinel 会认为"Master 宕机"，然后发起故障转移——但实际上 Master 根本没坏，这会制造一个分裂的集群。

所以 Sentinel 必须是**多个**——至少 3 个。多个 Sentinel 通过**投票**决定 Master 是否真的宕机——"单个人看走眼有可能，三个人的共识可信度就高得多"。

### 1.3 Sentinel 集群的工作方式

多个 Sentinel 不是由某个 Leader Sentinel 来指挥的——它们是去中心化的、平等的。每个 Sentinel 独立完成监控（向 Master、Slave、其他 Sentinel PING），但决策需要投票达成共识。Sentinel 之间通过 Master 的 `__sentinel__:hello` Pub/Sub 频道来互相发现——一个新 Sentinel 启动，往这个频道发一条消息，其他 Sentinel 就自动知道它的存在。

```
sentinel 之间互相发现的示意图：

  Sentinel-1 ──────────────────────────┐
       │ 订阅 __sentinel__:hello       │
       │                               │
       ▼                               ▼
  ┌────────────────────────────────────────┐
  │         Master 的 Pub/Sub 频道         │
  │         __sentinel__:hello             │
  └──┬──────────┬─────────────┬───────────┘
     │ 发布消息  │ 发布消息     │ 发布消息
     ▼           ▼             ▼
  Sentinel-2  Sentinel-3  (新来的Sentinel-4)
     │           │             │
     └───────────┴─────────────┘
       所有 Sentinel 都知道了对方的存在
```

---

## 第二章：监控——Sentinel 怎么发现 Master 挂了

### 2.1 三条心跳线

Sentinel 同时监控三类实体——Master、Slave、其他 Sentinel。每类实体有不同的监控频率和方式：

**心跳线一：Sentinel → Master（核心心跳）**

每个 Sentinel 每 1 秒（由 `sentinel-ping-interval` 控制，不能改）向 Master 发送 PING。如果 Master 在 `down-after-milliseconds` 时间内（默认 30 秒）没有回复 PONG——既不是"回复慢"、也不是"回复错误"，而是一点回应都没有——Sentinel 认为 Master 不可达。但这还只是 SDOWN（主观下线）——只代表"我联系不上"，不代表"大家都联系不上"。

**心跳线二：Sentinel → Sentinel（共识信息）**

每个 Sentinel 每 2 秒向 Master 的 `__sentinel__:hello` 频道发布一条消息，内容是对 Master 和所有 Slave 的监控信息。其他 Sentinel 订阅了这个频道，每收到一条消息就更新自己对集群状态的认知。这条心跳线的作用是：让所有 Sentinel 对集群状态形成一致的视图。

**心跳线三：Sentinel → Slave / 其他 Sentinel（INFO 采集）**

每个 Sentinel 每 10 秒向所有 Slave 和所有其他 Sentinel 发送 INFO 命令，收集它们的运行状态、复制偏移量、role 等信息。这些信息在故障转移时用来选择"谁当新 Master 最合适"。

### 2.2 SDOWN 和 ODOWN——主观和客观的区分

这两个概念是 Sentinel 最核心的设计。它们的区分解决了一个根本问题：**怎样确定一个节点是真的宕机了，还是只是"和我的网络断了"？**

**SDOWN（Subjectively Down，主观下线）**

一个 Sentinel 单独判断的结果。判断标准很简单：PING 超时 > `down-after-milliseconds`。但 SDOWN 只是一个 Sentinel 的"个人意见"——可能只是它自己的网络问题。

**ODOWN（Objectively Down，客观下线）**

多个 Sentinel 的"集体意见"。当一个 Sentinel 将 Master 标记为 SDOWN 后，它会向其他 Sentinel 发送 `SENTINEL is-master-down-by-addr` 命令，询问"你们也觉得 Master 挂了吗？"。如果 ≥ quorum 个 Sentinel 都认为 Master 不可达——Sentinel 将 Master 标记为 ODOWN。ODOWN 意味着"这不是我个人问题，是大家共识"——此时可以发起故障转移。

```
═══════════════════════════════════════════════════════════════
         SDOWN → ODOWN 的判定时间线
═══════════════════════════════════════════════════════════════

  t=0s    Master 正常运行
  t=1s    Master 宕机
  t=2s    Sent-1: PING 无响应 → 开始倒计时
  t=2s    Sent-2: PING 无响应 → 开始倒计时
  t=2s    Sent-3: PING 无响应 → 开始倒计时
  
  t=31s   Sent-1: 累积超时 = 30s = down-after-milliseconds
          → 标记 Master 为 SDOWN
          → 立刻向 Sent-2 和 Sent-3 发送查询：
            "你们也觉得 Master 挂了吗？"
            
  t=31.1s Sent-2 回复："是的，Master 无响应"（SDOWN）
          Sent-3 回复："是的，Master 无响应"（SDOWN）
          
  t=31.2s Sent-1: 统计结果 ≥ quorum(2) 个确认
          → 标记 Master 为 ODOWN
          → 开始选举执行故障转移的 Sentinel Leader

═══════════════════════════════════════════════════════════════
```

### 2.3 quorum 的设置与含义

`quorum` 的值在 `sentinel monitor` 中配置。例如：

```bash
sentinel monitor mymaster 192.168.1.100 6379 2
#                                               ↑
#                                          quorum = 2
# 至少需要 2 个 Sentinel 同意 Master 挂了，才能标记为 ODOWN
```

quorum 的值和 Sentinel 的总数决定了集群的容错能力：

- **3 个 Sentinel，quorum=2**：可以容忍 1 个 Sentinel 宕机，2 个正常即能判定 ODOWN、发起故障转移
- **5 个 Sentinel，quorum=3**：可以容忍 2 个 Sentinel 宕机

quorum 不能设为 1——因为 1 个 Sentinel 的 SDOWN 就触发故障转移 = 单 Sentinel 的不足。也不能设得 > N/2+1——太大了会导致很难达成共识。

---

## 第三章：故障转移——谁来切换？怎么切？

### 3.1 Sentinel Leader 选举

当一个 Sentinel 把 Master 标记为 ODOWN 后，并不是随便谁就能发起故障转移。多个 Sentinel 可能同时检测到 ODOWN——如果它们各自为政、各自提升不同的 Slave 为新 Master，整个集群就分裂了。

所以 Sentinel 需要先选出一个"Leader"——只有 Leader 才有资格决定提升哪个 Slave，其他 Sentinel 配合它的决定。

Leader 选举的规则类似于 Raft 的选举——先到先得：

- 发现 Master ODOWN 的 Sentinel 向其他 Sentinel 发送投票请求（`SENTINEL is-master-down-by-addr`，附带自己的 runid 作为"候选人 ID"）
- 收到请求的 Sentinel 如果还没有投过票，就投票给这个候选人，并记录下自己已经投过（每个"纪元 epoch"只能投一票）
- 候选人如果获得 ≥ 半数（N/2+1）的票数 → 当选为 Leader → 负责执行故障转移
- 如果这次选举没产生 Leader（多个候选人同时竞选，没人过半），等待一段时间后进入新的 epoch 重新选举

### 3.2 选出新 Master——多维度排序

Leader 选出后，接下来是最关键的一步——从所有在线 Slave 中选出新 Master。不是随机选，而是按照严格的多维度排序：

**步骤一：排除"不健康的"Slave**

先过滤掉明显不能用的 Slave。以下三种 Slave 直接出局：
- 状态为 `SRI_DISCONNECTED` 的 Slave——断连了，根本没在接收 Master 数据
- 最近 5 秒内没有回复过 INFO 命令的 Slave——网络不好，不可靠
- 与 Master 断开超过 `down-after-milliseconds × 10` 的 Slave——断连太久，数据可能差太远

**步骤二：多维度排序**

剩余的 Slave 按以下优先级排序：

第一优先：`replica-priority` 更低的 Slave。这是一个手动配置的权重——数字越小、优先级越高。你可以把一台高性能的 Slave 设为 `replica-priority 10`（优先当选），一台低配机器设为 `replica-priority 100`（最后考虑）。设为 0 的 Slave 永远不会被选为 Master。

第二优先：复制偏移量（offset）更大的 Slave。offset 越大 = 从 Master 接收的数据越多 = 数据越新。这就保证了"数据最完整的 Slave 被提升为 Master"——最大化数据安全性。

第三优先：runid 字典序更小的 Slave。如果多个 Slave 的前两项都相同，就按 runid 的字典序排序——只是一个稳定的 tiebreaker。

### 3.3 切换步骤——提升、通知、退位

把"手动切换"的所有步骤自动化：

```
步骤 1：SLAVEOF NO ONE
  选定的 Slave 收到 Leader 的命令 → 执行 REPLICAOF NO ONE
  → 该 Slave 中断与旧 Master 的复制连接
  → 角色从 Slave 转变为 Master
  → 开始接受客户端的读写请求

步骤 2：其他 Slave 重新定向
  Leader 向所有其他 Slave 发送命令：REPLICAOF 新Master的IP 新Master的Port
  → 它们断开和旧 Master 的连接（如果还能连上的话）
  → 开始从新 Master 同步数据

步骤 3：旧 Master 恢复后降级
  如果旧 Master 重启恢复了 → Sentinel 持续监控它
  → 旧 Master 上线后 → Leader 发送 REPLICAOF 命令
  → 旧 Master 自动降级为新 Master 的 Slave
  → 保证集群中只有一个 Master！

步骤 4：通知客户端
  故障转移完成后，Sentinel 通过 Pub/Sub 频道 +switch-master 广播新 Master 地址
  
  客户端库（如 Redisson、Lettuce）内部订阅了这个频道
  → 收到消息后自动重建到新 Master 的连接池
  → 业务代码完全无感知！
```

---

## 第四章：Sentinel 的部署与业务集成

### 4.1 生产部署架构——为什么要 3 台服务器

```
═══════════════════════════════════════════════════════════════
                推荐的 Sentinel 生产部署架构
═══════════════════════════════════════════════════════════════

        服务器 1                    服务器 2                    服务器 3
  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
  │  Master (6379)  │    │  Slave  (6379)  │    │  Slave  (6379)  │
  │  Sentinel(26379)│    │  Sentinel(26379)│    │  Sentinel(26379)│
  └─────────────────┘    └─────────────────┘    └─────────────────┘
          ↑                       ↑                       ↑
          │                       │                       │
          └───────────────────────┴───────────────────────┘
                        3 台服务器，3 个 Redis + 3 个 Sentinel

为什么每台机器上都部署一个 Sentinel？
  - Sentinel 不占内存（几 MB）→ 和 Redis 共存完全没问题
  - 如果 Sentinel 单独部署 → 多 3 台机器 → 浪费成本
  - Sentinel 和 Redis 在同一台 → 监控延迟最低（同机通信）

为什么至少 3 个 Sentinel？
  - 2 个 Sentinel → 1 台机器挂了 → quorum=1 无法达成 → 故障转移失败
  - 3 个 Sentinel → 1 台机器挂了 → 还剩 2 个 → quorum=2 → 仍然能切换

═══════════════════════════════════════════════════════════════
```

### 4.2 sentinel.conf 详解

```bash
# ===== sentinel.conf 的每一个关键配置 =====

# ① 监控配置（必须配）
# sentinel monitor <Master名> <Master IP> <Master Port> <quorum>
# quorum = 最少需要几个 Sentinel 同意 Master ODOWN
sentinel monitor mymaster 192.168.1.100 6379 2

# ② 判定下线时间（毫秒）
# Sentinel PING Master → Master 没有回复 → 等待这个时间后标记为 SDOWN
# 设太短 → 网络轻微抖动就误判
# 设太长 → Master 真的宕机了也要等半天 → 故障转移太慢
# 生产建议：15-30 秒
sentinel down-after-milliseconds mymaster 30000

# ③ 故障转移超时（毫秒）
# 从开始故障转移到新 Master 就位的最大允许时间
# 超过这个时间还没完成 → 重新发起 Leader 选举 → 新一轮切换
# 生产建议：180 秒（3 分钟），够大但不会无限等
sentinel failover-timeout mymaster 180000

# ④ 同时向新 Master 同步的 Slave 数量
# 设为 1 → 一个 Slave 同步完再下一个 → 慢但 Master RDB 传输压力小
# 设为 N → N 个 Slave 并行同步 → 快但 Master 的网络带宽被占满
sentinel parallel-syncs mymaster 1

# ⑤ Sentinel 进程端口（默认 26379）
port 26379
```

### 4.3 Java 客户端集成——Redisson

```java
/**
 * 使用 Redisson 的 Sentinel 模式
 * 业务代码不需要管理"当前 Master 是谁"——
 * Redisson 自动通过 Sentinel 发现 Master，Master 切换后自动重连
 */
@Configuration
public class SentinelRedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSentinelServers()
            // Master 名称——和 sentinel.conf 中配置的一致
            .setMasterName("mymaster")
            // 所有 Sentinel 的地址（Redisson 会从中选一个连接）
            .addSentinelAddress(
                "redis://192.168.1.100:26379",
                "redis://192.168.1.101:26379",
                "redis://192.168.1.102:26379")
            // 读取模式
            .setReadMode(ReadMode.SLAVE)           // 读走 Slave
            .setMasterConnectionPoolSize(32)       // Master 连接池
            .setSlaveConnectionPoolSize(64);       // Slave 连接池
        return Redisson.create(config);
    }
}
```

### 4.4 业务场景——支付系统的数据安全保障

支付系统对 Redis 的数据安全性要求非常高——丢一条扣款记录就是生产事故。下面是支付系统的 Sentinel 配置策略：

```bash
# ===== Master 的 redis.conf =====
# Master 必须开启持久化（否则重启后发给 Slave 的是空 RDB）
save 900 1
save 300 10
save 60 10000
# 数据安全性配置：至少 1 个 Slave 确认，且延迟 ≤ 10 秒
min-replicas-to-write 1
min-replicas-max-lag 10

# ===== sentinel.conf =====
sentinel monitor mymaster 192.168.1.100 6379 2   # 至少 2 个同意
sentinel down-after-milliseconds mymaster 15000   # 15 秒快速判定
sentinel failover-timeout mymaster 120000         # 2 分钟超时
sentinel parallel-syncs mymaster 1               # 逐台同步
```

这套配置的逻辑是：
- `min-replicas-to-write 1` → Master 写入时至少 1 个 Slave 确认 → 写入的数据不只在 Master 上
- `min-replicas-max-lag 10` → Slave 延迟 ≤ 10 秒 → 确认的数据是"新"的，不是"十分钟前就确认过但 Slave 现在离线了"
- Sentinel 的 `down-after-milliseconds` 设 15 秒（而非默认 30 秒）→ 更快检测到 Master 宕机 → 更快切换

---

## 第五章：脑裂——Sentinel 最头疼的问题

### 5.1 什么是脑裂？

脑裂（Split-Brain）是分布式系统中最经典的故障模式。在 Redis Sentinel 场景下，脑裂的场景是：Master 没有宕机，但 Master 和 Sentinel 集群之间的网络断了。

```
脑裂发生时：

  ┌─────────────┐         ┌────────────────────────┐
  │   Master    │   ╳     │  3 个 Sentinel +       │
  │  (还在运行)  │  网络断  │  2 个 Slave             │
  └──────┬──────┘         └───────────┬────────────┘
         │                           │
    收到客户端写请求            Sentinel 认为 Master 宕了
    继续处理写入！              → 开始故障转移
         │                    → 选举新 Master
         ▼                           │
  Master 又在                                    ▼
  自己写了                                     新 Master
  一堆数据                                    上位了！
         │                           │
         └───────────┬───────────────┘
                     │
              两个"Master"同时存在！
              数据分叉——两份数据不一样了
```

### 5.2 缓解方案

```bash
# Master 的 redis.conf 中配置

# 当 Master 发现自己没有 Slave 连接时（Slave 都去了新 Master 那边），
# 拒绝执行写操作！这样脑裂期间原 Master 不会写入新数据。
min-replicas-to-write 1
min-replicas-max-lag 10
```

在这个配置下，脑裂产生的副作用可以降到最低：原 Master 因为"没有 Slave 连接"→ 自动拒绝写操作。客户端收到写入失败的错误 → 通过 Sentinel 发现新 Master → 重新发送写操作到新 Master → 数据正确落盘。虽然脑裂瞬间可能产生短暂的数据不一致，但新 Master 上的数据是完整的（因为它拥有和 Sentinel 集群的连接和多数 Slave 的确认）。

---

## ⭐️ 面试题汇总

**Q1: Sentinel 的 SDOWN 和 ODOWN 有什么区别？为什么需要两种判定？**

> SDOWN = 一个 Sentinel 自己的判断（PING 超时→我认为它挂了），ODOWN = 多个 Sentinel 的共识（≥ quorum 个认为它挂了→确认挂了）。区分两者的原因：防止单个 Sentinel 因自身网络问题误判——如果 SDOWN 就触发故障转移→网络抖动导致频繁无效切换→集群不稳定。

**Q2: Sentinel 如何选出新 Master？排序规则是什么？**

> 排除不健康的 Slave→剩余 Slave 按多维排序：① `replica-priority` 低的优先（手动权重）；② offset 大的优先（数据越新）；③ runid 字典序小的优先（稳定 tiebreaker）。保证选出的新 Master 是"所有 Slave 中数据最完整的一台"。

**Q3: Sentinel 为什么至少需要 3 个节点？2 个不行吗？**

> 2 个 Sentinel→其中一个宕机或网络故障→只剩 1 个→quorum 无法达成→故障转移失败。3 个 Sentinel 可以容忍 1 个故障（quorum=2，还有 2 个在投票）。Sentinel 进程本身不占资源（几 MB 内存），多一个 Sentinel 几乎零成本。

**Q4: 脑裂是什么？Sentinel 怎么缓解脑裂的影响？**

> 脑裂 = Master 和 Sentinel 网络断开，Sentinel 选举了新 Master，但原 Master 还在运行→两个 Master 同时存在。缓解：`min-replicas-to-write 1`→原 Master 发现没有 Slave 连接了→拒绝写入→迫使客户端通过 Sentinel 发现新 Master→减少了脑裂期间的写入分叉。

**Q5: 故障转移期间客户端怎么办？能自动感知新 Master 吗？**

> 客户端不直接连 Master，而是通过 Sentinel 发现。Sentinel 完成切换后通过 `+switch-master` 频道广播新 Master 地址。Redisson/Lettuce 等客户端已经内置了这个逻辑→订阅频道→收到通知→自动重建到新 Master 的连接池→业务代码完全不感知。

---

*Created: 2026-05-26 | Category: 20-Redis高可用-哨兵Sentinel*
