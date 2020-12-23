# 多线程与并发-CyclicBarrier



## 什么是CyclicBarrier？与CountDownLatch有什么异同？

CyclicBarrier的作用是在多个线程之间协同工作。比如说线程约定到达某个点，到达这个点之后的线程就都停下来，直到最后一个线程也到达了这个点之后，所有的线程才会得到释放。此外，如果设置了自定义任务，则在当前这一轮结束后，下一轮开始前会执行自定义任务

常用的场景是：多个worker线程，每个线程都在循环地做一部分的工作，并且在最后await()方法设下约定点，当最后一个线程做完了工作也到达约定点之后，所有线程的得到释放，才开始下一轮的工作

与CountDownLatch相比：

1. 功能相同但CyclicBarrier可以重复使用，CountDownLatch是一次性使用
2. CyclicBarrier支持在最后一个线程到达后执行自定义任务
3. CountDownLatch是继承AQS实现加锁、释放锁几个方法实现的，而CyclicBarrier是通过使用重入锁实现的（重入锁底层也是AQS）



## CyclicBarrier原理？

首先，CyclicBarrier不是继承AQS来实现的，而是通过ReentrantLock+Condition实现的，它的内部维护了一个内部类：Generation，它只有一个字段broken，默认为false，表示当前栅栏没有被破坏，也可以理解为所有的线程还没有执行完，如果已经执行完了，正常情况下broken还是false，但是在线程被中断、dowait()设置了超时但超时时间<=0、reset()方法被调用情况下会broken会被设置为true，表示当前栅栏被破坏，后续不会再执行，会抛出BrokenBarrierException

```java
private static class Generation {
    boolean broken = false;
}
```



### 内部结构

```java
// 当前代，可以理解为每一轮所有线程执行都会对应一个Generation，每轮执行完成后会再次new Generation
private static class Generation {
    boolean broken = false;
}
// 重入锁
private final ReentrantLock lock = new ReentrantLock();
// 条件队列
private final Condition trip = lock.newCondition();
// 参与的线程数量（每一轮执行数量）
private final int parties;
// 最后一个线程达到后执行的任务
private final Runnable barrierCommand;
// 当前代
private Generation generation = new Generation();
// 正在等待进入栅栏的线程数量
// 也可以理解为剩余等待执行任务的线程数量
private int count;
```



### 构造函数

```java
// 传入初始线容量，及最后一个线程到达需要执行的任务
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
// 传入初始线容量
public CyclicBarrier(int parties) {
    this(parties, null);
}
```



### 主要实现方法



#### dowait()

使用CyclicBarrier，我们是使用它的await()或者await(超时设置)方法，而这2个方法底层都是调用的dowait方法，所以看下dowait方法的内部实现

```java
// 返回当前线程是第几个到达的/进入栅栏的
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    // 加锁，且不受线程中断的影响
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 获取当前代
        final Generation g = generation;
        // 如果栅栏被破坏了，则抛异常
        // 栅栏被破坏原因：线程被中断、reset方法被调用、dowait设置了超时但超时时间<=0
        if (g.broken)
            throw new BrokenBarrierException();
				// 如果线程被中断则设置broken为true、唤醒所有被wait的线程、重置count值
        if (Thread.interrupted()) {
            // 下面会介绍此方法
            breakBarrier();
            throw new InterruptedException();
        }
        // count-1，表示当前线程已进入栅栏
        int index = --count;
        // 如果是最后一个线程进入栅栏
        if (index == 0) {
            // 是否已执行自定义任务runnable
            boolean ranAction = false;
            try {
                // 获取自定义任务runnable
                final Runnable command = barrierCommand;
                if (command != null)
                    // 执行任务
                    command.run();
                // 这里无论是否执行了Runnable都设置为true
                // 也就是说正常情况下都是会为true
                ranAction = true;
                // 当前这一轮都执行完了，需重置当前代
                nextGeneration();
                return 0;
            } finally {
                // false，可能是command.run()异常导致ranAction=true没有赋值
                if (!ranAction)
                    // 设置broken为true、唤醒所有被wait的线程、重置count值
                    // 下面会介绍此方法
                    breakBarrier();
            }
        }

        // 自旋
        for (;;) {
            try {
                // 没有超时，则wait
                if (!timed)
                    trip.await();
                // 否则带超时的wait
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                // 线程被中断异常
                if (g == generation && ! g.broken) {
                    // 设置broken为true、唤醒所有被wait的线程、重置count值
                    // 下面会介绍此方法
                    breakBarrier();
                    throw ie;
                } else {
                    // 不是当前代或者栅栏被破坏了，就中断线程
                    Thread.currentThread().interrupt();
                }
            }
						// 栅栏被破坏抛异常
            if (g.broken)
                throw new BrokenBarrierException();
						
            if (g != generation)
                return index;
						// 设置了超时，但是超时时间<=0
            if (timed && nanos <= 0L) {
                // 设置broken为true、唤醒所有被wait的线程、重置count值
                // 下面会介绍此方法
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        // 解锁
        lock.unlock();
    }
}
```

从dowait方法源码可以看出执行的大致流程

[![r6nktS.jpg](https://s3.ax1x.com/2020/12/23/r6nktS.jpg)](https://imgchr.com/i/r6nktS)



#### breakBarrier()

此外，dowait方法中有多处地方调用了breakBarrier()，看下breakBarrier方法实现

```java
private void breakBarrier() {
    // 当前代栅栏被破坏
    generation.broken = true;
    // 重置count为初始值
    count = parties;
    // 唤醒所有被wait()的线程
    trip.signalAll();
}
```



#### reset()

用于手动重置栅栏

```java
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 设置当前代信息
        breakBarrier();  
        // 创建一个新的代，也可以理解为开始新的一轮
        nextGeneration();
    } finally {
        lock.unlock();
    }
}
```



#### nextGeneration()

创建一个新的代

```java
private void nextGeneration() {
    // 唤醒所有被wait的线程
    trip.signalAll();
    // 重置count值
    count = parties;
    // newGeneration
    generation = new Generation();
}
```





## 使用示例

```java
// 示例
public class Test {
    public static void main(String[] args) {
        System.out.println("比赛开始前出现2个问题，开始处理问题");
        CyclicBarrier cyclicBarrier = new CyclicBarrier(2, () -> {
            System.out.println("所有问题处理完成，开始准备比赛");
        });
        for (int i = 0; i < 2; i++) {
            new Thread(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + "处理问题完成");
                    Thread.sleep(1000);
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}

输出：
比赛开始前出现2个问题，开始处理问题
Thread-0处理问题完成
Thread-1处理问题完成
所有问题处理完成，开始准备比赛
```