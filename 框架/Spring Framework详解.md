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
   	// 初始化工厂，默认为DefaultListableBeanFactory
   	public GenericApplicationContext() {
   		this.beanFactory = new DefaultListableBeanFactory();
   	}
   }
   ```
   
2. 扫描配置的config类，封装成BeanDefinition放到BeanDefinitionMap

   ```java
   public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
   		// 这一步将配置类封装成BeanDefinition放到BeanDefinitionMap
       // 内部调用BeanDefinitionRegistry#registerBeanDefinition()方法注册的，就不细讲了
       register(annotatedClasses);
       // 这个先忽略
       // refresh();
   }
   ```

3. 调用invokeBeanFactoryPostProcessors，通过ConfigurationClassPostProcessor扫描有@Component、@Service、@Bean等注解的类封装成BeanDefinition并放到BeanDefinitionMap中，内部调用invokeBeanDefinitionRegistryPostProcessors执行BeanDefinition注册后期处理器，其内部调用了DefaultListableBeanFactory.registerBeanDefinition()方法扫描需要注册的类放到BeanDefinitionMap

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
            // 如果是FactoryBean的实现
            if (isFactoryBean(beanName)) {
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
             // postProcessor处理，例如：对@Autowired的处理
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
         // 将创建中的对象提前暴露出来，放到三级缓存中，同时从二级缓存中移除
         // 主要是为了解决循环依赖的问题
         addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
               return getEarlyBeanReference(beanName, mbd, bean);
            }
         });
      }
     Object exposedObject = bean;
     try {
         // 1.判断是否需要属性提供
         // 2.属性填充（依赖注入）
         // 3.此方法很重要
   			populateBean(beanName, mbd, instanceWrapper);
   			if (exposedObject != null) {
           // 对象创建后的初始化动作
           // 执行方法及顺序如下：
           // 1.invokeAwareMethods()方法,主要三个Aware的实现，设置钩子方法的值
           // 		1.1 BeanNameAware，BeanFactoryAware，BeanClassLoaderAware
           // 2.BeanPostProcessor.before()方法
           // 3.invokeInitMethods()方法，也就是@PostConstruct修饰的方法、InitializingBean、init()方法
           // 4.BeanPostProcessor.after方法，如AOP等
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
               if (method.getParameterTypes().length == 0) {
                  if (logger.isWarnEnabled()) {
                     logger.warn("Autowired annotation should only be used on methods with parameters: " +
                           method);
                  }
               }
               boolean required = determineRequiredStatus(ann);
               PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
               currElements.add(new AutowiredMethodElement(method, required, pd));
            }
         }
      });

      elements.addAll(0, currElements);
      targetClass = targetClass.getSuperclass();
   }
   while (targetClass != null && targetClass != Object.class);、
   return new InjectionMetadata(clazz, elements);
}
```





**@Resource同理，由CommonAnnotationBeanPostProcessor处理，它也实现了MergedBeanDefinitionPostProcessor**





### 如何解决循环依赖

#### 什么是循环依赖？

比如有2个类，A和B，互相引用
首先A被创建，经过一系列流程，到自动注入时获取B这个bean(getBean(B))
但此时B这个bean并没有被实例化(signleObjects中没有)，于是就会创建B，B也经过一系列流程，到自动注入时获取A(getBean(A))，发现A也没有被实例化(signleObjects中没有)，于是A又去找B...死循环



#### 解决方法？

比如有2个类，A和B，互相引用

    1.  首先A被创建，此时A会被放到singletonsCurrentlyInCreation这个set集合中，表示该对象是在创建过程中
       2.  经过一系列流程，到自动注入时获取B(getBean(B)),此时B并没有被实例化(signleObjects中没有)，就会创建B
       3.  B经过一系列流程，到自动注入时获取A(getBean(A))，发现A正在创建过程中(isSingletonCurrentlyInCreation(A)为ture)，于是从earlySingletonObjects(存放对象集合，map)中获取到A，如果取不到A，则从SignletonFactory.getObject()取A，因为实例化B并不需要A这个bean，只需要A对象
       4.  此时B就能被实例化成bean，A接下来获取到B也能实例化
       5.  获取循环依赖对象过程：先从singletonObjects拿，拿不到从earlySingletonObjects拿，再拿不到就从SignletonFactory拿，
           拿到后会put到earlySingletonObjects里同时从SignletonFactory移除掉，再有循环依赖的对象需要获取它的话，直接从earlySingletonObjects获取即可。因为直接从SignletonFactory.getObject拿的话性能不太高，getObject()会获取所有BeanPostProcessor做处理。 试想下，一个对象如果被多次循环引用。而SignletonFactory.getObject创建对象需要2秒，可想而知性能有多慢。



- **singletonObjects：存放真正实例化的bean。1级缓存**

- **earlySingletonObjects：存放早期的bean对象，此时还不是真正的bean。2级缓存**

- **SignletonFactory：存放工厂，解决循环依赖aop的问题，比如A和B循环依赖，A被aop，当A注入属性时获取B，B也能被Aop代理。3级缓存**

- **以上3个缓存，都存放在DefaultSingletonBeanRegistry中**

  

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
  
  @Bean
  // 等同于xml中的bean标签，方法名=bean标签的id属性，返回类型=bean标签的Class属性
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



## AOP

Aop是一个标准，Spring AOP 是它的一种实现

* `启用aop` @EnableAspectJAutoProxy

```
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

}
```

* `配置切面` @Aspect

```
@Aspect
public class NotVeryUsefulAspect {

}
```

* `配置切点` @Pointcut

```
@Aspect
public class NotVeryUsefulAspect {
    @Pointcut("execution(* transfer(..))") // 切点表达式
    private void anyOldTransfer() {} // 切点实现方法
}
```

* `配置通知` 

```
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

* `原理`

```
实现BeanPostProcessor, 在ioc doCreateBean() 方法中调用实现BeanPostProcessor的实现类(AnnotationAwareAspectJAutoProxyCreator)，完成对bean的aop增强并返回增强的代理对象(bean)。

通过aopProxy 生成aop代理对象。
aopProxy 有2个实现类 JDKDynamicAopProxy 和  cglibAopProxy
通过bean，执行不同的代理方式
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

