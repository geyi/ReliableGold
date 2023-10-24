# 性能监控

## 查看sql语句的执行时间
```sql
set profiling=1;
select * from test;
show profiles;
show profile;
-- 查看query id为2的sql的执行时间
show profile for query 2;
```

## performance_schema
MySQL的Performance Schema是一个功能，它提供了MySQL数据库服务器内部操作和性能特征的全面视图。它允许您监视和分析服务器活动的各个方面，例如查询执行时间、资源使用情况、锁定等等。

要在MySQL中启用Performance Schema，可以按照以下步骤进行操作：

1. 执行以下查询，检查Performance Schema是否已启用：
   ```
   SELECT @@performance_schema;
   ```

2. 如果结果是`OFF`，则需要通过编辑MySQL配置文件（通常为`my.cnf`或`my.ini`）来启用它。在`[mysqld]`节下添加或取消注释以下行：
   ```
   performance_schema=ON
   ```

3. 保存对配置文件的更改，并重新启动MySQL服务器以使新设置生效。

一旦启用了Performance Schema，您可以开始使用它来收集有关MySQL服务器性能的信息。一些常见的使用Performance Schema的方式包括：

- 查询分析：Performance Schema提供了详细的查询执行时间信息，帮助您识别慢查询并进行优化。

- 资源监控：您可以跟踪每个查询的CPU、内存和I/O使用情况，帮助您识别消耗大量资源的查询，从而影响服务器性能。

- 锁分析：Performance Schema可以帮助识别锁争用情况，并分析由于锁定而导致的延迟的查询。

- 用户和会话统计：您可以收集有关用户活动、连接和其他会话相关指标的数据，以了解应用行为和资源利用模式。

- 仪表化和跟踪：Performance Schema允许您对代码进行仪表化并跟踪MySQL服务器内部的函数调用和事件。

请注意，启用Performance Schema可能会对服务器性能产生轻微影响，特别是对于非常繁忙的系统。因此，建议在需要时谨慎使用，并在不需要时禁用它。

## show processlist;


# Schema与数据类型优化

## 更小的通常更好
应该尽量使用能够正确存储数据的最小数据类型，例如：
- int和tinyint的使用。当字段的取值只有0、1时应该使用tinyint类型。

## 简单就好
简单数据类型的操作通常需要更少的CPU资源，例如：
- 整型比字符串的比较成本更低，因为字符集和校对规则使字符比较比整型比较更复杂。
- 使用MySQL自建类型而不是字符串来存储日期和时间。
- 使用整型存储IP地址（INET_ATON(ip)）

## 尽量避免null
- 负向比较（!=）会引发全表扫描。
- 如果允许空值，不等于（!=）的查询，不会将空值行（row）包含进来，此时的结果集往往是不符合预期的，此时往往要加上一个or条件，把空值（is null）结果包含进来。
- NULL通过任一操作符与其它值比较都会得到NULL，除了<=>。
- 任何有返回值的表达式中有NULL参与时，都会得到另外一个NULL值。例如：`null + 10`，`concat('abc', null)`
- 使用`count(*)`或者`count(null column)`结果不同，`count(null column) <= count(*)`。
- 虽然`select NULL=NULL`的结果为false，但是在我们使用`distinct`，`group by`，`order by`时，NULL又被认为是相同值。
- MySQL中支持在含有NULL值的列上使用索引，但是Oracle不支持，这就是我们平时所说的如果列上含有NULL那么将会使索引失效。

## 数据类型

### 整型
可以使用的几种整型类型：TINYINT，SMALLINT，MEDIUMINT，INT，BIGINT分别使用8，16，24，32，64位存储空间。应尽量使用满足需求的最小数据类型。

