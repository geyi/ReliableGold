# Git

## 初始化一个项目并上传到GitHub
```
ssh-keygen -t rsa -C "geyichn@126.com" -f github
ssh-agent bash
ssh-add github
cat github.pub
```

```
git config --global user.email "geyichn@126.com"
git config --global user.name "geyi"
git init
git add <file>
git commit -m ''
git remote add origin git@github.com:geyi/ReliableGold.git
git push --set-upstream origin master
```

## Git Bash中文乱码问题
1. Options -> text -> Local & Character set
2. 执行 `git config --global core.quotepath false`
3. 使用UTF-8字符集编码保持文件，否则 `git diff` 会出现乱码

# 计算底层知识

## 汇编语言

* 本质：机器语言的助记符，汇编语言就是机器语言

## DMA

是一种硬件，通常我们读写文件是需要CPU参与的，读写的时候通常还要阻塞，等读写完成然后才继续下一次读写，有了DMA就相当于有个帮手，CPU告诉它有这么多数据要从A（内核缓存区）搬到B（网卡缓冲区），DMA知道后就自己开干了，CPU就可以不管了，该干啥干啥去

## CPU
- PC：Program Counter 程序计数器 （记录当前指令地址）
- Registers：暂时存储CPU计算需要用到的数据
- ALU：Arithmetic & Logic Unit 运算单元
- CU：Control Unit 控制单元
- MMU：Memory Management Unit 内存管理单元
- Cache

## 缓存行
- 缓存行对齐（伪共享），解决伪共享的方法是缓存行对齐。
- 缓存一致性协议，MESI（Modified Exclusive Shared Invalid）
- 缓存行大小，英特尔CPU的缓存行大小为64字节
- 有些无法被缓存的数据或者跨越多个缓存行的数据依然必须使用总线锁

缓存行对齐：对于有些特别敏感的数字，存在线程高竞争的访问时，为了保证不发生伪共享，可以使用缓存行对齐的编程方式
- JDK7中，很多采用long padding提高效率
- JDK8，加入了@Contended注解，需要加上：JVM -XX:-RestrictContended