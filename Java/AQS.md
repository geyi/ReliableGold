# AQS结构
AbstractQueuedSynchronizer（抽象的队列式的同步器）重要的成员变量
```java
/**
  * Head of the wait queue, lazily initialized.  Except for
  * initialization, it is modified only via method setHead.  Note:
  * If head exists, its waitStatus is guaranteed not to be
  * CANCELLED.
  */
// 阻塞队列的头节点，延迟初始化。
// 除了初始化，它只能通过setHead方法进行修改
// 注意：如果头节点存在，它的waitStatus属性一定不是CANCELLED（1）
// 直接把它当做当前持有锁的线程
// 阻塞队列不包含头节点
private transient volatile Node head;


/**
  * Tail of the wait queue, lazily initialized.  Modified only via
  * method enq to add new wait node.
  */
// 阻塞队列的尾节点，延迟初始化。
// 只能通过enq方法修改，以增加新的等待节点。
private transient volatile Node tail;


/**
  * The synchronization state.
  */
// 同步状态
// 当前锁的状态，0表示锁没有被占用，大于0表示有线程持有当前锁
// state可以大于1，说明锁可以重入，每次重入都在原来的基础上加1
private volatile int state;

/**
  * The current owner of exclusive mode synchronization.
  */
// 继承自AbstractOwnableSynchronizer
// 当前持有独占锁的线程
private transient Thread exclusiveOwnerThread;
```

![](../image/Java/aqs-0.png)

等待队列中的每个线程被包装成一个Node实例，数据结构是一个双向链表，Node源码
```java
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    // 标识这个节点在共享模式下
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    // 标识这个节点在独占模式下
    static final Node EXCLUSIVE = null;


    /** waitStatus value to indicate thread has cancelled */
    // 代表线程取消抢占锁
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    // 代表当前节点的后继节点对应的线程需要被唤醒
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    // 
    static final int CONDITION = -2;
    /**
      * waitStatus value to indicate the next acquireShared should
      * unconditionally propagate
      */
    static final int PROPAGATE = -3;


    /**
      * Status field, taking on only the values:
      *   SIGNAL:     The successor of this node is (or will soon be)
      *               blocked (via park), so the current node must
      *               unpark its successor when it releases or
      *               cancels. To avoid races, acquire methods must
      *               first indicate they need a signal,
      *               then retry the atomic acquire, and then,
      *               on failure, block.
      *   CANCELLED:  This node is cancelled due to timeout or interrupt.
      *               Nodes never leave this state. In particular,
      *               a thread with cancelled node never again blocks.
      *   CONDITION:  This node is currently on a condition queue.
      *               It will not be used as a sync queue node
      *               until transferred, at which time the status
      *               will be set to 0. (Use of this value here has
      *               nothing to do with the other uses of the
      *               field, but simplifies mechanics.)
      *   PROPAGATE:  A releaseShared should be propagated to other
      *               nodes. This is set (for head node only) in
      *               doReleaseShared to ensure propagation
      *               continues, even if other operations have
      *               since intervened.
      *   0:          None of the above
      *
      * The values are arranged numerically to simplify use.
      * Non-negative values mean that a node doesn't need to
      * signal. So, most code doesn't need to check for particular
      * values, just for sign.
      *
      * The field is initialized to 0 for normal sync nodes, and
      * CONDITION for condition nodes.  It is modified using CAS
      * (or when possible, unconditional volatile writes).
      */
    volatile int waitStatus;


    /**
      * Link to predecessor node that current node/thread relies on
      * for checking waitStatus. Assigned during enqueuing, and nulled
      * out (for sake of GC) only upon dequeuing.  Also, upon
      * cancellation of a predecessor, we short-circuit while
      * finding a non-cancelled one, which will always exist
      * because the head node is never cancelled: A node becomes
      * head only as a result of successful acquire. A
      * cancelled thread never succeeds in acquiring, and a thread only
      * cancels itself, not any other node.
      */
    // 前驱节点的引用
    volatile Node prev;


    /**
      * Link to the successor node that the current node/thread
      * unparks upon release. Assigned during enqueuing, adjusted
      * when bypassing cancelled predecessors, and nulled out (for
      * sake of GC) when dequeued.  The enq operation does not
      * assign next field of a predecessor until after attachment,
      * so seeing a null next field does not necessarily mean that
      * node is at end of queue. However, if a next field appears
      * to be null, we can scan prev's from the tail to
      * double-check.  The next field of cancelled nodes is set to
      * point to the node itself instead of null, to make life
      * easier for isOnSyncQueue.
      */
    // 后继节点的引用
    volatile Node next;


    /**
      * The thread that enqueued this node.  Initialized on
      * construction and nulled out after use.
      */
    // 线程的引用
    volatile Thread thread;
}
```

