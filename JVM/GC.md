# GC


* `常见GC算法`


```
怎么找到不被引用的对象（垃圾）？
reference count 引用计数 
缺点：N个对象互相持有引用且这N个对象没有被其他对象引用，那这N个对象都算是垃圾，但是引用计数器没法标记这N个对象是垃圾

Root Searching 根可达
Gc roots
哪些才算是根？
JVM stacks 
Native method stack 本地方法栈
runtime constant poll 运行时常量池
static references in method area 静态的引用
Clazz 类

算法：
1.Mark-Sweep 标记清除 
标记垃圾，清除掉，会产生碎片

2.copying 拷贝
分半回收，会造成内存浪费

3.Mark Compact 标记压缩
标记垃圾，清除掉，压缩剩余空间，好处是剩余的空间是连续的，效率略低于copy


分代算法
年轻代 YGC或Minor GC，当Old区不足时
将存活对象copy到survivor，然后将剩余垃圾对象全部回收，如果某些对应一直回收不了，会放入老年代，如果老年代放不下了会产生Full GC
注：不同的GC，放入老年代的计数次数不一样，比如G1是15次，CMS是6，ZGC则没有，因为他不分代
可通过-XX:MaxTenuringThreshold参数设置
到底对象多大会直接进入老年代？
设置XX:PretenureSizeThreshold=大小（字节数），默认为0，不管什么对应先往eden区放，放不下才会放到老年代

多数情况下，在Full GC时，会同时进行YGC

Serial：单线程copy算法，会停止用户线程(STW)，启用单线程去回收垃圾，搭配Serial Old使用
ParNew：多线程拷贝算法，会停止用户线程(STW)，启用多线程去回收垃圾，搭配CMS使用
Parallel Scavenge：多线程拷贝算法，会停止用户线程(STW)，启用多线程去回收垃圾，搭配Parallel Old使用，1.8版本默认

老年代 FGC或Major GC，当Eden区不足时
Serial Old：单线程标记清除/压缩算法，会停止用户线程(STW)，启用单线程去回收垃圾，搭配Serial使用
Parallel Old：多线程拷贝算法，会停止用户线程(STW)，启用多线程去回收垃圾，搭配Parallel Scavenge使用。1.8版本默认
CMS：三色标记算法，并发标记清除，STW(时间特别短)只找根，并发标记的时候顺着根去标记垃圾对象，与用户线程一起工作，搭配ParNew使用。流程：先初始标记（initMark），然后并发标记，过程中可能会有标记失误，此时会重新标记一次（重新标记会STW,停止用户线程的运行，不然又会造成标记失误，重新标记就没有意义了），把标记失误的地方修正，最后并发回收垃圾

不分代
G1：分区回收，STW,三色标记+SATB
跨区引用：通过collection set 集合保存被引用对象的指针，回收扫描的时候直接拿出指针找到这个对象，看这个对象有没有被引用即可。

ZGC：
Shenandoah：

总结：

new一个对象，看能不能先分配到栈，如果分配不了，则分配到Eden区，如果对象特别大，则直接进入到老年代
```
* `一个对象从分配到回收的流程`


![](media/15876133380962/15876252123637.jpg)

* `堆内存分区`


![](media/15876133380962/15876240609561.jpg)

* `垃圾回收器`

![](media/15876133380962/15876227506053.jpg)

* `调优`

```
TOP
%cpu 占用cpu
%MEM 占用内存
COMMAND 应用
PID  进程ID

jps：列出系统中关于java的进程
jstack：打印线程栈，主要对排查检测死锁有帮助，里面打印了每个线程的状态，调用的方法链等等
jinfo：jvm的一些信息, Supported versions are 25.151-b12
jstat -gc 进程id 间隔打印时间：Gc的一些占用信息，比如年轻代占用多少，老年代占用多少，如jstat -gc 12345 5000 就是每隔5秒打印一次1245进程的gc信息
jstat -gcnew 程id 间隔打印时间：打印新生代垃圾回收信息
jstat -gcnewcapacity 程id 间隔打印时间：打印新生代内存统计信息
jstat -gcold 程id 间隔打印时间：打印老年代垃圾回收信息
jstat -gcoldcapacity 程id 间隔打印时间：打印老年代内存统计信息

参数详解：
S0C：第一个幸存区的大小
S1C：第二个幸存区的大小
S0U：第一个幸存区的使用大小
S1U：第二个幸存区的使用大小
EC：伊甸园区的大小
EU：伊甸园区的使用大小
OC：老年代大小
OU：老年代使用大小
MC：方法区大小
MU：方法区使用大小
CCSC:压缩类空间大小
CCSU:压缩类空间使用大小
YGC：年轻代垃圾回收次数
YGCT：年轻代垃圾回收消耗时间
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间、


jmap：会暂定应用程序，dump堆快照文件、尽量在oom的机器上使用
arthas：https://alibaba.github.io/arthas/install-detail.html

java -XX:+PrintCommandLineFlags -version 查看当前JDK配置的GC
java -XX:+PrintGC 打印GC的过程日志

1.吞吐量
2.响应时间：stw越短，响应时间越好

首选要确定追求吞吐量还是响应时间？或者是在一定的响应时间内的吞吐量
什么是调优？
1.根据需求进行jvm规划和预调优
2.优化jvm运行环境，如：卡顿，响应时间长，吞吐量下降
3.解决jvm运行过程中出现的问题，如oom

先确定场景，再调优

-Xms  设置初始 Java 堆大小
-Xmx  设置最大 Java 堆大小
-Xss  设置 Java 线程堆栈大小
-XX:NewSize 年轻代大小
-XX:OldSize 老年代小

为什么有时候 xms 和 xmx设置的一样大？
避免不不要的扩容和收缩，这2个动作都是要消耗cpu资源的

jmap：分析内存
jmap -histo pid | head -20  打印前20条堆内存信息
jmap -dump:format=b;file=xxxx pid  转储堆内存（也可以用arthas的heapdump命令）
jmap直接在服务器执行会导致用户线程停止，所以是有风险的，可以用几种方式解决
1.多节点部署环境下，停掉有问题的节点，对服务不产生影响
2.设定了参数HeapDump，OOM的时候自动转储日志（-XX:HeapDumpOnOutOfMemoryError，XX:HeapDumpPath="堆快照保存路径.hprof"）
3.在线定位
4.在测试环境中还原场景
5.jhat分析(jhat -j-mx512M xxx.dump)或者java自带的JVVM工具分析
6.OQL：对堆快照文件按条件查询某些堆信息

```


