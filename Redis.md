# 背景常识

## 磁盘
1. 毫秒级寻址（10-20ms）
2. 带宽低（100MB/s）

> **平均寻址时间**  
寻址时间分为两个部分：  
第一：寻找目标磁道（t1）  
第二：找到磁道后，磁头等待欲读/写的磁道的区段旋转到磁头下所需要的时间（t2）  
`T = t1 + t2 = (t1max + t1min) / 2 + (t2max + t2min) / 2`  
例：磁盘机以7200rpm速度旋转，平均定位时间8ms  
因为磁头与欲读/写的磁道的距离是个均匀分布，所以平均寻道时间为磁盘转一周的时间的二分之一，`60 / 7200 / 2 = 4.165ms`  
所以平均寻址时间为`8 + 4.165 = 12.165ms`

## 内存
1. 纳秒级寻址（5、6、7、8、10ns）
2. 带宽高（10-50GB/s）

## 4K对齐
一个扇区512字节，那么1TB的磁盘空间就会有2^31个扇区，这时索引的成本会变大。所以有了4K对齐，即读写的最小单位不再是512字节的扇区，而是4KB（一页）。这时同样大小的磁盘空间创建出的索引更小。

## 数据库
- data page（数据和索引都是存储在磁盘中）
- schema（列的数量，类型），类型决定了字节宽度。行级存储。
- B+树（树干在内存中），树干包含了一段一段的索引区间和存储索引页的偏移。即，非叶子节点存储记录的PK，用于查询加速，适合内存存储；叶子节点存储实际记录行，记录行相对比较紧密的存储，适合大数据量磁盘存储；

### 表很大，性能会下降
- 如果表有索引，增删改变慢
- 一个或少量查询依然很快，当并发大的时候查询速度会受硬盘带宽影响。因为数量很大时，数据散落在更多的data page中，导致在高并发下可能要从更多data page中获取数据，这时磁盘带宽就成为性能瓶颈。

### SAP HANA
内存中的关系型数据库（内存2T），2亿

**以上问题说明了为什么要使用redis**

https://db-engines.com/en

# Redis
Redis 是一个开源的在内存中存储数据的结构化键值数据库，同时也可以被用作缓存、消息代理和流处理引擎。Redis 提供了多种数据结构，包括字符串（strings）、哈希（hashes）、列表（lists）、集合（sets）、有范围查询的有序集合（sorted sets）、位图（bitmaps）、HyperLogLogs、地理空间索引（geospatial indexes）和流（streams）。Redis 内置了复制（replication）、Lua 脚本、LRU 淘汰机制（Least Recently Used）、事务（transactions）以及不同级别的磁盘持久化，并通过 Redis Sentinel 实现了高可用性，还通过 Redis Cluster 实现了自动分区。

> memcached的value没有类型的概念，而redis的server对每种类型的数据都有自己的操作方法（计算向数据移动)

## 安装Redis
```
yum -y install wget
mkdir software
cd software
wget http://download.redis.io/releases/redis-6.0.6.tar.gz
tar zxf redis-6.0.6.tar.gz
cd redis-6.0.6
yum -y install gcc tcl
make (make distclean)
make test
make install PREFIX=/opt/redis6
vim /etc/profile
export REDIS_HOME=/opt/redis6
export PATH=$PATH:$REDIS_HOME/bin
source /etc/profile
cd utils
./install-server.sh
```

```
Port           : 6379
Config file    : /etc/redis/6379.conf
Log file       : /var/log/redis_6379.log
Data dir       : /var/lib/redis/6379
Executable     : /usr/local/bin/redis-server
Cli Executable : /usr/local/bin/redis-cli
```

在JVM中一个线程的成本大约是1MB
- 线程多了调度成本高
- 内存成本

**redis每连接内的命令是顺序处理的**

## VALUE
Redis中的每个对象都由一个redisObject结构表示，该结构中和保存数据有关的三个属性分别是type属性、 encoding属性和ptr属性。
```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    // 指向底层数据结构的指针
    void *ptr;
} robj;
```

type属性表示value的类型（命令：`type`）  
encoding属性表示value的编码方式（命令：`object encoding`）

