# CopyOnWriteArrayList


```
线程安全的arraylist
//使用重入锁保证线程安全
final transient ReentrantLock lock = new ReentrantLock();
//装数据的数组，这里用vlatile修饰，保证内存可见性
private transient volatile Object[] array;
//默认构造方法，初始化空数组
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
//get方法，通过下标获取数据
public E get(int index) {
    return get(getArray(), index);
}
//set方法
public E set(int index, E element) {
    //获取锁
    final ReentrantLock lock = this.lock; 
    //加锁
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);
        //注意。如果设置的值不等于之前存放的值，则将当前数组内的数据copy一份
        //至新数组，同时将新值插入对应下标内，copy动作对性能有损耗
        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            //如果相同，则不用设置新值，这里重新setarry是为了保证外部的非volatile变量的happen-before
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        //解锁
        lock.unlock();
    }
}

//add方法
public boolean add(E e) {
    //获取锁，加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //copy一份新数组
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        //解锁
        lock.unlock();
    }
}

//指定下标插入值
public void add(int index, E element) {
    //获取锁，加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
       //获取index之前需要移除的数据量，分段移除
        Object[] newElements;
        int numMoved = len - index;
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            //新数组：老数组容量+1
            newElements = new Object[len + 1];
            //将index+1至最后一个元素向前移动一格
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        newElements[index] = element;
        setArray(newElements);
    } finally {
        //解锁
        lock.unlock();
    }
}

```