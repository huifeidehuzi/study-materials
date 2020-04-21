# ConcurrentHashMap

### 1.7版本

```
initialCapacity：默认长度16，指HashEntry的个数为16，而不是Segment的个数
loadFactor：加载因子，0.75

concurrencyLevel：Segment个数，这个值用来确定Segment的个数，如果传入的
concurrencyLevel不为2的幂次方，则向上取整最近接近2的幂次方的数，与hashmap类似，这个数就是Segment数组的长度

计算每个Segment平均应该放置多少个元素？
例如：initialCapacity=33，concurrencyLevel=16，那么计算方式为：
int c = initialCapacity/sszie;
此时c为2，说明每个Segment最少装2个元素，或者说Segment内的数组长度为2
但是此时初始容量为33，比32多1个，怎么处理？看关键代码
if(c * ssize < initialCapacity) c++;
如果每个Segment装的元素个数 * Segment数量 < 初始容量，就+1,c=3
那么问题又来了，3不是2的幂次方，怎么办呢？看关键代码
if(cap<c) cap<<= 1 
取最接近2的幂次方的数

sszie：Segment数组的长度,取最接近2的幂次方的数，与hashmap类似
sshift：用来计算Segment数组的下标，2的幂次方，比如初始容量是16，那么sshift=4,因为16=2的4次方
segmentShift：32-sshift，用来计算Segment数组的下标,用32来减，是因为int=32位
segmentMask：ssize-1，用来计算Segment数组的下标
cap：Segment内的数组长度，默认为2

注：
1.Segment的长度和Segment内数组的长度均为2的幂次方
2.在ConcurrentHashMap中put的value不能为空，会抛NPE异常

Segment{
    this.loadFactor = lf;//负载因子 0.75
    this.threshold = threshold;//阈值
    this.table = tab;//主干数组即HashEntry数组
    HashEntry {
        key k,
        Value v
    }
}

计算Segment数组的下标
int j = (hash<<<segmentShift) & segmentMask
取hash值高4位


```

### 1.8版本


```
private transient volatile int sizeCtl; //互斥变量 小于0时表示有其他线程对hashmap进行扩容
```

```
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        //一个死循环，目的，并发情况下，也可以保障安全添加成功
        //原理：cas算法的循环比较，直至成功
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                //第一次添加，先初始化node数组
                //如果有其他线程扩容，会yeild()
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //计算出table[i]无节点，创建节点
                //casTabAt : 底层使用Unsafe.compareAndSwapObject 原子操作table[i]位置，如果为null，则添加新建的node节点，跳出循环，反之，再循环进入执行添加操作
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   
            }
            else if ((fh = f.hash) == MOVED)
                 //如果当前处于拓展状态，返回拓展后的tab，然后再进入循环执行添加操作
                tab = helpTransfer(tab, f);
            else {
                //链表中或红黑树中追加节点
                V oldVal = null;
                //使用synchronized 对 f 对象加锁， 这个f = tabAt(tab, i = (n - 1) & hash) ：table[i] 的node对象，并发环境保证线程操作安全
               //此处注意： 这里没有ReentrantLock，因为jdk1.8对synchronized 做了优化，其执行性能已经跟ReentrantLock不相上下。
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        //链表上追加节点
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //红黑树上追加节点
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
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
                    //节点数大于临界值，转换成红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
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