### 字符类型
1. CHAR长度固定，即每条数据占用等长字节空间；最大长度是255个字符，适用于存储身份证号、手机号等定长的字符串。
2. VARCHAR可变长度，长度可以指定为0到65535之间的值，适用于存储长度差异较大的字符串。与CHAR相比，VARCHAR值存储为1字节或2字节长的前缀 + 数据。如果数据长度不超过255个字节，则前缀使用一个长度字节；否则使用两个长度字节。
3. TEXT不设置长度，当不知道属性的最大长度时，适合用TEXT。
4. BLOB采用二进制的存储方式。

### 时间类型
1. datetime，占用8个字节，与时区无关，可保存到毫秒，可保存的时间范围大（1000-01-01 00:00:00 ~ 9999-12-31 23:59:59）
2. timestamp，占用4个字节，依赖数据库设置的时区，精确到秒，时间范围（1970-01-01 00:00:01 ~ 2037-12-31 23:59:59）
3. date，占用3个字节，时间范围（1000-01-01 ~ 9999-12-31）

## 范式和反范式

## 主键

## 字符集

## 存储引擎


# 执行计划
蚂蚁笔记 - MySQL查询优化器


# 索引

## 数据结构
1. 二叉树
2. AVL树（AVL树得名于它的发明者）
3. 红黑树
4. B数
5. B+数
6. 哈希表

## 索引的用处
- 大大减少查询时需要扫描的数据量
- 避免文件排序和使用临时表
- 随机I/O变成顺序I/O（可以想到的是，使用主键或者使用普通索引并且触发了索引覆盖时的范围查询是顺序I/O，而使用普通索引未触发索引覆盖的查询则是随机I/O）
- 快速查找匹配WHERE子句的行
- 从搜索范围内中消除行，如果可以在多个索引之间进行选择，MySQL通常会使用找到最少行的索引
- 如果表具有多列索引，则优化器可以使用索引的任何最左前缀来查找行
- 在执行连接时从其他表检索行（VARCHAR(10)和CHAR(10)被认为是相同的，但VARCHAR(10)和CHAR(15)不是）
- 查找特定索引列的min或max值
- 如果排序或分组时在可用索引的最左前缀上完成的，则对表进行排序和分组
- 在某些情况下，可以优化查询以检索值而无需查询数据行（索引覆盖）

> When a query needs to access most of the rows, reading sequentially is faster than working through an index. Sequential reads minimize disk seeks, even if not all the rows are needed for the query.  
> 当查询需要访问大部分行时，顺序读取比通过索引读取要快。顺序读取可以最小化磁盘搜索，甚至在不需要查询所有行时。

### 全表扫描
全表扫描通常发生在以下条件下：
- 表非常小，执行表扫描比按索引查找要快。这对于行数少于10行且行长较短的表很常见。
- 有在没有ON或WHERE索引列的可用限制时。
- 正在将索引列与常量值进行比较，并且MySQL已经计算出（基于索引树）常量覆盖了表的大部分，并且表扫描会更快。
- 正在使用基数（Cardinality）较低的键（许多行与键值匹配）的列。在这种情况下，MySQL假定通过使用键可能会执行许多键查找并且表扫描会更快。

对于小表，表扫描通常是合适的，性能影响可以忽略不计。对于大表，请尝试以下技术以避免优化器错误地选择表扫描：
- 使用`ANALYZE TABLE tbl_name`更新扫描表的键分布。
- 使用`FORCE INDEX`告诉MySQL该表扫描是非常昂贵的相比使用给定的索引：`SELECT * FROM t1, t2 FORCE INDEX (index_for_column) WHERE t1。col_name=t2。col_name;`
- 使用带有--max-seeks-for-key=1000选项的mysqld或使用SET max_seeks_for_key=1000告诉优化器：如果扫描全表时，扫描行数已达到1000行仍未找到匹配行，那么就使用索引扫描。

## 索引分类
- 主键索引
- 唯一索引
- 普通索引
- 组合索引
- 全文索引

