# JMM



### 主内存和工作内存

java内存模型规定了所有变量都存储在**主内存**中，每个线程都有自己的**工作内存**，工作内存中保存了被该线程使用的变量的主内存副本，线程对变量的所有操作都必须在工作内存中进行，不同线程之间无法访问对方工作内存中的内容，线程之间的变量值传递需要通过主内存来完成

![image-20200927144203381](https://s1.ax1x.com/2020/10/09/0r9r8O.png)



### 内存间交互操作

一个变量如何从主内存Copy到工作内存，如何从工作内存同步到主内存，java内存模型定义了8种操作来完成。jvm实现时必须保证8个操作都是原子性的

- lock：锁定，作用于主内存的变量，它把一个变量标识为线程独占的状态
- unlock：解锁，作用于主内存的变量，释放被线程独占的状态
- read：读取，作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中
- load：载入，作用于主内存的变量，它把read操作从主内存取到的值放入工作内存的变量副本中
- use：使用，作用于工作内存的变量，它把工作内存中的一个变量值传递给执行引擎
- assign：赋值，作用于工作内存的变量，它把一个从执行引擎接收的值赋值给工作内存的变量
- store：存储，作用于工作内存的变量，它把工作内存中一个变量值传递到主内存
- write：写入，作用于工作内存的变量，它把store操作获取到的变量值放入主内存中



如果把变量存主内存Copy到工作内存，需要按顺序执行read和load，如果把变量从工作内存同步给主内存，需要按顺序执行store和write，注意：这里只要求按顺序执行，而不是连续执行，在read和load、store和write之间可以插入其他指令

此外，java内存模型还规定了执行这8种操作必须满足的规则：

- 不允许read和load、store和write中的任何一个操作单独执行，也就是说不允许出现从工作内存同步给主内存，但是主内存不接受、从主内存读取copy给工作内存，工作内存不接受的情况
- 不允许一个线程丢弃它最近的assign操作，也就是说在工作内存中改变了变量值后必须要同步给主内存
- 不允许一个线程将没有变更过值的变量值同步给主内存
- 一个变量use和store之前，必须先load和assign
- 一个变量在同一时刻只允许一个线程lock，但是同一个线程可以多次lock一个变量，lock多少次就必须unlock多少次
- 一个变量如果没有被lock，那么就不能unlock
- 一个变量如果被lock，其他线程不允许对它进行unlock
- 一个变量进行unlock之前，必须先把变量值同步主内存



### 原子性、可见性与有序性

1. 原子性：
   1. 通过read、load、assign、use、store、write指令保证原子性，可以理解为，基本数据类型都可以保证原子性
   2. 如果有更大范围的原子性需求，则通过synchronized保证
2. 可见性：当一个线程修改某个值，另外的线程能立即得知这个修改
   1. volatile：保证变量是具有可见性的
   2. synchronized：也可以保证可见性，因为在unlock前，必须将变量先刷回主内存，而被synchronized修饰的代码同一时刻只能有1个线程执行
   3. final：被final修饰的字段在构造器中一旦被初始化完 成，并且构造器没有把“this”的引用传递出去（如果传递出去就会有this的逃逸，其他线程可能会访问到初始化一半的this），那么在其他线程中就能看见final字段的值
3. 有序性：
   1. volatile：禁止指令重排
   2. synchronized：由“一个变量在同一个时刻只允许一条线程对其进行lock操作”这条规则获得的，这个规则决定了持有同一个锁的两个同步块只能串行地进入



### 内存屏障

#### 介绍

分为`Load Barrier` 和 `Store Barrier`，也就是读屏障和写屏障



#### 作用

1. 阻止屏障两侧的指令重排序
2. 强制把写缓冲区/高速缓存中的脏数据等写回主内存，让其他线程工作内存中的数据失效



读屏障：在指令前插入【Load Barrier】可以让线程工作内存中的数据失效，所以必须强制从主内存加载最新数据到工作内存

写屏障：在指令后插入【Store Barrier】可以让缓存中的最新数据刷回主内存，让其他线程可见



#### 分类

共分为：LoadLoad、StoreStore、LoadStore、StoreLoad。实际上也就是读屏障和写屏障的组合

- **LoadLoad：**Load1; LoadLoad; Load2。在Load2及之后的指令读取变量A前，保证Load1对变量A的读取完成
- **StoreStore：**Store1; StoreStore; Store2。在Store2及之后的指令要赋值变量A前，保证Store1对变量A的赋值操作完成
- **LoadStore：**Load1; LoadStore; Store2。在Store2及之后的指令要赋值变量A前，保证Load1对变量A的读取完成
- **StoreLoad：**Store1; StoreLoad; Load2。在Load2及之后的指令要读取变量A前，保证Store1对变量A的赋值操作完成。**它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能\***



#### volatile语义中的内存屏障

1. 在volatile写操作的前面插入一个StoreStore屏障。保证了volatile写操作不会和之前的写操作从排序
2. 在volatile写操作后面插入一个StoreLoad屏障。保证了volatile的写操作不会和之后的读操作重排序
3. 在volatile读操作后面插入一个LoadLoad屏障+LoadStore屏障。保证volatile的读操作不会和后面的读操作和写操作重排序

这些操作都是为了阻止指令重排序，保证了执行顺序，不会因为重排序后的逻辑导致数据错误

例子：

```java
// 上下文是否加载完成
boolean contextReady = false;

// 在A线程中执行以下逻辑
context = loadContext();
contextReady = true;

// 在B线程执行以下逻辑
while(!contextReady ){
	sleep(200);
}
doAfterContextReady (context);
```

上面这段代码看似没有问题，但如果线程A的代码发生了指令重排，那么会变成如下：

```java
// 在A线程中执行以下逻辑
// 2行代码顺序换了位置
contextReady = true;
context = loadContext();
```

会导致A线程可能还没loadContext()执行完，B线程开始执行doAfterContextReady()方法，这样显然是不可行的



但是如果给contextReady加上volatile，那么会变成如下

```java
// 上下文是否加载完成
volatile boolean contextReady = false;

// 在A线程中执行以下逻辑
context = loadContext();
// 加上volatile后，会在这行代码前加上：StoreStore屏障
contextReady = true;
// 在这里加上：StreoLoad屏障

// 在B线程执行以下逻辑
// 这里会加上：LoadLoad屏障
while(!contextReady ){
  // 这里会加上：LoadStore屏障
	sleep(200);
}
doAfterContextReady (context);
```

从上面的例子可以看出， 加上volatile后，如果发生对contextReady变量的写操作，那么会在contextReady=true的前后加上StoreStore和StoreLoad屏障，阻止对contextReady的指令重排

