# Springboot 自动装配



Springboot的优势在于，省去了繁多的XML配置及POM的依赖，通过starter自动导入依赖，嵌入式tomcat，只需要配置yml或yaml或properties属性文件即可

那么Springboot是如何实现自动装配的？本文将简单分析自动装配过程以及如何自定义自动装配

本文基于Springboot 2.1.6版本进行分析，过程中有一些关键点会做版本间的比较和标注



## 启动类

```java
@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

上面是一个简单的启动类，通过run main()即可启动项目，粗略来看，相比传统的Spring项目，省去了配置Tomcat的麻烦

可以注意到，在Application上使用了@SpringBootApplication注解，这就是自动装配的入口，来看下它的实现



## @SpringBootApplication

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

   // 需要排除的class，在启动的时候不会被ComponentScan扫描到
   @AliasFor(annotation = EnableAutoConfiguration.class)
   Class<?>[] exclude() default {};

   // 需要排除的class，通过className排除，在启动的时候不会被ComponentScan扫描到
   @AliasFor(annotation = EnableAutoConfiguration.class)
   String[] excludeName() default {};

   // 需要扫描的的包，ComponentScan实现
   @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
   String[] scanBasePackages() default {};

   // 需要扫描的class，ComponentScan实现
   @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
   Class<?>[] scanBasePackageClasses() default {};
}
```

在@SpringBootApplication源码中，有三个重要的注解，分别是：@SpringBootConfiguration、@EnableAutoConfiguration、@ComponentScan

- @SpringBootConfiguration：使用了@Configuration，表示@SpringBootApplication是一个配置类
- @EnableAutoConfiguration：自动装配入口
- @ComponentScan：扫描包，将符合条件的类托管给Spring

下面着重介绍@EnableAutoConfiguration



## @EnableAutoConfiguration

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
	 // 自动装配配置类
   String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

   // 排除的class，配置后不会被自动装配
   Class<?>[] exclude() default {};
   // 排除的class，配置后不会被自动装配
   String[] excludeName() default {};
}
```

从源码中可以看到，它@Import了AutoConfigurationImportSelector，这个就是自动装配的入口（AutoConfigurationImportSelector逻辑会在Spring的IOC流程中被执行到，具体可以查看ConfigurationClassPostProcessor和ConfigurationClassParser了解对各种注解的解析和执行过程）

**注：在springboot 1.5.x版本中，@Import的是EnableAutoConfigurationImportSelector，它的父类就是AutoConfigurationImportSelector，而且在1.5.x版本中EnableAutoConfigurationImportSelector已经过时了，所以本文不会对它进行分析，只会分析AutoConfigurationImportSelector**

**在springboot 2.x版本开始，@Import的就是AutoConfigurationImportSelector**



### @AutoConfigurationImportSelector

它继承DeferredImportSelector，在ConfigurationClassParser.parse()中，2.x版本不会直接执行selectImports()方法，1.5.x会执行selectImports()方法

[![yGmQ0I.jpg](https://s3.ax1x.com/2021/02/05/yGmQ0I.jpg)](https://imgchr.com/i/yGmQ0I)

在2.x版本中，交由deferredImportSelectorHandler#handle()方法处理，最终会调用到AutoConfigurationImportSelector内部类AutoConfigurationGroup实现的process()方法中

```java
@Override
public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
   Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
         () -> String.format("Only %s implementations are supported, got %s",
               AutoConfigurationImportSelector.class.getSimpleName(),
               deferredImportSelector.getClass().getName()));
   // 重点在这一步getAutoConfigurationEntry()方法，此方法是在AutoConfigurationImportSelector中实现的
   AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
         .getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);
   this.autoConfigurationEntries.add(autoConfigurationEntry);
   for (String importClassName : autoConfigurationEntry.getConfigurations()) {
      this.entries.putIfAbsent(importClassName, annotationMetadata);
   }
}
```

AutoConfigurationImportSelector#getAutoConfigurationEntry()

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
      AnnotationMetadata annotationMetadata) {
   if (!isEnabled(annotationMetadata)) {
      return EMPTY_ENTRY;
   }
   // 获取@EnableAutoConfiguration的属性，如：exclude(),excludeName（）
   AnnotationAttributes attributes = getAttributes(annotationMetadata);
   // 获取所有需要自动装配Configuration全路径类名
   List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
   // 去除重复的configuration
   configurations = removeDuplicates(configurations);
   // 获取需要排除的configuration
   Set<String> exclusions = getExclusions(annotationMetadata, attributes);
   // 排除掉不需要自动装配的configuration
   checkExcludedClasses(configurations, exclusions);
   configurations.removeAll(exclusions);
   // 过滤
   configurations = filter(configurations, autoConfigurationMetadata);
   fireAutoConfigurationImportEvents(configurations, exclusions);
   // 返回真正需要自动装配的configuration
   return new AutoConfigurationEntry(configurations, exclusions);
}
```

