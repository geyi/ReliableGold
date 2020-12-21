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