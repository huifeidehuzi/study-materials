# Spring Framework

建议配合spring官网源码介绍阅读
此笔记依据4.3.14版本记录
以下简称Spring

Spring其实包含了很多东西，比如AOP、IOC、Test、MVC、CACHE、Groovy等等，以下文档着重分析常用的IOC和AOP



## IOC

### Bean的生命周期

在这之前，一定要分清楚对象和Bean的区别，对象的创建、初始化、Bean实例化的区别

- Bean：有一套完整的spring生命周期，spring会对其进行比如aop、初始化前、初始化后、销毁前、自动注入等等的操作
- 对象：就是一个普通的java对象



### 生命周期大致流程

[![s0kx3j.jpg](https://s3.ax1x.com/2021/01/15/s0kx3j.jpg)](https://imgchr.com/i/s0kx3j)



### 实例化Bean的步骤

1. 创建工厂

   ```java
   public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
       // 创建工厂，初始化
       public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
           // 这一步，会初始化工厂
           // 首先会执行父类GenericApplicationContext的无参构造，再执行自己的无参构造
           this();
           // 这俩方法先暂时忽略
           // register(annotatedClasses);
           // refresh();
       }
   }
   public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {
   	// 1. 初始化工厂，默认为DefaultListableBeanFactory
   	public GenericApplicationContext() {
   		this.beanFactory = new DefaultListableBeanFactory();
   	}
   }
   
   // 2. AnnotationConfigApplicationContext无参构造
   public AnnotationConfigApplicationContext() {
       // 初始化 读取器，用来注册”配置类“
       // 如：在AnnotationConfigApplicationContext(Class<?>... annotatedClasses)中的register()方法
   		this.reader = new AnnotatedBeanDefinitionReader(this);
       // 初始化 扫描器，用于扫描传入的basepackages中的类并且注册为BeanDefinition放到BeanDefinitionMap
       // 如：在AnnotationConfigApplicationContext(String... basePackages)中的scan()方法
   		this.scanner = new ClassPathBeanDefinitionScanner(this);
   }
   
   
   ```
   
2. 扫描配置的config类，封装成BeanDefinition放到BeanDefinitionMap

   ```java
   public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
   		// 这一步将配置类封装成BeanDefinition放到BeanDefinitionMap
       // 内部调用BeanDefinitionRegistry#registerBeanDefinition()方法注册的，就不细讲了
       // 这里调用的是AnnotatedBeanDefinitionReader#register()方法
       register(annotatedClasses);
       // 这个先忽略
       // refresh();
   }
   ```

3. 调用invokeBeanFactoryPostProcessors，通过ConfigurationClassPostProcessor扫描有@Component、@Service、@Bean等注解的类封装成BeanDefinition并放到BeanDefinitionMap中（详细步骤在【ConfigurationClassPostProcessor】章节有介绍），内部调用invokeBeanDefinitionRegistryPostProcessors执行BeanDefinition注册后期处理器，其内部调用了DefaultListableBeanFactory.registerBeanDefinition()方法扫描需要注册的类放到BeanDefinitionMap

   ```java
   public void refresh() throws BeansException, IllegalStateException {
      synchronized (this.startupShutdownMonitor) {
        // 执行BeanFactoryPostProcessor所有实现并且注册为bean
        // 先执行Spring内部的，再执行开发者自定义的
        invokeBeanFactoryPostProcessors(beanFactory);
      }
   }
   ```

4. 如果有自定义的BeanFactoryPostProcessor实现，则执行实现的自定义逻辑，到这里，基本上往BeanDefinitionMap放的操作已经结束了，下面就开始对这个map中的BeanDefinition实例化了

5. 调用finishBeanFactoryInitialization().preInstantiateSingletons()，实例化bean，并放到单例池(map)中
   
   
   ```java
   public void preInstantiateSingletons() throws BeansException {
      // 所有的beanName
      List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);
   
      for (String beanName : beanNames) {
         // 获取合并过，最新的RootBeanDefinition
         // 因为在这之前BeanFactoryPostProcessor可以对BeanDefinition进行修改，bean定义可能会有变化，需要重新合并，以保证合并的是最新的
      // RootBeanDefinition是BeanDefinition比较全面的一个子类
         RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
         // 如果BeanDefinition是抽象的、非单例的、懒加载的，是不会被实例化的
         if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // FactoryBean的实现
            if (isFactoryBean(beanName)) {
               // &+name 获取bean
               final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                     @Override
                     public Boolean run() {
                        return ((SmartFactoryBean<?>) factory).isEagerInit();
                     }
                  }, getAccessControlContext());
               }
               else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
            else {
               // 最重要的方法，如果get不到则创建bean
               // AbstractBeanFactory#getBean()
               getBean(beanName);
            }
         }
      }
   	 // ...省略部分代码
   }
   ```
   
   getBean调用AbstractBeanFactory#doGetBean()方法
   
   ```java
   protected <T> T doGetBean(
         final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
         throws BeansException {
   	 // 转换beanName，比如：如果是FactoryName,BeanName就是&+beanName
      final String beanName = transformedBeanName(name);
      Object bean;
   	 // 获取单例对象，第一次肯定为空
      Object sharedInstance = getSingleton(beanName);
      if (sharedInstance != null && args == null) {
         bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
      }
      else {
       		// 省略部分代码 
         try {
            // 获取BeanDefinition
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            // 检查BeanDefinition
            checkMergedBeanDefinition(mbd, beanName, args);
       		// 省略部分代码 
            // 重要：创建bean
            if (mbd.isSingleton()) {
               sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                  @Override
                  public Object getObject() throws BeansException {
                     try {
                        // 创建bean
                        return createBean(beanName, mbd, args);
                     }
                  }
               });
               bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
           // 省略部分代码
      }
      return (T) bean;
   }
   ```
   
   createBean方法实现
   
   ```java
   protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
      // 省略部分代码
      // 这是最重要的方法，创建bean
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isDebugEnabled()) {
         logger.debug("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   ```
   
   doCreateBean方法实现
   
   ```java
   protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
         throws BeanCreationException {
   
      // bean包装类
      BeanWrapper instanceWrapper = null;
      if (mbd.isSingleton()) {
         instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
      }
      if (instanceWrapper == null) {
         // 这里仅仅创建了java对象，而没有走完bean的整个流程
         // 此方法很重要
         // 注意：如果是@Bean作用的对象，会在instantiateUsingFactoryMethod()中处理返回，而不会走实例化流程
         instanceWrapper = createBeanInstance(beanName, mbd, args);
      }
      // bean
      final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
      // beanType
      Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
      mbd.resolvedTargetType = beanType;
      synchronized (mbd.postProcessingLock) {
   			if (!mbd.postProcessed) {
   				try {
             // postProcessor处理，例如：对@Autowired、@Resource的处理
             // 比如，调用AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition()方法
             // 具体处理请看【@Autowired的实现原理】章节
   					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
   				}
   				catch (Throwable ex) {
   					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
   							"Post-processing of merged bean definition failed", ex);
   				}
   				mbd.postProcessed = true;
   			}
   		}
   	 // bean是否支持循环依赖且bean正在创建中，如果支持则将bean提前暴露出来
      boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
      // 支持循环依赖
      if (earlySingletonExposure) {
         // 如果支持循环依赖，则将实例化后，注入属性之前的对象包装成factory，放到三级缓存中，同时从二级缓存中移除
         // 此方法主要是为了解决AOP循环依赖的问题，普通的循环依赖，三级缓存是没用的
         // 此方法逻辑：
         // 1. 如果一级缓存没有此bean，则放入三级缓存，同时从二级缓存移除。反之，一级缓存有，则不做任何操作
         // 2. 那么下次获取bean的时候会直接从三级缓存的工厂创建代理对象
         // 所以：普通的循环依赖，三级缓存是没用的
         // 流程走到这里，其实Spring是不知道当前的bean是否与其他bean有循环依赖，所以干脆都放到三级缓存中
         addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
               // 如果是AOP代理对象，这里是调用AbstractAutoProxyCreator#getEarlyBeanReference()方法,返回代理对象
               // 如果不是，则调用的是InstantiationAwareBeanPostProcessorAdapter#getEarlyBeanReference()方法，什么也不做，返回传入的bean
               return getEarlyBeanReference(beanName, mbd, bean);
            }
         });
      }
     Object exposedObject = bean;
     try {
         // 1.判断是否需要属性提供
         // 2.属性填充（依赖注入）
         // 3.此方法很重要
         // 4.@Autowired和@Resource的属性填充就是在这里完成的
         // 4.1 调用InstantiationAwareBeanPostProcessor#postProcessPropertyValues()方法
         // 5.循环依赖请查看【如何解决循环依赖】章节
   			populateBean(beanName, mbd, instanceWrapper);
   			if (exposedObject != null) {
           // 对象创建后的初始化动作
           // 执行方法及顺序如下：
           // 1.invokeAwareMethods()方法,主要三个Aware的实现，设置钩子方法的值
           // 		1.1 BeanNameAware，BeanFactoryAware，BeanClassLoaderAware
           // 2.BeanPostProcessor.before()方法
           // 3.invokeInitMethods()方法，也就是@PostConstruct修饰的方法、InitializingBean、init()方法
           // 4.BeanPostProcessor.after方法，如AOP等
           // 5.原理可参考【AOP实现原理】章节
   				exposedObject = initializeBean(beanName, exposedObject, mbd);
   			}
   		}
   	 // 支持循环依赖
      if (earlySingletonExposure) {
         // 获取循环依赖的对象
         Object earlySingletonReference = getSingleton(beanName, false);
         if (earlySingletonReference != null) {
            // 如果相等，则直接返回即可
            // 因为在BeanPostProcessor或者InitializingBean中，可以对Bean进行二次开发，可能会对Bean做更改，比如对class做修改或者做代理
            if (exposedObject == bean) {
               exposedObject = earlySingletonReference;
            }
           // 如果不相等，当前这个Bean被其他Bean当做字段使用了（有属性的依赖）
   				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
   		    	// 拿到依赖这个bean的所有bean
             String[] dependentBeans = getDependentBeans(beanName);
             Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
             for (String dependentBean : dependentBeans) {
                 // 如果存在已经创建完的bean（已经创建完的bean依赖该bean）
                 // 放到list中
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                 actualDependentBeans.add(dependentBean);
               }
             }
             // tip:如果真的存在，那么就会报错，为什么呢？下面会说 
             if (!actualDependentBeans.isEmpty()) {
               throw new BeanCurrentlyInCreationException(beanName,
                   "Bean with name '" + beanName + "' has been injected into other beans [" +
                   StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                   "] in its raw version as part of a circular reference, but has eventually been " +
                   "wrapped. This means that said other beans do not use the final version of the " +
                   "bean. This is often the result of over-eager type matching - consider using " +
                   "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
             }
           }
      	}
      // 注册bean的销毁，将bean放入需要销毁的map中，供后续工厂销毁时调用
      try {
         registerDisposableBeanIfNecessary(beanName, bean, mbd);
      }
      return exposedObject;
   }
   ```
   
   我们来设想一下，有A、B两个类互相循环引用。
   创建A的过程是这样的
   A->B （创建A，必须先创建B）
   B->A（创建B，又必须先创建A，因为A的引用已经提前暴露了，假设对象号为@1000）
   此时B创建完成，B中的对象A为@1000
   现在A可以继续初始化了（initializeBean），很不碰巧的是，A在这里居然被改变了，变成了一个代理对象，对象号为@1001
   然后到了第二个处理earlySingletonExposure的地方，发现从缓存中拿到的对象和当前对象不相等了（@1000 != @1001）
   接着就看一下是否有依赖A的Bean创建完成了，哎，发现还真的有，那就是B
   然后想啊，B中的A和现在初始化完的A它不一样啊，这个和单例的性质冲突了！所以，必定得报错！
   
   
   
   getSingleton实现
   
   ```java
   protected Object getSingleton(String beanName, boolean allowEarlyReference) {
     // 从单例池中获取对象 
     Object singletonObject = this.singletonObjects.get(beanName);
      // 如果获取不到，且这个bean正在创建中
      if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
         synchronized (this.singletonObjects) {
            // 从正在创建中的bean集合中获取此bean
            singletonObject = this.earlySingletonObjects.get(beanName);
            // 如果获取不到，表示这个bean已经被提前暴露出去了
            // 在doCreateBean()#addSingletonFactory()方法中会做这件事
            if (singletonObject == null && allowEarlyReference) {
               // 从三级缓存中获取，三级缓存会调用getObject()方法创建bean，此时bean还没有完成属性的依赖注入
               // 将创建好的bean放入二级缓存，同时从三级缓存中移除
               ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
               if (singletonFactory != null) {
                  singletonObject = singletonFactory.getObject();
                  this.earlySingletonObjects.put(beanName, singletonObject);
                  this.singletonFactories.remove(beanName);
               }
            }
         }
      }
      // 如果获取到了，直接返回即可
      return (singletonObject != NULL_OBJECT ? singletonObject : null);
   }
   ```

#### 创建

##### Spring什么时候创建对象？

1. 当对象作用域scope==singleton时，工厂创建的同时创建对象
2. 当对象作用域scope==prototype时，getBean时才会创建对象
3. 注：上述2种情况，@Lazy(false)或者lazy-init=false时才能生效，不然都是getBean的时候才会创建

示例：

```java
@Component
public class User {
    // 构造方法，User被创建会执行
    public User() {
        System.out.println("User.User");
    }
}

// 1.工厂创建的同时实例化Bean
public void test1(){
  // 创建工厂，伪代码
  ApplicationContext axt = new AnnotationConfigApplicationContext(App.class);
}
// 结果输出：
// User.User

// 2.1 getBean时才会实例化bean，注:需设置@Scope为true
public void test1(){
  // 创建工厂，伪代码
  ApplicationContext axt = new AnnotationConfigApplicationContext(App.class);
}
// 2.1示例无结果输出
// 2.2 加上getBean
public void test1(){
  // 创建工厂，伪代码
  ApplicationContext axt = new AnnotationConfigApplicationContext(App.class);
  axt.getBean(User.class);
}
// 结果输出：
// User.User
```



#### 初始化

Spring在对象创建后，调用对应对象的初始化方法，完成初始化操作，有如下几种方式

##### 1.InitializingBean

```java
public class MyInitializingBean implements InitializingBean {
		// 重写afterPropertiesSet
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("do init somthing");
    }
}
```



##### 2.提供init方法

此方式只需要提供普通的初始化方法

```java
// 1.提供初始化方法
public void MyInit(){
    System.out.println("do init somthing");
}
// 2.配置初始化方法
// <bean id="user" calss="com.test.User" init-method="MyInit"/>
```



##### 3.@PostConstruct

将此注解加到初始化方法上

```java
@PostConstruct
public void MyInit(){
    System.out.println("do init somthing");
}
```



**以上三种方式若同时存在，则按 @PostConstruct -> InitializingBean -> init() 顺序执行**



#### 销毁

Spring销毁对象前，会调用对象的销毁方法，完成销毁操作

**注：销毁方法只作用于scope==singleton**

```java
// 工厂close方法被调用时候销毁所有对象
AnnotationConfigApplicationContext axt = new AnnotationConfigApplicationContext(App.class);
axt.close();
```



##### 1.DisposableBean

```java
public class MyDestroyBean implements DisposableBean {
  // 实现销毁方法
    @Override
    public void destroy() throws Exception {
        System.out.println("do destroy somthing");
    }
}
```



##### 2.提供destroy方法

此方式只需要提供普通的销毁方法

```java
// 1.提供销毁方法
public void myDestroy(){
    System.out.println("do destroy somthing");
}

// 2.配置销毁方法
// <bean id="user" calss="com.test.User" destroy-method="myDestroy"/>
```



**如果以上2种方法同时存在，则按DisposableBean -> destroy() 顺序执行**



### BeanPostProcessor

Spring非常重要的一个扩展点，俗称**后置处理器**，对Spring工厂创建的Bean进行再次加工，Spring中许多地方都用到了，比如对@Autowired的解析，AOP的底层实现等等

**注意：5.0版本开始，这2个方法变成了default方法，默认return bean，而且它会对所有Spring创建的对象进行加工，所以在使用时记得做自己的逻辑判断，如果返回Null，会报错**

示例：

```java
// 重要：需要托管给Spring，所以加上@Component，也可以配置<bean/>标签
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    // 该方法在对象创建后、初始化之前执行
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        // 对bean做一些加工的操作
      	if (bean instanceof User){
          User user = (User) bean;
      		user.setName("张三");
        }
        return bean;
    }
    // 该方法在对象初始化之后执行
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        // 对bean做一些加工的操作
      	if (bean instanceof User){
          User user = (User) bean;
      		user.setName("李四");
        }
        return bean;
    }
}
```



### BeanFactoryPostProcessor

Spring提供的一个在Bean实例化之前的扩展点，在扫描之后执行，如果开发者想做一些个性化的开发，比如对某些BeanDefinition做一些修改、将自己的一些对象托管给Spring等等之类的，就可以实现此接口

```java
public interface BeanFactoryPostProcessor {

   // 实现此方法，Spring提供了beanFactory给开发者
   // 此方法会在bean实例化之前调用，更准确的说是在bean被new出来之前调用
   void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}


// 例子
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 修改user的id
        DefaultListableBeanFactory bf = (DefaultListableBeanFactory) beanFactory;
        BeanDefinition us = bf.getBeanDefinition("user");
        us.getPropertyValues().add("id", "123456789");
      
      	// 这里也可以注册bean
        // bf.registerBeanDefinition("test", new RootBeanDefinition());
    }
}

```



### BeanDefinitionRegistryPostProcessor

继承BeanFactoryPostProcessor，除了父类提供的postProcessBeanFactory()方法之外，本身还提供了postProcessBeanDefinitionRegistry()方法，在扫描之前执行

**注：postProcessBeanDefinitionRegistry()会比postProcessBeanFactory()先执行**

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
	 // 实现此方法，开发者可以将自己的一些对象托管给Spring
   // 其实，父类的postProcessBeanFactory()方法也可以实现这个功能，不过2个方法的执行先后不一样
   // 所以，需要开发者自己按实际需求实现不同的接口来玩
   void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}

// 例子
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        // 将自己的Product对象托管给Spring
        // 其实就是将Product封装成beanDefinition，这里只是做了一些简单的设置
      	RootBeanDefinition beanDefinition = new RootBeanDefinition(Product.class);
        beanDefinition.getPropertyValues().add("name", "哇哇");
        // 注册bean
        registry.registerBeanDefinition("product", beanDefinition);
        System.out.println("Testaaaaa.postProcessBeanDefinitionRegistry");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 这里是BeanFactoryPostProcessor的方法，上文有介绍，这里不做阐述
    }
}
```



### @ComponentScan

1. 通过ConfigurationClassPostProcessor#doProcessConfigurationClass()方法加载
2. ConfigurationClassPostProcessor实现了BeanFactoryPostProcessor#postProcessBeanFactory()方法
3. 在postProcessBeanFactory()实现中调用ConfigurationClassParser#parse()方法
4. ConfigurationClassParser内部调用doProcessConfigurationClass()方法，这一步扫描@ComponentScans和@ComponentScan注解
5. 扫描后，调用ComponentScanAnnotationParser#parse()方法
6. 然后调用ClassPathBeanDefinitionScanner#doScan()方法，解析@ComponentScan配置的basePackages
7. 扫描basePackages下的所有类，获取比如使用了@Service，@Controller，@Component等注解的类
8. 将这些类封装成BeanDefinition，调用托管给spring的beanDefinitionMap
9. 具体原理可参考【ConfigurationClassPostProcessor】章节源码分析



### @Autowired实现原理

在对象创建/构造完成后
通过AutowiredAnnotationBeanPostProcessor后置处理器，解析使用了@Autowired的方法或者字段，封装成AutowiredMethodElement、AutowiredFieldElement，然后进行属性注入

在**【实例化bean的步骤】**章节，doCreateBean方法中的applyMergedBeanDefinitionPostProcessors()方法对@Autowired进行了处理

```java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
   // 调用所有MergedBeanDefinitionPostProcessor的实现类
   // AutowiredAnnotationBeanPostProcessor就是其中之一
   for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof MergedBeanDefinitionPostProcessor) {
         MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
         bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
      }
   }
}
```

**AutowiredAnnotationBeanPostProcessor**

```java
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
   if (beanType != null) {
      // 处理@Autowired
      // 其内部调用了buildAutowiringMetadata()方法，针对使用了@Autowired的方法和字段分别进行处理
      InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
      metadata.checkConfigMembers(beanDefinition);
   }
}
```

**AutowiredAnnotationBeanPostProcessor#buildAutowiringMetadata()**

```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
   LinkedList<InjectionMetadata.InjectedElement> elements = new LinkedList<InjectionMetadata.InjectedElement>();
   Class<?> targetClass = clazz;

   do {
      final LinkedList<InjectionMetadata.InjectedElement> currElements =
            new LinkedList<InjectionMetadata.InjectedElement>();
			// 对使用了@Autowired的字段进行处理，通过反射，封装成AutowiredFieldElement
      ReflectionUtils.doWithLocalFields(targetClass, new ReflectionUtils.FieldCallback() {
         @Override
         public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
            AnnotationAttributes ann = findAutowiredAnnotation(field);
            if (ann != null) {
               // 注意：如果字段是静态的，则不支持注入
               if (Modifier.isStatic(field.getModifiers())) {
                  if (logger.isWarnEnabled()) {
                     logger.warn("Autowired annotation is not supported on static fields: " + field);
                  }
                  return;
               }
               boolean required = determineRequiredStatus(ann);
               // 缓存起来，供后续注入属性使用
               currElements.add(new AutowiredFieldElement(field, required));
            }
         }
      });
			
     	// 对使用了@Autowired的字段进行处理，通过反射，封装成AutowiredMethodElement
      ReflectionUtils.doWithLocalMethods(targetClass, new ReflectionUtils.MethodCallback() {
         @Override
         public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {
            Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
            if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
               return;
            }
            AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
            if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                // 注意：如果方法是静态的，则不支持注入
              	if (Modifier.isStatic(method.getModifiers())) {
                  if (logger.isWarnEnabled()) {
                     logger.warn("Autowired annotation is not supported on static methods: " + method);
                  }
                  return;
               }
               // 如果方法的参数为空，则不能注入（没入参，注入个毛啊）
               if (method.getParameterTypes().length == 0) {
                  if (logger.isWarnEnabled()) {
                     logger.warn("Autowired annotation should only be used on methods with parameters: " +
                           method);
                  }
               }
               boolean required = determineRequiredStatus(ann);
               PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
               // 缓存起来，供后续注入属性使用
               currElements.add(new AutowiredMethodElement(method, required, pd));
            }
         }
      });

      elements.addAll(0, currElements);
      // 递归父类，继续找@使用了@Autowired的字段和方法
      targetClass = targetClass.getSuperclass();
   }
   while (targetClass != null && targetClass != Object.class);、
   return new InjectionMetadata(clazz, elements);
}
```

**AutowiredMethodElement和AutowiredFieldElement都是继承InjectionMetadata，且实现了inject()方法**

在doCreateBean方法中，调用AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition()执行完后，继续往下走，会执行populateBean()方法进行属性的依赖注入，此方法中会调用InstantiationAwareBeanPostProcessor#postProcessPropertyValues()进行属性注入

AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor都实现了此接口，所以@Autowired和@Resource的注入都会在此方法中处理

```java
public PropertyValues postProcessPropertyValues(
      PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
	 // 从缓存获取元数据
   InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
   try {
      // 属性注入，细节这里不阐述了，太多了
      metadata.inject(bean, beanName, pvs);
   }
   catch (BeanCreationException ex) {
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
   }
   return pvs;
}
```







**@Resource同理，由CommonAnnotationBeanPostProcessor处理，它也实现了MergedBeanDefinitionPostProcessor**



### ConfigurationClassPostProcessor

ConfigurationClassPostProcessor主要功能是处理各种注解，如@Import、@ImportResource、@Bean、@ComponentScan和@ComponentScans注解

```java
// 实现了BeanDefinitionRegistryPostProcessor
// 实现了2个方法：postProcessBeanDefinitionRegistry和postProcessBeanFactory，这里只介绍前者
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
      PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
  
    // 注册bean到BeanDefinitionMap
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
      int registryId = System.identityHashCode(registry);
      if (this.registriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
            "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
      }
      if (this.factoriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + registry);
      }
      this.registriesPostProcessed.add(registryId);
      // 入口在这
      processConfigBeanDefinitions(registry);
    }
}
```

**ConfigurationClassPostProcessor#processConfigBeanDefinitions()**

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
   // 省略部分代码
   do {
      // 解析各种注解，包含@Bean，主要逻辑在这里
      parser.parse(candidates);
     
      // 省略部分代码
     
     	// 这里对解析出来的BeanDefinition做处理
      // 在上面的parse()方法中，对@Bean方法已经做处理了
      this.reader.loadBeanDefinitions(configClasses);
     
      // 省略部分代码
   }
   while (!candidates.isEmpty());
   // 省略部分代码
}
```

