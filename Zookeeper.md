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

使用文件系统目录树结构的数据模型

数据保存在内存中

持久节点
临时节点：session

读请求由每个服务器数据库的本地副本处理
写请求被转发到leader处理