Redis存储的数据是**二进制安全的**，它不会对存入的数据做任何处理。例如在存储一个“中”字时，用UTF-8编码，占3个字节，用GBK编码，占2个字节。因此为了确保数据的一致性，连接同一个Redis服务的客户端应该使用相同的编码格式进行读写操作。

### String（int、embstr、raw）
- 字符串
- 整数
- bitmap

Redis会根据当前值的类型和长度来决定使用哪种编码来实现。

Redis中的字符串分为两种存储方式，分别是embstr和raw，当字符串长度小于等于44的时候，Redis使用embstr来存储字符串，而当字符串长度超过44的时候，就需要用raw来存储。

embstr编码是专门用于保存短字符串的一种优化编码方式，但不同之处在于embstr只会调用一次内存分配函数来分配一块连续的内存空间用于保存redisObject和SDS。而raw编码则会调用两次内存分配函数分别分配两块空间来保存redisObject和SDS。

Redis并未直接使用C字符串，而是以struct的形式定义了一个SDS（Simple Dynamic String）类型。当Redis需要一个可以被修改的字符串时，就会使用SDS来表示。在Redis数据库里，包含字符串值的键值对都是由SDS实现的。SDS的结构如下：
```c
struct sdshdr {

    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 字节数组
    char buf[];
};
```
Redis使用SDS的原因：
- 获取字符串长度的时间复杂度为常数
- 杜绝缓冲区溢出
- 减少修改字符串时带来的内存重分配次数
- 二进制安全

#### bitmap（raw）
bitmap在实际生产过程中的一次应用，每日设备推送开关状态统计：
```
二进制从左往右的第一位表示id（自增主键）为1的设备，以此类推
A表示2020-1-1号的设备推送开关状态（0 0 1 1 1 1 0 0），B表示2020-1-2号的设备推送开关状态（1 1 1 1 0 0 1 0）
0 关
1 开
2020-1-1 4关4开
2020-1-2 3关5开
2号相比1号推送开关新增关闭的数量2
2号相比1号推送开关新增打开的数量3
C = A ^ B （1 1 0 0 1 1 1 0） 结果中1的数量表示开关发生变化的数量5
C & A （0 0 0 0 1 1 0 0） 结果中1的数量表示2号相比1号推送开关新关闭的数量2
C & B （1 1 0 0 0 0 1 0） 结果中1的数量表示2号相比1号推送开关新打开的数量3

8 * 1024 * 1024 = 8388608

1MB内存可以表示8388608个设备推送开关情况
```

类似的应用还有每日所有用户登录状态统计、一个用户每周/月/年的登录状态统计

### List（quicklist）
数据结构是一个双向链表，有一个头指针指向链表的第一个元素和一个尾指针指向链表的最后一个元素。但是考虑到链表的附加空间相对太高，prev 和 next 指针就要占去 16 个字节 (64bit 系统的指针是 8 个字节)，另外每个节点的内存都是单独分配，会加剧内存的碎片化，影响内存管理效率。因此Redis3.2版本开始对列表数据结构进行了改造，使用 quicklist 代替了 ziplist 和 linkedlist。

quicklist 实际上是 zipList 和 linkedList 的混合体，它将 linkedList 按段切分，每一段使用 zipList 来紧凑存储，多个 zipList 之间使用双向指针串接起来。

quicklist 内部默认定义的单个 ziplist 的大小为 8k 字节。超过这个大小，就会重新分配一个 ziplist 了。这个长度可以由参数list-max-ziplist-size控制。
```
# Lists are also encoded in a special way to save a lot of space.
# The number of entries allowed per internal list node can be specified
# as a fixed maximum size or a maximum number of elements.
# For a fixed maximum size, use -5 through -1, meaning:
# -5: max size: 64 Kb  <-- not recommended for normal workloads
# -4: max size: 32 Kb  <-- not recommended
# -3: max size: 16 Kb  <-- probably not recommended
# -2: max size: 8 Kb   <-- good
# -1: max size: 4 Kb   <-- good
# Positive numbers mean store up to _exactly_ that number of elements
# per list node.
# The highest performing option is usually -2 (8 Kb size) or -1 (4 Kb size),
# but if your use case is unique, adjust the settings as necessary.
list-max-ziplist-size -2
```

