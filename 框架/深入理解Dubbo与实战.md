

Dubbo 

## 版本

v2.6.5 release



## 总体分层

dubbo总体分为三层：biz业务层、RPC层、Remot层，如果把每一层细分，总共可以分为十层

![image-20200731212354742](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200731212354742.png)

其中Service和Config可以视为API层，提供API给开发者使用，其他层视为SPI层，提供扩展功能，



### 层介绍

| 层次命    | 作用                                                         |
| --------- | ------------------------------------------------------------ |
| Service   | 业务层，开发者具体实现的业务逻辑                             |
| Config    | 配置层，主要围绕ServiceConfig，和ReferenceConfig2个实现类，初始化配置信息，该层管理了Dubbo所有配置 |
| Proxy     | 代理层，无论生产者还是消费者，框架都会为其生成一个代理类，当调用1个接口时，代理层会自动做远程调用并返回结果，业务层完全无感知（就像调用本地方法一样） |
| Registry  | 注册层，负责服务的注册与发现，当有服务加入或下线时，注册中心都会感知并且通知给所有订阅的服务 |
| Cluster   | 集群容错层，远程调用失败时容错的策略（如是失败重试、快速失败）、负载均衡（如随机、一致性hash等）、特殊路径调用（如某个消费者只会调用某个IP的生产者） |
| Montior   | 监控层，负责监控统计调用次数和调用时间等                     |
| Protocol  | 远程调用层，封装RPC调用过程，Protocol是Invoker暴露（发布一个服务给其他服务调用）和引用（引用一个远程服务到本地）的主功能入口，它负责管理Invoker的生命周期，Invoker是Dubbo的核心模型，框架中所有模型向它靠齐或者转换成它，它可能是执行本地方法的实现，也可能是远程服务的实现，还可能是一个集群的实现 |
| Exchange  | 信息交换层，建立Rquest-Response模型，封装请求响应模式，如把同步请求转换成异步请求 |
| Transport | 网络传输层，把网络传输封装成统一的接口，如Mina和Netty虽然接口不一样，但是Dubbo封装了统一的接口，开发者也可以扩展接口添加更多传输方式 |
| Serialize | 序列化层，负责整个框架网络传输时的序列化和反序列化工作       |



## 注册中心

注册中心是核心组件之一，通过注册中心实现了分布式环境中服务间的注册和发现，作用如下：

1. 动态加入：一个服务提供者通过注册中心可以动态的把自己暴露给其他消费者
2. 动态发现：一个消费者可以动态的感知新的配置、路由、服务提供者，无需重启
3. 动态调整：支持参数的动态调整，新参数自动更新到所有相关服务
4. 统一配置：避免了本地配置导致各服务的配置不一致



### 注册模块介绍

dubbo的注册中心源码在模块dubbo-registry中，包含以下模块

| 名称                     | 介绍                                                         |
| ------------------------ | ------------------------------------------------------------ |
| dubbo-registry-api       | 包含注册中心所有API和抽象类                                  |
| dubbo-registry-zookeeper | 使用zookeeper作为实现（官方推荐使用）                        |
| dubbo-registry-redis     | 使用redis作为实现（没有长时间允许的可靠性验证，稳定是基于redis本身，阿里内部不使用） |
| dubbo-registry-defualt   | 基于内存的默认实现（不支持集群，可能会有单点故障）           |
| dubbo-registry-multicast | multicast模式的实现（服务提供者启动时会广播自己的地址，消费者通过广播订阅请求，服务提供者收到订阅请求后会广播给订阅者，不推荐在生产环境使用） |



### 工作流程

1. 服务提供者启动时，会向注册中心写入自己的元数据，同时订阅元数据配置信息
2. 消费者启动时，会想注册中心写入自己的元素数据，并订阅服务提供者、路由、配置元数据信息
3. 服务治理中心（dubbo-admin）启动时，会订阅所有消费者、服务提供者、路由、配置元数据信息
4. 当有服务提供者离开或者加入时，服务提供者目录会发生变化，变化信息会通知给消费者和治理中心
5. 当消费者发起调用时，会异步将调用、统计信息上报给监控中心（dubbo-monitor-simple）



![image-20200731221351476](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200731221351476.png)



### 数据结构

因为其他注册中心不常用，这里只介绍zookeeper

#### Zookeeper

树形结构，每个节点的类型分为持久节点、持久顺序节点、临时节点、临时顺序节点

1. 持久节点：服务注册后保证节点不丢失，重启也会存在
2. 持久顺序节点：在持久节点的基础上增加了节点先后顺序的功能
3. 临时节点：注册中心链接丢失或者session超时节点会自动移除
4. 临时顺序节点：在临时节点基础上增加了节点先后顺序的功能



duubo使用Zookeeper时，只会使用持久节点和临时节点，对创建的顺序没有要求（可能是有路由，对节点的顺序没要求），结构如下：

```java
+/ dubbo  //分组，下面有多个服务接口，分组值来自<dubbo:registy>的group属性，默认dubbo

+-- service  //服务接口
	
  // 如：dubbo://127.0.0.1:20880/com.zjf.service.UserSevice?application=provider_name&dubbo=2.2.6&interface=getUserInfo...信息
	+-- providers //服务提供者，保存URL元数据信息（ip、端口、权重、应用名等）
	
  // 如：consumer://127.0.0.1:20880/com.zjf.service.UserSevice?application=consumer_name&dubbo=2.2.6&interface=getUserInfo...信息
	+-- consumers //消费者，保存URL元数据信息（ip、端口、权重、应用名等）
	// 保存
	+-- routers //保存消费者理由策略URL元数据信息
  
	+-- configurators //保存服务提供者动态配置URL元数据信息
```

| 目录名称                    | 存储值                                                       |
| :-------------------------- | ------------------------------------------------------------ |
| /dubbo/serivce/proivders    | dubbp://ip:prot/service全路径?categry=proivder&key=value&... |
| /dubbo/serivce/comsumers    | comsumer://ip:prot/service全路径?categry=comsumer&key=value&... |
| /dubbo/serivce/routers      | condition://0.0.0.0/service全路径?categry=routers&key=value&... |
| /dubbo/serivce/configuators | override://0.0.0.0/service全路径?categry=configuators&key=value&... |

示例：/dubbo/serivce/proivders      

dubbp://192.168.0.1:8080/com.demo.userService?categry=proivder&name=userService



### 订阅/发布

因为其他注册中心不常用，这里只介绍zookeeper的实现



#### 发布的实现

上面提到过Registry注册层，dubbo在启动时会调用ZookeeperRegistry.doRegister(URL url)创建目录以及服务下线时调用ZookeeperRegistry.delete(String path)删除目录

```java
public class ZookeeperRegistry extends FailbackRegistry {
    // 默认端口
    private final static int DEFAULT_ZOOKEEPER_PORT = 2181;
    // 默认根节点
    private final static String DEFAULT_ROOT = "dubbo";
    // 根节点
    private final String root;
	// 所有服务集合
    private final Set<String> anyServices = new ConcurrentHashSet<String>();
	// zk客户端
    private final ZookeeperClient zkClient;
    
    public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter){
        // ...
        // 创建zkClient
        zkClient = zookeeperTransporter.connect(url);
        // ...
    }
    
	// 创建目录
    @Override
    protected void doRegister(URL url) {
        try {
            // url示例如下
            // dubbo://192.168.199.149:20880/com.dubbo.api.service.UserService?anyhost=true&application=producer_1&dubbo=2.6.2&generic=false&interface=com.dubbo.api.service.UserService&methods=getUserByName,getUserList&pid=11524&revision=1.0.0&side=provider&timestamp=1596290138311&version=1.0.0
            zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
    
    // 删除目录
    @Override
    protected void doUnregister(URL url) {
        try {
            zkClient.delete(toUrlPath(url));
        } catch (Throwable e) {
            throw new RpcException("Failed to unregister " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
}
```



#### 订阅的实现

通常有2种方式

1. pull：客户端定时拉取注册中心配置
2. push：注册中心主动推送给客户端

目前duubo使用的第一种方式，启动时拉取，后续接收时间重新拉取

在服务暴露时，服务端会订阅configurators监听动态配置，消费者启动时会订阅providers、routers、configurators

其中，dubbo在dubbo-remoting-zookeeper模块中山实现了zk客户端的统一封装：ZookeeperClient，它有2个实现类，分别是CuratorZookeeperClient和ZkclientZookeeperClient，用户可以在<dubbo: registry client='curator或zkclient'>使用不同的实现，如果不指定则默认使用curator



### 缓存机制

空间换时间，如果每次远程调用都需要从注册中心获取可用的服务列表会占用大量的资源和增加额外的开销，显然这是不合理的，因此注册中心实现了通用的缓存机制，在AbstractRegistry实现

消费者或dubbo-admin获取注册信息后会做本地缓存

1. 内存：保存在Properties对象
2. 磁盘：保存在用户系统磁盘.dubbo文件夹中



### 缓存的加载

dubbo在启动时，AbstractRegistry的构造方法会从本地磁盘缓存的cache文件中把注册数据读到properties对象，并加载到内存缓存中

```
public abstract class AbstractRegistry implements Registry {
    // 内存缓存
	private final Properties properties = new Properties();
    // 磁盘缓存
    private File file;
    
    public AbstractRegistry(URL url) {
        setUrl(url);
        // 初始化是否需要同步缓存
        syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
        // 获取磁盘缓存的文件命,示例如下
        // C:\Users\Mloong/.dubbo/dubbo-registry-producer_1-127.0.0.1:2181.cache
        // 用户user.home的.dubbo文件夹下，创建dubbo-registry-应用名-ip-端口.cache
        String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(Constants.APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
        File file = null;
        if (ConfigUtils.isNotEmpty(filename)) {
            file = new File(filename);
            if (!file.exists() && file.getParentFile() != null && !file.getParentFile().exists()) {
                if (!file.getParentFile().mkdirs()) {
                    throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
                }
            }
        }
        this.file = file;
        // 加载Properties对象，将磁盘缓存的数据口读取到Properties中
        loadProperties();
        // 通知各监听，里面会将数据保存在Properties中
        notify(url.getBackupUrls());
    }
}
```



### 重试

通过ZookeeperRegistr继承FailbackRegistry抽象类实现失败重试机制

FailbackRegistry中定义了一个ScheduledExecutorService线程池，默认每5秒调用retry()重试，此外，该抽象类中有五个重要的集合，如下：

| 名称                                      | 介绍                     |
| ----------------------------------------- | ------------------------ |
| ConcurrentHashSet<URL> failedRegistered   | 发起注册失败的URL集合    |
| ConcurrentHashSet<URL> failedUnregistered | 取消注册失败的URL集合    |
| ConcurrentMap> failedSubscribed           | 发起订阅失败的监听器集合 |
| ConcurrentMap> failedUnsubscribed         | 取消订阅失败的监听器集合 |
| ConcurrentMap» failedNotified             | 通知失败的URL集合        |

在retry()中会依次对这5个集合遍历重试，重试成功就从对应的集合remove

下面四个方法均由子类实现，如ZookeeperRegistr就实现了这四个方法

```
// 注册方法，由子类实现
protected abstract void doRegister(URL url);
// 取消注册方法，由子类实现
protected abstract void doUnregister(URL url);
// 订阅方法，由子类实现
protected abstract void doSubscribe(URL url, NotifyListener listener);
// 取消订阅方法，由子类实现
protected abstract void doUnsubscribe(URL url, NotifyListener listener);
```



### 设计模式

1. 模板方法：重试机制中就讲到了
2. 工厂：注册中心的由RegistryFactory接口统一提供,分别由不同的实现类创建对应的工厂

![image-20200804214158313](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200804214158313.png)





```java
@SPI("dubbo")
public interface RegistryFactory {
	// 通过配置的protocol获取对应实现类创建注册中心
    // 如dubbo.registry.protocol=zookeeper
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);
}
```



AbstractRegistryFactory实现了RegistryFactory的getRegistry(URL url)，由此实现方法提供加锁、创建注册中心、解锁等功能，代码如下

