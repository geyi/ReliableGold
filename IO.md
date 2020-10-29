程序想获取数据时

VFS：虚拟文件系统

把VFS当做暴露给用户空间程序的统一接口，用来访问不同的硬件设备（VFS挂载了不同的硬件设备）

两个程序如果访问的是同一个文件，文件的**inode id**和**pagecache**是共享的

dirty：程序修改过的pagecache被标记为脏

flush：
- 每次修改都将数据刷到磁盘（强一致性）
- 每次修改都将数据写入操作系统的缓冲区，固定时间间隔后将缓冲区的数据刷到磁盘（折中）
- 固定时间间隔后将数据写入操作系统缓冲区，并刷到磁盘（最佳性能）


FD：文件描述符

seek指针：移动指针到指定的偏移位置

实操验证：

umount：卸载
mount：挂载

# Linux
Linux下一切皆文件，由此可以引导出不同的文件类型
- -：普通文件（可执行，图片，文本）
- d：目录
- l：链接
- b：块设备（硬盘）
- c：字符设备（键盘）
- s：socket
- p：pipeline
- [eventpoll]：

> 硬链接：多个引用指向同一个物理文件，inode相同。文件硬链接数与引用数相同。  
  软连接：类似于快捷方式，inode不同。文件硬链接数不会随着软连接的个数增加。

## 模拟一个文件系统（/mnt/mydisk）
1. 创建一个100MB的磁盘镜像文件
2. 使用losetup命令将磁盘镜像文件虚拟成块设备
3. 将块设备格式化为ext2格式
4. 挂载块设备
5. 查看设备挂载情况
6. 在模拟文件系统中进行操作
```
dd if=/dev/zero of=mydisk.img bs=1048576 count=100
losetup /dev/loop0 mydisk.img
mke2fs /dev/loop0
mount -t ext2 /dev/loop0 /mnt/mydisk/
df -h
```
> Linux losetup命令用于设置循环设备。循环设备可把文件虚拟成区块设备，以此来模拟整个文件系统，让用户得以将其视为硬盘驱动器，光驱或软驱等设备，并挂入当作目录来使用。
```
[root@192 mydisk]# whereis bash
bash: /usr/bin/bash /usr/share/man/man1/bash.1.gz /usr/share/info/bash.info.gz
[root@192 mydisk]# mkdir bin
[root@192 mydisk]# cp /usr/bin/bash bin/
[root@192 mydisk]# ll bin/
总用量 1198
-rwxr-xr-x. 1 root root 1219248 8月  16 12:18 bash
[root@192 mydisk]# cd bin/
[root@192 bin]# ldd bash
        linux-vdso.so.1 (0x00007fffca10d000)
        libtinfo.so.6 => /lib64/libtinfo.so.6 (0x00007effabee6000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007effabce2000)
        libc.so.6 => /lib64/libc.so.6 (0x00007effab920000)
        /lib64/ld-linux-x86-64.so.2 (0x00007effac431000)
[root@192 bin]# cd ..
[root@192 mydisk]# mkdir lib64
[root@192 mydisk]# ll
总用量 16
drwxr-xr-x. 2 root root  1024 8月  16 12:18 bin
drwxr-xr-x. 2 root root  1024 8月  16 12:19 lib64
drwx------. 2 root root 12288 8月  16 12:14 lost+found
[root@192 mydisk]# cp /lib64/{libtinfo.so.6,libdl.so.2,libc.so.6,ld-linux-x86-64.so.2} lib64/
[root@192 mydisk]# ll lib64/
总用量 4641
-rwxr-xr-x. 1 root root  302552 8月  16 13:58 ld-linux-x86-64.so.2
-rwxr-xr-x. 1 root root 4176104 8月  16 13:58 libc.so.6
-rwxr-xr-x. 1 root root   36824 8月  16 13:58 libdl.so.2
-rwxr-xr-x. 1 root root  208616 8月  16 13:58 libtinfo.so.6
[root@192 mydisk]# chroot ./
bash-4.4# ls  echo $$
1963
bash-4.4# ls
bash: ls: command not found
bash-4.4# vi
bash: vi: command not found
bash-4.4# cat
bash: cat: command not found
bash-4.4# echo "hello bash" > abc.txt
bash-4.4# exit
exit
[root@192 mydisk]# ll
总用量 18
-rw-r--r--. 1 root root    11 8月  16 14:00 abc.txt
drwxr-xr-x. 2 root root  1024 8月  16 12:18 bin
drwxr-xr-x. 2 root root  1024 8月  16 13:58 lib64
drwx------. 2 root root 12288 8月  16 12:14 lost+found
```