## 技术名词
- 回表
- 最左匹配
- 索引覆盖（dcovering inex）
- 索引下推（index condition pushdown）

## MySQL中索引使用的数据结构
- B+树
- Hash表

## 索引匹配方式
- 全值匹配：与索引中的所有列进行匹配。
- 最左前缀匹配
- 列前缀匹配（'abc%'）
- 范围匹配
- 精准匹配某一列并范围匹配另一列：建立组合索引(a, b, c)。查询条件where a = 1 and b > 2 and c = 3，只会使用a，b索引列。即，范围列可以用到索引，但是范围列后面的列无法用到索引，索引最多用于一个范围列。
- 只访问索引列的查询（索引覆盖）

> source *.sql

## 哈希索引
特征：
- 仅用于使用=或者<=>运算符的比较中，不用于范围查找的运算符。
- 优化器无法使用哈希索引来加速ORDER BY操作。（此索引类型不能用于按顺序搜索下一个条目）
- MySQL无法确定两个值之间大约有多少行。
- 仅整个键可用于搜索行。（对于B树索引，键的任何最左边的前缀都可用于查找行）

## 组合索引

## 聚簇索引与非聚簇索引
应该理解为一种数据存储方式

## 索引覆盖

## 索引优化
- 当使用索引列进行查询的时候尽量不要使用表达式，把计算放到业务层而不是数据库层
- 尽量使用主键查询，而不是其他索引，因为主键查询不会触发回表查询
- 使用前缀索引（索引字段不宜太长）
  ```sql
  select
      count(distinct left(city,7))/count(*) as sel7,
      count(distinct left(city,8))/count(*) as sel8
  from
      city;
  ```
- 使用索引扫描来排序  
  以下两种情况不会使用索引排序
  ```sql
  where a > 1 order by b, c
  where a = 1 order by b asc, c desc
  ```
- union all, in, or都能够使用索引，但是推荐使用in
- 范围列可以用到索引
- 强制类型转换会导致全表扫描
- 更新十分频繁（页合并与页分裂），数据区分度不高的字段上不宜建立索引
- 创建索引的列，不允许为null，可能会得到不符合预期的结果（null不等于null）
- 当需要进行表连接时，关联条件字段的数据类型必须一致
- 能使用limit的时候尽量使用limit（关于MySQL对LIMIT查询的优化https://dev.mysql.com/doc/refman/5.7/en/limit-optimization.html）
- 单表索引个数建议控制在5个以内
- 组合索引字段数不允许超过5个
- 索引并不是越多越好

> join查询中on条件后的and条件用于在表连接前过滤表中哪些记录符合连接条件

> 块嵌套循环连接算法（Block Nested-Loop Join Algorithm）使用外部循环中读取的行的缓冲来**减少读取内部循环中的表的次数**。例如，如果将10行读入缓冲区并将缓冲区传递到下一个内部循环，则可以将内部循环中读取的每一行与缓冲区中的所有10行进行比较。这将内部表必须读取的次数减少了一个数量级。

## 索引监控
show status like 'Handler_read%';
- Handler_read_key：通过索引获取数据的次数。如果这个值很高，说明为现有查询建立了正确的索引。
- Handler_read_first：读取索引中第一个条目的次数。如果此值很高，则表明服务器正在执行大量完整索引扫描（例如SELECT col1 FROM foo，假设col1已编入索引）。
- Handler_read_last：读取索引最后一个条目的次数。该值增加基本上可以判定查询中使用了基于索引的order by desc子句。例如：基于主键的正向排序，会增加Handler_read_first和Handler_read_next。而基于主键的反向排序，会增加Handler_read_last和Handler_read_prev。
- Handler_read_next：通过索引读取下一行数据的次数，常用于基于索引的范围扫描和order by limit子句中。
- Handler_read_prev：通过索引读取上一行数据的次数，常用于基于索引的范围扫描和order by limit子句中。
- Handler_read_rnd：从固定位置读取数据的次数。如果您要执行很多需要对结果进行排序的查询，则此值很高。您可能有很多查询需要MySQL扫描整个表，或者您的联接没有正确使用键。
- Handler_read_rnd_next：从数据节点读取下一条数据的次数。如果要执行大量表扫描，则此值很高，这通常表明没有未表建立正确的索引，或者查询语句未使用现有的索引。

