# Redis 安全与 ACL 控制 —— 深度详解

---

## 前言：Redis 安全问题的历史背景

Redis 最初的设计理念是"信任的运行环境"——默认没有密码、所有用户拥有全部权限——被设计运行在受信网络内，不暴露在公网上。Redis 作者 antirez 甚至一度认为："安全应该由防火墙和运维层来保证，Redis 不需要内置复杂的安全机制。"

但现实不是理想国。2015 年前后，互联网上出现了大规模利用 Redis 未授权访问漏洞的蠕虫攻击——攻击者扫描公网 6379 端口，发现没有密码的 Redis 实例直接执行 `FLUSHALL` 清空数据，或通过 `CONFIG SET dir /var/spool/cron` 写入 SSH 公钥获取服务器权限。一时之间，无数暴露在公网的 Redis 沦陷。

这次安全事件推动了 Redis 安全机制的大规模改进。Redis 6.0 引入了完整的 **ACL（Access Control List）** 系统——这是一个和操作系统 ACL 类似、基于"用户+权限"的细粒度访问控制机制。

---

## 第一章：基础认证——requirepass

### 1.1 最简单的密码保护

```bash
# redis.conf
requirepass "your_strong_password_here"

# 客户端连接后必须 AUTH
127.0.0.1:6379> PING
(error) NOAUTH Authentication required.

127.0.0.1:6379> AUTH your_strong_password_here
OK

127.0.0.1:6379> PING
PONG
```

`requirepass` 是 Redis 第一道也是最基础的安全防线。它设了一个"全局密码"——所有客户端在连接 Redis 之后、发送任何命令之前，都需要通过 `AUTH <密码>` 验证身份。

### 1.2 requirepass 的局限性

但这种"全局密码"方式有三个明显的不足：

**问题一：所有客户端共用同一个密码**

一个 Redis 实例可能服务多个应用——订单系统、用户系统、推荐系统共享同一个 Redis。如果这些应用使用同一个 requirepass 密码——任何一个应用的密码泄露了，整个 Redis 对所有应用完全敞开。无法做到"订单系统只能访问 `order:*`"——这在传统数据库（MySQL 的用户权限）中是基本操作。

**问题二：无法限制"能执行哪些命令"**

requirepass 只做一个判断："有密码吗？"但无法控制"这个客户端能执行哪些命令"。比如：一个数据分析的应用只需要读取数据——理论上应该只能执行 GET、HGET 等读命令——不能执行 DEL、FLUSHALL。但 requirepass 认证通过后，所有命令都可以执行。

**问题三：密码是全局的，无法分权**

`CONFIG SET requirepass` 是最高权限操作——你拿到了这个密码就等于拥有了 Redis 的全部控制权。无法实现"部分管理员只能查看 INFO，不能修改配置"的细粒度权限控制。

---

## 第二章：ACL —— 细粒度访问控制

### 2.1 ACL 的基本概念

Redis 6.0 的 ACL 系统允许管理员创建多个"用户"，每个用户拥有独立的密码、独立的命令权限、独立的 key 访问权限。这和操作系统的用户权限管理非常类似：

```
Redis ACL 的核心概念：

用户（User）= 一个独立的身份标识
  ├── 密码：一个或多个登录密码
  ├── 命令权限：允许/禁止执行哪些命令（精细到单个命令级别）
  ├── key 权限：允许/禁止访问哪些模式的 key
  ├── 通道权限：允许/禁止订阅哪些 Pub/Sub 通道
  └── 是否启用：可以临时禁用一个用户而不删除它的配置
```

Redis ACL 有一系列命令来管理用户和权限：

```bash
# 查看所有用户
ACL LIST

# 查看当前用户的权限
ACL WHOAMI

# 创建一个新用户（或修改已有用户）
ACL SETUSER <用户名> [规则...]

# 删除一个用户
ACL DELUSER <用户名>

# 查看一个用户的详细权限
ACL GETUSER <用户名>

# 以另一个用户的身份执行命令（用于测试权限）
AUTH <用户名> <密码>

# 加载/保存 ACL 配置
ACL LOAD       # 从外部 aclfile 加载
ACL SAVE       # 保存到外部 aclfile
```

### 2.2 创建用户——ACL SETUSER

ACL 的核心命令是 `ACL SETUSER`。它的语法由一系列"规则"组成——每个规则描述"允许什么"或"禁止什么"：

```bash
# 创建一个"只读用户"的完整示例
ACL SETUSER reader_user                          \  # 用户名
          on                                     \  # 启用用户（off=禁用）
          >password123                           \  # 设置密码为 "password123"
          ~cache:*                               \  # 只允许访问以 "cache:" 开头的 key
          +@read                                 \  # 允许所有读命令
          -@write                                \  # 禁止所有写命令
          -@dangerous                             \  # 禁止危险命令组
          +ping                                  \  # 允许 PING
          +info                                   \  # 允许 INFO
```

