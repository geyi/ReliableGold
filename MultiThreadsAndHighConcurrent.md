# 程序，进程，线程

程序：数据加算法（静态）。  
进程：运行中的程序（动态）。  
线程：比进程中更小的执行单元，轻量级的进程。（一个程序里不同的执行路径）  
线程的启动方式：Thread，Runnable，Callable，线程池。

sleep：线程进入睡眠状态，**不释放锁**，不占用CPU资源。  
yield：线程回到等待执行队列，进入就绪状态。  
join：先去执行其他线程，用来等待一个线程的结束。  
wait：线程进入等待状态，**释放锁**。  
notify：唤醒处于等待状态的线程，**不释放锁**。

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
> 这里的原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何context switch（切换到另一个线程）。

Object o = new Object();
1. 分配内存
2. 调用构造方法初始化
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

# 两个题目

# AQS源码

VarHandle，除了完成普通操作之外，还可以完成原子性线程安全的操作（JDK9）。比反射快！

# ThreadLocal
一个线程一个ThreadLocalMap，里面可以存放多个ThreadLocal

应用场景：
1. 声明式事务，保证同一个connection
2. 请求的上下文，OAuthContext


## 强软弱虚
- 强引用：new出来的对象
- 软引用：如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。
- 弱引用：弱引用与软引用的区别在于，只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。
- 虚引用：虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。应用：回收堆外内存。


# HashMap
- 如果两个对象使用equals方法比较的结果是相等的，那么这两个对象分别调用hashCode方法应该得到相同的整型值。
- 如果两个对象使用equals方法比较的结果是不相等的，那么这两个对象分别调用hashCode方法应该得到不同的整型值，但这不是必须的。然而，程序员应该知道，为不相等的两个对象生成不同的整型值可以提高哈希表的性能。
- Object类中定义的hashCode方法为不同的对象返回了不同的整型值。这通常是通过将对象的内部地址转换为整数来实现。
> 以上三点说明了为什么重写equals方法的同时也要重写hashCode方法
- 调用resize方法进行扩容时，会将原来的链表拆分成两个相对更短的链表，然后分别放到扩容后的数组的j位置和(j + oldCap)位置。（树形结构的节点同样会拆分成更小的树）
- 数组长度使用2的N次方是因为这样可以利用位运算（提高计算效率）得到元素在数组中的位置。另一方面则是为了提高存储效率，例如容量为8时，任何数与0111做&运算都会落在[0,7]的区间内，即落入给定的8个哈希桶中，存储空间的利用率是100%。反例如容量为7，空间利率只有60%不到。
- 首次put元素时，会初始化一个长度为16的Node类型的数组。然后使用key的hash值与(数组长度 - 1)进行按位与运算，得出新元素在数组中的位置。如果该位置没有被其它元素占用，则直接将新元素放在该位置，否则会比较新元素与老元素的key的hash值及内容是否相等，相等则用新元素的value值替换老元素的value值。否则，判断老元素是否是TreeNode（红黑树），如果是，则将新元素放到红黑树中。如果不是树型结构，则为链表，对链表进行向后遍历，如果链表中存在老元素的key的hash值及内容与新元素相等，同上。否则，将新元素放到链表的末尾。
# LinkedHashMap
- 继承了HashMap，自己维护了一个双向链表，可以保证两种顺序，插入顺序和读顺序（读过往后放）
# TreeMap
- 底层是红黑树，可以根据key进行排序。比较的两种方式：实现Comparator接口或者实现Comparable接口。
# HashTable
- 使用synchronized同步方法保证线程安全。
- 数据结构是数组 + 链表。向链表中增加元素时使用头插法。
# Collections.synchronizedMap
- 使用synchronized同步代码块保证线程安全，锁是SynchronizedMap对象本身。
# ConcurrentHashMap
- JDK1.7版本的ConcurrentHashMap存在的问题
  - 使用segment之后，会增加ConcurrentHashMap的存储空间
  - 单个segment过大时，并发性能会急剧下降
