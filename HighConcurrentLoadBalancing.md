```shell
[root@192 ~]# cd /proc/$$/fd
[root@192 fd]# ll
总用量 0
lrwx------. 1 root root 64 4月  11 16:46 0 -> /dev/pts/0
lrwx------. 1 root root 64 4月  11 16:46 1 -> /dev/pts/0
lrwx------. 1 root root 64 4月  11 16:46 2 -> /dev/pts/0
lrwx------. 1 root root 64 4月  11 16:46 255 -> /dev/pts/0
lr-x------. 1 root root 64 4月  11 16:46 3 -> /var/lib/sss/mc/passwd
lrwx------. 1 root root 64 4月  11 16:46 4 -> 'socket:[29610]'
[root@192 fd]# exec 8<> /dev/tcp/www.baidu.com/80
[root@192 fd]# ll
总用量 0
lrwx------. 1 root root 64 4月  11 16:46 0 -> /dev/pts/0
lrwx------. 1 root root 64 4月  11 16:46 1 -> /dev/pts/0
lrwx------. 1 root root 64 4月  11 16:46 2 -> /dev/pts/0
lrwx------. 1 root root 64 4月  11 16:46 255 -> /dev/pts/0
lr-x------. 1 root root 64 4月  11 16:46 3 -> /var/lib/sss/mc/passwd
lrwx------. 1 root root 64 4月  11 16:46 4 -> 'socket:[29610]'
lrwx------. 1 root root 64 4月  11 16:46 8 -> 'socket:[29624]'
[root@192 fd]# echo -e 'GET / HTTP/1.0\n' >& 8
[root@192 fd]# cat <& 8
HTTP/1.0 200 OK
Accept-Ranges: bytes
Cache-Control: no-cache
Content-Length: 14615
Content-Type: text/html
Date: Sun, 11 Apr 2021 08:47:02 GMT
P3p: CP=" OTI DSP COR IVA OUR IND COM "
P3p: CP=" OTI DSP COR IVA OUR IND COM "
Pragma: no-cache
Server: BWS/1.1
Set-Cookie: BAIDUID=95E0E023FF721D2A7F839335514BBEAA:FG=1; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com
Set-Cookie: BIDUPSID=95E0E023FF721D2A7F839335514BBEAA; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com
Set-Cookie: PSTM=1618130822; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com
Set-Cookie: BAIDUID=95E0E023FF721D2A60BAF178D3F57461:FG=1; max-age=31536000; expires=Mon, 11-Apr-22 08:47:02 GMT; domain=.baidu.com; path=/; version=1; comment=bd
Traceid: 161813082204562252908104011806781447401
Vary: Accept-Encoding
X-Ua-Compatible: IE=Edge,chrome=1

<!DOCTYPE html><!--STATUS OK-->
<html>
<head>
        <meta http-equiv="content-type" content="text/html;charset=utf-8">
        <meta http-equiv="X-UA-Compatible" content="IE=Edge">
        <link rel="dns-prefetch" href="//s1.bdstatic.com"/>
        <link rel="dns-prefetch" href="//t1.baidu.com"/>
        <link rel="dns-prefetch" href="//t2.baidu.com"/>
        <link rel="dns-prefetch" href="//t3.baidu.com"/>
        <link rel="dns-prefetch" href="//t10.baidu.com"/>
        <link rel="dns-prefetch" href="//t11.baidu.com"/>
        <link rel="dns-prefetch" href="//t12.baidu.com"/>
        <link rel="dns-prefetch" href="//b1.bdstatic.com"/>
        <title>百度一下，你就知道</title>
省略......
```

第一步建立连接  
第二步发送数据（http协议：标准和规范）

# 传输控制层（TCP）
面向连接的：TCP会生成并检查每个包的校验和，保证数据按顺序正确的到达。（**一发送一确认**）

## 三次握手
1. SYN
2. SYN-ACK
3. ACK

