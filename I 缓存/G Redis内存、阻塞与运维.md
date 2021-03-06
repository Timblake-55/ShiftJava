[TOC]

### 一、Redis内存

Redis 就是**内存数据库**，所以理解 Redis 的内存是非常关键的。

#### 内存消耗

内存消耗分为**自身内存**消耗和**子进程内存**消耗。

##### 1. 内存使用统计

内存统计信息通过 **info memory** 指令查看。重点指标如下。

|           属性名            |                           属性说明                           |
| :-------------------------: | :----------------------------------------------------------: |
|       **used_memory**       | Redis 分配器分配的内存**总量**，也就是内部存储的所有数据内存占用量 |
|     **used_memory_rss**     |    从**操作系统**的角度显示 Redis 进程占用的物理内存总量     |
|    **used_memory_peak**     |       内存**使用的最大值**，表示 used_memory **峰值**        |
|   used_memory_peak_human    |            以**可读的格式**返回 used_memory peak             |
|       used_memory_lua       |                   Lua 引擎锁消耗的内存大小                   |
| **mem_fragmentation_ratio** |   used_memory_rss/used_memory **比值**，表示**内存碎片率**   |
|        mem_allocator        |           Redis 所使用的内存分配器 默认为 jemalloc           |

重点关注：**used_memory_rss**，**used_memory**，及两者比值 **mem_fragmentation_ratio**。

- 当 **mem_fragmentation_ratio >1** 时，说明 used_memory_rss-used_memory 多出的部分**内存并没有用于数据存储**，而是被内存碎片所消耗，如果两者相差很大，**说明内存碎片率严重**。

- 当 **mem_fragmentation_ratio <1** 时，这种情况一般出现在操作系统把 Redis 内存**交换到硬盘导致**，由于硬盘速度远远慢于内存，Redis 性能会变得很差，甚至僵死。

##### 2. 内存消耗划分

Redis 进程内存消耗主要有四个部分：**自身内存、对象内存、缓冲内存、内存碎片**。

<img src="assets/image-20200426234244499.png" alt="image-20200426234244499" style="zoom:67%;" />

###### (1) 自身内存

自身内存很小。

###### (2) 对象内存

是占用最多的内存，就是保存用户的**所有数据**。所有的数据都采用 key-value 的方式进行存储。键对象都是字符串，所以应该避免过长的键。不同数据结构占用内存不同。

###### (3) 缓冲区内存

缓冲内存就是**缓冲区**内存，是实现功能很重要的一部分。缓冲内存主要包括：**客户端缓冲、复制积压缓冲区、AOF 缓冲区**。

<img src="assets/image-20200426235212679.png" alt="image-20200426235212679" style="zoom:67%;" />

**① 客户端缓冲区**：客户端缓冲指的是所有接入到 Redis 服务器 TCP 连接的**输入输出缓冲**。**输入缓冲**无法控制，最大空间为1G，如果超过将断开连接。**输出缓冲**通过参数 client-output-buffer-limit 控制。客户端分为普通客户端、从客户端、订阅客户端，配置情况不同。

**② 复制积压缓冲区**：Redis2.8 以后提供了一个**可重用的固定大小缓冲区用于实现部分复制**功能，根据 repl-backlog-size 参数控制，默认 1MB。对于复制积压缓冲区**整个主节点只有一个**，所有的从节点**共享**此缓冲区，因此可以**设置较大**的缓冲区空间，如 100MB，这部分内存投入是有价值的，可以**有效避免全量复制**。

**③ AOF缓冲区**：这部分空间用于在 **AOF 重写期间保存最近的写入命令**。此空间的消耗用户无法控制，消耗的内存取决于AOF 重写时间和写入命令量，这部分空间占用通常很小。

###### (4) 内存碎片

为了更好的管理和重复利用内存，分配内存策略一般采用**固定范围的内存块**进行分配。在 64 位系统中将内存空间划分为：小、大、巨大三个范围。每个范围内又划分为多个小的**内存块单位**。比如保存 5KB 对象可能会采用 8KB 的块存储，而剩下的 3KB 空间变为了**内存碎片**不能再分配给其他对象存储。

正常的碎片率 **mem_fragmentation_ratio** 在 **1.03** 左右。

> **高内存碎片率的原因？**

当存储的数据**长短差异较大**时候，以下场景容易出现**高内存碎片**问题：

- **频繁做更新操作**，例如频繁对已存在的键执行 append，setrange 等更新操作。

- 大量过期**键删除**，键对象过期删除后，释放的空间无法得到充分利用，导致碎片率上升。

> **出现高内存碎片率的解决方法？**