**ConfigurationClassParser#parse()**

底层调用内部方法：ConfigurationClassParser()

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
      throws IOException {

   // 处理@PropertySource注册
   // Process any @PropertySource annotations
   for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
         sourceClass.getMetadata(), PropertySources.class,
         org.springframework.context.annotation.PropertySource.class)) {
      if (this.environment instanceof ConfigurableEnvironment) {
         processPropertySource(propertySource);
      }
      else {
         logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
               "]. Reason: Environment must implement ConfigurableEnvironment");
      }
   }
	 // 处理@ComponentScan和ComponentScans注解
   // Process any @ComponentScan annotations
   Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
         sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
   if (!componentScans.isEmpty() &&
         !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
      // 循环处理
      for (AnnotationAttributes componentScan : componentScans) {
         // 扫描到使用了@Component、@Service、@Controller等注册的类
         Set<BeanDefinitionHolder> scannedBeanDefinitions =
               this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
         // Check the set of scanned definitions for any further config classes and parse recursively if needed
         for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
            BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
            if (bdCand == null) {
               bdCand = holder.getBeanDefinition();
            }
            if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
               parse(bdCand.getBeanClassName(), holder.getBeanName());
            }
         }
      }
   }
		
   // 处理@Import注解
   // Process any @Import annotations
   processImports(configClass, sourceClass, getImports(sourceClass), true);

   // 处理@ImportResource注解
   // Process any @ImportResource annotations
   if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
      AnnotationAttributes importResource =
            AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
      String[] resources = importResource.getStringArray("locations");
      Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
      for (String resource : resources) {
         String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
         configClass.addImportedResource(resolvedResource, readerClass);
      }
   }
	
   // 处理@Bean作用的方法，添加到Set<BeanMethod>中
   // Process individual @Bean methods
   Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
   for (MethodMetadata methodMetadata : beanMethods) {
      configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
   }

   // 处理接口的默认方法
   // Process default methods on interfaces
   processInterfaces(configClass, sourceClass);

   // Process superclass, if any
   if (sourceClass.getMetadata().hasSuperClass()) {
      String superclass = sourceClass.getMetadata().getSuperClassName();
      if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
         this.knownSuperclasses.put(superclass, configClass);
         // Superclass found, return its annotation metadata and recurse
         return sourceClass.getSuperClass();
      }
   }

   // No superclass -> processing is complete
   return null;
}
```



**ConfigurationClassBeanDefinitionReader#loadBeanDefinitions()**

```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
   TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
   for (ConfigurationClass configClass : configurationModel) {
      // 其底层最终调用ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForBeanMethod()
      loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
   }
}
```

**ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForBeanMethod()**

```java
private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
   ConfigurationClass configClass = beanMethod.getConfigurationClass();
   MethodMetadata metadata = beanMethod.getMetadata();
   // 获取@Bean对象的方法名
   String methodName = metadata.getMethodName();

   // Do we need to mark the bean as skipped by its condition?
   // 是否需要根据配置的condition条件跳过
   if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
      configClass.skippedBeanMethods.add(methodName);
      return;
   }
   if (configClass.skippedBeanMethods.contains(methodName)) {
      return;
   }

   // 获取@Bean的注解配置，如：autowire、destroyMethod、name、initMethod等等
   AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);
   List<String> names = new ArrayList<String>(Arrays.asList(bean.getStringArray("name")));
   // bena的名称，优先取：自定义名称
   String beanName = (!names.isEmpty() ? names.remove(0) : methodName);

   // 注册别名
   for (String alias : names) {
      this.registry.registerAlias(beanName, alias);
   }

   // Has this effectively been overridden before (e.g. via XML)?
   if (isOverriddenByExistingDefinition(beanMethod, beanName)) {
      if (beanName.equals(beanMethod.getConfigurationClass().getBeanName())) {
         throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),
               beanName, "Bean name derived from @Bean method '" + beanMethod.getMetadata().getMethodName() +
               "' clashes with bean name for containing configuration class; please make those names unique!");
      }
      return;
   }
	 // ConfigurationClassBeanDefinition
   ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
   beanDef.setResource(configClass.getResource());
   beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));
	 
   // 如果@Bean描述的方法是static的
   if (metadata.isStatic()) {
      // static @Bean method
      // 设置BeanClassName为@Configuration描述类的className
      beanDef.setBeanClassName(configClass.getMetadata().getClassName());
      // 设置工厂方法
      beanDef.setFactoryMethodName(methodName);
   }
   else {
      // instance @Bean method
      // 设置FactoryBeanName，就是@Configuration描述类的beanName
      beanDef.setFactoryBeanName(configClass.getBeanName());
      // 设置工厂方法
      beanDef.setUniqueFactoryMethodName(methodName);
   }
   // 设置注入模型，默认为：构造方法注入
   beanDef.setAutowireMode(RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
   // 设置属性
   beanDef.setAttribute(RequiredAnnotationBeanPostProcessor.SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

   AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);
	 // 设置注入模型
   Autowire autowire = bean.getEnum("autowire");
   if (autowire.isAutowire()) {
      beanDef.setAutowireMode(autowire.value());
   }
	 // 初始化方法
   String initMethodName = bean.getString("initMethod");
   if (StringUtils.hasText(initMethodName)) {
      beanDef.setInitMethodName(initMethodName);
   }
	 // 销毁方法
   String destroyMethodName = bean.getString("destroyMethod");
   if (destroyMethodName != null) {
      beanDef.setDestroyMethodName(destroyMethodName);
   }

   // 作用域
   ScopedProxyMode proxyMode = ScopedProxyMode.NO;
   AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
   if (attributes != null) {
      beanDef.setScope(attributes.getString("value"));
      proxyMode = attributes.getEnum("proxyMode");
      if (proxyMode == ScopedProxyMode.DEFAULT) {
         proxyMode = ScopedProxyMode.NO;
      }
   }

   // Replace the original bean definition with the target one, if necessary
   BeanDefinition beanDefToRegister = beanDef;
   if (proxyMode != ScopedProxyMode.NO) {
      BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
            new BeanDefinitionHolder(beanDef, beanName), this.registry,
            proxyMode == ScopedProxyMode.TARGET_CLASS);
      beanDefToRegister = new ConfigurationClassBeanDefinition(
            (RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);
   }

   if (logger.isDebugEnabled()) {
      logger.debug(String.format("Registering bean definition for @Bean method %s.%s()",
            configClass.getMetadata().getClassName(), beanName));
   }
	 // 最终，注册到BeanDefinitionMap
   this.registry.registerBeanDefinition(beanName, beanDefToRegister);
}
```

**注：如果有2个返回相同类型对象的方法使用了@Bean，会报错，因为getBean的时候不知道用哪个，如果再其中一个方法上加上了@Primary，则getBean会返回此bean，如果2个方法加上了@Primary，则报错，因为因为getBean的时候不知道用哪个**





### 如何解决循环依赖

#### 什么是循环依赖？

比如有2个类，A和B，互相引用
首先A被创建，经过一系列流程，到自动注入时获取B这个bean(getBean(B))
但此时B这个bean并没有被实例化(signleObjects中没有)，于是就会创建B，B也经过一系列流程，到自动注入时获取A(getBean(A))，发现A也没有被实例化(signleObjects中没有)，于是A又去找B...死循环



#### 解决方法？

比如有2个类，A和B，互相引用

1.  首先A被创建，此时A会被放到singletonsCurrentlyInCreation这个set集合中，表示该对象是在创建过程
2.  经过一系列流程，到自动注入时获取B(getBean(B)),此时B并没有被实例化(signleObjects中没有)，就会创建B
3.  B经过一系列流程，到自动注入时获取A(getBean(A))，发现A正在创建过程中(isSingletonCurrentlyInCreation(A)为ture)，于是从earlySingletonObjects(存放对象集合，map)中获取到A，如果取不到A，则从SignletonFactory.getObject()取A，因为实例化B并不需要A这个bean，只需要A对象
4.  此时B就能被实例化成bean，A接下来获取到B也能实例化



- **singletonObjects：存放真正实例化的bean。1级缓存**

- **earlySingletonObjects：存放早期的bean对象，此时还不是真正的bean。2级缓存**

- **SignletonFactory：存放工厂，解决aop对象循环依赖的问题，比如A和B循环依赖，A被aop，当A注入属性时获取B，B也能被Aop代理。3级缓存**

- **以上3个缓存，都存放在DefaultSingletonBeanRegistry中**



#### 为什么要使用三级缓存呢？二级缓存能解决循环依赖吗？

如果要使用二级缓存解决循环依赖，意味着**所有Bean在实例化后就要完成AOP代理**，这样违背了Spring设计的原则，Spring在设计之初就是通过`AnnotationAwareAspectJAutoProxyCreator`这个后置处理器来在Bean生命周期的最后一步来完成AOP代理，而不是在实例化后就立马进行AOP代理



#### 为什么构造器注入循环依赖会报错？

例子1：

```java
@Component
public class TestService {

