# LinkedList
```
有序集合，链表，查询慢，插入快，线程不安全
```
- `数据结构`
    - `transient int size = 0` //元素数量
    - `transient Node<E> first` //首节点
    - `transient Node<E> last` //尾节点
    - `Node<E>` //元素节点，内部类
        - ```
          E item;
          Node<E> next; //持有的下一个元素节点 
          Node<E> prev; //持有的前一个元素节点
          
          Node(Node<E> prev, E element, Node<E> next) { //构造
            this.item = element;
            this.next = next;
            this.prev = prev;
          }
          ```
- `构造方法`:
    - `public LinkedList()` //构造一个空集合
    - `public LinkedList(Collection<? extends E> c)` //指定集合构造一个集合，调用addAll(c)
    
- `核心方法`
    - `private void linkFirst(E e)` //插入首节点元素，私有方法，内部调用
        - ```
          final Node<E> f = first; //获取首节点
          final Node<E> newNode = new Node<>(null, e, f); //构建新节点，下一节点持有首节点
          first = newNode; //设置首节点
          if (f == null) //首节点为空
            last = newNode; //设置新节点为尾节点
          else
            f.prev = newNode; //持有前一个节为新的首节点
          size++; //元素数量+1
          modCount++; //修改集合数量+1
          ```
    - `void linkLast(E e)` //插入尾节点元素，内部调用
        - ```
            final Node<E> l = last; //获取尾节点
            final Node<E> newNode = new Node<>(l, e, null); //构建新建，前一节点持有尾节点
            last = newNode;
            if (l == null)
                first = newNode;
            else
                l.next = newNode;
            size++;
            modCount++;
          ```