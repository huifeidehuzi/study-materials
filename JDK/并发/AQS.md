# AQS
###简介

```
CLH自旋阻塞队列 自旋+LockSupport+CAS实现
```
###原理

```
这里拿ReentrantLock的公平锁举例来分析AQS的加锁、入队、出队、解锁等原理
ReentrantLock内部声明了sync内部类，继承至AbstractQueuedSynchronizer（juc包下都是如此，比如读写锁、信号量、countDownLatch、线程池）

同时ReentrantLock又分为公平锁（FairSync）与非公平锁（NonfairSync），默认非公平锁，可能是因为不需要维护队列，减少开销的原因吧
   
```
* `队列` 队列第一个node永远为空（node对象不为空，里面的字段为空），试想一下，排队买票，队伍的第一个人是不需要排队的


```
队列主要结构
//头节点
private transient volatile Node head;
//尾节点
private transient volatile Node tail;
//同步状态/锁状态
private volatile int state;
//线程
volatile Thread thread;
//节点状态
volatile int waitStatus;
```
* `加锁` 


```
//1.调用FairSync.acquire方法，cas+1
final void lock() {
   acquire(1);
}
    
//2.调用AbstractQueuedSynchronizer的acquire()方法
public final void acquire(int arg) {
    //竞争锁失败&&入队
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE),arg))
            //当前竞争的线程中断
            selfInterrupt();
    }

//2.1 tryAcquire()分析
protected final boolean tryAcquire(int acquires) {
      //当前线程
    final Thread current = Thread.currentThread();
    //获取锁的状态
    int c = getState();
    //如果没有上锁
    if (c == 0) {
        //是否需要排队（分2种情况，下面介绍），如果不需要排队就cas加锁
        //1.队列没有初始化 h!=t,队列为空h和t都是null，返回false，不需要排队
        //2.队列已经被初始化，也就是h!=t成立
            //2.1 队列元素>1(已经有节点在排队了) 
                //2.1.1 h.next==null不成立，因为第二个节点不可能为空h.next.thread!=currentThread成立（第二个排队的节点不是当前线程），需要排队
                //2.1.2 h.next==null不成立，因为第二个节点不可能为空h.next.thread!=currentThread不成立（第二个排队的节点是当前线程），不需要排队
        if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
            //如果加锁成功，设置线程为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
    } 
    //判断是否为当前线程  
    else if (current == getExclusiveOwnerThread()) {
        //加锁状态+1
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        //设置状态值
        setState(nextc);
        return true;
    }
    return false;
}

//2.2 addWaiter() 分析
//入队方法
private Node addWaiter(Node mode) {
    //新建node
    Node node = new Node(Thread.currentThread(), mode);
    //队尾
    Node pred = tail;
    //如果队尾不为空，表示队列中有等待的线程
    if (pred != null) {
        //将队尾设置为新节点的上个节点引用
        node.prev = pred;
        //cas设置队尾为新节点
        if (compareAndSetTail(pred, node)) {
            //之前队尾的下一个节点=真正的队尾节点
            pred.next = node;
            return node;
        }
    }
    //死循环设置节点 可自行查阅源码
    //1.首次进入循环，尾节点tail为空，就New Node（node里的字段都是空的）设置新节点（compareAndSetHead(new Node())）并且将队列的头尾都指向自己tail=head
    //2.再次进入循环，尾节点t不为空，将需要排队的节点node的prev=t，此时需要排队的node节点就变成了尾节点（compareAndSetTail(t, node)），并且将之前为节点t.next=node(新节点)    
    enq(node);
    return node;
}

//2.3 acquireQueued(addWaiter(Node.EXCLUSIVE),arg))分析
//此方法功能是再自旋一次，尝试拿锁，因为在入队中/入队后过程中可能会有锁被释放
final boolean acquireQueued(final Node node, int arg) {
        //是否加锁失败
        boolean failed = true;
        try {
            //是否被中断
            boolean interrupted = false;
            for (;;) {
                //节点的上一个节点
                final Node p = node.predecessor();
                //如果是头节点，表示node就是队列的第二个节点，也就是需要排队的节点，那就尝试加锁（再自旋一次，因为可能在过程中有其他线程释放锁）
                if (p == head && tryAcquire(arg)) {
                    //设置node为头节点，并且将node的prev设置为空，也就是中断与之前头节点的关联，这就是出队的一个步骤，（加锁成功后需要从队列中移除，不需要再排队）
                    setHead(node);
                    p.next = null; //将之前的头节点的next引用置空，中断与现有头部节点的关联，现在就完成出队了，出队后，GC会回收此节点
                    failed = false;
                    return interrupted;
                }
                //如果加锁失败，入队后需要将上一个节点状态更改为-1
                //如果waitstatus=-1 则park线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            //加锁失败
            if (failed) 
                cancelAcquire(node);
        }
    }
```

`释放锁` 

```
//1.NonfairSync.lock()方法，调用tryRelease()

//尝试释放锁
protected final boolean tryRelease(int releases) {
    //状态-1
    int c = getState() - releases;
    //是否当前线程，如果不是就抛异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    //是否自由状态
    boolean free = false;
    if (c == 0) {
        free = true;
        //设置当前线程为空
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```