   private UserService userService;
	
   @Autowired
   public TestService(UserService userService){
       this.userService = userService;
   }
}

@Component
public class UserService {

   private TestService testService;
	
   @Lazy // 加上延迟加载，即可解决
   @Autowired
   public UserService(TestService testService){
       this.testService = testService;
   }
}
// @Autowired注解可加可不加
public void test() {
   ApplicationContext applicationContext =  new AnnotationConfigApplicationContext(ConfigTest.class);
}

报错信息
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'testService': Requested bean is currently in creation: Is there an unresolvable circular reference?
....
```



原因：

通过构造器注入，在实例化A时，会调用构造方法反射创建对象A，而在创建对象A之前会先实例化构造器传入的参数对象B，这个时候A还没创建，所以不会放入二级缓存，同理，在实例化B的时候，同样也会先实例化构造器传入的参数对象A，此时B也没有创建出来，也不会放入二级缓存，所以会一直这样死循环

而filed注入和Set注入不同的是，A对象会先通过构造器创建对象后，放入二级缓存，再进入属性=B对象的注入，当B去实例化，对A进行属性注入的时候，会从二级缓存取出A的实例注入，所以是没有问题的

**如果要解决，加上@Lazy即可**



### BeanDefinition

用来描述bean（因为JDK的Class对象无法满足），包含了比如是否需要懒加载isLazy、class、注入模型autoModel(byType,byName)、父类的名称parentName、是否抽象类isAbstract等、beanName、FactoryMethod等等字段

```java
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
        implements BeanDefinition, Cloneable {
        
    // bean的class
    @Nullable
    private volatile Object beanClass;
    // bean的作用域，比如单例，原型，不同的作用域处理方法不一样，默认：单例singleton
    // 这里默认是空字符串，但是Spring会将此处理为默认单例singleton
    @Nullable
    private String scope = SCOPE_DEFAULT;
    /**
     * 是否是抽象，对应bean属性abstract
     */
    private boolean abstractFlag = false;
    /**
     * 是否延迟加载，对应bean属性lazy-init
     */
    private boolean lazyInit = false;
    /**
     * 自动注入模式，对应bean属性autowire
     */
    private int autowireMode = AUTOWIRE_NO;
    /**
     * 依赖检查，Spring 3.0后弃用这个属性
     */
    private int dependencyCheck = DEPENDENCY_CHECK_NONE;
    /**
     * 用来表示一个bean的实例化依靠另一个bean先实例化，对应bean属性depend-on
     */
    @Nullable
    private String[] dependsOn;
    /**
     * autowire-candidate属性设置为false，这样容器在查找自动装配对象时，
     * 将不考虑该bean，即它不会被考虑作为其他bean自动装配的候选者，
     * 但是该bean本身还是可以使用自动装配来注入其他bean的
     */
    private boolean autowireCandidate = true;
    /**
     * 自动装配时出现多个bean候选者时，将作为首选者，对应bean属性primary
     */
    private boolean primary = false;
    /**
     * 用于记录Qualifier，对应子元素qualifier
     */
    private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>(0);

    @Nullable
    private Supplier<?> instanceSupplier;
    /**
     * 允许访问非公开的构造器和方法，程序设置
     */
    private boolean nonPublicAccessAllowed = true;
    /**
     * 是否以一种宽松的模式解析构造函数，默认为true，
     * 如果为false，则在以下情况
     * interface ITest{}
     * class ITestImpl implements ITest{};
     * class Main {
     *     Main(ITest i) {}
     *     Main(ITestImpl i) {}
     * }
     * 抛出异常，因为Spring无法准确定位哪个构造函数程序设置
     */
    private boolean lenientConstructorResolution = true;
    /**
     * 对应bean属性factory-bean，用法：
     * <bean id = "instanceFactoryBean" class = "example.chapter3.InstanceFactoryBean" />
     * <bean id = "currentTime" factory-bean = "instanceFactoryBean" factory-method = "createTime" />
     */
    @Nullable
    private String factoryBeanName;
    /**
     * 对应bean属性factory-method
     */
    @Nullable
    private String factoryMethodName;
    /**
     * 记录构造函数注入属性，对应bean属性constructor-arg
     */
    @Nullable
    private ConstructorArgumentValues constructorArgumentValues;
    /**
     * 普通属性集合
     */
    @Nullable
    private MutablePropertyValues propertyValues;
    /**
     * 方法重写的持有者，记录lookup-method、replaced-method元素
     */
    @Nullable
    private MethodOverrides methodOverrides;
    /**
     * 初始化方法，对应bean属性init-method
     */
    @Nullable
    private String initMethodName;
    /**
     * 销毁方法，对应bean属性destroy-method
     */
    @Nullable
    private String destroyMethodName;
    /**
     * 是否执行init-method，程序设置
     */
    private boolean enforceInitMethod = true;
    /**
     * 是否执行destroy-method，程序设置
     */
    private boolean enforceDestroyMethod = true;
    /**
     * 是否是用户定义的而不是应用程序本身定义的，创建AOP时候为true，程序设置
     */
    private boolean synthetic = false;
    /**
     * 定义这个bean的应用，APPLICATION：用户，INFRASTRUCTURE：完全内部使用，与用户无关，
     * SUPPORT：某些复杂配置的一部分
     * 程序设置
     */
    private int role = BeanDefinition.ROLE_APPLICATION;
    /**
     * bean的描述信息
     */
    @Nullable
    private String description;
    /**
     * 这个bean定义的资源
     */
    @Nullable
    private Resource resource;
}
```



### 关闭是否允许循环依赖


```
//默认为ture，设置为false关闭
AbstractAutowireCapableBeanfactory.allowcularReferences=false;
```



### 生命周期回调

```
1.bean实例化后（new 对象后）初始化
  1.1 实现InitializingBean.afterPropertiesSet()
  1.2 <bean id='xxx' init='init'>
  1.3 使用@PostConstruct作用于方法上
  注：1.1与1.2是调用InvokeInitMethods()，1.3是在BeanPostProcessor实现