ldd：print shared library dependencies

lsof：用于列出一个进程打开了哪些文件。如：`lsof -op $$`

proc是系统内存的映射，这个目录的内容不在硬盘上而是在内存里。
- /proc/$$（$$表示当前bash的pid（$BASHPID））
- /proc/$$/fd（lsof -op $$）

## 输入/输出重定向
任何程序都有
- 0：标准输入
- 1：标准输出
- 2：标准错误输出

### 重定向
```
[root@192 ~]# read a
aaa
[root@192 ~]# echo $a
aaa
[root@192 ~]# read a 0< cat.out
[root@192 ~]# echo $a
总用量 6292
```

### 重定向标准输出和标准错误输出
```
[root@192 ~]# ls ./ /ooxx 1> ls1.out 2> ls2.out
[root@192 ~]# ll
总用量 6308
-rw-------. 1 root root      1533 8月  15 23:06 anaconda-ks.cfg
-rw-r--r--. 1 root root       196 8月  16 19:59 cat.out
-rw-r--r--. 1 root root        62 8月  16 20:04 ls1.out
-rw-r--r--. 1 root root        51 8月  16 20:04 ls2.out
-rw-r--r--. 1 root root       196 8月  16 19:57 ls.out
-rw-r--r--. 1 root root 104857600 8月  16 16:21 mydisk.img
[root@192 ~]# cat ls1.out
./:
anaconda-ks.cfg
cat.out
ls1.out
ls2.out
ls.out
mydisk.img
[root@192 ~]# cat ls2.out
ls: 无法访问'/ooxx': No such file or directory
```

### 标准输出和标准错误输出重定向到同一个文件
使用如下写法将标准输出和标准错误输出重定向到同一个文件时，标准错误输出会被覆盖
```
[root@192 ~]# ls ./ /ooxx 1> ls3.out 2>ls3.out
[root@192 ~]# ll
总用量 6312
-rw-------. 1 root root      1533 8月  15 23:06 anaconda-ks.cfg
-rw-r--r--. 1 root root       196 8月  16 19:59 cat.out
-rw-r--r--. 1 root root        62 8月  16 20:04 ls1.out
-rw-r--r--. 1 root root        51 8月  16 20:04 ls2.out
-rw-r--r--. 1 root root        70 8月  16 20:05 ls3.out
-rw-r--r--. 1 root root       196 8月  16 19:57 ls.out
-rw-r--r--. 1 root root 104857600 8月  16 16:21 mydisk.img
[root@192 ~]# cat ls3.out
./:
anaconda-ks.cfg
cat.out
ls1.out
ls2.out
ls3.out
ls.out
mydisk.img
```

当我们需要将标准输出和标准错误输出重定向到同一个文件时，`2> 1`的写法实际上是将标准错误输出重定向到了名称为1的文件中。所以**如果我们要将一个文件描述符重定向到另一个文件描述符时，重定向符号后应跟着一个&符号，如：`2>& 1`。**
```
[root@192 ~]# ls ./ /ooxx 2> 1 1> ls4.out
[root@192 ~]# ll
总用量 6320
-rw-r--r--. 1 root root        51 8月  16 20:07 1
-rw-------. 1 root root      1533 8月  15 23:06 anaconda-ks.cfg
-rw-r--r--. 1 root root       196 8月  16 19:59 cat.out
-rw-r--r--. 1 root root        62 8月  16 20:04 ls1.out
-rw-r--r--. 1 root root        51 8月  16 20:04 ls2.out
-rw-r--r--. 1 root root        70 8月  16 20:05 ls3.out
-rw-r--r--. 1 root root        80 8月  16 20:07 ls4.out
-rw-r--r--. 1 root root       196 8月  16 19:57 ls.out
-rw-r--r--. 1 root root 104857600 8月  16 16:21 mydisk.img
[root@192 ~]# cat 1
ls: 无法访问'/ooxx': No such file or directory
[root@192 ~]# cat ls4.out
./:
1
anaconda-ks.cfg
cat.out
ls1.out
ls2.out
ls3.out
ls4.out
ls.out
mydisk.img
```

