# AtomicInteger


```

原子操作的integer，线程安全
//unsafe，原子操作底层类
private static final Unsafe unsafe = Unsafe.getUnsafe();
//偏移量
private static final long valueOffset;
//初始化偏移量
static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}
//值，volatile修饰保证内存可见性
private volatile int value;

//get
public final int get() {
    return value;
}
//set
public final void set(int newValue) {
    value = newValue;
}
//set值且返回新值
public final int getAndSet(int newValue) {
    //this指当前对线，valueOffset指当前对象的偏移量，通过this+偏移量找到内存中对应的值，从而设置新值，下同
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}
//比较并交换值
public final boolean compareAndSet(int expect, int update) {
    //比较期望的值（expect），先去主内存中拿到当前值，和传入的期望值比较，如果一致则修改为新值（update）
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
//在原有值基础上+1并返回当前最新值
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
//在原有值基础上-1并返回最新值
public final int getAndDecrement() {
    return unsafe.getAndAddInt(this, valueOffset, -1);
}
//在原有值基础上+传入的值并返回当前最新值
public final int getAndAdd(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta);
}
//在原有值基础上+1后再+1并返回当前最新值，注：返回的值是+1后再+1的值，get()获取的仍然是+1的值
//这个方法的目的是解决在多线程环境下i++的问题
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
//在原有值基础上-后再-1并返回当前最新值，注：返回的值是-1后再-1的值，get()获取的仍然是-1的值
//这个方法的目的是解决在多线程环境下i--的问题
public final int decrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
}
//在原有值基础上+delta后再+delta并返回当前最新值，注：返回的值是+1后再+delta的值，get()获取的仍然是+delta的值
//这个方法的目的是解决在多线程环境下i++的问题
public final int addAndGet(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}
//更新值并返回旧值
public final int getAndUpdate(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        //获取当前值
        prev = get();
        //更新值
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next)); //比较交换
    return prev;
}
//更新值并返回新值
public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return next;
}

```