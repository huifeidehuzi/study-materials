# ThreadLocal

ThreadLocal是线程的内部存储类，可以在指定线程内存储数据。只有指定线程可以得到存储数据。

每个线程都有一个ThreadLocalMap的实例对象，并且通过ThreadLocal管理ThreadLocalMap。
```
ThreadLocal.ThreadLocalMap threadLocals = null;
```
ThreadLocalMaps是延迟构造的，因此只有在至少要放置一个条目时才创建
ThreadLocalMap初始化时创建了默认长度是16的Entry数组。通过hashCode与length位运算确定索引值i。

每个Thread都有一个ThreadLocalMap类型。相当于每个线程Thread都有一个Entry型的数组table。而一切读取过程都是通过操作这个数组table进行的。

ThreadLocalMaps维护在ThreadLocal中。

```
//entry是弱引用，使用不当会导致内存泄漏哦
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```
## get & set & remove


```
public void set(T value) {
        //当前线程
        Thread t = Thread.currentThread();
        //拿到当前线程维护的map
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
//getMap方法
ThreadLocalMap getMap(Thread t) {
      //thred中维护了一个ThreadLocalMap
      return t.threadLocals;
 }

//createMap
void createMap(Thread t, T firstValue) {
      //实例化一个新的ThreadLocalMap，并赋值给线程的成员变量threadLocals
      t.threadLocals = new ThreadLocalMap(this, firstValue);
}


public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    
    
    
    private void remove(ThreadLocal<?> key) {
       Entry[] tab = table;
       int len = tab.length;
       //计算entry数组下标
       int i = key.threadLocalHashCode & (len-1);
       for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
           if (e.get() == key) {
               //引用持有置为空，帮助GC
               e.clear();
               expungeStaleEntry(i);
               return;
           }
       }
   }
```

## 发生内存泄漏的原因


```
1.ThreadLocal实例没有被外部强引用
2.ThreadLocal实例被回收，但是在ThreadLocalMap中的V没有被任何清理机制有效清理
3.当前Thread实例一直存在，则会一直强引用着ThreadLocalMap，也就是说ThreadLocalMap也不会被GC

也就是说，如果Thread实例还在，但是ThreadLocal实例却不在了，则ThreadLocal实例作为key所关联的value无法被外部访问，却还被强引用着，因此出现了内存泄露。
```