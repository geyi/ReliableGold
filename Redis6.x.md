# 9
## 标题
### blocking
dict *blocking_keys：redis使用一个阻塞键字典（key -> list(client)）来保存对某个key的阻塞操作（BRPOP）。这个阻塞操作并没有造成redis work线程的阻塞，redis仍然可以继续处理其他任务。
add value
signalKeyAsReady
dict *ready_keys：当有其他客户端生产数据时（LPUSH），redis会把key放入ready_keys里。
unblock client


notifyKeyspaceEvent：针对key的操作可以触发一些事件（回调，事件驱动），此功能默认是关闭的，可以通过在redis客户端使用命令`CONFIG SET notify-keyspace-events "KEgx"`开启此功能  
例如：  
订阅k1的相关事件：`subscribe __keyspace@0__:k1`  
订阅del事件：`subscribe __keyevent@0__:del`


### publish-subscribe
没有数据存储的过程，具有实时性
dict *pubsub_channels：字典结构（channelKey -> list(client)）


### 事务
原理：客户端A watch 了k1，在客户端A执行exec之前，客户端B修改了k1，这时 watch k1的客户端会A被标记为dirty，最后当客户端A执行exec时，发现自己被标记为dirty，则**不执行**

dict *watched_keys


## 小结
以上三点在cmd处理的时候，分别有对应的事件去处理：
1. ready_keys
2. __key*
3. watched_keys


## malloc
1. 分配内存：得到一个空间
2. 使用：初始化

空间其实就是一个指针。问题？：
1. 空间多大
2. 多大的语义其实对应的是（数据）类型

expect size + prefix size  
expect size表示用户期望分配的内存大小  
prefix size用来存储分配内存的大小

# 10
Redis的排他性：锁，静态的，是标志
Redis的过滤性：串行，incr/decr，list，set，他们是动态的

提高性能的思路：
- 减少无用的操作
- 多级缓存
- 线程池隔离：控制资源的分配、也可以控制有序性
- 分治

## 缓存的分类

### Cache Side
查询缓存，如果不存在则读取数据库并更新到缓存中。如果更新了数据库，那么操作完成数据库后，删除缓存。

### Read Through
先查询缓存中数据是否存在，如果存在则直接返回，如果不存在，则由缓存组件负责从数据库中同步加载数据。

### Write Through
先查询要写入的数据在缓存中是否已经存在，如果已经存在，则更新缓存中的数据，并且由缓存组件同步更新到数据库中。果缓存中数据不存在，我们把这种情况叫做Write Miss（写失效）。

### Write Back
Write Back是相较于Write Through而言的一种异步回写策略。异步回写时可以进行合并写的优化。