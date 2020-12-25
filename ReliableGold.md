
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





# SpringBoot启动流程概况

## 创建SpringApplication实例
1. 设置resourceLoader，当前为null。
2. 对primarySources参数进行不能为null的断言，primarySources为包含main函数的类。
3. 创建一个LinkedHashSet对象，其中包含primarySources，然后把对象赋值给SpringApplication的成员变量primarySources。
4. 判断web application的类型，包含SERVLET、REACTIVE、NONE
5. 从spring.factories文件中获取org.springframework.context.ApplicationContextInitializer接口的实现类，并实例化保存在一个ArrayList中（共11个），赋值给SpringApplication的成员变量initializers。
6. 同样从spring.factories文件中获取org.springframework.context.ApplicationListener接口的实现类，并实例化保存在一个ArrayList中（共7个），赋值给SpringApplication的成员变量listeners。
7. 获取main方法所在的类的Class对象，赋值给mainApplicationClass成员变量。

## 执行run方法
1. 设置启动时间
2. 定义应用上下文变量，当前为null
3. 定义异常报告集合变量，new ArrayList<>();
4. 设置java.awt.headless
5. 创建监听器对象SpringApplicationRunListener，从配置文件中读取到EventPublishingRunListener，并实例化。在实例化的过程中，创建了SimpleApplicationEventMulticaster对象（事件发生器），然后把之前的11个监听器设置到对象中。
使用事件发生器对监听器进行启动，找出监听了ApplicationStartingEvent事件的监听器，并进行启动。
6. 装配命令行参数
7. 准备应用程序运行的环境
`ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);`
8. 设置系统属性，保证某些bean不会添加到准备的环境中（忽略某些bean）
9. 打印banner
10. 创建应用程序上下文（org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext）
11. 给异常报告集合变量赋值
12. 准备上下文对象
13. 刷新上下文环境（跳转到Spring中的refresh方法）
14. afterRefresh（方便扩展）
15. 计时结束，并打印启动耗时
16. listeners.started(context);
17. listeners.running(context);

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





# IDEA
- 新项目上传到SVN，VCS -> Import Into Version Control -> Share Project(Subversion)




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
- Windows上计算文件的md5值：certutil -hashfile .\jdk-8u261-windows-x64.exe md5


## Linux

### Kubernetes
- kubectl get pods | grep commerce
- kubectl exec -it test-e-commerce-56cfdf7f58-qqt5p /bin/bash
- kubectl logs -f test-e-commerce-56cfdf7f58-qqt5p



# MacOS
- 查看CPU缓存行大小：sysctl -a | grep "cacheline"





# 数据结构与算法

## 页面置换算法
### LRU：最近最久未使用。最长时间未被使用的缓存页。
### LFU：最近最少使用。一定时间内被访问次数最少的缓存页。