2.bean实例化后销毁
  2.1 实现DisposableBean.destroy()
  2.2 <bean id='xxx' destroy-method='destroy'>
  2.3 使用@PreDestroy作用于方法上
```



### 核心类

- InitializingBean：bean初始化方法
- ApplicationContenxt：beanfactory的实现类，更多功能
- DefaultSingletonBeanRegistry 
      signleObjects:存放单例bean
      earlySingletonObjects:存放创建中的bean
      创建bean 等等功能
- AbstractBeanFactory 最终实现BeanFactory 创建、获取、销毁bean
- BeanPostProcessor 增强bean创建过程（对象初始化后）

- BeanFactoryPostProcessor 可实现方法，可以随心所欲修改所有beandefinition（对象初始化前）
- JDKDynamicAopProxy jdk动态代理类
- cglibAopProxy cglib动态代理类
- FactoryBean 特殊bean，实现FactoryBean.getObject() 会注册设置的bean，同时FactoryBean实现类本身也会注册为bean，beanName格式为：&+beanName，如果直接用beanName获取bean会返回在getObject()设置的bean。

- NameSpaceHandler：自定义标签扩展类



### FactoryBean

它是一个特殊的Bean，用于创建复杂对象，本质上是接口回调，实现FactoryBean.getObject() 会注册设置的bean，同时FactoryBean实现类本身也会注册为bean，beanName格式为：&+beanName，如果直接用beanName获取bean会返回在getObject()设置的bean

Mybatis整合Spring就是这么玩的，参考MapperFactoryBean



#### 为什么通过上下文getBean(beanName)获取到的是FactoryBean.getObject()返回的对象？为什么要通过getBean(&+beanName)才能获取到FactoryBean对象

这是因为Spring在实例化FactoryBean时，会判断是否是其实现，如果是，则会将BeanName特殊处理，前面会加上&符号

getBean(beanName)逻辑：

1. Object obj = applicationContext.getBean("object")
2. 判断是否FactoryBean实现，否：走doCreateBean逻辑，是：调用FactoryBean.getObject()方法获取对象，实例化



#### 示例

```java
/**
 * 自定义FactoryBean
* @author zhangjianfeng 2021/1/8
*/
public class MyFactoryBean implements FactoryBean<User> {
    // 返回自定义FactoryBean产生的对象
    @Override
    public User getObject() throws Exception {
        User user = new User();
        user.setId("factoryBean test");
        return user;
    }
  
