# Redis 常见阻塞原因 —— 生产排查全指南

---

## 前言：Redis 是单线程的，阻塞 = 一切都停了

在阅读本文前，需要先理解一个核心事实：**Redis 的主线程一次只能干一件事。** 

当你执行 `GET key` 时，主线程从 socket 读请求 → 解析命令 → 在 dict 中查找 key → 序列化返回→写回 socket。整个过程不到 1 微秒。上百个客户端同时发 `GET` ——主线程飞快地在它们之间切换——每个客户端只等几微秒——用户完全感知不到延迟。

但如果某一条命令需要执行 200 毫秒呢？比如你误执行了 `KEYS *`——200 毫秒内主线程被这条命令独占——其他几百个客户端的请求全部排队等待——几百个客户端同时感受到延迟——从他们的视角看，Redis "卡住了 200 毫秒"。

这就是 Redis 阻塞的根源。本文详细讲解 6 大类阻塞场景——从原理到排查到解决方案。

---

## 第一章：O(N) 命令——最直接也最常见的阻塞源

### 1.1 哪些命令是 O(N) 的？

Redis 大部分命令都是 O(1) 的：GET、SET、HGET、HSET、SADD、SISMEMBER、LPUSH、ZSCORE 等。这些命令无论数据多大，执行时间都差不多。

但有一批命令的时间复杂度随数据量线性增长——数据越多，执行越慢：

```
致命级（数据量大时绝对不能在生产中使用的）：
  KEYS *                     O(N) 遍历所有 key
  FLUSHALL / FLUSHDB         O(N) 清空所有 key
  SAVE                       O(N) 主线程做 RDB 快照

危险级（数据量大时严重阻塞）：
  SMEMBERS key               O(N) 返回整个 Set
  HGETALL key                O(N) 返回整个 Hash
  LRANGE key 0 -1            O(N) 返回整个 List
  SUNION/SINTER/SDIFF        O(N*M) 大集合运算
  ZRANGE key 0 -1            O(logN+M) 返回整个 ZSet
  SORT key                   O(N*logN) 排序

渐进式危险（offset 大时阻塞）：
  LINDEX key 10000000        O(N) 走到第 1000 万个元素
  ZRANGE key 100000 100010   O(logN+offset) offset 越大越慢
```

### 1.2 KEYS * 为什么是"生产禁令"？

KEYS 命令执行时，Redis 遍历整个全局键空间（全局 dict），检查每个 key 是否匹配给定的通配符模式。如果 Redis 中有 5000 万个 key，遍历一遍可能需要几百毫秒甚至几秒。这几秒期间，主线程被卡死，所有其他操作暂停。

生产环境中 `KEYS *` 的替代方案是 **SCAN 系列命令**：

```bash
# ❌ 阻塞：KEYS *
KEYS user:*

# ✅ 非阻塞：SCAN（每次返回少量 key）
SCAN 0 MATCH user:* COUNT 1000
# SCAN 返回一个游标 → 下次从游标处继续 → 重复直到游标为 0
# SCAN 不阻塞！每次只占用主线程微秒级的时间
```

SCAN 的注意事项：SCAN 不是原子的——遍历期间可能有 key 被创建或删除——你可能重复看到某个 key 或漏掉某个 key。但对于查找特定模式 key、分析 key 分布等场景，SCAN 已经足够。

### 1.3 替代方案速查表

| 危险命令 | 替代方案 |
|---------|---------|
| KEYS * | SCAN 分批遍历 |
| SMEMBERS | SSCAN 分批 |
| HGETALL | HSCAN 分批 |
| LRANGE 0 -1 | LRANGE 0 99 分页 |
| ZRANGE 0 -1 | ZSCAN 或 ZRANGE 0 99 分页 |
| FLUSHALL | FLUSHALL ASYNC（Redis 4.0+ 异步清空） |
| DEL bigkey | UNLINK（异步删除） |

---