* `arthas` 阿里开源的一款诊断工具


```
启动：
curl -O https://alibaba.github.io/arthas/arthas-boot.jar
java -jar arthas-boot.jar

基础命令：
help——查看命令帮助信息
cat——打印文件内容，和linux里的cat命令类似
echo–打印参数，和linux里的echo命令类似
grep——匹配查找，和linux里的grep命令类似
tee——复制标准输入到标准输出和指定的文件，和linux里的tee命令类
pwd——返回当前的工作目录，和linux命令类似
cls——清空当前屏幕区域
session——查看当前会话的信息
reset——重置增强类，将被 Arthas 增强过的类全部还原，Arthas 服务端关闭时会重置所有增强过的类
version——输出当前目标 Java 进程所加载的 Arthas 版本号
history——打印命令历史
quit——退出当前 Arthas 客户端，其他 Arthas 客户端不受影响
stop——关闭 Arthas 服务端，所有 Arthas 客户端全部退出
keymap——Arthas快捷键列表及自定义快捷键

常用命令（jvm）：
dashboard——当前系统的实时数据面板  *
thread——查看当前 JVM 的线程堆栈信息  *
jvm——查看当前 JVM 的信息 *
sysprop——查看和修改JVM的系统属性
sysenv——查看JVM的环境变量
vmoption——查看和修改JVM里诊断相关的option
perfcounter——查看当前 JVM 的Perf Counter信息
logger——查看和修改logger
getstatic——查看类的静态属性
ognl——执行ognl表达式
mbean——查看 Mbean 的信息
heapdump——dump java heap, 类似jmap命令的heap dump功能  *
heapdeum --live xxxx/xxx.hprof 也会暂停用户线程，会full gc



sc 查看当前加载所有的类
sm 查看所有的方法
trace 方法内部调用路径，并输出方法路径上的每个节点上耗时

jad：反编译
jad 类的全路径 方法名

redefine：内存直接更改代码，应急操作，比如改个判断？改个变量？，需要编译成class，替换class

```

* `设置日志`

```
-Xloggc:/xxx/xxx/xx-gc.log -XX:+UserGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M -XX:PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCCause
PrintGCDateStamps:输出GC的触发时间


```

* `三色标记`

```
黑：自己不是垃圾对象且引用的对象也标记完了，下次回收不会标记引用的其他节点了。
灰：自己不是垃圾对象但引用的对象还没标记，下次回收会继续标记引用的其他几点
白：还没标记的对象，下次回收继续标记

案例：A->B->D  黑->灰->白

第一种情况 B->D引用没了，没影响。

第二种情况 B->D引用没了,同时A->B，因为A是黑色的，下次回收轮询就不会去标记A的引用节点了，会产生漏标的问题，怎么解决？
CMS：Incrmental Update,把A标记为灰色，这样下次回收就会继续标记A的引用节点，但在并发标记下会产生漏标的问题，假设A有2个成员变量E和F ，此时E没有任何引用，垃圾回收线程此时将E标记完了，接来下标记F，与此同时用户线程将E->D，垃圾回收线程会认为A已被标记完了，将A变回黑色，那么下次扫描的时候，A就不会继续进行标记了，D就会漏标，怎么解决？
CMS会STW,重新remark一遍，解决漏标问题。
G1：SATB，通过collection set 集合保存被引用对象的指针，回收扫描的时候直接拿出指针找到这个对象，看这个对象有没有被引用即可

```