quicklist 可以对 ziplist 来进行压缩，而且可以指定压缩深度。由参数list-compress-depth控制。默认的压缩深度为 0, 也就是所有的节点都不压缩。为了支持快速的 push/pop 操作，quicklist 两端的第一个 ziplist 不进行压缩，这时压缩深度为 1。如果压缩深度为 2, 则是两端各自两个 ziplist 不压缩。
```
# Lists may also be compressed.
# Compress depth is the number of quicklist ziplist nodes from *each* side of
# the list to *exclude* from compression.  The head and tail of the list
# are always uncompressed for fast push/pop operations.  Settings are:
# 0: disable all list compression
# 1: depth 1 means "don't start compressing until after 1 node into the list,
#    going from either the head or tail"
#    So: [head]->node->node->...->node->[tail]
#    [head], [tail] will always be uncompressed; inner nodes will compress.
# 2: [head]->[next]->node->node->...->node->[prev]->[tail]
#    2 here means: don't compress head or head->next or tail->prev or tail,
#    but compress all nodes between them.
# 3: [head]->[next]->[next]->node->node->...->node->[prev]->[prev]->[tail]
# etc.
list-compress-depth 0
```

应用：
- 同向命令（LPUSH LPOP）实现了栈
- 反向命令（LPSUH RPOP）实现了队列
- 通过索引操作List（LINDEX LSET）实现了数组
- 阻塞的单播队列（BLPOP LPUSH）

### Hash（ziplist、hashtable）
应用场景：
- 保存用户信息
- 对hash中的元素进行数值计算

#### ziplist
```
非空 ziplist 示例图

area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|

size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
            +---------+--------+-------+--------+--------+--------+--------+-------+
component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
            +---------+--------+-------+--------+--------+--------+--------+-------+
                                       ^                          ^        ^
address                                |                          |        |
                                ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
                                                                  |
                                                        ZIPLIST_ENTRY_TAIL
```

ziplist中的每个元素都包含了两部分前置信息。首先，存储了上一个条目的长度，以便能够从后向前遍历列表。其次，提供了条目的编码方式。它表示条目的类型，是整数还是字符串，并且对于字符串而言，还表示了字符串有效负载的长度。因此一个完整条目存储看起来像这样：
```
<prevlen> <encoding> <entry-data>
```
有时编码本身就代表着条目的内容，比如对于小整数来说。在这种情况下，<entry-data>部分是不存在的，我们可以只使用编码本身即可：
```
<prevlen> <encoding>
```

### Set（intset、hashtable）
集合操作（交，并，差）

SRANDMEMBER

#### intset
整数集合intset是集合键的底层实现之一，当一个集合只包含整数值的元素，并且这个集合的元素数量不多时，redis会使用整数集合作为集合键的底层实现。

### Sorted_Set（ziplist、skiplist）
物理内存左小右大，不随命令发生变化

具有集合操作指令，同时带有权重、聚合指令

底层数据结构是**跳跃表**（随机造层）

应用场景：
- 排行榜
- 延时队列

## Redis 管道
一次发送多个命令，节省往返时间，降低了通信成本，例：
`echo -e 'set k1 99\n incr k1\n get k1' | nc localhost 6379`

> 集群模式下，对于分散在不同Redis实例的多个不同的key使用管道时，仍然会存在多次网络通信。使用Redisson的`RBatch`时，Redisson会自动合并属于相同Redis实例的key的操作命令，然后通过单次网络调用发送。
> 
> Redisson会定期从Redis集群（每次节点可能不同）同步一次集群节点的状态，因此在命令发送之前，客户端就能通过特定的算法知道要操作的key属于哪个slot（确定了slot就确定了具体的Redis实例）。

## Pub/Sub
广播

## Redis 事务
谁的EXEC先到达，就先执行谁的事务

## 布隆过滤器
### N个不同映射函数和一个二进制位数组
当向布隆过滤器中添加一个元素时，元素作为N个不同映射函数的入参，每个函数返回一个表示二进制位数组索引位置的值，然后将这个位置标记为1。


## Redis如何淘汰过期的keys
Redis keys过期有两种方式：被动和主动方式。
- 访问时判定
- 周期轮询判定  
   1. 测试随机的20个keys进行相关过期检测。  
   2. 删除所有已经过期的keys。  
   3. 如果有多于25%的keys过期，重复步奏1。

