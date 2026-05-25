# Redis 五种基础数据类型 —— Java 程序员视角的深度详解

---

## 前言：本文的使用方式

Redis 源码是用 C 语言写的。本文假设你 **只学过 Java，没学过 C**。所以每一个 C 语言概念出现时，都会先用 Java 类比讲清楚，再讲它在 Redis 中怎么用的。

读完你会理解：
- 为什么 " String 超过 44 字节就不一样了 "
- 为什么 Redis 用跳表而不用 Java 的 TreeMap（红黑树）
- 为什么 Hash 的 rehash 不会一次性做完
- 为什么 List 比 Java 的 LinkedList 省内存得多

---

## 第〇章：C 语言基础知识（Java 程序员视角）

在看 Redis 内部实现之前，先把 C 语言中几个绕不开的概念搞清楚。**不需要精通 C，理解类比即可。**

### 0.1 malloc：C 语言的 `new`

```java
// Java：创建一个对象
User u = new User("张三");
//  JVM 自动在堆上分配内存，自动 GC 回收
```

```c
// C：分配一块内存
void* ptr = malloc(64);
//  向操作系统要 64 字节的堆内存
//  malloc 返回这 64 字节的「起始地址」
//  C 没有 GC！用完了必须手动 free(ptr) 归还内存
```

**简单理解：`malloc(大小)` = Java 的 `new byte[大小]`。** 本质都是从堆（Heap）上分配一块内存，返回这块内存的地址。

那么问题来了——**什么是"地址"？**

### 0.2 指针（Pointer）：就是 Java 的"引用"

```java
// Java：
User u = new User();    // u 是一个引用，指向堆中 User 对象的「地址」
u.getName();            // 通过引用访问对象的字段
```

```c
// C：
User* u = malloc(sizeof(User));  // u 是一个指针，存的是「内存地址」
u->name;                         // 通过指针访问字段（-> 等价于 Java 的 .）
```

**Java 的引用 = C 的指针。** 本质上都是存一个内存地址。

差别在于：
- Java 的引用你只能访问对象的字段/方法，不能做地址运算
- C 的指针你可以直接操作地址（+1、-1），这也是它强大和危险的地方

**在本文中，你看到"ptr"、"pointer"、"指针"——一律翻译成 Java 的"引用"即可。**

Redis 源码中的 `void *ptr` 就相当于 Java 的 `Object ref`——一个"指向某种东西的引用，具体类型不定"。

### 0.3 struct：C 语言的"只有属性的 Class"

```java
// Java：一个类
class User {
    String name;   // 字符串引用，指向堆中的 String 对象
    int age;       // 存的是值本身（基本类型）
}
```

```c
// C：一个结构体（struct）
struct User {
    char* name;    // 字符串指针，指向堆中的字符串
    int age;       // 存的是值本身
};
```

**struct = 只有属性没有方法的 Java class。** C 不是面向对象的，所以没有方法/继承/多态。struct 就是把一组数据打包成一个类型。

**sizeof(struct)**：编译器自动计算一个 struct 占多少字节。比如上面的 `struct User`：`char*` 占 8 字节（64 位系统）+ `int` 占 4 字节 = 12 字节（实际会因为内存对齐变成 16 字节，后面讲）。

### 0.4 sizeof：测量一个类型占用多少字节

```java
// Java 没有 sizeof 这个概念。但你潜意识里知道：
// int    = 4 字节，范围 -21亿 ~ 21亿
// long   = 8 字节，范围 -9e18 ~ 9e18
// 一个对象引用 = 4 字节（32位JVM，压缩指针）或 8 字节（64位JVM）
// Java 会自动管理，你不需要关心
```

```c
// C 中你需要精确知道每个类型占多少字节：
sizeof(char)   = 1    // 1 字节（够存一个字符或 0-255 小整数）
sizeof(int)    = 4    // 4 字节
sizeof(long)   = 8    // 8 字节（64位系统）
sizeof(void*)  = 8    // 指针（地址值）占 8 字节（64位系统）
```

**在本文中，字节数非常重要**——Redis 追求极致内存利用率，每一个字节都要精打细算。比如为什么 44 字节是分界线？因为 64 字节的分配块，减去必要结构 20 字节 = 44 字节可用。

### 0.5 内存分配器（jemalloc）：C 语言的"堆内存管家"

**背景知识：**

```
Java 程序员眼中的"分配对象"：
  new User()  →  JVM 堆分配  →  GC 自动回收
  你不需要关心：这块内存是堆里的哪个位置、多大、什么时候回收

C 程序员眼中的"分配内存"：
  malloc(64)  →  操作系统分配  →  必须手动 free()
  你必须在乎：malloc 背后是谁在管理堆？jemalloc / glibc malloc / tcmalloc？
```

**jemalloc 是什么？**

当你调用 `malloc(64)` 时，不是直接向操作系统要内存（那样太慢），而是由 **内存分配器** 从预先申请的大块内存中切一块给你。jemalloc 就是其中一种分配器。

**Redis 为什么默认用 jemalloc？**
- jemalloc 的分配块大小是 **固定的几个档次**：8, 16, 32, 64, 128, 256, 512, 1024 ...（按 2^n 增长）
- 你申请 50 字节 → jemalloc 给你一个 64 字节的块（多出 14 字节暂时浪费，但分配极快）
- 你申请 70 字节 → jemalloc 给你一个 128 字节的块

**这与 Redis 的 embstr 分界线直接相关**——后面会详细推导。

**Java 中也有类似的机制！** JVM 的 TLAB（Thread Local Allocation Buffer）也是预先批量分配内存，只不过你不会直接接触到。

### 0.6 柔性数组（Flexible Array）：变长数组的 C 语言方案

Java 有 `ArrayList`——自动扩容的变长数组。C 语言没有，但有一个巧妙的技巧叫 **柔性数组（flexible array member）**：

```c
// C 语言 struct 的最后一个字段可以是不写长度的数组：char buf[];
struct sdshdr {
    int len;      // 固定 4 字节
    int free;     // 固定 4 字节
    char buf[];   // 柔性数组——不占 struct 本身的空间，
};                // 实际数据紧跟 struct 之后存放

// sizeof(sdshdr) = 8（只算 len + free，buf 不占空间！）

// 使用方式：
// 要存 "hello" (5字节)，分配 sdshdr + 6 字节一块连续内存：
struct sdshdr* s = malloc(sizeof(struct sdshdr) + 6);
// s->buf 自动指向 struct 之后的 6 字节空间
```

**内存布局（一次 malloc 的结果）：**

```
malloc(sizeof(sdshdr) + 6)  →  连续 14 字节：
┌──────┬──────┬──────────────────┐
│ len  │ free │ buf[0..5]       │
│ 4字节│ 4字节 │ 6字节(存"hello\0")│
└──────┴──────┴──────────────────┘
       ← sdshdr →
              ← buf 紧跟其后 →
```

**类比 Java：** 可以理解为一个特殊的 class，其中最后一个字段是可变大小的 `byte[]`，而且这个 byte[] 的数据和对象头是连续存放的（Java 做不到这点，Java 对象的数组字段是独立的引用指向独立的数组对象）。

这个技巧在 Redis 中大量使用——**它让 "元数据" 和 "实际数据" 在同一块连续内存中，减少 malloc 次数，提升 CPU 缓存友好度。**

### 0.7 内存对齐：为什么结构体实际比肉眼算的大

```
你肉眼算：
  struct { char a; int b; } → 1 + 4 = 5 字节

编译器实际算：= 8 字节！
  ┌──────┬──────┬────────────────┐
  │  a   │ pad  │      b         │
  │ 1字节│3字节空│    4字节       │
  └──────┴──────┴────────────────┘

原因：CPU 一次读 4 或 8 字节，int 必须放在 4 的倍数地址上才能一次读完。
      编译器自动插入「填充字节(padding)」让 b 对齐到 4 字节边界。
```

**Java 同样存在内存对齐**，只是 JVM 帮你做了你不需要关心。Redis 追求极致紧凑，会使用 `__attribute__((packed))` 等手段关闭对齐，节省每一字节。

### 0.8 二进制安全：Redis 凭什么能存图片的字节流？

**C 语言的"字符串"其实是个谎言：**

```c
char* str = "hello";
printf("%s", str);   // 打印到 \0 为止
// C 认为：字符串 = 从起始地址开始，一直到遇到 \0（值为0的字节）为止。
```

致命问题：如果字符串内容本身就包含 `\0`（比如图片的二进制数据）→ C 会提前截断。

**Redis 的 SDS 怎么解决的？**

```c
struct sdshdr {
    int len;      // ← "用了多少字节"用 len 记录，不依赖 \0
    char buf[];   // ← 数据中可以有任意字节，包括 \0
};
// 判断字符串结束：看 len，不是看 \0！
```

**Java 程序员完全可以跳过这个问题**——Java 的 String 底层是 `char[]` 或 `byte[]`，长度由数组的 `.length` 决定，天然就是二进制安全的。**C 不是，所以 Redis 需要自研 SDS。**

---

## 第一章：String（字符串）—— Redis 万物之基

### 1.1 高层视角：String 就是 Redis 版的 `Map<String, Object>`

```java
// Java 中使用 Redis String 的思维模型：
// Redis 就是一个巨大的 Map<String, byte[]>
redis.set("user:1", userJson.getBytes());  // key="user:1", value=字节数组
redis.get("user:1");                        // 返回字节数组，你反序列化
redis.incr("counter");                      // value 被当成数字 +1
```

### 1.2 从 Java 的 String 到 Redis 的 SDS

**Java String 的内存模型：**

```
String s = "hello";
┌──────────────┐
│ String 对象  │
│ ● value ────▶│  char[] {'h','e','l','l','o'}  ← 存在堆上
│ ● hash       │  数组对象有长度字段，所以 O(1) 知道长度
└──────────────┘
```

Java String 是完美的——长度 O(1)、自动扩容、有 GC。**但 C 语言的 `char*` 不是：**

```
C 语言的 char*：
  char* s = "hello";
  内存中：['h']['e']['l']['l']['o']['\0']
         s 指针指向 'h'
         没有长度字段！要知道长度必须从 'h' 开始逐字节遍历到 '\0' → O(N)
         想拼接 " world" → 原位置后面没有空间 → 必须 malloc 新空间 → 低效
         中间出现 '\0' → 提前截断 → 图片/音视频数据全废
```

**所以 Redis 造了 SDS（Simple Dynamic String）：**

```
SDS = Java String 的 C 语言替代品
    = char* + 长度字段 + 容量字段 + 自动扩容 + 二进制安全
```

### 1.3 SDS 的结构（完全版）

Redis 的 SDS 针对不同长度范围有 5 种变体，头部从 1 字节到 17 字节不等，极致压榨内存：

```
SDS 版本            │ 实际头大小 │ 适用数据长度    │ Redis 内部用名
────────────────────┼───────────┼────────────────┼──────────────
sdshdr5（已不用）    │ 1 字节    │ < 32 字节       │ TYPE_5
sdshdr8             │ 3 字节    │ < 256 字节      │ TYPE_8  ← embstr 用这个
sdshdr16            │ 5 字节    │ < 65536 字节    │ TYPE_16
sdshdr32            │ 9 字节    │ < 4GB           │ TYPE_32
sdshdr64            │ 17 字节   │ ≥ 4GB           │ TYPE_64
```

**以 sdshdr8 为例（存短字符串用，3 字节头）：**

```
sdshdr8 的内存布局（一共 3 字节头部 + 数据区）：
┌─────────────┬──────────────┬──────────────┬────────────────────┐
│ len         │ alloc        │ flags        │ buf[]（柔性数组）   │
│ uint8_t(1B) │ uint8_t(1B)  │ uint8_t(1B)  │ 实际数据 + '\0'    │
│ 已用长度     │ 总容量(-1)    │ 低3位=类型标记│                    │
└─────────────┴──────────────┴──────────────┴────────────────────┘
          ← 固定 3 字节，sizeof(sdshdr8) = 3 →
```

**逐个字段解释：**

| 字段 | 字节 | 含义 | Java 类比 |
|------|------|------|----------|
| `len` | 1 | 当前字符串长度 | `str.length()` |
| `alloc` | 1 | 总容量-1（不含头不含`\0`） | `ArrayList` 的 `elementData.length`（容量） |
| `flags` | 1 | 类型标记（低 3 位标识 TYPE_8） | 类似 `instanceof` 检查 |
| `buf[]` | 可变 | 实际数据 + 结尾自动加 `\0` | `ArrayList` 的 `elementData[]` |

### 1.4 SDS 的三个杀手特性

**特性一：O(1) 获取长度**

```c
// C 原生字符串要 O(N) 遍历到 \0
size_t len = strlen(cstr);  // 内部是 while(*p != '\0') p++;

// SDS：直接读字段
size_t len = s->len;        // 就是读一个内存值，纳秒级
```

**特性二：杜绝缓冲区溢出（自动扩容）**

```c
// C 原生 strcat：如果 dst 后面没空间 → 覆盖了别的东西的内存 → 崩溃或安全漏洞
strcat(dst, src);

// SDS sdscat：先检查 free 够不够 → 不够自动扩容 → 再拼接
s = sdscat(s, " world");  // 内部分配足够空间再拼接，永远不会溢出
```

扩容策略（类比 `ArrayList` 的 grow）：

```
增长后新长度 <  1MB → 扩容为 2 × 新长度（翻倍）
增长后新长度 ≥  1MB → 扩容为 新长度 + 1MB（固定增量，避免巨大翻倍）

// Java ArrayList 也是类似逻辑：
// int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5 倍
// Redis 比 Java 更激进（翻倍 vs 1.5倍），以空间换时间
```

**特性三：惰性空间释放**

