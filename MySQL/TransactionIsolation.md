InnoDB使用不同的锁策略(Locking Strategy)来实现不同的隔离级别。

# 读未提交（Read Uncommitted）
如果事务B能够读取到未提交事务A操作的记录。  
此时，可能读取到不一致的数据，即“脏读”。这是并发最高，一致性最差的隔离级别。  
这种事务隔离级别下，select语句不加锁。

# 读已提交（Read Committed）
一个事务只能读取到另一个事务已提交的数据（避免了脏读）  
此时，一个事务内相同的查询，得到了不同的结果，即“不可重复读”。
 
这是互联网最常用的隔离级别，RC下：  
普通读是快照读。  
加锁的select, update, delete等语句，除了在外键约束检查(foreign-key constraint checking)以及重复键检查(duplicate-key checking)时会封锁区间，其他时刻都只使用记录锁；  
**事务在每次read操作时，都会建立read view**

# 可重复读（Repeatable Read）
一个事务通过一条select语句只能看到相同的数据，即使另一个事务已提交数据。一个例外是，一个事务可以读取在同一事务中更改的数据。
 
可重复读是InnoDB默认的隔离级别，在RR下：  
普通的select使用快照读，这是一种不加锁的一致性读(Consistent Nonlocking Read)，底层使用MVCC来实现。  
加锁的select(select ... in share mode / select ... for update)，update, delete等语句，它们的锁依赖于是否在唯一索引(unique index)上使用了唯一的查询条件(unique search condition)，或者范围查询条件(range-type search condition)：
* 在唯一索引上使用唯一的查询条件，会使用记录锁(record lock)，而不会封锁记录之间的间隔，即不会使用间隙锁(gap lock)与临键锁(next-key lock)
* 范围查询条件，会使用间隙锁与临键锁，锁住索引记录之间的范围，避免范围间插入记录，以避免产生幻影行记录，以及避免不可重复的读

**事务在第一个read操作时，会建立read view**

# 总结
可以看到，并发的事务可能导致其它事务出现：
- 脏读
- 不可重复读
- 幻读

不同事务的隔离级别，实际上是并发性与一致性的一个权衡与折中。
1. 读未提交：select不加锁，可能出现读脏；
2. 读提交(RC)：普通select快照读，锁select /update /delete 会使用记录锁，可能出现不可重复读；
3. 可重复读(RR)：普通select快照读，锁select /update /delete 根据查询条件情况，会选择记录锁，或者间隙锁/临键锁，以防止读取到幻影记录；
4. 串行化：select隐式转化为select ... in share mode，会被update与delete互斥；