```java
public abstract class AbstractRegistryFactory implements RegistryFactory {
    // 统一的获取注册中心入口
    @Override
    public Registry getRegistry(URL url) {
        url = url.setPath(RegistryService.class.getName())
                .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
                .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
        String key = url.toServiceString();
        // 加锁，防止重复创建工厂
        LOCK.lock();
        try {
            // 缓存中存在直接返回
            Registry registry = REGISTRIES.get(key);
            if (registry != null) {
                return registry;
            }
            // 否则就创建工厂，createRegistry(url)是抽象方法，由子类实现具体的创建逻辑
            // 这里又用到了模板方法
            registry = createRegistry(url);
            if (registry == null) {
                throw new IllegalStateException("Can not create registry " + url);
            }
            // 放入缓存
            REGISTRIES.put(key, registry);
            return registry;
        } finally {
            // Release the lock
            LOCK.unlock();
        }
    }
}
```



## dubbo扩展机制

#### JAVA SPI

java spi的目的是将一些接口（功能）交由开发者实现，提供灵活的可扩展机制

java spi使用了策略模式，一个接口多种实现，具体的实现不在程序（如果是第三方框架则不是在框架的jar中）写，而是由开发者来提供具体实现，具体实现如下：

1. 定义接口及方法
2. 编写实现类
3. 在META-INF/services/目录下，创建一个接口全路径名称的文件，如com.zjf.spi.TestService文件
4. 文件内容为接口的实现类全路径命，如有多个则另起一行，如：com.zjf.spi.TestServiceImpl（此实现类由开发者自己实现，SPI会加载此实现类）
5. 通过ServiceLoader加载实现类

示例如下：

```java
public interface TestService{
	void printInfo();
}

public interface TestServiceImpl implements TestService{
	
    @Override
    public void printInfo(){
    	println("我是实现类");
    };
}

// SPI调用
public static void main(String[] args) ( 
	ServiceLoader<TestService> serviceServiceLoader =
										ServiceLoader.load(TestService.class);
	for (TestService testService : serviceServiceLoader) (
		// 获取所有的SPI实现，循环调用
		testService.printInfo();
	}
}
```



#### Dubbo SPI

相对于java spi，dubbo spi性能更好，功能更多，在dubbo官网中有一段是这么写的：

1. JDK标准的SPI会一次性实例化扩展点所有实现，没用的也会加载，如果某些扩展点非常耗时，则非常浪费资源
2. 如果加载失败，则连扩展的名称也找不到，比如某个jar包不存在，导致某个扩展点加载失败，异常会被吃掉，用户使用次扩展点时候会报一些奇怪的错，与本身的异常对应不起来
3. 新增了IOC和AOP，一个扩展可直接setter注入其他扩展，具体原理在下面文档中有介绍，此处不做详细介绍

我们拿TestService做Dubbo SPI的改造，代码如下：

1. 在META-INF/dubbo/internal/目录下新建com.zjf.spi.TestService文件
2. 文件内容为：impl=com.zjf.spi.TestServiceImpl，有多个实现另起一行即可（key=value）
3. 代码如下

```
// 新增SPI注解
// impl对应com.zjf.spi.TestService文件内的impl,可以随意取名，唯一即可
// SPI注解（）内可不加key
@SPI("impl")
public interface TestService{
	void printInfo();
}

public interface TestServiceImpl implements TestService{
	
    @Override
    public void printInfo(){
    	println("我是实现类");
    };
}

// SPI调用
public static void main(String[] args) ( 
	// 获取TestService的默认实现
	TestService testService = ExtensionLoader
						.getExtensionLoader(TestService.class)
						.getDefaultExtension();
	testService.printInfo();
	
	// 也可以通过key获取实现
	TestService testService = ExtensionLoader
						.getExtensionLoader(TestService.class)
						.getExtension("impl");
	testService.printInfo();
}
```



#### Dubbo SPI配置规范

dubbo启动时会默认扫描：META-INF/services 、META-INF/dubbo、META-INF/dubbo/internal三个目录的配置文件

| 规范名          | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| SPI配置文件路径 | META-INF/services/> META-INF/dubbo/> META-INF/dubbo/intemal/ |
| SPI配置文件名称 | 全路径类名                                                   |
| 文件内容格式    | key=value方式，多个用换行符分隔                              |



#### 扩展点的分类与缓存

Dubbo SPI可分为Class缓存，实例缓存，这2种缓存又能根据扩展类的种类分为普通扩展类，包装扩展类（Wapper类），自适应扩展类（Adaptive类）

1. Class缓存：获取扩展类时会先从缓存中取，取不到则加载配置文件，然后将Class放到缓存中
2. 实例缓存：出于性能考虑，dubbo在缓存class时也会缓存实例后的对象，获取时先从缓存中取，取不到则实例化对象，并且放到缓存中。

被缓存的Class和对象实例可以根据不同的特性分为不同的类别：

1. 普通扩展类：配置文件中的实现类
2. 包装扩展类：这种类没有具体实现，需要在构造方法中传入一个具体的实现类
3. 自适应扩展类：当一个接口有多个实现，但使用哪个实现不会写死在代码或者配置文件中，在运行的时候通过@Adaptive(参数)注解内的参数决定使用哪个实现类

| 集合                                                         | 类型                                                  |
| ------------------------------------------------------------ | ----------------------------------------------------- |
| Holder<Map<String, Class<?»> cachedClasses                   | 普通扩展类缓存，不包括自适应拓展类和Wrapper 类        |
| Set<Class<?» cachedWrapperClasses                            | Wrapper类缓存                                         |
| Class<?> cachedAdaptiveClass                                 | 自适应扩展类缓存                                      |
| ConcurrentMap<String, Holder<Object» cachedlnstances         | 扩展名与扩展对象缓存                                  |
| Holder<Object> cachedAdaptivelnstance                        | 实例化后的自适应（Adaptive 扩展对象，只能同时存在一个 |
| ConcurrentMap<Class<?>, String> cachedNames                  | 扩展类与扩展名缓存                                    |
| ConcurrentMap<Class<?>, ExtensionLoader<?» EXTENSION_LOADERS | 扩展类与对应的扩展类加载器缓存                        |
| ConcurrentMap<Class<?>, Object>EXTENSION_INSTANCES           | 扩展类与类初始化后的实例                              |
| Map<String, Activate> cachedActivates                        | 扩展名与@Adaptive的缓存                               |



#### 扩展点的特性

官网介绍：自动包装、自动加载、自适应、自动激活



##### 自动包装

将通用逻辑抽象出来，子类专注实现自己的业务即可，典型的装饰器模式

```java
// 示例
public class TestWapper implements BaseTest{
	private final BaseTest bTest;
	public TestWapper(BaseTest bTest){
        if(bTest == null){
            throw new IllegalArgumentException("bTest == null");
        }
        this.bTest = bTest;
    }
    
    public void baseInfo(){
        // 抽象逻辑 ..do something
        // 子类逻辑
        bTest.do();
    }
    
    protected void do(){
        // 子类实现
    }
}
```



##### 自动加载

除了自动包装，还可以使用setter方法设置属性值，如果某个扩展类是另一个扩展类的一个属性（字段），且拥有setter方法，那么ExtensionLoade会自动注入对应的扩展点实现，但如果这个扩展类是一个Interface且有多个实现，那么改注入哪一个实现呢？这就涉及到第三个特性---自适应



##### 自适应

在Dubbo SPI中，使用@Adaptive可以动态的通过URL中的参数来决定使用哪一个实现类

示例如下：

```
@SPI("netty")
public interface Transporter ( 

	@Adaptive{Constants SERVER_KEY, Constants.TRANSPORTER_KEY))
	Server bind(URL url, ChannelHandler handler) throws RemotingException;
)
```

@Adaptive中传入了2个参数，值分别为："server”和“transporter”，当bind方法被调用时，会动态的从url参数中获取key=server的value，如果value能匹配到某个实现类则直接使用，如果没有匹配上则继续通过第二个key=transporter获取value继续匹配，如果都没匹配上就抛出异常

这种方式只能匹配一个实现类，如果想同时匹配多个实现类，就涉及到第四个特性---自动激活

**注：此URL是com.alibaba.dubbo.common.URL，而非java.net.URL**



##### 自动激活

使用@Adaptivate，标记对应的扩展点被默认激活，还可以通过传入不同的参数设置扩展点在不同条件下被激活，主要的使用场景是某个扩展点的多个实现类需要同时启用(比如Filter扩展点)。在接下来的文档中会介绍以上几种注解



### 扩展点注解



#### @SPI

@SPI可以作用于类、接口、枚举上，dubbo中都是在接口上使用，它主要是标记这个接口是一个SPI接口（也就是一个扩展点），通过配置加载不同的实现类

```java
@Documented
// 作用于运行时
@Retention(RetentionPolicy.RUNTIME)
// 作用于类、接口、枚举
@Target({ElementType.TYPE})
public @interface SPI {

    /**
     * 默认扩展点名称
     * 指定名称则默认加载该名称对应的实现类
     */
    String value() default "";

}
```



#### ©Adaptive

@Adaptive注解可以标记在类、接口、枚举类和方法上，在dubbo中作用在类级别的只有AdaptiveExtensionFactory和AdaptiveCompile，作用在方法上则可以通过参数动态的加载实现类，在第一次调用ExtensionLoader.getExtension()方法时，会自动生成一个动态的Adaptive类（动态实现类）Class$Adaptive类，

类里会实现扩展类中存在的方法，通过@Adaptive传入的参数找到并调用对应的实现类。

**Transporter接口示例：**

![image-20200806223343927](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200806223343927.png)

从示例中可以看到，dubbo在自动生成的动态类中加入了一些抽象逻辑，比如获取url参数，通过url参数的value获取对应的实现类，如果获取失败则调用默认配置（@SPI的value）的实现类，最终还是会调用实现类对应的方法（动态代理）

当@Adaptive作用在实现类上时，该实现类会直接作为默认实现类，不在自动生成动态类，在一个接口（扩展点）有多个实现类时，只能有一个实现类可以加@Adaptive，如果不止一个实现类加@Adaptive，会抛出：More than 1 adaptive class found异常

```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    /**
     * 1.配置通过哪些参数名获取META-INF/duubo或META-INF/duubo/internal中的key
     * 2.dubbo会通过URL中的参数找到Adaptive对应配置key的value，通过value找到配置文件中对应的实现类
     * 3.传入的是数组，dubbo会按照数组顺序匹配实现类
     */
    String[] value() default {};
}
```

在初始化Adaptive注解的接口时，会对传入的URL进行key的匹配，找到对应的value从配置文件中获取实现类，如果第一个没匹配上则继续下一个，以此类推，如果全都没匹配到则使用**”驼峰规则“**匹配

驼峰规则：如果包装类（wapper，实现了接口（扩展点）且有构造方法能注入接口（装饰器））没有使用Adaptive，则dubbo会将该包装类名根据驼峰大小写拆分用"."连接，以此来作为默认实现类，如：com.zjf.test.TestUserWapper被拆分为test.user.wapper



#### @Activate

@Activate可以标记在类、接口、枚举上，可以通过不同条件激活多个实现类

```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Activate {
    /**
     * 匹配分组的Key，可以多个
     * 传入的分组名一致，则就匹配上对应实现类
     */
    String[] group() default {};

    /**
     * 匹配key，可以多个
     * URL中的key如果匹配且value不为空，则匹配上对应实现类
     * group()是第一层匹配，如果匹配上了则继续匹配value(),反之返回匹配group对应扩展实现
     */
    String[] value() default {};

    /**
	  * 设置哪些扩展点在当前扩展点之前
     */
    String[] before() default {};

    /**
	  * 设置哪些扩展点在当前扩展点之后
     */
    String[] after() default {};

    /**
	  * 排序优先级，值越小优先级越高，最低不得低于0
     */
    int order() default 0;
}
```

匹配的原理会在下面的文档中介绍



