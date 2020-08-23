程序想获取数据时

VFS：虚拟文件系统

把VFS当做暴露给用户空间程序的统一接口，用来访问不同的硬件设备（VFS挂载了不同的硬件设备）

两个程序如果访问的是同一个文件，文件的inode id和pagecache是共享的

dirty：程序修改过的pagecache被标记为脏

flush：
- 每次修改都将数据刷到磁盘
- 固定时间间隔后由系统内核将数据刷到磁盘


FD：文件描述符

seek指针

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
2. 使用 losetup将磁盘镜像文件虚拟成块设备
3. 将块设备格式化为ext2格式
4. 挂载块设备
5. 查看设备挂载情况
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

lsof：用于列出一个进程打开了哪些文件。如：`lsof -op $$`

proc是系统内存的映射，这个目录的内容不在硬盘上而是在内存里。
```
/proc
/proc/$$
$$表示当前bash的pid（$BASHPID）
/proc/$$/fd（lsof -op $$）
```

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
[root@192 ~]# ls ./ /ooxx 1> ls1.out 2>ls2.out
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

当标准输出和标准错误输出重定向到同一个文件时，标准错误输出会被覆盖
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

当我们需要将标准输出和标准错误输出重定向到同一个文件时，`2> 1`的写法实际上是将标准错误输出重定向到了名称为1的文件中。所以如果我们要讲一个文件描述符重定向到另一个文件描述符时，重定向符号后应跟着一个&符号，如：`2>& 1`。
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

同步
异步
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

### NIO多路复用器


## C10K