    // 返回自定义FactoryBean产生对象的class
    @Override
    public Class<?> getObjectType() {
        return User.class;
    }

    // 是否单例
    @Override
    public boolean isSingleton() {
        return true;
    }
}
```



## @Configuration

配置Bean，Spring在3.x开始提供的注解，用于替换XML文件。它也是@Component注解的衍生（扩展），其内部使用了@Component，通过AOP生成Cglib代理对象



在使用方面，之前是这么配置的

```xml
<beans>
  <!-- import配置文件 -->
	<import resoure="classpath:/applicationContext.cml"/>
  <!-- 扫描bean，如果要配置包含策略，则需要干掉Spring的默认扫描方式，use-default-filter=false -->
  <context:compoent-scan base-package="com.test" use-default-filter="false">
    <!-- 排除使用了Component注解的类不被扫描 -->
    <context:exclue-filter type="annotation" expression="org.springframework.stereotype.Component"/>
    <!-- 只扫描使用了Service的类注解 -->
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Service"/>
    
  </context:compoent-scan>
  <!-- 配置bean -->
  <bean id="user" class="com.test.User"/>
  <bean id="product" class="com.test.Product"/>
</beans>
```

对应的工厂为：

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("/spring.xml");
```



使用@Configuration后

**注意：如果同时对一个类型，如对@Component 使用了包含策略和排除策略，则使用了@Component的类还是会被排除掉，其他类型同理**