```java
// Java 类比：ArrayList 的 trimToSize()
List<String> list = new ArrayList<>(100);
list.add("a");
list.remove(0);
// 此时 elementData.length 还是 100，没有缩小！
// 除非你手动调 trimToSize()
```

```c
// SDS 同理—缩短字符串时只改 len，不回收 alloc 的内存
// 好处：下次再增长时，如果 free 空间够用，直接原地修改，不需要重新 malloc
s = sdstrim(s, " ");     // 去掉空格，len 变小，alloc 不变
// 下次 append → 发现 free 够用 → 直接写 → 极快
```

### 1.5 RedisObject：String 外层的"包装盒"

Redis 中 **每一个 key 和 value 都包在一个 `redisObject` 结构体中**——不管是 String / List / Hash / Set / ZSet。

**为什么需要这个包装盒？**

```java
// Java 类比：
Object obj = "hello";     // "hello" 是 String，但可以赋值给 Object 类型变量
obj.getClass();           // 运行时知道它实际是 String — RTTI（运行时类型信息）
```

Redis 用 C 写的，C 没有多态/RTTI。所以 Redis 自己造了一个统一的"头部"来描述任意数据：

```
redisObject 结构（16 字节）：
┌──────────────┬──────────────┬────────────────────┬────────────┬──────────────┐
│ type         │ encoding     │ lru                │ refcount   │ ptr          │
│ 4 bit        │ 4 bit        │ 24 bit             │ 4 字节     │ 8 字节       │
│ 逻辑类型      │ 内部编码方式  │ LRU/LFU 淘汰用     │ 引用计数    │ 指向实际数据   │
└──────────────┴──────────────┴────────────────────┴────────────┴──────────────┘
        ← 4 字节 →               ← 4 字节 →           ← 8 字节 →  = 共 16 字节
```

| 字段 | 大小 | 含义 | Java 类比 |
|------|------|------|----------|
| `type` | 4 bit | 用户看到的类型（STRING/LIST/HASH/SET/ZSET） | `obj instanceof String` |
| `encoding` | 4 bit | 内部实际用什么数据结构存的 | 不直接对应——类似"底层实现标记" |
| `lru` | 24 bit | 存 LRU 时间戳或 LFU 计数器（淘汰用） | 类似 Guava Cache 的 accessTime |
| `refcount` | 4 字节 | 引用计数（共享对象用，如 0~9999 整数） | 可以理解为一个简化版 GC 引用计数 |
| `ptr` | 8 字节 | 指向实际数据的指针（64 位系统指针=8 字节） | **这就是 Java 的引用！** |

**关键理解：type vs encoding 是解耦的**

```
type = 用户视角（OBJECT TYPE 命令查看）
encoding = 内部视角（OBJECT ENCODING 命令查看）

比如 Hash 类型：
  type 永远是 "hash"
  但 encoding 可能是 "listpack"（小数据）或 "hashtable"（大数据）
  用户无感切换，Redis 自动决定

就相当于 Java 的：
  interface List<T>      ← type（用户看到的接口）
  class ArrayList<T>     ← encoding（底层实现）
  class LinkedList<T>    ← encoding（另一种底层实现）
  用户只关心 List 接口，不关心底层怎么存
```

### 1.6 String 的三种内部编码

当你执行 `SET key value` 时，Redis 会根据 `value` 的内容自动选择最省内存的编码：

```
OBJECT ENCODING key → 返回 "int" / "embstr" / "raw"
```

#### 1.6.1 int 编码——整数的极致优化

**条件：** value 能解析为整数，且不超过 `long`（64 位有符号）范围。

**做法：** 不分配 SDS！**直接把整数值存在 `redisObject` 的 `ptr` 字段里。**

```
正常的 RedisObject：
┌─────────────┐  ptr(8B)  ┌──────────┐
│ redisObject │──────────▶│   SDS    │  ← 这个 SDS 需要额外 malloc
│   16 字节   │           │ 堆中分配  │
└─────────────┘           └──────────┘

int 编码的 RedisObject（零额外内存！）：
┌─────────────┐
│ redisObject │  ptr = (void*)12345  ← ptr 字段不存地址，直接存整数值！
│   16 字节   │  读的时候：(long)obj->ptr → 12345
└─────────────┘
```

**这就像：**

```java
// Java 中不可能这样做——引用就是引用，不能当数字用
// 但 C 的指针本质就是 8 字节的整数（存的地址值），所以可以"借用"来存别的
// Redis 作者利用了这个 hack：ptr 字段不用来指向东西，直接存数字
```

#### 1.6.2 embstr 编码——对象和数据黏在一起的优化

**条件：** value 是字符串，长度 ≤ 44 字节。

**做法：** `redisObject` 和 `SDS` **在一次 malloc 中连续分配**。对象和数据是"粘在一起"的。

```
embstr 的内存布局（一次 malloc，64 字节块）：
┌──────────────────────64 字节──────────────────────────┐
│ redisObject │ sdshdr8 头 │  buf (字符串数据)  │ '\0'  │
│   16 字节   │   3 字节   │    最多 44 字节     │ 1 字节 │
└──────────────────────64 字节──────────────────────────┘

为什么是 44 字节？
  jemalloc 的分配块大小 = 2^n：8, 16, 32, 64, 128 ...
  64 字节的块正好装下：16 + 3 + 1 = 20 字节固定开销，剩 44 字节给数据

如果数据是 45 字节 → 64 字节块装不下 → 退化为 raw 编码
```

**embstr 的优点（对比 raw）：**

| 维度 | embstr | raw |
|------|--------|-----|
| malloc 次数 | 1 次 | 2 次（redisObject + SDS 分开分配） |
| free 次数 | 1 次（释放一个块即释放所有） | 2 次 |
| CPU 缓存 | **连续内存，cache line 友好** | 两块内存可能相距很远，cache miss |
| 修改操作 | **不支持原地修改**，一改就变 raw | 支持原地修改（SDS 有自己的扩容空间） |

**理解"cache 友好"：** CPU 从内存读数据时，不是按字节读，而是一次读 64 字节（一个 cache line）。`embstr` 的 redisObject 和 SDS 在同一个 64 字节块里，CPU 一次就读进来了。`raw` 的两块内存可能相距很远，CPU 需要两次内存访问。

#### 1.6.3 raw 编码——大字符串

**条件：** value 长度 > 44 字节。

**做法：** `redisObject` 和 `SDS` **分两次 malloc 分配**，redisObject.ptr 指向独立的 SDS。

```
raw 的内存布局（两次 malloc）：
                  ┌───────────────────────┐
┌─────────────┐   │ sdshdr8/16/32 头      │
│ redisObject │   │ + buf (数据 > 44 字节)│
│ ptr ────────┼──▶└───────────────────────┘
└─────────────┘
```

### 1.7 embstr 的"只读"特性

**embstr 是只读的——任何修改操作都会让它退化为 raw。**

为什么？因为你分配的是 64 字节的 jemalloc 块，**块已经封顶了，无法扩容。** 要对 SDS 做 APPEND/SETRANGE，必须扩容 → 但原块找不到相邻空闲空间 → 只能搬到一个新的大块 → 此过程要求 redisObject 和 SDS 分离。

```
127.0.0.1:6379> SET k "hello"
OK
127.0.0.1:6379> OBJECT ENCODING k
"embstr"              ← 5 字节 ≤ 44，用 embstr

127.0.0.1:6379> APPEND k " world，Redis 真的很快，因为他在内存中操作数据"
(integer) 71
127.0.0.1:6379> OBJECT ENCODING k
"raw"                 ← 71 字节 > 44，已经退化为 raw

// 以后不管怎么改，永远都是 raw，不会再回到 embstr
```

**类比 Java：** `String` 是不可变的（immutable）→ 每次修改都生成新 String。`StringBuilder` 是可变的，内部 `char[]` 自动扩容。Redis 的 `embstr` 相当于 `String`（不可变），`raw` 相当于 `StringBuilder`（可变，有自己的扩容空间）。

### 1.8 INCR 原子操作的内部实现路径

```
客户端发送：INCR counter

步骤 1. 解析 RESP 协议：命令=INCR, key=counter

步骤 2. 在 Redis 的全局 dict（键空间）中查找 key="counter"
       ├─ 不存在 → 创建一个 value=0 的 RedisObject(int 编码)
       └─ 存在 → 检查 value 能否解析为整数

步骤 3. 解析当前值（getLongLongFromObject）：
       ├─ encoding=INT → 直接取 (long)obj->ptr 就是整数值
       ├─ encoding=EMBSTR/RAW → 把 SDS 里的字符串转 long（如 "123" → 123L）
       └─ 不是数字 → 报错 "value is not an integer"

步骤 4. val = 当前值 + 1

步骤 5. 新值如果还在 LONG 范围内 → obj->ptr = (void*)新值（仍在 int 编码）
       新值如果超出 LONG 范围 → 转 embstr/raw 编码

全程在 Redis 主线程中一次性完成，中间不会被其他命令打断（单线程优势）
```

**为什么 INCR 是原子的？** 不是因为锁——是因为 **Redis 命令执行全程单线程**。INCR 的"读→加→写"中间不可能插入别的命令。但如果你在 Java 代码中写成两个命令：

```java
// ❌ 不原子的做法：
String val = jedis.get("counter");    // 命令 1
jedis.set("counter", String.valueOf(Integer.parseInt(val) + 1)); // 命令 2
// 两个命令之间可能被其他客户端的 INCR 插入 → 丢失更新
```

### 1.9 String 常用命令详解

#### SET / GET —— 最常用，但参数很多

```bash
SET key value [NX|XX] [GET] [EX seconds|PX ms|EXAT unix-s|PXAT unix-ms] [KEEPTTL]
```

| 参数 | 含义 | 典型用途 |
|------|------|---------|
| `NX` | key 不存在才设置（Not eXists） | **分布式锁**：`SET lock:order uuid NX PX 30000` |
| `XX` | key 存在才设置（eXists） | **更新已存在的 key**，防止误创建 |
| `GET` | 返回旧值（Redis 6.2+） | **原子地替换并获取旧值**，替代旧的 `GETSET` |
| `EX seconds` | 秒级过期 | 缓存：`SET user:1 json EX 3600` |
| `PX ms` | 毫秒级过期 | 分布式锁精确控制：`PX 30000` |
| `KEEPTTL` | 保留原有 TTL（Redis 6.0+） | 更新 value 但不重置过期时间 |

**为什么 SET 要带这么多参数？** 为了**原子性**。在早期版本：

```bash
# 旧方式（两个命令，非原子）：
SETNX lock:order uuid    # 设置
EXPIRE lock:order 30     # 设过期 —— 中间宕机 → 死锁

# 新方式（一条命令，原子）：
SET lock:order uuid NX PX 30000
```

**SET key value GET —— 原子读 + 写：**

```java
// Redis 6.2 之前，获取旧值并设新值需要两个命令（非原子）：
String old = jedis.get("key");   // 1. 读
jedis.set("key", "new");         // 2. 写 —— 之间有竞态

// Redis 6.2+，一行搞定（原子）：
String old = jedis.set("key", "new", SetParams.setParams().get());
// Redis 内部就是一条 SET...GET 命令，中间不会被插入
```

#### MSET / MGET —— 批量操作，减少 RTT

```bash
MSET key1 v1 key2 v2 key3 v3    # 批量设置
MGET key1 key2 key3              # 批量获取 → ["v1","v2","v3"]
```

**MSET 是原子的**——要么全成功，要么全失败（一个 key 有问题则整个 MSET 失败）。不像 `SET key1 v1; SET key2 v2` 两个分开的命令。

**为什么用 MGET 而不是 3 次 GET？**

```
3 次 GET：App → Redis → App → Redis → App → Redis → App（3 × RTT）
1 次 MGET：App → Redis → App（1 × RTT）

跨机房 RTT 1ms，3 次 = 3ms，1 次 = 1ms。高并发下差异巨大。
```

#### INCR / DECR 家族 —— 原子的计数器

```bash
INCR key          # +1 → 返回新值
DECR key          # -1
INCRBY key 10     # +10
DECRBY key 5      # -5
INCRBYFLOAT key 1.5  # +1.5（浮点数）
```

**关键细节：**
- key 不存在 → 当作 0 处理，INCR 后 = 1
- value 不是数字 → 报错 `ERR value is not an integer or out of range`
- INCR 返回的是操作**后**的值
- 范围：-2^63 ~ 2^63-1（64 位有符号整数）

**INCRBYFLOAT 的精度陷阱：**

```bash
SET price "19.99"
INCRBYFLOAT price 0.01   # → "20.00"  浮点计算有舍入误差！
# 电商金额建议用 INCRBY 存「分」：INCRBY price_fen 1（加 1 分钱）
```

#### GETRANGE / SETRANGE —— 把 String 当字节数组用

```bash
SET key "Hello World"
GETRANGE key 0 4     # → "Hello"（含头含尾）
GETRANGE key -5 -1   # → "World"（负数从末尾倒数）
SETRANGE key 6 "Redis"  # → "Hello Redis"（覆盖指定位置）
```

SETRANGE 不会移动其他字符，只覆盖。如果 offset 超过当前长度，中间用 `\x00` 填充：

```bash
SET key "Hi"
SETRANGE key 10 "!"  # → "Hi\x00\x00\x00\x00\x00\x00\x00\x00!"
# STRLEN key → 11（\x00 也算长度）
```

#### GETBIT / SETBIT / BITCOUNT —— String 当位图用

```bash
SETBIT online:2026-05-25 1001 1   # 第 1001 位设为 1
GETBIT online:2026-05-25 1001     # → 1（此用户今天在线）
BITCOUNT online:2026-05-25        # → 今天在线总人数

# 多个日期的 BITOP 交集（连续登录）
BITOP AND online:last7 online:05-19 online:05-20 ... online:05-25
BITCOUNT online:last7   # 7 天连续在线用户数
```