## Redis RDB
Linux中的父子进程，父进程中的数据，子进程能不能看到？
- 常规思想，数据在进程之间是隔离的
- 进阶思想，父进程可以让子进程看到数据

在Linux中，export的环境变量，子进程的修改不会破坏父进程，父进程的修改也不会破坏子进程。（子进程中拥有环境变量的一个拷贝）

如果父进程是Redis，内存数据10GB
- 创建子进程的速度？
- 内存空间够不够？

fork系统调用，创建子线程速度快，需要的内存空间小。

copy on write，创建子进程时并不发生复制，使得创建进程变快了。根据经验，不可能父子进程把所有数据都改一遍。父进程修改数据时，就是父进程触发中断，结果就是父进程里面的地址空间被改变了。
```C
#include<stdio.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<unistd.h>
int main(){
    pid_t pid;
    int num=0;
    pid=fork();
    if(pid==0){
        num+=10;
        printf("child %d\n",num);
        printf("child %d\n",&num);
    }
    if(pid>0){
        num+=20;
        printf("parent %d\n",num);
        printf("parent %d\n",&num);
    }

    return 0;
}
```

综上所述：Redis父进程通过fork系统调用创建了一个子进程，由子进程对内存中的数据进行落盘。（时点性）

> save：阻塞。应用场景：关机维护时。
> 
> bgsave：fork创建子进程。应用场景：仍然需要对外提供服务时。

缺点：
1. 只有一个dump.rdb
2. 丢失数据相对多一些

优点：
1. 恢复速度相对快（序列化反序列化）

## Redis AOF
Redis的写操作记录到文件中

如果开启了AOF，恢复时只会用AOF

优点：
1. 丢失数据少

弊端：
1. 体量无限大，恢复慢

### 重写
bgrewriteaof：异步的重写AOF文件

4.0之前
- 删除抵消的命令
- 合并重复的命令

最终也是一个纯指令的日志文件