### ExtensionLoader

 ExtensionLoader是整个扩展机制的核心，这个类中实现了配置的加载、扩展类的缓存、动态类的生成



#### 工作流程

ExtensionLoader的逻辑入口有三个：getExtension()、getAdaptiveExtension()、getActivateExtension()，分别是获取普通扩展类，获取自适应扩展类，获取自动激活扩展类

![image-20200808221557474](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200808221557474.png)

getActivateExtension()会做一些通用的逻辑判断，如：接口是否使用了@Activate，条件是否匹配等等，最终调用getExtension()获取扩展点的实现类

**getExtension()是加载器最核心的方法，主要步骤如下**：

1. 加载所有配置文件里的实现类并缓存（这一步不会初始化实现类）
2. 根据URL中传入的key找到对应实现类并初始化
3. 这一步会尝试查找扩展点的包装类：包含扩展点类型的setter()，例如setUserService(UserService userSevice)，初始化完UserService后寻找构造方法中需要传入UserService的类，然后注入UserService并初始化这个类
4. 返回扩展点实现类实例



#### getExtension实现原理

下列代码清单依次列出了重要的几个步骤的方法实现

1. **加载文件**

```java
/**
  * 加载扩展实现类Class
**/
private Map<String, Class<?>> loadExtensionClasses() {
    // 获取当前扩展类的SPI注解
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    // 注解不为空时做一些校验
    if (defaultAnnotation != null) {
        // @SPI.value
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            // 配置的key不能有多个
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName() + ": " + Arrays.toString(names));
            }
            // 设置默认名称
            // 注：cachedDefaultName就是getExtension(name)的name=true时需要的
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    // 加载3个目录下的配置，通过io读取文件内容，获取到所有实现类的全路径类名
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    loadDirectory(extensionClasses, DUBBO_DIRECTORY);
    loadDirectory(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```

2. **获取扩展实现类**

```java
// 加载的配置文件目录
private static final String SERVICES_DIRECTORY = "META-INF/services/";
// 加载的配置文件目录
private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";
// 加载的配置文件目录
private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/"

// 扩展实现类名称（配置文件中的key，下同）和对应Class的缓存
private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String, Class<?>>>();

// 扩展实现类名称和对应对象的缓存
private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();

// 扩展实现类Class和对应实例的缓存
private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();

// 扩展类的包装类缓存
private Set<Class<?>> cachedWrapperClasses;

// 扩展类的默认名称缓存
private Strring cachedDefaultName;

/**
  * 根据扩展实现类名称获取对应的实现类
  * @Param name 扩展类名称
 **/
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    // 如果为true，返回默认配置的实现类
  	if ("true".equals(name)) {
        // cachedDefaultName，@SPI设置的value
        // 下文中会介绍cachedDefaultName变量赋值原理
        return getDefaultExtension();
    } 
    // 先从缓存中获取对象，不存在则先往缓存设置name，后续获取到对象后再设置value
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    // 如果对象为空，则创建实现类
    // 经典的双重锁，防止重复创建
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建扩展点的实现类
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

3. **创建扩展实现类**

```
/**
  * 根据name创建对应的实现类
  * @Param name 扩展实现类名称
 **/
private T createExtension(String name) {
	   // 从扩展实现类名称和对应Class的缓存中获取到Class，如果不存在则抛异常
	   // dubbo启动时会将所有配置文件内的实现类缓存到cachedClasses中
       Class<?> clazz = getExtensionClasses().get(name);
       if (clazz == null) {
           throw findException(name);
       }
       try {
           // 从扩展实现类Class和对应实例的缓存中获取，获取不到则实例化对象并放到缓存中
           T instance = (T) EXTENSION_INSTANCES.get(clazz);
           if (instance == null) {
               // 实例化对象并放到缓存中
               EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
               instance = (T) EXTENSION_INSTANCES.get(clazz);
           }
           // 设置属性，类似ioc的依赖注入
           // 1.循环所有方法，找到只有set开头、只有1个参数、修饰符为public的方法
           // 2.获取参数类型
           // 3.通过方法名截取到小写开头的类名，如setUserService，则截取后的结果为userService
           // 4.获取userService实例，原理与getExtension一致，不过这里调用的是objectFactory.getExtension(Class cls, String name)，后续会介绍
           // 5.如果获取到了实例，则调用set方法设置实例对象
           injectExtension(instance);
           // 找到该扩展点所有包装类（构造方法中传入此扩展点类型的类），注入扩展点实例
           Set<Class<?>> wrapperClasses = cachedWrapperClasses;
           if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
               for (Class<?> wrapperClass : wrapperClasses) {
                   // 通过包装类的构造方法实例化
                   instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
               }
           }
           return instance;
       } catch (Throwable t) {
           throw new IllegalStateException("Extension instance(name: " + name + ", class: " + type + ")  could not be instantiated: " + t.getMessage(), t);
       }
   }
```



#### getAdaptiveExtension实现原理

在getAdaptiveExtension()中会自动为扩展点自动生成实现类字符串，主要逻辑：为每个有@Adaptive的方法默认生成默认实现（没有用@Adaptive的方法生成空实现），每个默认实现会从URL中获取@Adaptive配置的值，加载对应实现，然后使用不同的编译器把实现类字符串编译为自适应类并返回

1. 生成package、import、类名称等信息，此处只会引用一个类：ExtensionLoader，其他字段和方法调用时候全部使用全路径类名，如：java.lang.String，类名称会变成："类名"+$Adaptive格式，如：UserService$Adaptive
2. 遍历类中所有方法，获取方法的返回类型、参数类型、异常类型等
3. 生成参数为空的校验代码，如有远程调用还会添加Invocation参数为空的校验
4. 生成默认实现类名称，如果@Adaptive没有配置值，则根据类名生成，如UserService=user.service
5. 生成获取扩展点名称的代码，根据@Adaptive配置的值生成不同的获取代码，如@Adaptive({"user"})，则会生成url.getUser()
6. 生成获取扩展点实现类的代码，调用getExtension(extName)获取扩展实现类
7. 拼接所有字符串，生成代码

**由于源码太长且比较简单（各种拼接代码），这里不做解释，直接上示例演示**

**示例如下：**

```java
// 配置
impl=com.zjf.service.UserServiceImpl

@SPI("impl")
public interface UserService{
    
    @Adaptive
    void printInfo(URL url);
}

// 调用
public static void main(String[] args) {
    UserService userService = ExtensionLoader.getExtensionLoader(UserService.class)
        .getAdaptiveExtension();
    userService.printInfo(URL.valueof("dubbo://127.0.0.1:8080/test"));
}  
```

**自动生成的代码如下：**

```java
// 生成package代码
package org.apache.dubbo.common.extensionloader.adaptive;
// 仅生成import ExtensionLoader代码
import org.apache.dubbo.common.extension.ExtensionLoader;

// 生成类名称UserService$Adaptive并实现扩展接口com.zjf.service.UserService
// 目的就是为了生成接口中的方法并调用，典型的代理模式
public class UserService$Adaptive implenments com.zjf.service.UserService{
    
    // 生成接口中的所有方法
    public void printInfo(org.apache,.dubbo.common.URL arg0){
        // 参数判空校验
        if (arg0 == null) {
            throw new IllegalArgumentException("url == null");
        }
        // 获取扩展名称
        // 此处如果@Adaptive没有配置值，则默认优先获取默认的类名称user.service，生成规则参考实现原理第4			步
        // 如果@Adaptive配置了值，如@Adaptive("key")，则生成：String extName = 							url.getParameter("key","impl");代码
        String extName = url.getParameter("user.service","impl");
        if (extName == null) {
            throw new IllegalStateException("extName == null");
        } 
        // 最终还是调用getExtension方法获取对应的扩展实现类，然后调用该方法返回结果
        com.zjf.service.UserService extension = (com.zjf.service.UserService)
            ExtensionLoader.getExtenloader(com.zjf.service.UserService extension.class)
            .getExtension(extName);
        return extension.printInfo(arg0);
    }
}

```

注：如果@SPI和@Adaptive都设置了值，则优先获取@Adaptive中的值，获取不到再获取@SPI的值，如果2个注解都没有设置值，则默认生成扩展点的名称，如扩展点为UserService，那么生成的名称为user.service



#### getActivateExtension 的实现原理

```java
// 获取自动激活的扩展实现类列表
public List<T> getActivateExtension(URL url, String key, String group) {
    String value = url.getParameter(key);
    // 对key进行”,“拆分，封装成数组
    return getActivateExtension(url, value == null || value.length() == 0 ? null : Constants.COMMA_SPLIT_PATTERN.split(value), group);
}

/**
  * 实际调用
  * @Param URL url
  * @Param values 需要自动激活的key列表（扩展点名称列表），匹配@Activate.values
  * @Param group 需要自动激活的group值，匹配@Activate.group
**/
public List<T> getActivateExtension(URL url, String[] values, String group) {
    // 需要自动激活的扩展实现类列表
    List<T> exts = new ArrayList<T>();
    // 需要自动激活的配置key列表
    List<String> names = values == null ? new ArrayList<String>(0) : Arrays.asList(values);
    // 如果需要自动激活的配置key列表列表中包含“-default"，则不加载默认的扩展实现类
    // 此段逻辑是加载dubbo默认的扩展实现类
    if (!names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {
        // 加载所有扩展实现类Class
        getExtensionClasses();
        // 遍历key和@Activate的缓存，将需要扩展的实现类放入列表
        for (Map.Entry<String, Activate> entry : cachedActivates.entrySet()) {
        	// 配置的扩展点key（配置文件中的key）
            String name = entry.getKey();
            // key对应的@Activate
            Activate activate = entry.getValue();
            // 通过传入的group值和@Activate配置的group匹配
            if (isMatchGroup(group, activate.group())) {
                // 如果匹配上则自动激活，获取key对应的扩展实现类
                T ext = getExtension(name);
                // 如果key包含"-"或者需要激活的key列表中不包含此key或者当前activate不是@Activate则不会自动激活
                if (!names.contains(name)
                        && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
                        && isActive(activate, url)) {
                    // 将需要自动激活的扩展实现类加入到列表
                    exts.add(ext);
                }
            }
        }
        // 将扩展实现类列表根据sort、before、after配置进行排序
        // 这里实现了Comparator接口，重写了排序规则
        Collections.sort(exts, ActivateComparator.COMPARATOR);
    }
    // 用户自定义需要被自动激活的扩展实现类列表
    List<T> usrs = new ArrayList<T>();
    // 对用户自定义需要自动激活的key遍历，找到对应的实现类，放入列表
    for (int i = 0; i < names.size(); i++) {
        String name = names.get(i);
        // 如果key以“-”开头或key包含"-"+key将不会被自动激活
        if (!name.startsWith(Constants.REMOVE_VALUE_PREFIX)
                && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
            // 如果name=default，则将用户自定义的扩展实现类放在前面
            if (Constants.DEFAULT_KEY.equals(name)) {
                if (!usrs.isEmpty()) {
                    exts.addAll(0, usrs);
                    usrs.clear();
                }
            } else {
                // 获取扩展实现类，放入列表
                T ext = getExtension(name);
                usrs.add(ext);
            }
        }
    }
    if (!usrs.isEmpty()) {
        exts.addAll(usrs);
    }
    // 返回所有需要自动激活的扩展实现类
    return exts;
}
```



#### ExtensionFactory的实现原理

通过上述文档我们可以得知ExtensionLoader是整个SPI的核心，那么ExtensionLoader是如何被创建的？

答案是：ExtensionFactory

```java
@SPI
public interface ExtensionFactory {
	<T> T getExtension(Class<T> type. String name);
}
```

ExtensionFactory是一个工厂，它有3个实现类，分别是：AdaptiveExtensionFactory、SpringExtensionFactory、SpiExtensionFactory，除了从SPI容器中获取扩展实现类，我们也可以从Spring容器中获取扩展实现类。

**AdaptiveExtensionFactory原理**

```
// 默认实现，使用了@Adaptive
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {
	
	// 所有工厂实现的集合
    private final List<ExtensionFactory> factories;
	
	// 通过ExtensionLoader获取ExtensionFactory所有的实现类
    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }
	
	// 遍历所有工厂获取扩展实现类
	// 最终还是通过AdaptiveExtensionFactory获取扩展实现类
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}
```



**SpiExtensionFactory原理**

```java
public class SpiExtensionFactory implements ExtensionFactory {
	// 重写方法
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        // 判断class是不是接口 & 是不是有@SPI
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            // 最终还是通过ExtensionLoader从缓存冲获取默认扩展实现
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            if (!loader.getSupportedExtensions().isEmpty()) {
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }

}
```



**SpringExtensionFactory原理**

```java
public class SpringExtensionFactory implements ExtensionFactory {
	// Spring上下文集合
    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
	