下面分析下AutoConfigurationImportSelector是怎么知道哪些configuration需要装配的，入口在getCandidateConfigurations()方法中

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
   // 借助SpringFactoriesLoader加载。下面会介绍SpringFactoriesLoader
   List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
         getBeanClassLoader());
   Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
         + "are using a custom packaging, make sure that file is correct.");
   return configurations;
}
```



## SpringFactoriesLoader

类似于JAVA SPI，是Spring提供的一种加载机制，只需要在classpath路径下新建META-INF/spring.factories，并且按照Properties格式配置好需要加载的类即可

```properties
// 如果需要加载多个，则用,隔开即可
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.test.config.UserDefAutoConfiguration
```

加载原理如下：

```java
// 入口
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
   String factoryClassName = factoryClass.getName();
   return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}

// spring.factories文件位置
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

// 实现，加载spring.factories
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
   MultiValueMap<String, String> result = cache.get(classLoader);
   if (result != null) {
      return result;
   }
   try {
      Enumeration<URL> urls = (classLoader != null ?
            classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
            ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
      result = new LinkedMultiValueMap<>();
      while (urls.hasMoreElements()) {
         URL url = urls.nextElement();
         UrlResource resource = new UrlResource(url);
         Properties properties = PropertiesLoaderUtils.loadProperties(resource);
         for (Map.Entry<?, ?> entry : properties.entrySet()) {
            String factoryClassName = ((String) entry.getKey()).trim();
            for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
               result.add(factoryClassName, factoryName.trim());
            }
         }
      }
      cache.put(classLoader, result);
      return result;
   }
   catch (IOException ex) {
      throw new IllegalArgumentException("Unable to load factories from location [" +
            FACTORIES_RESOURCE_LOCATION + "]", ex);
   }
}
```





## 如何自定义自动装配类

1.定义配置文件

```properties
user:
	info:
		userName: 测试
		age: 10
```



2.自定义配置类用来映射配置文件

```java
@ConfigurationProperties(prefix = "user.info")
public class UserDefProperties {
    private String userName;
    private Integer age;
}
```



3.定义启动配置类，用来转换配置类

```java
@Configuration
@EnableConfigurationProperties(value = UserDefProperties.class)
public class UserDefAutoConfiguration {
  
    @Autowired
    private UserDefProperties userDefProperties;
  
    @Bean
    public User user(){
        log.info("自定义自动装配UserDefAutoConfiguration.....");
        User user = new User();
        user.setAge(userDefProperties.getAge());
        user.setUserName(userDefProperties.getUserName());
        return user;
    }
}
```

 4.配置spring.factories文件

新建META-INFO文件夹，然后在文件夹内新建spring.factories文件，内容如下：

```properties
// 如果需要加载多个，则用,隔开即可
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.test.config.UserDefAutoConfiguration
```

