# Spring事务实现原理

众所周知，Spring的事务是通过AOP实现的，具体是怎么实现的？接下来做详细分析

在使用SpringBoot之前（在SpringBoot中会自动装配事务），一般是采用XML配置或者Config配置类来开启事务，如：

```java
@Configuration
// 开启事务
@EnableTransactionManagement
public class ConfigTest {
}
```

@EnableTransactionManagement就是事务开启的入口，本文将从这里展开分析



## @EnableTransactionManagement

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 重要：入口在这
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {

   // 是否代理目标类，设置为true会使用CGLIB代理
   boolean proxyTargetClass() default false;

   // 实现事务的方式，默认为PROXY，有PROXY和ASPECTJ 
   AdviceMode mode() default AdviceMode.PROXY;

   // 排序
   int order() default Ordered.LOWEST_PRECEDENCE;
}
```

在@EnableTransactionManagement中，@Import(TransactionManagementConfigurationSelector.class)事务的入口，看下它做了哪些事



## TransactionManagementConfigurationSelector

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

   // IOC过程中会调用此方法，对return的类做解析和实例化
   @Override
   protected String[] selectImports(AdviceMode adviceMode) {
      switch (adviceMode) {
         case PROXY:
            return new String[] {AutoProxyRegistrar.class.getName(), ProxyTransactionManagementConfiguration.class.getName()};
         case ASPECTJ:
            return new String[] {TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME};
         default:
            return null;
      }
   }
}
```

这里暂且只对PROXY方式做分析



### AutoProxyRegistrar

AutoProxyRegistrar实现了ImportBeanDefinitionRegistrar#registerBeanDefinitions()方法，它是用来注册BeanDefinition给Spring

```java
public class AutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
   @Override
   public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      boolean candidateFound = false;
      // 拿到配置类上所有注解
      Set<String> annoTypes = importingClassMetadata.getAnnotationTypes();
      for (String annoType : annoTypes) {
         // 拿到注解中的所有属性
         AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
         if (candidate == null) {
            continue;
         }
         // 获取@EnableTransactionManagement的mode属性
         Object mode = candidate.get("mode");
         // 获取@EnableTransactionManagement的proxyTargetClass属性
         Object proxyTargetClass = candidate.get("proxyTargetClass");
         if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
               Boolean.class == proxyTargetClass.getClass()) {
            candidateFound = true;
            // PROXY
            if (mode == AdviceMode.PROXY) {
               // 注册InfrastructureAdvisorAutoProxyCreator到BeanDefinitionMap
               AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
               // 如果要对目标类做CBLIB代理，则将BeanDefinition的proxyTargetClass属性设置为true
               if ((Boolean) proxyTargetClass) {
                  AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
                  return;
               }
            }
         }
      }
}
```



### InfrastructureAdvisorAutoProxyCreator