**内存爆炸性的省：** 1 亿用户 = 12.5MB；MySQL 存储 DAU 需要每行 (user_id bigint + date) = 至少 ~11 字节 × 1亿 = 1.1GB。差距 ~90 倍。

#### APPEND / STRLEN

```bash
APPEND key " world"    # 追加字符串，返回追加后长度
STRLEN key             # O(1) 获取长度，直接读 SDS 的 len 字段
```

### 1.10 String 使用场景详解

#### 场景一：常规缓存 —— "为什么用 String 存 JSON 而不直接用 Hash？"

```
方案 A：String 存 JSON
  SET user:1001 '{"name":"张三","age":25,"city":"北京"}' EX 3600

方案 B：Hash 存每个字段
  HSET user:1001 name "张三" age 25 city "北京"
  EXPIRE user:1001 3600
```

| 维度 | String + JSON | Hash |
|------|-------------|------|
| 读写整个对象 | 1 次 GET/SET | 1 次 HGETALL（大 Hash 危险）/ HMSET |
| 读写单个字段 | 需要整存整取，网络传输全量 | 只传需要的字段，HGET/HSET O(1) |
| 嵌套结构 | 天然支持（JSON 可以套） | **不支持**（只能 field-value 平层） |
| 序列化成本 | 应用层 JSON 序列化/反序列化 | 无，Redis 内都是字节 |
| 内存占用 | 1 个 key，大小取决于 JSON 长度 | N 个 field，小 Hash 用 listpack 省内存 |

**选择策略：** 大部分字段一起读 → String JSON；经常只更新个别字段（如只改昵称不改头像）→ Hash；有嵌套结构（如 `user.orders[0].items`）→ 必须 String JSON。

#### 场景二：计数器 —— "为什么用 INCR 而不是 Java 的 AtomicInteger？"

```java
// ❌ 单机：
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();  // 只在当前 JVM 生效，3 台实例 = 3 个计数器

// ✅ 分布式：
redis.incr("article:123:views");  // 所有实例共享同一个计数器
```

**INCR 的性能保证：** 纯内存操作 + 单线程，一次 INCR 约 1-2μs。单机 QPS 10w+ 毫无压力——打满千兆网卡之前 CPU 都跑不满。

#### 场景三：分布式 ID 生成

```java
// 用 INCR 生成全局递增 ID
long orderId = redis.incr("seq:order_id");
// 配合业务前缀：ORD20260525-{orderId}——全局唯一且趋势递增

// 不是 UUID 的原因：UUID 无序 → 插入 MySQL B+Tree 时页分裂严重 → 性能差
// Redis 生成的 ID 趋势递增 → MySQL 插入效率高
```

#### 场景四：Session 共享（String vs Hash 的选择）

```
# 方式 A：存为 String JSON
SET session:abc123 '{"userId":1001,"loginTime":1715000000,"roles":["user","vip"]}' EX 1800

# 方式 B：存为 Hash
HSET session:abc123 userId 1001 loginTime 1715000000
# 但 roles 是数组，Hash 存不了数组 → 只能转 JSON 字符串存为 field

结论：Session 数据如果是扁平 key-value → Hash；有复杂结构 → String JSON
```

#### 场景五：GETSET / SET...GET 实现简单锁

```java
// 原子获取旧值 + 设新值 —— 可用于"flag 切换"类操作
String oldStatus = jedis.set("order:123:status", "PAID", 
    SetParams.setParams().get());  // SET...GET
// oldStatus = "PENDING"，同时新值 "PAID" 已写入
// 可以先判断旧值是否为期望状态，避免并发重复操作
```

### 1.11 String 面试题

**Q1: embstr 为什么截在 44 字节？52 不行吗？**

答：不是 Redis 定的，是 **jemalloc 分配器的 64 字节块限制 + 结构体大小约束** 联立方程的结果：

```
变量：jemalloc 提供 64 字节块（2^6）
固定开销：redisObject(16B) + sdshdr8头(3B) + 结尾'\0'(1B) = 20B
可用 = 64 - 20 = 44B

如果选 128 字节块：可用 = 128-20 = 108 → 但 Redis 经过测试认为 44 是个好分界线
（大多数短字符串 ≤ 44，再多说明不是"小数据"，不如直接 raw）
```

**Q2: SET "12345"，encoding 是 int 还是 embstr？**

答：先尝试 `string2ll` 把 "12345" 转 `long` → 12345（成功，在 long 范围内）→ int 编码。`SET "99999999999999999999"`（20个9）超出 long 最大值 → 转 `long long` 也溢出 → 放弃 int → 走 embstr（如果 ≤44 字节）。

**Q3: MSET 原子吗？和多个 SET 有什么区别？**

答：MSET 原子——要么全成功要么全失败。而多个独立 `SET` 之间可以被其他客户端命令插入。比如 `MSET a 1 b 2`，不会有客户端在 a 更新后 b 更新前看到中间状态。

**Q4: 用 INCRBYFLOAT 存储金额可以吗？**

答：强烈不建议。浮点数有精度误差—— `1.2 + 1.1 ≠ 2.3000000000000003`。正确做法：用 INCRBY 存精度（存「分」而非「元」）。或者用 RedisJSON（支持 BigDecimal 的精度）、或者干脆不在 Redis 层做金额计算，仅做缓存读取。

**Q5: String 类型的 key 命名有什么工程规范？**

答：`业务模块:子模块:标识`，冒号分隔。例如 `user:profile:10086`、`order:info:ORD20260525001`。好处：① `keys user:*` 方便排查；② Redis Cluster 的 hash slot 按 key 中的 `{}` 或整体计算；③ 可读性好。反例：`u10086`、`order_ORD20260525001`（Key 一多完全不知道是什么东西）。

---

## 第二章：List（列表）—— 从 LinkedList 到 quicklist 的进化

### 2.1 Java LinkedList 的启示

```java
// Java 的 LinkedList —— 双向链表
LinkedList<String> list = new LinkedList<>();
list.add("a");  // 创建 Node{prev=null, item="a", next=null}
list.add("b");  // 创建 Node{prev=Node(a), item="b", next=null}

// 每个 Node 的内存开销：
// 对象头(~12B) + prev引用(4B) + next引用(4B) + item引用(4B) = ~24 字节
// 但实际数据 item="a" 不过 2 字节（一个 char）
// 结构开销是数据的 10+ 倍！小数据时极度浪费！
```

Redis 早期版本的 `linkedlist` 编码就面临这个问题——存 1 万个 3 字节的短字符串，光链表指针就有 1万×16字节=160KB 额外开销。所以 Redis 一直在优化这件事。

### 2.2 List 的三代进化

```
Redis 2.x ~ 3.0：
  ziplist（小数据紧凑存储） / linkedlist（大数据双向链表）
  问题：linkedlist 内存碎片严重，两种编码切换不优雅

Redis 3.2 ~ 6.2：
  quicklist（统一实现） = 双向链表做骨干 + 每个节点内嵌 ziplist
  原理：链表少（比如 50 个节点），每个节点里装压缩的数组（省内存）

Redis 7.0+：
  quicklist 中的 ziplist 替换为 listpack
  原因：消除 ziplist 的"连锁更新"安全隐患
```

### 2.3 ziplist 的核心思想——连续内存代替指针

**先理解问题：内存中的指针非常贵。**

```
存 1000 个 "a"：

如果像 Java LinkedList 一样每个元素一个 Node 对象：
  Node (对象头 ~12B) + prev 引用(4B) + next 引用(4B) + item 引用(4B) ≈ 24B
  1000 个 Node = 24,000B，但实际 1000 个 "a" = 1000B
  浪费比 = 24000 : 1000 = 24 倍浪费！

如果用 ziplist（连续内存压在一起）：
  ziplist header 11B + 1000 × 1B（小整数编码，"a" 的整数表示） = 1011B
  vs 24000B，相差 20+ 倍
```

**ziplist 的原理——把所有元素连续排在一起，用相对偏移代替指针：**

```
ziplist 的整体结构（一整块连续内存）：

┌──────────┬──────────┬──────────┬─────────┬─────────┬─────┬─────────┬────────┐
│ zlbytes  │ zltail   │ zllen    │ entry1  │ entry2  │ ... │ entryN  │ zlend  │
│ uint32_t │ uint32_t │ uint16_t │ 变长    │ 变长    │     │ 变长    │ 0xFF   │
│ 总字节数  │ 尾偏移量  │ 元素数量  │         │         │     │         │ 结束标记 │
└──────────┴──────────┴──────────┴─────────┴─────────┴─────┴─────────┴────────┘
     ←──────────── 固定 10 字节 header ────────────→
```

**每个 entry 的内部结构：**

```
┌──────────────────┬────────────────────┬──────────────────┐
│ prevlen          │ encoding + length  │ data             │
│ 前一个 entry 长度 │ 自己的数据类型+长度  │ 实际数据          │
│ (1 或 5 字节)     │ (1/2/5 字节)       │ (变长)           │
└──────────────────┴────────────────────┴──────────────────┘
```

**prevlen 的编码规则（这是 ziplist 最大的坑）：**

```
前驱 entry 长度 < 254 字节 → prevlen 用 1 字节存储
前驱 entry 长度 ≥ 254 字节 → prevlen 用 1 字节标记(0xFE) + 4 字节存实际值 = 5 字节

为什么有这道坎？→ 1 字节最大存 255，但留了一个 0xFF 给 zlend（结束标记）用
                  所以 254 以上需要"跳转标记" + 4 字节
```

### 2.4 ziplist 为什么被 listpack 取代——连锁更新详解

**这是 Redis 数据结构中最精妙的陷阱之一。**

**场景：** 假设所有 entry 长度刚好在 253 字节（都在 prevlen=1 字节的这个舒适区）：

```
初始状态：
┌──────┬──────────┬──────┬──────────┬──────┬──────────┐
│ 1B   │  entry1  │ 1B   │  entry2  │ 1B   │  entry3  │
│prev=1│  253B   │prev=1│  253B    │prev=1│  253B    │  ← 每个 entry 共 254B
└──────┴──────────┴──────┴──────────┴──────┴──────────┘

在头部插入一个 254 字节的新 entry：
┌──────┬──────────┬─????─┬──────────┬─????─┬──────────┐
│ ?    │  new     │ prev │  entry2  │ prev │  entry3  │
│      │  254B    │      │  253B    │      │  253B    │
└──────┴──────────┴──────┴──────────┴──────┴──────────┘

entry2 的前驱变成了 254 字节！→ prevlen 必须从 1B 变 5B
→ entry2 本身从 254B 变成了 258B
→ entry3 的前驱从 253B 变成了 258B → prevlen 也必须从 1B 变 5B
→ entry3 本身从 254B 变成了 258B
→ entry4 同理...一直传到最后一个 entry！

最坏情况：每个 entry 都要扩容 → 每次扩容都要 memmove 搬迁后面所有数据
          → O(N²) 时间复杂度
```

**现实生产中极少触发——** 所有 entry 整齐落在 253/254 边界本身就需要巧合。但理论上它存在，Redis 7.0 决定彻底根治。

**listpack 怎么解决的——把 `prevlen` 改成 `backlen`（存自身长度，不存前驱长度）：**

```
ziplist entry：
  [prevlen][encoding+len][data]  ← prevlen 存前驱长度→连带受前驱影响

listpack entry：
  [encoding+len][data][backlen]  ← backlen 存自身长度→不受任何前驱影响！
  
反向遍历时：读当前 entry 的 backlen → 知道当前 entry 总长度 → 跳到前一个 entry
```

### 2.5 quicklist——链表的中间地带

纯 ziplist/listpack 虽然省内存，但插入中间时需要搬移数据（O(N)）；纯 linkedlist 虽然插入快，但指针开销大。**quicklist 取两者之长。**

```
quicklist = LinkedList<Ziplist>：

┌─────────────────┐  prev/next  ┌─────────────────┐         ┌─────────────────┐
│  quicklistNode  │◄──────────▶│  quicklistNode  │◄───────▶│  quicklistNode  │
│  · ziplist      │            │  · ziplist      │         │  · ziplist      │
│  [a, b, c, d]   │            │  [e, f, g, h]   │         │  [i, j, k, l]   │
└─────────────────┘            └─────────────────┘         └─────────────────┘
   4个元素在一个节点               4个元素在一个节点             4个元素在一个节点

总共 12 个元素，只有 3 个链表节点 = 3 组前后指针（6 个 8 字节指针 = 48 字节）
vs 纯 linkedlist = 12 个 Node，每组 16 字节前后指针 = 192 字节
```

**head 和 tail 节点是特殊的（不压缩）：**

```
list-compress-depth 1 的含义：

  [不压缩] ←→ [压缩] ←→ [压缩] ←→ ... ←→ [压缩] ←→ [不压缩]
    head                                        tail

两端各 1 个不压缩：保证 LPUSH/LPOP/RPUSH/RPOP 极快（不需要解压）
中间的节点 LZF 压缩（类似 zip，极轻量，约 50% 压缩率）
```