- **数据对齐**：尽量做数据对齐，比如数据尽量采用数字类型或者固定长度字符串等，但是这要看具体业务。

- **安全重启**：重启节点可以做到内存碎片重启整理，因此可以利用高可用架构，如 sentinel 或者 cluster ，将碎片率过高的主节点转换为**从节点**，进行安全重启。

##### 3. 子进程内存消耗

主要是指执行 **RDB、AOF 重写**时 Redis 创建的**子进程内存**消耗。Redis 执行 fork 操作产生的子进程内存占用量对外表现为与父进程**相同**，理论上需要**一倍的物理内存**来完成重写操作。但 Linux 具有**写时复制**技术，父子进程会**共享相同的物理内存页**，当**父进程处理写请求**时会对**需要修改的页复制出一份副本完成写操作**，而子进程依然**读取 fork 时整个父进程的内存快照**。

子进程内存消耗总结如下：

- Redis 产生的子进程并**不需要消耗 1 倍**（**理论上是 1 倍但是一般不需要这么多**）的父进程内存，实际消耗根据期间写入命令量决定，但是依然要预留出一些内存防止溢出。
- 需要设置 sysctl vm.overcommit_memory=1 允许内核可以分配所有的物理内存，防止 Redis 进程 执行 fork 时候因系统剩余内存不足而失败。
- 排查当前系统是否支持并开启 THP，如果开启建议**关闭**，防止 copy-on-write 期间内存过度消耗。



#### 内存管理

Redis 主要通过**控制内存上限和回收策略**来实现内存管理。

##### 1. 设置内存上限

Redis 使用 **maxmemory** 参数限制**最大可用内存**。限制内存目的：

1. 用于缓存场景，当超出内存上限 maxmemory 时使用 LRU 等**删除策略释放空间**。
2. 防止所用内存**超过**服务器物理内存。

需要注意的是，maxmemory 限制的是 Redis **实际使用的内存量**，也就是 used_memory 统计项对应内存。由于**内存碎片率**的存在，**实际消耗**的内存可能会比 maxmemory 设置的**更大**，实际使用时要小心这部分内存**溢出**。

Redis 的内存上限可以通过 **config set maxmemory** 进程**动态修改**，即修改最大可用内存，实现动态伸缩 Redis 内存。

##### 2. 内存回收策略

内存回收机制主要体现在两个方面：

- 删除到达**过期**时间的键对象。
- 内存使用达到 **maxmemory** 上限时触发内存溢出控制策略。

###### (1) 删除过期键对象

精准维护每个键过期时间会导致消耗大量的 CPU，对于单线程的 Redis 来说**成本过高**，因此采用**惰性删除**和**定时任务删除**机制实现过期键的内存回收。

**==惰性删除==：**用于当客户端**读取带有超时属性**的键时，如果**已经超过键设置的过期时间**，会**执行删除操作并返回空**，这种策略是出于节省 CPU 成本考虑，不需要单独维护 TTL 链表来处理过期键的删除。但是单独用这种方式**存在内存泄露的问题**，当过期键**一直没有访问**将无法得到及时的删除，从而导致内存不能及时释放。

**==定时删除任务==：**Redis 内部维护一个**定时任务**，默认每秒运行 **10 次**（通过配置 hz 控制）。定时任务中删除过期键逻辑采用了**自适应算法**，根据键的过期比例，使用**快慢两种速率**模式回收键。可以用作惰性删除的补充。从每个数据库随机检查 20 个键如果超过 25% 的键过期了则循环执行回收逻辑直到不足 25% 或者运行超时为止。流程如下：

<img src="assets/image-20200429202654432.png" alt="image-20200429202654432" style="zoom:87%;" />

###### (2) 内存溢出控制策略

> MySQL 里有 2000w 数据，Redis 中只存 20w 的数据，如何保证 Redis 中的数据都是**热点数据?**

内存使用达到 **maxmemory 上限**时触发内存溢出控制策略。具体由 **maxmemory-policy** 参数控制。

Redis 支持 6 种策略。

|         策略         |                             说明                             |
| :------------------: | :----------------------------------------------------------: |
|      noeviction      |     **默认**策略，**拒绝写入**，不删除数据，只响应读操作     |
|  **volatitle-lru**   | 根据 **LRU 算法**删除**过期的键**，直到腾出足够空间，如果**没有可删除的键**则会**退到默认策略** |
|     allkeys-lru      | 根据 **LRU 算法**删除键，不管是否超时，**直到腾出足够空间**  |
|    allkeys-random    |           **随机删除所有键**，直到腾出足够空间为止           |
| **volatitle-random** |             **随机删除过期键**，直到腾出足够空间             |
|  **volatitle-ttl**   | 根据键对象的 TTL，删除**最近要过期**的键。如果没有可删除的键则会退到默认策略 |