```java
@Configuration
// 等同xml中的context:compoent-scan标签，用于扫描需要交给Spring托管的bean
// 比如使用了@value @Compoent @Service等等的类
// 1.排除使用了@Component注解的类，不被扫描
// 2.排除User类不被扫描
// 3.通过AOP切面表达式排除com.test及子包不被扫描
// 4.如果要使用包含策略，则需要干掉Spring默认扫描方式useDefaultFilters = false
// 5.只扫描使用了@Component注解的类
@ComponentScan(basePackages = "com",
        useDefaultFilters = false,
        excludeFilters = {
                @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, classes = User.class),
                @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Component.class),
                @ComponentScan.Filter(type = FilterType.ASPECTJ, pattern = {"com.test"})
        },
        includeFilters = {
                @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Component.class)
        })
public class Config{
  
  // 等同于xml中的bean标签，方法名=bean标签的id属性，返回类型=bean标签的Class属性
  @Bean
  // 是否主要的，如果设置了，则以此bean为主，覆盖其他同类型的bean
  // 比如，有2个@bean 返回user，如果其中1个使用了@Primary，则getBean的时候返回使用了@Primary的bean
  // 注：如果2个都使用了@Primary，则报错，因为getBean不知道用哪个，如果2个都没用@Primary，也会报错，因为getBean也不知道用哪个
  @Primary
  public User user(){
    return new User();
  }
}
```

