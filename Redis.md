# 常识

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
- schema（列的数量，类型），类型决定了字节宽带。行级存储。
- B+树（树干在内存中），树干包含了一段一段的索引区间和存储索引页的偏移。即，非叶子节点存储记录的PK，用于查询加速，适合内存存储；叶子节点存储实际记录行，记录行相对比较紧密的存储，适合大数据量磁盘存储；

### 表很大，性能会下降
- 如果表有索引，增删改变慢
- 一个或少量查询依然很快，当并发大的时候查询速度会受硬盘带宽影响。因为数量很大时，数据散落在更多的data page中，导致在高并发下可能要从更多data page中获取数据，这时磁盘带宽就成为性能瓶颈。

### SAP HANA
内存中的关系型数据库（内存2T），2亿

**以上问题说明了为什么要使用redis**

https://db-engines.com/en

# Redis
Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。
它支持多种类型的数据结构，如 字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets）， 有序集合（sorted sets） 与范围查询， bitmaps， hyperloglogs 和 地理空间（geospatial） 索引半径查询。
Redis 内置了 复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的 磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动 分区（Cluster）提供高可用性（high availability）。

memcached的value没有类型的概念

redis的server对每种类型的数据都有自己的操作方法（计算向数据移动)

# 安装Redis
```
yum -y install wget
mkdir software
cd software
wget http://download.redis.io/releases/redis-6.0.6.tar.gz
tar zxf redis-6.0.6.tar.gz
cd redis-6.0.6
make
yum -y install gcc
make distclean
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

# KEY
type属性表示value的类型  
encoding属性表示value的编码方式

redis存储的数据是**二进制安全的**

# VALUE的类型

## String（int、embstr、raw）
- 字符串
- 数值
- bitmap

## List（ziplist、linkedlist）
数据结构是一个双向链表，有一个头指针指向链表的第一个元素和一个尾指针指向链表的最后一个元素

- 同向命令（LPUSH LPOP）实现了栈
- 反向命令（LPSUH RPOP）实现了队列
- 通过索引操作List（LINDEX LSET）实现了数组
- 阻塞的单播队列（BLPOP LPUSH）

## Hash（ziplist、hashtable）
应用场景：
- 保存用户信息
- 对hash中的元素进行数值计算

## Set（intset、hashtable）
集合操作（交，并，差）

SRANDMEMBER

## Sorted_Set（ziplist、skiplist）
物理内存左小右大，不随命令发生变化

具有集合操作指令，同时带有权重、聚合指令

底层数据结构是**跳跃表**（随机造层）

应用场景：
- 排行榜

# Redis 管道
一次发送多个命令，节省往返时间，降低了通信成本，例：
`echo -e 'set k1 99\n incr k1\n get k1' | nc localhost 6379`

# Pub/Sub
广播

# Redis 事务
谁的EXEC先到达，就先执行谁的事务

# 布隆过滤器
## N个不同映射函数和一个二进制位数组
当向布隆过滤器中添加一个元素时，元素作为N个不同映射函数的入参，每个函数返回一个表示二进制位数组索引位置的值，然后将这个位置标记为1。


# Redis如何淘汰过期的keys
Redis keys过期有两种方式：被动和主动方式。
- 访问时判定
- 周期轮询判定  
   1. 测试随机的20个keys进行相关过期检测。  
   2. 删除所有已经过期的keys。  
   3. 如果有多于25%的keys过期，重复步奏1。

# Redis RDB
Linux中的父子进程，父进程中的数据，子进程能不能看到？
- 常规思想，数据在进程之间是隔离的
- 进阶思想，父进程可以让子进程看到数据

在Linux中，export的环境变量，子进程的修改不会破坏父进程，父进程的修改也不会破坏子进程。（子进程中拥有环境变量的一个拷贝）

如果父进程是Redis，内存数据10GB
- 创建子进程的速度？
- 内存空间够不够？

fork系统调用，创建子线程速度快，需要的内存空间小。

copy on write，创建子进程时并不发生复制，使得创建进程变快了。根据经验，不可能父子进程把所有数据都改一遍。

综上所述：Redis父进程通过fork系统调用创建了一个子进程，由子进程对内存中的数据进行落盘。（时点性）

> save：阻塞。应用场景：关机维护时。
> 
> bgsave：fork创建子进程。应用场景：仍然需要对外提供服务时。

缺点：
1. 只有一个dump.rdb
2. 丢失数据相对多一些

优点：
1. 恢复速度相对快（序列化反序列化）

# Redis AOF
Redis的写操作记录到文件中

如果开启了AOF，恢复时只会用AOF

优点：
1. 丢失数据少

弊端：
1. 体量无限大，恢复慢

## 重写
bgrewriteaof：异步的重写AOF文件

4.0之前
- 删除抵消的命令
- 合并重复的命令

最终也是一个纯指令的日志文件

4.0之后
- 将老的数据RDB到AOF文件中，将增量的以指令的的方式append到AOF中。

记录指令时触发IO的三种策略
- NO
- ALWAYS
- everysec

# Redis 集群
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

# Redis 复制
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

# Redis Sentinel
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