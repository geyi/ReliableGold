在同一个topic下
- 无关的数据就分散到不同的分区里，以追求并发并行
- 有关的数据，一定要按原有顺序发送到同一个分区里（分区内部是有序的，分区的外部是无序的）

在Kafka中分区具有副本的概念。既然有副本，那么就可以有读写分离，但是读写分离会带来一致性问题。所以Kafka规定只能在主分区上进行W/R

Kafka的broker的partition保存了从producer发送来的数据 -> 数据可以被重复利用（数据的重复利用是站在group上的，但是group内要保证如下单一场景的描述）

在单一场景下，即便在追求性能使用多个consumer进行消费，但是一个partition不能由多个consumer消费。即：partition与consumer的对应关系只存在1:1及N:1的关系，不允许1:N的关系（无法保证顺序行）


# kafka配置文件
- broker.id=0
- listeners=PLAINTEXT://node01:9092
- log.dirs=/var/kafka_data
- zookeeper.connect=localhost:2181/kafka

# 命令
启动kafka：`kafka-server-start.sh server-1.properties`  
创建topic：`kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic test_topic --partitions 2 --replication-factor 1`  
查看topic列表：`kafka-topics.sh --zookeeper localhost:2181/kafka --list`  
查看topic详情：`kafka-topics.sh --zookeeper localhost:2181/kafka --describe`

创建消费者：`kafka-console-consumer.sh --bootstrap-server localhost:9093 --topic test_topic --group group1`  
创建生产者：`kafka-console-producer.sh --broker-list localhost:9093 --topic test_topic`  
显示组：`kafka-consumer-groups.sh --bootstrap-server localhost:9093 --list`

查看kafka文件内容：`kafka-dump-log.sh --files *.log | *.index | *.timeindex`

# 有序性
消息是K-V，相同的key一定去到同一个分区里（hash路由）。borker会保证producer推送的消息的顺序，一个分区可能有不同的key，且不同的key是交叉的（相同的key在一个分区里没有排列在一起）

# 推送拉取
推送：推送说的是server主动将消息推送给消费者，如果推送速率大于消费者的消费能力，那么消费者网卡会被打满。如果通过反馈机制控制推送速率，那么消息就没有了实时性。  
拉取：消费者自主，按需去订阅拉取server的数据。

## 拉取粒度
批次拉取，如何维护offset

单线程，一条一条按顺序处理消息，并更新offset  
多线程，流式的多线程，能多线程的多线程（更多的去利用CPU、网卡等硬件资源），但是整个批次的事务环节交给一个线程，做到这个批次要么成功，要么失败。减少DB的压力，和offset的更新压力。

# 概念

## 一致性

强一致性，所有节点必须存活，强一致性破坏了可用性

最终一致性，**过半通过**。最常用的分布式一致性解决方案

Kafka选出quorum的方式略有不同，Kafka不是通过majority投票，而是动态维护了一组同步副本（ISR）。只有ISR集合的成员才有资格被选举为领导者。在ISR集合内的所有副本都写入之前，不会认为对Kafka分区的写入已提交。每当ISR集合发生变化时，这个ISR集合就会被持久化到Zookeeper。有了这个ISR模型和f + 1个副本，Kafka的主题可以容忍f个副本不可用而不会丢失已提交的消息。

- ISR（In-Sync Replicas）连通的&活跃的
- OSR（Out-Sync Replicas）超过阈值时间（10秒）没有心跳
- AR（Assigned Replicas）面向分区的副本集合，创建topic的时候给出了分区的副本数，那么controller在创建topic时就确定了broker和分区副本的对应关系，并得出了该分区的broker集合
- AR = ISR + OSR
- HW（High Water）最高水位。表示消费者能够消费的最大偏移
- LEO（Log End Offset）数据日志文件结束偏移量。只有副本的LEO大于主分区的HW时，该副本才有资格进入ISR集合

## Kafka ASKS
### ASKS为0
生产者不等待确认响应

### ASKS为1
生产者等待leader的确认响应

### ASKS为-1
只要ISR集合内的所有broker确认了消息则回复确认，follower在阈值时间内没有做出同步响应则会被移到OSR集合。在这种模式下多个broker的消费进度是一致的。

进一步思考，asks=-1就不会出现丢失消息的情况吗？答案是否。当ISR列表只剩Leader的情况下，asks=-1相当于asks=1，这种情况下如果节点宕机了，必然会出现数据丢失的情况。


在Java中，传统的I/O的flush方法是一个空实现，没有物理刷盘，而是依赖内核的dirty刷盘，所以会丢数据。

在APP层级
- 调用了OIO的write方法，这个时候数据只到达了内核，性能很快，但是会丢数据。
- 只有NIO的filechannel，调用write() + force()，才真正写到磁盘，性能极低。

kafka的索引文件保存着**部分**消息的偏移（offset）和位置（position）数据，类似于稀疏索引。当要寻找某一条消息时，kafka先通过**seek**函数将指针移动到距离目标消息最近的索引上，然后遍历寻找。最后通过sendfile将消息发送出去。


# Kafka与磁盘和网卡的技术点

# 案例

## Kafka 的一个节点宕机后为什么不可用？
Broker节点数是3，Topic副本数为3，Partition数为6，Asks参数为1。

当三个节点中某个节点宕机后，集群首先会怎么做？集群发现有Partition的Leader失效了，这个时候就要从ISR列表中重新选举Leader。如果ISR列表为空是不是就不可用了？并不会，而是从Partition存活的副本中选择一个作为Leader，不过这就有潜在的数据丢失的隐患了。

所以，只要将Topic副本个数设置为和Broker个数一样，Kafka的多副本冗余设计是可以保证高可用的，不会出现一宕机就不可用的情况（不过需要注意的是Kafka有一个保护策略，当一半以上的节点不可用时Kafka就会停止）。那仔细一想，Kafka上是不是有副本个数为1的Topic？

问题出在了__consumer_offset上，__consumer_offset是一个Kafka自动创建的Topic，用来存储消费者消费的offset（偏移量）信息，默认Partition数为50。而就是这个Topic，它的默认副本数为1。如果所有的Partition都存在于同一台机器上，那就是很明显的单点故障了！当将存储__consumer_offset的Partition的Broker给Kill后，会发现所有的消费者都停止消费了（__consumer_offset的Partition会出现只存储在一个Broker上而不是分布在各个Broker上）。