对应工厂为：

```java
// 这种方式是直接加载配置类
ApplicationContext cxt = new AnnotationConfigApplicationContext(Config.class);
// 这种方式是扫描对应包下使用了@Configuration的配置类
ApplicationContext cxt = new AnnotationConfigApplicationContext("com.test");
```



## @ConfigurationProperties

属性文件配置，用于读取属性文件中一段内容

```java
@Data
@Component
// 读取下面配置文件中的serialnumber配置段落
@ConfigurationProperties("serialnumber")
public class SerialNumberProperties {
    private String bizCode;
    private String prefix;
}
```

属性文件

```properties
server:
	port: 8080
serialnumber:
  host: localhost:8888
  bizCode: test
  prefix: AP
```



## 有哪几种方式获取Spring上下文

1. 实现ApplicationContextAware

2. ApplicationContext ap = new ClassPathXmlApplicationContext("applicationContext.xml");

3. ApplicationContext axt = new AnnotationConfigApplicationContext(App.class);

4. Springboot可以通过启动类获取

   

   ```java
   public class MainApplication {
     public static void main(String[] args) {
         ConfigurableApplicationContext context = SpringApplication.run(MainApplication.class, args);
         SpringContextUtil3.setApplicationContext(context);
     }
   }
   ```
   

5. 直接注入

   ```java
   @Autowired
   public ApplicationContext applicationContext;
   ```

   





## AOP

Aop是一个标准，Spring AOP 是它的一种实现

### 使用

* `启用aop` @EnableAspectJAutoProxy

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

}
```

* `配置切面` @Aspect

```java
@Aspect
public class NotVeryUsefulAspect {

}
```

* `配置切点` @Pointcut

```java
@Aspect
public class NotVeryUsefulAspect {
    @Pointcut("execution(* transfer(..))") // 切点表达式
    private void anyOldTransfer() {} // 切点实现方法
}
```

* `配置通知` **Advice**

```java
@Aspect
public class BeforeExample {
    
    @Pointcut("execution(* transfer(..))") // 切点表达式（连接点集合）
    private void anyOldTransfer() { // 切点实现方法
        // 增强逻辑
    } 
    
    //在切点之前增强
    @Before("anyOldTransfer()")  //增强的方法名
    public void doAccessCheck() {
        // 增强逻辑
    }
    //在切点之后增强
    @After("anyOldTransfer()")  //增强的方法名
    public void doAccessCheck() {
        // 增强逻辑
    }

}
```



### 实现原理

实现BeanPostProcessor, 在ioc doCreateBean() 方法中调用实现BeanPostProcessor的实现类(AnnotationAwareAspectJAutoProxyCreator)，完成对bean的aop增强并返回增强的代理对象(bean)。

通过AopProxy 生成aop代理对象。
AopProxy 有2个实现类 JDKDynamicAopProxy 和  CglibAopProxy
通过bean，执行不同的代理方式

参考【实例化Bean的步骤】章节中，doCreateBean()方法中的initializeBean()中，实现了AOP的后置处理

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged(new PrivilegedAction<Object>() {
         @Override
         public Object run() {
            // 执行三个Aware回调方法
            invokeAwareMethods(beanName, bean);
            return null;
         }
      }, getAccessControlContext());
   }
   else {
      // 执行三个Aware回调方法
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      // 执行BeanPostProcessor.before方法
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      // 执行init方法
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
      // 重点：这里是执行BeanPostProcessor.after方法，AOP实现就是其中之一
      // AOP的BeanPostProcessor实现为：
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }
   return wrappedBean;
}
```

```java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
      throws BeansException {

   Object result = existingBean;
   // 循环所有的处理器，执行after方法，aop的处理实现为：AbstractAutoProxyCreator
   for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
      result = beanProcessor.postProcessAfterInitialization(result, beanName);
      if (result == null) {
         return result;
      }
   }
   return result;
}
```

继续看下AbstractAutoProxyCreator是怎么创建AOP代理对象的

AbstractAutoProxyCreator#postProcessAfterInitialization()

```java
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   if (bean != null) {
      // 先获取缓存，因为创建好代理对象后，会将代理对象缓存起来，不可能每次都创建代理
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         // 生成AOP代理对象逻辑在这
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

AbstractAutoProxyCreator#wrapIfNecessary()

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // 创建代理对象，前提是配置了advice增强
   // Create proxy if we have advice.
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      // 创建代理对象，通过ProxyFactory.getProxy()创建
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      // 将代理对象缓存起来
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }
   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```






## 事务


* `@EnableTranscationManagement` springboot1.4之后不用配置，系统会默认使用DataSourceTranscationManager

```
import(TransactionManagementConfigurationSelector)  

```

* `PlatformTranscationManager` 事务管理器，应用了模板方法设计模式

```
如：DataSrouceTranscationManager 

getTransaction(TransactionDefinition d)：获取当前事务

rollback(TransactionStatus s)：回滚

commit(TransactionStatus s)：提交

```

* `TransactionStatus`  事务状态

```

```

