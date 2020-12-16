# 多线程与并发-Synchronized



## Synchronized可以作用在哪里?

可以作用在对象、方法、静态方法（类锁），下面将一一介绍

在使用Synchronized时需要注意以下几点

- 一把锁只能同时被一个线程获取，没有获取到锁的线程则等待
- 每个实例都有自己对应的一把锁，互不影响，但是如果被锁的类型是class或被synchronized修饰的是静态方法时，那么所有线程都会共用一把锁
- synchronized修饰的方法无论是否异常都会释放锁（2个monitorexit指令）



### 对象锁

包含方法锁和代码块锁，锁的对象可以自定义也可以使用this

```java
// 对象锁实例
public class SynchronizedTest {
    // 如果将synchronized修饰在 test（）方法效果与同步代码块一致
    public void test() {
        // 同步代码块
        // 如果此处不是this，而是自定义一个对象，则锁的是自定义的对象，与当前对象无关
        synchronized (this) {
            System.out.println("我是线程：" + Thread.currentThread().getName());
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("线程：" + Thread.currentThread().getName() + "结束");
        }
    }
}

public class Test {
    public static void main(String[] args) {
        // 对同一对象加锁，互斥
        SynchronizedTest synchronizedTest = new SynchronizedTest();
        new Thread(()->{
            synchronizedTest.test();
        }).start();
        new Thread(()->{
            synchronizedTest.test();
        }).start();
    }
输出：
我是线程：Thread-0
线程：Thread-0结束
我是线程：Thread-1
线程：Thread-1结束
  

  public static void main(String[] args) {
    		// 2个对象，不互斥
        SynchronizedTest synchronizedTest1 = new SynchronizedTest();
        SynchronizedTest synchronizedTest2 = new SynchronizedTest();

        new Thread(()->{
            synchronizedTest1.test();
        }).start();
        new Thread(()->{
            synchronizedTest2.test();
        }).start();
    }
  
输出：
我是线程：Thread-0
我是线程：Thread-1
线程：Thread-0结束
线程：Thread-1结束
}
```



### 静态方法锁（类锁）

指synchronize修饰静态的方法或指定锁对象为Class对象



```java
public class SynchronizedTest {
		// 修饰静态方法，锁的是SynchronizedTest.class
    // 所有线程会共用一把锁，跟对象就没关系了
    public static synchronized void test() {
        System.out.println("我是线程：" + Thread.currentThread().getName());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("线程：" + Thread.currentThread().getName() + "结束");
    }
}

public class Test {
    public static void main(String[] args) {
        // 这里new了2个对象，结果还是互斥执行，所有锁的跟对象没关系，是锁的SynchronizedTest.class
        SynchronizedTest synchronizedTest1 = new SynchronizedTest();
        SynchronizedTest synchronizedTest2 = new SynchronizedTest();
        new Thread(()->{
            synchronizedTest1.test();
        }).start();
        new Thread(()->{
            synchronizedTest2.test();
        }).start();
    }

输出：
我是线程：Thread-0
线程：Thread-0结束
我是线程：Thread-1
线程：Thread-1结束
  
  public static void main(String[] args) {
        // 这里new了1个对象，结果还是一样
        SynchronizedTest synchronizedTest = new SynchronizedTest();
        new Thread(()->{
            synchronizedTest.test();
        }).start();
        new Thread(()->{
            synchronizedTest.test();
        }).start();
    }
输出：
我是线程：Thread-0
线程：Thread-0结束
我是线程：Thread-1
线程：Thread-1结束
}
```

**注：synchronized修饰静态方法后，方法内不能再加同步代码块了，会编译报错，synchronized(传入对象报错)，但是可以new 一个对象放进去，不会报错，但是这就失去了加锁的意义**

## 原理

> 现象、时机(内置锁this)、深入JVM看字节码(反编译看monitor指令)

举个同步代码块例子：

```java
// 准备一个测试类
public class SynchronizedTest {
    private final Object o = new Object();
    public void test() {
        synchronized (o) {
        }
    }
}
```

```basic
// 将测试类生成class文件
>javac SynchronizedTest.java
  
// 然后反编译查看class文件信息
>javap -verbose SynchronizedTest.class
```

反编译后的class文件信息

```basic
 // 这里只记录我们需要查看的相关class信息
 public void test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: getfield      #3                  // Field o:Ljava/lang/Object;
         4: dup
         5: astore_1
         6: monitorenter
         7: aload_1
         8: monitorexit
         9: goto          17
        12: astore_2
        13: aload_1
        14: monitorexit
        15: aload_2
        16: athrow
        17: return
```