### 总结
1. Handler_read_key的值越大越好，代表基于索引的查询较多。
2. Handler_read_first，Handler_read_last，Handler_read_next，Handler_read_prev都会利用索引。但查询是否高效还需要结合其它Handler_read值来判断。
3. Handler_read_rnd不宜过大。
4. Handler_read_rnd_next不宜过大，过大的话，代表全表扫描过多，要引起足够的警惕。

# 查询优化

## 查询慢的原因
- 网络I/O
- 磁盘I/O
- CPU
- 上下文切换
- 系统调用
- 锁等待时间
- 生成统计信息

## 优化特定类型的查询

### 优化limit分页
- 优化前的SQL语句：select * from t_employeelog_logdetail tel limit 48000, 10;
- 优化后的SQL语句：select * from t_employeelog_logdetail tel inner join (select FID from t_employeelog_logdetail tel2 limit 48000, 10) as t2 using(fid);

优化此类查询的最简单的办法就是尽可能地使用覆盖索引，而不是查询所有的列。

### 优化union查询
除非确实需要MySQL服务器消除重复的行，否则一定要使用UNION ALL，因为在没有ALL关键字时，则相当于UNION DISTINCT，MySQL会使用临时表去重，这个操作的成本很高。


# 分区表

In MySQL, the InnoDB storage engine has long supported the notion of a tablespace, and the MySQL Server, even prior to the introduction of partitioning, could be configured to employ different physical directories for storing different databases (see Section 8.12.3, “Using Symbolic Links”, for an explanation of how this is done).  
在MySQL中，InnoDB存储引擎长期以来一直支持表空间的概念，而且在引入分区之前，MySQL服务器就支持为不同的数据库配置不同的物理空间。

> 为不同的数据库配置不同的物理空间类似于**分库**

Partitioning takes this notion a step further, by enabling you to distribute portions of individual tables across a file system according to rules which you can set largely as needed. In effect, different portions of a table are stored as separate tables in different locations.   
分区使您可以根据设置的规则在文件系统中分布各个表的各个部分，从而使这一概念更进一步。实际上，表的不同部分作为单独的表存储在不同的位置。

> 分区使一个表的不同部分存储在不同的位置类似于**分表**

## 类型
- 范围分区（RANGE Partitioning）：这种类型的分区根据列值在给定范围内将行分配给分区。
- 列表分区（LIST Partitioning）：类似于范围分区，两种类型的分区的主要区别在于，在列表分区中，每个分区都是基于一组值列表中的一个而不是一组连续范围中的列值来定义和选择分区的。
- 列分区（COLUMNS Partitioning）：
- 哈希分区（HASH Partitioning）：根据用户定义的表达式返回的值来选择一个分区，该表达式对将要插入表中的行的列值进行操作。
- 键分区（KEY Partitioning）：类似于哈希分区，不同之处在于，仅提供一个或多个要评估的列，并且MySQL服务器提供了自己的哈希函数。这些列可以包含非整数值，因为MySQL提供的哈希函数可以保证整数结果，而与列数据类型无关。

> https://dev.mysql.com/doc/refman/5.7/en/partitioning-types.html

## 优点
- 分区使在一个表中存储的数据比在单个磁盘或文件系统分区中存储的数据更多。(因为不同的分区可以存储在不同的物理设备上，进而也避免了单个物理设备带来的性能瓶颈)

- 通常，通过删除仅包含该数据的一个或多个分区，可以轻松地从分区表中删除失去其用途的数据。相反，在某些情况下，通过添加一个或多个用于专门存储该数据的新分区，可以大大简化添加新数据的过程。

