# 基础规范
- 无特殊情况表存储引擎使用InnoDB
- 表字符集默认使用utf8，必要时使用utf8mb4
  1. 通用，无乱码风险，汉字3字节，英文1字节；
  2. utf8mb4是utf8的超集，有存储4字节符号时使用它，例如：表情符号。
- 禁止使用存储过程，视图，触发器，Event
  1. 对数据性能影响较大，能让站点层和服务层干的事情，就不要交到数据库层
  2. 调试，排错，迁移都比较困难，扩展性较差
- 禁止在数据库中存储大文件，例如：照片，可以将大文件存储在对象存储系统，数据库中存储路径
- 禁止在线上环境做数据库压力测试
- 测试，开发，线上数据库环境必须隔离

# 命名规范
- 库名，表名，列名必须用小写，采用下划线分隔，长度不要超过32个字符

# 表设计规范
- 单库表个数必须控制在2000个以内
- 单表分表个数必须控制在1024个以内
- 表必须有主键，推荐使用UNSIGNED整数为主键。删除无主键的表，如果是row模式的主从架构，从库会挂住
- 禁止使用外键，如果要保证完整性，应由应用程序实现
- 建议将大字段，访问频率低的字段拆分到单独的表中存储，分离冷热数据

# 列设计规范
- 根据业务区分使用tinyint、int、bigint，分别会占用1、4、8字节
- 根据业务区分使用char/varchar
  1. 字段长度固定，或者长度近似的业务场景，适合使用char，能减少碎片，查询性能高
  2. 字段长度相差较大，或者更新较少的业务场景，适合使用varchar，能够减少空间
- 根据业务区分使用datetime、timestamp。前者占用8个字节，后者占用4个字节，存储日期使用DATE，存储时间使用datetime
- 必须把字段定义为NOT NULL并设默认值
  1. NULL的列使用索引，索引统计，值都更加复杂，MySQL更难优化
  2. NULL需要更多的存储空间
  3. NULL只能采用IS NULL或者IS NOT NULL，而在=/!=/in/not in时有大坑
- 使用INT UNSIGNED存储IPv4，不要用char(15)
- 使用TINYINT来代替ENUM。ENUM增加新值要进行DDL操作

# 索引规范
- 唯一索引使用uniq_[字段名]来命名
- 非唯一索引使用idx_[字段名]来命名
- 单张表索引数量建议控制在5个以内
  1. 互联网高并发业务，太多索引会影响写性能
  2. 生成执行计划时，如果索引太多，会降低性能，并可能导致MySQL选择不到最优索引
  3. 异常复杂的查询需求，可以选择ES等更为适合的方式存储
- 组合索引字段数不建议超过5个
- 不建议在频繁更新的字段上建立索引
- 非必要不要进行JOIN查询，如果要进行JOIN查询，被JOIN的字段必须类型相同，并建立索引
- 理解组合索引最左前缀原则，避免重复建设索引，如果建立了(a,b,c)，相当于建立了(a), (a,b), (a,b,c)

# SQL规范
- 禁止使用select *，只获取必要字段
  1. select *会增加cpu/io/内存/带宽的消耗
  2. 指定字段能有效利用索引覆盖
  3. 指定字段查询，在表结构变更时，能保证对应用程序无影响
- insert必须指定字段，禁止使用insert into T values()
- 隐式类型转换会使索引失效，导致全表扫描
- 禁止在where条件列使用函数或者表达式
- 禁止负向查询以及%开头的模糊查询
- 禁止大表JOIN和子查询
- 同一个字段上的OR必须改写问IN，IN的值必须少于50个
- 应用程序必须捕获SQL异常