**quicklist 的节点源码定义（逐字段解释）：**

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;  // 前驱节点引用（8字节）
    struct quicklistNode *next;  // 后继节点引用（8字节）
    unsigned char *entry;        // 指向 ziplist/listpack 数据的引用（8字节）
    size_t sz;                   // entry 的总字节数（8字节）
    unsigned int count : 16;     // 本节点内有多少个元素（2字节）
    unsigned int encoding : 2;   // 编码：1=无压缩, 2=LZF压缩（2bit）
    unsigned int recompress : 1; // 是否待重新压缩（1bit）
    // ...
} quicklistNode;
```

### 2.6 从 Java 角度看 quicklist

```java
// 你可以把 quicklist 想象成这样（仅是概念类比，不是源码）：
class QuickList {
    // 双向链表的节点，但不是每个节点只存 1 个元素，而是存一个压缩块
    static class QuickListNode {
        QuickListNode prev;
        QuickListNode next;
        byte[] compressedBlock;  // 里面装了多个元素（类似批量打包）
        int blockSize;           // 这个块的字节数
        int elementCount;        // 这个块里装了多少元素
        boolean isCompressed;    // 是否被 LZF 压缩了
    }
    
    QuickListNode head;
    QuickListNode tail;
    long totalElements;          // 所有节点合计元素总数
    int nodeCount;               // 节点个数
}
```

**当你 LPUSH 一个元素时：**

```
1. 找到 head 节点（链表头）
2. 检查 head 节点里的 ziplist 还有空间吗（< fill 限制，默认 8KB）
   ├─ 有空间 → 直接在 ziplist 头部插入
   └─ 无空间 → 创建新的 quicklistNode（含新 ziplist），元素放进去
                新节点成为 head，原 head 变为第二个
3. LPUSH 完成，O(1)
```

### 2.7 List 常用命令详解

#### LPUSH / RPUSH —— 左推右推

```bash
LPUSH key v1 v2 v3    # 逐个从左侧推入 → [v3, v2, v1]（写在前面的在右边）
RPUSH key v1 v2 v3    # 逐个从右侧推入 → [v1, v2, v3]（写在前面的在左边）

# 性能：O(N)，N = 推入元素数
# 都支持批量，比逐个 LPUSH 更省网络 RTT
```

#### LPOP / RPOP —— 左弹右弹

```bash
LPOP key              # 从左侧弹出，返回弹出的元素
RPOP key              # 从右侧弹出

# Redis 6.2+ 支持 count 参数：
LPOP key 3            # 一次弹出 3 个 → ["a","b","c"]，O(N) N=弹几个
```

**弹空后：** key 被自动删除。这让 List 天然适合做消费型队列——消费完就释放内存。

#### BLPOP / BRPOP —— 阻塞弹出（消息队列的灵魂）

```bash
BLPOP queue1 queue2 30     # 阻塞等 30 秒，哪个队列先有数据就弹哪个
BRPOP queue 0              # 0 = 无限阻塞，数据不来就一直等
```

**关键实现细节：**

```
1. BLPOP 检查 queue 是否非空 → 非空直接弹出，不阻塞
2. 如果空 → 阻塞当前客户端，阻塞队列中记录该客户端
3. 另一个客户端 LPUSH → 数据推入成功后，检查阻塞队列 → 把数据返回给等待者
4. 超时 → 返回 nil

这不是轮询！是事件驱动的——数据到达才会唤醒阻塞的客户端。
```

**多个 Key 的优先级规则：** `BLPOP q1 q2 0`——q1 有数据就不看 q2；只有 q1 空且 q2 有，才弹 q2。适合给不同优先级队列排序。

#### LLEN —— O(1) 获取长度

```bash
LLEN key        # 直接读 quicklist->count，O(1)，不遍历
```

#### LRANGE —— 获取范围内的元素

```bash
LRANGE key 0 9       # 前 10 个元素（索引 0 ~ 9）
LRANGE key -5 -1     # 最后 5 个（负数倒着数）
LRANGE key 0 -1      # 全部（⚠️ 元素太多会阻塞！）

# 复杂度：O(S+N)，S=start 的偏移量，N=返回个数
# 所以 LRANGE key 0 9 很快（O(1+10)），不管 List 总长度多大都很快
# 但 LRANGE key 0 -1 如果 List 有 100 万元素 → 阻塞！
```

**为什么 LRANGE 0 9 快？** 跳表/quicklist 定位到起点需要跳过头节点 → 跳过 start 个元素（O(N)），但 0 的话就不用跳。**大 List千万不要 LRANGE 0 -1。**

#### LINDEX —— 按索引取一个

```bash
LINDEX key 0      # 第一个，O(N)（要遍历到目标位置）
LINDEX key -1     # 最后一个，O(1)（直接取 tail）
```

#### LTRIM —— 截断（实现"保留最近 N 条"的利器）

```bash
RPUSH timeline "new post 1"
LTRIM timeline 0 99      # 只保留最近 100 条，O(N) N=被裁剪的元素数
```

**经典的"最新动态"实现：**

```bash
LPUSH user:100:timeline "动态内容"
LTRIM user:100:timeline 0 999   # 只保留最近 1000 条，超过的被自动删除
```

#### LINSERT —— 在指定元素前/后插入

```bash
LINSERT key BEFORE "pivot" "new_value"   # 在 pivot 之前插入
LINSERT key AFTER "pivot" "new_value"    # 在 pivot 之后插入
# O(N) —— 需先遍历找到 pivot 再插
# 如果有重复的 pivot → 匹配找到的第一个
```

#### LREM —— 删除匹配元素

```bash
LREM key 2 "hello"      # 从左删除 2 个值为 "hello" 的元素
LREM key -1 "hello"     # 从右删除 1 个
LREM key 0 "hello"      # 删除所有 "hello"
# O(N+M)，N=List长度，M=删除个数
```

#### LMOVE / BLMOVE —— 跨队列移动（Redis 6.2+）

```bash
LMOVE src dst LEFT RIGHT     # 从 src 左侧弹出，推入 dst 右侧
BLMOVE src dst LEFT RIGHT 5  # 阻塞版，等 5 秒
```

这是 `BRPOPLPUSH`（已废弃）的替代，更强——可指定源/目标的左右方向。适合**可靠消息队列**的备份模式。

### 2.8 List 使用场景详解

#### 场景一：消息队列 —— "LPUSH + BRPOP 经典模式"

```java
// 生产者
jedis.lpush("queue:order", JSON.toJSONString(order));

// 消费者（阻塞等待）
while (true) {
    List<String> msg = jedis.brpop(5, "queue:order");
    if (msg != null) {
        process(msg.get(1));  // msg[0] = key, msg[1] = value
    }
}
```

**为什么 List 天然适合队列？**
- 头尾操作 O(1)：push 一头，pop 另一头 → 完全的 FIFO
- BRPOP 阻塞等 → 没有数据时 CPU 空转
- 弹空自动删除 → 没有残留 key

**但它的致命缺陷（面试重点）：**

```
消费者 BRPOP 拿到消息 → 开始处理 → 处理到一半崩溃了 → 消息已弹出，丢了！

ACL（Access Control List）证明 List 无 ACK 机制。
解决：
  方案 A：BRPOPLPUSH / BLMOVE → 弹出同时备份到另一个 List
          消费者处理完 → LREM 从备份 List 删除
          消费者崩溃 → 备份 List 中仍是 pending 状态 → 定时任务扫描恢复
  
  方案 B：Redis 5.0+ Stream → 原生 ACK 机制、PEL（Pending List）、消费组
          严肃场景直接上 Stream 或专业 MQ
```

#### 场景二：最新动态 / 时间线 —— "LPUSH + LTRIM"

```java
// 发布动态
public void addTimeline(long userId, String feedJson) {
    String key = "timeline:user:" + userId;
    jedis.lpush(key, feedJson);
    jedis.ltrim(key, 0, 999);  // 只保留最新 1000 条
}

// 拉取最新 20 条
public List<String> getLatest(long userId, int count) {
    return jedis.lrange("timeline:user:" + userId, 0, count - 1);
}
```

**为什么用 List 而不是 ZSet？**
- 时间线不需要"按分数排序"（时间戳天然是插入顺序）
- List 的 LPUSH 比 ZSet ZADD + 时间戳 → 快且省内存
- 如果要做"推荐排名的 feed 流"则用 ZSet（多权重排序）

#### 场景三：栈 —— "LPUSH + LPOP"

```bash
LPUSH stack:undo "action1"    # 入栈
LPUSH stack:undo "action2"    
LPOP stack:undo               # 出栈 → "action2"（后进先出）
```

适用：撤销队列、计算器操作回退。

#### 场景四：可靠消息队列 —— "LMOVE 备份模式"

```java
// 消费者：从主队列弹出 → 原子推入备份队列
String msg = jedis.lmove("queue:main", "queue:backup", 
    LMoveArgs.Builder.leftRight());

try {
    process(msg);
    // 处理完 → 从备份队列删除
    jedis.lrem("queue:backup", 1, msg);
} catch (Exception e) {
    // 没处理完 → 备份队列留存 → 定时恢复任务扫描
}
```

### 2.9 List 面试题

**Q1: 为什么 Redis 不用 Java 的 LinkedList 那种做法（每个元素一个 Node）？**

答：记忆溢出。Java LinkedList 每个 Node 大约 24 字节（对象头+3个引用），而 listpack 里的小元素只需要 1-2 字节。存百万级小数据时，quicklist 可能只要 ~3MB，纯 linkedlist 需要 ~24MB。Redis 是内存数据库，每一字节都要省。

**Q2: quicklist 比纯 linkedlist 慢在哪里？**

答：LINDEX（按索引查找元素）——要先定位到是第几个 quicklistNode，再在那个节点的 ziplist 里线性查找。O(N) 不变，但常数略大（两层遍历）。不过 Redis 定位是"内存操作快过网络 I/O"，这点开销在微秒级，相比网络往返可忽略。

**Q3: List 做消息队列和 Redis Stream、Kafka 的差别？**

答：List 是最简版——没有 ACK、没有消费组、无持久化保证、无消息回溯。Stream 补了 ACK + 消费组 + PEL——但仍是内存队列，可靠性不如 Kafka。Kafka 是专业 MQ——磁盘大存储 + 多副本 ISR + Connector 生态。一句话：List 是玩具、Stream 是轻量、Kafka 是生产。

**Q4: LRANGE 0 -1 为什么危险？如何安全遍历大 List？**

答：LRANGE 0 -1 一次性返回 List 的全部元素（O(N)），如果 List 有 10 万元素 → Redis 主线程阻塞 → 所有请求排队。稳妥做法：① `LRANGE 0 99`、`LRANGE 100 199` 分段分页；② 或根本不存大 List——限制每个 List 上限（LPUSH + LTRIM 自动截断）。

---

## 第三章：Hash（哈希）—— Redis 的"小 HashMap"

### 3.1 Java 的 HashMap 是怎么存储的？

先回顾 Java 的 HashMap，方便后面对比：

```java
// Java HashMap 的内部结构：
HashMap<String, User> map = new HashMap<>();
map.put("user:1", new User("张三"));

内部存储：
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12 │13 │14 │15 │ ← 桶数组
└───┴───┴───┴─│─┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
               │
               ▼
            ┌─────────────┐
            │ Node(key=   │
            │ "user:1",   │
            │ value=User, │
            │ next=null)  │
            └─────────────┘
            
"user:1".hashCode() → 某个 int 值 → 取模映射到桶索引 3 → 放在桶[3] 上
如果哈希冲突 → 链表 / 红黑树
如果元素太多 → 扩容（容量翻倍），所有元素 rehash 重新打散
```

### 3.2 Redis Hash 的两种内部编码

```
小 Hash → listpack（内存紧凑）    类似 ArrayList<byte[]> 顺序存储
大 Hash → dict（真正哈希表）      类似 Java HashMap

切换阈值：
  hash-max-listpack-entries 512    ← field 个数 > 512 → 切换
  hash-max-listpack-value 64       ← 任一 field/value 长度 > 64 字节 → 切换
  两个条件都满足才切换
```

### 3.3 listpack 编码的 Hash

```
Hash 存在一个 listpack 中，field-value 交替排列：
[f1][v1][f2][v2][f3][v3]...

HGET key f2：
  从头遍历 → 找到 f2 → 读取下一个 entry = v2 → 返回
  O(N)，但 N ≤ 512，且连续内存无 cache miss，遍历一次可能只要几微秒
```

**什么时候用 listpack？** field 少、field/value 都短。比如 `HSET user:1 name "张三" age "25"`——3 个 field，一行紧凑内存搞定。

### 3.4 dict 编码——Redis 的"哈希表引擎"

**dict 是 Redis 中最核心的数据结构——全局键空间、Hash 类型大数据、Set 大数据、ZSet 的 member→score 映射都用 dict。**

#### 3.4.1 dict 的三层结构

```
dict 整体的三层嵌套（从外到内）：

dict（字典）
 └─ dictht（哈希表）×2 —— ht[0] 当前用，ht[1] rehash 目标
     └─ dictEntry[]（桶数组）—— 每个桶是一个链表的头
         └─ dictEntry → dictEntry → dictEntry（拉链法）

就相当于：
class RedisDict {
    HashTable ht0;  // 当前使用的哈希表
    HashTable ht1;  // 扩容时的目标表（平时为空）
    int rehashIdx;  // -1 = 不在 rehash；≥0 = 正在 rehash
}

class HashTable {
    DictEntry[] table;  // 桶数组
    int size;           // 桶数量（总是 2^n）
    int used;           // 已有 entry 数量
}

