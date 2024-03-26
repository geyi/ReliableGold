# OSI七层参考模型
***应用层***：为用户提供直接的应用程序服务，包括电子邮件、文件传输、网络浏览器等。该层使用特定的协议和标准来实现应用程序之间的通信。  
***表示层***：负责数据格式的转换和加密解密，确保不同系统之间的数据可以正确解释和理解。它处理数据的压缩、加密、编码和解码等功能。  
***会话层***：管理不同应用程序之间的会话（Session）。它建立、维护和结束会话，提供数据交换的控制和同步机制。  
***传输层***：提供端到端的通信服务，确保数据传输的可靠性和完整性。该层通常使用TCP协议（面向连接）或UDP协议（无连接）来处理数据分段、重传、流量控制等。（如何建立连接，如何传输，应该是成功的还是失败）  
***网络层***：负责在整个网络中寻址和路由数据包。该层使用IP地址来标识网络上的不同节点，并通过路由器进行数据包的选择和转发。  
***数据链路层***：在直接相连的两个节点之间提供可靠的数据传输。它将原始比特流转换为帧（Frame）并进行错误检测和纠正，通过物理地址（MAC地址）识别设备。  
***物理层***：该层负责传输原始比特流，处理物理介质、电压等物理特性。它定义了电缆、光纤、网卡等硬件设备的规范。

# 应用层（HTTP）
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
[root@192 fd]# exec 8<&-
```

第一步建立连接  
第二步发送数据（http协议：标准和规范）

> 在 Linux 的 `/dev` 目录下，通常包含了许多设备文件来表示硬件设备和其他系统资源。但是，`/dev/tcp` 并不是一个实际存在的设备文件或目录。它是一种 Bash shell 的特性，用于使用 TCP 协议进行网络连接。
>
> `exec 8<> /dev/tcp/www.baidu.com/80` 表示创建了一个名为 8 的文件描述符，并将其绑定到指定的路径 /dev/tcp/www.baidu.com/80 上。后续可以使用这个文件描述符向服务器发送请求或读取数据。

# 传输控制层（TCP）
面向连接的：TCP会生成并检查每个包的校验和，保证数据按顺序正确的到达。（**一发送一确认**）

## 三次握手
通信双方将通过三次TCP报文实现对以下信息的了解：
1. 对方报文发送的开始序号
2. 对方发送数据的缓冲区大小
3. 能被接收的最大报文段长度MSS
4. 被支持的TCP选项

```
t  c            s
|  |----SYN---->|
|  |            |
|  |<--SYN-ACK--|
|  |            |
|  |----ACK---->|
|  |            |
V
```
前面两次客户端已确定自己的output和intput都是可用的  
服务端则需要继续等待客户端响应是否接收到自己发送的确认消息（即客户端向服务端发送ACK）  
三趟走完后，服务端才会在自己的内存中建立相应的资源。

## 四次分手
为什么要有分手？因为资源是有限的（端口）  
为什么是四次？因为TCP是面向连接的可靠的传输协议

三次握手 -> 数据传输 -> 四次分手，成为一个最小的粒度，不能被分割。

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
当前IP路由表包含两条路由信息。  
通过`ping www.baidu.com`知道目标ip地址是14.215.177.39。与子网掩码0.0.0.0做与运算得到0.0.0.0，所以得到下一跳的地址是192.168.1.1。

## traceroute
一种网络诊断工具，用于跟踪数据包从源IP地址到目标IP地址之间的路由路径。它可以显示数据包在每个路由器上的经过时间，并列出每个路由器的IP地址和主机名。
```
[root@node03 ~]# traceroute 110.242.68.66
traceroute to 110.242.68.66 (110.242.68.66), 30 hops max, 60 byte packets
 1  * * *
 2  10.240.0.253 (10.240.0.253)  7.294 ms  4.432 ms  4.406 ms
 3  10.240.0.246 (10.240.0.246)  2.759 ms  2.732 ms  2.668 ms
 4  * * *
 5  * * *
 6  * * *
 7  * * *
