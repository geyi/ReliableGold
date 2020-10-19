# 程序，进程，线程

程序：数据加算法（静态）。  
进程：运行中的程序（动态）。  
线程：比进程中更小的执行单元，轻量级的进程。（一个程序里不同的执行路径）  
线程的启动方式：Thread，Runnable，Callable，线程池。

sleep：线程进入睡眠状态，不释放锁，不占用CPU资源。  
yield：线程回到等待执行队列，进入就绪状态。  
join：先去执行其他线程，用来等待一个线程的结束。  
wait：线程进入等待状态，释放锁。  
notify：唤醒处于等待状态的线程，不释放锁。

## 线程状态
- NEW（出生）
- RUNNABLE（就绪，运行）
- BLOCKED（阻塞）
- TIMED_WAITING（睡眠）
- WAITING（等待）
- TERMINATED（死亡）

getState方法可以获取线程状态

# synchronized
1. 锁的是对象不是代码。
2. 同步方法和非同步方法可以同时执行。
3. 可重入。
4. 抛出异常时自动释放锁。
5. **锁升级**（偏向锁、轻量级锁、自旋锁、重量级锁）
6. 不要用String常量作为锁对象。
7. 

执行时间短，并发线程少，用自旋锁。执行时间长，并发线程多，用synchronized。

# volatile
- 保证可见性（因为每个线程都有自己的工作内存，所以存在可见性问题）
- 禁止指令重排（双重检查锁的单例模式）

**volatile不能保证原子性**
> 所谓原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何context switch（切换到另一个线程）。

Object o = new Object();
1. 分配内存
2. 初始化
3. 将变量o指向对象的内存地址

存在1、3、2的指令重排

count++并不是原子操作。

# 锁优化
锁细化：减小同步代码块的范围  
锁粗化：如果一个方法内有多个同步代码块，不如直接使用synchronized修饰方法  
锁对象使用final修饰，避免指向锁对象引用被修改

# CAS
CAS是CPU原语指令级的原子操作

ABA问题，使用版本号解决，版本号是递增的。

LongAdder使用了分段锁

# ReentrantLock
ReentrantLock与Synchronized
- 尝试获取锁（tryLock）
- 可打断的加锁（lockInterruptibly）
- 公平锁非公平锁（ReentrantLock带参数的构造方法）
- 可以实现选择性通知（Condition）

# LockSupport
- park与unpark（线程挂起与唤醒，在AQS中使用）。
- unpark可以先于park调用。

# CountDownLatch
倒数锁。等待一组线程执行完，再继续执行await后的操作。

# CyclicBarrier
CyclicBarrier：循环栅栏。直到指定数量的参与者都调用了await方法，栅栏才会被推倒，参与者继续执行。

# Phaser
Phaser [feɪz] ：阶段性栅栏。参加婚礼的例子。

# ReentrantReadWriteLock
ReentrantReadWriteLock：读写锁。读读并行，读写互斥，写写互斥。

# Semaphore
Semaphore [ˈseməfɔːr] ：信号量。控制并行量。

# Exchanger
Exchanger：两个线程间交换数据。线程间通信。

两个题目

AQS源码

VarHandle，除了完成普通操作之外，还可以完成原子性线程安全的操作

ThreadLocal
一个线程一个ThreadLocalMap，里面可以存放多个ThreadLocal
应用场景：
1. 声明式事务，保证同一个connection
2. 请求的上下文，OAuthContext


强软弱虚
强引用：new出来的对象
软引用：如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。
弱引用：弱引用与软引用的区别在于，只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。
虚引用：虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。应用：回收堆外内存。


HashTable
HashMap
Collections.synchronizedMap
ConcurrentHashMap：线程安全实现Synchronized CAS

ArrayList
Vector
ConcurrentLinkedQueue：单向链表，add remove offer peek poll。线程安全实现CAS

ConcurrentHashMap
ConcurrentSkipListMap：跳表，有序

CopyOnWriteArrayList：底层数据结构数组，添加元素时加锁（ReentrantLock），写时复制，读取时不加锁。
CopyOnWriteArraySet：底层数据结构数组，先判断要加入的元素是不是已存在，然后加锁（ReentrantLock），复制数组，添加元素。

LinkedBlockingQueue：单向链表，put take。阻塞实现 ReentrantLock Condition
ArrayBlockingQueue：数组，put take。阻塞实现 ReentrantLock Condition

DelayQueue
应用场景：1. 按时间进行任务调度

SynchronousQueue：同步移交队列。线程间通信。

TransferQueue：如果已经有一个消费者在等待消费，那么transfer方法会立刻返回，否则一直阻塞，直到有一个消费者接收到传递的元素。

ArrayList
LinkedList
Vector
CopyOnWriteArrayList
ConcurrentLinkedQueue
ArrayBlockingQueue
LinkedBlockingQueue

HashMap
LinkedHashMap
TreeMap
HashTable
ConcurrentHashMap
ConcurrentSkipListMap

HashSet
LinkedHashSet
TreeSet
CopyOnWriteArrarySet
ConcurrentSkipListSet

Executor：把定义与运行分开
ExecutorService：

Future：代表一个线程任务的运行结果
FutureTask：既是一个可运行的线程任务，又是一个保存运行结果的对象

Executors：线程池的工厂
newFixed
newCached
newSingle
newScheduled

线程池大小设置，N(threads) = N(cpu) * U(cpu) * (1 + W/C)

并发是指任务提交，并行是指任务执行。
并行是并发的子集。

ForkJoinPool
分解汇总的任务

WorkStealing
ParallelStream



