ReentrantLock的静态内部类Sync继承了AbstractQueuedSynchronizer，Sync有两个实现，分别为NonfairSync（非公平锁）和FairSync（公平锁）。以下是公平锁加锁的实现：
```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;


    final void lock() {
        acquire(1);
    }

    // 继承自AbstractQueuedSynchronizer类，为了方便分析把它copy到FairSync中
    public final void acquire(int arg) {
        // 尝试获取锁
        // 如果获取成功此方法直接结束
        // 失败则将当前线程包装成Node实例，压到阻塞队列中
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    /**
      * Fair version of tryAcquire.  Don't grant access unless
      * recursive call or no waiters or is first.
      */
    // 尝试获取锁，返回值为boolean类型，代表是否获取到锁
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // state == 0，表示没有线程占用锁
        if (c == 0) {
            // 判断阻塞队列中是否有其它线程在等待锁（因为是公平锁，讲究一个“先来后到”）
            // 有则直接返回false，获取锁失败
            if (!hasQueuedPredecessors() &&
                // 如果没有线程在等待，则CAS尝试将state由0改为1，成功就获取到了锁，返回true
                // 不成功，则说明被其它线程抢先了，返回false
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果state不等于0（重入），则判断当前线程是不是持有锁的那个线程
        else if (current == getExclusiveOwnerThread()) {
            // 是，则将state在原来的基础上加1
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            // 此时没有并发安全问题，直接设置state
            setState(nextc);
            return true;
        }
        return false;
    }

    // 继承自AbstractQueuedSynchronizer类
    private Node addWaiter(Node mode) {
        // 将当前线程封装成一个Node类型的实例，参数mode此时是Node.EXCLUSIVE，代表独占模式
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            // 将原来的尾节点设置为当前线程节点的前驱节点
            node.prev = pred;
            // 使用CAS设置tail变量
            if (compareAndSetTail(pred, node)) {
                // 设置成功则将原来的尾节点的后继节点设置为当前线程节点
                pred.next = node;
                return node;
            }
        }
        // 如果代码走到了这里，说明
        // 1. 尾节点为空，2. CAS设置tail失败
        enq(node);
        return node;
    }

    // 继承自AbstractQueuedSynchronizer类
    // 自旋方式加入到队列的尾部
    private Node enq(final Node node) {
        // 当有其它线程竞争插入阻塞队列队尾时，通过无限循环来解决，直到当前线程排到队尾为止
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                // 初始化head节点，使用CAS设置head变量
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // 把尾节点设置为当前节点的前驱节点
                node.prev = t;
                // 使用CAS设置tail变量
                if (compareAndSetTail(t, node)) {
                    // 设置tail成功，则将尾节点的后继节点设为当前节点
                    t.next = node;
                    return t;
                }
            }
        }
    }

    // 继承自AbstractQueuedSynchronizer类
    // 入参node，由addWaiter(Node.EXCLUSIVE)返回，此时已经进入阻塞队列
    // 如果acquireQueued(addWaiter(Node.EXCLUSIVE), arg))返回true的话
    // 意味着上面这段代码将进入selfInterrupt()，所以正常情况下，下面应该返回false
    // 这个方法非常重要，应该说真正的线程挂起，然后被唤醒后去获取锁，都在这个方法里了
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 获取当前线程节点（后简称“当前节点”）的前一个节点
                final Node p = node.predecessor();
                // p等于head，说明当前节点是阻塞队列的第一个，因为它的前驱是head
                // 注意，阻塞队列不包含head节点，head一般指的是占有锁的线程，head后面的才称为阻塞队列
                // 所以当前节点可以去尝试获取锁，原因如下：
                // 因为当前节点是阻塞队列的第一个
                // 又因为head是延迟初始化的，如果是刚刚初始化的，那么new Node()没有设置任何线程
                // 也就是说head不属于任何一个线程，所以作为阻塞队列的第一个节点可以尝试获取锁
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 走到这里说明当前节点不是阻塞队列的第一个节点或者抢锁失败
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    // 继承自AbstractQueuedSynchronizer类
    // 是否需要挂起线程
    // 入参pred表示前驱节点，node表示当前节点
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        // 前驱节点的waitStatus等于-1，说明前驱节点的状态正常，当前线程需要挂起，直接返回true
        if (ws == Node.SIGNAL)
            /*
              * This node has already set status asking a release
              * to signal it, so it can safely park.
              */
            return true;
        // 前驱节点的waitStatus大于0，说明前驱节点取消了排队获取锁
        // 这里需要知道一点：进入阻塞队列的线程会被挂起，而唤醒操作由前驱节点完成
        // 所以这个if中的操作是向前遍历将当前节点的pref变量指向一个waitStatus小于等于0的节点
        if (ws > 0) {
            /*
              * Predecessor was cancelled. Skip over predecessors and
              * indicate retry.
              */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
              * waitStatus must be 0 or PROPAGATE.  Indicate that we
              * need a signal, but don't park yet.  Caller will need to
              * retry to make sure it cannot acquire before parking.
              */
            // waitStatus等于0、-2、-3的情况
            // 每个新的Node入队时，waitStatus都是0
            // 所以，正常情况下，前驱节点是之前tail，那么它的waitStatus应该是0
            // 使用CAS将前驱节点设置为Node.SIGNAL（-1）
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

    // 继承自AbstractQueuedSynchronizer类
    // 如果shouldParkAfterFailedAcquire(p, node)返回true，说明线程需要被挂起
    // 此方法调用LockSupport.park(this)挂起线程
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
}
```

