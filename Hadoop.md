分治的思想

题目一：大小为1TB的包含N行字符串的文件，其中有两行字符串相同，如何使用有限的计算机资源找出这两行字符串？
解一：哈希表
解二：多机处理

题目二：如何使用有限的计算机资源对N个数字进行排序？（N >= 1000000000）
解一：内部无序，外部有序 -- 分治
解二：内部有序，外部无序 -- 归并

学习大数据技术时需要关心的重点：
- 分而治之
- 并行计算
- 计算向数据移动
- 数据本地化读取

# Hadoop

Modules
The project includes these modules:
- Hadoop Common: The common utilities that support the other Hadoop modules.
- Hadoop Distributed File System (HDFS™): A distributed file system that provides high-throughput access to application data.
- Hadoop YARN: A framework for job scheduling and cluster resource management.
- Hadoop MapReduce: A YARN-based system for parallel processing of large data sets.

理论知识点
- 存储模型
- 架构设计
- 角色功能
- 元数据持久化
- 安全模式
- 副本放置策略
- 读写流程
- 安全策略

### 存储模型
- 文件线性按**字节**切割成**块（block）**，具有offset，id
- 文件与文件的block大小可以不一样
- 一个文件除最后一个block，其他block大小一致
- block的大小依据硬件的I/O特性调整
- block被分散存放在集群的节点中，具有location（block的地址）
- block具有副本（replication），没有主从概念，副本不能出现在同一个节点
- 副本是满足可靠性和性能的关键
- 文件上传可以指定block大小和副本数，上传后只能修改副本数
- 一次写入多次读取，不支持修改
- 支持追加数据

### 架构设计
- HDFS是一个主从架构
- 由一个NameNode和一些DataNode组成
- 面向文件包含：文件数据（data）和文件元数据（metadata）
- NameNode负责存储和管理文件元数据，并维护了一个层次型的文件目录树
- DataNode负责存储文件数据（block），并提供block的读写
- DataNode与NameNode维持心跳，并汇报自己持有的block信息
- Client和NameNode交互文件元数据和DataNode交互文件block数据

### 角色功能
NameNode
- 完全基于内存存储文件元数据、目录结构、文件block的映射
- 需要持久化方案保证数据可靠性
- 提供**副本**放置策略

DataNode
- 基于本地磁盘存储block
- 并保存block的check sum以此来保证block的可靠性
- 与NameNode保持心跳，汇报block列表状态

### 元数据持久化
- 任何对文件系统元数据产生修改的操作，NameNode都会使用一种称为EditLog的事务日志记录下来
- 使用FsImage存储内存所有的元数据状态
- 使用本地磁盘保存EditLog和FsImage
- EditLog具有完整性高，数据丢失少，但恢复速度慢，并有体积膨胀的风险
- FsImage具有恢复速度快，体积与内存数据相当，但不能实时保存，数据丢失多
- NameNode使用了EditLog + FsImage整合的方案：滚动将增量的EditLog更新到FsImage，以此来保证更近时点的FsImage和更小的EditLog体积

> 保证数据可靠性的方式：
> 1. 日志文件：记录所有发生的增删改操作。完整性比较好，加载恢复数据慢。
> 2. 快照：间隔的，内存全量数据基于某一个时间点做的向磁盘的写入。恢复速度快，但是容易丢失一部分数据。

### 安全模式
- HDFS搭建时会格式化，格式化操作会产生一个空的FsImage
- 当NameNode启动时，它会从硬盘中读取FsImage和EditLog
- 将所有EditLog中的事务作用在内存中的FsImage上
- 并将这个新版本的FsImage从内存中保存到本地磁盘上
- 然后删除旧的EditLog，因为这个旧的EditLog的事务都已经作用的FsImage上了
- NameNode启动后会进入一个称为安全模式的特殊状态
- 处于安全模式的NameNode是不会进行数据块的复制的
- NameNode从所有的DataNode接收心跳信号和块状态报告
- 每当NameNode检测某个数据块的副本数目达到这个最小值，那么该数据块就会被认为是副本安全（safety replicated）的
- 在一定百分比（这个参数可配置）的数据块被NameNode检测确认是安全之后（加上一个额外的30秒等待时间），NameNode将退出安全模式状态
- 接下来它会确认还有哪些数据块的副本没有达到指定数目，并将这些数据块复制到其它DataNode上

