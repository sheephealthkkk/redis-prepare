# Redis 深度讲解提示词 —— 四章合集

> 目标：冲击大厂（字节/阿里/腾讯）后端岗位
> 要求：源码级深度、Mermaid 可视化、面试追问全覆盖

---

# 第一章：Redis 主从复制

```markdown
# 角色设定
你是一位资深的Redis内核专家，正在为一位准备冲击大厂（字节/阿里/腾讯）的后端工程师做技术讲解。请用极具深度的方式讲解Redis主从复制，要求：
- 深入到源码级原理，而非仅停留在"是什么"的层面
- 每个知识点都要回答"为什么要这样设计"
- 使用 Mermaid 图直观展示核心流程
- 穿插大厂面试高频追问
- 语言精炼、不废话，直击要害

---

# 第一章：Redis主从复制

## 1.1 为什么需要主从复制
- 从三个维度讲清楚价值：读写分离（性能）、数据冗余（可靠性）、为哨兵/集群做铺垫（架构演进）
- 用 Mermaid 画一个「单体Redis → 一主多从」的架构演进图

## 1.2 主从复制的配置方式
- `replicaof` / `slaveof` 命令的用法与区别（需指出 Redis 5.0 的命名变更）
- 配置文件 vs 动态命令两种方式
- 演示：一主两从的搭建命令（用伪终端的方式展示）
- 强调：**replica 默认只读**（`replica-read-only yes`），为什么不能写？
- 追问：如果 replica 上误写入数据会发生什么？

## 1.3 主从复制的核心流程——深入源码级分析

### 1.3.1 建立连接阶段
- replica 向 master 发送 `PSYNC ? -1`（首次）或 `PSYNC <replid> <offset>`（重连）
- 用 Mermaid 序列图画出 replica ↔ master 的握手过程
- 关键源码逻辑（用伪代码表示）：
  - `replicationSetMaster()` → `connectWithMaster()` → `syncWithMaster()`

### 1.3.2 全量复制（Full Resynchronization）
- 用 Mermaid 流程图画出全量复制的完整时序：
  1. master 收到 PSYNC，返回 `+FULLRESYNC <replid> <offset>`
  2. master 执行 `bgsave` fork 子进程生成 RDB
  3. master 将 RDB 期间的写命令写入 **client-output-buffer**（replication buffer）
  4. RDB 生成完毕，发送给 replica
  5. replica 清空旧数据 → 加载 RDB → 回放 buffer 中积压的命令
- **深度追问**：
  - fork 子进程时，master 能继续处理写请求吗？（COW 机制）
  - 如果 RDB 文件很大（如 20GB），会有什么问题？（复制超时、buffer 溢出）
  - `client-output-buffer-limit slave` 参数的作用与设置公式

### 1.3.3 部分复制（Partial Resynchronization）
- **为什么需要部分复制**：解决短暂断连后全量复制的开销
- **三大核心数据结构**（用 Mermaid 画关系图）：
  1. `replicationID`（主库唯一标识，主库切换后会变化）
  2. `replication offset`（主从各自的复制偏移量）
  3. `replication backlog`（环形缓冲区，默认 1MB）
- 用 Mermaid 序列图画出部分复制的交互：
  1. replica 重连，发送 `PSYNC <old_replid> <offset>`
  2. master 校验 replid 匹配 && offset 在 backlog 范围内
  3. master 返回 `+CONTINUE`，从 backlog 中发送增量数据
- **追问**：
  - backlog 大小如何设置？（`repl-backlog-size`，建议 = 断线时长 × 写入速率 × 2）
  - 什么情况下部分复制会退化为全量复制？
  - 主库重启后 replicationID 变了，所有从库都会触发全量复制吗？

### 1.3.4 命令传播阶段
- 全量/部分复制完成后进入**异步传播模式**
- master 每执行一条写命令，异步广播给所有 replica
- **追问**：异步复制意味着什么？——数据一致性问题（最终一致性）

## 1.4 心跳与连接维护
- `REPLCONF ACK <offset>`：replica 每秒向 master 上报自己的 offset
- 用 Mermaid 画出心跳交互图
- 心跳的三大作用：
  1. 检测主从连接状态
  2. 上报自身 offset，辅助 master 判断数据一致性
  3. 辅助 `min-replicas-to-write` 功能
- **追问详解**：
  - `min-replicas-to-write 3` 和 `min-replicas-max-lag 10` 配合使用 → 实现「写操作需要至少3个从库确认才返回」。**这能保证强一致性吗？为什么？**
  - `repl-timeout` 参数的设置：默认60秒，太大了会怎样？太小了会怎样？

## 1.5 关键配置参数深度解析（表格形式）
| 参数 | 默认值 | 含义 | 调优建议 |
|------|--------|------|----------|
| repl-backlog-size | 1MB | 复制积压缓冲区大小 | 根据写入量调大 |
| repl-timeout | 60s | 复制超时 | 内网可适当调小 |
| client-output-buffer-limit slave | 256MB/64MB/60s | replica 输出缓冲区限制 | 大RDB场景需调大 |
| replica-read-only | yes | 从库只读 | 除非做数据预处理，否则不要改为 no |
| min-replicas-to-write | 0 | 最少在线从库数 | 高可靠性场景设为 >=1 |
| min-replicas-max-lag | 10 | 从库最大延迟秒数 | 根据业务容忍度调整 |

## 1.6 常见问题与事故排查
- **问题1**：从库追不上主库，延迟越来越大
  - 原因：写入量大 + 从库单线程处理不过来
  - 定位：`INFO replication` 看 offset 差值
  - 解决：增加从库机器性能、或做读写分离的二次路由

- **问题2**：主库重启后，所有从库触发全量复制，导致主库CPU/内存/网络暴涨
  - 原因：replicationID 变化
  - 解决：Redis 4.0+ 支持 RDB 中保存 repl-id（`rdb-save-incremental-fsync` 相关），或使用哨兵做故障转移避免直接重启主库

- **问题3**：全量复制期间，replication buffer 不够导致复制反复失败
  - 用 Mermaid 画出此问题的时序图
  - 解决：增大 `client-output-buffer-limit slave`，或调整 `repl-diskless-sync`

- **问题4**：复制风暴——一个主库挂载太多从库
  - 用 Mermaid 画出树状级联复制架构（主 → 从 → 从）解决

## 1.7 Redis 2.8 前后的复制演进
- 2.8 之前：只有 `SYNC`，每次重连都全量复制
- 2.8 之后：`PSYNC`，支持部分复制
- 4.0 之后：`PSYNC2`，解决主从切换后的 repl-id 不匹配问题
- **追问**：PSYNC2 相比 PSYNC1 的核心改进是什么？（`repl-id2`）

## 1.8 面试模拟——10个高频追问
请以"面试官问 → 候选人的满分回答"的形式，给出以下问题的深度解答：

1. Redis主从复制是全量还是增量？
2. 全量复制期间，master的写请求怎么处理？会丢吗？
3. 部分复制的 backlog 是什么数据结构？大小如何计算？
4. 主从断开重连后，什么情况下走部分复制，什么情况下走全量复制？
5. Redis主从复制的原理是什么？尽可能详细地说出完整流程。
6. 从库会删除过期key吗？策略是什么？
7. 主从复制有哪些延迟问题？如何监控和优化？
8. 为什么建议Redis主从都开启持久化？不开启会怎样？
9. `replica-serve-stale-data` 参数的三个选项（yes/no/clients）有什么区别？
10. 如果让你设计一个Redis全量同步的优化方案（如无盘复制），你怎么做？

## 1.9 总结
- 用 Mermaid 画出一张「全景架构图」，把全量复制、部分复制、命令传播、心跳机制串在一起
- 一页纸总结核心要点，方便面试前快速回顾
```

