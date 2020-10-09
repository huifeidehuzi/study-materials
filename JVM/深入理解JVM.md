## Java内存区域与内存溢出异常

### 运行时数据区

在java程序运行过程中会将内存划分为若干个不同的区域，其中有的区域随着jvm进程的启动而一直存在，有的区域随着用户线程生命周期存在和销毁，jvm管理的内存包含以下运行时数据区

1. 方法区
2. 虚拟机栈
3. 本地方法栈
4. 堆
5. 程序计数器

**其中方法区、堆是所有用户线程共享的，其他区域为线程私有的**



#### 程序计数器

程序计数器是当前线程执行字节码行号的指示器

字节码解释器工作时通过改变计数器的值来获取下一条需要执行的字节码指令，它是程序流程控制器。分支、循环、跳转、异常处理、线程恢复等基础功能都需要计数器来完成

因为jvm是多线程轮流切换，分配处理器执行时间实现的，在任何一个确定的时刻，一个处理器只会执行一条线程中的指令，因此为了线程切换后能恢复到之前执行的位置，每个线程都需要有自己的程序计数器。程序计数器是唯一一个没有OOM的区域



#### 虚拟机栈

虚拟机栈的生命周期随着线程创建和销毁，当方法被执行时，每个线程会创建一个栈帧（后续有介绍）用来存储局部变量表、操作数栈、动态链接、方法出口等信息，方法被调用到执行完毕的过程，对应一个栈帧在虚拟机栈中出入栈到出栈的过程

局部变量表存储了编译时期可知的数据类型，如8种基本类型、reference对象引用地址、returnAddress字节码指令地址

这些数据类型在局部变量表中以”局部变量槽Slot“来表示，其中double和long占用2个槽位，其余类型占用1个槽位。当调用一个方法时局部变量表的大小（槽的数量）是可以确定的，且在方法运行期间局部变量表的大小是不会被更改的

此区域如果线程请求的栈深度大于虚拟机设置的深度将抛出StackOverflowError；如果虚拟机栈可以动态扩展，当栈扩展时无法申请到足够的内存会OOM或者线程申请栈空间失败也会OOM



##### 栈帧

在同一时刻，同个线程里所有方法都同时处于执行状态，而对于执行引擎来说，在活动线程中只有位于栈定的方法才是在运行的，只有位于栈顶的栈帧才是生效的（“当前栈帧”），与栈帧关联的方法被称为“当前方法”，执行引擎允许的所有字节码只针对当前栈帧进行操作

###### 局部变量表

一组变量值的存储空间，存放方法参数和方法内的局部变量，以变量槽（slot）为最小单位，每个槽可以存储boolean、 byte、char、short、int、float、reference或returnAddress。此外double和long分别占2个槽

jvm通过索引定位局部变量表中的变量，范围为：0~最大槽数量



###### 操作数栈/操作栈

一个先进后出的栈，当一个方法刚刚开始执行的时，该方法的操作数栈是空的，在方法执行过程中会有各种字节码指令往里面写入和提取内容，也就是入栈和出栈。

比如：计算2个int变量值的总和，iadd指令就需要从操作数栈中取出（出栈）最近2个int变量的值，然后执行运算，将结果压入（入栈）栈中



#### 本地方法栈

本地方法栈存储的是native方法，与虚拟机栈一样，也会有StackOverflowError和OOM的情况



#### 堆

堆在jvm启动时候创建，主要存放对象实例，在G1之前可以划分为新生代、s1、s2、老年代等区域，G1出现后划分为N个同等大小的区域块，当然，如果申请内存失败时也会出现OOM



#### 方法区

与堆一样，是线程共享的区域，主要存储被jvm加载的类型信息、常量、运行时常量池、静态变量等数据，在JDK1.8之前，方法区称为”永久代“，是直接放在堆上的，在JDK1.8级之后，这块区域被放在本地内存中，称为”元空间“，如果申请内存不足会出现OOM



#### 运行时常量池

运行时常量池是方法区的一部分，具有”动态性“的特性，可以理解为在程序运行期间产生的常量

好处：

1. 避免频繁创建带来的性能消耗
2. 实现了对象的共享
3. 节省内存，比如多个值相同的变量，指向的都是堆中同一块区域
4. 提高比较效率，==比equals快

主要存储

1. 字面量：被final修饰的变量、字符串、基本包装类型、静态变量等
2. 符号引用：
   1. 类和接口的全路径名
   2. 字段名和描述符（如public）
   3. 方法名和描述符（如public）



#### 直接内存

直接内存不属于运行时数据区的一部分，也会有OOM的问题，可通过-XX:MaxDirectMemorySize参数配置直接内存大小

所以在配置jvm堆大小的时候要考虑直接内存的大小，避免因忽略直接内存大小导致各区域的大小总和>物理内存



### HotSpot虚拟机对象探秘

