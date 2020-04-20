# HashMap

### 1.7版本

* `HashMap`
![](media/15790733129668/15791390134521.jpg)


```
结构：hash+数组+单向链表

无参构造不会分配空间 put的时候才会 目的是减少内存占用
为什么一定要是2的n次方？
为了提高效率，用位运算计算数组下标
默认长度16，如果传入的size不为2的幂次方，则向上取整最近接近2的幂次方的数
扩容：当entry数量大于等于map长度x加载因子的时候会扩容，比如默认长度是16，那么会在16x0.75=12的时候扩容
2个问题：
1.扩容死锁
2.安全隐患，可以恶意构造非常多的hashcode相同的数据，导致链表很长，会占用非常多的空间和cpu，拖垮服务器
链表是头部插入元素
扩容2倍
扩容时会创建新的hash数组，把旧的hash数组里的元素重新计算hash值和下标放进去

put(k,v)实现
public V put(K key, V value) {
        //初始化数组
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        // 若“key为null”，则将该键值对添加到table[0]
        if (key == null) 
            return putForNullKey(value); 
        //获取key的hash值
        int hash = hash(key);
        //计算下标
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            //如果key已存在，则设置新value，返回旧value
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        //将key-value添加到table[i]处
        addEntry(hash, key, value, i);
        return null;
    }
```
* `扩容死锁分析`


```
在高并发场景下，可能有多个线程同时对hashmap扩容（扩容至原有长度的2倍），比如t1、t2 两个线程同时对hashmap扩容
扩容逻辑：
void transfer(Entry[] newTable, boolean rehash) {
        //扩容的新数组长度
        int newCapacity = newTable.length;
        //循环老数组
        for (Entry<K,V> e : table) {
            //节点是否为空
            while(null != e) {
                //获取链表的元素
                Entry<K,V> next = e.next;
                //是否需要重新设置hash值
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                //获取数组下标
                int i = indexFor(e.hash, newCapacity);
                //设置当前元素的next值
                e.next = newTable[i];
                //将元素放到新数组的i下标
                newTable[i] = e;
                //继续下一个元素的循环
                e = next;
            }
        }
    }
假设现在有个key=name value是个链表，值为[李四，王五]

首先t1执行到 Entry<K,V> next = e.next;
此时e是name节点，值为李四，e.next的值为王五
正好t1允许到这里被阻塞了

t2过来将name节点复制到新数组中去了，由于1.7版本中是头部插入法，倒序复制到新数组中的链表顺序为[王五、李四]，此时王五.next=李四

此时t1又恢复继续执行代码，倒序复制到新数组中的链表顺序为[李四、王五]（这是因为t2已经将链表数据倒序过了），李四.next=王五，现在链表中2个元素的next都指向对方，会形成死循环
```


### 1.8版本
* `红黑树`

![](media/15790733129668/15791421176363.jpg)


* `HashMap`

```
结构：hash+数组+单向链表+红黑树
链表是尾部插入元素
扩容因子是0.75 

链表转红黑树
链表长度>=8转红黑树
红黑树节点<=6就会转回链表

为什么转红黑树的阈值是8？
因为符合泊松分布，当阈值为8的时候，hash碰撞的几率非常小，大概是千万分之一
hash冲突还有哪些解决办法？
hashmap里面再套一个hshmap
数组可以用链表代替吗？
不可以，数组O（1）链表是O（n）

get：如果是第一个则直接返回，如果是红黑树直接用红黑树的方法获取，否则循环数组获取
```