	// 设置上下文
    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);
    }
	
	// 移除上下文
    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }
	
	// 重写方法
    @Override
    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {
    	// 遍历每个Spring上下文，从容器中获取bean
        for (ApplicationContext context : contexts) {
            if (context.containsBean(name)) {
                Object bean = context.getBean(name);
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }
        return null;
    }


```



### Dubbo启停原理解析



##### XML配置解析原理

主要逻辑入口是在DubboNamespaceHandler中完成的，DubboNamespaceHandler继承NamespaceHandlerSupport，主要功能是实现Spring个性化的配置，具体原理可自行查阅源码

```
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
	// 检查包冲突，这里不做着重介绍
    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }
	
	// 实现init方法
	// 将各个配置交由DubboBeanDefinitionParser处理
	// DubboBeanDefinitionParser会将各个配置生成BeanDefinition交给Spring托管
    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```



**DubboBeanDefinitionParser**

由于**DubboBeanDefinitionParser**的parse方法太长，这里我们将分段介绍



1. 将标签解析成BeanDefinition

```java
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
	// BeanDefinition 这里就不多讲了
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
    // 设置BeanDefinition的一些信息，比如class和是否懒加载
    beanDefinition.setBeanClass(beanClass);
    beanDefinition.setLazyInit(false);
    // 获取标签中的id属性，如<dubbo:service id="userService">
    String id = element.getAttribute("id");
    // 如果没有配置id，则获取name的值作为beanName
    if ((id == null || id.length() == 0) && required) {
        String generatedBeanName = element.getAttribute("name");
        // 如果name还为空，则获取标签中interface的值作为beanName
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            // 如果是协议配置类，则默认是dubbo
            if (ProtocolConfig.class.equals(beanClass)) {
                generatedBeanName = "dubbo";
            } else {
                // 否则获取interface的值作为beanName
                generatedBeanName = element.getAttribute("interface");
            }
        }
        // 如果id、name、interface的值都为空，则获取class的名称作为beanName
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            generatedBeanName = beanClass.getName();
        }
        id = generatedBeanName;
        // 这里为啥是2
        int counter = 2;
        // 检查BeanDefinitionMap中是否已经存在相同beanName的BeanDefinition
        while (parserContext.getRegistry().containsBeanDefinition(id)) {
            // 如果存在则生成唯一id
            id = generatedBeanName + (counter++);
        }
    }
    if (id != null && id.length() > 0) {
        // 再次检查BeanDefinitionMap中是否已经存在相同beanName
        if (parserContext.getRegistry().containsBeanDefinition(id)) {
            throw new IllegalStateException("Duplicate spring bean id " + id);
        }
        // 往BeanDefinitionMap中放入BeanDefinition
        parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
        // 设置BeanDefinition的id
        beanDefinition.getPropertyValues().addPropertyValue("id", id);
    }
    
    ...
}
```

2. service标签解析

```java
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
	...
    else if (ServiceBean.class.equals(beanClass)) {
        // 获取service标签的calss，<dubbo:service class="com.zjf.service.UserService">
        String className = element.getAttribute("class");
        if (className != null && className.length() > 0) {
            RootBeanDefinition classDefinition = new RootBeanDefinition();
            classDefinition.setBeanClass(ReflectUtils.forName(className));
            classDefinition.setLazyInit(false);
            // 解析标签中的属性，如：class,name,ref,value，并将这些属性填充到BeanDefinition中
            parseProperties(element.getChildNodes(), classDefinition);
            // 将service标签ref的值注册为BeanDefinition
            // <bean id="UserServiceImpl" class="com.zjf.service.impl.UserServiceImpl">
            // <dubbo:service class="com.zjf.service.UserService" ref="UserServiceImpl">
            // 也就是将具体的实现类注册为BeanDefinition
            beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
        }
    } 
    ...
}
```



##### 注解解析原理

注解配置使用@EnableDubbo

###### @EnableDubbo

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {
    /**
     * 扫描配置包下使用了@Service的类
     * 配置包的全路径名，如：com.zjf.service
     */
    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};

    /**
     * 扫描配置包下使用了@Service的类
	 * 配置类的class，如：UserService.Class
     */
    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};


    /**
     * 配置是否多注册中心
     */
    @AliasFor(annotation = EnableDubboConfig.class, attribute = "multiple")
    boolean multipleConfig() default false;

}
```

###### @EnableDubboConfig

在@EnableDubbo中，可以看到有2个注解，分别是：@EnableDubboConfig、@DubboComponentScan。

@DubboComponentScan用来扫描使用了@Service的类生成ServiceBean，@EnableDubboConfig用来解析属性文件的配置

我们来看@EnableDubboConfig做了些什么

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
// 这里使用了@Import，当Spring启动时，会调用DubboConfigConfigurationSelector
@Import(DubboConfigConfigurationSelector.class)
public @interface EnableDubboConfig {

    /**
     * 配置是否多注册中心
     */
    boolean multiple() default false;
}


public class DubboConfigConfigurationSelector implements ImportSelector, Ordered {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
    importingClassMetadata.getAnnotationAttributes(EnableDubboConfig.class.getName()));
        boolean multiple = attributes.getBoolean("multiple");
        // 加载单或多注册中心
        if (multiple) {
            return of(DubboConfigConfiguration.Multiple.class.getName());
        } else {
            return of(DubboConfigConfiguration.Single.class.getName());
        }
    }
    