- 线程安全实现Synchronized + CAS
- 当put的元素在哈希桶数组中不存在时，则直接CAS进行写操作（tabAt -> getObjectVolatile，casTabAt -> compareAndSwapObject）。当put的元素在哈希桶数组中存在，并且不处于扩容状态时，则使用synchronized锁定哈希桶数组中第i个位置中的第一个元素，接着进行double check。校验通过后，会遍历当前冲突链上的元素，并选择合适的位置进行put操作。
- ForwardingNode是Node的子类，hash值为-1
# ConcurrentSkipListMap
- 底层数据结构是跳表，实现了有序，空间换时间。
- 从跳跃表中搜索节点的过程是先找到目标节点的前驱结点（q是前驱节点，r是被比较的节点），然后获取前驱结点的下一个节点进行比较。
- 节点类型有：HeadIndex（每一层级的头节点），Index（跳表层级的节点），Node（常规节点）

# ArrayList
- 底层数据结构为数组，支持随机访问，读取指定元素的时间复杂度为O(1)，是一种读取很快的集合类型。初始容量10，从索引0开始添加元素，超过数组容量会创建一个新的数组，容量为 oldCapacity + (oldCapacity >> 1)，然后使用System.arraycopy方法将原数组的元素复制到新数组中。为了避免不必要的复制，在提前知道元素个数时应该使用带有初始容量参数的构造方法创建ArrayList。
# LinkedList
- 底层数据结构为双向链表，不支持随机访问。新元素直接添加到链表的尾部，是一种插入和删除都很快的集合类型。适用于需要频繁插入删除元素的场景。
# Vector
- 底层数据结构为数组，默认的扩容策略是扩容到原来容量的两倍。是线程安全的，因为所有对集合的操作都加了锁（方法被Synchronized关键字修饰）。
# CopyOnWriteArrayList
- 底层数据结构数组，添加元素时加锁（ReentrantLock），写时复制，读取时不加锁。（用在读多写少，数据一致性要求不高的场景）
# ConcurrentLinkedQueue
- 单向链表，add remove offer poll peek（不从队列中移除元素）。线程安全实现CAS。
# ArrayBlockingQueue
- 数组，put take。阻塞实现 ReentrantLock Condition。有界队列。（阻塞指的是，队列满时put阻塞，队列为空时take阻塞）
# LinkedBlockingQueue
- 单向链表，put take。阻塞实现 ReentrantLock Condition。无界队列。

# Set
- 元素无序，不可重复
# HashSet
- 基于HashMap实现的
# LinkedHashSet
- 基于HashSet实现，实际是一个LinkedHashMap
# TreeSet
- 基于TreeMap实现
# CopyOnWriteArraySet
- 底层数据结构数组，先判断要加入的元素是不是已存在，然后加锁（ReentrantLock），复制数组，添加元素。
# ConcurrentSkipListSet

# DelayQueue
- 内部有一个PriorityQueue
- 应用场景：1. 按时间进行任务调度（获取推送统计数据）

# SynchronousQueue
- 同步移交队列，put元素时会一直阻塞，直到有一个消费者（线程）来消费。线程间通信。

# TransferQueue
- 如果已经有一个消费者在等待消费，那么transfer方法会立刻返回，否则一直阻塞，直到有一个消费者接收到传递的元素。

# 常见的Collections、Map
```
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
CopyOnWriteArraySet
ConcurrentSkipListSet
```


Executor：把定义与运行分开  
ExecutorService：

Future：代表一个线程任务的运行结果  
FutureTask：既是一个可运行的线程任务，又是一个保存运行结果的对象

Executors：线程池的工厂
- newFixed - 任务量相对固定
- newCached - 大量的执行时间短的任务
- newSingle - 单线程，顺序执行任务
- newScheduled - 定时任务

线程池大小设置，N(threads) = N(cpu) * U(cpu) * (1 + W/C)
- N(cpu) - 处理器的核心数目
- U(cpu) - 期望的CPU的利用率
- W/C - 等待时间与计算时间的比率

并发是指任务提交，并行是指任务执行。
并行是并发的子集。

ForkJoinPool：分解汇总的任务

WorkStealing  
ParallelStream



























