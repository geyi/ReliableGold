
# Spring

## Annotation

### AliasoFor
- 作用之一：若自定义注解继承了另一个注解，要想让调用方能够设置继承过来的属性值，就必须在自定义注解中重新定义一个属性，同时声明该属性是父注解某个属性的别名。
- 作用之二：互为别名

### ImportBeanDefinitionRegistrar动态注册bean
ImportBeanDefinitionRegistrar类只能通过其他类@Import的方式来加载，通常是启动类或配置类。  
使用@Import，如果括号中的类是ImportBeanDefinitionRegistrar的实现类，则会调用接口方法，将其中要注册的类注册成bean。实现该接口的类拥有注册bean的能力。





# 架构

## 理论知识

### 单体应用程序
- 修改了一小部分功能却要求整个应用重新构建和部署。
- 随着时间推移很难再保持一个好的模块化结构，使得一个模块的变更很难不影响到其它模块。
- 扩展需要进行整个应用程序的扩展，而不能进行部分扩展。





# JVM
- -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/kuaidi100/runnable_jar/order-sync/
- -XX:CMSMaxAbortablePrecleanTime=5000 ，默认值5s，代表该阶段最大的持续时间
- -XX:CMSScheduleRemarkEdenPenetration=50 ，默认值50%，代表Eden区使用比例超过50%就结束该阶段进入remark





# SpringBoot启动流程概况

## 创建SpringApplication实例
1. 设置resourceLoader，当前为null。
2. 对primarySources参数进行不能为null的断言，primarySources为包含main函数的类。
3. 创建一个LinkedHashSet对象，其中包含primarySources，然后把对象赋值给SpringApplication的成员变量primarySources。
4. 判断web application的类型，包含SERVLET、REACTIVE、NONE（可查看WebApplicationType枚举类了解详情）
5. 从spring.factories文件中获取org.springframework.context.ApplicationContextInitializer接口的实现类，并实例化保存在一个ArrayList中（共11个），赋值给SpringApplication的成员变量initializers。
6. 同样从spring.factories文件中获取org.springframework.context.ApplicationListener接口的实现类，并实例化保存在一个ArrayList中（共7个），赋值给SpringApplication的成员变量listeners。
7. 获取main方法所在的类的Class对象，赋值给mainApplicationClass成员变量。

## 执行run方法
1. 设置启动时间
2. 定义应用上下文变量，当前为null
3. 定义异常报告集合变量，new ArrayList<>();
4. 设置java.awt.headless（Headless模式是系统的一种配置模式。在系统可能缺少显示设备、键盘或鼠标这些外设的情况下可以使用该模式。）
5. 创建监听器对象SpringApplicationRunListener，从配置文件中读取到EventPublishingRunListener，并实例化。在实例化的过程中，创建了SimpleApplicationEventMulticaster对象（事件发生器），然后把之前的7个监听器设置到对象中。
6. 使用事件发生器对监听器进行启动，找出监听了ApplicationStartingEvent事件的监听器，并进行启动。
7. 装配命令行参数
8. 准备应用程序运行的环境
`ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);`
9. 设置系统属性，保证某些bean不会添加到准备的环境中（忽略某些bean）
10. 打印banner
11. 创建应用程序上下文（org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext）
12. 给异常报告集合变量赋值
13. 准备上下文对象
14. 刷新上下文环境（跳转到Spring中的refresh方法）
15. afterRefresh（方便扩展）
16. 计时结束，并打印启动耗时
17. listeners.started(context);
18. listeners.running(context);

## 重要的方法
1. getSpringFactoriesInstances(Class<T> type)
2. listeners.starting();

multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType)
匹配不同类型的事件，过滤掉不符合条件的监听器，返回符合条件的，循环执行具体监听器中自己的逻辑。
```
for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
	if (executor != null) {
		executor.execute(() -> invokeListener(listener, event));
	}
	else {
		invokeListener(listener, event);
	}
}
```
同样的，在springboot启动的不同阶段，会发生不同监听事件，包含的各种事件在SpringApplicationRunListener接口中。




# Oracle

## Oracle分页
select * from(select a.*, ROWNUM rn from(sql) a where ROWNUM <= (firstIndex + pageSize)) where rn > firstIndex

## AWR报告
`@?/rdbms/admin/awrrpt.sql`
https://www.cnblogs.com/yaoyangding/p/14206123.html

## 设置闪回区大小：
ALTER SYSTEM SET db_recovery_file_dest_size=200G;

## 查看归档日志应用情况：
SELECT DEST_ID, SEQUENCE#, APPLIED FROM V$ARCHIVED_LOG;

