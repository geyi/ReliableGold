# 多级缓存架构

## 划分域名
按业务或者是功能划分域名，不同的域名被绑定到不同的接入机。

## CDN
- 静态缓存：图片，html，css，js
- 动态缓存：少量一致性要求低的数据，例如：电商系统的商家列表，商品分类列表，搜索结果页（搜索结果页可以使用定时任务更新）

## 接入层架构
1. LVS
2. 反向代理 Nginx
3. 应用层 Nginx 整合Lua脚本直接访问Redis
4. 应用层网关 Zuul SpringCloudGateway
5. 应用服务 Tomcat

> `request_rate_limiter.<userId>.token`中保存了令牌的数量，当数量为0时表示没有可用令牌，请求将会被拒绝。当key不存在时则表示该用户第一次发起请求。


# Redis + Lua
```shell
# 让redis服务器执行一段lua脚本
[root@server133 ~]# redis-cli eval "return 1 + 1" 0
(integer) 2
[root@server133 ~]# redis-cli eval "local msg='hello ' return msg..ARGV[1]" 1 v1 world
"hello world"
[root@server133 ~]# redis-cli eval "local msg='hello ' return msg..KEYS[1]" 1 v1 world
"hello v1"
# 让redis服务器执行lua脚本文件
[root@server133 ~]# vi test.lua
[root@server133 ~]# redis-cli --eval test.lua 
(integer) 0
```

将lua脚本加载到redis：`redis-cli script load "$(cat test.lua)"`

lua脚本会阻塞redis的工作线程，如果lua脚本执行时间过长，那么在脚本执行完之前，redis都无法处理其它脚本。


# OpenResty（Nginx + Lua）