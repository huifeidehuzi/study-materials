# List
## ArrayList

####数据结构：  

- `private static final int DEFAULT_CAPACITY =10;` //默认扩容长度  
- `private static final Object[] EMPTY_ELEMENTDATA = {};` //默认初始化值  
- `private static final object[] elementData;` //存放元素的数组   
- `private static int size;` //元素数组长度  

####构造方法：  
- `public ArrayList(int initialCapacity) ` //构造指定长度的集合  
- `public ArrayList()` //构造默认空集合
- `public ArrayList(Collection<? extends E> c)` //构造指定集合

####核心方法:  
- `public int size()` //获取元素数量  
- `public boolean isEmpty()` //集合是否为空(判断元素数量)  
- `public boolean contains(Object o)` //集合是否包含某个元素(调用indexOf(o))  
- `public int indexOf(Object o)` //正序获取集合是某个元素下标  
- `public int lastIndexOf(Object o)` //倒序获取集合某个元素下标  
- `public E get(int index)` //按下标获取元素
    - `rangeCheck(index)` //检查下标，如果下标>=元素数量则抛异常
    - `elementData(index)` //获取元素，根据元素数组取下标值
- `public E set(int index, E element)` //按下标设置元素
    - `rangeCheck(index)` //检查下标，如果下标>=元素数量则抛异常
    - `E oldValue = elementData(index)` //获取下标原有元素
    - `elementData[index] = element` 设置下标元素
    - `return oldValue` //返回该下标原有元素
- `public boolean add(E e)` //添加元素
    - `ensureCapacityInternal(size + 1)` //检查长度，扩容
    - `elementData[size++] = e` //设置元素
    - `return true` //返回true
    
    