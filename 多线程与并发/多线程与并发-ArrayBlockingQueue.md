# 多线程与并发-ArrayBlockingQueue



## 什么是ArrayBlockingQueue，适用场景？

线程安全的阻塞队列，底层由数组+重入锁实现

适用于多线程环境下共享数据存储/获取的场景，比如生产者/消费者



## 结构

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    // 存储队列元素的数组
    final Object[] items;
    // 拿数据的索引，用于take，poll，peek，remove方法
    int takeIndex;
    // 放数据的索引，用于put，offer，add方法
    int putIndex;
    // 队列元素数量
    int count;
    // 重入锁保证线程安全
    final ReentrantLock lock;
    // “队列不为空”条件对象，由lock创建
    // 获取元素，元素数量=0时wait、put元素后signal
    private final Condition notEmpty;
    // ”队列未放满“条件对象，由lock创建
    // 获取元素后signal，put元素时满了就wait
    private final Condition notFull;
}
```



## 实现原理

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
		
    // put元素
    private void enqueue(E x) {
        // 获取元素数组
        final Object[] items = this.items;
        // 指定下标存储元素
        items[putIndex] = x;
        // 如果数组满了，put下标置为0
        if (++putIndex == items.length)
            putIndex = 0;
        // 元素数量+1
        count++;
        // 唤醒当前等待的线程获取元素
        notEmpty.signal();
    }

    // 获取元素
    private E dequeue() {
        // 获取当前元素数组
        final Object[] items = this.items;
        // 获取元素
        E x = (E) items[takeIndex];
        // 置为null，帮助GC
        items[takeIndex] = null;
        // 如果数组为空，则takeIndex置为0
        if (++takeIndex == items.length)
            takeIndex = 0;
        // 元素数量-1
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        // 唤醒当前等待put元素的线程，继续put元素
        notFull.signal();
        return x;
    }

    // 构造方法，传入数组容量
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    // 构造方法，传入数组容量，是否公平锁，默认非公平
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }

    // 构造方法，传入数量容量，是否非公平，要存储的列表
    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        this(capacity, fair);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int i = 0;
            try {
                for (E e : c) {
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i;
            putIndex = (i == capacity) ? 0 : i;
        } finally {
            lock.unlock();
        }
    }

    // put元素，调用父类的add方法，父类的add方法调用子类的offer实现
    // 实现在下面offer方法
    public boolean add(E e) {
        return super.add(e);
    }

    // put元素
    public boolean offer(E e) {
        checkNotNull(e);
        // 加锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
             // 如果数组满了，则不能继续put
            if (count == items.length)
                return false;
            else {
                // put元素
                enqueue(e);
                return true;
            }
        } finally {
            // 解锁
            lock.unlock();
        }
    }

    // put元素，如果数组满了，则一直wait
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        // 加锁，优先响应中断，如果线程被中断，则抛出InterruptedException，不能继续put元素
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            // 如果数组满了，则wait，等待take唤醒
            while (count == items.length)
                notFull.await();
            // put元素
            enqueue(e);
        } finally {
            // 解锁
            lock.unlock();
        }
    }

    // 带超时时间的put元素
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        checkNotNull(e);
        long nanos = unit.toNanos(timeout);
        // 加锁，优先响应中断，如果线程被中断，则抛出InterruptedException，不能继续put元素
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            // 如果数组满了且超时时间为0，直接return false
            while (count == items.length) {
                if (nanos <= 0)
                    return false;
                // 否则等待指定的时间
                nanos = notFull.awaitNanos(nanos);
            }
            // put元素
            enqueue(e);
            return true;
        } finally {
            // 解锁
            lock.unlock();
        }
    }
    // 获取元素
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
		
    // 获取元素
    public E take() throws InterruptedException {
        // 加锁，优先响应中断，如果线程被中断，则抛出InterruptedException，不能继续take元素
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            // 数组为空，则wait，等待put唤醒
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
		
    // 带超时时间的获取元素
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
}
```