// 加载属性文件中的配置
public class DubboConfigConfiguration {
    /**
     * Single Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableDubboConfigBindings({
            @EnableDubboConfigBinding(prefix = "dubbo.application", type = ApplicationConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.module", type = ModuleConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.registry", type = RegistryConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.protocol", type = ProtocolConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.monitor", type = MonitorConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.provider", type = ProviderConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.consumer", type = ConsumerConfig.class)
    })
    public static class Single {

    }

    /**
     * Multiple Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableDubboConfigBindings({
            @EnableDubboConfigBinding(prefix = "dubbo.applications", type = ApplicationConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.modules", type = ModuleConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.registries", type = RegistryConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.protocols", type = ProtocolConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.monitors", type = MonitorConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.providers", type = ProviderConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.consumers", type = ConsumerConfig.class, multiple = true)
    })
    public static class Multiple {

    }
}
```

看到这里我们就明白了，Dubbo使用@EnableDubboConfigBindings来加载属性文件中的配置，在@EnableDubboConfigBindings中，同样@Import(DubboConfigBindingsRegistrar.class)，我们来看下DubboConfigBindingsRegistrar做了些什么

```java
public class DubboConfigBindingsRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {

    private ConfigurableEnvironment environment;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(  importingClassMetadata.getAnnotationAttributes(EnableDubboConfigBindings.class.getName()));
		// 拿到EnableDubboConfigBindings属性的值
        AnnotationAttributes[] annotationAttributes = attributes.getAnnotationArray("value");

        DubboConfigBindingRegistrar registrar = new DubboConfigBindingRegistrar();
        registrar.setEnvironment(environment);

        for (AnnotationAttributes element : annotationAttributes) {
			// 注册BeanDefinition，托管给Spring
			// 如：dubbo.application.name,则会创建对应的ApplicationConfig
			// 具体的bean可以查看上面@EnableDubboConfigBinding配置的Class
            // 底层调用器DubboConfigBindingBeanPostProcessor注册和设置bean的属性值
            registrar.registerBeanDefinitions(element, registry);
        }
    }
}
```



###### @DubboComponentScan

@DubboComponentScan主要功能是扫描使用了@Service和@Reference的类，将其生成对应的Bean

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 这里使用了DubboComponentScanRegistrar
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {

    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

}
```

```java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		// 得到扫描的包全路径名，如：com.zjf.service
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
		// 注册@Service的处理器，在bean初始化之前和之后做操作
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);
		// 注册@Reference的处理器，在bean初始化之前和之后做操作
        // 这一步会初始化ReferenceBean
        registerReferenceAnnotationBeanPostProcessor(registry);
    }
}
```

1. **ServiceAnnotationBeanPostProcessor**

```java
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {
            
       @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
		// 需要扫描的包，如：com.zjf.service
        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);
        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            // 注册ServiceBean
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }

    }
       // 注册ServiceBean
       private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
			
        // 类扫描器
        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);
        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
        scanner.setBeanNameGenerator(beanNameGenerator)
        // 指定扫描@Service
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));
		// 遍历包
        for (String packageToScan : packagesToScan) {
            // 指定扫描的包
            scanner.scan(packageToScan);
            // 扫描指定包下所有的@Service，创建对应的BeanDefinitionHolder
            // BeanDefinitionHolder持有BeanDefinition
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);
            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    // 注册ServiceBean
                    // 在ServiceBean初始化时，dubbo会对它进行各种属性赋值，在服务暴露章节会介绍
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }
				// ...
                // ...
            }
        }
    } 
}
```

2. **ReferenceAnnotationBeanPostProcessor**

```java
public class ReferenceAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
        implements MergedBeanDefinitionPostProcessor, PriorityOrdered, ApplicationContextAware, BeanClassLoaderAware,
        DisposableBean {
            
    // 因为实现了InstantiationAwareBeanPostProcessorAdapter
    // 所以在Spring启时候会触发此方法
    @Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
        // 获取所有使用了@Reference的字段和方法
        InjectionMetadata metadata = findReferenceMetadata(beanName, bean.getClass(), pvs);
        try {
            // 依赖注入
            metadata.inject(bean, beanName, pvs);
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of @Reference dependencies failed", ex);
        }
        return pvs;
    }
            
    // 获取所有使用了@Reference的字段（也可以说是类）  
    private List<ReferenceFieldElement> findFieldReferenceMetadata(final Class<?> beanClass) {
        final List<ReferenceFieldElement> elements = new LinkedList<ReferenceFieldElement>();
        ReflectionUtils.doWithFields(beanClass, new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
                // 获取@Reference
                Reference reference = getAnnotation(field, Reference.class);
                if (reference != null) {
                    // 使用了@Reference的字段不能被static修饰
                    if (Modifier.isStatic(field.getModifiers())) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("@Reference annotation is not supported on static fields: " + field);
                        }
                        return;
                    }
                    // 将使用了@Reference的字段添加到集合中
                    // 字段如：@Reference
                    //        private UserService userService;
                    elements.add(new ReferenceFieldElement(field, reference));
                }
            }
        });
        return elements;
    }
}
```



#### 服务暴露

在上述流程走完后，我们知道会生成ServiceBean的BeanDefinition，Spring在接下来的IOC流程中会将此BeanDefinition初始化为Spring的Bean，因为ServiceBean实现了InitializingBean接口，所以在Bean初始化前会调用afterPropertiesSet()方法，duubo在此方法内实现了自己的初始化逻辑，比如获取各个配置，暴露服务等

```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware {
    
    // 对初始化前的BeanDefinition
    public void afterPropertiesSet() throws Exception {
        // 各种配置的加载和拼装
        ...
        ...
            
       if (!isDelay()) {
           // 暴露服务，最终还是调用的ServiceConfig中的doExportUrls()
            export();
        }
    }
}
```

我们来看下doExportUrls()做了一些什么事

```java
private void doExportUrls() {
    // 获取所有的协议
    // 如：<dubbo:protocol name="dubbo" prompt="20890" id="protocolTest" />
    // 或者ProtocolConfig
    List<URL> registryURLs = loadRegistries(true);
    // 多协议暴露
    // dubbo支持多协议暴露，如果需要支持多协议，可以声明多个 <dubbo:protocol> 标签，并在 				<dubbo:service> 中通过 protocol 属性指定使用的协议。
    // 后续会根据Protocol找到对应协议实现类
    for (ProtocolConfig protocolConfig : protocols) {
        // 根据协议暴露服务
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```

继续看下doExportUrlsFor1Protocol()做了些什么

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        String name = protocolConfig.getName();
        if (name == null || name.length() == 0) {
            name = "dubbo";
        }
		// map用于存放各种配置属性，用于后面构造URL
        Map<String, String> map = new HashMap<String, String>();
        map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
        map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
        map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
        if (ConfigUtils.getPid() > 0) {
            map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
        }
        // 比如application、module、provider等
        appendParameters(map, application);
        appendParameters(map, module);
        appendParameters(map, provider, Constants.DEFAULT_KEY);
        appendParameters(map, protocolConfig);
        appendParameters(map, this)
        ...
         // 判断暴露的作用域，如本地暴露，远程暴露 
         if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
            // 如果不是远程暴露，则本地暴露
            if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
                exportLocal(url);
            }
            // 反之远程暴露
            if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
                // 有注册中心
                if (registryURLs != null && !registryURLs.isEmpty()) {
                    for (URL registryURL : registryURLs) {
                        // 获取动态注册参数dynamic，如果URL中dynamic为空就取注册中心的dynamic
                        // dynamic：服务是否动态注册，如果为空或者false，注册后为disable状态，需人工							开启
                        // 反之动态注册
                        // 具体的各种参数可以前往官方文档阅读
                        // http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-							service.html
                        url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                        // 获取监控中心地址，将服务暴露信息上报给监控中心
                        URL monitorUrl = loadMonitor(registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                        }
                        // 动态代理生成Invoker,所有的调用都会委托给代理
                        // 默认使用javassist动态代理，可通过<dubbo:service proxy='代理方式'/>指							 定，可选：jdk/javassist
                        // ref：需要暴露的实现类，如：com.zjf.service.UserServiceImpl
                        // interfaceClass为对应的实现类Class
                        // registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString())是向URL追加export=xxxx数据
                        // 如：registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=provider_1&dubbo=2.6.2&export=dubbo://192.168.199.149:20880/com.dubbo.api.service.UserServiceanyhost=true&application=provider_1&bind.ip=192.168.199.149&bind.port=20880&dubbo=2.6.2&dynamic=true&generic=false&interface=com.dubbo.api.service.UserService&methods=getUserByName,getUserList&pid=9536&prompt=20890&proxy=jdk&side=provider&timestamp=1597674777771
                        // Invoker包含了代理类、url、wapper包装类等属性
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
                        // 向注册中心注册服务信息，默认dubbo，可通过protocol参数指定
                        // 这里也是通过@Adaptive根据指定的protocol动态获取协议实现类
                        Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
                } else {
                    // 如果没有注册中心，则直接暴露服务
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            }
        }
        this.urls.add(url);
}
```



为了更直接的展示服务暴露和注册中心的关系，我们列出有无注册中心的URL和说明

```java
// 有注册中心，会追加=dubbo://ip:port/xxx?..
registry://host:port/com.alibaba.dubbo.registry.RegistryService?protocol=zo
okeeper&export=dubbo://ip:port/xxx?..。 
// 没有注册中心
dubbo://ip:host/xxx.Service?timeout=1000&..
```

1. protocol会根据URL自动适配，有注册中心的则会取出对应的协议，如：zookeeper
2. 创建注册中心实例
3. 获取export参数的value，如：export=dubbo://192.168.199.149:20880/com.dubbo.api.service.UserServiceanyhost=true&application=provider_1&bind.ip=192.168.199.149&bind.port=20880&dubbo=2.6.2&dynamic=true&generic=false&interface=com.dubbo.api.service.UserService&methods=getUserByName,getUserList&pid=9536&prompt=20890&proxy=jdk&side=provider&timestamp=1597674777771
4. 使用对应的协议（如：dubbo）进行服务暴露，暴露成功后向注册中心注册服务数据
5. 如果没有注册中心则直接暴露服务



##### RegistryProtocol.export

有注册中心的情况下（URL以registry:开头），会先调用RegistryProtocol.export()，然后再调用具体协议（如dubbo）的export()，我们先来看看RegistryProtocol.export()做了些什么

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        // 本地暴露服务，存储在内存中（Injvm://）
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
		// 注册中心URL
    	// 如：zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?x=x
        URL registryUrl = getRegistryUrl(originInvoker);

        // 创建注册中心实例
        final Registry registry = getRegistry(originInvoker);
        // 需要注册的URL
        final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
        // registedProviderUrl是否需要注册，默认为true
        boolean register = registedProviderUrl.getParameter("register", true);
    	// 将接口(如:com.zjf.service.UserService)缓存到服务提供者的map中，方便下次使用
    	// 服务提供者放入服务提供者缓存表ProviderConsumerRegTable.providerInvokers
    	// 消费者放入服务提供者缓存表ProviderConsumerRegTable.consumerInvokers
        ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);
		// 注册服务
        if (register) {
            // 注册服务
            // 通过@Adaptive({"protocol"})获取registryUrl中protocol对应的注册实现类，							如：ZookeeperRegistry
            // registryUrl：注册中心地址
            // registedProviderUrl：接口服务地址
            register(registryUrl, registedProviderUrl);
            // 将该服务标记为已注册
            ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
        }
		// ...
        // ...
        return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registedProviderUrl);
    }
```

通过源码可以看到，RegistryProtocol.export方法会先服务暴露，然后注册到注册中心并缓存到本地的注册表中



##### 拦截器

在服务调用的时候会触发各种拦截器，那么这些拦截器是什么时候被初始化的？

从dubbo SPI文档中我们知道有几个特性，其中就有”自动包装“，在服务暴露前，dubbo通过SPI加载protocol实现类，此时会加载包装类Protocol Listenerwrapper 和 ProtocolFilterWrapper

###### Listenerwrapper 

```java
public class ProtocolListenerWrapper implements Protocol {

    private final Protocol protocol;
	// 在protocol被SPI加载的时候会自动触发自动包装特性调用此构造方法
    public ProtocolListenerWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
    
    // 接下来会调用此方法
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            return protocol.export(invoker);
        }
        // 获取所有用户实现的的监听器Listener，执行相关业务逻辑
        // ExporterListener有2个方法，分别是expored()和unexpored()，需结合@SPI和@Activate使用才会被dubbo的SPI加载到，示例在下面
        return new ListenerExporterWrapper<T>(protocol.export(invoker),      Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                        .getActivateExtension(invoker.getUrl(), Constants.EXPORTER_LISTENER_KEY)));
    }
}
```

###### ExporterListener示例

用户自定义实现ExporterListener可以在服务暴露或者消费/引用完成后做自己的业务逻辑

```java
// META-INF/dubbo配置，文件为：com.alibaba.dubbo.rpc.ExporterListener
listener=com.dubbo.producer.listener.MyListener

// service配置
<dubbo:service interface="com.dubbo.api.service.UserService" ref="userService" id="aaaaa" dynamic="true" proxy="jdk">
    // 配置listener属性，key为exporter.listener，value为自定义配置文件中的key
    <property name="exporter.listener" value="listener"/>
</dubbo:service>
    

// 设置@Activate
@Activate
public class MyListener implements ExporterListener {

    @Override
    public void exported(Exporter<?> exporter) throws RpcException {
        System.out.println("暴露完成调用"+exporter.getInvoker().getInterface().getName());
    }
    
    @Override
    public void unexported(Exporter<?> exporter) {
        System.out.println("销毁完成调用"+exporter.getInvoker().getInterface().getName());
    }
}
```

###### ProtocolFilterWrapper

ProtocolFilterWrapper也是protocol的包装类，所以在spi加载的时候会自动包装该类，调用其export方法

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            return protocol.export(invoker);
    }
    // 构建拦截器链，通过group包含provider和key=service.filterkey的扩展类
    return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
}
```

```java
/**
	* 构建拦截器链
	* @Param key  url中对应的属性名
	* @Param group @Activate中配置的group
**/
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
      Invoker<T> last = invoker;
  		// 获取所有的filter，通过url中service.filterkey的value找到@Activate.group包含provider的扩展				 拦截器
      // filters包含了用户自己实现的filter和dubbo自带的filter
      List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
        if (!filters.isEmpty()) {
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
              	// 将暴露的服务放到拦截器链的最后面，保证最后调用
                final Invoker<T> next = last;
                last = new Invoker<T>() {

                    @Override
                    public Class<T> getInterface() {
                        return invoker.getInterface();
                    }

                    @Override
                    public URL getUrl() {
                        return invoker.getUrl();
                    }

                    @Override
                    public boolean isAvailable() {
                        return invoker.isAvailable();
                    }
										// 设置下一次调用的拦截器，形成拦截器链
                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
                        return filter.invoke(next, invocation);
                    }

                    @Override
                    public void destroy() {
                        invoker.destroy();
                    }

                    @Override
                    public String toString() {
                        return invoker.toString();
                    }
                };
            }
        }
        return last;
    }
```

在服务暴露前dubbo会先加载listener和构建filter链，并将暴露的服务放到拦截链的最后面，保证最后调用。

在加载完listener和构建完filter后，如果有注册中心，dubbo会调用RegistryProtocol.export()进行注册中心暴露（上述文档中有讲），然后再调用DubboProtocol.export()，反之直接调用DubboProtocol.export()

**注：Protocol的@SPI默认值为dubbo，所以会调用DubboProtocol.export()**



##### Dubbo协议暴露

在构建完拦截链后，dubbo会调用RegistryProtocol.export()将服务注册到注册中心，然后会调用具体的协议进行服务的远程暴露，默认为dubbo，可通过<dubbo:procotol name="协议名">指定，如：dubbo，rmi，http，等

```java
// 此map在父类AbstractProtocol中，用来缓存暴露的服务和对应的export对象，方便使用，不用重复生成
// 在dubboProcotol中，也声明了此map且使用的也是此map
protected final Map<String, Exporter<?>> exporterMap = new ConcurrentHashMap<String, Exporter<?>>();

