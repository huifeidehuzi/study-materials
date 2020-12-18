# 多线程与并发-LockSupport



## 什么是LockSupport

`LockSupport`是一个线程阻塞工具类，可以使用它来阻塞和唤醒线程，它是用来创建锁和其他同步类的基本线程阻塞原语，一些锁或有同步功能的工具类都使用到了它，比如AQS。

简而言之，调用LockSupport.park()时，当前线程会阻塞，直到LockSupport.unpark(Thread)时，线程才能继续执行

它与Object.wait()和notify()、notifyAll()功能类似，但是也有不同之处，其实也就是synchronized和Lock的区别，因为Object.wait()和notify()需要配合synchronized使用，Lock底层就有LockSupoort，这里就不多做介绍了





## 为什么LockSupport也是核心基础类

因为AQS借助2个类，Unsafe(提供CAS操作)和LockSupport(提供park/unpark操作)



## 写出分别通过wait/notify和LockSupport的park/unpark实现同步?

wait/notify

```java
public class SynchronizedTest {
    public synchronized void testNotify() {
        System.out.println("notify begin");
        notify();
        System.out.println("notify end");
    }

    public synchronized void testWait() {
        System.out.println("wait begin");
        try {
            wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("wait end");
    }
}

public class Test {
    public static void main(String[] args) {
        SynchronizedTest test = new SynchronizedTest();
        new Thread(() -> {
            test.testWait();
        }).start();

        new Thread(() -> {
            test.testNotify();
        }).start();
    }
}

输出：
wait begin
notify begin
notify end
wait end
```



park/unpark

```java
public class LockSupportTest {
    public void testPart() {
        System.out.println("park begin:" + Thread.currentThread().getName());
        LockSupport.park();
        System.out.println("park end:" + Thread.currentThread().getName());
    }

    public void testUnPark(Thread thread) {
        thread = thread == null ? Thread.currentThread() : thread;
        System.out.println("unpark begin:" + thread.getName());
        LockSupport.unpark(thread);
        System.out.println("unpark end:" + thread.getName());
    }
}

public class Test {
    public static void main(String[] args) {
        LockSupportTest lockSupportTest = new LockSupportTest();
        Thread thread = new Thread(()->{
            lockSupportTest.testPart();
        });
        thread.start();
      
        new Thread(()->{
            lockSupportTest.testUnPark(thread);
        }).start();
    }
}
输出：
park begin:Thread-2
unpark begin:Thread-2
unpark end:Thread-2
park end:Thread-2
```

此外，线程中断也能代替unpark()

```java
public class LockSupportTest {
    public void testPart() {
        System.out.println("park begin:" + Thread.currentThread().getName());
        LockSupport.park();
        System.out.println("park end:" + Thread.currentThread().getName());
    }


    public void testUnPark(Thread thread) {
        thread = thread == null ? Thread.currentThread() : thread;
        System.out.println("unpark begin:" + thread.getName());
        // LockSupport.unpark(thread);
        // 这里替换为中断
        thread.interrupt();
        System.out.println("unpark end:" + thread.getName());
    }
}

public class Test {
    public static void main(String[] args) {
        LockSupportTest lockSupportTest = new LockSupportTest();
        Thread thread = new Thread(()->{
            lockSupportTest.testPart();
        });
        thread.start();

        new Thread(()->{
            lockSupportTest.testUnPark(thread);
        }).start();
    }
}
输出：
park begin:Thread-2
unpark begin:Thread-2
unpark end:Thread-2
park end:Thread-2
```



## LockSupport.park()会释放锁资源吗? 那么Condition.await()呢?

不会，它只负责阻塞当前线程，Condition.await()会释放资源，它底层是调用的park()，只不过会再park()之前/之后做一些逻辑，比如释放当前线程占用的锁，也就是将state设置为0

```java
// AbstractQueuedSynchronizer.wait()
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    // 在这一步释放当前线程占用的锁，内部调用tryRelease()方法，将state设置为0
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 是否在同步队列中，不在则挂起当前线程
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```



## 实现原理？

```java
public class LockSupport {
    // 不能被初始化
    private LockSupport() {} 
   
    // Hotspot implementation via intrinsics API
    private static final sun.misc.Unsafe UNSAFE;
    // 下面四个字段表示内存中的偏移地址
    private static final long parkBlockerOffset;
    private static final long SEED;
    private static final long PROBE;
    private static final long SECONDARY;
    static {
        try {
            // 分别获取偏移地址
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> tk = Thread.class;
            parkBlockerOffset = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("parkBlocker"));
            SEED = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSeed"));
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
            SECONDARY = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
        } catch (Exception ex) { throw new Error(ex); }
    }

}
```

这些都没什么好说的，直接来看核心方法吧

```java
public class LockSupport {
  	// 阻塞当前线程，不会释放CPU资源，直到调用unpark()放会释放
    // 此外，如果线程被中断、设置的时间到了（time>0且isAbsolute==true）也会释放
  	public static void park() {
        UNSAFE.park(false, 0L);
    }
  
  // park,设置blocker，可以理解为当前挂起线程的是哪个
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        // 设置blocker
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        // 如果park被唤醒，则将设置blocker为null
        setBlocker(t, null);
    }
  
  	// 挂起当前线程，超过nanos纳秒自动唤醒
    public static void parkNanos(Object blocker, long nanos) {
        if (nanos > 0) {
            Thread t = Thread.currentThread();
            setBlocker(t, blocker);
            UNSAFE.park(false, nanos);
            setBlocker(t, null);
        }
    }
  
  	// 挂起当前线程，超过deadline时间就会自动唤醒
    public static void parkUntil(Object blocker, long deadline) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(true, deadline);
        setBlocker(t, null);
    }


    // 释放指定线程
    // 如果直接调用unpark（不先park当前线程），不会报错
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
}
```



## Thread.sleep()和LockSupport.park()的区别？

1. 功能都一样，都是阻塞当前线程，且不会释放资源
2. sleep只能等自己醒来，park可以被唤醒
3. sleep需要捕获中断异常（声明sleep的地方需要try-catch），park不需要
4. park比sleep更灵活

