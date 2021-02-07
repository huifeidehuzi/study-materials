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

[![ytxmm8.jpg](https://s3.ax1x.com/2021/02/07/ytxmm8.jpg)](https://imgchr.com/i/ytxmm8)

从上图中可以看出，首先获取@Transactional，然后对其进行解析，解析字段图中有标注



#### TransactionInterceptor

事务拦截器