## 表空间未同步创建，导致同步断开，修复步骤：
1. alter database create datafile '/oracle/ora112/dbs/UNNAMED00028' as '/oracle/oradata/metadb/elec/tbs_elec14.dbf';
2. alter system set standby_file_management=AUTO;
3. alter database recover managed standby database using current loafile disconnect from session;

## 查看归档空间大小
```
SELECT ROUND(SUM(BLOCKS*BLOCK_SIZE)/1024/1024, 2) AS "Archived Space (MB)" FROM V$ARCHIVED_LOG;
```

## 删除归档日志
```
rman target /
list archivelog all;
delete noprompt force archivelog all completed before 'sysdate-7';
```

# MongoDB
- 大于等于：`db.appads.find({"isDelete":0,"endtime":{"$gte":"2020-08-11"}}).pretty()`
- 某字段存在：`db.appads.find({"privilegeState":{"$exists":true}}).pretty()`
- 增加字段：`db.appModule.update({},{$set:{label:""}},{multi:1})`，multi：默认是false，只更新找到的第一条记录。如果为true，把按条件查询出来的记录全部更新
- 排序：`db.pushbind2.find({"third_type":"VIVO"}).sort({createtime:-1})`
- 更新字段：`db.appads.update({"_id":"0bdd4668-f989-4c37-8bef-eb301fbed57c"},{$set:{"pos":"APP_MODULE_2"}});`
- 更新所有记录：`db.appModule.updateMany({"code":"MODULE_2"},{$set:{"logo":"https://cdn.kuaidi100.com/images/open/appItems/second-module.png?v=3"}});`
- 查看集合状态：`db.pushbind2.stats()`
- 导出集合数据：`./mongoexport -u kuaidi -p kingdee -d admin -c pushbind2 -o ./pushbind2.json --type json`
- 导入集合数据：`./mongoimport -u kuaidi -p kingdee -d admin -c adsidqslist --file ./adsid.json`
- 查询指定的列：`db.adsconfig.find({},{_id:1})`
- 聚合函数：`db.pushBizStat.aggregate([{$group:{_id:"$pushDate", sendTotalCount:{$sum:"$sendCount"}, receiveTotalCount:{$sum:"$receiveCount"}, clickTotalCount:{$sum:"$clickCount"}}}])`
- 创建root角色的用户：`db.createUser({user:"root", pwd:"root",roles:[ {role:"root", db:"admin" } ]})`
- 查看正在执行的操作：`db.currentOp()`
- 杀死正在执行的操作：`db.killOp(opid)`
- 创建索引：`db.poster.createIndex({"show_time":-1})`
- 为null或者不存在：`db.test.find({"test":null});`
- 不为null并且存在：`db.test.find({"test":{"$ne":null, $exists:true}});`
- 存在：`db.test.find({"test":{$exists:true}});`
- 不存在：`db.test.find({"test":{$exists:false}});`
- 删除字段：`db.ad_switch_config.updateMany({"switch": {"$exists": true}}, {"$unset": {"switch":""}}, {}, {multi: true});`
- 聚合查询：`db.pushBizStat2.aggregate([{$match:{"pushDate":{$gte:"2021-06-18"},"sendCount":{$ne:0},"platform":"WECHAT","msgType":{$nin:[1,16]}}},{$group:{_id:{pushDate:"$pushDate",channel:"$channel",templateCode:"$templateCode"}, sendTotalCount:{$sum:"$sendCount"}, receiveTotalCount:{$sum:"$receiveCount"}, clickTotalCount:{$sum:"$clickCount"}}}])`



# IDEA
- 新项目上传到SVN，VCS -> Import Into Version Control -> Share Project(Subversion)
- 自动导入依赖类的设置：Editor -> Code Style -> Java -> Imports




# 其他
- SVN帐号缓存：cd ~/.subversion/auth




# Eureka

## 启动流程
三个eureka服务：01 02 03
1. 01先启动，01会先从02拉取配置，再将自己注册到02。但是此时02还没启动。（问题：为什么没有去03拉取和注册？01跟02关联）
2. 同样，02启动时，也无法从03拉取和注册。但此时02接收了01的注册（01不停的向02注册）。此时02包含01。
3. 03启动时，向01注册。此时01包含03。
4. 01接收了03的注册后，会把03同步到02。此时02包含01，03。
5. 03为什么是去02拉取？
6. 03启动，接收02的注册。此时03包含01，02，03

