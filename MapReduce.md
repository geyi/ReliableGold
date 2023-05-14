# MapReduce
计算向数据移动。  
数据以一条记录为单位，经过map方法映射成KV，相同的key为一组数据作为reduce方法的输入，reduce方法内对这一组数据进行迭代计算。

Map：以一条记录为单位做映射。  
- 映射、过滤、变换
- 1进N出

Reduce：以一组为单位做计算。（抽取相同的特征）
- 分解、归纳、缩小
- 一组进N出

block 块（物理）
split 切片（逻辑）

默认情况下 切片 == 块

切片与块的关系有：> < =（为了控制并行度）

Reduce的并行度由开发者来决定，默认是1。

block:split
- 1:1
- N:1
- 1:N

split:map
- 1:1

map:reduce
- N:1
- N:N
- 1:1
- 1:N

group:partition
- 1:1
- N:1

## Map Task And Reduce Task
1. 切片会格式化出记录，以记录为单位调用map方法。 
2. map的输出映射成KV，然后拿着Key算出Partition。
3. 内存缓冲区溢写磁盘时，做一个2次排序，分区有序，且分区内key有序，未来相同的一组key会相邻的排在一起。
4. reduce的归并排序其实可以和reduce方法的计算同时发生，尽量减少IO。

> MapTask的输出是一个文件，存储在本地的文件系统中。

map和reduce是一种阻塞关系

## MapReduce中的角色

### JobTracker
- 资源管理
- 任务调度

### TaskTracker
- 任务管理
- 资源汇报

### Client
1. client会根据每次的计算数据，向NameNode咨询计算数据的元数据，根据block计算出split清单。block具有offset和location信息，映射到split，split会包含offset，以及split对应的map任务应该移动到哪些节点（location）
2. client生产计算程序未来运行时的相关配置文件（）
3. client会将jar，split清单，配置文件上传到HDFS的目录中（副本数10）
4. client会调用JobTracker，通知要启动一个计算程序了，并且告知文件都放在HDFS的哪些地方

JobTracker收到启动程序之后：
1. 从HDFS中取回split清单
2. 根据自己收到的的TaskTracker汇报的资源，最终确定每一个split对应的map应该去到哪一个节点
3. 未来TaskTracker在心跳的时候会取回分配给自己的任务信息

JobTacker的问题：单点故障、压力过大、资源管理与任务调度耦合

TaskTracker
1. 在心跳取回任务
2. 在HDFS中下载jar，配置文件到本机
3. 启动任务描述中的MapTask/ReduceTask

## YARN

### 模型
container
逻辑的：
物理的：JVM进程。
- NodeManager会有线程监控container资源使用情况，超额，NodeManager直接kill掉。
- cgroup内核级技术，在启动JVM进程时，由kernel约束资源

### 实现
ResourceManager：负责整体资源的管理
NodeManager：向ResourceManager汇报心跳，提交自己的资源情况

MapReduce On Yarn
1. MR-Client（jar、切片清单、配置文件，上传到HDFS），访问RM申请AppMaster
2. RM选择一台相对不忙的节点通知NM启动一个Container，在里面反射一个MRAppMaster
3. 启动MRAppMaster，从HDFS下载切片清单，向RM申请资源
4. 由RM根据自己掌握的资源情况得到一个确定清单，通知NM来启动Container
5. Container启动后会反向注册到MRAppMaster进程
6. MRAppMaster（不带资源管理的阉割版的JobTracker）将任务消息发送给Container
7. Container会反射相应的Task类为对象，调用方法执行，其结果就是我们的业务逻辑代码的执行
8. 计算框架都有Task失败重试的机制

问题解决：
1. 单点故障（JobTracker挂了整个结算层没有了调度）  
   yarn：每个计算程序有一个自己的AppMaster调度，支持失败重试
2. 压力过大  
   yarn中每个计算程序自有AppMaster，每个AppMaster只负责自己计算程序的任务调度
3. 资源管理与任务调度耦合  
   yarn只负责资源管理，不负责具体的任务调度。只要计算框架继承yarn的AppMaster，大家都可以使用一个统一视图的资源层

> JT，TT是MR的常服务。2.x之后没有了这些服务，相对的，MR的client、调度、任务，这些都是临时服务