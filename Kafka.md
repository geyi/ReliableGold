在同一个topic下
- 无关的数据就分散到不同的分区里，以追求并发并行
- 有关的数据，一定要按原有顺序发送到同一个分区里

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

# 有序性
消息是K-V，相同的key一定去到同一个分区里（hash路由）。borker会保证producer推送的消息的顺序，一个分区可能有不同的key，且不同的key是交叉的（相同的key在一个分区里没有排列在一起）

# 推送拉取
推送：推送说的是server主动将消息推送给消费者，如果推送速率大于消费者的消费能力，那么消费者网卡会被打满。如果通过反馈机制控制推送速率，那么消息就没有了实时性。  
拉取：消费者自主，按需去订阅拉取server的数据。

## 拉取粒度
批次拉取，如何维护offset

单线程，一条一条按顺序处理消息，并更新offset  
多线程，流式的多线程，能多线程的多线程（更多的去利用CPU、网卡等硬件资源），但是整个批次的事务环节交给一个线程，做到这个批次要么成功，要么失败。减少DB的压力，和offset的更新压力。


# Kafka与磁盘和网卡的技术点