这时的标准错误输出仍然被打印在屏幕上了
```
[root@192 ~]# ls ./ /ooxx 2>& 1 1> ls5.out
ls: 无法访问'/ooxx': No such file or directory
```

讲标准输出和标准错误输出到同一个文件的正确写法
```
[root@192 ~]# ls ./ /ooxx 1> ls6.out 2>& 1
[root@192 ~]# cat ls6.out
ls: 无法访问'/ooxx': No such file or directory
./:
1
anaconda-ks.cfg
cat.out
ls1.out
ls2.out
ls3.out
ls4.out
ls5.out
ls6.out
ls.out
mydisk.img
```

## 管道
head -8 test.txt | tail -1

### 基本特征
1. 前面的输出作为后面输入

> 父子进程：

> 变量：


### int 0x80 中断
0x80对应中断描述符表中的数值：0 1 2 ... 128 ... 255

# PageCache
page cache是内核维护的中间层。

```
# 当脏页所占内存空间大小达到设定的值时，内核的pdflush线程开始回写脏页
vm.dirty_background_bytes = 0
# 当脏页所占可用内存的百分比达到设定的值时（这里是10%），内核的pdflush线程开始回写脏页
# 增大该值会使更多的内存用于缓冲，可以提高系统的读写性能。当需要持续、恒定的写入场景时，应该减小该值
# dirty_background_ratio参数与dirty_background_bytes参数只能指定其中一个
vm.dirty_background_ratio = 10
#
vm.dirty_bytes = 0
vm.dirty_ratio = 30
vm.dirty_expire_centisecs = 3000
vm.dirty_writeback_centisecs = 500
vm.dirtytime_expire_seconds = 43200
```

在Java中进行IO操作时使用Buffer可以减少系统调用，提高性能

page cache是优化IO性能的，但却带来了数据丢失的问题




# 网络IO

## 命令
- 监听某端口的TCP情况：tcpdump -nn -i eth0 port 9090
- 监控用户空间进程和内核的交互，比如系统调用、信号传递、进程状态变更等：strace -ff -o out `cmd`
- route -n
- nc localhost 9090
- nc -l localhost 9090
- ulimit -SHn 655360

##  系统调用TCP
面向连接的，可靠的传输协议

三次握手 -> 内核开辟资源（握手的过程在内核完成，即便没有调用ServerSocket的accept方法）

### SOCKET
socket是一个四元组，即：客户端的IP地址、客户端的端口号 + 服务端的IP地址、服务端的端口号
> XIP_APORT + YIP_APROT : FD3
> 
> 服务端是否需要为client的连接分配一个随机端口号，答：不需要

MTU 数据包的大小
MSS 数据内容的大小

## 网络IO模型

同步：应用程序自己R/W
异步：内核完成R/W
阻塞
非阻塞

### BIO系统调用
1. socket 创建一个server socket 返回sfd（服务端的文件描述符）
2. bind sfd到一个地址 端口
3. listen sfd
4. accept sfd 返回接收的socket的cfd（客户端的文件描述符） **阻塞的** accept是一次系统调用
5. clone 创建子线程处理客户端的socket（因为阻塞所以使用创建线程的方式处理客户端链接） clone是一次系统调用（导致连接建立慢的原因，优化方案：线程池化。但是要注意过多的线程会导致CPU时间浪费在线程的上下文切换上）
6. recv cfd **阻塞的**

BIO的弊端：所有内核的调用都是阻塞

### NIO
1. socket 创建一个server socket 返回sfd（服务端的文件描述符）
2. bind sfd到一个地址 端口
3. listen sfd
4. accept sfd 返回接收的socket的cfd（客户端的文件描述符）或者-1 **非阻塞的** accept是一次系统调用
5. recv cfd **非阻塞**

NIO的弊端：虽然recv的系统调用是非阻塞的，但是很多次的调用是无意义的（没有数据到来），浪费的


### NIO多路复用器
多条 **路（IO）** 通过一次系统调用，获得IO的状态，然后由程序自己对感兴趣的状态的IO进行R/W

#### SELECT
synchronous I/O multiplexing
文件描述符个数的限制：FD_SETSIZE
```
yum install man man-pages
man 2 select
cd /proc/sys/fs/epoll
```