...
```
记录按序列号从1开始，每个记录就是一跳，每跳表示一个网关。每行有三个时间，单位是ms，它们表示向每个网关发送三个探测数据包后，得到响应的时间。有一些行以星号表示的，可能是防火墙封掉了ICMP的返回信息。

## Gateway 0.0.0.0的含义
```
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 ens33
```
如上IP路由信息代表ens33网卡在192.168.1.0局域网中，而同一局域网中两台主机之间的通信不需要下一跳转发（网与网之间才需要下一跳的转发）。同一局域网不需要网关，不需要走路由，直接通过交换机转发。所以Gateway 0.0.0.0表示当前设备在Destination中有一个具体的IP。

> 同一网络中IP地址不能重复

# 链路层
网络层根据IP路由表找到下一跳的地址是192.168.1.1，但是实际要访问的地址是14.215.177.39（百度），那么数据是如何发送给192.168.1.1的？

## ARP协议
arp解析IP地址与网卡硬件地址的映射（`arp -an`）。arp属于网络层协议？

**链路层封装了MAC地址，每次下一跳MAC地址都会发生变化**

# 总结
TCP/IP协议是基于下一跳机制，IP地址是端到端（两台主机），MAC地址是点到点。


当一台主机需要找出同网络中的另一台主机的物理地址时，它就可以发送一个ARP请求报文，包含内容大致如下：
```
目标MAC地址：FF:FF:FF:FF:FF:FF
目标IP地址：192.168.1.1
源MAC地址：本机的MAC地址，如00:0c:29:29:3a:41
源IP地址：192.168.1.12
```
交换机首先收到这个数据包，如果发现目标MAC地址是FF:FF:FF:FF:FF:FF，那么交换机会把这个包广播出去。收到这个数据包的计算机会使用本地IP地址对比数据包中的目标IP地址，如果不相符，则丢弃这个数据包。否则，则会响应这个数据包，响应数据包内容大致如下：
```
目标MAC地址：00:0c:29:29:3a:41
目标IP地址：192.168.1.12
源MAC地址：8c:14:b4:48:bf:50
源IP地址：192.168.1.1
```
当IP地址为192.168.1.12的计算机接收到上述数据包时，就能得到一条IP地址与MAC地址的映射。

> 交换机（二层交换机）具有“学习”功能，它能够记录物理端口与MAC地址的映射关系。  
> 交换机衔接同一网络（二层），路由器衔接不同网络（三层）

> 增加一块虚拟网卡：`ifconfig eth0:3 192.168.88.88/24`  
> 增加一条路由条目：`route add -host 192.168.88.88 gw 192.168.1.12`（net需要给出子网掩码）  
> 删除：`route del -host 192.168.88.88`

# 负载均衡服务器（LVS）
1. 数据包级别的转发
2. 不发生三次握手

所以此时要求后端服务器是镜像的，因为当前的负载均衡服务器无法判断数据包应该丢给哪个服务器。

Tomcat为什么慢
1. 在OSI参考模型中，首先Tomcat位于最末端的应用层
2. 其次数据的交换存在用户态与内核态的切换
3. 运行在JVM之上，使用Java开发（但不是说使用Java就很慢）

Nginx想要知道请求的URI一定要有一次三次握手的过程，所以Nginx的并发是有上限的（官方5W并发）。

所以整体的架构，LVS（流量） -> Nginx（握手） -> Tomcat

## NAT（Network Address Translation，网络地址转换）
IP和端口都会转换
- SNET：源地址转换
- DNET：目标地址转换

### 缺点
- 带宽成为瓶颈（上下行的流量非对称，往往下行会成为瓶颈），消耗算力（**基于3层来回都要修改IP地址**）。  
- 要求real server的GW要指向负载均衡服务器。（出去的流量子网掩码0.0.0.0，默认网关指向负载均衡服务器）

端口多路复用（Port address Translation）是指改变外出数据包的源端口并进行端口转换，即端口地址转换。采用端口多路复用方式，内部网络的所有主机均可共享一个合法外部IP地址实现对Internet的访问，从而可以最大限度地节约IP地址资源。同时，又可隐藏网络内部的所有主机，有效避免来自internet的攻击。因此，目前网络中应用最多的就是端口多路复用方式。

## DR（Direct Route，直接路由）
基于2层，MAC地址欺骗，成本低速度快。
负载均衡服务器和real server要在同一个局域网

隐藏VIP方法：对外隐藏，对内可见

kernel parameter：
目标MAC地址为全F，交换机触发广播

/proc/sys/net/ipv4/conf/*IF*/

arp_ignore：定义接收到ARP请求时的响应级别  
0：响应任意网卡上接收到的对本机IP地址的arp请求，而不管目标IP是否在接收的网卡上。  
1：只响应目的IP地址为接收网卡上的本地地址的arp请求。

arp_announce：定义将自己的IP地址向外通告时的通告级别  
0：允许使用任意网卡上的IP地址作为arp请求的源IP。（发送所有地址）  
1：尽量避免使用不属于该发送网卡子网的本地地址作为arp请求的源IP。（网卡上的所有地址）  
2：选择发送网卡上最合适的本地地址作为arp请求的源IP。（网卡上的指定地址）

### 负载均衡策略
#### 静态
- rr
- wrr
- dh：目标地址散列
- sh：源地址散列

#### 动态调度方法
- lc：最少连接（偷窥数据包）
- wlc：加权最少连接
- sed：最短期望延迟
- nq：never queue
- LBLC：基于本地的最少连接
- DH：目标地址散列
- LBLCR：基于本地的带复制功能的最少连接

### IPVS
`yum install ipvsadm -y`
管理集群服务
```
添加：-A -t|u|f service-address [-s scheduler]
-t：TCP协议的集群
-u：UDP协议的集群
service-address：IP:PORT
-f：FWM 防火墙标记
service-address：Mark Number
修改：-E
删除：-D -t|u|f service-address
ipvsadm -A -t 192.168.1.12:80 -s rr
```

管理集群服务中的RS
```
添加：-a -t|u|f service-address -r server-address [-g|i|m] [-w weight]
-t|u|f service-address：事先定义好的某集群服务
-r server-address: 某RS的地址，在NAT模型中，可使用IP:PORT实现端口映射
[-g|i|m]：LVS类型
-g：DR
-i：TUN
-m：NAT
[-w weight]：定义服务器权重
修改：-e
删除：-d -t|u|f service-address -r server-address
# ipvsadm -a -t 192.168.1.12:80 -r 192.168.1.100 -g
# ipvsadm -a -t 192.168.1.12:80 -r 192.168.1.101 -g
查看
-L
-n：数字格式显示主机地址和端口
--stats：统计数据
--rate：速率
--timeout：显示tcp，tcpfin和udp的会话超时时长
-c：显示当前的ipvs连接状况
删除所有集群服务
-C：清空ipvs规则
保存规则
-S
# ipvsadm -S > /path/to/somefile
从文件中载入规则
-R
# ipvsadm -R < /path/to/somefile
```

### DR模式的LVS搭建
node01 192.168.10.128  
node02 192.168.10.129  
node03 192.168.10.130

在Node01上新增一块网卡：`ifconfig ens33:0 192.168.10.100/24`

Node02、Node03  
1. 修改内核参数（重启后失效）
   ```shell
   echo 1 > /proc/sys/net/ipv4/conf/eth0/arp_ignore 
   echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore 
   echo 2 > /proc/sys/net/ipv4/conf/eth0/arp_announce 
   echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce 
   ```
   保证永久有效的方法是修改配置文件/etc/sysctl.conf
   ```shell
   net.ipv4.conf.ens33.arp_ignore=1
   net.ipv4.conf.all.arp_ignore=1
   net.ipv4.conf.ens33.arp_announce=2
   net.ipv4.conf.all.arp_announce=2
   ```
2. 设置隐藏的VIP：`ifconfig lo:0 192.168.42.100/32`
3. 安装httpd并启动：  
   1）`yum -y install httpd`  
   2）`service httpd start`

LVS服务配置（Node01）
```
yum install ipvsadm 
ipvsadm -A  -t 192.168.42.100:80  -s rr
ipvsadm -a  -t 192.168.42.100:80  -r  192.168.42.129 -g -w 1
ipvsadm -a  -t 192.168.42.100:80  -r  192.168.42.130 -g -w 1
ipvsadm -ln
```

### keepalived + LVS
1. Node01、Node04安装keepalived：`yum install keepalived ipvsadm -y`
2. 修改配置文件/etc/keepalived/keepalived.conf

## TUN 隧道技术
一个IP包“背着”一个IP包
```
-----------------------------
|            -------------- |
| DIP -> RIP | CIP -> VIP | |
|            -------------- |
-----------------------------
```