解锁
```java
class ReentrantLock {
    public void unlock() {
        sync.release(1);
    }

    // 继承自AbstractQueuedSynchronizer类
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // 如果c等于0，说明没有嵌套锁了，可以释放，否则还不能释放
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    // 继承自AbstractQueuedSynchronizer类
    // 唤醒后继节点
    // 入参node为head节点
    private void unparkSuccessor(Node node) {
        /*
          * If status is negative (i.e., possibly needing signal) try
          * to clear in anticipation of signalling.  It is OK if this
          * fails or if status is changed by waiting thread.
          */
        int ws = node.waitStatus;
        // 如果head节点的waitStatus小于0，则将其修改为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);


        /*
          * Thread to unpark is held in successor, which is normally
          * just the next node.  But if cancelled or apparently null,
          * traverse backwards from tail to find the actual
          * non-cancelled successor.
          */
        // 唤醒后继节点
        // 但是后继节点有取消排队情况（waitStatus等于1），这时从尾节点向前遍历，找到排在最前的waitStatus小于等于0的节点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            // 唤醒线程
            LockSupport.unpark(s.thread);
    }
}
```

公平锁和非公平锁有两处不同：
1. 非公平锁在调用 lock 后，首先就会使用CAS 进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。
2. 非公平锁在 CAS 失败后，和公平锁一样都会进入到 tryAcquire 方法，在 tryAcquire 方法中，如果这个时候发现锁被释放了（state == 0），非公平锁会直接 CAS 抢锁，但是公平锁会判断阻塞队列是否有线程处于等待状态，如果有则不去抢锁，乖乖排到后面。

> 原文：https://javadoop.com/post/AbstractQueuedSynchronizer