---

# 第二章：Redis 哨兵（Sentinel）

```markdown
# 角色设定
你是一位资深的Redis内核专家，正在为一位准备冲击大厂（字节/阿里/腾讯）的后端工程师做技术讲解。请用极具深度的方式讲解Redis哨兵（Sentinel），要求：
- 深入到源码级原理，而非仅停留在"是什么"的层面
- 每个知识点都要回答"为什么要这样设计"
- 使用 Mermaid 图直观展示核心流程
- 穿插大厂面试高频追问
- 语言精炼，不废话，直击要害

---

# 第二章：Redis哨兵（Sentinel）

## 2.1 为什么需要哨兵
- **痛点**：纯主从架构中，master 挂了你得手动切。半夜3点 master 挂了怎么办？
- 哨兵的本质：运行在**特殊模式下的 Redis 进程**（不处理数据，只做监控和协调）
- 哨兵四大核心能力（用 Mermaid 画功能全景图）：
  1. **监控（Monitoring）**：周期性检测主从存活
  2. **通知（Notification）**：将故障通知给运维/其他系统
  3. **自动故障转移（Automatic Failover）**：主挂了，选一个新主
  4. **配置提供者（Configuration Provider）**：客户端通过哨兵获取当前 master 地址
- **追问**：哨兵本身能用单节点吗？为什么最少要3个？

## 2.2 哨兵的部署架构
- 用 Mermaid 画一个「3哨兵 + 1主2从」的经典部署拓扑图
- **关键原则**：哨兵节点数必须是奇数且 ≥ 3，为什么？
  - 与多数派投票机制有关，引出 Quorum 概念
- 哨兵与 Redis 节点的网络交互全景图（Mermaid）

## 2.3 哨兵的工作原理——深度剖析

### 2.3.1 三个定时任务（核心监控机制）
用 Mermaid 时间线图画出三个任务并行执行的时序关系：

**任务1：每 10s 一次 `INFO` 命令**（对主库和对从库都发）
- 目的：发现拓扑变化（新从库上线、从库下线）
- **追问**：哨兵是如何自动发现其他哨兵和其他从库的？
  - 答：通过订阅 master 的 `__sentinel__:hello` 频道

**任务2：每 2s 一次 `PING` 命令**（对所有节点，包括其他哨兵）
- 目的：判断节点是否存活
- **追问**：PING 一下没回就是挂了？——引出 `down-after-milliseconds`

**任务3：每 1s 一次 `PUBLISH` 消息**发送到 `__sentinel__:hello` 频道
- 消息内容：`<ip> <port> <runid> <current_epoch> <master_name> <master_ip> <master_port> <config_epoch>`
- 目的：哨兵之间交换对主库的监控信息和集群配置
- 用 Mermaid 画出哨兵间通过 Pub/Sub 发现彼此的流程图

### 2.3.2 主观下线（SDOWN）vs 客观下线（ODOWN）
- **主观下线（Subjective Down，SDOWN）**：
  - 单个哨兵在规定时间（`down-after-milliseconds`）内没收到某节点的有效 PING 回复
  - 此时该哨兵**自己**认为节点挂了
  
- **客观下线（Objectively Down，ODOWN）**：
  - 针对**主库**才有（从库和哨兵只有 SDOWN）
  - 当 ≥ Quorum 个哨兵都认为主库 SDOWN → 主库被标记为 ODOWN
  - 用 Mermaid 画出 SDOWN → ODOWN 的判断流程图

- **追问**：ODOWN 是客观的吗？如果 Quorum 个哨兵都网络分区了呢？
  - 引出：ODOWN 只是故障转移的**前置条件**，不是充分条件

- **参数对比**：
  ```
  down-after-milliseconds  → 多久无响应判定为 SDOWN
  quorum                  → 多少个哨兵同意才判定为 ODOWN
  ```

- **追问**：`down-after-milliseconds` 设置多长合适？太短误切、太长慢切，怎么平衡？

### 2.3.3 哨兵领导者选举（Leader Election）
- 当主库被标记为 ODOWN 后，哨兵之间需要选出一个**Leader**来执行故障转移
- 用 Mermaid 时序图画出的 **Raft-like 选举算法**：
  1. 发现 ODOWN 的哨兵向其他哨兵发送 `SENTINEL is-master-down-by-addr` 命令，请求投票
  2. 投票规则：先到先得、每个 epoch 只能投一票
  3. 获得 ≥ `max(quorum, N/2+1)` 票的哨兵成为 Leader
- **追问**：为什么是 `max(quorum, N/2+1)`？
  - quorum 太小 → 可能导致脑裂（两个 Leader 同时执行故障转移）
  - `N/2+1` 保证了多数派原则

- **追问详解**：epoch（配置纪元）的作用
  - 类似 Raft 的 term，每次故障转移 epoch + 1
  - 防止旧 Leader 的过期指令生效

- 用 Mermaid 画一个「3哨兵但 quorum=1」的脑裂场景 → 说明为什么 quorum 不能太小

### 2.3.4 故障转移执行（Failover Execution）
用 Mermaid 流程图/序列图画出完整的故障转移步骤：

**Step 1：选出新主库**
- Leader 哨兵从所有在线从库中筛选新主库，筛选规则（优先级从高到低）：
  1. 排除不健康的从库（SDOWN / ODOWN / 断连超过 `down-after-milliseconds * 10`）
  2. 按 `slave-priority` 升序（数字越小优先级越高，0 表示永不提升）
  3. 按 `replication offset` 降序（数据最新的优先）
  4. 按 `runid` 字典序（兜底）
- 追问：如果一个从库数据比主库旧很多，被选为主库后会丢失数据吗？

**Step 2：`SLAVEOF NO ONE`**
- Leader 哨兵向选中的从库发送 `SLAVEOF NO ONE`，该从库升级为主库
- 追问：这个命令执行期间，客户端请求怎么处理？——需要 `replica-serve-stale-data` 配合

**Step 3：重新绑定从库**
- Leader 哨兵向剩余从库发送 `SLAVEOF <new_master_ip> <new_master_port>`
- 这些从库会重新与新主库进行全量/部分同步

**Step 4：旧主库回归**
- 当旧主库恢复后，哨兵将其降级为从库，绑定到新主库
- 用 Mermaid 画出「旧主恢复 → 发现自己是旧主 → 被降级为从库」的时序

- **追问**：如果旧主库在故障转移期间还接收了客户端写入（脑裂场景），这些数据怎么处理？

## 2.4 哨兵与客户端的交互
- 用 Mermaid 画出客户端通过哨兵发现主库的完整流程：
  1. 客户端连接哨兵，发送 `SENTINEL get-master-addr-by-name <master_name>`
  2. 哨兵返回当前主库的 IP:Port
  3. 客户端订阅哨兵的 `+switch-master` 频道
  4. 主库切换时，哨兵推送新主库地址
  5. 客户端收到通知，重建连接

- **追问**：客户端在收到切换通知之前，向旧主发起的写请求怎么处理？（引出读写分离客户端的容错设计）
- 常见客户端库（Jedis/Lettuce/Redisson）的哨兵接入方式

## 2.5 关键配置参数深度解析

| 参数 | 默认值 | 含义 | 调优建议 |
|------|--------|------|----------|
| sentinel monitor mymaster 127.0.0.1 6379 2 | - | 监控主库，最后一个数字是 quorum | quorum 建议 = N/2+1（取整） |
| sentinel down-after-milliseconds mymaster 30000 | 30000ms | SDOWN 判定时间 | 内网可调至 5000-10000ms |
| sentinel parallel-syncs mymaster 1 | 1 | 故障转移后同时同步的新从库数量 | 从库多时可适当调大 |
| sentinel failover-timeout mymaster 180000 | 3分钟 | 故障转移超时时间 | 大内存实例适当调大 |
| sentinel auth-pass mymaster password | - | 主库密码 | 生产环境必须设置 |
| sentinel notification-script | - | 告警脚本 | 接入钉钉/企微/飞书告警 |

- **追问详解**：`sentinel parallel-syncs` 为什么默认是 1？
  - 多个从库同时全量同步 → 主库 fork 压力大、网络带宽打满
  - 串行化同步但故障恢复时间变长 → 需要权衡

## 2.6 哨兵配置的自动重写
- 哨兵会**自动修改** `sentinel.conf` 文件
- 哪些内容会被自动修改：已知从库信息、其他哨兵信息、当前 epoch 等
- **追问**：为什么不要手动编辑运行中的 sentinel.conf？
  - 会被覆盖
  - 正确的变更方式：`SENTINEL SET` / `SENTINEL MONITOR` / `SENTINEL REMOVE` 命令

## 2.7 常见故障场景演练
用 Mermaid 时序图画出以下4个故障场景的完整流程：

**场景1：主库进程崩溃 → 哨兵自动切换**
- 时序：PING 超时 → SDOWN → ODOWN（投票） → Leader选举 → 故障转移 → 通知客户端

**场景2：主库整机宕机 → 旧主恢复后回归**
- 时序：同场景1，但多了旧主恢复后被降级为从库的过程

**场景3：哨兵集群自身故障**
- 3个哨兵挂了1个：quorum=2，正常判定 ODOWN，正常切换
- 3个哨兵挂了2个：只剩1个，够不成多数派，**故障转移不会执行**，用 Mermaid 画出这个僵死状态
- **追问**：挂2个哨兵后，主库真的挂了怎么办？——人工介入

**场景4：网络分区（脑裂）**
- 哨兵集群与主库之间网络不通，但主库仍能给部分客户端写数据
- 哨兵判定 ODOWN → 选出新主 → **双主并存（脑裂）**
- 用 Mermaid 画出脑裂场景，及旧主恢复后的数据冲突
- **追问**：如何尽量减少脑裂的数据丢失？
  - 答案：`min-replicas-to-write` + `min-replicas-max-lag` 让旧主在从库不够时拒绝写入

## 2.8 哨兵日志解读与运维技巧
- 关键日志的分析模板：
  ```
  +sdown master mymaster 127.0.0.1 6379        # 主观下线
  +odown master mymaster 127.0.0.1 6379 #quorum 3/2  # 客观下线
  +try-failover master mymaster 127.0.0.1 6379  # 尝试故障转移
  +vote-for-leader <runid> <epoch>               # 投票
  +elected-leader master mymaster 127.0.0.1 6379 # 当选 Leader
  +failover-state-select-slave                   # 选择新主阶段
  +selected-slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster  # 选定新主
  +failover-state-send-slaveof-noone slave 127.0.0.1:6380  # 提升为新主
  +failover-end master mymaster                  # 故障转移完成
  +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6380  # 主库切换
  -sdown slave 127.0.0.1:6379                    # 旧主恢复，SDOWN 解除
  +convert-to-slave slave 127.0.0.1:6379         # 旧主被降级为从库
  ```
- 哨兵日志的分析思路与故障回溯方法

## 2.9 面试模拟——12个高频追问
请以"面试官问 → 候选人的满分回答"的形式，给出以下问题的深度解答：

1. 哨兵的工作原理是什么？请完整描述从监控到故障转移的全流程。
2. 什么是主观下线（SDOWN）和客观下线（ODOWN）？区别是什么？
3. 哨兵选主的逻辑是什么？哪个从库会被提升为主库？
4. 哨兵之间是如何通信的？哨兵如何发现其他哨兵？
5. 哨兵集群的 quorum 和 majority 有什么区别？分别影响什么？
6. quorum 设成1可以吗？会有什么问题？
7. 哨兵集群至少需要几个节点？为什么？
8. 哨兵在故障转移时，客户端如何感知到主库变了？
9. 哨兵本身的故障怎么办？哨兵集群挂了会影响 Redis 数据吗？
10. 脑裂是怎么发生的？哨兵能完全避免脑裂吗？
11. 请比较哨兵集群和 Cluster 集群的区别，各自的适用场景是什么？
12. 哨兵切换过程中，如果有客户端持续向旧主写入数据，这部分数据会丢失吗？如何尽量降低丢失量？

## 2.10 生产实战配置模板
给出一个可直接用于生产环境的 `sentinel.conf` 模板（带详细注释）：
```ini
# 监控主库
sentinel monitor mymaster 10.0.1.10 6379 2
# 密码
sentinel auth-pass mymaster <password>
# SDOWN 判定：5秒无响应
sentinel down-after-milliseconds mymaster 5000
# 故障转移时只有一个从库与新主同步
sentinel parallel-syncs mymaster 1
# 故障转移超时：3分钟
sentinel failover-timeout mymaster 180000
# 告警脚本
sentinel notification-script mymaster /etc/redis/notify.sh
# 客户端重配置脚本
sentinel client-reconfig-script mymaster /etc/redis/client-reconfig.sh
```

## 2.11 总结
- 用 Mermaid 画出一张「哨兵全景架构图」：3哨兵 + 1主2从 + Pub/Sub 通信 + 客户端交互
- 一页纸速查表：哨兵核心流程、关键参数、常见故障处理方案
```