#### 对象的创建

当jvm执行到字节码new指令时，首先要检查该指令的参数是否能在常量池中定位到一个类的符号引用，如User user = new User(),则需要检查 User类或者user变量或者User()是否在常量池中存在，并且需要检查此符号引用对应的类是否已经被加载、解析、初始化过，如果没有则需要先执行类加载

接下来需要为对象分配内存空间，对象所需空间在类加载完后可以确定（//todo 待补充确定逻辑），为对象分配空间实际上是把一块确定大小的内存空间从堆中划出来，这里涉及到2个分配的点

1. 指针碰撞：假设堆中的内存是连续、规整的，左边是空闲的右边是已分配的，中间由一个指针作为分界点，那么分配内存只需要将指针往左边移动即可
2. 空闲列表：如果堆中的内存不是连续、规整的，空间和已分配的内存相互交错，那么就没法做指针碰撞的分配，此时虚拟机需要维护一个列表来记录哪些区域的内存块是可用的，在给对象分配内存时从列表中找出一块足够大的内存区域划分给对象即可

这2种方式的选择是由jvm根据堆内存是否规整决定的，而堆内存是否规整是根据jvm使用GC回收算法决定的，比如Serial、ParNew是压缩算法，那么jvm会采用指针碰撞方式分配内存，像CMS这种是标记清除算法会采用空闲列表方式分配内存

分配内存也同样带来线程不安全的问题，因为创建对象是非常频繁的操作，比如对象A正在分配内存，指针往左移动过程中对象B也来申请内存，此时对象A的分配还未结束，指针还在移动过程中，所以对象B也会在指针原位置进行指针移动，解决这个问题有2种方式：

1. 分配动作进行同步处理，jvm是使用的cas+失败重试的机制保证
2. 使用TLAB，但是TLAB缺点也有，比如本地线程申请了100KB，但是对象只有80KB，就浪费了20KB



内存分配完成后，jvm必须将分配到的内存空间（不包含对象头）都初始化为零值，如果使用了TLAB，则在TLAB分配时进行这个操作。此操作保证了对象的实例字段在java中可以不赋初始值就可以使用

接下来jvm还需要设置对象头信息，如分代年龄、hashcode、元数据、锁标识等等

最后，执行对象的init方法（构造方法），一个对象就构建完成了



#### 对象的访问

创建对象后，后续是通过虚拟机栈存储的reference对象引用地址对象访问，目前主流的方式2种

