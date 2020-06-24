# HashSet


```
//内部实现调用hashmap保证不重复
private transient HashMap<E,Object> map;
//每次默认set值
private static final Object PRESENT = new Object();

//构造方法，默认容量16
public HashSet() {
    map = new HashMap<>();
}
//构造方法，传入hashmap初始容量
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}
//构造方法，传入hashmap初始容量及加载因子
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}
//hashmap迭代器
public Iterator<E> iterator() {
    return map.keySet().iterator();
}
//hashmap size
public int size() {
    return map.size();
}
//set值
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
//removew
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}

```