class DictEntry {
    Object key;        // SDS 字符串
    Object value;      // 实际数据（可能是 RedisObject、long、double）
    DictEntry next;    // 指向链表下一个（拉链法解决冲突）
}
```

**dictEntry 的 struct：**

```c
typedef struct dictEntry {
    void *key;                    // 指向 key（SDS 字符串）；指针 8 字节
    union {                       // 联合体——多个字段共享同一内存，只有一个有效
        void *val;                // 指向 value（指针）；8 字节
        uint64_t u64;            // 或直接存 64 位无符号整数；8 字节
        int64_t s64;             // 或直接存 64 位有符号整数；8 字节
        double d;                 // 或直接存 double；8 字节
    } v;                          // 联合体总共 8 字节
    struct dictEntry *next;       // 指向链表下一个 entry；指针 8 字节
} dictEntry;                      // 共 24 字节 = 3 × 8
```

**union 是什么？** Java 没有直接的对应物。可以理解为"一个字段，多种身份"——这个 8 字节的空间，有时候存指针、有时候存整数、有时候存 double，同一时间只有一种身份有效。Redis 用这个技巧避免为不同 value 类型分配不同大小的 entry。

#### 3.4.2 哈希表——桶数组（dictht）

```c
typedef struct dictht {
    dictEntry **table;        // 指向桶数组的指针（指针的指针）
                              // table[0] 是第一个桶，table[1] 第二个...每个桶是一个链表头
    unsigned long size;       // 桶数组大小（总是 2^n：4,8,16,32...）
    unsigned long sizemask;   // = size - 1（用于快速 hash & sizemask 替代 % 取模）
    unsigned long used;       // 已经装了多少个 entry
} dictht;
```

**`dictEntry **table` 怎么理解？**

```java
// Java 类比：
// dictEntry** 等价于 Java 的 DictEntry[]（数组的引用）
// 每个 table[i] 是一个链表头（DictEntry 的引用）

DictEntry[] table;  // Java 版——数组，每个元素是链表的头节点引用
table = new DictEntry[16];  // 16 个桶
table[3] = new DictEntry(key1, val1, null);  // 桶[3]上放一个 entry
table[3].next = new DictEntry(key2, val2, null);  // 桶[3]上挂第二个 entry
```

#### 3.4.3 哈希取模的技巧——为什么桶数总是 2^n

```java
// Java：hash & (table.length - 1)   ← 位运算取模，前提是 length = 2^n
int index = hash & (table.length - 1);

// 如果 size = 16 (2^4)，sizemask = 15 (binary: 0000 1111)
// hash & 15 = hash 的低 4 位 → 正好映射到 0~15 → 等价于 hash % 16，但速度更快

// 扩容后 size = 32 (2^5)，sizemask = 31 (binary: 0001 1111)
// hash & 31 = hash 的低 5 位 → key 可能被重新分配到 16+新位置（稍后详述）
```

#### 3.4.4 扩容触发条件

```
扩容（ht[1].size = 大于等于 ht[0].used*2 的最小 2^n）：
  ├─ 负载因子 ≥ 1（used/size ≥ 1） 且 没有子进程在跑（dict_can_resize=1）
  └─ 负载因子 ≥ 5  →  强制扩容，不管子进程

缩容（used / size < 0.1，且 used > 4）：
  ht[1].size = 大于等于 used 的最小 2^n
```

**负载因子是什么？** 类比 Java HashMap 的 `threshold = capacity * loadFactor`。Redis 的负载因子阈值是 1——比 Java 的 0.75 更激进（Redis 愿意承受更多冲突以节省内存）。

### 3.5 渐进式 Rehash——Redis 最重要的性能设计之一

#### 3.5.1 为什么不能一次性 rehash？

```java
// Java HashMap 的扩容：一次性迁移所有元素
// 如果 HashMap 里有 100 万个 entry → 一次性 rehash 可能要几十毫秒
// Java 多线程环境下，这会导致一条线程长时间占有 CPU
// Redis 单线程环境 → 几十毫秒 = 几万个请求排队 → 用户明显感知延迟
```

**Redis 做法：把一个"大搬迁"拆成"无数次小搬迁"。**

#### 3.5.2 渐进式 Rehash 的执行流程

```
启动 rehash：
  1. ht[1].size = 大于等于 ht[0].used*2 的最小 2^n
     例如 ht[0].used = 5, ht[0].size = 4 → ht[1].size = 16 (不小于 10 的 2^n)
  2. 分配 ht[1] 的桶数组
  3. rehashidx = 0（开始从 ht[0] 的第 0 个桶搬迁）

每次搬迁：
  dict 被访问时（增/删/改/查），顺带搬迁 ht[0].table[rehashidx]
  把一个桶及其链表上所有 entry 重新 hash 搬到 ht[1]
  rehashidx++

搬迁完成：
  当 rehashidx == ht[0].size（所有桶搬完）
  → 释放 ht[0]，ht[1] 升格为新的 ht[0]
  → rehashidx = -1（标志着 rehash 已完成）
```

**rehash 期间的查找：** 先去 ht[0] 找 → 没找到 + 正在 rehash → 再去 ht[1] 找。因为新增只写 ht[1]，所以 ht[1] 可能有 ht[0] 没有的 key。

#### 3.5.3 每次搬一个桶是什么体验？

```
假设 ht[0] 有 512 个桶，每个桶平均 2 个 entry：

客户端发了 512 次请求（每次请求触发一次搬迁）→ rehash 完成
用户完全感知不到——每次请求只是多了 "搬一个桶（平均 2 个 entry）的重新哈希" 这点时间
```

**如果没有任何请求怎么办？** Redis 的定时任务 `serverCron` 每 100ms 运行一次，每次用 1ms 时间片专门做 rehash 搬迁，确保空实例的 rehash 也会向前推进。

**类比 Java：** Java 的 `ConcurrentHashMap` 扩容时也是多线程分段迁移，但那是为了并发——Redis 单线程，分段迁移是 **为了不卡住主线程**。

### 3.6 Hash 常用命令详解

#### HSET / HGET / HMSET / HMGET —— 增删改查四联

```bash
HSET key field1 value1 field2 value2 ...   # 批量设值，返回新增的 field 数
HGET key field                              # 取单个 field
HMSET key f1 v1 f2 v2                       # 已废弃（HSET 现在就是批量了）
HMGET key f1 f2 f3                          # 批量取 → ["v1","v2","v3"]
```

**HSET 的复用性：** Redis 4.0 起 HSET 就可以接受多个 field-value 对，HMSET 已被废弃。HSET 返回的是**新增** field 数量（更新已有 field 不算新增）。

**HMGET 的性能优势与 MGET 同理**——一次 RTT 取回多个 field，而不是 N 次 HGET：

```
// 一次网络往返拿 3 个字段
List<String> vals = jedis.hmget("user:1001", "name", "age", "city");
```

#### HGETALL —— 最常用的同时也是最危险的

```bash
HGETALL key     # 返回所有 field-value，O(N)，N = field 数量
                # 返回值：[f1, v1, f2, v2, f3, v3 ...]
```

**HGETALL 的阻塞风险：**

```
Hash 有 1 万个 field → 遍历 1 万个 dictEntry → 序列化 2 万个字符串 → 
返回客户端（分配缓冲区）→ 可能阻塞 Redis 主线程 10-50ms

10ms 在 Redis 世界是很长的——意味着几千个请求排队。
```

**安全替代方案——HSCAN：**

```bash
HSCAN key 0 MATCH * COUNT 100    # 每次返回最多 100 对 + 新 cursor
# cursor 为 0 表示遍历结束
```

```java
// Java 中使用 HSCAN 安全遍历
String cursor = "0";
do {
    ScanResult<Map.Entry<String, String>> result = 
        jedis.hscan("user:1001", cursor, new ScanParams().count(100));
    for (Map.Entry<String, String> entry : result.getResult()) {
        // 处理每个 field-value
    }
    cursor = result.getCursor();
} while (!"0".equals(cursor));
```

**HSCAN 的注意事项：**
- 不阻塞，但**非原子**——遍历过程中元素可能被增删，导致重复或遗漏
- 保证：同一元素在整个遍历中最多返回一次，不会无限循环
- COUNT 是**建议值，不是精确**——Redis 内部会根据桶密度调整实际返回数

#### HKEYS / HVALS / HLEN

```bash
HKEYS key       # 返回所有 field，O(N) —— ⚠️ 大 Hash 同样危险
HVALS key       # 返回所有 value，O(N) —— ⚠️ 同上
HLEN key        # field 数量，O(1) —— 直接读 dict->used
```

#### HEXISTS / HDEL

```bash
HEXISTS key field     # 判断 field 是否存在，O(1)
HDEL key f1 f2 f3     # 删除多个 field，O(N) N=删除个数
                      # 返回成功删除的个数
```

#### HINCRBY / HINCRBYFLOAT —— Hash 内的原子计数器

```bash
HINCRBY key field 1         # +1
HINCRBYFLOAT key field 0.5  # +0.5
```

**和 String INCR 比的优势：** 可以把一组相关的计数器存在一个 Hash 里：

```bash
# 不用：
INCR page:home:uv
INCR page:home:pv
INCR page:detail:uv
INCR page:detail:pv
# ... key 数量爆炸

# 用了：
HINCRBY stats:page home:uv 1
HINCRBY stats:page home:pv 1
HINCRBY stats:page detail:uv 1
# 所有页面统计在一个key里，降低全局 key 空间膨胀
```

#### HRANDFIELD —— 随机取 field（Redis 6.2+）

```bash
HRANDFIELD key 3              # 随机 3 个 field（可能重复）
HRANDFIELD key 3 WITHVALUES   # 随机 3 个 field + value
HRANDFIELD key -3             # 随机 3 个不重复的 field
```

适用：随机推荐、抽奖（Hash 版 SPOP）。

### 3.7 Hash 使用场景详解

#### 场景一：对象存储 —— "什么时候用 Hash 替代 String JSON？"

**场景：** 用户 profile，经常更新某个字段（如状态、头像、最后登录时间）

```java
// String JSON 方式：
String json = jedis.get("user:1001");           // 完整 JSON 序列化/反序列化
User user = JSON.parseObject(json, User.class);
user.setStatus("online");
jedis.set("user:1001", JSON.toJSONString(user)); // 完整 JSON 写回

// Hash 方式：
jedis.hset("user:1001", "status", "online");     // 只改一个 field
```
**核心优势：**
- 原子性：HSET 单个 field 天然原子，不需要乐观锁（CAS）防并发
- 节省序列化：JSON 的序列化/反序列化的 CPU 开销 > 单纯字节拷贝
- 网络省流量：只传 status 的 value，不传整个 user JSON

#### 场景二：购物车 —— "为什么用 Hash 而不是 String？"

```
Key: cart:user:{userId}
Field: SKU_ID
Value: 数量

HSET cart:user:1001 "SKU001" 2      # 放两件 SKU001
HINCRBY cart:user:1001 "SKU001" 1   # 加一件 → 3

# 查看购物车所有商品：
HGETALL cart:user:1001  →  ["SKU001","3","SKU002","1"]

# 删除单品：
HDEL cart:user:1001 "SKU001"

# 购物车总数：
HLEN cart:user:1001 → 2 种商品
```

**如果用 String JSON：** 整个 `{SKU001:3,SKU002:1}` 的 JSON 要大搞特搞序列化 → 修改再写回 → 超多 CPU 开销 + 并发要 CAS → 超复杂。Hash 轻松一招 `HINCRBY cart:user:1001 SKU001 1`。

#### 场景三：计数器分组 —— "Hash 减少内存碎片"

```java
// ❌ 每个指标一个 String key：
INCR "stats:20260525:page_home:uv"     // key length = ~30
INCR "stats:20260525:page_home:pv"     // key length = ~30
INCR "stats:20260525:page_detail:uv"
INCR "stats:20260525:page_detail:pv"
// 4 个 key，每个 key 在全局 dict 中有一个 dictEntry + redisObject ≈ ~60B 开销
// 4 × 60 = 240B 固定开销，数据本身才 4B

// ✅ 一个 Hash：
HINCRBY "stats:20260525:page_home" "uv" 1
HINCRBY "stats:20260525:page_home" "pv" 1
// 1 个 key (60B 开销) + 各 field-value 在 listpack 中紧凑共存
// 60B + ~20B 数据 = 80B，省了 160B（对 4 项来说）
// 1 万个 counter → 省 1.6MB，百万级 → 省 160MB
```

#### 场景四：用户签到（Hash + Bitmap 组合）

```
# 用户签到状态（这应该用 Bitmap）
SETBIT checkin:20260525 userId 1 

# 但用户的签到次数统计、连续签到天数等 —→ Hash：
HINCRBY user:1001 "total_checkins" 1
HSET user:1001 "last_checkin_date" "20260525"
```

### 3.8 Hash 面试题

**Q1: HGETALL 为什么不安全？**

答：HGETALL 返回 Hash 中所有 field-value，O(N)。Hash 有几万个 field → 遍历+序列化+返回需要缓冲区 → 可能阻塞几十甚至上百毫秒，整台 Redis 被卡住。替代：`HSCAN cursor` 分批次遍历（非阻塞但遍历过程中元素可能变化），或设计层面避免大 Hash。

**Q2: Listpack（原 ziplist）和 dict 编码的切换条件是什么？**

答：当 Hash 的所有 field 长度都 ≤ `hash-max-listpack-value`（默认 64 字节）且 field 总数 ≤ `hash-max-listpack-entries`（默认 512）时使用 listpack，超出任一条件则升级为 dict（hashtable）。升级是单向的，dict 不会降级回 listpack。

**Q3: 为什么购物车推荐用 Hash 而不是 String JSON？**

答：HINCRBY 原子增减单品数量；HSET 更新单个字段不改其他；HMGET 一次交互拿购物车中想要的信息；无需 JSON 序列化开销；无需 CAS 并发控制——所有操作都是原子的。

---

## 第四章：Set（集合）—— 去重 + 集合运算的三板斧

### 4.1 Set 的本质

```
Set = 不关心顺序，只关心"存在与否"的去重集合
    = Java 的 HashSet 语义
```

### 4.2 Set 的两种内部编码

```
小 Set（全为整数 + ≤ 512 元素）→ intset（有序整数数组，二分查找）
大 Set 或 有非整数 → dict（哈希表，value=NULL）

