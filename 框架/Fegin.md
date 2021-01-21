# Fegin



## feigin只声明了接口，没有实现类，它是如何工作的？

首先，使用fegin需要在启动类启动fegin

```java
// 使用fegin
@EnableFeignClients
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

通过查看@EnableFeignClients源码，可以看到它通过@Import(FeignClientsRegistrar.class)来实现的

```java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar,
      ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
   
   // 省略部分代码
   
   // 实现ImportBeanDefinitionRegistrar
   // 此方法会在IOC流程中的ConfigurationClass调用
   // 将声明的fegin接口托管给Spring
   @Override
   public void registerBeanDefinitions(AnnotationMetadata metadata,
         BeanDefinitionRegistry registry) {
      // 注册Configuration。@EnableFeignClients
      registerDefaultConfiguration(metadata, registry);
      // 注册fegin接口
      registerFeignClients(metadata, registry);
   }
}
```

FeignClientsRegistrar#registerFeignClients()实现

```java
public void registerFeignClients(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {
   // 获取扫描器
   ClassPathScanningCandidateComponentProvider scanner = getScanner();
   scanner.setResourceLoader(this.resourceLoader);
   // 需要扫描的包
   Set<String> basePackages;
   // 获取@EnableFeignClients配置的参数信息
   Map<String, Object> attrs = metadata
         .getAnnotationAttributes(EnableFeignClients.class.getName());
   AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
         FeignClient.class);
   // 获取@EnableFeignClients配置的clients参数值
   // 也就是用户声明的fegin接口
   // 如：@EnableFeignClients(clients={TestFegin.class, TestFegin2.class})
   // 获取的就是TestFegin和TestFegin2，那么fegin也会只加载这2个接口
   final Class<?>[] clients = attrs == null ? null
         : (Class<?>[]) attrs.get("clients");
   // 如果为空，则获取配置的包名，如果未配置，则basePackages取@EnableFeignClients声明目录
   // 如：@EnableFeignClients声明在com.test,则basePackages=com.test
   if (clients == null || clients.length == 0) {
      scanner.addIncludeFilter(annotationTypeFilter);
      basePackages = getBasePackages(metadata);
   }
   else {
      // 否则加载配置的clients
      // 如：@EnableFeignClients(clients={TestFegin.class, TestFegin2.class})
   		// 获取的就是TestFegin和TestFegin2，那么fegin也会只加载这2个接口
      final Set<String> clientClasses = new HashSet<>();
      basePackages = new HashSet<>();
      for (Class<?> clazz : clients) {
         basePackages.add(ClassUtils.getPackageName(clazz));
         clientClasses.add(clazz.getCanonicalName());
      }
      AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
         @Override
         protected boolean match(ClassMetadata metadata) {
            String cleaned = metadata.getClassName().replaceAll("\\$", ".");
            return clientClasses.contains(cleaned);
         }
      };
      scanner.addIncludeFilter(
            new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
   }
   // 循环需要扫描的包
   for (String basePackage : basePackages) {
      // 获取需要注册的接口，注意：这里只会获取到使用了@FeignClient的接口
      Set<BeanDefinition> candidateComponents = scanner
            .findCandidateComponents(basePackage);
      // 循环，注册BeanDefinition
      for (BeanDefinition candidateComponent : candidateComponents) {
         if (candidateComponent instanceof AnnotatedBeanDefinition) {
            // verify annotated class is an interface
            AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
            AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
            Assert.isTrue(annotationMetadata.isInterface(),
                  "@FeignClient can only be specified on an interface");
						// 获取@FeignClient配置的参数，如name，url、fallback等等
            Map<String, Object> attributes = annotationMetadata
                  .getAnnotationAttributes(
                        FeignClient.class.getCanonicalName());
 						// name
            String name = getClientName(attributes);
            registerClientConfiguration(registry, name,
                  attributes.get("configuration"));
					  // 注册到BeanDefinitionMap
            registerFeignClient(registry, annotationMetadata, attributes);
         }
      }
   }
}
```

FeignClientsRegistrar#registerFeignClient()实现

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
      AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
   String className = annotationMetadata.getClassName();
   // 通过FeignClientFactoryBean生成fegin接口的代理对象
   BeanDefinitionBuilder definition = BeanDefinitionBuilder
         .genericBeanDefinition(FeignClientFactoryBean.class);
   validate(attributes);
   // 设置属性
   definition.addPropertyValue("url", getUrl(attributes));
   definition.addPropertyValue("path", getPath(attributes));
   String name = getName(attributes);
   definition.addPropertyValue("name", name);
   definition.addPropertyValue("type", className);
   definition.addPropertyValue("decode404", attributes.get("decode404"));
   definition.addPropertyValue("fallback", attributes.get("fallback"));
   definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
   definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
	 // 设置别名
   String alias = name + "FeignClient";
   AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
	
   boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

   beanDefinition.setPrimary(primary);

   String qualifier = getQualifier(attributes);
   if (StringUtils.hasText(qualifier)) {
      alias = qualifier;
   }
	
   BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
         new String[] { alias });
   // 注册到BeanDefinitionMap，托管给Spring
   BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

重点在：FeignClientFactoryBean，它是如何生成代理对象的，看下实现

```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean,
		ApplicationContextAware {
	
  // 省略部分接口

	@Override
	public Object getObject() throws Exception {
		FeignContext context = applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);
		
    // 如果未配置URL，则通过serviceId请求，走负载
    // fegin有2种配置方式：1.url 2.serviceId，想了解的可以自己查阅相关资料
		if (!StringUtils.hasText(this.url)) {
			String url;
			if (!this.name.startsWith("http")) {
				url = "http://" + this.name;
			}
			else {
				url = this.name;
			}
			url += cleanPath();
      // 走负载生成代理对象
      // 底层通过Targeter.target()生成代理对象
			return loadBalance(builder, context, new HardCodedTarget<>(this.type,
					this.name, url));
		}
    // 配置了url，则拼接url
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not lod balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient)client).getDelegate();
			}
			builder.client(client);
		}
		Targeter targeter = get(context, Targeter.class);
    // 生成fegin接口的代理对象
		return targeter.target(this, builder, context, new HardCodedTarget<>(
				this.type, this.name, url));
	}
}
```

Targeter#target()

Targeter是一个接口，他有2个实现类，分别是：DefaultTargeter和HystrixTargeter

```java
class DefaultTargeter implements Targeter {

