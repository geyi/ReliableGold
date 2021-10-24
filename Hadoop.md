分治的思想

题目一：大小为1TB的包含N行字符串的文件，其中有两行字符串相同，如何使用有限的计算机资源找出这两行字符串？
解一：哈希表
解二：多机处理

题目二：如何使用有限的计算机资源对N个数字进行排序？（N >= 1000000000）
解一：内部无序，外部有序 -- 分治
解二：内部有序，外部无序 -- 归并

分而治之
并行计算
计算向数据移动
数据本地化读取

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
- block被分散存放在集群的节点中，具有location
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

> 保证数据可靠性的方式：
> 1. 日志文件：记录所有发生的增删改操作。完整性比较好，加载恢复数据慢。
> 2. 快照：间隔的，内存全量数据基于某一个时间点做的的向磁盘的溢写。恢复速度快，但是容易丢失一部分数据。