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

分布式中唯一的一个问题：对某事达成一致

## 基础的复制算法
1. 主从异步复制：客服端收到一个**数据已经安全**的消息，跟**数据真正安全**（数据复制到全部机器上）在时间上有一个空隙。这段时间负责接收客户端请求的那台机器（master）如果发生了磁盘损坏，那数据就可能会丢失，因此这是一个不可靠的复制策略。
2. 主从同步复制：提供了完整的可靠性，直到数据真的安全的全部复制到所有机器上之后，master才告知客户端数据已经安全。但主从同步复制有一个致命的缺点就是整个系统中只要有任何一台机器宕机，写入就进行不下去了。相当于系统的可用性随着副本数量增加而降低。
3. 主从半同步复制：在同步与异常之间做一个折中，这就是半同步复制。他要求master在应答客户端之前必须把数据复制到足够多的机器上（1 <= x <= n），但不需要是全部。这样副本数够多可以提供比较高的可靠性，1台机器宕机也不会让系统停止写入。但是它还是不完美，例如数据a复制到slave-1，但没有到达slave-2；数据b复制到了slave-2但没有到达slave-1，这时如果master挂掉了需要从某个slave恢复出数据，任何一个slave都不能提供完整的数据。所以在整个系统中，数据存在某种不一致。
4. 多数派写（读）：为了解决半同步复制中的数据不一致的问题，可以将复制策略再做一个改进 -- 每条数据必须写入到半数以上的机器上，每次读取数据都必须检查半数以上的机器上是否有这条数据。客户端写入W >= N / 2 + 1个节点。多数派读 W + R > N; R >= N / 2 + 1

## 多数派写，后写入优胜
然而多数派读写的策略也有个但是，就是对于一条数据的更新时，会产生不一致的状态。例如:
- node-1，node-2都写入了a=x
- 下一次更新时node-2，node-3写入了a=y
这时，一个要进行读取a的客户端如果联系到了node-1和node-2, 它将看到2条不同的数据。

为了不产生歧义，多数派读写还必须给每笔写入增加一个全局递增的时间戳。更大时间戳的记录如果被看见，就应该忽略小时间戳的记录。

## PAXOS
但是又来了，这种带时间戳的多数派读写依然有问题。就是在客户端没有完成一次完整的多数派写的时候: 例如，上面的例子中写入，a=x₁写入了node-1和node-2，a=y₂时只有node-3写成功了，然后客户端进程就挂掉了，留下系统中的状态如下:
- node-1 a=x₁
- node-2 a=x₁
- node-3 a=y₂
这是整个系统对外提供的数据仍然是不一致的。paxos可以认为是多数派读写的进一步升级，paxos中通过2次原本并不严谨的多数派读写，实现了严谨的强一致consensus算法。

### 推导
通过一个incr操作来推导paxos对一致性问题的解决方法

inc操作在逻辑上很简单：读取一个变量的值i₁，给它加上一个数字得到i₂，再通过多数派把i₂写回到系统中。

很容易看出当两个线程同时执行自增操作时，存在着更新丢失的问题

看似通过写前读取能够解决自增操作的线程安全问题，但是！！！并发写前读取仍然有线程安全问题
> 写前读取：在一次写操作前，先进行一次读操作，以便确认是否有其它客户端在进行写操作

解决上面的问题，存储节点需要增加一个功能，就是它必须记住谁是最后一个做过写前读取的操作。并且只允许最后一个完成写前读取的进程可以进行后续写入，同时拒绝之前做过写前读取的进程写入的权限。

这个方法之所以能工作，也是因为多数派写中一个系统最多只能允许一个多数派写成功。

paxos就是通过2次多数派读写来实现的强一致。

术语描述：
- Proposer：发起paxos的进程
- Acceptor：存储节点
- Quorum：多数派
- Round：一次paxos算法实例，每个round是2次多数派读写：算法描述里分别用phase-1和phase-2标识。同时为了简单和明确，算法中也规定了每个Proposer都必须生成全局单调递增的round number，这样round number既能用来区分先后也能用来区分不同的Proposer
- last_rnd：标识Acceptor记住的最后一次进行写前读取的Proposer是谁，以此来决定谁可以在后面真正把一个值写到存储中。
- v：最后被写入的值
- vrnd：跟v是一对，它记录了在哪个Round中v被写入了

首先是paxos的phase-1，它相当于之前提到的写前读取过程。它用来在存储节点(Acceptor)上记录一个标识：我后面要写入，并从Acceptor上读出是否有之前未完成的paxos运行。如果有则尝试恢复它，如果没有则继续做自己想做的事情。

当Acceptor收到phase-1的请求时：
- 如果请求中rnd比Acceptor中的last_rnd小，则拒绝请求
- 否则将请求中rnd保存到本地的last_rnd（从此这个Acceptor只接受带有这个last_rnd的phase-2请求）
- 返回应答，并带上自己之前的last_rnd和之前已接受的v

当Proposer收到Acceptor发回的应答：
- 如果应答中的last_rnd大于rnd则退出
- 从所有应答中选择vrnd最大的v（不能改变（可能）已经确定的值）
- 如果所有应答的v都是空，可以选择自己要写入的v
- 如果应答不够多数派则退出

phase-2，Proposer将它选定的值写入到Acceptor中，这个值可能是它自己要写入的值，或者是它从某个Acceptor上读到的v（需要尝试进行修复的值）

Proposer：发送phase-2，带上rnd和phase-1决定的v

Acceptor：
- 拒绝rnd不等于Acceptor的last_rnd的请求
- 否则将phase-2请求中的v写入本地，记此v为已接受的值


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
1. 首先发现leader宕机的follower会发起leader选举，假设node03先发现，node03先投自己一票（node03投node03一票），然后会将(7, 3)广播出去。
2. node01接收到(7, 3)时与自己的(8, 1)比较后，淘汰(7, 3)，并将自己的(8, 1)发送回去（node03投node01一票）。
3. node02接收到(7, 3)时与自己的(8, 2)比较后，淘汰(7, 3)，并将自己的(8, 2)发送回去（node03投node02一票）。
4. node01紧接着投自己一票（node01投node01一票）并将自己的投票(8, 1)广播出去，node02淘汰node01。
5. node02紧接着投自己一票（node02投node02一票）并将自己的投票(8, 2)广播出去，node01接收到(8, 2)（node01投node02一票），最终node02被选举为Leader。
6. 最终所有节点都会投票给node2

其实，只要有节点发起投票，就一定会触发准leader发起自己的投票，并且最终所有节点都会把票投给leader。由此看出Zookeeper的选举是推选制的，所以Zookeeper能在故障时200ms内选出leader（快速恢复）。


**Redis比Zookeeper更快，但Zookeeper比Redis更可靠。Zookeeper更适合读多写少的场景。**

# 配置中心
参考basic项目com.airing.zookeeper.conf包下的demo

# 分布式锁
参考basic项目com.airing.zookeeper.lock包下的demo

1. 争抢锁，只有一个人能获得锁
2. 获得锁的服务如果宕机无法释放锁怎么办？（临时节点）
3. 获得锁的服务在任务完成后释放锁
4. 锁被释放时其他服务怎么知道？
   1. 主动轮询。弊端：延迟，压力。
   2. watch。解决了延迟问题，压力大的问题依然存在（当锁释放时，zk服务器需要回调watch它的所有客户端）。
   3. sequential + watch。每一个节点watch它的前一个节点，最小的节点获得锁。一旦最小的节点释放了锁，zk服务器只需要回调一个节点。