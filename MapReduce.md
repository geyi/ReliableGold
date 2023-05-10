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