---

# 第三章：Redis 分片集群（Cluster）

```markdown
# 角色设定
你是一位资深的Redis内核专家，正在为一位准备冲击大厂（字节/阿里/腾讯）的后端工程师做技术讲解。请用极具深度的方式讲解Redis分片集群（Cluster），要求：
- 深入到源码级原理，而非仅停留在"是什么"的层面
- 每个知识点都要回答"为什么要这样设计"
- 使用 Mermaid 图直观展示核心流程
- 穿插大厂面试高频追问
- 语言精炼，不废话，直击要害

---

# 第三章：Redis分片集群（Cluster）

## 3.1 为什么需要分片集群
- 回顾架构演进线（用 Mermaid 画演进图）：
  ```
  单机Redis → 主从复制（读写分离）→ 哨兵（高可用）→ Cluster（横向扩展 + 高可用一体）
  ```
- **核心矛盾**：
  | 架构 | 解决了什么 | 解决不了什么 |
  |------|-----------|-------------|
  | 主从 | 读写分离 + 数据冗余 | 单机内存上限、写性能瓶颈 |
  | 哨兵 | 自动故障转移 | 依然单机内存上限、写QPS线性增长 |
  | 分片集群 | 数据分片 + 水平扩展 + 高可用 | 运维复杂度、跨槽事务受限 |
  
- **追问**：什么业务量级才需要上集群？日活100万？1亿？
  - 答：不是看日活，是看**内存使用量**和**写 QPS**。单机 32GB 撑不住就得上集群。

## 3.2 数据分片——哈希槽（Hash Slot）

### 3.2.1 为什么是 16384 个槽
- CRC16 取模 → 16384 个哈希槽，范围 0 ~ 16383
- **追问**：为什么是 16384，而不是更多或更少？
  - CRC16 能表示 65535 个值，Redis 只用了 16384
  - 原因1：**心跳消息体大小**。集群节点间 gossip 消息需要携带槽位信息，16384 个槽只需要 2KB 的 bitmap（16384/8=2048字节）
  - 原因2：**节点数限制**。Redis推荐节点数不超过1000，16384已足够均匀分布
  - 用 Mermaid 画出「key → CRC16 → 槽 → 节点」的映射流程图

### 3.2.2 槽与节点的映射
- 每个 master 节点负责一部分槽
- 用 Mermaid 画一个「3主3从、16384槽均匀分配」的集群拓扑图
- **Hash Tag** 机制：`{user:1001}:age` 和 `{user:1001}:name` 中的 `{}` 部分参与哈希
  - 用 Mermaid 画图对比：不使用 Hash Tag（数据散列到不同节点） vs 使用 Hash Tag（同一用户数据落在同一节点）
  - **追问**：Hash Tag 会带来什么问题？——数据倾斜，热 key 问题

### 3.2.3 键到槽的源码级路径
```
key = "order:10086"
slot = CRC16(key) & 0x3FFF    → 等价于 CRC16(key) % 16384
node = cluster->slots[slot]   → 查槽-节点映射表
```
- **追问**：CRC16 和 CRC32 有什么区别？为什么不用一致性哈希？
  - Redis 没选一致性哈希的原因：节点变更时数据迁移的复杂度、槽的重分配更灵活

## 3.3 集群通信——Gossip 协议

### 3.3.1 节点间通信机制
- 用 Mermaid 画出一个「两两互联」的集群通信拓扑（Meet 消息 → 全网互联）
- **Gossip 协议核心本质**：每个节点随机选择其他节点，周期性交换元数据
- Redis 集群端口 = 业务端口 + 10000（如 6379 → 16379 为集群总线端口）
- **追问**：为什么不用集中式元数据存储（如 ZooKeeper / etcd）？
  - 去中心化、无单点故障、无需额外依赖

### 3.3.2 Gossip 消息类型（5种）
用 Mermaid 状态图画出5种消息的发送场景：

| 消息类型 | 中文含义 | 发送时机 |
|---------|---------|---------|
| MEET | 加入集群 | 新节点加入时，管理员执行 `CLUSTER MEET` |
| PING | 心跳检测 | 每秒随机挑选节点发送（附带自身元数据） |
| PONG | 回复 PING / MEET | 收到 PING/MEET 后回复 |
| FAIL | 标记故障 | 节点被多数派认定为 FAIL 后广播 |
| PUBLISH | 消息发布 | 客户端向任意节点 PUBLISH，节点广播到全集群 |

- **追问详解**：PING 消息里携带了什么？
  - 当前节点信息（id, ip, port, flags）
  - 当前节点负责的槽位 bitmap
  - 当前节点视角下**部分其他节点**的信息（Gossip 传染）
  - 为什么只携带部分？——控制消息体大小，用「传染」代替「广播全量」

### 3.3.3 Gossip 的传染机制
- 用 Mermaid 画出「节点A → 节点B（携带A+C的元数据）→ 节点B更新C的状态」的 Gossip 传染过程
- **追问**：最终一致性 vs 强一致性——Gossip 的元数据传播有延迟，会带来什么问题？
  - 槽信息不一致 → 客户端收到 MOVED 重定向
  - 故障判定不一致 → 短暂的双主可能

### 3.3.4 节点数量与Gossip消息量的关系
- 集群节点越多，Gossip 消息越多，O(n²) 复杂度
- **追问**：Redis Cluster 的推荐节点上限是多少？为什么？
  - 官方推荐不超过 1000 个节点
  - 节点太多 → Gossip 消息爆炸 + 槽分配不均匀

## 3.4 客户端重定向——MOVED 与 ASK

### 3.4.1 MOVED 重定向（永久性）
- 用 Mermaid 序列图画出：
  1. 客户端向节点A执行 `GET order:10086`
  2. 节点A计算槽 → 不在自己范围内 → 返回 `MOVED 12706 10.0.0.2:6379`
  3. 客户端收到 MOVED → 向节点B重新请求 → 得到结果
  4. 客户端**更新本地槽-节点映射缓存**
- **追问**：为什么客户端要缓存槽映射？
  - 避免每次请求都走 MOVED 重定向（额外一次网络往返）
  - 智能客户端（JedisCluster / Lettuce）都内置了槽缓存

### 3.4.2 ASK 重定向（临时性）
- 场景：槽正在迁移中，部分 key 还没迁走
- 用 Mermaid 序列图画出槽迁移期间的 ASK 重定向：
  1. 客户端向节点A请求 key → 节点A发现该 key 已迁移到节点B
  2. 节点A返回 `ASK 12706 10.0.0.2:6379`
  3. 客户端向节点B发送 `ASKING` + 原命令
  4. 节点B处理请求并返回结果
- **追问**：MOVED 和 ASK 的本质区别是什么？
  | 维度 | MOVED | ASK |
  |------|-------|-----|
  | 含义 | 槽永久迁走了 | 槽正在迁移中，key 临时在目标节点 |
  | 客户端行为 | 更新槽缓存 | 不更新槽缓存，下次还要走 ASK |
  | 前导命令 | 无需 | 必须发 `ASKING` 命令 |

- **追问**：`ASKING` 命令的作用是什么？
  - 临时让目标节点忽略「该槽不属于我」的校验，强制处理该请求
  - 仅对下一次命令生效

## 3.5 集群故障检测与自动转移

### 3.5.1 故障检测流程（PFail → Fail）
用 Mermaid 流程图画出两段式故障判定：

**第一阶段：PFail（疑似故障）**
- 节点A在规定时间内没收到节点B的 PONG → 将B标记为 PFail
- 此时只是节点A**自己的看法**，不会触发故障转移

**第二阶段：Fail（确认故障）**
- 节点A通过 Gossip 传播 B 的 PFail 信息
- 当**半数以上主节点**都将 B 标记为 PFail → B 被标记为 Fail
- 标记为 Fail 后，广播 FAIL 消息 → 触发从节点选举
- **追问**：这里和哨兵的 SDOWN/ODOWN 机制有什么异同？
  - 相似：都是两阶段判定
  - 不同：哨兵是独立进程组投票，集群是节点间 Gossip 投票，去中心化

### 3.5.2 故障转移流程
用 Mermaid 序列图画出：

1. **从节点发现主节点 Fail**
2. **从节点增加 currentEpoch**，发起选举
3. **从节点向所有主节点请求投票**（`FAILOVER_AUTH_REQUEST`）
4. **主节点投票规则**：每个 epoch 只投一票、先到先得
5. **获得 ≥ N/2+1 票** → 当选新主
6. **新主执行 `clusterDelSlot` + `clusterAddSlot`**，接管槽位
7. **新主广播 PONG**，宣告自己是新主
- **追问**：从节点检测到主 FAIL 后，**不是立刻选举**，而是等待一段随机时间（0.5s~1s）。为什么？
  - 错峰选举：如果主节点下有多个从节点，避免同时发起选举
  - offset 越大的从节点**等待时间越短** → 数据最新的从节点更可能当选

### 3.5.3 集群的 epoch 机制
- `currentEpoch`：全局逻辑时钟，每次选举 +1
- `configEpoch`：槽配置版本号，解决槽所有权冲突
- **追问**：epoch 和哨兵的 epoch 有什么类似和不同？
  - 类似：都是逻辑时钟，防止过期消息
  - 不同：哨兵是配置纪元，集群还用于槽配置的冲突解决

## 3.6 集群扩容与缩容——数据迁移

### 3.6.1 槽迁移的完整流程
用 Mermaid 画出槽从一个节点迁移到另一个节点的完整流程：

```
源节点A（槽12706）                        目标节点B
    │                                        │
    │ ① CLUSTER SETSLOT 12706 importing B    │
    │                                        │ ② CLUSTER SETSLOT 12706 migrating A
    │                                        │ 
    │ ③ CLUSTER GETKEYSINSLOT 12706 <count>  │
    │ ← 返回N个key                           │
    │                                        │
    │ ④ MIGRATE B IP PORT key 0 <timeout>    │ ← 原子迁移单个key
    │                                        │
    │ ⑤ 重复③④直到槽清空                      │
    │                                        │
    │ ⑥ CLUSTER SETSLOT 12706 node B        │ ← 告知全集群槽已迁移
    │                                        │ ⑥ CLUSTER SETSLOT 12706 node B