## 第二章：SAVE 创建 RDB 导致的阻塞

### 2.1 SAVE vs BGSAVE

```bash
# SAVE：主线程执行！完全阻塞！
SAVE
# Redis 停止处理请求，把所有内存数据写入 dump.rdb
# 10GB 数据可能需要几秒到十几秒

# BGSAVE：fork 子进程执行！仅 fork 瞬间阻塞（~100ms）
BGSAVE
# 主线程调用 fork() → 子进程写 dump.rdb → 主线程继续处理请求
# fork 期间才阻塞，通常 < 200ms
```

`SAVE` 命令让 Redis 主线程直接执行数据持久化操作——它会遍历所有数据写入 RDB 文件。在此期间，Redis 不会处理任何客户端请求。对于任何规模的生产环境，这都是绝对不能容忍的。而 `BGSAVE` 把实际写盘工作交给了 fork 出的子进程——主线程只在 fork 的瞬间被阻塞（拷贝页表）。

**排查方法：** 查看 `INFO stats` 中的 `latest_fork_usec`——如果很大（>1秒），说明 fork 期间阻塞严重，可能是内存太大或 THP 开启。另外查看 Slowlog 中是否有 SAVE 命令出现——有的话立即排查是谁在执行 SAVE。

---

## 第三章：AOF 相关的三种阻塞

### 3.1 AOF 日志记录阻塞

AOF 日志记录发生在主线程执行写命令之后。主线程把写命令追加到 `aof_buf`（一个内存缓冲区）后——如果配置了 `appendfsync always`——主线程需要等待 `fsync` 完成才能处理下一条命令。因为 `always` 模式下每条写命令都要等待磁盘 I/O——主线程被磁盘 I/O 卡住。

这就是为什么官方强烈不建议生产环境使用 `always`。`everysec` 模式下，fsync 由后台线程 BIO-2 执行——主线程只负责写到 aof_buf——不直接接触磁盘 I/O。

### 3.2 AOF 刷盘阻塞（everysec 下的"阻塞"）

`everysec` 模式下，后台线程 BIO-2 每秒执行一次 `fsync`。但在一种极端情况下，主线程也会被 AOF 刷盘阻塞：

```
主线程和 BIO-2 线程竞争同一把文件锁：

  ① 主线程：客户端写了一大堆命令 → 大量数据写入 aof_buf
  ② 主线程：把 aof_buf 刷到 OS buffer（write() 调用）
  ③ BIO-2：定时到了 → 执行 fsync（把 OS buffer 刷到磁盘）
     但②中 write() 的数据量非常大 → fsync 要花几十甚至几百毫秒
  ④ 主线程：下一个写命令来了 → 又需要 write() 
     但此时 BIO-2 还在 fsync → 文件被锁 → 主线程被迫等待 fsync 完成
     → 阻塞！
```

这也是为什么官方建议不把 AOF 文件放在网络文件系统（NFS、NAS）上的原因——这些文件系统的 fsync 可能更慢。

**`no-appendfsync-on-rewrite` 的作用：** 当该配置为 `yes` 时，在 BGSAVE 或 BGREWRITEAOF 期间，主线程的 fsync 被暂停——数据缓存在 OS buffer 中（最多 30 秒 OS 会自动刷）。这避免了子进程写盘和主进程 fsync 争抢同一块物理磁盘——减少了主线程被磁盘 I/O 阻塞的几率。代价是 BGSAVE 期间如果 Redis 宕机——这部分未 fsync 的 AOF 数据会丢失。

### 3.3 AOF 重写阻塞

AOF 重写（BGREWRITEAOF）本身不阻塞主线程——它和 BGSAVE 一样，通过 fork 子进程来做。但 fork 本身会阻塞（拷贝页表）。此外，AOF 重写过程中，主线程需要往"重写缓冲区"中追加新命令——这个过程也消耗 CPU。