* `TransactionInterceptor` 事务拦截器

```
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass, final InvocationCallback invocation) throws Throwable {

		// 获取事务属性源~
		TransactionAttributeSource tas = getTransactionAttributeSource();
		// 获取该方法对应的事务属性（这个特别重要）
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);

		// 这个厉害了：就是去找到一个合适的事务管理器（具体策略详见方法~~~)
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		// 拿到目标方法唯一标识（类.方法，如service.UserServiceImpl.save）
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		// 如果txAttr为空或者tm 属于非CallbackPreferringPlatformTransactionManager，执行目标增强
		// 在TransactionManager上，CallbackPreferringPlatformTransactionManager实现PlatformTransactionManager接口，暴露出一个方法用于执行事务处理中的回调
		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
		
			// 看是否有必要创建一个事务，根据`事务传播行为`，做出相应的判断
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			
			Object retVal = null;
			try {
				//回调方法执行，执行目标方法（原有的业务逻辑）
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// 出现异常了，进行回滚（注意：并不是所有异常都会rollback的）
				// 备注：此处若没有事务属性   会commit 兼容编程式事务吧
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				//清除信息
				cleanupTransactionInfo(txInfo);
			}
	
			// 目标方法完全执行完成后，提交事务~~~
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		//编程式事务处理(CallbackPreferringPlatformTransactionManager) 会走这里 
		// 原理也差不太多，这里不做详解~~~~
		else {
			final ThrowableHolder throwableHolder = new ThrowableHolder();

			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr, status -> {
					TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
					try {
						return invocation.proceedWithInvocation();
					}
					catch (Throwable ex) {
						if (txAttr.rollbackOn(ex)) {
							// A RuntimeException: will lead to a rollback.
							if (ex instanceof RuntimeException) {
								throw (RuntimeException) ex;
							}
							else {
								throw new ThrowableHolderException(ex);
							}
						}
						else {
							// A normal return value: will lead to a commit.
							throwableHolder.throwable = ex;
							return null;
						}
					}
					finally {
						cleanupTransactionInfo(txInfo);
					}
				});

				// Check result state: It might indicate a Throwable to rethrow.
				if (throwableHolder.throwable != null) {
					throw throwableHolder.throwable;
				}
				return result;
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
			catch (TransactionSystemException ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
					ex2.initApplicationException(throwableHolder.throwable);
				}
				throw ex2;
			}
			catch (Throwable ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				}
				throw ex2;
			}
		}
	}

	// 从容器中找到一个事务管理器
	@Nullable
	protected PlatformTransactionManager determineTransactionManager(@Nullable TransactionAttribute txAttr) {
		// 如果这两个都没配置，所以肯定是手动设置了PlatformTransactionManager的，那就直接返回即可
		if (txAttr == null || this.beanFactory == null) {
			return getTransactionManager();
		}

		// qualifier 就在此处发挥作用了，他就相当于BeanName
		String qualifier = txAttr.getQualifier();
		if (StringUtils.hasText(qualifier)) {
			// 根据此名称 以及PlatformTransactionManager.class 去容器内招
			return determineQualifiedTransactionManager(this.beanFactory, qualifier);
		}
		// 若没有指定qualifier   那再看看是否指定了 transactionManagerBeanName
		else if (StringUtils.hasText(this.transactionManagerBeanName)) {
			return determineQualifiedTransactionManager(this.beanFactory, this.transactionManagerBeanName);
		}
		// 若都没指定，那就不管了。直接根据类型去容器里找 getBean(Class)
		// 此处：若容器内有两个PlatformTransactionManager ，那就铁定会报错啦~~~
		else {
			PlatformTransactionManager defaultTransactionManager = getTransactionManager();
			if (defaultTransactionManager == null) {
				defaultTransactionManager = this.transactionManagerCache.get(DEFAULT_TRANSACTION_MANAGER_KEY);
				if (defaultTransactionManager == null) {
					defaultTransactionManager = this.beanFactory.getBean(PlatformTransactionManager.class);
					this.transactionManagerCache.putIfAbsent(
							DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
				}
			}
			return defaultTransactionManager;
		}
	}
```

* `TranscationDefinition` 封装了事务的隔离级别，传播机制

```
传播机制：
1.PROPAGATION_REQUIRED  : 支持当前事务,有事务就用事务,没事务就新建个事务
如：今天下班回家有饭就吃饭，没饭就做饭吃

2.PROPAGATION_SUPPORTS : 支持当前事务,如果当前没有事务,就以非事务的方式执行
如：今天下班回家有饭吃就吃，没有就饿肚子

3.PROPAGATION_MANDATORY : 支持当前事务,如果当前没有事务,就抛异常
如：今天下班回家有饭就吃饭，没有就发脾气

4.PROPAGATION_REQUIRES_NEW : 新建事务,如果当前有事务,则挂起
如：今天下班回家有饭吃，不吃，非要自己做。比如A调用B方法，B传播机制是REQUIRES_NEW，那么B方法会自己新建个事务，不论A的逻辑是否异常都不影响B方法的逻辑

5.PROPAGATION_NOT_SUPPORTED : 以非事务方式执行,如果当前有事务,则挂起
如：互不影响，事务不生效

6.PROPAGATION_NEVER : 以非事务方式执行,如果当前有事务,抛异常
如：不支持事务

7.PROPAGATION_NESTED : 如果当前有事务,则嵌套事务内执行,反之新建一个事务执行

当前事务：谁调用了我 ，谁就是我的当前事务

隔离级别：
1.ISOLATION_DEFAULT:  使用数据库默认的隔离级别
2.ISOLATION_READ_UNCOMMITTED 读未提交: 允许读取未提交的更改,可能造成幻读,脏读,不可重复读
3.ISOLATION_READ_COMMITTED 读已提交: 允许从已提交的事务读取,可防止幻读,但可能会造成脏读或不可重复读
4.ISOLATION_REPEATABLE_READ 可重复度: 对相同字段多次读取的结果是一致的,除非数据被当前事务改变,可防止脏读和不可重复读,但可能发生幻读
5.ISOLATION_SERIALIZABLE 串行化: 不会发生幻读,脏读,不可重复读,但性能是最慢的(锁表完成操作) 
mysql默认REPEATABLE_READ
```

* `TransacationAttribute` 事务属性，实现对回滚规则的扩展，继承TranscationDefinition

```
rollbackOn(Throwable ex) : 回滚扩展方法

```

* `事务不生效场景`


```
1.方法必须的public，非final的，否则aop代理不了，事务无法生效
2.要抛出runtime异常才生效，spring认为check异常不应该由框架来处理
3.类内部方法调用事务不生效，比如同个类中的方法A，调用this.B() ，需要使用代理对象调用才会生效，因为代理对象是会被Aop代理的
4.保证调用方的方法上有@Transcational注解，事务会生效
```

## NameSpaceHandler

自定义标签配置，如<zjf:user/>

1. 继承NameSpaceHandler
2. 实现init()方法
3. 自定义BeanDefinitionParser实现类
4. 调用NameSpaceHandler.registerBeanDefinitionParser方法
5. 将NameSpaceHandler放入spring.handlers文件中



### 自定义实现jar包，如何托管给spring

```
通过spring.factories配置需要扫描的类全路径，springboot就是这么玩的
```