ACL 规则的构成：

```
命令权限规则：
  +<命令>      → 允许某个命令（如 +GET）
  -<命令>      → 禁止某个命令（如 -DEL）
  +@<类别>     → 允许某个命令类别（如 +@read 允许所有读命令）
  -@<类别>     → 禁止某个命令类别（如 -@write 禁止所有写命令）
  +@all        → 允许所有命令
  allcommands  → 同上（别名）
  nocommands   → 禁止所有命令（白板，等后续规则授权）

Key 权限规则：
  ~<模式>      → 允许访问匹配该模式的 key（如 ~cache:*）
  %R~<模式>    → 只允许读取匹配该模式的 key
  %W~<模式>    → 只允许写入匹配该模式的 key
  allkeys      → 允许访问所有 key
  resetkeys    → 清除之前的 key 权限（重新开始）

密码规则：
  >密码        → 添加登录密码
  <密码        → 删除密码
  #密码哈希    → 添加密码（哈希格式，在配置文件中使用）
  nopass       → 不需要密码
  resetpass    → 清除所有密码

用户规则：
  on           → 启用用户
  off          → 禁用用户
```

### 2.3 命令类别——Redis 预置的 21 个命令组

Redis 把 200 多个命令预分成了 21 个类别——方便你通过 `+@组名` 一次性管理。常用的组名如下：

| 类别 | 包含的命令 | 典型用途 |
|------|----------|---------|
| `@read` | GET, HGET, LRANGE, SMEMBERS... | 只读用户 |
| `@write` | SET, DEL, LPUSH, SADD... | 只写用户 |
| `@admin` | CONFIG, SHUTDOWN, DEBUG... | 管理员 |
| `@dangerous` | FLUSHALL, KEYS, SHUTDOWN... | 需要限制的危险命令 |
| `@keyspace` | DEL, EXISTS, EXPIRE, RENAME... | 影响 key 的命令 |
| `@string` | GET, SET, INCR, APPEND... | String 类型操作 |
| `@hash` | HGET, HSET, HGETALL... | Hash 类型操作 |
| `@list` | LPUSH, RPOP, LRANGE... | List 类型操作 |
| `@set` | SADD, SMEMBERS, SINTER... | Set 类型操作 |
| `@sortedset` | ZADD, ZRANGE, ZSCORE... | ZSet 类型操作 |
| `@pubsub` | PUBLISH, SUBSCRIBE... | Pub/Sub |
| `@slow` | SAVE, BGSAVE, FLUSHALL... | 慢（可能阻塞主线程） |
| `@fast` | GET, SET, HGET... | 快（微秒级） |
| `@connection` | AUTH, PING, QUIT... | 连接管理 |
| `@stream` | XADD, XREAD, XGROUP... | Stream 类型 |

你还可以创建自己的"命令选择器"（Selector）——通过 `ACL SETUSER` 加 `+选择器名`——不限于这 21 个预置组。

### 2.4 完整示例——为三个不同的应用建三个用户

```bash
# ===== 业务应用用户（只读 + 特定前缀写入） =====
ACL SETUSER app_order_service                     \
          on                                      \
          >secure_password_123                    \
          ~order:*                                \   # 只能访问 order:* 的 key
          +@read                                  \   # 允许所有读命令
          +@write                                 \   # 允许所有写命令
          +@hash +@string +@list +@set +@sortedset \
          -FLUSHALL -FLUSHDB                      \   # 禁止清空
          -CONFIG -SHUTDOWN -DEBUG                 \   # 禁止管理命令
          -KEYS                                    \   # 禁止 KEYS（用 SCAN 代替）


# ===== 数据分析用户（纯只读） =====
ACL SETUSER data_analyst                          \
          on                                      \
          >analyst_pass                           \
          ~*                                       \   # 可以访问所有 key
          +@read                                  \   # 只允许读命令
          -@write -@dangerous -@admin             \   # 禁止写、危险和管理


# ===== 管理员用户（全权限） =====
ACL SETUSER redis_admin                           \
          on                                      \
          >admin_very_strong_password             \
          ~*                                       \   # 所有 key
          +@all                                    \   # 所有命令
          
# 验证
redis-cli -a secure_password_123 --user app_order_service
# 连接后只能看到 order:* 的 key
```

### 2.5 ACL 文件的持久化

ACL 配置有两种存储方式：

**方式一：写在 redis.conf 中**

```bash
# redis.conf
aclfile /etc/redis/users.acl
# 指定外部 ACL 文件路径（不配置则用 redis.conf 中的 ACL 规则）
```

**方式二：独立 ACL 文件**

```bash
# /etc/redis/users.acl
user default off
user app_order_service on >secure_password_123 ~order:* +@all -@admin -@dangerous
user data_analyst on >analyst_pass ~* +@read -@write -@dangerous
user redis_admin on >admin_very_strong_password ~* +@all
```