eureka client
- 启动：
1. 读取与EurekaServer交互的配置信息封装成EurekaClientConfig
2. 读取自身配置信息封装成EurekaInstanceConfig
3. 从EurekaServer拉取注册列表，缓存到本地
4. 将自己注册到EurekaServer
5. 初始化3个定时器：发送心跳，缓存刷新，按需注册（客户端配置发生改变时，重新注册）

- 执行：执行以上3个定时任务

- 销毁：从EurekaServer销毁自身。一般情况下，应用服务在关闭的时候，EurekaClient会主动向EurekaServer注销自身在注册列表中的信息。





# Command

## Windows
- Windows上计算文件的md5值：`certutil -hashfile .\jdk-8u261-windows-x64.exe md5`
- 查询占用端口的进程id：`netstat -nao | findstr "9001"`
- 杀死指定的进程id：`taskkill /f /pid 26988`


## Linux

- 查看进程占用内存大小：`ps aux | grep demo-0.0.1-SNAPSHOT.jar | grep -v grep | awk '{print $11 "\t" $6/1024"MB" }'`
- Linux TCP连接IP排名：`netstat -natp | awk '{print $5}' | grep -v 'Address' | awk -F: '{print $1}' | sort | uniq -c | sort -nr`

### vi
批量添加注释：:起始行号,结束行号s/^/注释符/g
批量删除注释：:起始行号,结束行号s/^注释符//g

### Kubernetes
- kubectl get pods | grep commerce
- kubectl exec -it test-e-commerce-56cfdf7f58-qqt5p /bin/bash
- kubectl logs -f test-e-commerce-56cfdf7f58-qqt5p
- kubectl top pod test-advertisement-85864cc58c-8sshf
- sudo kubectl edit deploy market-elec
- kubectl apply -f shipper.ingress.yml
- kubectl logs -f nginx-ingress-controller-cf766b499-7xmks -n ingress-nginx --tail=10000 > /tmp/ingress.shipper.log
- kubectl delete pod --force --grace-period=0
- kubectl get pods --all-namespaces --field-selector=status.phase=Failed | grep Evicted | grep market-elec-5968d6d645 | awk '{print $2 " -n " $1}' | xargs -r -n 3 kubectl delete pod

> https://www.cnblogs.com/wuxinchun/p/15218227.html



## MacOS
- 查看CPU缓存行大小：sysctl -a | grep "cacheline"





# 数据结构与算法

## 页面置换算法
### LRU：最近最久未使用。最长时间未被使用的缓存页。
### LFU：最近最少使用。一定时间内被访问次数最少的缓存页。


# Redis
获取客户端缓存区限制：`CONFIG GET client-output-buffer-limit`
设置客户端缓存区限制：`CONFIG SET "client-output-buffer-limit" "normal 0 0 0 slave 1073741824 268435456 60 pubsub 33554432 8388608 60"`
连接redis的另一种方式：`nc localhost 6379`

# Kafka
`docker run -d --rm -p 9000:9000 -e KAFKA_BROKERCONNECT=192.168.249.101:9090,192.168.249.102:9090 -e JVM_OPTS="-Xms64M -Xmx64M" -e SERVER_SERVLET_CONTEXTPATH="/" obsidiandynamics/kafdrop:latest`
`docker run -d --rm -p 9001:9000 -e KAFKA_BROKERCONNECT=192.168.248.60:9090,192.168.248.61:9090,192.168.248.62:9090 -e JVM_OPTS="-Xms64M -Xmx64M" -e SERVER_SERVLET_CONTEXTPATH="/" obsidiandynamics/kafdrop:latest`


# SVN
以data-report项目为例来演示后端代码分支管理的改造：
1. 从svn服务器上检出data-report项目：`svn checkout svn://192.168.249.200/kuaidi100/kuaidi100V5/data-report`。其中包含了三个文件夹，分别是branches、tags、trunk。
2. 创建patch文件夹：`svn mkdir trunk\patch`
3. 在tags文件夹下创建以下三个文件夹：tests、releases、base。
   ```
   svn mkdir tags\tests
   svn mkdir tags\releases
   svn mkdir tags\base
   ```
4. 创建基线分支：`svn copy trunk\ branches\1.0.0`
5. 创建基线tag：`svn copy trunk\ tags\base\base_1_0_0`
6. 创建测试tag：`svn copy trunk\ tags\tests\test_1_0_1_0`或`svn copy svn://192.168.249.200/kuaidi100/kuaidi100V5/personal/advertisement/trunk tags\tests\test_1_0_1_0`
7. 创建发布tag：`svn copy tags\tests\test_1_0_1_1 tags\releases\release_1_0_1`