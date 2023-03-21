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
> 2. 快照：间隔的，内存全量数据基于某一个时间点做的的向磁盘的溢写。恢复速度快，但是容易丢失一部分数据。

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
- 安装JDK（使用rpm文件安装，命令：`rpm -i filename`）。**有些软件只认/usr/java/default**。
- 设置环境变量：JAVA_HOME、PATH
- 设置SSH免密。只要一台机器（B）的~/.ssh/authorized_keys文件中包含一台机器（A）的公钥，那么A就能免密登录B
```
  $ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
  $ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  $ chmod 0600 ~/.ssh/authorized_keys
```

## Hadoop配置

### 伪分布式配置
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
`cd   $HADOOP_HOME/etc/hadoop`

`vi hadoop-env.sh`，设置JAVA_HOME
```sh
export JAVA_HOME=/opt/jdk1.8.0_261
```

`vi core-site.xml`，给出NN角色在哪里启动
```xml
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://centos:9000</value>
</property>
```

`vi hdfs-site.xml`，配置HDFS
```xml
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
```

`vi slaves`，配置DN这个角色再那里启动  
centos

**初始化**
`hdfs namenode -format`

**启动**
`start-dfs.sh`