```

- **追问**：`MIGRATE` 命令是原子的吗？迁移期间有客户端请求该 key 怎么办？
  - MIGRATE 本身是原子操作（DEL旧 + RESTORE新）
  - 迁移期间的请求走 ASK 重定向（3.4.2 已讲）

### 3.6.2 rehash 期间的边角 case
- 槽迁移中断了怎么办？——需要 `CLUSTER REPLICATE` / `CLUSTER RESET` 手动处理
- 源节点宕机了怎么办？——槽迁移失败，数据在源节点丢失（除非有持久化）
- **追问**：如何安全地进行在线扩容？
  1. 新节点 `CLUSTER MEET` 加入集群
  2. 使用 `redis-cli --cluster reshard` 或 `CLUSTER SETSLOT` 逐步迁移
  3. 逐槽迁移，每次少量 key，控制迁移期间的抖动

### 3.6.3 缩容——节点下线
- 用 Mermaid 画出「节点下线前先迁移所有槽 → `CLUSTER FORGET`」的流程
- 任意节点都可以 `CLUSTER FORGET`，但会被 Gossip 重新发现 → 需要在**所有节点**上都执行 `CLUSTER FORGET`

## 3.7 集群的限制
用表格明确列出 Redis Cluster 不支持或受限的操作：

| 限制 | 说明 | 替代方案 |
|------|------|---------|
| 多 key 操作 | mget/mset 的 key 必须在同一槽 | 用 Hash Tag 约束到同一槽 |
| 跨节点事务 | 无法保证跨槽 ACID | 业务层补偿、TCC、SAGA |
| Lua 脚本 | 脚本中的 key 必须在同一节点 | 用 Hash Tag + `EVALSHA` |
| Pub/Sub | 消息会广播到所有节点 | 小型集群OK，大集群慎用 |
| 管道（Pipeline） | 跨节点管道不保证原子性 | 按槽分组发送管道 |

- **追问**：如果业务强依赖跨 key 的事务，用什么方案？
  - 一致性哈希 + Twemproxy/Codis（集中式路由，事务在代理层处理）
  - 或者直接用关系型数据库
  - 或者业务层引入分布式事务（TCC / SAGA）

## 3.8 集群配置深度解析

### 3.8.1 关键配置参数

| 参数 | 默认值 | 含义 | 调优建议 |
|------|--------|------|----------|
| cluster-enabled yes | no | 开启集群模式 | - |
| cluster-config-file nodes.conf | - | 集群节点信息持久化文件 | 不要手动编辑 |
| cluster-node-timeout 15000 | 15s | 节点超时时间 | 内网可调至5000ms |
| cluster-slave-validity-factor 10 | 10 | 从节点有效因子 | 大内存实例调大 |
| cluster-migration-barrier 1 | 1 | 从节点迁移屏障 | 高可用场景设为 >= 2 |
| cluster-require-full-coverage yes | yes | 是否要求全槽覆盖 | 生产环境保持 yes |
| cluster-replica-no-failover no | no | 禁止从节点参与选举 | 专用只读从节点可设为 yes |

- **追问详解**：`cluster-node-timeout` 设置多少合适？
  - 默认 15s 偏保守
  - 内网建议 5s，该值影响故障检测速度
  - 太短 → 网络抖动导致误判 Fail、频繁选举
  - 太长 → 故障发现慢、恢复时间长

- **追问详解**：`cluster-migration-barrier` 的作用
  - 主节点挂了，它的从节点会自动迁移到**没有从节点的主节点**下
  - barrier = N 表示：只有 ≥ N 个从节点的主节点才放行自己的从节点去迁移
  - 防止主节点迁移后自己没有从节点了

### 3.8.2 nodes.conf 文件解析
```
<node_id> <ip:port@cport> <flags> <master_id> <ping_sent> <pong_recv> <config_epoch> <link_state> <slot_range>
```
- 逐字段解释含义
- **追问**：为什么 `nodes.conf` 不要手动编辑？
  - 节点运行时会自动更新，手动修改会被覆盖
  - 所有变更通过 `CLUSTER` 命令完成

## 3.9 常见故障场景与排查

### 3.9.1 cluster state: fail
- 原因：有槽没有节点负责（如主节点挂了，从节点又没来得及接管）
- `cluster-require-full-coverage = yes` 时，整个集群拒绝服务
- 排查：`CLUSTER INFO` → `CLUSTER NODES` → 确认哪些槽没有分配
- **追问**：为什么有这个设计？宁愿全挂也不丢一致性？
  - 是的。部分数据不可用比返回错误数据更安全

### 3.9.2 脑裂（Split Brain）
- 用 Mermaid 画出集群网络分区导致脑裂的场景
- 两半集群各自认为自己持有所有槽 → 双主写同一槽 → 数据冲突
- **追问**：集群脑裂和哨兵脑裂有什么区别？
  - 哨兵脑裂：哨兵集群与主库分区
  - 集群脑裂：业务节点之间分区，更难检测和恢复

### 3.9.3 数据倾斜
- 造成热点槽的三大原因：Hash Tag 滥用、业务 key 分布不均、槽分配不均
- 排查方法：`CLUSTER INFO` → 各节点 `DBSIZE` → `INFO memory` → `redis-cli --bigkeys`
- 解决：业务改造（去掉不必要的 Hash Tag）、拆分热 key（加随机后缀）

### 3.9.4 节点反复上线/下线（Flapping）
- `cluster-node-timeout` 设置太小 + 网络抖动 → 节点被反复标记 Fail → 反复选举
- 解决：调大 `cluster-node-timeout`、排查网络

## 3.10 面试模拟——15个高频追问
请以"面试官问 → 候选人的满分回答"的形式，给出以下问题的深度解答：

1. Redis Cluster 的数据是如何分片的？讲一下哈希槽的原理。
2. 槽的数量为什么是 16384？能不能改成 65535？（提示：不可以，代码写死的）
3. 集群节点之间是怎么通信的？Gossip 协议讲一下。
4. 集群的 Gossip 协议包含了哪些消息？PING 消息里有什么？
5. MOVED 和 ASK 重定向有什么区别？客户端分别如何处理？
6. 集群如何判断一个节点挂了？完整描述 PFail → Fail 的过程。
7. 集群的主从切换过程和哨兵有什么不同？
8. 集群的选举机制是怎样的？为什么从节点要随机等待一段时间才发起选举？
9. Hash Tag 是什么？为什么要用 Hash Tag？带来什么问题？
10. 线上扩容（加节点）和缩容（下节点）怎么操作？数据如何迁移？
11. 集群的 Epoch 机制是什么？有什么用？
12. Redis Cluster 有哪些限制？跨槽事务为什么不能做？
13. 集群脑裂了怎么办？如何尽量减少脑裂的数据丢失？
14. 不用 Redis Cluster，用 Codis / Twemproxy 有什么区别？
15. 讲一下 `cluster-migration-barrier` 和 `cluster-slave-validity-factor` 这两个参数。

## 3.11 集群搭建实战（伪终端演示）
给出一个从零搭建 3主3从 集群的完整命令序列（带注释）：
```bash
# 1. 启动6个Redis实例（端口 6379~6384），配置中开启 cluster-enabled yes

