# 多线程与并发-Semaphore

 

## 什么是Semaphore？

Semaphore 通常我们叫它信号量， 可以用来控制同时访问特定资源的线程数量，通过协调各个线程，以保证合理的使用资源。

可以把它简单的理解成我们停车场入口立着的那个显示屏，每有一辆车进入停车场显示屏就会显示剩余车位减1，每有一辆车从停车场出去，显示屏上显示的剩余车辆就会加1，当显示屏上的剩余车位为0时，停车场入口的栏杆就不会再打开，车辆就无法进入停车场了，直到有一辆车从停车场出去为止。



## 使用场景？



主要用于那些资源有明确访问数量限制的场景，常用于限流

比如：

1. 数据库连接池，同时进行连接的线程有数量限制，连接不能超过一定的数量，当连接达到了限制数量后，后面的线程只能排队等前面的线程释放了数据库连接才能获得数据库连接

2. 停车场场景，车位数量有限，同时只能容纳多少台车，车位满了之后只有等里面的车离开停车场外面的车才可以进入



## 内部原理?

Semaphore内部继承AQS实现，支持公平/非公平机制

```java
public class Semaphore implements java.io.Serializable {
    // AQS同步对象
    private final Sync sync;

    // 继承AQS
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;
				// 设置状态数
        Sync(int permits) {
            setState(permits);
        }
				// 获取许可（剩余状态数）
        final int getPermits() {
            return getState();
        }
				
        // 非公平共享锁实现
        final int nonfairTryAcquireShared(int acquires) {
            // 自旋
            for (;;) {
                // 获取许可（剩余状态数）
                int available = getState();
                // 剩余的许可数量：剩余状态数-本次申请数量
                int remaining = available - acquires;
                // CAS操作设置剩余许可数量（state）
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
				
        // 共享释放许可实现
        protected final boolean tryReleaseShared(int releases) {
            // 自旋
            for (;;) {
                // 获取许可（剩余状态数）
                int current = getState();
                // 设置剩余许可：当前剩余许可数+本次释放许可数
                // 注：这种方式会让state可能大于初始数量
                // 比如：初始许可数量是3，acquire(1)，然后release(3)，那么state最终是5
                // 下文会有场景例子介绍
                int next = current + releases;
                // 不能小于原始剩余许可数量
                if (next < current) 
                    throw new Error("Maximum permit count exceeded");
                // CAS设置state
                if (compareAndSetState(current, next))
                    return true;
            }
        }
				
        // 指定数量减少许可数
        final void reducePermits(int reductions) {
            // 自旋
            for (;;) {
                // 当前可用许可数量
                int current = getState();
                // 计算剩余许可数量：当前可用数-本次申请数
                int next = current - reductions;
                if (next > current)
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }
				
        // 清空所有许可（set state=0）
        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    // 非公平，重写了tryAcquireShared方法
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            // 直接尝试获取许可
            return nonfairTryAcquireShared(acquires);
        }
    }

    // 公平，重写了AQS的tryAcquireShared()方法
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }
				
        // 重写了AQS的tryAcquireShared()方法
        protected int tryAcquireShared(int acquires) {
            // 先判断等待队列中是否有线程，如果有则直接return，反之获取许可
            // CAS+自旋
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

    // 构造方法，默认非公平
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    // 构造方法，指定公平/非公平
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }

    // 申请许可，线程可被中断，每次-1
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    // 申请许可，线程不可被中断，每次-1
    public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }

    // 申请许可，非公平机制，每次-1，返回是否申请成功
    public boolean tryAcquire() {
        return sync.nonfairTryAcquireShared(1) >= 0;
    }

    // 带超时时间的申请许可，每次-1，超过设置时间未申请到许可会抛出异常
    public boolean tryAcquire(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    // 释放许可，每次+1
    public void release() {
        sync.releaseShared(1);
    }

    

    // 获取剩余可用许可数量
    public int availablePermits() {
        return sync.getPermits();
    }

    // 清空许可，剩余可用许可数量设置为0
    public int drainPermits() {
        return sync.drainPermits();
    }

    // ... 忽略部分不重要方法
}
```

其实，最重要的几个方法是：nonfairTryAcquireShared、tryReleaseShared、tryAcquireShared



## 场景问题



#### 初始化10个令牌，11个线程同时各调用1次acquire方法，会发生什么?

最后一个线程阻塞，等待其他线程释放许可才能继续申请许可



#### 初始化10个令牌，一个线程重复调用11次acquire方法，会发生什么?

线程在第11次调用acquire方法时阻塞，信号量不涉及到重入，所以每调用一次acquire()方法则获取一次许可



#### 初始化1个令牌，1个线程调用一次acquire方法，然后调用两次release方法，之后另外一个线程调用acquire(2)方法，此线程能够获取到足够的令牌并继续运行吗?

可以，原因：首先，state初始化值是1，1个线程调用一次acquire方法，现在state为0，然后该线程调用两次release方法，现在state为2，所以线程能继续获取到足够的许可

这个问题可以阅读下release()方法的源码，释放许可是不限制上限的，也就是说释放后的许可数量不一定是初始容量，可能会大于，也可能会小于



#### 初始化2个令牌，一个线程调用1次release方法，然后一次性获取3个令牌，会获取到吗？

可以，原因：首先，state初始化值是2，调用1次release()方法后，state值为3，所以一次性获取3个令牌是可以获取到的