切换阈值：set-max-intset-entries 512
```

### 4.3 intset——Redis 最精致的整数压缩结构

```
intset = 有序整数数组，不带任何指针

┌───────────────┬───────────────┬───────────────────────┐
│ encoding      │ length        │ contents[]            │
│ uint32_t(4B)  │ uint32_t(4B)  │ 按 encoding 定长存储   │
│ 编码类型       │ 元素个数       │ 升序排列，二分查找      │
└───────────────┴───────────────┴───────────────────────┘
    固定 8 字节头

encoding 三种取值：
  INTSET_ENC_INT16 = 2   → 每个元素占 2 字节（范围 -32768 ~ 32767）
  INTSET_ENC_INT32 = 4   → 每个元素占 4 字节（范围 -2^31 ~ 2^31-1）
  INTSET_ENC_INT64 = 8   → 每个元素占 8 字节（范围 -2^63 ~ 2^63-1）
```

**元素存储示例：**

```
encoding=INT16，length=3，存 [1, 100, 30000]

内存布局：
┌────┬────┬────┬────┬────┬────┬────┬────┬──────────┐
│enc │enc │len │len │  1 │ 100│30000│30000│          │
│    │    │    │    │(2B)│(2B)│(2B) │(2B) │          │
└────┴────┴────┴────┴────┴────┴────┴────┴──────────┘
 ← 4B header →← 4B header →← 3×2B = 6B contents →

一共 14 字节 = 3 个整数。如果存在 dict 里，仅指针就数倍于此。
```

**编码升级（intsetUpgradeAndAdd）：**

```
现有 encoding=INT16（2字节/元素），存 [1, 2, 3]
要插入 65536（超出 INT16 范围）

升级步骤：
  1. 新 encoding = INT32（4字节/元素）
  2. 重新分配 contents 数组：
     新大小 = 8B(header) + 4元素×4B = 24B
  3. 从后往前搬数据（关键！避免覆盖）：
     先把 3 从 2 字节转 4 字节 → 放在末尾
     再把 2 从 2 字节转 4 字节 → 放在倒数第二
     再把 1 从 2 字节转 4 字节 → 放在倒数第三
     65536 放在 1 之前(如果没找到位置)
  4. encoding = INT32, length = 4
  5. 释放旧 contents

只升级不降级：删掉 65536 后 encoding 仍然是 INT32
```

**为什么要保持有序？** 因为要支持 **二分查找**，O(logN)。增/删/查都是先二分定位再操作。

### 4.4 Set 的 dict 编码——空 value 技巧

```
当 Set 不是纯整数、或元素 > 512 → dict 编码

dict 中的每个 entry：
  key = member（SDS 字符串）
  value = NULL  ← 不存任何东西！key 本身就够判断"存在与否"

为什么存 NULL？
  Set 只关心 "member 在不在"，不需要额外存 value
  NULL 在 dictEntry 的 union 中就是 (void*)0 —— 零开销
```

### 4.5 集合运算——Set 的杀手锏

**SINTER（交集）内部算法：**

```
SINTER key1 key2 key3：

1. 把所有集合按 size 排序：key3(100个) < key2(1000) < key1(10000)
2. 遍历最小的集合 key3 的每个 member
3. 对每个 member，查它在 key2 和 key1 中是否存在
4. 都存在 → 加入结果

为什么从最小集合开始？→ 每个元素最多检查 (M-1) 次
总操作次数 = size(最小) × 检查次数 = 100 × (3-1) = 200 次 dict 查找
如果从最大开始 = 10000 × 2 = 20000 次 → 理论上差 100 倍
```

### 4.6 Set 常用命令详解

#### SADD / SREM / SISMEMBER —— 增删查

```bash
SADD key m1 m2 m3       # 批量加，返回新增的元素数（跳过已存在的）
SREM key m1 m2 m3       # 批量删，返回成功删掉的元素数
SISMEMBER key member    # 判断存在，O(1)

# SADD 天然保证只加不重复的！不需要 SELECT 检查再 INSERT
```

**SADD 的返回值深意：**

```bash
SADD likes:article:123 user:456  # → 1（新元素，成功点赞）
SADD likes:article:123 user:456  # → 0（已存在，重复点赞被忽略）
# 返回 0 = 已存在 = 用户已点赞 = 重复请求被幂等，无需额外检查
```

#### SCARD / SMEMBERS —— 总数和全量

```bash
SCARD key               # 元素总数，O(1)
SMEMBERS key            # 所有元素，O(N) —— ⚠️ 大集合慎用！

# 10 万个元素 → 阻塞数毫秒，不建议
```

#### SRANDMEMBER / SPOP —— 随机取

```bash
SRANDMEMBER key 3       # 随机 3 个，不会删除（可重复）
SRANDMEMBER key -3      # 随机 3 个，不会删除（不重复）
SPOP key                # 随机弹出一个——删掉的，再也不回来
SPOP key 3              # 随机弹出 3 个
```

**SPOP vs SRANDMEMBER 的区别（重要）：**
- SPOP：弹出即删除（符合抽奖/验证码语义）
- SRANDMEMBER：只读不删（符合随机推荐语义）

#### SSCAN —— 安全的遍历大 Set

```bash
SSCAN key cursor COUNT 100   # 分批次遍历，非阻塞
# 使用方式与 HSCAN 完全一致
```

#### SMOVE —— 元素在两个 Set 间移动

```bash
SMOVE source dest member     # 从 source 移除 member，加入 dest
# 原子操作（移除 + 加入一气呵成）
```

适用：用户从一个分组移动到另一个分组（如 "未读" → "已读"）

#### 集合运算（Set 的杀手锏）

```bash
# 交集（共同好友 / 共同标签）
SINTER key1 key2 key3            # 返回所有 key 的交集
SINTERSTORE dest key1 key2 key3  # 交集存入 dest（建议用这个避免重复算）

# 并集（合并去重）
SUNION key1 key2
SUNIONSTORE dest key1 key2

# 差集（key1 有但 key2 没有的）
SDIFF key1 key2                  # "我有他没有"
SDIFFSTORE dest key1 key2
```

**SINTER / SUNION / SDIFF 的复杂度与风险：**

```
SINTER key1 key2 key3：
  复杂度：O(N*M)，N = 最小集合的元素数，M = 集合个数
  内部算法：
    1. 选最小集合为"驱动表"（比如 key3 只有 100 个元素）
    2. 遍历驱动表每个元素，到其他集合中查是否存在
    3. 全在 → 加入结果

风险：
  如果三个集合各有 100 万元素 → 0.1M × 2 = 20w 次 dict 查找
  → 每百万次查找约 10-50ms → 可能成为大瓶颈

减小风险的策略：
  ① 永远用 STORE 版本存结果，不要每次请求都现算
  ② 对交并运算做缓存 + TTL
  ③ 设计层面避免超大集合
```

### 4.7 Set 使用场景详解

#### 场景一：点赞/收藏/已读 —— "Set = 天然的幂等去重"

```java
// 点赞（幂等）
boolean liked = jedis.sadd("likes:article:123", "user:456") == 1;
// liked = true → 新点赞；false → 已经点过（防止重复）

// 取消点赞
jedis.srem("likes:article:123", "user:456");

// 是否已点赞
boolean hasLiked = jedis.sismember("likes:article:123", "user:456");

// 点赞总数
long likeCount = jedis.scard("likes:article:123");
```

**为什么不用 String INCR 计数？** INCR 只管数量不管"谁点了"。你要防止同一个人点两次，必须查有没有点过 → Set 的 SADD 返回 0 天然阻止重复。

#### 场景二：共同好友 / 推荐关注 —— "SINTER 和 SDIFF"

```java
// A 和 B 的共同好友
Set<String> common = jedis.sinter("following:user:A", "following:user:B");
// → [C, D] → 推荐给另一个用户："你和 B 都关注了 C、D"

// 推荐关注：A 关注了但 B 没关注的
Set<String> suggest = jedis.sdiff("following:user:A", "following:user:B");
// → [E, F] → B 可以对 E 和 F 展示"你可能认识的人"
```

**为什么是 Set？** 两个集合交集取共同——数据结构课就该用 Set。

#### 场景三：抽奖 / 随机抽取 —— "SPOP"

```java
// 从候选用户池中随机抽取 3 个中奖者
Set<String> winners = jedis.spop("lottery:candidates", 3);
// 被弹出的用户不会再次参与（天然的"一人只能中一次"）

// 或者用 SRANDMEMBER（不删除）
Set<String> lucky = jedis.srandmember("lottery:candidates", 3);
// 下次还可以再选（允许重复中奖的抽奖）
```

#### 场景四：标签系统 —— "Set 天然支持标签的增删和交集检索"

```java
// 给文章打标签
jedis.sadd("article:123:tags", "redis", "database", "backend");
// 给标签建立反向索引
jedis.sadd("tag:redis:articles", "123");
jedis.sadd("tag:database:articles", "123");

// 查询同时带有 "redis" 和 "database" 标签的文章
Set<String> articles = jedis.sinter("tag:redis:articles", "tag:database:articles");
```

**反向索引的威力：** 一个文章有多个标签，每个标签又索引到多篇文章 → SINTER 两个标签 = 找到满足两个标签的所有文章 → 比 SQL `WHERE tag IN ('a','b')` 更贴合多对多场景。

#### 场景五：活跃用户 / DAU 精确去重

```java
// 用 Set 存每天的精确活跃用户（百万级可以，亿级建议用 HyperLogLog）
jedis.sadd("dau:2026-05-25", "user:12345");

// 7 天都活跃的用户（连续活跃）
jedis.sinterstore("dau:week-21", 
    "dau:05-25", "dau:05-24", "dau:05-23", 
    "dau:05-22", "dau:05-21", "dau:05-20", "dau:05-19");
long consecutiveUsers = jedis.scard("dau:week-21");

// 新用户（今天活跃但昨天不活跃）
jedis.sdiffstore("dau:new-05-25", "dau:05-25", "dau:05-24");
long newUsers = jedis.scard("dau:new-05-25");
```

### 4.8 Set 面试题

**Q1: intset 的二分查找 O(logN)，dict 的哈希查找 O(1)——但为什么 intset 在 ≤512 时反而更快？**

答：大 O 不计算常数因子。intset 的 512 个整数在 2KB 的连续内存中，二分查找 9 次比较（log₂512），每次比较都在 L1/L2 cache 中 → 总计约 20ns。而 dict 的哈希查找需要一次哈希计算 + 一次随机内存访问（指针跳转），如果目标不在 cache 中 → 50~100ns。所以 O(logN) < O(1) 在小 N 下成立——**CPU cache 决定了常数因子**。

**Q2: 大集合做 SINTER 会阻塞吗？如何避免？**

答：会。SINTER 复杂度 O(N×M)，N 是最小集合大小。如果两个 100 万集合取交集，遍历 100 万 × dict 查找 ≈ 可能数百毫秒 → 严重阻塞。避免：① SINTERSTORE 结果存到新 key（加 TTL），不要每次现算；② 预估集合规模，超过几千的不用 SMEMBERS/SINTER；③ 分层——热点数据的集合拆分到多个小 Set。

**Q3: SMEMBERS 和 SSCAN 的区别？**

答：SMEMBERS 原子返回所有元素，阻塞 O(N)。SSCAN 分批返回，非阻塞，但不原子——遍历过程中元素的增删可能导致遗漏或重复。遍历大 Set 只能用 SSCAN。

**Q4: SPOP 和 SRANDMEMBER 哪个适合抽奖？**

答：SPOP（弹出删除）适合"不重复中奖"的抽奖——中过一次就被踢出池子。SRANDMEMBER 适合"随机推荐展示"——推荐不删除数据。如果抽奖池很大（百万级），SPOP 有内存分配开销（从 dict 删 entry），但几率很小。

---

## 第五章：ZSet（有序集合）—— 跳表的精妙

### 5.1 先看一个具体例子——从外部命令到内部存储

假设你执行这些命令：

```bash
ZADD rank 100 "alice"
ZADD rank 95  "bob"
ZADD rank 110 "charlie"
```

Redis 内部生成了什么样的数据结构？答案是 **三样东西协同工作**：

```
┌──────────────────────────────────────────────────────────┐
│                       ZSet (总控)                        │
│                                                          │
│  ┌─────────────────┐       ┌─────────────────────────┐  │
│  │     dict         │       │     skiplist (跳表)      │  │
│  │                  │       │                         │  │
│  │ "alice"  → 100   │       │  [HEAD]                │  │
│  │ "bob"    → 95    │       │    │                    │  │
│  │ "charlie"→ 110   │       │    ▼                    │  │
│  │                  │       │  bob(95) ← 最低分       │  │
│  │ 作用：O(1) 查分   │       │    │                    │  │
│  │ ZSCORE 直接查    │       │    ▼                    │  │
│  └─────────────────┘       │  alice(100)             │  │
│                            │    │                    │  │
│                            │    ▼                    │  │
│                            │  charlie(110) ← 最高分   │  │
│                            │                         │  │
│                            │ 作用：O(logN) 范围查询    │  │
│                            │ ZRANGE / ZRANK 靠它      │  │
│                            └─────────────────────────┘  │
│                                                          │
│  ★ dict 和 skiplist 中的 "alice"、"bob"、"charlie"      │
│    是同一份 SDS 数据，两个结构共享指针，不占双份内存       │
└──────────────────────────────────────────────────────────┘
```

**为什么需要两个结构？只用一个不行：**

| 操作 | 只有 dict | 只有 skiplist |
|------|----------|-------------|
| ZSCORE "alice" | ✅ O(1) 直接查 | ❌ O(logN) 在跳表中搜索 |
| ZRANGE 0 2 | ❌ dict 无序，需要全部取出来排序 | ✅ O(logN+3) |
| ZRANK "alice" | ❌ 不知道怎么算排名 | ✅ O(logN) 通过 span 算 |

**结论：** dict 负责"按名字查分数"（类似 HashMap），skiplist 负责"按分数排序和查排名"（类似 TreeMap）。member 的 SDS 字符串在两个结构之间**共享同一份内存**，没有浪费。

---

### 5.2 跳表（skiplist）长什么样？

#### 5.2.1 先从最简单的排好序的单链表开始

```
想快速查找 30：

  HEAD → 10 → 20 → 30 → 40 → 50
  从头一个个往后找 → 找 4 步 → O(N)
