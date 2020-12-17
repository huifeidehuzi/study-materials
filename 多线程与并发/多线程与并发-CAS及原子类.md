# 多线程与并发-CAS及原子类



## 线程安全的实现方法有哪些?

- 互斥同步: synchronized 和 Lock
- 非阻塞同步: CAS, Atomic原子类
- 无同步方案: 栈封闭，Thread Local，可重入代码



## 什么是CAS?

全称：compare-and-set，比较并交换，是一条CPU原子指令，其原理是：CAS需要2个值，分别是旧值和新值，先比较旧值有没有发生变量，如果有，则不交换新值，否则交换新值



## CAS会有哪些问题?

### ABA问题

如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时则会发现它的值没有发生变化，但是实际上却变化了。

解决办法：带上时间戳或者版本号，每次更新值的同时版本号+1或者带上时间戳，那么A->B->A就会变成1A->2B->3A（版本号方式）

JDK1.5开始，JUC.Atomic包下提供了AtomicStampedReference来解决ABA问题，他是通过pair存储值和版本号实现的

AtomicMarkableReference也可以解决，它是通过一个boolean mark字段标记值是否被修改



### 循环时间长开销大

自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。



### 只能保证一个变量的原子操作

当对一个变量进行原子操作时可以用CAS，但是对多个变量操作时，CAS就没法保证原子性了，这种情况下，或许可以将多个变量放到一个对象内，然后对这个对象进行CAS操作，比方说用AtomicReference，这也是JDK1.5 JUC.Atomic包下提供的，





## AtomicInteger的原理?



### AtomicInteger中常用方法

```java
public final int get()：获取当前的值
public final int getAndSet(int newValue)：获取当前的值，并设置新的值
public final int getAndIncrement()：获取当前的值，并自增
public final int getAndDecrement()：获取当前的值，并自减
public final int getAndAdd(int delta)：获取当前的值，并加上预期的值
void lazySet(int newValue): 最终会设置成newValue,使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

相比Integer，AtomicInteger使用起来太方便了，比如我们以前对Integer++，应该这么写

```java
private volatile int count = 0;
// 若要线程安全执行执行 count++，需要加锁
public synchronized void increment() {
   count++;
}
public int getCount() {
   return count;
}
```

使用AtomicInteger后

```java
private AtomicInteger count = new AtomicInteger();
public void increment() {
    count.incrementAndGet();
}
// 使用 AtomicInteger 后，不需要加锁，也可以实现线程安全
public int getCount() {
    return count.get();
}
```



### 实现解析

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // value在内存中对应的偏移量
    private static final long valueOffset;
    static {
        try {
            // 用于获取value字段相对当前对象的“起始地址”的偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
	
    // 存放值的value
    private volatile int value;
		
    // 获取值
    public final int get() {
        return value;
    }

   	// 获取当前值，并设置新值
    public final int getAndSet(int newValue) {
        // 调用unsafe的方法，通过偏移量和当前AtomicInteger对象获取内存中的旧值，然后比较旧值
        // 如果一致则替换新值
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

    // 同上，但是不返回当前值
    // expect:旧值  update:想替换的新值
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
   	
    // 同上，旧值+1
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    // 同上，旧值+1
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }
}
```

**我们可以看到，value被volatile修饰，然后调用unsafe的CAS方法实现，需要注意2个点**

1. **volatile：用volatile修饰value，是为了保证线程间的可见性**
2. **CAS：CAS操作保证原子性**



其实，除了AtomicInteger，还有很多原子操作类，比如**AtomicBoolean、AtomicInteger、AtomicLong**，它们的原理都一样，所以这里只拿AtomicInteger举例



## AtomicStampedReference是什么？怎么解决的ABA问题？

AtomicStampedReference是原子且带版本号的操作类，它内部维护了一个对象引用和一个版本号

```java
public class AtomicStampedReference<V> {
	  
    // pair维护了对象引用和版本号
    private static class Pair<T> {
        // 对象引用
        final T reference; 
        // 版本号
        final int stamp;
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }
		
    // 持有Pair对象，操作都是基于此对象
    private volatile Pair<V> pair;

   	 
		/**
      * expectedReference ：更新之前的原始值
      * newReference : 将要更新的新值
      * expectedStamp : 期待更新的标志版本
      * newStamp : 将要更新的标志版本
      */
     
    public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            // 旧值 == 当前值
            expectedReference == current.reference &&
            // 旧版本号 == 当前版本号
            expectedStamp == current.stamp &&
            // 新值 == 当前值
            ((newReference == current.reference &&
              // 新版本号 == 当前版本号
              // 以上四个判断，表示数据没有更新过，直接返回即可
              newStamp == current.stamp) ||
             // 否则直接CAS操作更新
             casPair(current, Pair.of(newReference, newStamp)));
    }
 
		private boolean casPair(Pair<V> cmp, Pair<V> val) {
        // 调用Unsafe的compareAndSwapObject()方法CAS更新pair的引用为新引用
        return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
    }
}
```