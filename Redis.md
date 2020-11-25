# 常识

磁盘：
1. 毫秒级寻址
2. 带宽低

内存：
1. 纳秒级寻址
2. 带宽高

一个扇区512字节，那么1TB的磁盘空间就会有2^31个扇区，这时索引的成本会变大。所以有了4K对齐，即读写的最小单位是一页（4KB）

数据库
data page（数据和索引都是存储在磁盘中）
schema（列的数量，类型），类型决定了字节宽带。行级存储。
B+树（树干在内存中），树干是一段一段的索引区间和存储索引页的偏移。

表很大，性能会下降
如果表有索引，增删改变慢
一个或少量查询依然很快，当并发大的时候查询速度会受硬盘带宽影响。因为数量很大时，数据散落在更多的data page中，导致在高并发下可能要从更多data page中获取数据，这时磁盘带宽就成为性能瓶颈。

SAP HANA内存级的关系型数据库

为什么要使用redis

https://db-engines.com/en

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。
它支持多种类型的数据结构，如 字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets）， 有序集合（sorted sets） 与范围查询， bitmaps， hyperloglogs 和 地理空间（geospatial） 索引半径查询。
Redis 内置了 复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的 磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动 分区（Cluster）提供高可用性（high availability）。

memcached的value没有类型的概念

redis的server对每种类型的数据都有自己的操作方法
计算向数据移动

# 安装Redis
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
export PATH=$PAHT:$REDIS_HOME/bin
source /etc/profile
cd utils
./install-server.sh

JVM 一个线程的成本1MB
线程多了调度成本高
内存成本

redis每连接内的命令是顺序处理的

redis-cli
help
help @string


key
type属性表示value的类型
encoding属性表示value的编码方式

redis存储的数据是**进制安全的**

# VALUE的类型

## String
- 字符串
- 数值
- bitmap

## List
数据结构是一个双向链表，有一个头指针指向链表的第一个元素和一个尾指针指向链表的最后一个元素

同向命令（LPUSH LPOP）实现了栈，反向命令（LPSUH RPOP）实现了队列