其实无论是NIO，还是SELECT或者POLL，他们都要遍历所有的I/O询问状态。
只不过在NIO中，由应用程序执行遍历操作，这个遍历过程的成本在用户态内核态的切换。
而SELECT/POLL的遍历过程只触发了一次系统调用，此时由应用程序将fds传递给内核，由内核执行遍历操作。

selec/poll的弊端：每次都要重新重复的传递fds，并进行全量遍历

> 中断 -》 回调  
> 在epoll之前的回调：只是完成了将网卡发来的数据走内核网络协议，最终关联到fd的buffer。所以，应用程序在某一时间如果询问内核某一个或某些fd是否读写时，会有状态返回。
> 如果内核在回调处理中加入

### TCP连接状态
FIN_WAIT2：发起连接断开的一端，在收到对端的FIN之前，本端连接处于FIN_WAIT2状态。
CLOSE_WAIT：在被动关闭连接的情况下，已经接收到FIN，但是还没有发送自己的FIN时，连接处于CLOSE_WAIT状态。
TIME_WAIT：发起连接断开的一端，在收到对端的FIN，并发送了ACK之后，连接处于TIME_WAIT状态。

> 为什么要有TIME_WAIT状态
> 假设最终的ACK丢失，主机2将重发FIN（假设先由主机1发起的连接断开请求），主机1必须维护TCP状态信息以便可以重发最终的ACK，否则会发送RST，结果主机2认为发生错误。TCP实现必须可靠地终止连接的两个方向，主机1必须进入TIME_WAIT状态，因为主机1可能面临重发最终ACK的情形。

### Java中使用EPOLL时的系统调用
1. socket(AF_INET6, SOCK_STREAM, IPPROTO_IP) = 8
2. fcntl(8, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
3. bind(8, {sa_family=AF_INET6, sin6_port=htons(9000), inet_pton(AF_INET6, "::", &sin6_addr), sin6_flowinfo=htonl(0), sin6_scope_id=0}, 28) = 0
4. listen(8, 50)
5. epoll_create(256)
6. epoll_ctl(11, EPOLL_CTL_ADD, 8, {EPOLLIN, {u32=8, u64=140681653780488}}) = 0
7. epoll_wait(11, [{EPOLLIN, {u32=8, u64=140681653780488}}], 8192, -1) = 1
8. accept(8, {sa_family=AF_INET6, sin6_port=htons(45812), inet_pton(AF_INET6, "::1", &sin6_addr), sin6_flowinfo=htonl(0), sin6_scope_id=0}, [28]) = 12
9. fcntl(12, F_SETFL, O_RDWR|O_NONBLOCK)   = 0
10. epoll_ctl(11, EPOLL_CTL_ADD, 12, {EPOLLIN, {u32=12, u64=140681653780492}}) = 0
11. epoll_wait(11, [{EPOLLIN, {u32=12, u64=140681653780492}}], 8192, -1) = 1
12. read(12, "123123123\n", 1024)           = 10
13. write(12, "123123123\n", 10)            = 10

当有N个fd有R/W处理的时候，将N个fd分组，每一组一个selector，将一个selector压到一个线程上。即，一个线程一个selector，处理一批fd，且处理过程是线性的。当有一组这样selector时，就实现了并行的处理fd。

![Netty架构图](https://github.com/geyi/ReliableGold/blob/master/image/Netty/Netty%E6%9E%B6%E6%9E%84%E5%9B%BE.png?raw=true)

## C10K

http://www.kegel.com/c10k.html


## RPC
- netty中，一个ServerBootstrap可以绑定多个端口，每个端口对应着相同的handler。也可以多个ServerBootstrap分别绑定到不同的端口，每个端口拥有不同的handler。
- 将NioEventLoopGroup视为CPU资源。
- 只有块设备才能做mmap映射
- 操作系统通过关闭中断直接干预数据从网卡拷贝到内核空间的套接字缓存。同样在设计程序时，程序员也应该尽可能快的把数据从内核空间的套接字缓存区读到用户空间。
- I/O读取是线性的（从内核到app）
- 无状态：在连接池取连接，并在发送和返回的生命周期里锁定连接。
- 有状态：consumer + provider端同步实现有状态协议（requestID）。发送和接受可以异步，连接可以共享使用。