如果**内存频繁溢出，会频繁触发内存回收操作，非常影响性能**。如果当前 Redis 有从节点 ，回收内存操作对应的删除命令会**同步**到从节点，导致**写放大**的问题。

#### 内存优化

##### 1. RedisObject对象

Redis 存储所有**值对象**在内部定义为 **redisObject** 结构体。Redis 存储数据都采用 redisObject 封装。理解 redisObject 对内存优化非常有帮助。

<img src="assets/image-20200429203916651.png" alt="image-20200429203916651" style="zoom:83%;" />

字段含义：

- **type**：表示当前对象使用的**数据类型**。
- **encoding**：表示 Redis 内部**编码类型**。
- **lru**：记录对象最后一次**被访问**的时间。用于辅助 LRU 算法实现内存回收。
- **refcount**：记录当前对象**被引用**的次数，用于通过引用次数回收内存。
- ***ptr**：与对象的数据**内容相关**，如果是整数，直接存储数据；否则表示指向数据的**指针**。

高并发写入场景中，在条件允许情况下，建议字符串**长度控制在 39 字节**以内，减少**创建 redisObject** 内存分配次数，从而提高性能。

##### 2. 缩减键值对象

**减少内存**最直接的就是缩减 key 和 value **长度**。常见需求是把业务对象**序列化**成二进制数组放入 Redis。所以可以采用**高效**的序列化工具降低数组大小。也可以使用一些压缩算法对值压缩。

##### 3. 共享对象池

**共享对象池**是指在 Redis 内部维护 **[0-9999] 的整数对象池**。创建大量的**整数**类型 redisObject 存在内存开销，每个redisObject 内部结构至少占用 16 个字节，甚至超过了整数自身空间消耗。所以 Redis 内存维护一个【0-9999】整数对象池，用于**节约内存**。除了整数值对象，其他类型如 list、hash、set、zset 内部元素也可以使用整数对象池。因此开发中在满足需求的前提下，尽可能使用整数对象以节省内存。

可见当数据**大量使用【0-9999】的整数**时，共享对象池可以节约大量内存。当多一个引用时，数据的**引用数**会 + 1。

**注意**：当设置 maxmemory 并**启用 LRU 相关淘汰策略**：volatile-lru，allkeys-lru 时，Redis **禁止使用**共享对象池。

> **为啥开启maxmemory和LRU淘汰策略后对象池无效？**

LRU 算法晓获取对象**最后被访问时间**，以便淘汰**最长未访问**数据，每个对象最后访问时间存储在 redisObject 对象的 **lru 字段**。对象共享意味着多个引用共享**同一个** redisObject  这时 lru 字段也会**被共享**，导致**无法获取**每个对象的最后访问时间。如果没有设置 maxmemory，直到内存被用尽 Redis 也不会触发内存回收，所以共享对象池可以正常工作。

综上所述，**共享对象池与 maxmemory+LRU 策略冲突，使用时需要注意**。

此外，对于 **ziplist 编码**的值对象，即使内部数据为整数也**无法**使用共享对象池，因为 ziplist 使用**压缩且内存连续**的结构，对象共享判断成本过高。

> **为啥只有整数对象池？**

首先整数对象池**复用的几率最大**，其次对象共享的一个关键操作就是判断相等性，Redis 之所以只有整数对象池，是因为整数**比较算法**时间复杂度 **O(1)**, 只保留一万个整数为了防止对象池浪费。

如果**字符串判断相等性**，时间复杂度 O(N) 特别是长字符串更消耗性能 (浮点数在Redis内部使用字符串存储)。

对于更复杂的数据结构如 hash/list 等，相等性判断需要 O(N^2)。这样的开销显然不合理，因此 Redis 只保留整数共享对象池。

##### 4. 字符串优化

字符串对象是 Redis 内部最常用的数据类型。所有的**键都是字符串**类型，值对象数据除了整数之外都使用字符串存储。

Redis 没有采用原生 C 语言字符串类型而是自己实现了字符串结构：**内部简单动态字符串(SDS)** 。SDS 存在空间预分配机制，字符串之所以采用预分配的方式是**防止修改操作需要不断重分配内存和字节数据拷贝**。但这同样会造成**内存浪费**。

**优化：**

- **尽量减少字符串频繁修改操作如 append、setrange等，改为直接使用 set 修改字符串，降低预分配带来的内存浪费和内存碎片化。**
- **字符串重构**：不一定把每份数据作为字符串整体存储，像 json 这样的数据可以使用 hash 结构，使用**二级结构**存储也能节省内存。

