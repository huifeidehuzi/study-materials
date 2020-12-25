# 多线程与并发-ReentrantReadWriteLock



## 为了有了ReentrantLock还需要ReentrantReadWriteLock?

ReentrantLock虽然可以重入，但是不可以共享，ReentrantReadWriteLock读锁可以共享，使用ReentrantReadWriteLock可以推广到大部分读，少量写的场景，因为读线程之间没有竞争



## ReentrantReadWriteLock底层实现原理？

底层基于ReentrantLock+AQS，支持公平/非公平机制



### ReentrantReadWriteLock类关系图

[![rWiSUO.jpg](https://s3.ax1x.com/2020/12/25/rWiSUO.jpg)](https://imgchr.com/i/rWiSUO)

从类结构图来看，ReentrantReadWriteLock维护了Sync对象，Sync对象继承至AQS，支持公平/非公平机制，此外，ReentrantReadWriteLock还维护了ReadLock、WriteLock两把锁，实现至Lock

其中，对ReentrantReadWriteLock的操作大多数维护在Sync中



### ReentrantReadWriteLock结构

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    // 读锁
    private final ReentrantReadWriteLock.ReadLock readerLock;
    // 写锁
    private final ReentrantReadWriteLock.WriteLock writerLock;
    // AQS实现对象
    final Sync sync;
    
    // 具体的ReadLock和WriteLock不做分析，他们调用的都是Sync内维护的方法（实现在Sync中）
}
```



### 具体的ReadLock和WriteLock不做分析，他们调用的都是Sync内维护的方法（实现在Sync中）



### Sync实现

#### Sync结构

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 读写锁状态
    // 高16位为读锁，低16位为写锁
    static final int SHARED_SHIFT   = 16;
    // 读锁单位
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    // 读锁最大数量：65535
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    // 写锁最大数量：65535
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

    // 当前持有读锁的线程数量
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    // 当前持有写锁的线程数量
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

    // 计数器,与读锁配套使用
    static final class HoldCounter {
        // 计数，某个读线程重入的次数
        int count = 0;
        // 当前该线程的tid字段的值
        final long tid = getThreadId(Thread.currentThread());
    }

    // 本地线程计数器
    // 可以看到这里使用ThreadLocal将HoldCounter绑定到当前线程上，同时HoldCounter也持有线程Id，这样在释放锁的时候才能知道ReadWriteLock里面缓存的上一个读取线程（cachedHoldCounter）是否是当前线程。这样做的好处是可以减少ThreadLocal.get()的次数，因为这也是一个耗时操作。需要说明的是这样HoldCounter绑定线程id而不绑定线程对象的原因是避免HoldCounter和ThreadLocal互相绑定而GC难以释放它们（尽管GC能够智能的发现这种引用而回收它们，但是这需要一定的代价），所以其实这样做只是为了帮助GC快速回收对象而已
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }

    // 当前持有读锁线程的缓存
    private transient ThreadLocalHoldCounter readHolds;  
    // 上一个持有读锁线程的计数器
    private transient HoldCounter cachedHoldCounter;
    // 第一个持有读锁的线程
    private transient Thread firstReader = null;
    // 第一个持有读锁的线程的计数
    private transient int firstReaderHoldCount;
}
```



#### Sync构造

```java
Sync() {
    // 初始化读锁计数器
    readHolds = new ThreadLocalHoldCounter();
    // 设置AQS.state
    setState(getState());
}
```



#### Sync公平/非公平锁实现

```java
// 非公平锁
static final class NonfairSync extends Sync {
    // 写锁不需要排队
    final boolean writerShouldBlock() {
        return false;
    }
    // 这只是一种启发式地避免写锁无限等待的做法，它在遇到同步队列的head后继为写锁节点时，会让readerShouldBlock返回true代表新来的读锁（new reader）需要阻塞等待这个head后继。但只是一定概率下能起到作用，如果同步队列的head后继是一个读锁，之后才是写锁的话，readerShouldBlock就肯定会返回false了
    // 可以理解为，等待队列中的head.next == 写锁时，需要排队，为读锁时，不需要排队
    final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();
    }
}

// 公平锁
static final class FairSync extends Sync {
    // 读锁和写锁需要判断等待队列中是否有排队的线程，如果有则需要排队获取锁
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}
```



#### Sync核心方法

##### tryRelease 释放独占锁（写锁）

```java
// 尝试释放独占锁
protected final boolean tryRelease(int releases) {
    // 持有锁的线程是否当前线程，避免释放其他线程的锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 计算AQS.state状态
    int nextc = getState() - releases;
    // 释放锁是否成功
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        // 释放成功则清空独占线程
        setExclusiveOwnerThread(null);
    // 设置状态
    setState(nextc);
    return free;
}
```



##### tryAcquire 申请独占锁（写锁）

```java
// 尝试申请独占锁
protected final boolean tryAcquire(int acquires) {
    // 当前线程
    Thread current = Thread.currentThread();
    // 状态
    int c = getState();
    // 持有写锁的线程数量
    int w = exclusiveCount(c);
    // 状态不为0，表示当前有线程持有写锁
    if (c != 0) {
        // 持有写锁线程数量为0或者持有写锁的线程不是当前线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 持有写锁数量线程超过最大值
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 设置AQS.state
        setState(c + acquires);
        return true;
    }
    // c==0，当前没有任何线程持有写锁，则直接申请加锁
    // 公平锁 || CAS state失败
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    // 设置持有写锁线程为当前线程
    setExclusiveOwnerThread(current);
    return true;
}
```



##### tryReleaseShared 释放共享锁（读锁）

```java
// 尝试释放读锁
protected final boolean tryReleaseShared(int unused) {
    // 当前线程
    Thread current = Thread.currentThread();
    // 当前线程是一个持有读锁的线程
    if (firstReader == current) {
        // 设置写锁计数缓存
        // 持有读锁线程数量只有1个的话，直接设置为空
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            // 否则计数缓存-1
            firstReaderHoldCount--;
    } else {
        // 获取当前线程持有的计数器
        HoldCounter rh = cachedHoldCounter;
        // 计数器为空或者计数器的tid不为当前线程的tid，则重新获取缓存计数器
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        // 如果计数<=1则移除缓存
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        // 缓存计数-1
        --rh.count;
    }
    // 自旋+CAS设置AQS.state
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```



##### tryAcquireShared 申请共享锁（读锁）

```java
// 尝试申请共享锁
protected final int tryAcquireShared(int unused) {
    // 当前线程
    Thread current = Thread.currentThread();
    int c = getState();
    // 持有写锁的线程数量不为0且持有锁的线程不是当前线程，则不能申请锁
    // 也就是说有正在写的操作，不能有读的动作（线程重入除外）
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    // 持有读锁线程的数量
    int r = sharedCount(c);
    // 不需要排队（非公平锁）且数量没有超过最大值且申请锁成功
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        // 设置读锁计数器的缓存
        if (r == 0) {
            // 设置第一个持有读锁的线程及数量
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            // 线程重入，计数+1
            firstReaderHoldCount++;
        } else {
            // 非线程重入，设置缓存计数
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    // 执行到这里表示申请读锁失败或者读锁线程需要排队（公平锁）
    // 则再次尝试申请读锁，内部是自旋+CAS申请，逻辑与此方法差不多，这里不做分析
    return fullTryAcquireShared(current);
}
```



## 读锁和写锁的最大数量是多少?

都是65535，参考Sync结构中字段说明



## 本地线程计数器ThreadLocalHoldCounter是用来做什么的？

使用ThreadLocal将HoldCounter绑定到当前线程上，同时HoldCounter也持有线程Id，这样在释放锁的时候才能知道ReadWriteLock里面缓存的上一个读线程（cachedHoldCounter）是否是当前线程。这样做的好处是可以减少ThreadLocal.get()的次数，因为这也是一个耗时操作

可以理解为，便于在tryReleaseShared时更快的判断申请释放读锁的线程是否为当前线程



## 写锁的获取与释放是怎么实现的？

参考tryAcquire方法和tryRelease方法实现，注：加锁逻辑，如果实现是公平的，则需要排队，非公平则不需要排队



## 读锁的获取与释放是怎么实现的?

参考tryAcquireShared方法和tryReleaseShared方法，注：加锁逻辑，如果实现是公平的，则需要排队，非公平也需要判断等待队列中的head.next是否为写锁节点，如果是则需要排队，反之则不需要



## 什么是锁的升降级? RentrantReadWriteLock为什么不支持锁升级?

锁降级指：写锁降级为读锁。**注：如果当前线程持有写锁，然后释放，最后再获取读锁，这种分阶段的不是降级，真正的降级是指：当前线程持有写锁后，继续持有读锁，在读锁未释放前，将持有的写锁释放的过程**

不支持锁升级的原因：**保证数据可见性，如果读锁已被多个线程获取，其中任意线程成功获取了写锁并更新了数据，该更新对其他获取到读锁的线程是不可见的**

