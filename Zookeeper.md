# Zookeeper

分布式协调服务

提供了一组简单的原语
- create
- delete
- exists
- get data
- set data
- get children
- sync

特征/保障
- 顺序一致性：客户端的更新将按发送顺序应用。
- 原子性：更新成功或失败，没有部分结果。
- 统一视图：无论客户端连接到哪个服务器，客户端都将看到相同的服务视图。
- 可靠性：一旦应用了更新，它将从那时起持续到客户端覆盖更新。
- 及时性：系统的客户视图保证在特定时间范围内是最新的。

使用文件系统目录树结构的数据模型

数据保存在内存中

持久节点
临时节点：session

读请求由每个服务器数据库的本地副本处理
写请求被转发到leader处理

节点存储的信息
- czxid 创建的事务ID，前32位表示leader的纪元，后32位表示一个自增顺序
- pzxid 当前节点下最后被创建的子节点的czxid
- ephemeralOwner 临时节点所属的客户端
- 客户端连接zk节点时被连接的节点会通过leader将session ID写入所有节点，这时会消耗一个事务ID。同样客户端断开连接触发删除时也会消耗一个事务ID。

# 应用
- 统一配置管理 <- 1MB数据
- 分组管理 <- path结构
- 统一命名 <- sequential
- 同步 <- 临时节点

# 安装Zookeeper
```
准备node01-node04四台服务器
安装JDK，并设置JAVA_HOME
wget https://mirrors.bfsu.edu.cn/apache/zookeeper/zookeeper-3.6.2/apache-zookeeper-3.6.2-bin.tar.gz
tar -zxf apache-zookeeper-3.6.2-bin.tar.gz
mv apache-zookeeper-3.6.2-bin /opt/
vi /etc/profile
export ZOOKEEPER_HOME=/opt/apache-zookeeper-3.6.2-bin
export PATH=$PATH:$ZOOKEEPER/bin
source /etc/profile
cd apache-zookeeper-3.6.2-bin/conf
cp zoo_sample.cfg zoo.cfg
vi zoo.cfg
dataDir=/var/zookeeper
server.1=node01:2888:3888
server.2=node02:2888:3888
server.3=node03:2888:3888
server.4=node04:2888:3888
mkdir /var/zookeeper
echo 1 > /var/zookeeper/myid
cd /opt && scp ./zookeeper/ node02:`pwd`
node02-node04创建myid
启动顺序1，2，3，4
zkServer.sh start-forceground
```

> 2888端口用于leader接受write请求，3888端口用于选主投票

# 连接及操作
```
help
ls /
create /hello ""
create -s /aaa/bbb（创建顺序节点）
create -e /aaa/ccc（创建临时节点）
get /hello
```

netstat -natp | grep -E '(2888|3888)'

# PAXOS
提议：议员只会接收大于当前编号的提议（包括已生效的和未生效的），每个提议都需要过半的议员同意才能生效。  
提议编号：编号是一直增长的，不能倒退。  
2PC：议员发起提议并收到超过半数的议员回复，就立即向所有议员发送通知，提议生效。收到生效消息的议员会将记录改成正式的法令。


# ZK Cluster

## 创建请求
假设在一个有三个节点的ZK集群中（1个leader，2个follower）
1. client发起创建请求
2. 如果client连接的是follower，follower会将创建请求转发给leader
3. leader自己维护了两个队列（先进先出），分别对应两个follower。leader通过两个队列将创建请求分别发送给两个follower。
4. follower接收到创建请求并记录日志，最后回复leader一个成功的状态。
5. 在此集群中只要有一个follower返回成功的状态，leader便告诉两个follower创建请求生效。

## Leader选举
**ZXID大的优先为Leader，如果相同则比较myid，myid大的优先作为Leader**

假设在一个有四个节点的ZK集群中（1个Leader，3个Follower）
- node01 (8, 1) Follower
- node02 (8, 2) Follower
- node03 (7, 3) Follower
- node04 (8, 4) Leader 宕机
1. 首先发现leader宕机的follower会发起leader选举，假设node03先发现，它会将(7, 3)广播出去。
2. node01接收到(7, 3)时与自己的(8, 1)比较，这时不需要修改自身的投票，仍然保持(8, 1)。
3. node01紧接着将自己的投票(8, 1)广播出去，同样node02不需要修改自身的投票，node03则会将自身投票修改为(8, 1)，并广播出去，但此次广告不会引起任何投票变化。
4. node02接收到(7, 3)时与自己的(8, 2)比较，这时不需要修改自身的投票，仍然保持(8, 2)。
5. node02紧接着将自己的投票(8, 2)广播出去，node01会将自身的投票修改(8, 2)，并广播出去。此时三个节点中已经有两个节点（node01, node02）投票(8, 2)，最终node02被选举为Leader。


Redis比Zookeeper更快，但Zookeeper比Redis更可靠。Zookeeper更适合读多写少的场景。