   @Override
   public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
                  Target.HardCodedTarget<T> target) {
      return feign.target(target);
   }
}
```

在本人的测试下，无论hystrix.enabled设置为true还是false，都是走的HystrixTargeter，不过这2个实现的底层都是通过JDK动态代理生成代理对象的

HystrixTargeter#target()实现

```java
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
               Target.HardCodedTarget<T> target) {
   if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
      return feign.target(target);
   }
   feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
   SetterFactory setterFactory = getOptional(factory.getName(), context,
      SetterFactory.class);
   if (setterFactory != null) {
      builder.setterFactory(setterFactory);
   }
   // 1.如果fallback熔断的实现有返回值，则设置熔断，并生成代理对象返回
   Class<?> fallback = factory.getFallback();
   if (fallback != void.class) {
      return targetWithFallback(factory.getName(), context, target, builder, fallback);
   }
   // 2.如果fallbackFactory熔断的实现有返回值，则设置熔断，并生成代理对象返回
   Class<?> fallbackFactory = factory.getFallbackFactory();
   if (fallbackFactory != void.class) {
      return targetWithFallbackFactory(factory.getName(), context, target, builder, fallbackFactory);
   }
	 // 没有设置熔断，则直接生成代理对象
   // 底层调用ReflectiveFeign.newInstance()
   return feign.target(target);
}
```

ReflectiveFeign#newInstance()

```java
public <T> T newInstance(Target<T> target) {
  Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
  Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
  List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
  // 循环fegin接口内的所有方法
  for (Method method : target.type().getMethods()) {
    // 如果是object的方法，则不处理
    if (method.getDeclaringClass() == Object.class) {
      continue;
    } else if(Util.isDefault(method)) {
      // 接口的默认方法处理
      DefaultMethodHandler handler = new DefaultMethodHandler(method);
      defaultMethodHandlers.add(handler);
      methodToHandler.put(method, handler);
    } else {
      methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
    }
  }
  // 代理实现类InvocationHandler
  InvocationHandler handler = factory.create(target, methodToHandler);
  // 生成代理对象
  T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);
	
  for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
    defaultMethodHandler.bindTo(proxy);
  }
  return proxy;
}
```



## 总结

1. 使用fegin，需开启@EnableFeignClients

2. @EnableFeignClients中通过@Import(FeignClientsRegistrar.class)来实现扫描fegin接口

3. 根据@EnableFeignClients配置的扫描包或者clients扫描需要注册的fegin接口，将fegin接口注册成BeanDefinition托管给Spring

4. 通过FeignClientFactoryBean.getObject()方法，将注册的BeanDefinition通过JDK代理生成代理类

   