##### 5. 编码优化

所谓编码就是具体使用哪种**底层数据结构**来实现。编码不同将直接影响数据的内存占用和读写效率。使用 **object encoding {key}** 命令获取编码类型。

通过不同编码实现**效率和空间的平衡**。比如当我们的存储只有 10 个元素的**列表**，当使用**双向链表**数据结构时，必然需要维护大量的内部字段如每个元素需要：前置指针，后置指针，数据指针等，造成空间浪费，如果采用**连续线性内存结构**的压缩列表(ziplist) 将会节省大量内存而由于数据长度较小，存取操作时间复杂度即使为 O(N^2) 也可以满足要求。**ziplist 编码**是应用范围最广的一种，可以分别作为 hash/ list / zset /类型的底层数据结构实现。ziplist 就是**效率和空间的平衡**的重要体现。

编码类型转换在 Redis 写入数据时**自动完成**，这个转换过程是**不可逆**的。转换规则只能从**小内存编码向大内存编码**转换。

此外，**intset** 是集合 (set) 的类型编码的一种，当使用**整数集合**时尽量使用 intset 编码。

##### 6. 控制键的数量

当使用 Redis 存储大量数据时，通常会存在**大量键**，过多的键同样会消耗大量内存。

对于存储相同的数据内容利用 Redis 的数据结构**降低外层键的数量**，也可以节省大量内存，如使用 hash 结构。

对于**大量小对象**的存储场景，非常适合用 **ziplist 编码的 hash 类型**控制键的规模来降低内存。



### 二、Redis阻塞

Redis 是单线程架构，阻塞起来很难受啊，这里总结一下可能导致 Redis 阻塞的情况。主要分为内在原因和外在原因。

**内在原因**：不合理的使用 API 或数据结构、CPU 饱和、持久化阻塞等。

**外在原因**：CPU 竞争、内存交换、网络问题等。

#### 阻塞发现

如何发现阻塞问题？先寻找是不是 Redis 内在原因，如果不是的话，再定位阻塞发生可能的外在原因。

- **应用层可以加入异常监控逻辑**，比如引入日志系统，统计阻塞的情况。
- 可以借助一些 Redis 监控系统进行阻塞监控。
- 可以通过定期分析 Redis 日志的方式来发现阻塞的问题。
- 使用 Redis 提供的慢查询功能进行慢查询分析，如 slowlog get {n} 获取慢查询记录。但是慢查询只记录命令执行的时间，而阻塞原因是多种多样的。

#### 阻塞内在原因

##### 1. API或数据结构使用不合理

有的命令的时间复杂度是 **O(N)** 的，如果数据量比较大（**大对象**），那么就容易造成阻塞。**应该避免在大对象上执行复杂度超过 O(N) 的命令。**

这类原因适合用**慢查询记录**来进行查找。大对象的查找可以通过 **redis-cli --bigkeys** 命令进行扫描寻找。

解决方法：

① 修改为时间**复杂度较低**的算法。比如 hegtall 修改为 hmget，禁用 keys、sort 命令等。

② **调整大对象**。将大对象数据拆分成多个小对象，防止一次操作读取过多数据。

##### 2. CPU饱和

Redis 执行命令只会用一个 CPU，CPU 饱和是指 Redis 把单核 CPU 使用到了接近 100%。使用 **top 命令**可以识别出 Redis 进程是否占用了较多 CPU。

##### 3. 持久化阻塞

持久化过程挺多造成阻塞的原因。

###### (1) fork阻塞

fork 操作发生在**执行 RDB 或 AOF 的重写**时，主线程调用 fork 产生**子进程**完成持久化操作。

这个过程也可能阻塞。

可以执行 **info stats** 命令获取到 **latest_fork_usec** 指标查看最近一次 fork 操作的耗时。

###### (2) AOF刷盘阻塞

开启 AOF 持久化时，文件刷盘方式一般采用每秒 1 次，后台线程每秒对 AOF 文件做 **fsync** 操作。当主线程发现距离上一次的 fsync 成功**超过 2 秒**时，为了数据安全性它会**阻塞**直到后台线程执行 fsync **成功**。

这种阻塞行为主要是由**硬盘压力**引起的，比如其他进程正在大量占用硬盘资源，导致 Redis 进程刷盘超时。可以使用 **iotop** 指令查看进程使用**硬盘资源**的情况。

###### (3) HugePage写操作阻塞

子进程在执行重写期间利用 Linux 写时复制技术降低内存开销，因此只有在写操作的时候 Redis 才复制要修改的内存页。

对于**开启了 Transparent HugePage** 的操作系统，每次**写命令**引起的复制内存页单位由 4K 变为 2MB，增大了 512倍，会**拖慢写操作**执行时间。

