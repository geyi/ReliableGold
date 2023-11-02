# 自适应哈希索引
1. 使用InnoDB引擎时，用户无法手动创建哈希索引。
2. InnoDB会自调优，如果判定建立自适应哈希索引，能够提升查询效率，InnoDB自己会建立相关的哈希索引。

在MySql运行过程中，如果InnoDB发现，有很多记录定位的寻路路径（search path）都很长，并且有很多SQL会命中相同的页，InnoDB会在自己的内存缓冲区里，开辟一块区域，建立自适应哈希索引，以加速查询。

哈希结构：key是索引键值，value是索引记录的页面位置。

当业务场景为下面几种情况，AHI往往是有效：
* 很多单行记录查询
* 所有记录内存能放得下

当业务有大量like或者join，AHI的维护反而可能成为负担，降低系统效率，此时可以手动关闭AHI功能。