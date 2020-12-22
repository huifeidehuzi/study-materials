# 多线程与并发-FutureTask



## FutureTask是什么？用来解决什么问题？

可以看做一个异步执行Task或者任务，它实现Future接口，实现了基础功能，比如get和cancel方法，用来封装Callable和Runnable，它也可以作为一个任务提交给线程池执行，此外，我们还可以继承它用来自定义一些逻辑，如重写done、set、setException方法

主要用来解决：

1. 多任务计算（异步获取执行结果）
2. 解决耗时任务重复计算的问题（使用本地缓存（如map）+ FutureTask）



## FutureTask类结构？

[![r0BIzV.jpg](https://s3.ax1x.com/2020/12/21/r0BIzV.jpg)](https://imgchr.com/i/r0BIzV)

可以看到，FutureTask实现了RunnableFuture，而RunnableFuture继承了Runnable和Future，所以FutureTask可以作为当做一个Runnable传入Thread执行、也可以作为一个Future执行Callable来获得Callable的返回结果

```java
// 1.执行Callable并获取返回结果
FutureTask futureTask = new FutureTask(() -> "demo");
// 2.当做Runnable参数交给Thread执行
new Thread(futureTask).start();
```



## FutureTask的线程安全是由什么保证的?

CAS



## FutureTask原理？

FutureTask是实现Future的，先看下Future的结构

```java
public interface Future<V> {

 		// 取消任务，如果任务已完成或已取消，或由于某些原因不能被取消，会返回false
    // 如果任务还未执行，则返回true，且任务不会被执行
    // 如果任务在执行中，若入参mayInterruptIfRunning为true，则立即中断执行任务的执行并返回true
    // 如果入参mayInterruptIfRunning为false，则会返回true，但是不会中断执行任务的线程
    boolean cancel(boolean mayInterruptIfRunning);

    // 判断任务是否已取消完成，如果任务在结束前被取消则返回true，否则返回false
    boolean isCancelled();

    // 判断任务是否执行完成
    // 注意：任务执行过程中发生异常、被取消也属于任务已完成，也会返回true
    boolean isDone();
	
    // 获取任务执行结果，如果任务没执行完，则一直等到执行完成为止
    // 如果任务执行过程中发生异常，则会抛出ExecutionException
    // 如果任务被取消，则抛出CancellationException
    // 如果阻塞等待获取结果过程中，线程被中断则抛出InterruptedException
    V get() throws InterruptedException, ExecutionException;
		
    // 带超时时间的get，同上，如果阻塞获取结果超时则抛出TimeoutException
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

下面看下FutureTask的实现

```java
public class FutureTask<V> implements RunnableFuture<V> {
    /* 
     * 任务状态及状态流转
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

    // 内部持有的callable任务，运行完毕后置为null
    // 此参数通过构造方法入参传进来，类型可以是Runnable和Callable
    private Callable<V> callable;
    // 从get()中返回的结果或抛出的异常
    // 如果run的时候异常，则设置异常信息，否则设置任务返回结果
    private Object outcome; 
    // 允许callable的线程
    private volatile Thread runner;
    // 使用Treiber栈保存等待线程
    private volatile WaitNode waiters;

    // 构造函数，传入callable任务，callable不能为空
    // 初始状态为NEW
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;
    }

    // 构造函数，传入runnable任务和执行完runnable后返回的rensult
    // 初始状态为NEW
    public FutureTask(Runnable runnable, V result) {
        // 底层还是将runnbale包装成callable了
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;
    }
    	
    // 是否已取消完成
    public boolean isCancelled() {
        return state >= CANCELLED;
    }
    
    // 是否已完成
    public boolean isDone() {
        return state != NEW;
    }
    
    // 取消任务
    public boolean cancel(boolean mayInterruptIfRunning) {
        // 状态==NEW 且 通过CAS 将状态更改为INTERRUPTING或者CANCELLED
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // 如果为true，则中断线程
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            // 移除并唤醒所有等待线程
            finishCompletion();
        }
        return true;
    }

    // 阻塞获取结果
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        // 如果当前任务状态在允许中或者还未允许，则阻塞等待结果
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        // 获取结果
        return report(s);
    }
  	// 获取结果
  	private V report(int s) throws ExecutionException {
        // 获取结果
        Object x = outcome;
        // 当前任务状态为完成
        if (s == NORMAL)
            // 返回结果
            return (V)x;
        // 当前任务状态已经取消
        if (s >= CANCELLED) 
            // 抛出异常
            throw new CancellationException();
        // 抛出异常
        throw new ExecutionException((Throwable)x);
    }

    // 带超时时间的阻塞获取结果
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }

    // 任务执行完成后被调用的方法，交给子类自定义扩展实现
    protected void done() { }

    // CAS设置任务结果
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }

    // CAS设置任务异常
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }
		
    // 核心方法run
    public void run() {
        // 未运行的新任务，将运行线程替换为当前线程
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            // 待运行的任务
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                // 任务返回结果
                V result;
                // 热门无是否已执行
                boolean ran;
                try {
                    // 调用callable.call方法
                    result = c.call();
                    // 任务已运行
                    ran = true;
                } catch (Throwable ex) {
                    // 异常，则设置任务异常信息
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    // 否则，设置任务返回结果
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

    

    /**
     * Simple linked list nodes to record waiting threads in a Treiber
     * stack.  See other classes such as Phaser and SynchronousQueue
     * for more detailed explanation.
     */
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }

    // 任务执行后，唤醒等待线程
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        // //唤醒等待线程
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // 移除引用，帮助gc
                    q = next;
                }
                break;
            }
        }
        // 自定义扩展
        done();
        // 任务置空
        callable = null;   
    }

    // 带超时时间，等待任务执行完成
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        // 自旋
        for (;;) {
            // 如果线程已被中断，则移除WaitNode，并抛出中断异常InterruptedException
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            // 如果当前任务状态已完成（包含已执行完成/异常结束/已取消完成）
            if (s > COMPLETING) {
                if (q != null)
                    // 置空等待节点的线程
                    q.thread = null;
                return s;
            }
            // 如果当前任务正在运行中，则让出线程资源
            else if (s == COMPLETING)
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long stateOffset;
    private static final long runnerOffset;
    private static final long waitersOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = FutureTask.class;
            stateOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("state"));
            runnerOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("runner"));
            waitersOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

}
```