public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();
    // 
    // 生成缓存key，通过<dubbo:service>中的group+version+interface和端口号属性value组成
    // 如：testGroup/com.zjf.service.UserService:1.0:20880
    String key = serviceKey(url);
    // 创建暴露协议的对象，继承至Invoker
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    // 缓存服务和对应的export对象
    exporterMap.put(key, exporter);

    ...
    ...
    
    // 首次暴露服务会创建监听服务器
    openServer(url);
    // 序列化
    optimizeSerialization(url);
    return exporter;
}
```



##### 本地暴露

本地暴露很简单，在ServiceConfig的服务暴露方法中，会判断scope是否是本地暴露，如果是则会调用injvmProcotol.export()进行服务暴露

```java
private void exportLocal(URL url) {
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        // 指定协议为injvm，ip为127.0.0.1，端口为0
        URL local = URL.valueOf(url.toFullString())
                .setProtocol(Constants.LOCAL_PROTOCOL)
                .setHost(LOCALHOST)
                .setPort(0);
        ServiceClassHolder.getInstance().pushServiceClass(getServiceClass(ref));
        // 调用InjvmProcotol.export()暴露服务
        // 不做端口打开操作，服务保存在内存中（exporterMap中），直接返回InjvmExport对象
        Exporter<?> exporter = protocol.export(
                proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
    }
}
```



#### 服务消费

服务暴露完成后，接下里就是服务消费/引用，本章节主要探讨服务是如何被消费的，在这之前先来了解下消费原理

![image-20200825212839776](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200825212839776.png)

通过上图可以看到，服务消费分为2大部分

1. 通过协议将远程服务转换成Invoker
2. 将Invoker通过动态代理生成代理类

服务引用的入口在ReferenceBean.getObject()，熟悉Spring的大佬应该知道这个方法是FactoryBean中的方法，是用来获取bean的

因为ReferenceBean实现了InitializingBean接口，所以在Bean初始化前会调用afterPropertiesSet()方法，afterPropertiesSet跟ServiceBean的一样，也是拼装各种配置，完事后会判断是否已初始化完成，如果已经初始化完成则会调用getObject()

getObject()调用了ReferenceConfig的get()，get()内会获取服务引用，如果服务引用为空则会初始化

```java
private void init() {
	 try {
        // 通过反射获取Class
        interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                .getContextClassLoader());
    } catch (ClassNotFoundException e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
    // 创建对应Class代理
    ref = createProxy(map);
}
```

上述init()已删除掉不重要代码



###### createProxy

下面是createProxy的实现

```java
private T createProxy(Map<String, String> map) {
    URL tmpUrl = new URL("temp", "localhost", 0, map);
    final boolean isJvmRefer;
    // 判断是否在同一虚拟机内，如果在就不用走远程调用
    if (isInjvm() == null) {
        if (url != null && url.length() > 0) { 
            isJvmRefer = false;、
        } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
            isJvmRefer = true;
        } else {
            isJvmRefer = false;
        }
    } else {
        isJvmRefer = isInjvm().booleanValue();
    }
    // 校验Class是否是一个interface，引用的服务方法是否存在
    checkInterfaceAndMethods(interfaceClass, methods);
    // 如果在同一虚拟机内，则直接通过exporterMap获取已经暴露的服务实例即可
    if (isJvmRefer) {
        URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
        invoker = refprotocol.refer(interfaceClass, url);
    } else {
        // 直连消费
        if (url != null && url.length() > 0) {
            String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (url.getPath() == null || url.getPath().length() == 0) {
                        url = url.setPath(interfaceName);
                    }
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        // 追加ref引用，追加信息为引用的接口信息
                        // 如：registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=comsumer_1&dubbo=2.6.2&pid=76979&refer=application%3Dcomsumer_1%26dubbo%3D2.6.2%26interface%3Dcom.dubbo.api.service.UserService%26methods%3DgetUserByName%2CgetUserList%26pid%3D76979%26register.ip%3D10.8.19.13%26scope%3Dremote%26side%3Dconsumer%26timestamp%3D1598404453465&registry=zookeeper&timestamp=1598404467441
                        urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    } else {
                        // 直接某一台服务提供者，不通过注册中心消费服务
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else { 
            // 获取注册中心
            List<URL> us = loadRegistries(false);
            if (us != null && !us.isEmpty()) {
                for (URL u : us) {
                    URL monitorUrl = loadMonitor(u);
                    // 添加监控
                    if (monitorUrl != null) {
                        map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                    }
                    // 追加ref引用，追加信息为引用的接口信息
                    // registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=comsumer_1&dubbo=2.6.2&pid=76979&refer=application%3Dcomsumer_1%26dubbo%3D2.6.2%26interface%3Dcom.dubbo.api.service.UserService%26methods%3DgetUserByName%2CgetUserList%26pid%3D76979%26register.ip%3D10.8.19.13%26scope%3Dremote%26side%3Dconsumer%26timestamp%3D1598404453465&registry=zookeeper&timestamp=1598404467441
                    urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                }
            } 
      	 }
        // 单注册中心
        if (urls.size() == 1) {
            // 直接引用服务
            // 调用RegistryProtocol和默认DubboProtocol的refer()创建Invoker
            invoker = refprotocol.refer(interfaceClass, urls.get(0));
        } else {
            // 多注册中心
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
            for (URL url : urls) {
                // 从每个注册中心引用服务
                // 调用RegistryProtocol和默认DubboProtocol的refer()创建Invoker
                invokers.add(refprotocol.refer(interfaceClass, url));
                if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    registryURL = url;
                }
            }
            if (registryURL != null) {
                // 将多个URL转换成一个Invoker
                URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                invoker = cluster.join(new StaticDirectory(u, invokers));
            } else { 
                invoker = cluster.join(new StaticDirectory(invokers));
           }
        }
    }
    // 拿到调用RegistryProtocol和默认DubboProtocol的refer()创建的Invoker
    // 将Invoker通过工厂创建服务的代理，默认为javassist
    return (T) proxyFactory.getProxy(invoker);
}
```

在上述服务消费流程中refprotocol.refer()方法，如有注册中心（URL以registry:开头）dubbo会调用RegistryProtocol.refer()，然后再根据具体协议配置调用不同实现类的refer()，默认为dubbo



###### RegistryProtocol.refer

```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    // 指定协议，默认为dubbo
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    // 获取注册中心实例
    // url为注册中心url，引用的服务是ref参数值
    Registry registry = registryFactory.getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // 对ref后的url进行分组聚合,前提是暴露服务或者引用服务配置了group，group不匹配则默认为dubbo
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    String group = qs.get(Constants.GROUP_KEY);
    if (group != null && group.length() > 0) {
        if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
                || "*".equals(group)) {
            return doRefer(getMergeableCluster(), registry, type, url);
       }
    }
    // 引用服务逻辑
    return doRefer(cluster, registry, type, url);
}
```

我们继续看下doRefer实现

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    // 构建RegistryDirectory
    // 持有Invoker和接收订阅通知，如服务有变更则会收到消息做重新引用操作
    // 实现了NotifyListener.notify()，服务注册、变更、重试、销毁都会回调这个方法
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    // 设置注册中心和协议
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    // 将url中属性转换成map，如side、methods、scope、interface等等
    Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
    // 订阅url
    // 如：consumer://10.8.19.13/com.dubbo.api.service.UserService?application=comsumer_1&dubbo=2.6.2&interface=com.dubbo.api.service.UserService&methods=getUserByName,getUserList&pid=80034&scope=remote&side=consumer&timestamp=1598409765245
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
    if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
        // 将服务引用注册到注册中心
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,Constants.CHECK_KEY, String.valueOf(false)));
    }
    // 订阅providers、routes、configurators
	// 之后服务、配置等信息有变更能通过监听器获取到
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
            Constants.PROVIDERS_CATEGORY
                    + "," + Constants.CONFIGURATORS_CATEGORY
                    + "," + Constants.ROUTERS_CATEGORY));
    Invoker invoker = cluster.join(directory);
    // 将引用的服务添加到本地缓存表中
    ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
    return invoker;
}
```



###### DubboProtocol.refer

RegistryProtocol.refer()调用完成后，dubbo会继续调用具体实现类的refer()，默认为DubboProtocol.refer()

```java
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    // 序列化
    optimizeSerialization(url);
    // 创建远程连接
    // getClients(url)：获取连接，如果当前没有连接则会初始化连接
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

initClient

```java
private ExchangeClient initClient(URL url) {
    // 获取连接的配置，默认为netty，可通过server参数指定
	String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));
    // 设置编码
	url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    // 新增开启心跳机制，默认为dubbo
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
	
    ExchangeClient client;
    try {
        // 是否延迟创建连接，通过lazy属性配置，默认为false
        if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
            // 真正调用才会创建连接
            client = new LazyConnectExchangeClient(url, requestHandler);
        } else {
            // 立即创建连接
            client = Exchangers.connect(url, requestHandler);
        }
    } catch (RemotingException e) {
        throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
    }
    return client;
}
```



#### 优雅停机

1. 收到kill -9 命令，Spring容器销毁接收JVM退出事件（Runtime.getRuntime().addShutdownHook()）
2. 如果是consumer停机，consumer会停止消费新的服务，取消掉所有已订阅的信息，如路由、配置、服务
3. 如果是provider停机，则provider停止注册服务，comsumer会收到最新的服务列表（不包含准备停机的服务）
4. 如果是provider停机，考虑到comsumer获取可用服务列表需要一些时间和监听服务信息变更有延迟，所以dubbo会发送readonly通知对应的consumer，consumer会将消费的provider设置为不可用，下次负载不会选择停用的provider
5. 等待正在执行的任务执行完成，拒绝新任务提交



### 集群容错

在实际的生产环境中，为了保证服务的高可用，服务通常是以集群方式部署，但这也不能保证集群中每个节点每时每刻都是好用的，如果调某个接口出现异常，比如网络波动，节点挂掉的情况，这个时候就需要集群容错机制来应对这种场景

##### Cluster 层

Cluster层是集群容错层，仅仅是个抽象概念，该层包含Cluster、Directory、Router、LoadBalance主要核心接口，这里的Cluster层和Cluster接口是2个东西，不要搞混了，Cluster层是概念，Cluster是具体的接口，提供N个容错策略，如：Failover、Failfast等

由于Cluster层的实现太多，因此这里只介绍基于于Abstractclusterinvoker的全流程，Cluster层工作流程大致为以下几步

1. 生成Invoker
2. 获取可用的服务列表
3. 负载
4. RPC调用

![image-20200830221327895](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200830221327895.png)



其中，1、2、3步都在Abstractclusterinvoker中实现，这是一个抽象的公共流程，最终调用是通过doInvoker()，这个方法是由子类实现



##### 容错机制的实现

Cluster接口一共有9种实现，每个实现对应不同的Clusterlnvoker，由于Merge和Mock是特殊实现，所以本章节只介绍其中的7种实现，Merge和Mock会在其他章节介绍

| 机制      | 介绍                                                         | 是否对请求做负载 |
| --------- | ------------------------------------------------------------ | ---------------- |
| Failover  | 重试（默认），通过<dubbo:service >、<dubbo:provider >、<dubbo:reference > 、<dubbo:consumer >的retries属性设置，默认值为2次，重试次数不包含第一次调用，如果两方都配置了，则优先使用消费方 | 是               |
| Failfast  | 快速失败，返回异常结果                                       | 是               |
| Failsafe  | 出现异常会忽略，不关心调用的结果，不影响下面的业务流程，如：日志同步， | 是               |
| Failback  | 请求失败后进入失败队列，由一个定时线程池重试，重试成功会从队列中移除，否则一直重试。适用于异步保持一致性的业务 | 是               |
| Forking   | 同时调用多个服务，通过forks参数配置同时调用服务数量，只要其中一个服务有返回则立即返回结果 。假设forks=5，可用服务s=10，则同时请求5个节点，如果forks=10，可用服务s=8，则同时请求8个节点 | 否               |
| Broadcast | 广播调用所有服务，只要其中一个服务返回错误/异常，则报错，适用于如更新某个信息后广播给其他系统 | 否               |
| Mock      | 提供调用失败时，返回伪造的响应结果。或直接强制返回伪造的结果，不会发起远程调用 | 否               |
| Available | 遍历所有服务，调用第一个服务并返回结果，如果服务列表为空则直接抛出异常 | 否               |
| Mergeable | 自动把多个节点请求得到的结果进行合并                         | 否               |



##### Cluster接口关系

dubbo服务只需通过Cluster层就可以完成容错逻辑的调用，包含了获取可用服务列表、路由、负载等等，Cluster接口串联了整个逻辑，其中Clusterlnvoker实现了容错策略，Directory实现获取服务列表、Route实现路由、LoadBalance实现负载，下面分别详细介绍这几个的具体实现

###### Cluster接口

```java
// SPI加载对应实现类，默认failover
@SPI(FailoverCluster.NAME)
public interface Cluster {