[![ytXfts.jpg](https://s3.ax1x.com/2021/02/07/ytXfts.jpg)](https://imgchr.com/i/ytXfts)

 从类关系图中可以看出，InfrastructureAdvisorAutoProxyCreator继承了AbstractAutoProxyCreator，而AbstractAutoProxyCreator实现了BeanPostProcessor，可以在bean实例化前后做操作，通过查看AbstractAutoProxyCreator的源码可以发现，在Bean初始化后对Bean做了代理

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         // 生成代理，具体原理可自行查阅源码，或在本人的【Spring详解】文章有介绍，主要功能是生成代理对象
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

所以，InfrastructureAdvisorAutoProxyCreator正是通过此方式来为切面做代理增强的





### ProxyTransactionManagementConfiguration

ProxyTransactionManagementConfiguration是一个配置类，主要提供了BeanFactoryTransactionAttributeSourceAdvisor、TransactionAttributeSource、TransactionInterceptor三个Bean的实例化

```java
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

   // 事务增强器，它有2个重要的字段
   // 1.AnnotationTransactionAttributeSource：用于解析事务注解的信息
   // 2.TransactionInterceptor：事务拦截器，执行逻辑在这个里面
   @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
      BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
      // 设置事务注解解析器
      advisor.setTransactionAttributeSource(transactionAttributeSource());
      // 设置事务拦截器
      advisor.setAdvice(transactionInterceptor());
      // 设置排序字段
      advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
      return advisor;
   }
 
   // 用于解析事务注解的信息
   @Bean
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public TransactionAttributeSource transactionAttributeSource() {
      return new AnnotationTransactionAttributeSource();
   }

   // 事务拦截器
   @Bean
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public TransactionInterceptor transactionInterceptor() {
      TransactionInterceptor interceptor = new TransactionInterceptor();
      interceptor.setTransactionAttributeSource(transactionAttributeSource());
      if (this.txManager != null) {
         interceptor.setTransactionManager(this.txManager);
      }
      return interceptor;
   }
}
```



#### TransactionAttributeSource

事务解析器，通过对上述源码的查看，transactionAttributeSource()方法直接new了一个AnnotationTransactionAttributeSource

```java
// new的话就调用此构造器
public AnnotationTransactionAttributeSource() {
   this(true);
}

// 具体实现
public AnnotationTransactionAttributeSource(boolean publicMethodsOnly) {
	this.publicMethodsOnly = publicMethodsOnly;
  // 具体的事务注解解析器
	this.annotationParsers = new LinkedHashSet<TransactionAnnotationParser>(2);
	this.annotationParsers.add(new SpringTransactionAnnotationParser());
  // JTA 1.2事务注解是否被使用，我不懂
	if (jta12Present) {
		this.annotationParsers.add(new JtaTransactionAnnotationParser());
	}
  // EJB 3 事务注解是否被使用，我不懂
	if (ejb3Present) {
		this.annotationParsers.add(new Ejb3TransactionAnnotationParser());
	}
}
```

着重看下SpringTransactionAnnotationParser的实现吧，它是怎么做的解析

[![yNndsS.png](https://s3.ax1x.com/2021/02/07/yNndsS.png)](https://imgchr.com/i/yNndsS)

从上图中可以看出，首先获取@Transactional，然后对其进行解析，解析字段图中有标注



#### TransactionInterceptor

事务拦截器。负责执行事务方法和对事务的提交回滚等行为做控制

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {

   @Override
   public Object invoke(final MethodInvocation invocation) throws Throwable {
      // 获取被代理的class
      Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
       // 执行事务方法
      return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
         @Override
         public Object proceedWithInvocation() throws Throwable {
            return invocation.proceed();
         }
      });
   }
}
```

从源码中看出，TransactionInterceptor继承TransactionAspectSupport，实现MethodInterceptor。

- TransactionAspectSupport：对事务切面逻辑支撑的抽象基类，其提供给子类的公共方法是：invokeWithinTransaction()，此方法会将事务方法前后设置事务逻辑
- MethodInterceptor：功能类似于JDK动态代理接口InvocationHandler，负责提供Invoke()方法，具体实现交由子类



#### **TransactionAspectSupport#invokeWithinTransaction()**

```java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
      throws Throwable {
   // 如果当前事务属性为空，那么当前方法是没有使用@transactional的
   final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
   // 获取PlatformTransactionManager
   final PlatformTransactionManager tm = determineTransactionManager(txAttr);
   // 获取被切的方法，如:test.service.userServiceImpl.test()
   final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);
	 // 
   if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
      // 开启一个事务，根据设置的传播行为开启，如果是非事务方式运行的传播行为，则不会开启事务
      // 开启后会将此事务与当前线程绑定，ThreadLoacl实现
      TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
      Object retVal = null;
      try {
         // 执行方法
         retVal = invocation.proceedWithInvocation();
      }
      catch (Throwable ex) {
         // 1.判断异常是否==rollbackFor设置的异常，如果是，执行rollback回滚
         // 2.反之则commit提交事务
         // 3.无论提交还是回滚都是由PlatformTransactionManager负责执行
         completeTransactionAfterThrowing(txInfo, ex);
         throw ex;
      }
      finally {
         // 清除事务，实际上是将ThreadLocal中的事务设置为old Transaction
         cleanupTransactionInfo(txInfo);
      }
      // 提交事务，由PlatformTransactionManager负责提交
      commitTransactionAfterReturning(txInfo);
      return retVal;
   }

   else {
      // 生命式事务逻辑处理
      final ThrowableHolder throwableHolder = new ThrowableHolder();
      try {
         Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
               new TransactionCallback<Object>() {
                  @Override
                  public Object doInTransaction(TransactionStatus status) {
                     // 也是开启事务，与上面的逻辑一致
                     TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
                     try {
                        // 执行方法
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
                        // 清理事务，与上面的逻辑一致
                        cleanupTransactionInfo(txInfo);
                     }
                  }
               });
         if (throwableHolder.throwable != null) {
            throw throwableHolder.throwable;
         }
         return result;
      }
      // 忽略部分代码
   }
}	
```



#### 开启事务原理

[![yN1S2R.jpg](https://s3.ax1x.com/2021/02/07/yN1S2R.jpg)](https://imgchr.com/i/yN1S2R)



## 事务失效场景

1. @Transactional作用在非public方法上。这是因为TransactionInterceptor事务拦截器会检查方法修饰符，如果非public则直接return，且这种情况不会抛异常

2. 数据库引擎不支持事务，比如myisam

3. 传播行为设置错误：
   1. **PROPAGATION_SUPPORTS：**如果当前有事务则使用该事务，否则以非事务的方式执行
   2. **PROPAGATION_NOT_SUPPORTED：**如果当前有事务则挂起当前事务，以非事务的方式执行
   3. **PROPAGATION_NEVE：**如果有事务则抛异常，以非事务的方式执行
4. 如果是自定义的异常需要指定rollbackFor的class，因为Spring只有RuntimeException和Error才会执行回滚
5. 同个类中的方法互相调用事务会失效，因为AOP没办法切到，可以通过getBean重新调用
6. 异常被catch了，需要手动抛出异常