**开销最大的一个场景：** 重写结束时，主线程需要把重写缓冲区的增量命令追加到子进程生成的新 AOF 文件末尾，然后做一次 `rename` 原子替换旧 AOF。如果重写缓冲区的数据很大——这次 `rename` 前的文件写操作可能有一定耗时。

---

## 第四章：BigKey —— 最隐蔽的阻塞源

### 4.1 查找 BigKey —— 扫出隐患

查找 BigKey 分为三步：

**第一步：用 `redis-cli --bigkeys` 找。** 这是内置工具，背后用 SCAN 命令遍历所有 key，统计每种类型的"最大 key"。但它只输出每种类型的最大一个 key——如果第二名、第三名也是 BigKey 就没给到；而且它只统计元素数，不统计内存占用。

**第二步：用 `MEMORY USAGE key` 看实际内存。** 上面的工具说某个 key 有 10000 个元素——但这 10000 个元素每个多大、实际内存占用多少——需要 `MEMORY USAGE key` 来看。

**第三步：用 `INFO commandstats` 间接找。** 如果 `cmdstat_hgetall:usec_per_call=50000`——说明平均每次 HGETALL 耗时 50ms——肯定有大 Hash。如果 `cmdstat_del:usec_per_call=200000`——说明有在删 BigKey（DEL 阻塞了）。

### 4.2 删除 BigKey —— 三种方案对比

**方案一：UNLINK（首选）**

```bash
# ❌ DEL：阻塞（主线程遍历所有元素逐个 free）
DEL big:hash

# ✅ UNLINK：非阻塞（主线程只摘掉引用，BIO-3 后台线程慢慢 free）
UNLINK big:hash
```

**方案二：分批渐进式删除（适合不能 UNLINK 的旧版本）**

对于 Hash / Set / ZSet / List 等，不要一次性 DEL。用 SCAN 每批取 1000 个元素 → 删除 → sleep 10ms → 继续。每批之间的 sleep 是为了让主线程有时间服务其他客户端请求——如果不 sleep，虽然分批了，但 CPU 还是被你自己的脚本占满了。

**方案三：从设计层面防止 BigKey 产生**

- 给缓存 key 设 TTL（数据自动过期）
- List 限制最大长度（`LTRIM key 0 999`）
- Hash/Set/ZSet 按 ID 取模或按日期拆分（而不是全塞在一个 key 里）
- 消息队列的消费速度做监控——消费跟不上就告警

---

## 第五章：清空数据库——FLUSHALL / FLUSHDB

```bash
# ❌ FLUSHALL：主线程同步清空所有 key
# 如果有 5000 万个 key → 可能阻塞几十秒
FLUSHALL

# ✅ FLUSHALL ASYNC：异步清空（Redis 4.0+）
# 主线程只摘掉键空间的引用，BIO-3 后台线程慢慢释放内存
FLUSHALL ASYNC
```

这和 DEL vs UNLINK 是同一个原理——同步操作（FLUSHALL）是在主线程中释放所有 key，异步操作（FLUSHALL ASYNC）是把释放内存的工作交给后台线程。

---

## 第六章：集群扩容——slot 迁移引发的阻塞:o::o:

Redis Cluster 扩缩容需要把 slot 从旧节点迁移到新节点。迁移使用 `MIGRATE` 命令把每个 key 从源节点搬到目标节点。`MIGRATE` 在源节点上是一个**同步阻塞**操作——它把 key 序列化、发送给目标节点、等待目标节点确认、再在本地删除 key——这期间这个 key 在源节点上是"锁定"的。

正常情况下，一个 key 的 MIGRATE 只需要不到 1 毫秒——因为 key 通常是小的。但如果迁移的 slot 中包含 BigKey——比如一个 5000 万成员的 Set——`MIGRATE` 需要把整个 Set 序列化 → 发送 → 目标节点接收 → 反序列化 → 存储——这个过程中源节点的主线程被完全阻塞。5000 万元素的 Set 的 MIGRATE 可能阻塞几秒。