    // 构造具体的Clusterlnvoker，比如FailoverClusterInvoker
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;
}
```

Cluster通过SPI加载URL中配置的容错机制，具体的实现类通过join方法构造具体的lnvoker对象返回，每个实现类返回一个具体的Invoker实现，比如FailoverCluster的join方法会new FailoverClusterInvoker返回，每个实现都继承至AbstractClusterInvoker，且都会重写doInvoke方法，实现不同的容错策略

AbstractClusterInvoker封装了统一的逻辑，比如获取服务列表，负载，服务调用，其中服务调用由子类实现



##### 调用流程

1. 调用AbstractClusterInvoker.invoker方法
2. 通过Directory.list方法获取可使用的服务列表，list方法由子类重写，分别有RegistryDirectory和StaticDirectory
3. 获取负载均衡LoadBalance，默认为random随机
4. 调用抽象方法doInvoke（由子类实现）

其中服务列表、负载的公共逻辑是统一由AbstractClusterInvoker.invoker管理，doInvoke方法由子类实现不同的容错策略和服务调用

AbstractClusterInvoker还提供了select方法供子类使用，select方法主要功能是将获取服务列表、路由、负载公共逻辑管理，子类不用关心具体实现，是需要做自己的容错实现即可

```java
protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
    // 是否开启了粘滞连接
    // 粘滞连接：尽可能的调用同一个服务节点，除非该节点挂了
    // 通过<dubbo:reference>sticky属性配置，默认为false
    boolean sticky = invokers.get(0).getUrl().getMethodParameter(methodName, Constants.CLUSTER_STICKY_KEY, Constants.DEFAULT_CLUSTER_STICKY);
    {
        // 如果服务列表不包含粘滞连接过的Invoker，则置为空
        if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
            stickyInvoker = null;
        }
        // 如果开启了粘滞连接并且没有重复调用
        if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
            // 如果粘滞连接的Invoker可用则直接返回
            if (availablecheck && stickyInvoker.isAvailable()) {
                return stickyInvoker;
            }
        }
    }
    // 空值校验，服务列表不能为空
    if (invokers == null || invokers.isEmpty())
        return null;
    // 获取本次调用的方法名，如getUserInfo
    String methodName = invocation == null ? "" : invocation.getMethodName();
	// 省略部分不重要代码
    // 获取可用服务列表
    Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);
    // 开启了粘滞连接，则设置粘滞连接Invoker
    if (sticky) {
        stickyInvoker = invoker;
    }
    // 返回通过负载后获取的invoker
    return invoker;
}
```

看下doSelect方法的实现

```java
private Invoker<T> doSelect(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
    // 空值校验，服务列表不能为空
    if (invokers == null || invokers.isEmpty())
        return null;
    // 如果服务列表只有1个，则直接返回，不用做负载
    if (invokers.size() == 1)
        return invokers.get(0);
    // 如果服务列表只有2个，则优先使用已经负载过的invoker实例
    if (invokers.size() == 2 && selected != null && !selected.isEmpty()) {
        // 如果服务列表第一个被调用过了则调用第二个，反之调用第一个
        return selected.get(0) == invokers.get(0) ? invokers.get(1) : invokers.get(0);
    }
    // 未配置负载策略
    if (loadbalance == null) {
        // 获取默认负载策略，默认为：random
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
    }
    // 通过负载获取负载后的Invoker实例，具体的负载实现会在后续文档中介绍
    Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);
    // 如果invoker是之前已经调用过的实例 或 配置了”需要检查invoker是否可用“且“invoker不可用”则需要重新选择Invoker
    if ((selected != null && selected.contains(invoker))
            || (!invoker.isAvailable() && getUrl() != null && availablecheck)) {
        try {
            // 重新选择可用Invoker
            // 如果开启了“需要检查invoker是否可用”配置，则遍历服务列表检查每个服务是否可用
            Invoker<T> rinvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
            // 重新选择到了Invoker则赋值即可
            if (rinvoker != null) {
                invoker = rinvoker;
            } else {
              // 如果没有重选到Invoker，则获取通过第一次负载得到的invoker在服务列表的下标位置
              int index = invokers.indexOf(invoker);
                try {
                   // 如果invoker不在服务列表末尾，则获取invoker后一位的invoker实例，反之使用此							invoker，这么做是防止获取服务的碰撞
                   invoker = index < invokers.size() - 1 ? invokers.get(index + 1) : invoker;
                } catch (Exception e) {
                   // 错误日志
                }
            }
        } catch (Throwable t) {
            // 错误日志
        }
    }
    return invoker;
}
```



##### 具体容错实现

**Failover**

```java
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyinvokers = invokers;
        checkInvokers(copyinvokers, invocation);
    	// 获取配置的重试次数，不包含第一次调用，所以要+1
        int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
    	// 未配置重试次数则不重试
        if (len <= 0) {
            len = 1;
        }
        // 记录最后一次异常信息
        RpcException le = null; 
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); 
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
            // 如果不是第一次调用，则表明当前正在重试，需要重新check信息
            if (i > 0) {
                checkWhetherDestroyed();
                // 重新获取服务列表，检查信息
                copyinvokers = list(invocation);
                checkInvokers(copyinvokers, invocation);
            }
            // 调用select方法获取invoker实例
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
            // 添加到已调用/选择过的列表中，为接下来的重试使用
            invoked.add(invoker);
            // 将已调用/选择过的invoker实例放到上下文中
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                // 服务调用
                Result result = invoker.invoke(invocation);
                // 返回结果
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                // 记录调用过的服务地址信息，为下面抛出异常信息服务
                providers.add(invoker.getUrl().getAddress());
            }
        }
        // 抛出服务异常信息
        throw new RpcException("异常信息，不记录，不重要");
    }
```

**Failfast**

```java
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
      checkInvokers(invokers, invocation);
      // 调用select方法获取invoker实例
      Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
      try {
           // 服务调用
     	   return invoker.invoke(invocation);
      } catch (Throwable e) {
          // 有异常直接抛出
          if (e instanceof RpcException && ((RpcException) e).isBiz()) {
              throw (RpcException) e;
          }
          throw new RpcException("异常信息，不记录，不重要");
      }
  }
```

**Failsafe**

```java
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        checkInvokers(invokers, invocation);
        // 调用select方法获取invoker实例
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        // 服务调用
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        // 忽略异常，返回空结果
        return new RpcResult();
    }
}
```

**Fallback**

```java
protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        checkInvokers(invokers, invocation);
         // 调用select方法获取invoker实例
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
		// 服务调用
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        // 加入失败集合，定期重试，五秒一次
        addFailed(invocation, this);
        return new RpcResult();
    }
}
// 失败集合
private final ConcurrentMap<Invocation, AbstractClusterInvoker<?>> failed = new ConcurrentHashMap<Invocation, AbstractClusterInvoker<?>>();

private void addFailed(Invocation invocation, AbstractClusterInvoker<?> router) {
    // 双重锁，防止重复创建定时调度线程池
    if (retryFuture == null) {
        synchronized (this) {
            if (retryFuture == null) {
                // 创建定时调度线程池
                retryFuture = scheduledExecutorService.scheduleWithFixedDelay(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            // 对map：failed集合进行重试，五秒一次
                            // 调用成功则从失败集合中移除
                            retryFailed();
                        } catch (Throwable t) {
                        }
                    }
                }, RETRY_FAILED_PERIOD, RETRY_FAILED_PERIOD, TimeUnit.MILLISECONDS);
            }
        }
    }
    failed.put(invocation, router);
}
```

**Available** 

```java
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    // 遍历服务列表，调用可用的第一个服务，失败或无可用服务则抛出异常
    for (Invoker<T> invoker : invokers) {
        if (invoker.isAvailable()) {
            return invoker.invoke(invocation);
        }
    }
    throw new RpcException("No provider available in " + invokers);
}
```

**Broadcast** 

```java
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    // 设置上下文
    RpcContext.getContext().setInvokers((List) invokers);
    // 记录异常信息，抛出最后一次异常
    // 如果中间发生调用异常，则记录，后续再发生异常则覆盖
    RpcException exception = null;
    // 调用返回结果，返回最后一次调用结果
    Result result = null;
    for (Invoker<T> invoker : invokers) {
        try {
            // 调用服务
            result = invoker.invoke(invocation);
        } catch (RpcException e) {
            exception = e;
            logger.warn(e.getMessage(), e);
        } catch (Throwable e) {
            exception = new RpcException(e.getMessage(), e);
            logger.warn(e.getMessage(), e);
        }
    }
    // 只要发生异常则抛出
    if (exception != null) {
        throw exception;
    }
    return result;
}
```

**Forking**

```java
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    // 已经选择可调用的服务
    final List<Invoker<T>> selected;
    // 获取配置的forks并行调用数量，默认并行调用数量为2
    final int forks = getUrl().getParameter(Constants.FORKS_KEY, Constants.DEFAULT_FORKS);
    // 获取配置的调用超时时间，默认超时时间为1s
    final int timeout = getUrl().getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
    // 如果并行调用数量<=0 或者 并行调用数量大于服务数量，则并行调用所有服务
    if (forks <= 0 || forks >= invokers.size()) {
        selected = invokers;
    } else {
        selected = new ArrayList<Invoker<T>>();
        // 遍历并行调用数量，通过select方法获取可调用服务
        for (int i = 0; i < forks; i++) {
            Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
            if (!selected.contains(invoker)) {
                // 放入已选择可调用服务列表
                selected.add(invoker);
            }
        }
    }
    RpcContext.getContext().setInvokers((List) selected);
    // 调用服务发生异常次数
    final AtomicInteger count = new AtomicInteger();
    // 存储调用服务返回结果或异常信息的阻塞队列
    final BlockingQueue<Object> ref = new LinkedBlockingQueue<Object>();
    for (final Invoker<T> invoker : selected) {
        // 使用线程池异步
        并行调用服务，线程池为：Executors.newCachedThreadPool()
        executor.execute(new Runnable() {
           @Override
            public void run() {
                try {
                    // 调用服务
                    Result result = invoker.invoke(invocation);
                    // 存储调用结果
                    ref.offer(result);
                } catch (Throwable e) {
                    // 异常次数+1
                    int value = count.incrementAndGet();
                    // 如果异常次数>=已选择服务列表数量，则存入异常信息
                    // 也就是说所有的服务调用都失败了
                    if (value >= selected.size()) {
                        ref.offer(e);
                    }
                }
            }
        });
    }
    try {
        // 上面线程池在异步调用，这里阻塞等待第一个结果，有结果直接返回
        // 如果是异常则抛出异常，反之返回调用结果
        Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);
        if (ret instanceof Throwable) {
            Throwable e = (Throwable) ret;
            throw new RpcException();
        }
        return (Result) ret;
    } catch (InterruptedException e) {
        throw new RpcException();
    }
}
```



##### 获取服务列表

在调用服务容错流程中，服务列表是通过Directory.list()获取的，Directory是一个接口，定义了list()，它有一个抽象类AbstractDirectory，该抽象类实现list()定义了获取服务列表的公共逻辑，而且还提供了doList()给子类实现

AbstractDirectory共有2个实现类，分别是：动态RegistryDirectory和静态StaticDirectory，由于StaticDirectory.doList()是直接返回传入的服务列表，所以这里不做阐述

**RegistryDirectory的作用**

1. **实现doList()，返回服务列表**
2. **通过实现NotifyListener.notify()监听配置、服务、路由信息，有变化则更新到本地缓存**



下面是RegistryDirectory.doList()的实现

```java
 public List<Invoker<T>> doList(Invocation invocation) {
    // 如果服务提供者被禁用则抛异常
    // 1.没有服务提供者 2.服务提供者被禁用
    if (forbidden) {
        throw new RpcException();
    }
    // 返回的服务列表
    List<Invoker<T>> invokers = null;
    // 本地引用的服务缓存，数据来源是通过监听更新到methodInvokerMap中
    // 其中key=方法名 value=服务列表
    Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap; 
    if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
        // 方法名
        String methodName = RpcUtils.getMethodName(invocation);
        // 方法中的参数
        Object[] args = RpcUtils.getArguments(invocation);
        // 如果有参数，则通过方法名+第一个参数获取服务列表，看不懂为啥这么做，没应用场景
        if (args != null && args.length > 0 && args[0] != null
                && (args[0] instanceof String || args[0].getClass().isEnum())) {
            invokers = localMethodInvokerMap.get(methodName + "." + args[0]);
        // 为空就通过方法名获取服务列表
        if (invokers == null) {
       		invokers = localMethodInvokerMap.get(methodName);
        }
        // 还为空，就通过"*"获取服务列表，也就是获取所有的服务列表
        if (invokers == null) {
            invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
        }
        // 仍然为空，返回最后一个服务列表
        if (invokers == null) {
            Iterator<List<Invoker<T>>> iterator = localMethodInvokerMap.values().iterator();
             if (iterator.hasNext()) {
                invokers = iterator.next();
            }
        }
   	}
    return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
}
```



##### 负载均衡实现

整个服务调用过程，经过获取服务列表、路由，接下来就是通过对服务列表做负载均衡，返回可用的一个Invoker

在调用过程中，我们发现并不是直接调用LoadBalance接口，而是通过调用Abstractclusterinvoker.select()做负载均衡，dubbo在LoadBalance接口层面上再次封装了一层Clusterinvoker，目的是为了做一些公共的逻辑和负载流程的管理

Abstractclusterinvoker公共特性：

1. 在上述文档"调用过程"章节中提到过的粘滞连接就是在Abstractclusterinvoker中做的，dubbo会优先检测粘滞连接，然后再做负载
2. 避免重复调用
3. 负载均衡选出可用Invoker

具体的源码实现在"调用过程"章节中有介绍，这里不再做阐述

下面是负载均衡的接口代码

```java
// 通过SPI加载对应的实现类，默认为random
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {

    /**
     * 通过URL中loadbalance配置的value从服务列表中选出一个可用的Invoker
     *
     * @param invokers   服务列表.
     * @param url        消费/引用的服务URL
     * @param invocation 参数.
     * @return 负载得到的Invoker.
     */
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
}
```

在实际的调用过程中，并不是直接调用LoadBalance，而是调用它的抽象实现类AbstractLoadBalance，它为LoadBalance做了统一的公共逻辑封装，而且还提供了doSelect方法给子类实现自己的负载策略，下面会介绍



###### dubbo负载均衡策略

| 名称                       | 介绍                                                         |
| -------------------------- | ------------------------------------------------------------ |
| Random LoadBalance         | 随机，通过配置的权限设置随机获取的概率，权重通过<dubbo:service >weight属性配置，权重是通过计算得来的，而非是按照配置来 |
| RoundRobin LoadBalance     | 轮询，按配置的权重设置轮询的比例，与随机的区别是，随机是通过随机数选择，轮询是一台一台的来 |
| LeastActive LoadBalance    | 最少活跃调用数，如果活跃度调用数相同则随机                   |
| ConsistentHash LoadBalance | 一致性hash                                                   |



在分别介绍每个负载均衡的实现原理前，我们需要先了解下权重是如何实现的，因为随机和轮询都用到了权重，在前面提到过的AbstractLoadBalance中，做了统一的公共逻辑封装，权重计算逻辑就是封装在该抽象类中

```java
protected int getWeight(Invoker<?> invoker, Invocation invocation) {
    // 从 url 中获取权重 weight 配置值，默认为100
    int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT);
    if (weight > 0) {
        // 获取服务提供者启动时间戳
        long timestamp = invoker.getUrl().getParameter(Constants.REMOTE_TIMESTAMP_KEY, 0L);
        if (timestamp > 0L) {
            // 计算服务提供者运行时长
            int uptime = (int) (System.currentTimeMillis() - timestamp);
            // 获取服务预热时间，默认为10分钟
            // 通过warmup属性配置
            int warmup = invoker.getUrl().getParameter(Constants.WARMUP_KEY, Constants.DEFAULT_WARMUP);
            // 如果服务运行时间小于预热时间，则重新计算服务权重，即降权
            if (uptime > 0 && uptime < warmup) {
                // 重新计算服务权重
                weight = calculateWarmupWeight(uptime, warmup, weight);
            }
        }
    }
    return weight;
}