```

**核心痛点：链表不能像数组一样二分查找。** 怎么办？**给它加一层"快速通道"：**

```
加了一层索引的链表（跳表雏形）：

  level 1: HEAD ──────────────▶ 30 ──────────────▶ 50    ← 快速通道
  level 0: HEAD → 10 → 20 → 30 → 40 → 50              ← 全量链表

查找 30：
  从 level 1 开始：HEAD → 30 → 找到了！2 步，vs 原来 4 步
查找 25：
  level 1: HEAD → 30（30 > 25，说明走过了）→ 回退到 HEAD
  level 0: HEAD → 10 → 20 → 30（30 > 25）→ 25 不存在
  → 找不到，但只走了 3 步，原来要走 4 步
```

**关键直觉——"高速公路"：** 高层跳过更多节点，低层是完整链表。查找时**从高层往下层**走，每层都尽量往前进，直到 next 比目标大才降层。

#### 5.2.2 Redis 的跳表有多少层？

Redis 不是每两个节点选一个做索引（那是固定结构），而是 **随机层数**——每个节点插入时抛硬币决定它有基层：

```
每个节点插入时随机生成层数：
  level=1：概率 75%
  level=2：概率 75% × 25% = 18.75%
  level=3：概率 75% × 25%² = 4.69%
  level=4：概率 75% × 25%³ = 1.17%
  ...更高层：越来越稀

大部分节点只有 1 层，少数节点有高层。
这就像在每个分数段随机选"代表"往上层站——越往上节点越少。
```

**这样做的效果：** 一个 1000 个节点的跳表可能长这样：

```
level 3: HEAD ─────────────────────────────────► NULL  (0个节点)
level 2: HEAD ──────────► 30 ─────────────────► NULL  (1个节点)
level 1: HEAD ──► 10 ──► 30 ──► 55 ──► 70 ──► NULL  (4个节点)  
level 0: HEAD ► 5► 10► 15► 20► 30► 40► 55► 60► 70► 95► NULL  (10个节点)
                                                          ← 完整链表
```

- level 0 总是包含所有节点（完整有序链表）
- 越往上节点越少（只有被随机选中"升层"的节点才出现在高层）
- 查找永远从**最高层**开始，往下走

---

### 5.3 Redis 跳表的三件套（Java 类比）

Redis 的跳表由三个 C 结构体构成，我用 Java class 来类比：

```java
// ========== 总控 ==========
// 对应 Redis 的 zset 结构体
class ZSet {
    Dict<String, Double> dict;    // member → score，O(1) 查分
    SkipList skiplist;            // 跳表，O(logN) 排序和排名
}

// ========== 跳表本身 ==========
// 对应 Redis 的 zskiplist 结构体
class SkipList {
    SkipListNode header;    // 头节点（固定 64 层，不计入 length）
    SkipListNode tail;      // 尾节点
    int length;             // 节点总数（不含 header）
    int maxLevel;           // 当前最高层数（比如节点最高 4 层，maxLevel=4）
}

// ========== 跳表节点 ==========
// 对应 Redis 的 zskiplistNode 结构体
class SkipListNode {
    String member;          // 成员名（SDS，和 dict 的那份共享）
    double score;           // 分值
    
    SkipListNode backward;  // 后退指针（只有 level 0 有，用于 ZREVRANGE 反向遍历）
    
    Level[] levels;         // 该节点的"层数组"，每一层有两个信息：
                            //   forward → 这一层上，下一个节点是谁
                            //   span    → 这一层上，从当前跳到 forward，中间隔了几个节点
}

class Level {
    SkipListNode forward;   // 这一层上，下一个节点
    int span;               // 这一层上，跳几步能到 forward
}
```

**关键理解一：** 每个节点的高度不同。一个 level=2 的节点有 `levels[0]` 和 `levels[1]`；一个 level=3 的节点有 `levels[0]`、`levels[1]`、`levels[2]`。**越高层的节点越稀少。**

**关键理解二：** `span` 是 Redis 跳表特有的设计——记录了"在这一层上，从当前节点跳到 forward 节点，中间跳过了多少个 level-0 的节点"。这是 ZRANK（查排名）能做到 O(logN) 的秘密所在。

---

### 5.4 用具体例子理解 span 和 ZRANK

假设此刻跳表中有 5 个节点，我们用一个具体例子把 span 展示清楚：

```
跳表中现有 5 个节点：alice(90), bob(95), charlie(100), dave(100), eve(110)

level 2: HEAD ────────(span=3)────────► charlie(100) ────────► NULL
level 1: HEAD ──(1)──► bob(95) ──(1)──► charlie(100) ──(2)──► eve(110)
level 0: HEAD ► alice ► bob ► charlie ► dave ► eve ► NULL
              (span=1)(span=1)(span=1)  (span=1)(span=1)

span 的含义（以 level 1 为例）：
  HEAD.level[1].span = 1 → 从 HEAD 跳到 bob，中间跳过了 1 个节点(alice)
  bob.level[1].span   = 1 → 从 bob 跳到 charlie，中间跳过了 1 个节点(没有，就是紧邻)
  charlie.level[1].span = 2 → 从 charlie 跳到 eve，中间跳过了 2 个节点(dave)
  
level 0 的 span 都是 1（因为 level 0 是完整链表，每个节点紧挨着）
```

**现在用这个例子跟踪 ZRANK "eve" 的过程：**

```
ZRANK rank "eve" 内部执行：

1. 从 dict 查 "eve" → score=110 → O(1)

2. 在跳表中按 (score=110, member="eve") 搜索，
   搜索过程中累计 rank（前面有多少个节点）：

   从 HEAD 开始，当前 rank = 0：

   level 2: HEAD → charlie(100) → 100 < 110，继续
            累计 rank += HEAD.level[2].span = 0 + 3 = 3
            x = charlie

            charlie.level[2].forward = NULL → 降层

   level 1: x=charlie, charlie.level[1].forward = eve(110)
            110 ≤ 110 且 member="eve" ≥ "eve" → 目标！不前进，降层
            （这里 score 相同，比较 member 字典序）

   level 0: x=charlie, charlie.level[0].forward = dave(100)
            100 < 110，继续
            累计 rank += charlie.level[0].span = 3 + 1 = 4
            x = dave

            dave.level[0].forward = eve(110)
            找到了！
            累计 rank += dave.level[0].span = 4 + 1 = 5

3. rank = 5（从 0 开始，即 eve 是第 6 个节点）
   返回 rank

如果没有 span，必须从 HEAD 一路数到 eve（遍历 6 次）。
有了 span，只需在搜索路径上累加 3 个 span 值——O(logN)。
```

**span 的本质：** 把"遍历 level 0 链表的 O(N) 计数"变成了"在搜索路径上顺手累加"的操作，没有额外开销。

---

### 5.5 ZADD 的完整执行过程（一步一步来）

假设当前跳表已有 3 个节点，高分到低分排列（level 0 是从小到大）：

```
已有数据：alice(100), bob(95), dave(110)
跳表当前 maxLevel = 2

level 2: HEAD ──────────────► dave ──► NULL
level 1: HEAD ──► bob ──► alice ──► dave ──► NULL
level 0: HEAD ► bob ► alice ► dave ► NULL
              (95)  (100)   (110)

现在执行：ZADD rank 105 "charlie"
```

**步骤 1：在跳表的每一层找到"插入位置的前驱节点"**

从最高层 (level 2) 开始往下走，对每一层记录"新节点应该插在谁后面"：

```
当前 x = HEAD:

level 2: HEAD.forward = dave(110), 110 > 105 → dave 比 charlie 大
         → 新节点应该插在 HEAD 后面（这一层）
         → update[2] = HEAD  ← 记录：level 2 层，HEAD 是新节点的前驱

level 1: x = update[2] = HEAD
         HEAD.forward = bob(95), 95 < 105 → 继续前进
         x = bob
         bob.forward = alice(100), 100 < 105 → 继续前进
         x = alice
         alice.forward = dave(110), 110 > 105 → dave 比 charlie 大
         → update[1] = alice  ← 记录：level 1 层，alice 是新节点的前驱

level 0: x = update[1] = alice
         alice.forward = dave(110), 110 > 105
         → update[0] = alice  ← 记录：level 0 层，alice 是新节点的前驱

最终 update[] = [alice, alice, HEAD]
                level0   level1   level2
```

**步骤 2：随机生成新节点的层高**

```
zslRandomLevel():
  level = 1
  随机数 < 0.25 → level = 2
  随机数 < 0.25 → level = 3
  随机数 ≥ 0.25 → 停止，返回 level=3

假设这次随机到了 level=3，即新节点有 3 层（levels[0],[1],[2]）
```

**步骤 3：创建新节点并分层插入**

```
创建 SkipListNode:
  member = "charlie" (SDS)
  score  = 105.0
  levels = new Level[3];  // level 0, 1, 2

分层插入（类似链表"插到中间"的操作，但每一层分别做）：

level 0: charlie 插在 alice 和 dave 之间
         charlie.levels[0].forward = dave
         alice.levels[0].forward = charlie
         更新 span: charlie.span = 1, alice.span = 1

level 1: charlie 插在 alice 和 dave 之间（同上）
         charlie.levels[1].forward = dave
         alice.levels[1].forward = charlie

level 2: charlie 现在是这一层唯一的节点（之前 HEAD 直接连 dave）
         charlie.levels[2].forward = dave
         HEAD.levels[2].forward = charlie
         更新 span: charlie.span = 1, HEAD.span = 3 (跳过了 bob, alice, charlie 共 3 个)

如果 charlie 的 level 超过了当前 maxLevel → maxLevel 更新为 3
```

**步骤 4：更新 backward 指针（level 0 专用）**

```
charlie.backward = alice   (前驱)
dave.backward = charlie    (charlie 的后继需要更新)
```

**插入后的完整跳表：**

```
level 2: HEAD ──────────────► charlie(105) ──► dave(110) ──► NULL
level 1: HEAD ──► bob(95) ──► alice(100) ───► charlie ────► dave ──► NULL
level 0: HEAD ► bob ► alice ► charlie ► dave ► NULL
              (95)  (100)   (105)     (110)
```

**步骤 5：同时在 dict 中插入 `"charlie" → 105`**

ZADD 完成！整个过程在 Redis 主线程一次完成，单线程保证整个过程不被其他命令打断。

---

### 5.6 ZSCORE 和 ZRANK 的实现（一句话总结）

```
ZSCORE rank "charlie"：
  → 直接查 dict["charlie"] → 105  ← O(1)，不经过跳表

ZRANK rank "charlie"：
  → 在跳表中搜索 (105, "charlie")
  → 搜索过程中累加 span 值：
      HEAD.level[2].span +
      HEAD.level[0].span（经过 bob）+
      bob.level[0].span（经过 alice）+
      alice.level[0].span（到 charlie）
      = 3 + 1 + 1 + 1 = 位次 5（从 0 开始，即第 6 位）
  → O(logN)
```

---

### 5.7 ZADD 更新已有 member 的处理

```
ZADD rank 120 "alice"    ← alice 已存在，score 从 100 → 120

处理过程：
  1. 在 dict 中查到 alice 已存在，score=100
  2. 因为 score 变了（100 ≠ 120）→ 不能原地修改 score
     因为排序位置从 (alice 在 charlie 之前) 变成 (alice 在 dave 之后)
  3. 先从 skiplist 中删除 alice 节点（断开各层指针，decrease span）
  4. 在 dict 中更新 score = 120
  5. 重新执行一遍 ZADD 的插入流程（生成新层高，在正确的新位置插入）

如果旧 score == 新 score → 什么都不做（NOP），直接返回 0
```

---

### 5.8 ZSet 的 listpack 编码（小数据）

```
小 ZSet（≤128 元素，每个 member ≤64 字节）→ 用一个 listpack：

listpack 内： [score1][member1][score2][member2][score3][member3]...
         按 score 从小到大排好序

ZADD：遍历找插入位置 O(N) → memmove 后边数据 → 插入
ZSCORE：遍历找 member O(N) → 返回紧跟着的 score
ZRANGE 0 9：直接从头读 10 对 O(10)

N ≤ 128 时所有操作都是微秒级，内存仅仅是几十到几百字节。
数据一大就自动切换到 dict + skiplist，用户完全无感。
```

### 5.4 ZSet 常用命令详解

#### ZADD —— 最复杂的 Redis 命令之一

```bash
ZADD key [NX|XX] [GT|LT] [CH] [INCR] score member [score member ...]
```

| 参数 | 含义 |
|------|------|
| `NX` | member 不存在才添加（Not eXists）——只允许新增 |
| `XX` | member 已存在才更新（eXists）——只允许更新 |
| `GT` | 仅新 score > 当前 score 时更新（Greater Than） |
| `LT` | 仅新 score < 当前 score 时更新（Less Than） |
| `CH` | 返回值改为"被修改的 member 数"（而非新增数） |
| `INCR` | 递增模式，ZADD 直接当 ZINCRBY 用（仅限单 member） |

**ZADD 的返回值陷阱：**

```bash
ZADD rank 100 "alice"       # → 1（新增 1 个 member）
ZADD rank 200 "alice"       # → 0（member 已存在，更新 score，但不计入新增）