# 2. 创建集群（Redis 5.0+）
redis-cli --cluster create \
  127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 \
  127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 \
  --cluster-replicas 1

# 3. 检查集群状态
redis-cli -p 6379 CLUSTER INFO
redis-cli -p 6379 CLUSTER NODES

# 4. 扩容：加一个新主节点
redis-cli --cluster add-node 127.0.0.1:6385 127.0.0.1:6379

# 5. 迁移槽到新节点
redis-cli --cluster reshard 127.0.0.1:6379 --cluster-from <src_id> --cluster-to <dst_id> --cluster-slots 4096

# 6. 添加从节点
redis-cli --cluster add-node 127.0.0.1:6386 127.0.0.1:6379 --cluster-slave --cluster-master-id <master_id>
```

## 3.12 总结
- 用 Mermaid 画一张「集群全景架构图」：N主N从 + 16384槽分布 + Gossip 通信 + 客户端重定向 + 故障转移
- 一张表格对比：哨兵架构 vs Cluster 架构（选型决策表）
  | 维度 | 哨兵 | 集群 |
  |------|------|------|
  | 数据量 | 单机内存上限 | 所有节点内存之和 |
  | 写性能 | 单机写 QPS | N × 单机写 QPS |
  | 运维复杂度 | 低 | 中 |
  | 事务/Lua | 完全支持 | 单槽内支持 |
  | 适用场景 | 中小规模（< 32GB） | 大规模（> 32GB） |

- 一页纸速查卡：集群核心概念、关键参数、故障排查速查表
```