通过 ACL 文件管理可以把 ACL 配置和 Redis 主配置文件分开——更新 ACL 时只需要 `ACL LOAD` 重新加载，不需要重启 Redis。

### 2.6 default 用户——改掉这个"默认全权用户"

Redis 启动时有一个内置的 `default` 用户。这个用户的权限是 `+@all ~*`——所有命令、所有 key——相当于 `requirepass` 时代的"密码保护针对的就是这个 default 用户"。

如果启用了 ACL 但没有为 default 配置密码——default 用户依然是"无需密码即可登录且拥有全部权限"——任何从本机连接过来的人默认就是 default。所以启用 ACL 后的第一个安全问题就是：给 default 用户设密码，或者直接禁用 default 用户：

```bash
# 方案 A：给 default 设密码
ACL SETUSER default on >default_pass ~* +@all

# 方案 B：直接禁用 default（推荐——所有连接必须指名用户）
ACL SETUSER default off
```

---

## 第三章：生产安全加固全面指南

### 3.1 网络层——防火墙 + 绑定接口

```bash
# redis.conf
bind 127.0.0.1 192.168.1.100    # 只监听这些网络接口（不是"白名单"！）

# bind 不是 IP 白名单——bind 表示"Redis 只监听指定的网络接口"
# 比如 Redis 部署在 192.168.1.100 这台机器上
# bind 127.0.0.1 192.168.1.100 → 只接受本机和内网其他机器的连接
# 公网 IP 无法连接

# 加强：通过 iptables 只允许特定机器访问 6379
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 6379 -j ACCEPT
iptables -A INPUT -p tcp --dport 6379 -j DROP
```

### 3.2 命令层——禁用危险命令

即使不用 ACL，也可以通过 `rename-command` 把危险命令重命名为"无法被执行的随机字符串"：

```bash
# redis.conf
rename-command FLUSHALL ""              # 完全禁用（改为空字符串）
rename-command FLUSHDB   ""
rename-command KEYS      ""
rename-command CONFIG    "CONFIG_d0a8f3b2"  # 重命名为随机字符串（只有管理员知道）
rename-command SHUTDOWN  ""
rename-command DEBUG     ""
rename-command SAVE      ""

# 但是！rename-command 对所有客户端生效——管理员也执行不了 CONFIG 了
# 所以 rename-command 的限制粒度不如 ACL
```

### 3.3 运行环境层——最小权限

```bash
# 不要用 root 用户运行 Redis！
# 创建一个专门的 redis 用户
useradd -r -s /bin/false redis
# -r = 创建系统用户, -s /bin/false = 无登录 shell

# 保护配置文件（只有 redis 用户可读——密码在其中）
chown redis:redis /etc/redis/redis.conf
chmod 600 /etc/redis/redis.conf

# 保护数据文件目录
chown redis:redis /var/lib/redis
chmod 700 /var/lib/redis
```

### 3.4 连接层——TLS 加密（Redis 6.0+）

```bash
# redis.conf
port 0                              # 关闭非 TLS 端口
tls-port 6379                       # 开启 TLS 端口
tls-cert-file /path/to/redis.crt
tls-key-file  /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt

# 客户端连接
redis-cli --tls --cert /path/to/client.crt --key /path/to/client.key
```

---

## ⭐️ 面试题汇总

**Q1: Redis 默认没有密码，怎么防止未授权访问？**

> ① `requirepass` 设密码（最基本的防线）；② `bind` 限制监听的网络接口（不是 IP 白名单）；③ iptables 防火墙限制访问来源；④ 不要用 root 用户运行 Redis；⑤ 如果用 Redis 6.0+，启用 ACL 系统，为不同应用创建独立的用户和权限。

**Q2: ACL 和 requirepass 的本质区别是什么？**

> requirepass 是"全局密码"——认证通过后可以执行任何命令、访问任何 key。ACL 是"细粒度权限"——可以创建多个用户，每个用户有独立的密码、独立的命令权限（能执行哪些命令）、独立的 key 权限（能访问哪些 key）。ACL 允许一个 Redis 实例安全地服务多个应用。

**Q3: 如何禁止一个用户执行 FLUSHALL 但允许他执行 SET 和 GET？**

> 用 ACL 的命令权限规则：`-FLUSHALL +@read +@write`——显式禁用 FLUSHALL，启用读写命令。或更细粒度：`+SET +GET -FLUSHALL`——只允许 SET 和 GET，其余都不行。

**Q4: 生产环境建议的安全加固有哪些？**

> ① 开启 ACL 或 requirepass；② 重命名/禁用危险命令（FLUSHALL、KEYS、CONFIG、SHUTDOWN）；③ 限制网络访问（bind + iptables）；④ 以非 root 用户运行；⑤ 保护配置文件和 ACL 文件权限；⑥ 开启 AOF/RDB 持久化防数据丢失。

---

*Created: 2026-05-26 | Category: 22-Redis安全与ACL控制*