ZADD rank CH 200 "alice"    # → 1（CH 标志——计入被修改的 member 数）
```

**INCR 模式（原子地原子递增）：**

```bash
ZADD rank INCR 10 "alice"   # alice 的 score +10，返回新 score
                             # 等价于 ZINCRBY rank 10 alice，但一行搞定
```

**GT / LT 的使用场景：**
- `LT`：记录个人最佳成绩（只有新分数 < 旧分数才更新）——如跑男记录
- `GT`：记录最高分（只有新分数 > 旧分数才更新）——如游戏积分

#### ZREM / ZSCORE / ZRANK / ZREVRANK

```bash
ZREM key m1 m2 m3           # 删，返回删除数，O(M*logN)
ZSCORE key member           # 查分，O(1)（dict 查找）
ZRANK key member            # 正排名（从 0 起，分低的在前），O(logN)
ZREVRANK key member         # 逆排名（从 0 起，分高的在前），O(logN)
```

**排名的 0-based 注意：** ZRANK 返回 0 = 第一名（score 最小）。ZREVRANK 返回 0 = 第一名（score 最大）。前端展示时通常要 +1。

#### ZCARD / ZCOUNT

```bash
ZCARD key                   # 元素总数，O(1)
ZCOUNT key 1000 2000         # score 在 [1000, 2000] 内的元素数，O(logN)

# ZCOUNT 不含数据返回 = 仅计数，不传数据 → 高效
```

#### ZRANGE / ZREVRANGE 家族（Redis 6.2 重大重构）

```bash
# 按排名范围（传统方式）
ZRANGE key 0 9 WITHSCORES               # Top 10（score 从小到大）
ZREVRANGE key 0 9 WITHSCORES             # Top 10（score 从大到小）

# Redis 6.2+ 统一语法（按 score 范围——替代 ZRANGEBYSCORE）：
ZRANGE key 1000 2000 BYSCORE WITHSCORES LIMIT 0 10

# Redis 6.2+ 统一语法（按字典序范围——替代 ZRANGEBYLEX）：
ZRANGE key [a (c BYLEX   # 字典序范围：[a, c)，'['包含 '('不包含

# REV 后缀支持逆序（替代 ZREVRANGEBYSCORE/ZREVRANGEBYLEX）：
ZRANGE key 2000 1000 BYSCORE REV WITHSCORES LIMIT 0 10
```

**ZRANGE 的分页优化：** `LIMIT offset count` 不遍历 offset 条再取，而是在 skiplist 定位后直接跳，O(logN + offset + count)。但 offset 还是大（亿级），因为要维护 span 累加 offset 个节点——同 MySQL `LIMIT 1000000, 10` 的问题一样。

#### ZPOPMIN / ZPOPMAX —— 弹出操作

```bash
ZPOPMIN key 3              # 弹出 3 个 score 最低的元素（消费型）
ZPOPMAX key 5              # 弹出 5 个 score 最高的
```

#### ZINCRBY —— 原子增减分数

```bash
ZINCRBY key 10 "alice"     # alice +10 分，返回新 score
ZINCRBY key -5 "alice"     # alice -5 分（减少）
```

#### ZINTERSTORE / ZUNIONSTORE —— 集合运算

```bash
# 交集：alice 同时在 rank1 和 rank2 中，score 按 WEIGHTS 加权
ZINTERSTORE dest 2 rank1 rank2 WEIGHTS 2 1 AGGREGATE SUM

# 并集：所有出现过的 member 汇总，同 member 的 score 相加
ZUNIONSTORE dest 2 rank1 rank2 WEIGHTS 1 1 AGGREGATE MAX
```

**参数解释：**
- `WEIGHTS`：每个 ZSet 的 score 乘的权重。如 `WEIGHTS 2 1` = rank1×2 + rank2×1
- `AGGREGATE SUM/MIN/MAX`：同 member 的多个 score 怎么合并

**典型：多维度综合排行**

```bash
# 阅读权重 0.3 + 点赞权重 0.5 + 评论权重 0.2 → 综合热度
ZUNIONSTORE hot:daily 3 \
  rank:reads  rank:likes  rank:comments \
  WEIGHTS 0.3 0.5 0.2 AGGREGATE SUM
```

### 5.5 ZSet 使用场景详解

#### 场景一：实时排行榜 —— "ZSet 的诞生动机"

```java
// 玩家得分
redis.zadd("rank:game:daily", 9800, "player:1001");
redis.zadd("rank:game:daily", 9500, "player:1002");
redis.zincrby("rank:game:daily", 200, "player:1001");  // player1001 冲到 10000

// Top 10 排行
Set<ZSetOperations.TypedTuple<String>> top10 = 
    redis.opsForZSet().reverseRangeWithScores("rank:game:daily", 0, 9);

// 玩家排名（第几名）
Long rank = redis.opsForZSet().reverseRank("rank:game:daily", "player:1001");
// rank + 1 = 第几名

// 查看自己前后的排名（竞争区间）
Set<String> around = redis.opsForZSet()
    .reverseRange("rank:game:daily", rank - 2, rank + 2);  // 前后各 2 人
```

**百万用户榜单性能：** ZADD O(logN) ≈ 20 次跳表比较；ZREVRANGE 0 9 O(logN + 10) → 微秒级。不论多少用户，Top 10 永远瞬间返回。

#### 场景二：延迟队列 —— "score = 到期时间戳"

```java
// 生产者：10 秒后执行
long executeAt = System.currentTimeMillis() + 10000;
redis.zadd("delay:queue", executeAt, "task:send_sms:user123");

// 消费者：轮询到期任务
while (true) {
    long now = System.currentTimeMillis();
    Set<String> tasks = redis.zrangeByScore("delay:queue", 0, now, 0, 10);
    for (String task : tasks) {
        // ZREM 确认拿到（防重复消费）
        if (redis.zrem("delay:queue", task) > 0) {
            process(task);
        }
    }
    Thread.sleep(500);  // 轮询间隔
}
```

**为什么用 ZSet 而不是 RabbitMQ 延时队列？** ZSet 方案零依赖（不需要新的中间件），精确定时（score 毫秒级时间戳）。缺点：轮询有 CPU 开销、无 ACK 机制。

#### 场景三：优先级队列 —— "score = 优先级"

```java
// 高优先级任务（score 大 = 高优）
redis.zadd("queue:task", 100, "task:urgent");
redis.zadd("queue:task", 10, "task:normal");
redis.zadd("queue:task", 1, "task:low");

// 消费者优先处理高优先级
Set<String> task = redis.zpopmax("queue:task", 1);
// "task:urgent" 先被弹出（score=100 最高）
```

#### 场景四：滑动窗口限流 —— "score = 时间戳"

```java
/**
 * 每个用户每 60 秒最多 10 次请求（ZSet 滑动窗口）
 */
public boolean isAllowed(String userId) {
    String key = "rate:" + userId;
    long now = System.currentTimeMillis();
    long windowStart = now - 60000;  // 60 秒前
    
    String lua = 
        "redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1]) " +  // 清窗口外
        "local cnt = redis.call('ZCARD', KEYS[1]) " +
        "if cnt < tonumber(ARGV[2]) then " +
        "    redis.call('ZADD', KEYS[1], ARGV[3], ARGV[4]) " +
        "    redis.call('EXPIRE', KEYS[1], ARGV[5]) " +
        "    return 1 " +
        "end " +
        "return 0";
    
    Object result = jedis.eval(lua,
        Collections.singletonList(key),
        Arrays.asList(
            String.valueOf(windowStart), String.valueOf(10),
            String.valueOf(now), UUID.randomUUID().toString(),
            "61"
        ));
    return "1".equals(result.toString());
}
```

**为什么不用 String 的固定窗口（"1 分钟 10 次"）？** 固定窗口边界处可绕过去（0:59 秒 10次 + 1:00 秒 10次 = 2 秒内 20 次）。ZSet 滑动窗口以当前时间为窗口终点，平滑统计，没有漏洞。

### 5.6 ZSet 面试题

**Q1: 为什么是跳表而不是红黑树？**

答：（1）跳表实现约 200 行，红黑树 2000+ 行——简单即正确；（2）跳表的范围查询极度直观：找到起点顺着链表走就是；红黑树要做中序遍历（需要处理非叶子节点）；（3）跳表通过 span 天然支持 O(logN) 的 ZRANK；红黑树需要额外维护子树大小；（4）跳表的随机平衡比红黑树的旋转维护简单得多。

**Q2: ZSet 一定需要 dict + skiplist 两个结构吗？只用一个行不行？**

答：不行。只有 dict → ZRANGE 需要先获取所有 member 排序，O(NlogN) 还要分配内存。只有 skiplist → ZSCORE 查分需要 O(logN) 而非 O(1)。两个结构互补，且 member 的 SDS 对象在两个结构中共享指针，不会多占用内存。

**Q3: 跳表的层高是如何随机生成的？为什么用 1/4 而非经典跳表的 1/2？**

答：每层独立以 1/4 概率递增——大部分节点只有 1~2 层。期望高度 = 1/(1-1/4) = 1.33 层。1/2 会生成更多高层节点（期望 2 层），插入时需要多更新指针，多出来的查找效率提升微乎其微。Redis 选择 1/4，让跳表更"矮胖"，插入更快。

**Q4: ZRANGE 大分页（offset 很大）会慢吗？**

答：会。ZRANGE 100000 100009 虽然只返回 10 条，但定位起点时需要在 skiplist 的 level 0 链表累加 span 走过 10 万个节点 → O(logN + offset) ≈ O(100000)，相当于遍历。解决方案：① 做好分页限制（每页最多 100 条，offset 不超过 1000）；② 大深度分页用 SCROLL（但 ZSet 没有 scroll）；③ 替换为按 score 范围查询（ZRANGEBYSCORE），定位是 O(logN)。

**Q5: ZSCORE 返回的是 double，精度在超高并发计分时会有问题吗？**

答：double 的 53 位尾数有效范围是 ±2^53 ≈ ±9×10^15。如果积分累加到超过这个范围（约几千万亿），末位精度丢失。比如 9,007,199,254,740,991（2^53）存不进 double 的最低整数位——`ZINCRBY key 1` 不再增加。遇这种极端场景可以让 Redis 存储 String 自己用 BigDecimal 管理分数，但一般不需要。

---

## ⭐️ 全类型总结

| 类型 | 小数据编码 | 大数据编码 | 切换条件 |
|------|-----------|-----------|---------|
| **String** | int / embstr | raw | 非整数 or >44字节 |
| **List** | quicklist（统一实现） | ← 不分大小 | fill=8KB/节点 |
| **Hash** | listpack | dict (hashtable) | field>512 or 单 value>64B |
| **Set** | intset（有序数组） | dict (空value) | >512 元素 or 非整数 |
| **ZSet** | listpack | dict + skiplist | >128 元素 or 单 member>64B |

**核心设计哲学：**
- **小数据用紧凑结构（listpack/intset/embstr）**——连续内存、无指针、cache 友好
- **大数据用高效结构（dict/skiplist/quicklist+listpack）**——保证操作复杂度可控
- **编码从紧凑结构升到高效结构是单向不可逆的**（分配代价大，没必要降回来）

---

## ⭐️ 综合面试题

**Q1: Redis 为什么每种类型搞多种底层编码？和 Java 的面向接口编程有什么相通之处？**

答：Java 中 `List<String> list = new ArrayList<>()`——你用的是 List 接口，底层可能是 ArrayList 也可能是 LinkedList。Redis 也是：Hash 类型对外统一接口（HSET/HGET/HGETALL），底层到底是 listpack 还是 dict ——使用者不需要知道。这本质上是 **策略模式**——小数据用紧凑实现（省内存），大数据换高效实现（保性能），运行中自动切换。

**Q2: Redis Object 的 encoding 字段有 4 bit，最多表示 16 种编码——现在用了哪些？**

答：`OBJ_ENCODING_RAW(0)`、`OBJ_ENCODING_INT(1)`、`OBJ_ENCODING_HT(2)`（即 dict）、`OBJ_ENCODING_ZIPLIST(5)`（旧版）、`OBJ_ENCODING_LINKEDLIST(4)`（已废弃）、`OBJ_ENCODING_INTSET(6)`、`OBJ_ENCODING_SKIPLIST(7)`、`OBJ_ENCODING_EMBSTR(8)`、`OBJ_ENCODING_QUICKLIST(9)`、`OBJ_ENCODING_STREAM(10)`、`OBJ_ENCODING_LISTPACK(11)`。11 种用了 4 bit 的 16 个位置。

**Q3: DEL 一个大 Hash（500 万 field）会发生什么？**

答：DEL 在主线程释放 dict（遍历所有 dictEntry 逐个 free），500 万的释放可能阻塞 Redis **几百毫秒到秒级**。解决：Redis 4.0+ 提供 `UNLINK` 异步删除（主线程只把 key 从键空间摘掉，实际内存释放交给后台线程 `BIO_LAZY_FREE`）。`DEL` 是同步阻塞的，`UNLINK` 是非阻塞的。这是大厂生产环境排查"Redis 为什么偶发 latency spike"的一个高频原因。

**Q4: 使用 `redis-cli --bigkeys` 扫描大 Key 会阻塞吗？**

答：不会。`--bigkeys` 用 `SCAN` 系列命令（分批返回），不会阻塞主线程。但它只统计元素数（SCARD/HLEN/LLEN/ZCARD），不统计内存占用。要看真实内存用 `MEMORY USAGE key`。

---

*Created: 2026-05-25 · Updated: Java 程序员视角全量重写 · Category: 03-Redis数据类型*