---

# 第四章：Redis 分布式锁

```markdown
# 角色设定
你是一位资深的Redis内核专家，正在为一位准备冲击大厂（字节/阿里/腾讯）的后端工程师做技术讲解。请用极具深度的方式讲解Redis分布式锁，要求：
- 深入到源码级原理，而非仅停留在"是什么"的层面
- 每个知识点都要回答"为什么要这样设计"
- 使用 Mermaid 图直观展示核心流程
- 穿插大厂面试高频追问
- 语言精炼，不废话，直击要害

---

# 第四章：Redis分布式锁

## 4.1 为什么需要分布式锁

### 4.1.1 从单机锁到分布式锁
- 用 Mermaid 画出「单机部署 → 多实例分布式部署 → 锁失效」的问题演进图
- 单机场景：`synchronized` / `ReentrantLock` / `Mutex` 完全够用
- 分布式场景：多个服务实例竞争**同一份共享资源**（库存扣减、订单号生成、定时任务）
- **追问**：为什么不直接用数据库的行锁做分布式锁？
  - 行锁依赖数据库，高并发下数据库成为瓶颈
  - Redis 基于内存，性能高多个数量级

### 4.1.2 分布式锁的三大核心要求
用表格明确列出：

| 要求 | 含义 | 违反后果 |
|------|------|---------|
| 互斥性 | 同一时刻只有一个客户端持有锁 | 库存超卖 | 
| 防死锁 | 锁一定会被释放（不会因客户端崩溃而永久阻塞） | 业务停滞 |
| 解铃还须系铃人 | 锁只能由加锁的客户端释放 | 锁被他人误删 |

- **追问**：除了这三个基础要求，生产环境还需要什么？
  - 可重入、阻塞/非阻塞获取、锁续期（watchdog）、高性能/低延迟

## 4.2 分布式锁的演进——从青铜到王者

### 4.2.1 第一代：`SETNX` + `DEL`（已淘汰）
```bash
SETNX lock_key 1      # 尝试获取锁
# ... 执行业务逻辑 ...
DEL lock_key           # 释放锁
```
- 用 Mermaid 序列图画出这个流程
- **致命问题1**：没有设置过期时间 → 客户端崩溃 → **死锁**
- **致命问题2**：执行业务过程中锁过期 → 误删其他客户端的锁
- **致命问题3**：`SETNX` 在 Redis 2.6.12 之后已被废弃

### 4.2.2 第二代：`SET key value NX EX seconds`（有缺陷的原子加锁）
```bash
SET lock_key unique_value NX EX 30    # 原子：加锁 + 过期
# ... 执行业务逻辑 ...
if GET lock_key == unique_value:      # 先查后删（非原子）
    DEL lock_key                       # 释放锁
