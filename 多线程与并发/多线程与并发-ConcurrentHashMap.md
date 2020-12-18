# 多线程与并发-ConcurrentHashMap



## ConcurrentHashMap是什么？put方法原理知道吗？

下面分别介绍1.7版本和1.8版本的ConcurrentHashMap



### ConcurrentHashMap - JDK 1.7

JDK1.5-1.7采用的是分段锁机制来实现的，可以理解为ConcurrentHashMap中有个长度为16的数组（Segment），每个数组节点（Segment）装了一个HashMap，所以每次put数据只需要知道hash后的key下标在哪个数组节点，然后对该节点加锁即可



#### 数据结构

[![rJIe5d.jpg](https://s3.ax1x.com/2020/12/18/rJIe5d.jpg)](https://imgchr.com/i/rJIe5d)

从上图可以看出，整个ConcurrentHashMap被分为16个Segment，数组存储在每个Segment中，每个Segment通过继承ReentrantLock来进行加锁，这样每次只需要锁住key所在的Segment即可

**Segment是数组+链表实现的**



#### 初始化参数

`concurrencyLevel`：Segment的数量，默认16个，初始化后不能再更改，如果传入的
concurrencyLevel不为2的幂次方，则向上取整最近接近2的幂次方的数

`initialCapacity`：初始容量，指整个ConcurrentHashMap的容量，需要平均分配给每个Segment，下面会介绍分配的计算方式

`loadFactor`：负载因子，默认0.75，因为Segment的数量是不能扩容的，所以加载因子是给每个Segment内部使用的



#### 分配Segment内部数量计算方式

例如：initialCapacity=33，concurrencyLevel=16，那么计算方式为：
int c = initialCapacity/concurrencyLevel;
此时c为2，说明每个Segment最少装2个元素，或者说Segment内的数组长度为2
但是此时初始容量为33，比32多1个，怎么处理？看关键代码
`if(c * ssize < initialCapacity) c++;`
如果每个Segment装的元素个数 * Segment数量 < 初始容量，就+1,c=3
那么问题又来了，3不是2的幂次方，怎么办呢？看关键代码
`if(cap<c) cap<<= 1` 
取最接近2的幂次方的数



#### PUT原理

我们先看下put的入口及整体流程

```java
public V put(K key, V value) {
    // key对应的Segment
    Segment<K,V> s;
    // 不能put空值
    if (value == null)
        throw new NullPointerException();
    // hash key
    int hash = hash(key);
    // 取key对应的Segment
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject           
         (segments, (j << SSHIFT) + SBASE)) == null) 
        // 获取key对应的Segment
        s = ensureSegment(j);
    // 调用Segment的put方法
    return s.put(key, hash, value, false);
}
```

继续看下Segment的put方法

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 获取对应Segment的Lock
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        // Segment内的数组，用来存储数据，与hashMap一致
        HashEntry<K,V>[] tab = table;
        // 利用hash计算key对应的数组下标
        int index = (tab.length - 1) & hash;
        // 数组[index]的链表表头
        // 也可以理解为index下标处已存储的元素
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            // 如果表头不为空，表示index处已有元素了
            // 可能是有hash冲突，可能是相同的key插入导致
            if (e != null) {
                K k;
                // 如果是相同的key，则覆盖旧值
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                // 循环链表，获取至链表最后一个元素
                e = e.next;
            }
            else {
                // 如果不为 null，那就直接将它设置为链表表头；如果是null，初始化并设置为链表表头
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                // 如果元素数量超过segment的阈值，则扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    // 扩容下面会介绍
                    rehash(node);
                else
                    // 将元素插入至数组index处
                    // 也就是放入数组index所在的链表头部
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 解锁
        unlock();
    }
    return oldValue;
}
```

具体`扩容`逻辑来看下

```java
private void rehash(HashEntry<K,V> node) {
    // 老数组
    HashEntry<K,V>[] oldTable = table;
    // 老数组容量/长度
    int oldCapacity = oldTable.length;
    // 新数组长度=老数组容量*2
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    // 创建新数组
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    // 遍历老数组，将元素放入新数组
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            // 重新计算元素放入新数组的下标
            int idx = e.hash & sizeMask;
            // next为空表示当前节点只有一个元素，直接设置即可
            if (next == null) 
                newTable[idx] = e;
            else { 
                // 如果不为空，表示链表中元素有多个
                // lastrun == 当前entry
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                // 循环链表
                // 找到第一个后续元素的新位置都不变的节点，然后把这个节点当成头结点，直接把后续的整个链表都放入新table中
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                // 直接把整个连续的位置不变的节点组成的链表加入到新桶中
                newTable[lastIdx] = lastRun;
                // 把剩下的节点（都在搬迁链表的前端）一个个放入到新桶中。
                // 同样，每次都是加入到桶的最前端
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    // 将本次put的node加入新数组中
    int nodeIndex = node.hash & sizeMask;
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```





### ConcurrentHashMap - JDK 1.8

在JDK1.5-1.7，ConcurrentHashMap是通过分段锁机制实现的，其并发度受Segment数量限制，因为1.8开始，ConcurrentHashMap的实现抛弃了分段锁，而是采用CAS+synchronized+数组+链表+红黑树方式实现



#### 数据结构

[![rYeIIO.jpg](https://s3.ax1x.com/2020/12/18/rYeIIO.jpg)](https://imgchr.com/i/rYeIIO)



从上图可以看出，结构与HashMap一致，我们来看下JDK1.8对ConcurrentHashMap的实现



#### 初始化

```java
// 无参构造啥也不干
public ConcurrentHashMap() {
}

// 指定容量构造
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```

通过提供初始容量，计算了 sizeCtl，sizeCtl = 【 (1.5 * initialCapacity + 1)，然后向上取最近的 2 的 n 次方】。如 initialCapacity 为 10，那么得到 sizeCtl 为 16，如果 initialCapacity 为 11，得到 sizeCtl 为 32



#### PUT原理

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 不允许put空key和value，1.7版本允许put空key，但是不允许put空value
    if (key == null || value == null) throw new NullPointerException();
    // 计算hash值
    int hash = spread(key.hashCode());
    // 用于记录相应链表的长度
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果数组"空"，进行数组初始化
        if (tab == null || (n = tab.length) == 0)
            // 初始化数组，后续会介绍
            tab = initTable();
        // 找到hash值对应的数组下标节点
        // i = (n - 1) & hash 就是计算下标
        // 如果节点为空，则put
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 一次CAS操作，如果CAS成功则直接到方法尾部了
            // 如果失败则继续循环table
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                  
        }
        // 扩容操作
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            // 进入此判断，表示对应下标处已经有元素了
            // 且f 是链表的头节点
            V oldVal = null;
            // 对该节点加锁
            synchronized (f) {
                // 再次验证节点是否一致，因为可能有其他线程更改了
                if (tabAt(tab, i) == f) {
                    // 如果头节点hash不为空，则表示链表元素数量>1
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // key相等，value做覆盖即可
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // next==null表示是链表最后一个元素
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                // 将新的Node放到链表尾部
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        // 插入红黑树
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 判断是否到了转红黑树的阈值（8）
                if (binCount >= TREEIFY_THRESHOLD)
                    // 注：内部原理是数组长度小于64才会转红黑树，hashmap也是如此
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```



#### 初始化数组

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 如果正在有其他线程扩容，则yield，让出线程资源
        if ((sc = sizeCtl) < 0)
            Thread.yield();
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // 进入此方法，则表示抢到了锁
            // 抢锁原理：CAS，将sizeCtl设置为-1
            try {
                // 如果数组为空，则初始化数组
                if ((tab = table) == null || tab.length == 0) {
                    // 数组容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    // 将数组赋值给table，table是volatile的，所以table值会被其他线程见到（可见性）
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```



#### 转红黑树

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        // 如果数组长度<64，则对数组扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            // 数组扩容
            tryPresize(n << 1);
        // b是数组index下标处的链表头节点
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 对需要转红黑树的节点b加锁，防止其他线程转红黑树
            synchronized (b) {
                // 再次校验节点，防止其他线程转红黑树
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    // 遍历链表，转红黑树
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    // 将红黑树赋值到数组的index处
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```



## 为什么HashTable慢? 它的并发度是什么? 那么ConcurrentHashMap并发度是什么?

拿put来说，HashTable是用synchronized对整个put方法加锁，加锁对象为当前HashTable，所以性能慢，

ConcurrentHashMap在1.7版本是对单个Segment加锁，1.8版本开始是对单个entry加锁，所以性能比HashTable快