**排查方式：** 查看集群扩缩容期间的延迟抖动——如果延迟飙升和 reshard 的时间点完全吻合——大概率是 BigKey 在 MIGRATE 期间造成的阻塞。

**解决方案：** 集群扩缩容前先用 `redis-cli --bigkeys` 扫描一遍，确认有哪些 BigKey 需要处理。如果 BigKey 无法拆分，迁移时选择业务低峰期并降低迁移速度（通过 `migrate-timeout` 和迁移控制参数）。

---

## 第七章：Swap（内存交换）——阻塞的终极根源:o::o::o::o::o::o::o::o::o:

### 7.1 Redis 最怕 Swap

Swap 是 Linux 的内存交换机制——当物理内存不够时，操作系统会把"暂时不用的内存页"换出到磁盘（Swap 分区），等需要用时再换回来。这是 Linux 的保护机制——防止系统因为内存不够而 OOM。但 Swap 对 Redis 来说是一场灾难。

Redis 的数据全在内存中，它依赖"内存访问 = 纳秒级"的核心优势。一旦操作系统中有一部分 Redis 内存被换出到了磁盘上——CPU 访问到那些页时——触发缺页中断——需要从磁盘把数据读回来——几十毫秒起——在此期间 Redis 主线程被完全阻塞，等待磁盘 I/O 完成。

**排查 Swap 的三个步骤：**

```bash
# ① 查看 Redis 进程是否被 Swap 了
cat /proc/$(pgrep redis-server)/smaps | grep -i swap
# 有非零 Swap 值 → Redis 正在被换出！

# ② 查看系统 Swap 总体用量
free -m
# Swap 的 used 列 > 0 → 系统在换出内存

# ③ 查看碎片率——如果 used_memory_rss < used_memory → 被 Swap 了
redis-cli INFO memory | grep -E "used_memory_rss|used_memory\b"
```

**如果发现 Redis 被 Swap 了，怎么解决？**

- 立即可做：重启 Redis（Swap 全部清零，数据从 AOF/RDB 恢复回物理内存）
- 长期解决：调低 `maxmemory`，让 Redis 用的内存不超过物理内存的 50-70%，留空间给 OS + COW + 其他进程
- 根本解决：加物理内存——Redis 不应该运行在"内存不够"的机器上

**预防 Swap：** `vm.swappiness = 0`（Linux 内核参数）——但这只是降低了 Swap 的概率，不能完全阻止 Swap。最好的预防是"物理内存足够大，让 Redis 的数据全部装得下"。

---

## 第八章：CPU 竞争

### 8.1 Redis 和其他进程抢 CPU

Redis 是单线程的——它只能利用一个 CPU 核心。但如果同一台机器上还有其他进程在大量消耗 CPU——比如定期脚本、日志清理、备份任务、甚至其他 Redis 实例——它们也会抢占 CPU。Redis 的"公平调度"在 CPU 繁忙时会被操作系统频繁挂起——原本 1 微秒的 GET 被拉长到几十微秒——用户感知到延迟。

**排查：** `htop` 或 `top` 看 CPU 使用率。如果 Redis 进程 CPU 很高但 QPS 不高——可能有慢命令。如果其他进程 CPU 很高——Redis 被抢占了。

**解决方案：** Redis 实例绑定到固定的 CPU 核心（`taskset`）；其他耗 CPU 任务放到不同核心上；同一台服务器不要同时跑多个 Redis 实例。

### 8.2 Redis 6.0+ 多线程 I/O 对 CPU 的影响

Redis 6.0+ 引入了多线程 I/O——配置 `io-threads 4` 可以让主线程以外的 3 个线程分担网络读写。但如果机器只有 4 核 → 这 4 个线程已经把 CPU 占满，其他系统进程无法得到调度 → 整体响应变慢。`io-threads` 不要超过逻辑核心数，且通常建议设为 2-4。