4.0之后
- 将老的数据RDB到AOF文件中，将增量的以指令的的方式append到AOF中。(https://raw.githubusercontent.com/antirez/redis/4.0/00-RELEASENOTES)

记录指令时触发IO的三种策略
- NO
- ALWAYS
- everysec

## Redis 集群
主备：客户端只能访问主，备机用于在主机不可用时代替主机继续提供服务。
主从：客户端即可以访问主，也可以访问从。

单机，单节点，单实例
1. 单点故障
2. 容量有限
3. 性能瓶颈

AKF
X：全量，镜像
Y：业务，功能
Z：优先级，逻辑再拆分（数据分区）

新的问题
1. 数据一致性  
   强一致性：所有节点阻塞，直到数据全部一致。破坏了可用性。（同步阻塞）
   丢失数据：容忍数据丢失一部分。（异步）
   最终一致性：同步阻塞的将数据发送到一个可靠的，响应速度够快的中间件（Kafka），实现最终一致性。


在2个从节点的集群中，2个节点处于不同状态时谁也无法说服谁。网络分区，脑裂

3个从节点  
2个写成功时（过半），成功解决脑裂问题  
3个都写成功时，即数据的强一致性，降低了可用性

使用奇数台
例如，3台和4台服务器的集群都只能容忍一台服务器宕机。但是4台的成本高于3台，而且出现故障的概率更高。

### 集群搭建
使用单台虚拟机搭建伪集群，机器IP地址192.168.10.130

1. 创建根目录：mkdir /opt/redis-cluster
2. 创建子目录
    ```
    .
    ├── 6379
    │   ├── data
    │   └── logs
    ├── 6380
    │   ├── data
    │   └── logs
    ├── 6381
    │   ├── data
    │   └── logs
    ├── 6382
    │   ├── data
    │   └── logs
    ├── 6383
    │   ├── data
    │   └── logs
    └── 6384
        ├── data
        └── logs
    ```
3. 对默认的配置文件进行如下修改
    ```
    daemonize yes
    appendonly yes
    cluster-enabled yes
    ```
4. 为每个Redis实例创建单独的配置文件（不同实例之间的配置只有端口号和绝对路径的差异）
    ```
    include /opt/redis-cluster/redis.conf
    bind 192.168.10.130
    port 6379
    pidfile "/root/redis-cluster/6379/redis_6379.pid"
    logfile "/root/redis-cluster/6379/logs/redis.log"
    dir "/root/redis-cluster/6379/data"
    cluster-config-file "nodes-6379.conf"
    ```
5. 使用脚本启动所有实例
    ```
    #!/bin/bash
    redis-server /root/redis-cluster/6379/data/6379.conf
    redis-server /root/redis-cluster/6380/data/6380.conf
    redis-server /root/redis-cluster/6381/data/6381.conf
    redis-server /root/redis-cluster/6382/data/6382.conf
    redis-server /root/redis-cluster/6383/data/6383.conf
    redis-server /root/redis-cluster/6384/data/6384.conf

    sleep 5

    p=`netstat -nltp | grep -E '192.168.10.130:(6379|6380|6381|6382|6383|6384)' | wc -l`
    echo "redis start successful $p instance"
    ```
6. 创建集群：`redis-cli --cluster create $HOSTS --cluster-replicas $REPLICAS`

## Redis 复制
Redis默认使用异步复制，其特点是低延迟和高性能

设置从节点追随的主节点：`replicaof host port`。设置追随时发生了什么：
- 主节点发生RDB落盘，并将数据同步给跟随的从节点
- 从节点删除自身的删除，应用从主节点同步过来RDB文件

故障恢复重新启动从节点：`redis-server ./6381.conf --replicaof 127.0.0.1 6379`。这时不会发生RDB落盘，使用的是增量同步。

redis-server ./6381.conf --replicaof 127.0.0.1 6379 --appendonly yes
带上--appendonly yes参数启动时，会触发RDB落盘。从节点应用同步过来的RDB文件，然后执行AOF重写，因为当前使用的AOF模式（但是AOF文件中没有记录replica id）。

在主上可以看到有哪些从节点追随

从节点切换成主节点：`replicaof no one`

主从复制相关配置参数
- relicaof <primaryid> <primaryport>：设置追随的主节点
- replica-serve-stale-data yes：从节点同步数据时，是否继续响应客户端的查询请求
- replica-read-only yes：从节点是否只读
- repl-diskless-sync no：是否直接将数据通过网络传输到从节点（取决于网络带宽的大小和磁盘性能）
- repl-backlog-size 1mb：增量复制（同步）的队列大小，更新操作很多时需要设置大些
- min-replicas-to-write 3：最小几个从节点写成功

主从复制的模式需要人工处理故障

## Redis Sentinel
```
port 26379
sentinel monitor mymaster 127.0.0.1 6379 2
```
启动哨兵：redis-server ./26379.conf --sentinel

一哨兵是怎么知道其他哨兵的（发布订阅）

# 单点容量问题
## 数据分类，按业务拆分


## 数据分片
数据分片的三种模式

### hash + 取模  
缺点：取模的数必须固定，影响分布式下的扩展。

### 随机


### 一致性哈希
虚拟节点：解决数据倾斜问题

优点：增加新的节点不会造成所有数据重新分片  
缺点：新增节点时造成一小部分数据不能命中，出现缓存击穿。解决方案：每次读取数据时，先后尝试从离我最近的两个节点读取。如果都没有，再去数据库读取。（更倾向于作为缓存，而不是数据库）

# 反向代理
当所有的客户端直接连接Redis服务器时，随着连接数的增加，Redis Server连接压力越来越大。

方案：在客户端与Redis Server之间增加一个反向代理层（twemproxy，predixy，cluster，codies）

各个代理的对比：https://blog.csdn.net/rebaic/article/details/76384028

> 将数据分片的算法放到代理层

# 预分区
![预分区](https://raw.githubusercontent.com/geyi/ReliableGold/master/image/Redis/pre-parition.png)

![Redis Cluster](https://raw.githubusercontent.com/geyi/ReliableGold/master/image/Redis/cluster.png)

数据分治带来的问题
1. 事务一致性问题
2. 跨节点关联查询
3. 跨节点分页，排序
4. 主键避重
5. 公共表

使用{}使数据被分配到同一个节点

# 分布式锁
1. setnx
2. 过期时间
3. 守护线程延长过期时间