前面两次客户端已确定自己的output和intput都是可用的  
服务端则需要继续等待客户端响应是否接收到自己发送的确认消息（即客户端向服务端发送ACK）  
三趟走完后，双方才会自己的内存中建立相应的资源。

## 四次分手
为什么要有分手？因为资源是有限的（端口）  
为什么是四次？因为TCP是面向连接的可靠的传输协议


# 网路层（IP）
```shell
[root@192 network-scripts]# cat ifcfg-ens33
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="31e486a0-0403-446d-9b2f-4ee51ad48c0c"
DEVICE="ens33"
ONBOOT="yes"
IPADDR=192.168.1.12
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
[root@192 network-scripts]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    100    0        0 ens33
192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 ens33
[root@192 network-scripts]# ping www.baidu.com
PING www.a.shifen.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=1 ttl=56 time=6.84 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=2 ttl=56 time=6.70 ms
^C
--- www.a.shifen.com ping statistics ---
3 packets transmitted, 2 received, 33.3333% packet loss, time 7ms
rtt min/avg/max/mdev = 6.698/6.767/6.836/0.069 ms
```
当前路由表包含两条路由信息，这两条路由信息都是根据ifcfg-ens33配置文件生成的。  
通过ping www.baidu.com知道目标ip地址是14.215.177.39。与子网掩码0.0.0.0做与运算得到0.0.0.0，所以得到下一跳是192.168.1.1

数据是如何发送给192.168.1.1的？

# 链路层

arp解释IP地址与网卡硬件地址的映射（`arp -an`）

链路层封装了MAC地址

# 总结
TCP/IP协议是基于吓一跳机制，IP地址是端点间，MAC地址是节点间。


网卡上线后会发送一个arp数据包，包含内容大致如下：
```
目标MAC地址：FF:FF:FF:FF:FF:FF
目标IP地址：192.168.1.1
源MAC地址：本机的MAC地址，如00:0c:29:29:3a:41
源IP地址：192.168.1.12
```
交换机首先收到这个数据包，如果发现目标MAC地址是FF:FF:FF:FF:FF:FF，那么交换机会把这个包广播出去。收到这个数据包的计算机会使用本地IP地址对比数据包中的目标IP地址，如果不相符，则丢弃这个数据包。否则，则会响应这个数据包。响应数据包内容大致如下：
```
目标MAC地址：00:0c:29:29:3a:41
目标IP地址：192.168.1.12
源MAC地址：8c:14:b4:48:bf:50
源IP地址：192.168.1.1
```
当1.12计算机接收到上述数据包时，就能得到一条IP地址与MAC地址的映射。

> 交换机（二层交换机）具有“学习”功能，它能够记录端口与MAC地址的映射关系。

增加一块虚拟网卡：`inconfig eth0:3 192.168.88.88/24`  
增加一条路由条目：`reoute add -host 192.168.88.88 gw 192.168.1.12`

Tomcat为什么慢
1. 在ISO参考模型中，首先Tomcat位于最末端的应用层
2. 其次数据的交换存在用户态与内核态的切换
3. JVM

负载均衡服务器（LVS）
1. 数据包级别的转发
2. 不发生三次握手
所以此时要求后端服务器是镜像的，因为当前的负载均衡服务器无法判断数据包应该丢给哪个服务器。

Nginx想要知道请求的URI一定要有一次三次握手的过程，所以Nginx的并发是有上限的（官方5W并发）。

所以整体的架构，LVS -> Nginx -> Tomcat

# NAT（Network Address Translation，网络地址转换）
基于3层，带宽成为瓶颈，消耗算力

端口多路复用（Port address Translation）是指改变外出数据包的源端口并进行端口转换，即端口地址转换。采用端口多路复用方式。内部网络的所有主机均可共享一个合法外部IP地址实现对Internet的访问，从而可以最大限度地节约IP地址资源。同时，又可隐藏网络内部的所有主机，有效避免来自internet的攻击。因此，目前网络中应用最多的就是端口多路复用方式。

# DR
基于2层，MAC地址欺骗，速度快成本低