// 重新计算权重
// uptime：服务运行时间  warmup：服务预热时间  weight：初次权重值
static int calculateWarmupWeight(int uptime, int warmup, int weight) {
    // 计算权重，下面代码逻辑上形似于 (uptime / warmup) * weight。
    // 随着服务运行时间 uptime 增大，权重计算值 ww 会慢慢接近配置值 weight
    int ww = (int) ((float) uptime / ((float) warmup / (float) weight));
    // 如果重新计算的权重大于初次权重则使用初次权重，因为要降权不能比初次权重大
    return ww < 1 ? 1 : (ww > weight ? weight : ww);
}
```

了解完权重计算逻辑后，我们来看下各个负载的doSelect方法的实现



###### Random

随机负载，通过权重尽可能的让请求均匀的随机

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    int length = invokers.size();
    // 总权重
  	int totalWeight = 0; 
    // Invoker权重是否相同
    boolean sameWeight = true;
    for (int i = 0; i < length; i++) {
        // 获取服务权重值
        int weight = getWeight(invokers.get(i), invocation);
        // 累加到总权重
        totalWeight += weight;
        // 判断当前Invoker是否与上一个Invoker权重相同
        if (sameWeight && i > 0
                && weight != getWeight(invokers.get(i - 1), invocation)) {
            sameWeight = false;
        }
    }
    if (totalWeight > 0 && !sameWeight) {
        // 如果权重不相同则根据总权重值随机出一个偏移量
        // 1.比如invokers内有4个Invoker，权重分别是：4、3、2、1，总权重为10，则每个Invoker的概率为：4/10、3/10、2/10、1/10
        // 2.通过totalWeight总权重获取0~10的随机数offset，假设为8
        // 3.偏移量递减，例如：8-4=3，3不小于0，则继续，3-3=0，0不小于0，则继续，0-2=-2，-2小于0，则返回此区间内的Invoker
        // 4.直到递减后的offset<0，则返回此偏移量区间内的invoker
        int offset = random.nextInt(totalWeight);
        for (int i = 0; i < length; i++) {
            // 偏移量递减，获取随机的区间
           	offset -= getWeight(invokers.get(i), invocation);
            if (offset < 0) {
                // 返回区间内的Invoker
                return invokers.get(i);
            }
        }
    }
    // 如果权重相同则随机即可
    return invokers.get(random.nextInt(length));
}
```



###### RoundRobin

轮询负载，通过权限尽可能让请求均匀的轮询

缺陷：2.6.4版本前的代码如下，如果权重非常大，请求次数也很大，则mod的值也会非常大，那么需要很多次计算才能把mod减至0，时间复杂度为o(mod)，性能很低，dubbo在2.6.4之后版本修复了这个问题，有兴趣的可以自行阅读源码（官网有）

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    // 缓存key，服务名.方法名，如：com.zjf.service.getUserInfo
    String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
    // 服务列表长度
    int length = invokers.size();
    // 最大权重
    int maxWeight = 0;
  	// 最小权重，默认最大
    int minWeight = Integer.MAX_VALUE;
    // 缓存每个Invoker权重
    final LinkedHashMap<Invoker<T>, IntegerWrapper> invokerToWeightMap = new LinkedHashMap<Invoker<T>, IntegerWrapper>();
    // 权重总和
    int weightSum = 0;
    for (int i = 0; i < length; i++) {
        // 获取Invoker权重
   			int weight = getWeight(invokers.get(i), invocation);
        // 设置最大权重值
        maxWeight = Math.max(maxWeight, weight);
        // 设置最小权重值
        minWeight = Math.min(minWeight, weight);
        // 服务权重>0
        if (weight > 0) {
           // 缓存invoker和对应的权重信息
           invokerToWeightMap.put(invokers.get(i), new IntegerWrapper(weight));
           // 总和累加
           weightSum += weight;
       }
    }
    // 计数器，通过服务名.方法名获取
    AtomicPositiveInteger sequence = sequences.get(key);
   	if (sequence == null) {
        sequences.putIfAbsent(key, new AtomicPositiveInteger());
        sequence = sequences.get(key);
    }
    // 当前调用次数，第一次为0，第二次为1...
    int currentSequence = sequence.getAndIncrement();
    // 如果配置了权重或者权重不一样，则按权重轮询
    if (maxWeight > 0 && minWeight < maxWeight) {
        // 剩余比较次数
        // 比如currentSequence=0，weightSum=20，则第一次调用。mod=0
        // 当currentSequence=1，weightSum=20，则第二次调用。mod=1
        // 当currentSequence=3，weightSum=20，则第三次调用。mod=3 以此类推
        int mod = currentSequence % weightSum;
        for (int i = 0; i < maxWeight; i++) {
            // 循环每个Invoker和对应的权重，权重越大轮询次数越多
            for (Map.Entry<Invoker<T>, IntegerWrapper> each : invokerToWeightMap.entrySet()) {
                final Invoker<T> k = each.getKey();
                final IntegerWrapper v = each.getValue();
                // 做比较，当mode=0，且剩余权重>0时，返回invoker
                // 也就是说当请求次数越多，做递减操作后，权重低的invoker.value将会被减至0，就不会返回此				   invoker
                if (mod == 0 && v.getValue() > 0) {
                    return k;
                }
                // 如果mod>0，则做递减操作，继续循环比较，直到mod=0返回invoker
                // 目的是为了权重低的invoker不会得到轮询的机会
                if (v.getValue() > 0) {
                    v.decrement();
                    mod--;
                }
            }
        }
    }
    // 如果没有配置权重或者权重相等或者上面循环内权重被递减至0时，则通过请求次数%服务列表数量取余，轮询获取Invoker
    // 比如currentSequence=0，length=2，则返回第一个invoker
  	// 当currentSequence=1，length=2，则返回第二个invoker
    return invokers.get(currentSequence % length);
}
```



###### LeastActive

最小活跃数，当多个Invoker最小活跃数一致，但权重不相等时，使用随机+权重算法获取对应Invoker

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    // 服务列表数量
    int length = invokers.size();
    // 服务列表中最小的活跃数
    int leastActive = -1;
    // 最小活跃数相等的Invoker数量
    int leastCount = 0;
    // 存储最小活跃数相等的Invoker在invokers的下标
    int[] leastIndexs = new int[length];
    // 总权重
    int totalWeight = 0;
    // 第一个最小活跃数Invoker的权重
    int firstWeight = 0;
    // 用来判断Invoker的权重是否相等
    boolean sameWeight = true;
    for (int i = 0; i < length; i++) {
        Invoker<T> invoker = invokers.get(i);
        // invoker最小活跃数值
        int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive();
        // invoker权重值
        int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT); 
        // 如果有更小活跃数的invoker
        if (leastActive == -1 || active < leastActive) { 
            // 设置最小活跃数值
            leastActive = active;
            // 最小活跃数的invoker数量为1
            leastCount = 1;
            // 存储invoker在invokers中的下标值
            leastIndexs[0] = i;
            // 总权重值
            totalWeight = weight; 
            // 第一个最小活跃数invoker权重值
            firstWeight = weight;
            sameWeight = true;
        } else if (active == leastActive) { // 如果当前invoker最小活跃数与服务列表中最小活跃数一致
            // 存储invoker在invokers中的下标值
            leastIndexs[leastCount++] = i;
            // 累加权重值
            totalWeight += weight;
            // 如果权重值不一样，则设置sameWeight=false
            if (sameWeight && i > 0
                    && weight != firstWeight) {
                sameWeight = false;
            }
        }
    }
    // 如果只有一个最小活跃数的invoker则直接返回
    if (leastCount == 1) {
        return invokers.get(leastIndexs[0]);
    }
    // 多个最小活跃数相同，权重不相同的invoker，
    if (!sameWeight && totalWeight > 0) {
        // 生成0~总权重值之间的随机数
        int offsetWeight = random.nextInt(totalWeight);
        // 根据权重值随机获取invoker返回，与随机负载均衡机制逻辑一致，可参考“Random”章节
        for (int i = 0; i < leastCount; i++) {
            int leastIndex = leastIndexs[i];
            offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
            if (offsetWeight <= 0)
               return invokers.get(leastIndex);
        }
    }
    // 如果最小活跃数、权重都一致，则随机返回对应invoker即可
    return invokers.get(leastIndexs[random.nextInt(leastCount)]);
}
```



###### ConsistentHash

一致性hash，待更新