1. 句柄访问：堆中会划分出一块内存来存储句柄（句柄池），reference存储的就是对象的句柄地址，该句柄中包含了对象实例数据与类型数据的具体地址信息

   ![image-20200915151634516](https://imgchr.com/i/0r93CT)

2. 指针访问：reference存储的就是对象的地址，减少了一次指针定位的开销，jvm主要使用这种方式访问对象

   ![image-20200915151646892](https://github.com/huifeidehuzi/images/blob/master/data/image-20200915151646892.png)



**对象实例数据：对象中各个实例字段的数据**

**对象类型数据：对象的类型、父类、实现的接口、方法等**



### OOM各种场景示例

#### 堆溢出

保证GC Roots到对象之间有可达路径，不被GC回收，当对象达到一定数量后，容量>=堆的总容量，申请内存失败则会产生内存溢出异常

```java
/**
 * 下面是jvm设置的初始分配堆大小、最大运行堆大小、生成OOM快照
 * VM Args:-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class OOMTest {

    static class OOMObject { }

    public static void main(String[] args) { List<OOMObject> list = new ArrayList<>();
        while (true) {
            list.add(new OOMObject());
        }
    }
}

// 输出
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid58688.hprof ...
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:261)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:235)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:227)
	at java.util.ArrayList.add(ArrayList.java:458)
	at com.taoche.OOMTest.main(OOMTest.java:15)
Heap dump file created [28397327 bytes in 0.194 secs]
```

后续我们可以通过比如JVVM工具分析生成的hprof文件，首先确定导致OOM产生的对象，然后分清楚到底是内存泄漏还是内存溢出

如果是内存泄漏，需要确认引用链，到底是因为什么哪些引用导致一直不被回收，反之如果是内存溢出，那么需要确认JVM的堆参数是否设置合理，也可以通过加配置或者优化代码解决



#### 虚拟机栈和本地方法栈溢出

由于HotSpot不区分虚拟机栈和本地方法栈，所以-Xoss参数（本地方法栈大小设置）没用，只能通过-Xss参数来设置栈的大小。关于栈的异常，有2种

1. 请求的深度大于jvm允许的深度，抛出StackOverflowError
2. 如果jvm允许动态扩展内存，当无法申请到足够内存时，抛出OOM（HotSpot没有此异常）

下面列举出2个列子，分别验证调用深度和创建大量局部变量是否会导致StackOverflowError



##### 调用深度验证

```java
/**
 * 配置栈大小 128k
 * 注意：不同版本不同操作系统，栈的最小值可能有出入，具体数值可以在不同操作系统下尝试
 * 本人mac os操作系统默认最小值为：160kb
 * 小于160Kb会给出提示：The stack size specified is too small, Specify at least 160k
 * VM Args:-Xss128k
 */
public class StackTest {
    public void test(){
        test();
    }
    public static void main(String[] args) {
        StackTest test = new StackTest();
        test.test();
    }
}

// 输出
Exception in thread "main" java.lang.StackOverflowError
	at com.taoche.StackTest.test(StackTest.java:10)
	at com.taoche.StackTest.test(StackTest.java:10)
```



##### 创建大量局部变量验证

```java
/**
 * 配置栈大小 128k
 * 注意：不同版本不同操作系统，栈的最小值可能有出入，具体数值可以在不同操作系统下尝试
 * 本人mac os操作系统默认最小值为：160kb
 * 小于160Kb会给出提示：The stack size specified is too small, Specify at least 160k
 * VM Args:-Xss128k
 */
public class StackTest {

    public void test() {
        int a0,a1,a2,a3,a4,a5,a6,a7,a8,a9,a10,a11,a12,a13,a14,a15,a16.....至1000 = 0;

    }
    public static void main(String[] args) {
        StackTest test = new StackTest();
        test.test();
    }
}

// 输出
Exception in thread "main" java.lang.StackOverflowError
	at com.taoche.StackTest.main(StackTest.java:17)
```

2个实现得出结论，当新的栈帧无法申请足够的内存时，jvm抛出的都是StackOverflowError



#### 方法区溢出

```java
/**
 * 设置方法区最小值为6m，最大值为6m
 * JDK1.8 VM Args:-XX:MetaspaceSize=6M -XX:MaxMetaspaceSize=6M
 */
public class FinalTest {
    public static void main(String[] args) {
        Set<String> set = new HashSet<>();
        short i = 0;
        while (true) {
            set.add(String.valueOf(i++).intern());
        }
    }
}

// 输出
Error occurred during initialization of VM
OutOfMemoryError: Metaspace
```



### 垃圾收集器和分配策略

#### 引用计数器

在对象中添加一个引用计数器，每次被引用计数器+1，引用失效计数器-1，当计数器值为0的时候表示当前对象没有被引用，为垃圾对象，可被GC回收

但是当2个或者以上的对象互相持有对方引用时，计数器就失效了，会一直不会被GC回收，最终导致OOM（泄漏）



#### 可达性分析

通过一系列”GC Roots“根对象作为起始点，从这些节点往下搜索，搜索过程中走过的路径为”引用链“，如果某个对象与"GC Roots"没有引用链，则此对象为GC可回收对象

![image-20200917113825958](https://github.com/huifeidehuzi/images/blob/master/data/image-20200917113825958.png)

如图：就算object5、6、7之间持有引用，但是与”GC Roots”没有引用链相连，那么这3个对象为可回收对象

**GC Roots：**

1. 虚拟机栈的局部变量表中的对象，如方法内的局部变量，入参等
2. 方法区中类的静态属性对象，如 final User
3. 方法区中的常量，如常量池中的对象
4. 本地方法栈中的对象
5. jvm内部的引用，如基本数据类型的class对象，NPE、OOM异常、类加载器等
6. 所有被synchronized持有的对象
7. 反映Java虚拟机内部情况的JM XBean、JVM TI中注册的回调、本地代码缓存等



#### 引用

JDK1.2之后，java对引用类型进行了扩容，分别为：强引用、软引用、弱引用、虚引用



##### 强引用

普遍存在的引用，如：User user = new User()，任何情况下，只要引用在，GC永远不会回收

```java
Object o1 = new Object();
Object o2 = o1;
System.out.println(o1); // java.lang.Object@23a5fd2
System.out.println(o2); // java.lang.Object@23a5fd2
o1=null;  // 置空，去掉引用
System.gc(); // gc回收
System.out.println(o1); // null，被回收了 
System.out.println(o2); //java.lang.Object@23a5fd2
```

##### 软引用

被软引用关联的对象，内存不足时才会放入回收范围内进行第二次回收

```java
Object o1 = new Object();
SoftReference<Object> softReference = new SoftReference<>(o1);
System.out.println(o1); //java.lang.Object@23a5fd2
System.out.println(softReference.get()); //java.lang.Object@23a5fd2
o1=null;  // 置空，去掉引用
System.gc(); // gc回收
System.out.println(o1); //null，被回收了
System.out.println(softReference.get()); //java.lang.Object@23a5fd2
```

##### 弱引用

被弱引用关联的对象，只能存活到下一次GC之前，也就是说GC时无论该对象是否存活都会被回收

```java
Object o1 = new Object();
WeakReference<Object> softReference = new WeakReference<>(o1);
System.out.println(o1); //java.lang.Object@23a5fd2
System.out.println(softReference.get()); //java.lang.Object@23a5fd2
o1=null; // 置空，去掉引用
System.gc(); // gc回收
System.out.println(o1); //null，被回收了
System.out.println(softReference.get()); //null，被回收了
```



#### 方法区GC

方法区的GC主要回收两部分内容：

1. 废物常量
2. 不再使用的类型



#### 垃圾收集算法

##### 标记-清除

标记-清除（Mark-Sweep）分为”标记“和”清除“2个阶段，首先标记出需要回收的对象，标记完成后，回收掉被标记的对象

主要缺点有2个：

1. 执行效率不稳定：如果有大量的对象需要回收，会导致STW时间过长，效率很低
2. 空间碎片化：产生大量不连续的内存，后续有大对象需要分配的时候，不得不再次GC

用于老年代



##### 标记-复制

前言：Appel式回收，将新生代划分为eden和两块Survivor，每次分配内存只使用eden和其中一块Survivor（S1），在垃圾搜集时，将存活的对象复制到另外一块Survivor（S2）中，然后直接清理掉eden和S1空间，新生代eden和S1、S2的比例默认是8：1：1

标记-复制（Mark-Copy）也是分为”标记“和”复制“两部分，首先将需要回收的对象标记，将存活对象复制到S2，然后清理调eden和S1即可

标记-复制也有缺点：如果存活的对象较多，就需要进行多次的复制，降低性能

用于新生代



##### 标记-整理

标记-整理算法与标记-清除一样，但是后续标记-整理不是直接对垃圾对象进行回收，而是将存活对象往内存空间一端移动，然后清理掉存活对象内存空间以外的所有内存

标记-整理算法一般用于老年代，但是老年代每次回收都有大量的存活对象，大量的移动这些对象和更新对象的内存地址是一项非常耗时、耗费性能的工作，而且在操作过程中是需要STW的

**如果关注吞吐量可以使用标记-整理算法的回收器，如果关注延迟低可以使用标记-整理的回收器**



#### 并发的可达性分析

可达性分析理论上要求全过程基于一个能保障一致性的快照中才能进行分析，也就是说全过程都必须停止用户线程，不然用户线程还是会继续产生新对象或者更改对象间的引用导致漏标或者标记错误，当堆中的对象越多，通过GC Roots往下查找引用链的次数越多，时间越长，用户线程停止的时间越长

如果想降低用户线程停止的时间，需要先清楚为什么必须在一致性的快照中才能进行对GC Roots的遍历，我们可以通过三色标记来了解

三色标记：按照”是否访问过“这个条件将对象标记为三种颜色的其中一种

- 白色：表示对象还没有被垃圾回收器遍历过，在可达性分析开始阶段，所有对象都为白色，可达性分析结束，如果对象仍然是白色，表示此对象为不可达对象，就可以被垃圾回收器回收

- 灰色：表示对象被垃圾回收器访问过，但最少还有一个引用没被访问过
- 黑色：表示对象被垃圾回收器访问过，且对象的所有引用被扫描过，黑色对象代表存活，不需要被垃圾回收器回收，如果其他对象引用了该对象，不需要重新再扫描一遍所有引用。

如果在三色标记过程中，用户线程是停止的，那是没问题的，反之用户线程与标记线程同步进行，那么可能会造成漏标或者标记错误的问题

解决该问题，有2种方式：

1. 增量更新（IU）：当黑色对象引用某个白色对象时，将该引用记录下来，等标记结束，通过记录下来的引用重新扫描一次，可以理解为黑色对象引用某个白色对象时，就需要变更为灰色对象。**CMS使用**
2. 原始快照（SATB）：当灰色对象删除引用的某个白色对象的关系时，将该删除引用记录下来，等标记结束，通过记录下来的引用再扫描一次。**G1使用**



#### 垃圾回收器

##### 年轻代回收器

###### Serial

基于标记-复制算法单线程回收垃圾，在回收垃圾时，需要STW停止用户线程直到垃圾回收完成

![image-20200921093551639](https://github.com/huifeidehuzi/images/blob/master/data/image-20200921093527662.png)

###### ParNew

基于标记-复制算法多线程回收垃圾，ParNew实际上是Serial的多线程版本，它也会STW，只是在回收垃圾时GC线程是多个

除了Serial外，目前只有ParNew是搭配CMS使用的

ParNew默认开启的GC线程是CPU核心数，可通过用-XX：ParallelGCThreads参数调整GC线程数量

![image-20200921093608927](https://github.com/huifeidehuzi/images/blob/master/data/image-20200921093608927.png)



###### Parallel Scavenge

基于标记-复制算法多线程回收垃圾

Parallel Scavenge的目标是尽可能达到一个可控制的吞吐量，也就是：用户代码时间与GC+用户代码时间的比值，比如用户代码与GC时间总值为10分钟，GC时间为1分钟，那么吞吐量为90%

它提供了2个用于控制吞吐量的参数

- -XX：MaxGCPauseMillis：大于0的毫秒数，回收器尽可能的保证GC的时间不超过设置的时间，可以理解为STW停顿的时间。不过不要认为这个值设置的越小GC的速度就越快，GC停顿的时间是牺牲新生代内存大小和吞吐量换取的，当新生代内存变小，GC时间肯定越短，但是这也会导致GC的频率越高
- -XX：GCTimeRatio：1-100直接的整数值，也就是GC时间占总时间的比例，计算公式为：1/(1+设定的值)，比如设定为20，那么GC的时间占比为4%。默认值为99，比例为1%

此外，它还提供了+UseAdaptiveSizePolicy参数，如果设置为开启，那么就不需要我们来手动调节新生代的大小、eden和S1、S2的比例、晋升老年代的对象大小等参数，它会自动根据运行情况收集的监控信息调节最合适的停顿时间（吞吐量）

![image-20200921093608927](https://github.com/huifeidehuzi/images/blob/master/data/image-20200921093647578.png)



##### 老年代回收器

###### Serial Old

基于标记-整理算法的单线程回收器，GC时也会STW

![image-20200921093527662](https://github.com/huifeidehuzi/images/blob/master/data/image-20200921093527662.png)

###### Parallel Old

基于标记-整理算法的多线程回收器，注重吞吐量可以使用Parallel Old+Parallel Scavenge的搭配使用

![image-20200921093647578](https://github.com/huifeidehuzi/images/blob/master/data/image-20200921093647578.png)

###### CMS

基于标记-清除算法的多线程回收期，它的目标是使STW的停顿时间越短越好，它的回收流程大概分为四个步骤

1. 初始标记：标记GC Roots关联的对象，注意：仅仅是标记关联的对象，不会做可达性分析，所以速度是非常快的，此操作会STW
2. 并发标记：通过初始标记出的关联对象往下做引用链搜索，此操作与用户线程并发执行
3. 重新标记：因为并发标记是与用户线程并发执行的，所以在标记期间用户线程是可能会修改或新增某些对象间的引用关系的，所以需要重新标记一次，此操作会STW
4. 并发清除：并发清除掉垃圾对象，由于不需要移动内存，所以可以与用户线程一起执行



![image-20200921105742475](https://github.com/huifeidehuzi/images/blob/master/data/image-20200921105742475.png)

CMS也有它的缺点，分别如下：

1. **导致应用程序性能降低：**由于是并发回收，所以对处理器资源非常敏感，在CPU核心数4个及以上时，CMS并发回收线程数不会超过CPU核心数的25%，但是CPU核心数4个以上下时，GC回收线程数量会占用一半的CPU核心数，也就是说应用程序的性能将会大幅降低。**GC线程数计算公式：（CPU核心数+3）/4**
2. **无法处理浮动垃圾：**由于并发标记和并发清除是与用户线程并发执行的，所以在标记或清除时用户线程可能会产生新的垃圾对象，这些垃圾对象是在并发标记或者清除后 CMS无法回收的对象，只能在下一次GC时清除掉，这一部分对象称为**”浮动垃圾“**。另外因为CMS是并发标记/清除垃圾对象的，所以CMS需要预留足够的空间给用户线程来存储新产生的对象，**JDK6开始，CMS GC的阈值调整为92%**，如果在GC期间新产生的对象大小>预留空间大小，jvm会停止所有用户线程，启动Serial Old来回收垃圾。**-XX:CMSInitiatingOccu-pancyFraction调整GC阈值**
3. **产生大量内存碎片：**由于CMS是基于**“标记-清除”**来回收垃圾对象，这意味着会产生大量的内存碎片，当大对象被进来的时候，没有足够连续内存空间来分配的话就不得不再次FULL GC一次，为了解决这个问题，CMS提供了**”-XX:+UseCMS-CompactAtFullCollection“** 用于上述情况下开启合并内存碎片整理的操作（默认为开启，JDK9废除），这个操作是停止所有用户线程的，这样一来停顿的时间就长了，所以CMS又提供了另一个参数**”-XX:CM SFullGCsBefore- Compaction“** 用来在CMS执行过N次（由设定的值决定）不整理内存碎片的FULL GC后，下一次FULL GC之前先整理内存碎片的操作。



###### Garbage First（G1）

G1在本质上就区别于其他垃圾回收器，它是基于Region的堆内存布局，将连续的堆划分为N个大小相等的独立区域（Region），每个Region可以根据需要被划分为eden或survivor或old，回收器根据不同角色的Region使用不同的策略处理

此外，Region中还有一个特殊的**Humongous**区域专门存储大对象，只要大小超过Region的一半就是大对象，**Region的大小可以通过-XX:G1HeapRegionSize设定，取值范围为：1M~32M，且应为2的N次幂**，而对于超过Region整个大小的对象则会存放在N个连续的**Humongous**中，在G1中，大多数行为将**Humongous**作为老年代的一部分来看待

G1将Region作为每次回收的最小单元，每次回收的空间是Region大小的整倍数，这样可以避免对整个堆内存做回收处理，而且G1会优先回收**"收益最大"**的Region，收益大可理解为回收这个Region所花费的时间和回收掉的内存空间大小

G1的回收步骤大致也分为如下：

1. 初始标记：标记GC Roots关联的对象，注意：仅仅是标记关联的对象，不会做可达性分析，所以速度是非常快的，此操作会STW
2. 并发标记：通过初始标记的对象做可达性分析，标记可回收的对象，此操作与用户线程并发执行
3. 最终标记：并发标记结束后，对SATB记录做可达性分析，标记可回收对象，此操作会STW
4. 筛选回收：对每个Region的最大收益进行排序，根据用户设定的停顿时间来指定回收计划，构成回收集，然后把存活的Region内的对象复制到空闲的Region中，最后回收掉垃圾对象。此操作会STW，多线程回收

![image-20200921172213842](https://github.com/huifeidehuzi/images/blob/master/data/image-20200921172213842.png)



相比CMS，G1的优点是不会产生内存碎片（G1整理来看使用的是标记-整理，局部来看，比如2个Region中对象的移动就使用了标记-复制，CMS使用的是标记-清除），G1的缺点是为了维护每个Region的记忆集导致占用堆容量的20%甚至更多，但是CMS只需要维护一份记忆集，记录老年代都新生代的引用即可





##### 搭配



![image-20200921093713794](https://github.com/huifeidehuzi/images/blob/master/data/image-20200921093713794.png)

Serial ---------------CMS和Serial Old

ParNew--------------CMS和Serial Old

Parallel Scavenge ------------- CMS 和Parallel Old

注：其中CMS----serial old，serial Old是在CMS GC期间剩余可用内存不足时导致对象分配失败后的备选方案，停止用户线程选用serial Old  GC



##### 常用命令

| 参数                   | 说明                                                         |
| ---------------------- | :----------------------------------------------------------- |
| UseSerialGC            | 使用Serial+Serial Old                                        |
| UseParNewGC            | 使用ParNew+Serial Old，JDK9后不再支持                        |
| UseConcMarkSweepGC     | 使用ParNew+CMS+Serial Old，注：当CMS GC时剩余内存不够新对象的分配，将会启用Serial Old回收老年代 |
| UseParallelGC          | 使用Parallel Scavenge+Serial Old                             |
| UseParallelOldGC       | 使用Parallel Scavenge+Parallel Old                           |
| UseG1GC                | 使用G1                                                       |
| SurvivorRatio          | eden与Survivor容量比例，默认为8，eden=8，Survivor=1，比如设为7，那么eden=7，Survivor=2。注：剩下的1是Survivor2区域的比例 |
| PretenureSizeThreshold | 新生代晋升到老年代对象的大小，如果超过设置的值，则直接进入老年代，只对Serial和ParNew有效，默认为0 |
| MaxTenuringThreshold   | 新生代晋升到老年代的年龄，每次年轻代GC存活的对象年龄会+1     |
| PrintGCDetails         | GC时和进程退出时打印GC详细日志                               |



#### 内存分配与回收示例

##### 优先分配至eden

```java
/**
 * 堆大小为20M，不允许扩展，年轻代为10M，老年代为10M，打印GC日志，eden区比例为80%
 * 版本：JDK1.8，使用CMS+ParNew
 * VM参数:-Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8  
 *       -XX:+UseConcMarkSweepGC -XX:+PrintCommandLineFlags
 */
public class Test {
    private final int _1mb = 1024 * 1024;
    public void test() {
        
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[2 * _1mb];
        allocation2 = new byte[2 * _1mb];
        allocation3 = new byte[2 * _1mb];
        // 发生minor GC
        allocation4 = new byte[4 * _1mb];
    }

    public static void main(String[] args) {
        Test test = new Test();
        test.test();
    }
}
// 输出
// 年轻代无法分配内存了
[GC (Allocation Failure) [ParNew: 6481K->1024K(9216K), 0.0174012 secs] 6481K->1529K(19456K), 0.0175088 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[GC (Allocation Failure) [ParNew: 7409K->337K(9216K), 0.0131560 secs] 7914K->8005K(19456K), 0.0131817 secs] [Times: user=0.04 sys=0.02, real=0.01 secs] 
// CMS初始标记
[GC (CMS Initial Mark) [1 CMS-initial-mark: 7667K(10240K)] 12101K(19456K), 0.0004724 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
Heap
 // 年轻代此时占用4M，也就是allocation4分配后占用的内存大小
 par new generation   total 9216K, used 4515K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  // eden占用51%，也就是allocation4分配后占用的内存大小                                             
  eden space 8192K,  51% used [0x00000007bec00000, 0x00000007bf014930, 0x00000007bf400000)
  from space 1024K,  32% used [0x00000007bf400000, 0x00000007bf454570, 0x00000007bf500000)
  to   space 1024K,   0% used [0x00000007bf500000, 0x00000007bf500000, 0x00000007bf600000)
 // 老年代占用7M，也就是allocation1、allocation2、allocation3从eden区转移过来占用的内存大小
 concurrent mark-sweep generation total 10240K, used 7667K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 3344K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 367K, capacity 388K, committed 512K, reserved 1048576K
[CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 

```

上述test()，将3个大小为2M和1个大小为4M的对象分配到堆，在执行到allocation4赋值时会产生一次minor GC，因为年轻代大小为10M，allocation4赋值时内存不够，所以会GC，GC时候发现allocation1、allocation2、allocation3无法回收，而且不能复制到S1，因为S1容量仅为1M，所以只能将allocation1、allocation2、allocation3转移到老年代

GC结束，allocation4被分配至eden，此时eden区占用内存为4M



##### 大对象直接进入老年代

大对象：需要占用大量连续内存空间的对象，典型的大对象比如有很长的字符串、大量元素的数组

大对象对jvm是非常不友好的，写代码的时候应该避免产生大对象，比如可以分批处理，因为当一个大对象分配空间时，如果空间不够就会不得不GC，不然没有足够的空间存储它。我们可以通过-XX:PretenureSizeThreshold设定大对象的阈值，如果超过这个阈值的对象则会直接分配进入老年代

```java
/**
 * 堆大小为20M，不允许扩展，年轻代为10M，老年代为10M，打印GC日志，eden区比例为80%，使用CMS，大对象阈值为    *  3M
 * 版本：JDK1.8
 * VM参数:-Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:+UseConcMarkSweepGC -XX:+PrintCommandLineFlags -XX:PretenureSizeThreshold=3145728
 */
public class Test {
    private final int _1mb = 1024 * 1024;

    public void test() {

        byte[] allocation4;
        allocation4 = new byte[4 * _1mb];
    }

    public static void main(String[] args) throws InterruptedException {
        Test test = new Test();
        test.test();
    }
}

// 输出
Heap
 // 年轻代占用约7M，本人用工具分析了下，这7M不是allocation4内存，应该是java程序启动时加载的一些系统类
 par new generation   total 9216K, used 6977K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  eden space 8192K,  85% used [0x00000007bec00000, 0x00000007bf2d04f8, 0x00000007bf400000)
  from space 1024K,   0% used [0x00000007bf400000, 0x00000007bf400000, 0x00000007bf500000)
  to   space 1024K,   0% used [0x00000007bf500000, 0x00000007bf500000, 0x00000007bf600000)
 // allocation4被分配至老年代，内存占用4M                           
 concurrent mark-sweep generation total 10240K, used 4096K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 3394K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 373K, capacity 388K, committed 512K, reserved 1048576K
```



##### 长期存活对象进入老年代

每个对象的对象头中”对象年龄“字段存储了每个对象的分代年龄，当对象经过第一次minor GC后仍然能存活下来，并且Survivor内存足够的话，该对象会被复制到Survivor，同时分代年龄+1，接下来每次minor GC该对象仍然能存活下来的话，则年代年龄每次都会+1，当分代年龄达到设置定的阈值（通过：+XX:M axTenuringThreshold参数设置，默认15）就会晋升到老年代

// 示例略略略略略略略





### JVM执行子系统



#### 类的加载时机

一个类型（类/接口等，下同）被加载到jvm内存中到卸载出内存，整个生命周期为：**加载->验证->准备->解析->初始化->使用->卸载。其中验证->准备->解析统称为“链接”**

![image-20200923151557257](https://github.com/huifeidehuzi/images/blob/master/data/image-20200923151557257.png)

​	

在初始化阶段，《java虚拟机规范》规定了有且只有六种情况必须对类进行初始化

1. 遇到new、getstatic、putstatic、invokestatic四个字节码指令时，如果类型没有进行过初始化，则需要先进行初始化（注意：new只能new此类型，不能new该类型的数组，否则不会初始化改类型）。能够生成这四条指令的场景有：

   1. new 对象时
   2. 读取或者设置一个类型的静态字段时（被final修饰或赋了初始值的除外）
   3. 调用一个类型的静态方法时

2. 使用java.lang.reflect包的方法对类型进行反射调用时，如果类型没有初始化，则需要先初始化

3. 当初始化类的时候如果其父类没有进行过初始化，则先初始化父类

4. 当jvm启动时，如果需要制定一个要执行的主类（包含main方法类），jvm会先初始化此类

5. 当一个接口定义了default方法（JDK8），如果该接口有实现类，则先初始化default方法，再初始化实现类

6. 当jdk支持动态语言，如果一个java.lang.invoke.MethodHandle实例最后的解 析结果为REF_getStatic、REF_p utStatic、REF_invokeStatic、REF_newInvokeSp ecial四种类型的方法句 柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化

   此外，如果一个接口继承了另一个接口，只有当使用到父接口中的属性/方式时才会初始化父类



#### 类的加载过程

##### 加载

加载是整个类加载过程中的一个阶段，它主要做了三件事：

- 通过一个类的全限定名来获取此类的二进制流
  - 从网络中获取
  - 从压缩包获取，比如jar，war
  - 动态生成，比如动态代理
  - 文件生成，比如jsp
- 将字节流转换为方法区的数据结构，比如类名，方法名、字段名等
- 在内存中生成一个class对象



##### 验证

###### 文件格式验证

验证字节流是否符合Class文件格式的规范，且能被当前版本的jvm处理

- 是否以魔数0xCAFEBABE开头
- 主次版本号是否在jvm接收范围之内
- 常量池的常量是否有不支持的类型
- 指向常量的各种索引值是否存在执行不存在的常量或类型



###### 元数据验证

验证字节码描述的信息进行语义分析

- 此类是否有父类（java.lang.Object除外），所有类应当有父类
- 此类的父类是否继承了不允许被继承的类（被final修饰的类）
- 此类如果不是抽象类，是否实现了其父类或接口必须实现的方法
- 此类中的字段、方法是否与父类产生矛盾，比如重写了父类的final字段



###### 字节码验证

对类的方法体进行校验分析，确保类中的方法不会有危害jvm的行为

- 保证任意时刻操作数栈的数据类型与指令代码都能配合工作
- 保证任何跳转指令都不会跳转到方法体以外的字节码指令上
- 保证方法体中的类型转换都是有效的



###### 符号引用验证

确保解析行为能正常秩序

- 符号引用中通过全限定名是否能找到对应的类
- 类、字段、方法的修饰符(private,public,protected)是否合理
- 是否存在符合方法的字段描述符及简单名称所描述的方法和字段



##### 准备

为类中定义的静态变量（被static修饰的变量）分配内存并设置变量的初始值，注意：在这个时候进行内存分配的仅包括类变量（静态变量），而实例变量（非静态变量）是随着对象实例化分配内存的

其次这里说的设置变量的初始值通常情况下是数据类型的零值，如：public static int value = 123; 那么变量value在准备阶段过后的初始值为0，而不是123，因为此时未开始执行任何java代码，而将123赋值给value是在putstatic指令被编译后存放在**类构造器方法**中，所以value的赋值是在类初始化时才会执行

**以下是各基本类型的零值**

| 数据类型  | 零值       |
| --------- | ---------- |
| int       | 0          |
| long      | 0L         |
| short     | （short）0 |
| char      | '\u0000'   |
| byte      | （byte）   |
| boolean   | false      |
| float     | 0.0f       |
| double    | 0.0d       |
| reference | null       |

上面提到的都是“通常情况下”初始值都是零值，但是如果字段属性表中存在**Constant value属性**，那么在准备节点就会执行赋值操作，比如：public static final int value = 456，准备节点就会将456赋值给value

**注：被static 和final同时修饰的变量都会有Constant value属性，比如上面的例子，value就是Constant value属性**





### 高效并发



#### Java内存模型与线程

##### 主内存和工作内存

java内存模型规定了所有变量都存储在**主内存**中，每个线程都有自己的**工作内存**，工作内存中保存了被该线程使用的变量的主内存副本，线程对变量的所有操作都必须在工作内存中进行，不同线程之间无法访问对方工作内存中的内容，线程之间的变量值传递需要通过主内存来完成

![image-20200927144203381](https://github.com/huifeidehuzi/images/blob/master/data/image-20200927144203381.png)



##### 内存间交互操作

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



##### 线程状态切换

java定义了6种线程状态，分别是：

1. 新建（New）：创建后未启动的线程
2. 运行（Runnable）：正在执行且未结束的、等待分配CPU的线程
3. 无限等待（Waiting）：
   1. 未设置超时时间的wait()
   2. 未设置超时时间的join()
   3. LockSupport的park()
4. 有限等待（Timed Waiting）：
   1. 设置了超时时间的wait()
   2. 设置了超时时间的join()
   3. sleep()
   4. LockSupport的parkNanos()
   5. LockSupport的parkUntil()
5. 阻塞（Blocked）：程序进入同步区域时会被阻塞
6. 结束（Terminated）：线程已执行结束

![image-20201009165603078](https://github.com/huifeidehuzi/images/blob/master/data/image-20201009165603078.png)