- 满足以下条件的某些查询可以大大优化：满足给定WHERE子句的数据只能存储在一个或多个分区上（即不是所有分区上），这会自动从搜索中排除任何剩余的分区。由于可以在创建分区表之后更改分区，因此您可以重新组织数据以增强（优化，适应）在首次设置分区方案时可能不经常使用的频繁查询。这种排除不匹配分区（以及因此包含的任何行）的能力通常称为**分区修剪**。有关更多信息，请参见第21.4节“分区修剪”。

- 另外，MySQL支持显式的分区选择查询。例如，`SELECT * FROM t PARTITION (p0,p1) WHERE c < 5`仅选择那些在分区行p0和p1其匹配的WHERE条件。在这种情况下，MySQL不检查table的任何其他分区；当您已经知道要检查的分区时，这可以大大加快查询速度。选择分区还支持数据修改语句DELETE，INSERT，REPLACE，UPDATE和LOAD DATA，LOAD XML。有关更多信息和示例，请参见这些语句的描述。

## 限制
- 最大分区数：对于不使用NDB存储引擎的给定表，最大分区数为8192。此数目包括子分区。
- 分区键的数据类型：分区键必须是整数列或可解析为整数的表达式。

> https://dev.mysql.com/doc/refman/5.7/en/partitioning-limitations.html


# MySQL服务器参数设置

## general
```sql
show variables like 'datadir';
-- 服务器与本地客户端进行通信的套接字文件的位置
show variables like 'socket';
show variables like 'pid_file';
show variables like 'port';
show variables like 'default_storage_engine';
```

## character
```sql
show variables like 'character_set_client';
show variables like 'character_set_connection';
show variables like 'character_set_results';
show variables like 'character_set_database';
show variables like 'character_set_server';
```

## connection
```sql
-- 最大连接数
show variables like 'max_connections';
-- 每个用户的最大连接数
show variables like 'max_user_connections';
-- 可缓存等待连接的最大数
show variables like 'back_log';
-- MySQL关闭一个非交互式连接前需要等待的时间
show variables like 'wait_timeout';
-- MySQL关闭一个交互式连接前需要等待的时间
show variables like 'interactive_timeout';
```

## log
```sql
-- 系统变量
-- 指定错误日志文件名称，用于记录当MySQL启动和停止时，以及服务器在运行中发生任何严重错误时的相关信息
show variables like 'log_error';
-- 二进制日志记录选项和变量
-- 是否开启二进制日志记录（主从复制，备份恢复）
show variables like 'log_bin';
-- 二进制日志记录选项和变量
-- 二进制日志文件的基本名称和路径
show variables like 'log_bin_basename';
-- 主节点状态
mysql > SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000003 | 73       | test         | manual,mysql     |
+------------------+----------+--------------+------------------+
Binlog_Do_DB：指定将更新记录到二进制日志的数据库，其他所有没有显式指定的数据库更新将忽略，不记录在日志中
Binlog_Ignore_DB：指定不将更新记录到二进制日志的数据库
-- 二进制日志记录选项和变量
-- 控制MySQL服务器将二进制日志同步到磁盘的频率
-- 0禁用MySQL服务器将二进制日志同步到磁盘的功能。取而代之的是，MySQL服务器依靠操作系统将二进制日志刷新到磁盘上，就像处理任何其他文件一样。此设置可提供最佳性能，但是在电源故障或操作系统崩溃的情况下，服务器可能已提交尚未同步到二进制日志的事务。
-- 1在提交事务之前将二进制日志刷新到磁盘。这是最安全的设置，但是由于磁盘写入次数增加，可能会对性能产生负面影响。如果发生电源故障或操作系统崩溃，二进制日志中缺少的事务将仅处于准备状态。这允许自动恢复例程回滚事务，从而保证二进制日志中不会丢失任何事务。
-- N在N次二进制日志提交之后，二进制日志将同步到磁盘 。在电源故障或操作系统崩溃的情况下，服务器可能提交了尚未刷新到二进制日志的事务。由于磁盘写入次数的增加，此设置可能会对性能产生负面影响。较高的值可以提高性能，但会增加数据丢失的风险。
show variables like 'sync_binlog';
-- 系统变量
-- 是否开启记录查询日志
show variables like 'general_log';
-- 系统变量
-- 查询日志文件的名称。默认值为 host_name.log
show variables like 'general_log_file';
-- 系统变量
-- 是否开启记录慢查询日志
show variables like 'slow_query_log';
-- 系统变量
-- 查询日志文件的名称。默认值为 host_name-slow.log
show variables like 'slow_query_log_file';
-- 系统变量
-- 被标记为慢查询语句的阈值
show variables like 'long_query_time';
-- 系统变量
-- 是否将管理语句记录到慢查询日志，包括ALTER TABLE， ANALYZE TABLE， CHECK TABLE， CREATE INDEX， DROP INDEX， OPTIMIZE TABLE和REPAIR TABLE
show variables like 'log_slow_admin_statements';
```

