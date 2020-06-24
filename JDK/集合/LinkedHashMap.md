# LinkedHashMap

## 结构
```
//ture：基于访问顺序，false：基础插入顺序，默认false
final boolean accessOrder;

/**
 * 内部entry，实现至hashmap的entry
 */
static class Entry<K,V> extends HashMap.Node<K,V> {
    //节点持有的pre节点和next节点
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

/**
 * 头结点
 */
transient LinkedHashMap.Entry<K,V> head;
/**
 * 尾结点
 */
transient LinkedHashMap.Entry<K,V> tail;

```

## 特性

```
linkedhashmap如何保证节点的顺序？

1.LinkedHashMap调用的是hashmap的put方法
2.重写了newNode()方法

Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        //将节点插入链尾
        linkNodeLast(p);
        return p;
    }
    
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        // 链表为空，则将p设置为头节点
        if (last == null)
            head = p;
        else {
           //将当前的last节点设置为p的前一个节点
            p.before = last;
            //将p设置为last节点的下一节点
            last.after = p;
        }
    }
    
    
2.如何基于accessOrder顺序访问？
public V get(Object key) {
        Node<K,V> e;
        //调用hashmap的getnode方法获取
        if ((e = getNode(hash(key), key)) == null)
            return null;
        //如果为ture，被访问的节点被置于双向链表尾部
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }

```