---

## 第九章：网络问题

### 9.1 网络带宽被打满

Redis 的响应数据通过网络传输。如果出流量打满网卡——后续请求的回复排不进发送队列——Redis 主线程在写 Socket 时可能被阻塞。典型场景：一个 BigKey String（50MB）频繁被 GET→每次返回 50MB→20 次/秒就把千兆网卡打满了。

**排查：** 用 `iftop` 或 `nload` 看实时带宽。如果接近网卡上限——排查哪个 key 在占用流量（BigKey 或热 key 读取量）。

### 9.2 客户端连接数的隐形影响

Redis 主线程每次事件循环要轮询所有客户端连接的 Socket。如果连接数过多（比如 5 万+），即使大部分是空闲连接，主线程在 `epoll_wait` 和事件分发上也会消耗更多 CPU。而且每个连接维护一个输出缓冲区——如果缓冲区设置过大，总内存也很可观。

**排查：** `redis-cli INFO clients | grep connected_clients`——如果超过几万，排查哪个业务创建了大量空闲连接。

### 9.3 TCP backlog 和连接风暴

Redis 使用 TCP 的 backlog 队列存放尚未 accept 的连接。如果瞬时涌入大量连接（如服务重启后所有实例同时连接 Redis）→ backlog 溢出 → 新连接被拒绝。设置 `tcp-backlog 511`（默认）可以根据操作系统内核的 `net.core.somaxconn` 来调大。

---

## ⭐️ 面试题汇总

**Q1: 线上 Redis 出现周期性延迟毛刺（每几分钟卡几十毫秒），怎么排查？**

> ① `SLOWLOG GET 10` 看是否有慢命令（KEYS/HGETALL/DEL BigKey）；② `INFO commandstats` 看 `usec_per_call` 是否有异常高的；③ `INFO stats` 看 `latest_fork_usec`——有可能是 BGSAVE 的 fork 或 AOF 重写的 fork；④ `redis-cli --bigkeys` 扫描是否有 BigKey 被定期删除；⑤ 检查系统 `free -m` 看 Swap。

**Q2: KEYS * 为什么是生产禁令？替代方案是什么？**

> KEYS 遍历所有 key O(N)，几百万 key 可能阻塞数百毫秒——期间 Redis 无法处理其他请求。替代：SCAN 命令分批遍历，每次只处理少量 key，不阻塞。

**Q3: DEL 命令也会阻塞，为什么？怎么解决？**

> DEL 在主线程遍历所有元素逐个 free 内存——1000 万成员的 Set = 可能阻塞几百毫秒。解决：UNLINK（异步删除）——主线程只摘引用，后台线程慢慢 free。或分批 SCAN + HDEL/SREM，每批之间 sleep 10ms 给主线程喘息。

**Q4: Swap 为什么对 Redis 是致命打击？如何排查和解决？**

> Redis 数据在内存，Swap 把部分内存换到磁盘 → CPU 访问时触发缺页中断 → 从磁盘读回 → 几十毫秒延迟。排查：`cat /proc/<pid>/smaps | grep Swap` 看是否非零；`free -m` 看系统 Swap 用量。解决：调低 maxmemory 保证物理内存够用；或加物理内存；根本不要让 Redis 碰上 Swap。

**Q5: 所有类型 BigKey 各自对应的阻塞命令是什么？**

> String BigKey：GET/SET/DEL 都是 O(1) 但数据量大（带宽、内存拷贝）；List BigKey：LRANGE 0 -1 和 LINDEX 大索引 → O(N)；Hash BigKey：HGETALL O(N)；Set BigKey：SMEMBERS O(N)、SINTER/SUNION/SDIFF O(N*M)；ZSet BigKey：ZRANGE 0 -1 O(logN+M)、ZRANGEBYSCORE -inf +inf。

---

*Created: 2026-05-26 | Category: 23-Redis常见阻塞原因*