#### 阻塞外在原因

##### 1. CPU竞争

Redis 是典型的 CPU 密集型应用，不建议和其他密集型服务一起部署。其他进程可能竞争 CPU。此外，如果做了 CPU 绑定，那么子进程和父进程会共享使用同一个 CPU，子进程占用 CPU 通常较大，对于开启了持久化或者参与复制的主节点不建议绑定 CPU。

##### 2. 内存交换

Redis 本来就是基于内存工作的，如果**发生了内存交换**那么对性能影响**极大**。排查方式如下。

- 查询 Redis 进程号。

```bash
redis-cli -p 6739 info server | grep process_id
process_id:4476
```

- 根据进程号查看内存交换信息。

```bash
cat /proc/4476/smaps | grep Swap
```

为防止内存交换，需要保证机器**内存充足**，也确保所有 Redis 实例**设置最大可用内存**。

##### 3. 网络问题

通信通过网络进行，网络也可能出现问题导致阻塞。常见问题有：**连接拒绝、网络延迟、网卡软中断**等。



### 三、Redis运维

总结一下 Redis 运维指南。

#### Linux相关优化

Redis 部署在 Linux 上，所以 Linux 的配置也会影响 Redis 性能。

- Linux > 3.5， vm.swappiness 建议为 1， 否则建议为 0。
- Transparent Huge Pages（THP）建议关闭掉。
- 建议对 Redis 所有节点所在机器使用 **NTP** 服务。
- 合理设置 ulimit 保证网络连接正常。
- 合理设置 tcp-backlog 参数。

#### Redis安全建议

- 根据网络环境决定是否设置 Redis 密码。
- 合理的防火墙可以防止攻击。
- bind 可以将 Redis 的访问绑定到指定网卡上。
- 可以适当错开 Redis 默认端口启动，防止被攻击。

#### 其他

##### 1. bigkey

bigkey 可能造成数据倾斜、超时阻塞、网络拥塞，是 Redis 生产环境的一颗定时炸弹。删除 bigkey 通常使用**渐进式**遍历的方式进行，防止出现阻塞。

##### 2. 热点key

可以通过客户端、代理、monitor、抓包等方式找到热点 key 进行分析。

#### Redis 配置文件详解

注意：如果想在一台机器上不同端口启动多个 Redis 服务，就把后面的配置文件修改下，后面加上配置文件路径启动即可。端口，和数据，日志等改下。

