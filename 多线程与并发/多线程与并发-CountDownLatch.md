# 多线程与并发-CountDownLatch



## 什么是CountDownLatch?

CountDownLatch是一个同步工具类，用来协调多个线程之间的同步，它能使一个线程在等待另一个线程执行完后再继续执行，底层使用AQS实现，它有个计数器，构造CountDownLatch需指定该计数器的值，每次调用countDown()后计数器-1直至为0，当计数器为0，被CountDownLatch.await()后面的逻辑才会开始执行



## CountDownLatch适用于什么场景?

做线程同步，比如让线程交替执行，还可以让一组线程执行完之后再执行其他业务逻辑



## 调用流程图

### countDown方法大致调用流程图

[![ryiwVJ.jpg](https://s3.ax1x.com/2020/12/23/ryiwVJ.jpg)](https://imgchr.com/i/ryiwVJ)



### await大致调用流程图

[![ryiy26.jpg](https://s3.ax1x.com/2020/12/23/ryiy26.jpg)](https://imgchr.com/i/ryiy26)



## CountDownLatch底层实现原理?

```java
public class CountDownLatch {
    // 使用AQS实现
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
			  // 设置state值，最多能countDown多少次
        Sync(int count) {
            setState(count);
        }
				// 获取count数量
        int getCount() {
            return getState();
        }
				// 获取共享锁状态，为0表示当前没有锁，不为0表示有锁
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
			  
        // 尝试释放锁（或者说是尝试将state-1）
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                // 每次释放，state-1
                int nextc = c-1;
                // CAS操作释放
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
    
    // 实现AQS的对象
    private final Sync sync;

    // 构造方法，传入count（至多只能countDown()”count“次）
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    // await，调用AQS的acquireSharedInterruptibly方法
    // acquireSharedInterruptibly内部调用的是tryAcquireShared()方法，此方法是交由子类实现的
    // 所以这里调用的是CountDownLatch的tryAcquireShared()方法
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    // 带超时时间的await，原理同上
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    // 释放锁一次，或者说state-1，直到为0，调用的是AQS的releaseShared()
    // releaseShared底层调用的的tryReleaseShared()方法，此方法是交由子类实现的
    // 所以调用的是CountDownLatch.tryReleaseShared()方法
    public void countDown() {
        sync.releaseShared(1);
    }

    // 获取当前state，也就是剩余能countDown的次数
    public long getCount() {
        return sync.getCount();
    }
}
```