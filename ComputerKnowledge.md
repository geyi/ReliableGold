# 汇编语言
* 本质：机器语言的助记符，汇编语言就是机器语言

# DMA
是一种硬件，通常我们读写文件是需要CPU参与的，读写的时候通常还要阻塞，等读写完成然后才继续下一次读写，有了DMA就相当于有个帮手，CPU告诉它有这么多数据要从A（内核缓存区）搬到B（网卡缓冲区），DMA知道后就自己开干了，CPU就可以不管了，该干啥干啥去

# CPU
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

## 指令重排
CPU为了提高指令的执行效率，会在执行一条指令的过程中，去同时执行另一条指令，前提是两条指令没有依赖关系。

### 乱序的证明：com.airing.disorder.Disorder

new #2 <T>  
invokespecial #3 <T.<init>>  
astore_1

### 有序性保障

#### cpu内存屏障
1. sfence：写屏障
2. lfence：读屏障
3. mfence：读写屏障

#### lock指令
lock指令是一个Full Barrier，执行时会锁住内存子系统来确保执行顺序，甚至跨多个CPU。Software Locks通常使用了内存屏障或原子指令来实现变量可见性和保持程序顺序。

#### Java汇编指令内存屏障

- LoadLoad屏障：对于这样的语句Load1; LoadLoad; Load2;，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
- StoreStore屏障：对于这样的语句Store1; StoreStore; Store2;，在Store2及后续写操作执行前，保证Store1的写入操作对其它处理器可见。
- LoadStore屏障：对于这样的语句Load1; LoadStore; Store2;，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
- StoreLoad屏障：对于这样的语句Store1; StoreLoad; Load2;，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。

### volatile
volatile在不同层面的实现

- 字节码层面  
ACC_VOLATILE

- JVM层面  
> StoreStoreBarrier  
volatile写操作   
StoreLoadBarrier

> LoadLoadBarrier  
volatile读操作  
LoadStoreBarrier

### hanppens-before原则
- 程序次序规则：同一个线程内，按照代码出现的顺序，前面的代码先行于后面的代码，准确的说是控制流顺序，因为要考虑到分支和循环结构。
- 管程锁定规则：一个unlock操作先行发生于后面（时间上）对同一个锁的lock的操作。
- volatile变量规则：对一个volatile变量的写操作先行发生于后面（时间上）对这个变量的读操作。
- 线程启动规则：Thread的start()方法先行发生于这个线程的每一个操作。
- 线程终止规则：线程的所有操作都先行于此线程的终止检测。可以通过Thread.join()方法结束、Thread.isAlive()的返回值等手段检测线程的终止。
- 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupt()方法检测线程是否中断。
- 对象终结规则：一个对象的初始化完成先行于发生它的finalize()方法的开始。
- 传递性：如果操作A先行于操作B，操作B先行于操作C，那么操作A先行于操作C。

### as if serial
不管如何重排序，单线程执行结果不会改变。

## 合并写
Write Combining Buffer

## NUMA
Non Uniform Memory Access

# OS

推荐书籍：《Linux内核设计与实现》

## 内核分类
- 微内核 - 弹性部署、5G、IoT
- 宏内核 - PC、phone
- 外核 - 科研，实验中。为应用程序定制的操作系统

## 用户态和内核态
Linux内核跑在ring 0级， 用户程序跑在ring 3级，对于系统的关键访问，需要经过kernel的同意，保证系统健壮性。

## 进程 线程 纤程

进程与线程的区别？  
进程是操作系统进行资源分配的基本单位，线程是执行调度的基本单位。进程与进程之间互相拥有自己独立的内存空间，而线程没有自己独立的内存空间，它共享进程的内存空间。

### 纤程
用户态的线程，切换和调度不需要经过操作系统，优点：
1. 占用资源少
2. 切换简单