```properties
# Redis配置文件样例

# Note on units: when memory size is needed, it is possible to specifiy
# it in the usual form of 1k 5GB 4M and so forth:
#
# 1k => 1000 bytes
# 1kb => 1024 bytes
# 1m => 1000000 bytes
# 1mb => 1024*1024 bytes
# 1g => 1000000000 bytes
# 1gb => 1024*1024*1024 bytes
#
# units are case insensitive so 1GB 1Gb 1gB are all the same.

# Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
# 启用守护进程后，Redis会把pid写到一个pidfile中，在/var/run/redis.pid
daemonize yes

# 当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid文件，可以通过pidfile指定
pidfile /var/run/redis.pid

# 指定Redis监听端口，默认端口为6379
# 如果指定0端口，表示Redis不监听TCP连接
port 6379

# 绑定的主机地址
# 你可以绑定单一接口，如果没有绑定，所有接口都会监听到来的连接
# bind 127.0.0.1

# Specify the path for the unix socket that will be used to listen for
# incoming connections. There is no default, so Redis will not listen
# on a unix socket when not specified.
#
# unixsocket /tmp/redis.sock
# unixsocketperm 755

# 当客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
timeout 0

# 指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，默认为verbose
# debug (很多信息, 对开发／测试比较有用)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably)
# warning (only very important / critical messages are logged)
loglevel verbose

# 日志记录方式，默认为标准输出，如果配置为redis为守护进程方式运行，而这里又配置为标准输出，则日志将会发送给/dev/null
logfile /var/log/redis.log

# To enable logging to the system logger, just set 'syslog-enabled' to yes,
# and optionally update the other syslog parameters to suit your needs.
# syslog-enabled no

# Specify the syslog identity.
# syslog-ident redis

# Specify the syslog facility.  Must be USER or between LOCAL0-LOCAL7.
# syslog-facility local0

# 设置数据库的数量，默认数据库为0，可以使用select <dbid>命令在连接上指定数据库id
# dbid是从0到‘databases’-1的数目
databases 16

################################ SNAPSHOTTING  #################################
# 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
# Save the DB on disk:
#
#   save <seconds> <changes>
#
#   Will save the DB if both the given number of seconds and the given
#   number of write operations against the DB occurred.
#
#   满足以下条件将会同步数据:
#   900秒（15分钟）内有1个更改
#   300秒（5分钟）内有10个更改
#   60秒内有10000个更改
#   Note: 可以把所有“save”行注释掉，这样就取消同步操作了

save 900 1
save 300 10
save 60 10000

# 指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，可以关闭该选项，但会导致数据库文件变的巨大
rdbcompression yes

# 指定本地数据库文件名，默认值为dump.rdb
dbfilename /usr/local/redis/dump.rdb

# 工作目录.
# 指定本地数据库存放目录，文件名由上一个dbfilename配置项指定
# 
# Also the Append Only File will be created inside this directory.
# 
# 注意，这里只能指定一个目录，不能指定文件名
dir /usr/local/redis

################################# REPLICATION #################################

# 主从复制。使用slaveof从 Redis服务器复制一个Redis实例。注意，该配置仅限于当前slave有效
# so for example it is possible to configure the slave to save the DB with a
# different interval, or to listen to another port, and so on.
# 设置当本机为slav服务时，设置master服务的ip地址及端口，在Redis启动时，它会自动从master进行数据同步
# slaveof <masterip> <masterport>


# 当master服务设置了密码保护时，slav服务连接master的密码
# 下文的“requirepass”配置项可以指定密码
# masterauth <master-password>

# When a slave lost the connection with the master, or when the replication
# is still in progress, the slave can act in two different ways:
#
# 1) if slave-serve-stale-data is set to 'yes' (the default) the slave will
#    still reply to client requests, possibly with out of data data, or the
#    data set may just be empty if this is the first synchronization.
#
# 2) if slave-serve-stale data is set to 'no' the slave will reply with
#    an error "SYNC with master in progress" to all the kind of commands
#    but to INFO and SLAVEOF.
#
slave-serve-stale-data yes

# Slaves send PINGs to server in a predefined interval. It's possible to change
# this interval with the repl_ping_slave_period option. The default value is 10
# seconds.
#
# repl-ping-slave-period 10

# The following option sets a timeout for both Bulk transfer I/O timeout and
# master data or ping response timeout. The default value is 60 seconds.
#
# It is important to make sure that this value is greater than the value
# specified for repl-ping-slave-period otherwise a timeout will be detected
# every time there is low traffic between the master and the slave.
#
# repl-timeout 60

################################## SECURITY ###################################

# Warning: since Redis is pretty fast an outside user can try up to
# 150k passwords per second against a good box. This means that you should
# use a very strong password otherwise it will be very easy to break.
# 设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过auth <password>命令提供密码，默认关闭
# requirepass foobared

# Command renaming.
#
# It is possilbe to change the name of dangerous commands in a shared
# environment. For instance the CONFIG command may be renamed into something
# of hard to guess so that it will be still available for internal-use
# tools but not available for general clients.
#
# Example:
#
# rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
#
# It is also possilbe to completely kill a command renaming it into
# an empty string:
#
# rename-command CONFIG ""

################################### LIMITS ####################################

# 设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，
# 如果设置maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max Number of clients reached错误信息
# maxclients 128

# Don't use more memory than the specified amount of bytes.
# When the memory limit is reached Redis will try to remove keys with an
# EXPIRE set. It will try to start freeing keys that are going to expire
# in little time and preserve keys with a longer time to live.
# Redis will also try to remove objects from free lists if possible.
#
# If all this fails, Redis will start to reply with errors to commands
# that will use more memory, like SET, LPUSH, and so on, and will continue
# to reply to most read-only commands like GET.
#
# WARNING: maxmemory can be a good idea mainly if you want to use Redis as a
# 'state' server or cache, not as a real DB. When Redis is used as a real
# database the memory usage will grow over the weeks, it will be obvious if
# it is going to use too much memory in the long run, and you'll have the time
# to upgrade. With maxmemory after the limit is reached you'll start to get
# errors for write operations, and this may even lead to DB inconsistency.
# 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，
# 当此方法处理后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。
# Redis新的vm机制，会把Key存放内存，Value会存放在swap区
# maxmemory <bytes>

# MAXMEMORY POLICY: how Redis will select what to remove when maxmemory
# is reached? You can select among five behavior:
# 
# volatile-lru -> remove the key with an expire set using an LRU algorithm
# allkeys-lru -> remove any key accordingly to the LRU algorithm
# volatile-random -> remove a random key with an expire set
# allkeys->random -> remove a random key, any key
# volatile-ttl -> remove the key with the nearest expire time (minor TTL)
# noeviction -> don't expire at all, just return an error on write operations
# 
# Note: with all the kind of policies, Redis will return an error on write
#       operations, when there are not suitable keys for eviction.
#
#       At the date of writing this commands are: set setnx setex append
#       incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
#       sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
#       zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
#       getset mset msetnx exec sort
#
# The default is:
#
# maxmemory-policy volatile-lru

# LRU and minimal TTL algorithms are not precise algorithms but approximated
# algorithms (in order to save memory), so you can select as well the sample
# size to check. For instance for default Redis will check three keys and
# pick the one that was used less recently, you can change the sample size
# using the following configuration directive.
#
# maxmemory-samples 3

############################## APPEND ONLY MODE ###############################

# 
# Note that you can have both the async dumps and the append only file if you
# like (you have to comment the "save" statements above to disable the dumps).
# Still if append only mode is enabled Redis will load the data from the
# log file at startup ignoring the dump.rdb file.
# 指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。
# 因为redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no
# IMPORTANT: Check the BGREWRITEAOF to check how to rewrite the append
# log file in background when it gets too big.

appendonly no

# 指定更新日志文件名，默认为appendonly.aof
# appendfilename appendonly.aof

# The fsync() call tells the Operating System to actually write data on disk
# instead to wait for more data in the output buffer. Some OS will really flush 
# data on disk, some other OS will just try to do it ASAP.

# 指定更新日志条件，共有3个可选值：
# no:表示等操作系统进行数据缓存同步到磁盘（快）
# always:表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全）
# everysec:表示每秒同步一次（折衷，默认值）

appendfsync everysec
# appendfsync no

# When the AOF fsync policy is set to always or everysec, and a background
# saving process (a background save or AOF log background rewriting) is
# performing a lot of I/O against the disk, in some Linux configurations
# Redis may block too long on the fsync() call. Note that there is no fix for
# this currently, as even performing fsync in a different thread will block
# our synchronous write(2) call.
#
# In order to mitigate this problem it's possible to use the following option
# that will prevent fsync() from being called in the main process while a
# BGSAVE or BGREWRITEAOF is in progress.
#
# This means that while another child is saving the durability of Redis is
# the same as "appendfsync none", that in pratical terms means that it is
# possible to lost up to 30 seconds of log in the worst scenario (with the
# default Linux settings).
# 
# If you have latency problems turn this to "yes". Otherwise leave it as
# "no" that is the safest pick from the point of view of durability.
no-appendfsync-on-rewrite no

# Automatic rewrite of the append only file.
# Redis is able to automatically rewrite the log file implicitly calling
# BGREWRITEAOF when the AOF log size will growth by the specified percentage.
# 
# This is how it works: Redis remembers the size of the AOF file after the
# latest rewrite (or if no rewrite happened since the restart, the size of
# the AOF at startup is used).
#
# This base size is compared to the current size. If the current size is
# bigger than the specified percentage, the rewrite is triggered. Also
# you need to specify a minimal size for the AOF file to be rewritten, this
# is useful to avoid rewriting the AOF file even if the percentage increase
# is reached but it is still pretty small.
#
# Specify a precentage of zero in order to disable the automatic AOF
# rewrite feature.

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

################################## SLOW LOG ###################################

# The Redis Slow Log is a system to log queries that exceeded a specified
# execution time. The execution time does not include the I/O operations
# like talking with the client, sending the reply and so forth,
# but just the time needed to actually execute the command (this is the only
# stage of command execution where the thread is blocked and can not serve
# other requests in the meantime).
# 
# You can configure the slow log with two parameters: one tells Redis
# what is the execution time, in microseconds, to exceed in order for the
# command to get logged, and the other parameter is the length of the
# slow log. When a new command is logged the oldest one is removed from the
# queue of logged commands.

# The following time is expressed in microseconds, so 1000000 is equivalent
# to one second. Note that a negative number disables the slow log, while
# a value of zero forces the logging of every command.
slowlog-log-slower-than 10000

# There is no limit to this length. Just be aware that it will consume memory.
# You can reclaim memory used by the slow log with SLOWLOG RESET.
slowlog-max-len 1024

################################ VIRTUAL MEMORY ###############################

### WARNING! Virtual Memory is deprecated in Redis 2.4
### The use of Virtual Memory is strongly discouraged.

### WARNING! Virtual Memory is deprecated in Redis 2.4
### The use of Virtual Memory is strongly discouraged.

# Virtual Memory allows Redis to work with datasets bigger than the actual
# amount of RAM needed to hold the whole dataset in memory.
# In order to do so very used keys are taken in memory while the other keys
# are swapped into a swap file, similarly to what operating systems do
# with memory pages.
# 指定是否启用虚拟内存机制，默认值为no，
# VM机制将数据分页存放，由Redis将访问量较少的页即冷数据swap到磁盘上，访问多的页面由磁盘自动换出到内存中
# 把vm-enabled设置为yes，根据需要设置好接下来的三个VM参数，就可以启动VM了
vm-enabled no
# vm-enabled yes

# This is the path of the Redis swap file. As you can guess, swap files
# can't be shared by different Redis instances, so make sure to use a swap
# file for every redis process you are running. Redis will complain if the
# swap file is already in use.
#
# Redis交换文件最好的存储是SSD（固态硬盘）
# 虚拟内存文件路径，默认值为/tmp/redis.swap，不可多个Redis实例共享
# *** WARNING *** if you are using a shared hosting the default of putting
# the swap file under /tmp is not secure. Create a dir with access granted
# only to Redis user and configure Redis to create the swap file there.
vm-swap-file /tmp/redis.swap

# With vm-max-memory 0 the system will swap everything it can. Not a good
# default, just specify the max amount of RAM you can in bytes, but it's
# better to leave some margin. For instance specify an amount of RAM
# that's more or less between 60 and 80% of your free RAM.
# 将所有大于vm-max-memory的数据存入虚拟内存，无论vm-max-memory设置多少，所有索引数据都是内存存储的（Redis的索引数据就是keys）
# 也就是说当vm-max-memory设置为0的时候，其实是所有value都存在于磁盘。默认值为0
vm-max-memory 0

# Redis swap文件分成了很多的page，一个对象可以保存在多个page上面，但一个page上不能被多个对象共享，vm-page-size是要根据存储的数据大小来设定的。
# 建议如果存储很多小对象，page大小最后设置为32或64bytes；如果存储很大的对象，则可以使用更大的page，如果不确定，就使用默认值
vm-page-size 32

# 设置swap文件中的page数量由于页表（一种表示页面空闲或使用的bitmap）是存放在内存中的，在磁盘上每8个pages将消耗1byte的内存
# swap空间总容量为 vm-page-size * vm-pages
#
# With the default of 32-bytes memory pages and 134217728 pages Redis will
# use a 4 GB swap file, that will use 16 MB of RAM for the page table.
#
# It's better to use the smallest acceptable value for your application,
# but the default is large in order to work in most conditions.
vm-pages 134217728

# Max number of VM I/O threads running at the same time.
# This threads are used to read/write data from/to swap file, since they
# also encode and decode objects from disk to memory or the reverse, a bigger
# number of threads can help with big objects even if they can't help with
# I/O itself as the physical device may not be able to couple with many
# reads/writes operations at the same time.
# 设置访问swap文件的I/O线程数，最后不要超过机器的核数，如果设置为0，那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟，默认值为4
vm-max-threads 4

############################### ADVANCED CONFIG ###############################

# Hashes are encoded in a special way (much more memory efficient) when they
# have at max a given numer of elements, and the biggest element does not
# exceed a given threshold. You can configure this limits with the following
# configuration directives.
# 指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法
hash-max-zipmap-entries 512
hash-max-zipmap-value 64

# Similarly to hashes, small lists are also encoded in a special way in order
# to save a lot of space. The special representation is only used when
# you are under the following limits:
list-max-ziplist-entries 512
list-max-ziplist-value 64

# Sets have a special encoding in just one case: when a set is composed
# of just strings that happens to be integers in radix 10 in the range
# of 64 bit signed integers.
# The following configuration setting sets the limit in the size of the
# set in order to use this special memory saving encoding.
set-max-intset-entries 512

# Similarly to hashes and lists, sorted sets are also specially encoded in
# order to save a lot of space. This encoding is only used when the length and
# elements of a sorted set are below the following limits:
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

# Active rehashing uses 1 millisecond every 100 milliseconds of CPU time in
# order to help rehashing the main Redis hash table (the one mapping top-level
# keys to values). The hash table implementation redis uses (see dict.c)
# performs a lazy rehashing: the more operation you run into an hash table
# that is rhashing, the more rehashing "steps" are performed, so if the
# server is idle the rehashing is never complete and some more memory is used
# by the hash table.
# 
# The default is to use this millisecond 10 times every second in order to
# active rehashing the main dictionaries, freeing memory when possible.
#
# If unsure:
# use "activerehashing no" if you have hard latency requirements and it is
# not a good thing in your environment that Redis can reply form time to time
# to queries with 2 milliseconds delay.
# 指定是否激活重置哈希，默认为开启
activerehashing yes

################################## INCLUDES ###################################

# 指定包含其他的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各实例又拥有自己的特定配置文件
# include /path/to/local.conf
# include /path/to/other.conf
```







#### 参考资料

- 《Redis 开发与运维》