```
- 用 Mermaid 序列图画出流程
- **改进**：解决了死锁问题（有超时）
- **残留问题**：**判断和删除不是原子操作** → "查到我持锁"和"删除锁"之间，锁过期了、被其他客户端获取 → 误删
  - 用 Mermaid 时序图直观展示这个并发竞态问题

- **追问**：为什么用 `unique_value`（UUID / 线程ID）？
  - 防止误删他人的锁。每个客户端的 value 唯一。

### 4.2.3 第三代：Lua 脚本原子释放（单节点够用）
```lua
-- 原子判断 + 删除
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```
- 用 Mermaid 画出 Lua 脚本执行的原子性保障
- **追问**：Lua 脚本在 Redis 中执行的原子性原理是什么？
  - Redis 单线程执行 Lua 脚本，执行期间不会穿插其他命令
  - 但脚本不宜太长，否则会阻塞其他请求

- **还存在的问题**：
  1. **锁过期而业务未完成**：业务执行超过30s → 锁自动释放 → 其他客户端获取锁 → 并发
  2. **单点故障**：Redis 是单节点，挂了锁全失效
  3. **不可重入**：同一客户端无法多次获取同一把锁

### 4.2.4 第四代：Redisson 的分布式锁（工业级）
- Redisson 如何解决上述三大问题：

| 问题 | Redisson 方案 |
|------|-------------|
| 锁过期业务未完成 | **Watchdog 看门狗自动续期** |
| 不可重入 | **Redis Hash + 计数器**实现可重入 |
| 锁释放的非原子性 | **Lua 脚本原子释放** |
| 阻塞获取 | **Redis Pub/Sub + 信号量**实现等待通知 |

## 4.3 Redisson 分布式锁深度剖析

### 4.3.1 可重入锁的数据结构
- 锁的 Redis 存储结构是一个 **Hash**：
  ```
  KEY:  lock_key
  HASH: {
      "<client_uuid>:<thread_id>": 1    ← 重入次数
  }
  ```
- 用 Mermaid ER 图画出锁 key 与 value 的结构关系

### 4.3.2 加锁流程——源码级分析
用 Mermaid 流程图（带泳道：客户端 / Redis）画出完整加锁流程：

1. 客户端构造 Lua 脚本：
   ```lua
   -- 如果锁不存在 或 当前线程已持有锁（可重入）
   if (redis.call('exists', KEYS[1]) == 0) or 
      (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
       redis.call('hincrby', KEYS[1], ARGV[2], 1)     -- 重入次数 +1
       redis.call('pexpire', KEYS[1], ARGV[1])          -- 设置过期
       return nil
   end
   return redis.call('pttl', KEYS[1])                   -- 返回剩余过期时间
   ```
   - 参数：`KEYS[1]` = 锁名, `ARGV[1]` = 过期毫秒, `ARGV[2]` = `UUID:ThreadID`

2. 脚本返回 `nil` → 加锁成功
3. 脚本返回 `pttl` → 加锁失败，返回锁的剩余存活时间

- **关键参数**：加锁成功后，锁的默认过期时间是 **30s**（`lockWatchdogTimeout`）
- **追问**：为什么默认30s？为什么不是10s或60s？

### 4.3.3 Watchdog 看门狗机制——核心亮点
用 Mermaid 时序图重点画出 Watchdog 的工作流程：

```
Client-1                     Redis
   │                           │
   │─── 加锁成功，expire=30s ──→│
   │                           │
   │ ←── 启动Watchdog定时器 ───│  (每 10s = 30s/3 执行一次)
   │                           │
   │─── (10s后) 续期到30s ────→│  PEXPIRE lock_key 30000
   │                           │
   │─── (20s后) 续期到30s ────→│  PEXPIRE lock_key 30000
   │                           │
   │─── 业务完成，主动释放锁 ────→│
   │─── 关闭Watchdog ──────────→│
```

- **追问详解**：
  - Watchdog 的续期间隔：`lockWatchdogTimeout / 3`（默认 10s）
  - Watchdog 通过 Netty 的 **HashedWheelTimer** 实现定时任务
  - 如果业务完成但 Watchdog 没来得及关闭怎么办？→ 主动释放锁时会取消定时任务
  - 如果客户端进程挂了，Watchdog 失效 → 30s 后锁自动释放 → 不会死锁

- **追问**：如果显式指定了锁的过期时间（`lock.lock(5, TimeUnit.SECONDS)`），Watchdog 还会工作吗？
  - **不会！** Watchdog 只在**未指定 leaseTime** 时启用
  - 这是设计决策：用户指定过期时间 → 用户自己保证业务在时间内完成

### 4.3.4 解锁流程——源码级分析
用 Mermaid 流程图画出解锁流程：

```lua
-- 释放锁的 Lua 脚本
if (redis.call('hexists', KEYS[1], ARGV[2]) == 0) then
    return nil  -- 锁不属于当前线程
end
local counter = redis.call('hincrby', KEYS[1], ARGV[2], -1)
if (counter > 0) then
    redis.call('pexpire', KEYS[1], ARGV[1])   -- 重入次数 > 0，只减计数不删锁
    return 0
else
    redis.call('del', KEYS[1])                  -- 重入次数 = 0，删除锁
    redis.call('publish', KEYS[2], ARGV[3])     -- 通知等待队列
    return 1
end
```
- 参数：`KEYS[1]` = 锁名, `KEYS[2]` = 频道名, `ARGV[1]` = 过期毫秒, `ARGV[2]` = `UUID:ThreadID`, `ARGV[3]` = 解锁消息

- **追问**：`publish` 的作用是什么？
  - 通知阻塞等待的线程「锁已释放，可以来竞争了」
  - 配合 Redis Pub/Sub 实现**锁的等待-通知机制**，避免轮询自旋

### 4.3.5 阻塞等待与重试机制
用 Mermaid 时序图画出「Client-1 持有锁，Client-2 阻塞等待」的完整过程：

1. Client-2 尝试加锁 → Redis 返回锁的剩余 ttl
2. Client-2 订阅 `redisson_lock__channel:{lock_key}` 频道
3. Client-2 使用信号量（`Semaphore`）阻塞等待，最大等待时间 = ttl
4. Client-1 释放锁 → Redis Publish 解锁消息
5. Client-2 收到消息 → 重新尝试加锁 → 成功
6. Client-2 取消订阅

- **追问**：如果锁提前释放了，Client-2 还被 `Semaphore` 阻塞着怎么办？
  - `Semaphore.tryAcquire(ttl, TimeUnit.MILLISECONDS)` 设置了超时
  - 超时后自动醒来重新尝试

- **追问**：多个客户端同时等待同一把锁，谁先拿到？
  - **非公平锁（默认）**：Pub/Sub 通知所有等待者，各凭本事竞争
  - 可能导致「惊群效应」，但 Redisson 通过信号量的超时错峰降低了影响

### 4.3.6 公平锁 vs 非公平锁
| 维度 | 非公平锁（默认） | 公平锁 |
|------|----------------|--------|
| 获取顺序 | 不保证顺序 | 先到先得 |
| 性能 | 高 | 较低（需要维护等待队列） |
| 实现 | Pub/Sub 广播通知 | Redis List + 排队 |
| 惊群效应 | 有 | 无 |
| 适用场景 | 大部分场景 | 需要严格按序处理 |

## 4.4 Redlock 算法——多节点 Redis 分布式锁

### 4.4.1 为什么需要 Redlock
- 单节点 Redis 的问题：主节点宕机，从节点提升，**锁数据可能丢失**（异步复制）
- 用 Mermaid 画出这个锁丢失的时序：
  1. Client-A 在 Master 上获取锁
  2. Master 宕机，锁数据还未同步到 Slave
  3. Slave 提升为新 Master（锁信息丢失）
  4. Client-B 在新 Master 上获取**同一把锁** → **互斥性被打破** → 并发

### 4.4.2 Redlock 算法流程
用 Mermaid 序列图画出 Redlock 在 5 个独立 Redis 节点上的加锁流程：

```
Client                  Redis-1  Redis-2  Redis-3  Redis-4  Redis-5
  │                        │        │        │        │        │
  │─── 获取当前时间 T1 ────│        │        │        │        │
  │                        │        │        │        │        │
  │── SET NX EX (并发) ───→│───────→│───────→│───────→│───────→│
  │←────────── OK ─────────│        │        │        │        │
  │←────────── OK ─────────────────│        │        │        │
  │←────────── OK ─────────────────────────│        │        │
  │←─────── 超时/NOK ──────────────────────────────│        │
  │←─────── 超时/NOK ───────────────────────────────────────│
  │                        │        │        │        │        │
  │─── 获取当前时间 T2 ────│        │        │        │        │
  │                        │        │        │        │        │
  │  判断：(T2-T1) < TTL   │        │        │        │        │
  │  且 成功数 ≥ N/2+1 (3) │        │        │        │        │
  │                        │        │        │        │        │
  │  → 加锁成功！           │        │        │        │        │
  │  锁有效时长 = TTL - (T2-T1) │    │        │        │        │
```

- **关键点**：
  1. N 个独立 Redis 节点（通常 5 个），**没有主从关系**
  2. 客户端**并发**向所有节点发起加锁请求
  3. 加锁成功的判定：**成功数 ≥ N/2+1**（过半数）
  4. 总耗时 < 锁的有效期（TTL）
  5. 锁的实际有效时长 = TTL - (T2 - T1)
  6. 加锁失败 → 向所有节点（包括已成功的）发送释放命令

- **追问**：为什么 Redlock 要求 5 个节点？3 个不行吗？
  - N = 2f+1，可以容忍 f 个节点故障
  - 5 个节点可容忍 2 个故障，3 个只能容忍 1 个
  - 3 个理论可行，但容错能力弱

### 4.4.3 Redlock 续期问题
- Redlock 没有内置 Watchdog → 必须在加锁时确定 TTL
- 如果业务超过 TTL，锁自动释放 → **无续期机制**
- **追问**：Redlock 能做续期吗？怎么做？
  - 可以手动实现：业务线程定期续期，但复杂度高
  - 这就是为什么 Redisson 的 RedLock 实现中**也支持 Watchdog**

### 4.4.4 Redlock 的争议——Martin Kleppmann 的批评
- 用 Mermaid 时序图还原争议场景：
  1. Client-A 在 3/5 节点上加锁成功
  2. Client-A 因 GC 停顿（STW），锁过期但 Client-A 不知情
  3. Client-B 在锁过期后成功获取锁
  4. Client-A 从 GC 中恢复，继续执行业务 → **两个客户端同时认为自己持有锁**

- **Martin 的观点**：分布式锁不能依赖时间，需要 fencing token（令牌）
- **Antirez 的反驳**：GC 停顿是极端情况，绝大多数业务场景下 Redlock 足够安全

- **追问**：你在大厂项目中怎么选？Redlock 到底安不安全？
  - 答：看场景。库存扣减 → Redlock 足够；银行转账 → 需要 fencing token 或直接上 TCC/事务

### 4.4.5 Redisson 的 RedLock 实现
```java
// Redisson 多锁 / RedLock
RLock lock1 = redisson1.getLock("lock");
RLock lock2 = redisson2.getLock("lock");
RLock lock3 = redisson3.getLock("lock");
RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
redLock.lock();  // 内部就是 Redlock 算法
```
- **追问**：RedissonRedLock 和普通的 `RLock` 有什么不同？
  - RLock 是单节点锁 + Watchdog
  - RedissonRedLock 是**多独立节点** + Redlock 算法 + Watchdog
  - RedissonRedLock 已经内置续期

## 4.5 分布式锁的典型应用场景

用 Mermaid 架构图展示以下 4 个场景中分布式锁的位置：

| 场景 | 说明 | 注意事项 |
|------|------|---------|
| 秒杀/库存扣减 | 防止超卖 | 锁粒度要细（按 SKU 加锁） |
| 订单号生成 | 防止重复单号 | 优先用雪花算法等无锁方案 |
| 定时任务 | 多实例互斥执行 | 配合 XXL-Job / SchedulerX |
| 幂等性保障 | 用锁+业务唯一键做幂等 | 锁的 TTL 要大于业务执行时间 |

## 4.6 分布式锁的常见问题与踩坑

### 4.6.1 锁过期了，业务还没执行完
- 场景：执行了 40s，锁 30s 过期 → 其他线程拿到锁
- 方案1（Redisson）：Watchdog 自动续期
- 方案2（自己实现）：启动守护线程定期续期
- 方案3：预估好 TTL，留足安全边际

### 4.6.2 锁被「误删」
- 场景：A 的锁过期 → B 拿到锁 → A 释放（删了 B 的锁）
- 解决：必用 Lua 脚本，比对 `unique_value` 后再删

### 4.6.3 Redis 主从切换导致锁丢失
- 场景已在 4.4.1 描述
- 解决：Redlock 多节点 or 容忍偶发的并发（业务层做幂等兜底）

### 4.6.4 可重入性问题
- 线程A加锁后，同一线程内的方法B也要加同一把锁
- Redisson 用 Hash + 计数器天然支持
- 自己用 Lua 实现的版本注意 Hash value 用 `threadId` 做判断

### 4.6.5 锁的性能问题
- 加锁/释放都会有一次网络 RTT（~1ms）
- **追问**：高并发下怎么减少锁的开销？
  - 减小锁粒度（一个 SKU 一把锁 > 一把全局锁）
  - 无锁化替代（CAS、原子操作、本地缓存 + 异步同步）

### 4.6.6 时间漂移（Clock Drift）
- 影响 Redlock 的安全保证
- Redis 节点时钟不同步 → 锁的超时判断不准
- 生产建议：所有 Redis 节点启用 NTP 时间同步

## 4.7 分布式锁方案对比

| 方案 | 实现复杂度 | 性能 | 可靠性 | 适合场景 |
|------|----------|------|--------|---------|
| Redis SET NX + Lua | 低 | 极高 | 中（单点） | 一般业务 |
| Redisson RLock | 低 | 极高 | 中（主从可能丢） | 企业级业务 |
| RedLock（5节点） | 中 | 高 | 高 | 高可靠性要求 |
| Zookeeper 临时顺序节点 | 中 | 低 | 极高 | 强一致性要求 |
| etcd 分布式锁 | 中 | 低 | 极高 | 强一致性要求 |
| 数据库乐观锁 | 低 | 低 | 极高 | 低频操作 |

- **追问**：Zookeeper 实现分布式锁的原理是什么？和 Redis 的核心区别是什么？
  - 答：ZK 用临时顺序节点 + Watch 机制，CP 系统，保证强一致性
  - Redis 是 AP 系统，优先保证可用性和性能
  - 用 Mermaid 画出 ZK 分布式锁的临时节点排队机制

## 4.8 面试模拟——15个高频追问
请以"面试官问 → 候选人的满分回答"的形式，给出以下问题的深度解答：

1. 说一下你用 Redis 实现分布式锁的完整方案。
2. 为什么 SETNX 不能直接用了？锁的超时时间怎么设置？
3. 锁的过期时间到了但业务还没执行完，怎么办？（Watchdog 原理）
4. 怎么保证释放锁的原子性？（Lua 脚本）
5. 可重入锁是怎么实现的？（Redis Hash 数据结构）
6. 讲一下 Redlock 算法的原理和流程。
7. Redlock 有什么缺陷？Martin Kleppmann 的批评是什么？
8. 分布式锁和数据库乐观锁 / Zookeeper 锁有什么区别？怎么选型？
9. 主从切换导致锁丢失怎么办？
10. 多个客户端同时等锁，Redisson 是怎么通知的？（Pub/Sub + Semaphore）
11. 公平锁和非公平锁的区别？怎么实现公平锁？
12. 高并发场景下，分布式锁成为瓶颈怎么办？
13. 如果让你从零设计一个分布式锁，你会考虑哪些问题？
14. Watchdog 的续期间隔为什么是 `lockWatchdogTimeout / 3`？
15. Redisson 的锁在 Redis 集群模式下能用吗？有什么限制？

## 4.9 生产级代码示例
给出一个使用 Redisson 的完整生产级分布式锁示例（Java）：

```java
public String deductStock(String skuId, int quantity) {
    String lockKey = "lock:stock:" + skuId;  // 细粒度锁
    RLock lock = redissonClient.getLock(lockKey);
    
    try {
        // tryLock(等待时间, 锁过期时间, 时间单位)
        // 指定 leaseTime → Watchdog 不启用
        if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
            // 1. 查库存
            int stock = stockMapper.getStock(skuId);
            if (stock < quantity) {
                return "库存不足";
            }
            // 2. 扣减库存
            stockMapper.deductStock(skuId, quantity);
            return "扣减成功";
        } else {
            return "系统繁忙，请稍后重试";
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return "系统异常";
    } finally {
        // 只有自己持有锁才释放
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

- **追问**：这里指定了 `leaseTime = 10s`，Watchdog 还工作吗？如果业务需要 15s 怎么办？

## 4.10 总结
- 用 Mermaid 画出一张「分布式锁全景决策树」
  ```
  需要分布式锁？
  ├── 单节点够用 → SET + Lua 原子释放（自己实现）
  ├── 需要可重入/自动续期 → Redisson RLock
  ├── 需要高可靠性 → Redlock 多节点
  └── 需要强一致性 → ZK / etcd
  ```
- 一页纸速查卡：Redisson RLock 核心参数、加锁/解锁 Lua 脚本速查、常见踩坑清单
```

---

# 附录：四章全景总览

| 章节 | 主题 | 核心深度点 |
|------|------|-----------|
| 第一章 | 主从复制 | PSYNC全量/部分复制、replication backlog、COW机制、buffer溢出、复制风暴 |
| 第二章 | 哨兵 | SDOWN/ODOWN两阶段判定、Raft-like选举、epoch机制、脑裂、Quorum vs Majority |
| 第三章 | 分片集群 | 16384哈希槽、Gossip协议5种消息、MOVED vs ASK、PFail→Fail、槽迁移、epoch |
| 第四章 | 分布式锁 | SETNX演进史、Redisson Watchdog、可重入Hash结构、Redlock算法、Martin争议 |