从class信息中可以看到`monitorenter`和`monitorexit`指令，其中`monitorexit`有2个，1个是正常运行后退出使用的，1个是遇到异常后退出使用的，也就印证了Synchronized无论是否异常都会正常释放锁

`monitorenter`和`monitorexit`指令，会再对象在执行时，其锁计数器+1或-1，每个对象同一时间只能与一个monitor锁相关联，而同一个monitor在同一时间只能被一个线程获取，一个对象尝试获取与其相关联的monitor的所有权时，`monitorenter`指令会发生如下三种情况之一

- monitor计数器为0，这意味着当前对象monitor未被其他线程获取，那么当前线程会立马将monitor计数器+1，其他线程只能等待
- 如果monitor计数器不为0，当前线程重入了此monitor，那么锁计数器再次+1，并且随着重入的次数，锁机器数会一直累加
- 如果monitor计数器不为0，被其他线程占用，那么只能等待释放



`monitorexit`指令用来释放对monitor的所有权，当遇到`monitorexit`指令，将计数器-1，如果-1后计数器不为0，则表示当前线程是重入进来的，此时应继续持有此monitor的所有权，如果-1后计数器为0，则当前线程会释放对monitor的所有权，也就是释放锁



## JVM如何进行锁优化？

`monitorenter`和`monitorexit`指令依赖于底层操作系统的`Mutex Lock`，由于`Mutex Lock`需要将当前线程挂起然后切换用户态到内核态，这种切换的成本是十分高昂的，在大部分情况下，同步方法是运行在单线程环境下的，如果每次都调用`Mutex Lock`性能会有很大的影响，不过JDK在1.6开始对Synchronized做了大量优化,如锁粗化、锁消除、轻量级锁、偏向锁、重量级锁，自旋



### 锁的类型

JDK1.6开始，Synchronized一共有四种锁，分别是：无锁、轻量级锁、偏向锁、重量级锁，锁会随着竞争而升级，并且锁可以升级但不可以降级，这样做的目的是为了提高获取锁和释放锁的效率

> 锁膨胀方向： 无锁 → 偏向锁 → 轻量级锁 → 重量级锁 (此过程是不可逆的)

###  

### 自旋锁与自适应自旋锁

#### 自旋锁

背景：在JDK.16之前， Synchronized在多线程竞争的场景下，当一个线程获取锁时，它会阻塞其他所有竞争线程，由于挂起和唤醒线程都需要从用户态切换到内核态，这对操作系统的性能带来很大的影响，而且在很多情况下，锁的时间都是非常短的，为了这段时间取挂起和唤醒线程是不值得的，在如今多核处理器的环境下，完全可以让其他未获取到锁的线程多等待一会儿（自旋），但是不放弃CPU资源，等待持有锁的线程很快释放锁就行，在这段时间内只需要将其他线程执行一个自循环即可，这就是自旋锁的由来

自旋锁在JDK1.4就引入了，但是默认是关闭的，在1.6开始默认开启，如果锁的占用时间非常短，那么自旋的性能是非常好的，反之，如果因为长时间自旋，白白占用CPU时间片，那么性能就非常受影响，所以自旋的时间要有一定的限度，如果超过这个限度还没有获取到锁，则使用`Mutex Lock`挂起线程，自旋锁的自旋默认次数是10，可以使用`-XX:PreBlockSpin`设置

此外，除了默认10次的自旋，JDK1.6还引入“自适应自旋锁”



#### 自适应自旋锁

与自旋锁不同的是，自适应自旋锁的次数是不固定的，它会由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定的。



### 锁消除

指JVM即时编译器运行时，对一些代码层面有同步逻辑，但是不可能存在共享数据竞争时，会对锁进行消除。

锁消除的依据是逃逸分析的数据支持，也就是说：JVM会判断在同步逻辑块中的数据不会被其他线程访问到，那么JVM会将这些数据当做栈上数据对待（栈是线程私有的），认为这些数据是独立的，不需要加锁

此外，在实际开发中，开发人员是很清楚哪些地方是不需要加锁的，但是在JAVA API中很多方法都是加了同步的，那么JVM会判断如果数据不会逃逸，则会将锁消除



### 锁粗化

正常情况下，开发人员在加锁的时候是明确知道对那些地方需要加锁，且加锁的颗粒度尽可能的小，这是因为能尽快的释放锁

但是如果有连续对同个对象反复加锁和解锁，即使没有多线程竞争，频繁的互斥操作也会导致影响性能，JVM针对这种场景，会将加锁颗粒度提升至整个操作，比如new StringBuffer()，append3次，JVM会优化成加锁一次



**轻量级锁、偏向锁、重量级锁暂时不做分析**