### HDFS中的SNN
- SecondaryNameNode
- 在非HA模式下，SNN一般是独立的节点，周期的完成NameNode的EditLog向FsImage合并，减小EditLog的大小，减少NameNode的启动时间
- 根据配置文件设置的时间间隔（fs.checkpoint.period），默认3600秒
- 根据配置文件设置EditLog大小（fs.checkpoint.size），默认64MB

### Block的**副本**的放置策略
- 第一个副本：放置在上传文件的DataNode；如果是集群外提交，则随机挑选一台磁盘不太满，CPU不太忙的节点。
- 第二个副本：放置在与第一个副本不同机架的节点上。
- 第三个副本：与第二个副本相同机架的节点。
- 更多副本：随机节点。

### HDFS写流程
- Client和NameNode连接创建文件元数据
- NameNode判定元数据是否有效
- NameNode触发副本放置策略，返回一个有序的DataNode列表
- Client和DataNode建立Pipeline连接
- Client将块切分成packet（64KB），并使用chunk（512B）+ checksum（4B）填充
- Client将packet放入发送队列中，并向第一个DataNode发送
- 第一个DataNode收到packet后本地保存并发送给第二个DataNode
- 第二个DataNode收到packet后本地保存并发送给第三个DataNode
- 这一个过程中，上游节点同时发送下一个packet
- 生活中类比工厂的流水线；结论：流式其实也是变种的并行
- HDFS使用这种传输方式，副本数对于client是透明的
- 当block传输完成，DataNode各自向NameNode汇报，同时client继续传输下一个block
- 所以，client的传输和block的汇报也是并行的

### HDFS读流程
- 为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本
- 如果在读取程序的同一个机架上有一个副本，那么就读取该副本
- 如果一个HDFS集群跨越多个数据中心，那么客户端也将首先读本地数据中心的副本
- 语义：下载一个文件
  - Client和NameNode交互文件元数据获取FileBlockLocation
  - NameNode会按距离排序返回
  - Client尝试下载block并校验完整性
- 语义：下载一个文件其实是获取文件所有的block元数据，那么子集获取某些block应该成立
  - HDFS支持Client给出文件的offset自定义连接哪些block的DataNode，自定义获取数据（HDFS拥有读取文件任意block的能力）
  - 这个是支持计算层的分治，并行计算的核心


# Hadoop集群搭建
ssh除了可以远程登录，还可以远程执行命令  
学习新技术时，先把官方文档过一遍

## 基础设施
- 关闭防火墙和selinux
- 设置IP、主机名、hosts
- 设置时间同步（ntp）
- 安装JDK
- 设置环境变量：JAVA_HOME、PATH
- 设置SSH免密。只要一台机器（B）的~/.ssh/authorized_keys文件中包含一台机器（A）的公钥，那么A就能免密登录B
```
  $ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
  $ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  $ chmod 0600 ~/.ssh/authorized_keys
```
> 将本机的公钥发送给node02主机：ssh-copy-id -i id_rsa node02

## Hadoop配置

### 伪分布式
**解压压缩包，并移至opt目录**
```
mkdir /opt/bigdata
tar xf hadoop-2.6.5.tar.gz
mv hadoop-2.6.5  /opt/bigdata/
```

**设置HADOOP_HOME、PATH环境变量**  
`vi /etc/profile`，增加如下配置：
```
export HADOOP_HOME=/opt/bigdata/hadoop-2.6.5
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
使配置生效：`source /etc/profile`

**修改Hadoop配置**  
`cd $HADOOP_HOME/etc/hadoop`

`vi hadoop-env.sh`，设置JAVA_HOME
```sh
export JAVA_HOME=/opt/jdk1.8.0_261
```

`vi core-site.xml`，给出NN角色在哪里启动
```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://centos:9000</value>
  </property>
</configuration>
```

`vi hdfs-site.xml`，配置HDFS
```xml
<configuration>
  <!-- block的默认副本数 -->
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <!-- FsImage在本地文件系统中的存储位置。如果这是一个逗号分隔的目录列表，那么FsImage将被 复制到所有目录中，以实现冗余 -->
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/var/bigdata/hadoop/local/dfs/name</value>
  </property>
  <!-- block在本地文件系统中的存储位置 -->
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/var/bigdata/hadoop/local/dfs/data</value>
  </property>
  <!-- SNN http服务的地址和端口 -->
  <property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>centos:50090</value>
  </property>
  <!-- SNN在本地文件系统上存储要合并的临时映像的位置 -->
  <property>
    <name>dfs.namenode.checkpoint.dir</name>
    <value>/var/bigdata/hadoop/local/dfs/secondary</value>
  </property>
