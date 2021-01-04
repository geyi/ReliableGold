# 查看sql语句的执行时间
```
set profiling=1;
select * from test;
show profiles;
show profile;
-- 查看query id为2的sql的执行时间
show profile for query 2;
```

# performance_schema

# show processlist;