## cache
```sql
-- https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html
-- 索引缓存区的大小（只对myisam表起作用）
show variables like 'key_buffer_size';
-- 查询缓存区的大小（从MySQL 5.7.20开始，查询缓存已弃用，并在MySQL 8.0中删除）
show variables like 'query_cache_size';
-- 不缓存超出此字节数的查询结果
show variables like 'query_cache_limit';
-- 查询缓存分配的块的最小大小（以字节为单位）。默认值为4096（4KB）
show variables like 'query_cache_min_res_unit';
-- 查询缓存的类型
-- 0禁用查询缓存，但这不会取消分配查询缓冲区。为此，应将query_cache_size设置为0
-- 1缓存所有可缓存的查询结果，除非在SELECT语句中使用了SQL_NO_CACHE
-- 2仅缓存使用了SQL_CACHE关键字的查询语句
show variables like 'query_cache_type';
-- 排序缓冲区大小，每个必须执行排序的会话都会分配此大小的缓冲区
show variables like 'sort_buffer_size';
-- 数据包的最大大小
show variables like 'max_allowed_packet';
-- 用于普通索引扫描，范围索引扫描和不使用索引从而执行全表扫描的联接的缓冲区的最小大小。通常，获得快速联接的最佳方法是添加索引。join_buffer_size当无法添加索引时，请增加该值的大小以获得更快的完全联接。为每个多表之间的完全联接分配一个联接缓冲区。对于不使用索引的多个表之间的复杂联接，可能需要多个联接缓冲区。
show variables like 'join_buffer_size';
-- 服务器应缓存多少线程以供重用
show variables like 'thread_cache_size';
```

## InnoDB
```sql
-- 数据和索引的缓存池大小
show variables like 'innodb_buffer_pool_size';
-- 0 每秒写page cache并刷磁盘
-- 1 每次写page cache并刷磁盘
-- 2 每次写page cache，每秒刷磁盘
show variables like 'innodb_flush_log_at_trx_commit';
-- 并发线程数，0表示不限制
show variables like 'innodb_thread_concurrency';
-- 写日志文件的缓存区大小
show variables like 'innodb_log_buffer_size';
-- 日志文件的大小
show variables like 'innodb_log_file_size';
show variables like 'innodb_log_files_in_group';
-- 读缓存区大小，对表进行顺序扫描的请求将分配到一个读入缓冲区
show variables like 'read_buffer_size';
-- 随机读缓冲区大小
show variables like 'read_rnd_buffer_size';
-- 为每张表分配一个新的文件（最佳实践）
show variables like 'innodb_file_per_table';
```

> [MySQL 5.7 Reference Manual](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/index.html)