</configuration>
```

`vi slaves`，配置DN这个角色再那里启动  
```
centos
```

**初始化**
`hdfs namenode -format`

**启动**
`start-dfs.sh`

**HDFS COMMAND**
```
[root@centos ~]# hdfs dfs -help
Usage: hadoop fs [generic options]
	[-appendToFile <localsrc> ... <dst>]
	[-cat [-ignoreCrc] <src> ...]
	[-checksum <src> ...]
	[-chgrp [-R] GROUP PATH...]
	[-chmod [-R] <MODE[,MODE]... | OCTALMODE> PATH...]
	[-chown [-R] [OWNER][:[GROUP]] PATH...]
	[-copyFromLocal [-f] [-p] [-l] <localsrc> ... <dst>]
	[-copyToLocal [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
	[-count [-q] [-h] <path> ...]
	[-cp [-f] [-p | -p[topax]] <src> ... <dst>]
	[-createSnapshot <snapshotDir> [<snapshotName>]]
	[-deleteSnapshot <snapshotDir> <snapshotName>]
	[-df [-h] [<path> ...]]
	[-du [-s] [-h] <path> ...]
	[-expunge]
	[-get [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
	[-getfacl [-R] <path>]
	[-getfattr [-R] {-n name | -d} [-e en] <path>]
	[-getmerge [-nl] <src> <localdst>]
	[-help [cmd ...]]
	[-ls [-d] [-h] [-R] [<path> ...]]
	[-mkdir [-p] <path> ...]
	[-moveFromLocal <localsrc> ... <dst>]
	[-moveToLocal <src> <localdst>]
	[-mv <src> ... <dst>]
	[-put [-f] [-p] [-l] <localsrc> ... <dst>]
	[-renameSnapshot <snapshotDir> <oldName> <newName>]
	[-rm [-f] [-r|-R] [-skipTrash] <src> ...]
	[-rmdir [--ignore-fail-on-non-empty] <dir> ...]
	[-setfacl [-R] [{-b|-k} {-m|-x <acl_spec>} <path>]|[--set <acl_spec> <path>]]
	[-setfattr {-n name [-v value] | -x name} <path>]
	[-setrep [-R] [-w] <rep> <path> ...]
	[-stat [format] <path> ...]
	[-tail [-f] <file>]
	[-test -[defsz] <path>]
	[-text [-ignoreCrc] <src> ...]
	[-touchz <path> ...]
	[-usage [cmd ...]]
```

### 完全分布式
**角色规划**
host   | NN | SNN | DN
--     | -  | -   | -
node01 | √  |     |
node02 |    | √   | √
node03 |    |     | √
node04 |    |     | √

**免密登录**  
在哪台主机节点启动Hadoop集群（执行start-dfs.sh），那么这台主机就要对集群内的其它节点公开自己的公钥。

**修改配置**  
`cd $HADOOP_HOME/etc/hadoop`

`vi hadoop-env.sh`，设置JAVA_HOME
```sh
export JAVA_HOME=/opt/jdk1.8.0_261
```

`vi core-site.xml`，给出NN角色在哪里启动
```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://node01:9000</value>
  </property>
</configuration>
```

`vi hdfs-site.xml`，配置HDFS
```xml
<configuration>
  <!-- block的默认副本数 -->
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <!-- FsImage在本地文件系统中的存储位置。如果这是一个逗号分隔的目录列表，那么FsImage将被 复制到所有目录中，以实现冗余 -->
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/var/bigdata/hadoop/fully/dfs/name</value>
  </property>
  <!-- block在本地文件系统中的存储位置 -->
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/var/bigdata/hadoop/fully/dfs/data</value>
  </property>
  <!-- SNN http服务的地址和端口 -->
  <property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>node02:50090</value>
  </property>
  <!-- SNN在本地文件系统上存储要合并的临时映像的位置 -->
  <property>
    <name>dfs.namenode.checkpoint.dir</name>
    <value>/var/bigdata/hadoop/fully/dfs/secondary</value>
  </property>
</configuration>
```

`vi slaves`，配置DN这个角色再那里启动  
```
node02
node03
node04
```

**初始化**
`hdfs namenode -format`

**启动**
`start-dfs.sh`


# HDFS架构
NameNode：单点，数据一致性好掌握
问题：
- 单点故障，集群整体不可用
- 压力过大，内存受限

单点故障：多个NameNode，主备切换  
压力过大，内存受限：多个NameNode，管理不同的元数据。联邦机制，Federation（元数据分片）

Hadoop 2.x 只支持一主一备的HA

## HDFS-HA解决方案
NameNode的元数据：
1. client与NameNode交互操作产生的数据
2. DN提交的block的元数据

分布式中数据同步算法：
- 同步
- 异步
- 半同步
- Paxos

**CAP理论**

**HDFS-HA解决方案**
- 多台NN主备模式，主节点处于Active状态，备节点处于Standby状态。处于Active状态的节点对外提供服务。
- 增加了**JournalNode**角色，负责同步NN的EditLog。
- 增加了**ZKFC**角色，与NN同节点，同ZK集群协调NN的主从选举和切换。
- DN同时向多台NN同步block的元数据。

> 在HA模式中没有SNN

## HDFS-Federation解决方案
元数据分治，数据访问具有隔离性，复用DB存储。


# HDFS-HA搭建
**角色规划**
host   | NN | JNN | DN | ZKFC | ZK
--     | -  | -   | -  | -    | -
node01 | √  | √   |    | √    |
node02 | √  | √   | √  | √    | √
node03 |    | √   | √  |      | √
node04 |    |     | √  |      | √

**准备**
- node01、node02互相设置免密登录
- 搭建ZK集群

**修改配置**  
`vi core-site.xml`，给出NN角色在哪里启动
```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://mycluster</value>
  </property>
  <property>
    <name>ha.zookeeper.quorum</name>
    <value>node02:2181,node03:2181,node04:2181</value>
  </property>
</configuration>
```

`vi hdfs-site.xml`，配置HDFS
```xml
<configuration>
  <!-- 逻辑到物理节点的映射 -->
  <property>
    <name>dfs.nameservices</name>
    <value>mycluster</value>
  </property>
  <property>
    <name>dfs.ha.namenodes.mycluster</name>
    <value>nn1,nn2</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.mycluster.nn1</name>
    <value>node01:8020</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.mycluster.nn2</name>
    <value>node02:8020</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.mycluster.nn1</name>
    <value>node01:50070</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.mycluster.nn2</name>
    <value>node02:50070</value>
  </property>
  <!-- JN的启动位置及数据存储路劲 -->
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://node01:8485;node02:8485;node03:8485/mycluster</value>
  </property>
  <property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/var/bigdata/hadoop/ha/dfs/jn</value>
  </property>
  <!-- HA角色切换的代理类和实现方法 -->
  <property>
    <name>dfs.client.failover.proxy.provider.mycluster</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>
  <property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
  </property>
  <property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/root/.ssh/id_dsa</value>
  </property>
  <!-- 开启自动化故障转移 -->
  <property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property>
</configuration>
```

**在node01 02 03启动JN**：`hadoop-daemon.sh start journalnode`

**选择一台NN执行格式化**：`hdfs namenode -format`

**启动一台NN**：`hadoop-daemon.sh start namenode`

**在另一台NN执行同步命令**：`hdfs namenode -bootstrapStandby`

**格式化ZK**：`hdfs zkfc -formatZK`

**启动HDFS**：`start-dfs.sh`

# 使用非root用户完成以上操作
在Linux中root用户拥有对文件系统、内存、网络设置等方面的完全控制权，并且可以执行任意命令和程序。这使得root用户成为了Linux系统管理工作中不可或缺的一部分。

在Linux系统中，root用户通常被视为超级管理员（superuser），用于进行各种敏感操作。例如，安装软件包、配置系统环境、更改用户权限等操作都需要root用户才能执行。因此，在普通情况下，不建议使用root用户进行常规任务操作。

需要注意的是，由于root用户权限非常高，如果在误操作的情况下，会导致系统崩溃、数据丢失等严重后果。因此，在使用root用户时需要格外谨慎，遵循最佳实践来保护系统的安全性和稳定性。例如，可以使用sudo命令来限制root用户的使用范围，或者使用SELinux等安全机制来防止非法入侵和攻击。