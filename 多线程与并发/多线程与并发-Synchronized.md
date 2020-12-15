# 多线程与并发-Synchronized



## Synchronized可以作用在哪里?

可以作用在对象、方法、静态方法（类锁），下面将一一介绍

在使用Synchronized时需要注意以下几点

- 一把锁只能同时被一个线程获取，没有获取到锁的线程则等待
- 每个实例都有自己对应的一把锁，互不影响，但是如果被锁的类型是class或被synchronized修饰的是静态方法时，那么所有线程都会共用一把锁
- synchronized修饰的方法无论是否异常都会释放锁



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