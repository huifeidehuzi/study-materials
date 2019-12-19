# Dubbo
###官方文档：http://dubbo.apache.org/zh-cn/
###简介

```
dubbo是一款高性能RPC框架，具有以下特性：
1.面向接口代理的高性能RPC调用
2.可配置的负载均衡
3.服务自动注册与发现
4.可视化的服务治理与运维
5.高度可扩展能力
6.没有任何api侵入
......
```
###配置：内容为官网文档提供，更多配置细节可参考官方文档的`配置`和`示例`
* `xml配置`

```
1.服务提供者provider.xml示例

<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <!-- 服务名称 -->
    <dubbo:application name="demo-provider"/>
    <!-- 注册中心 -->
    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>
    <!-- 协议 -->
    <dubbo:protocol name="dubbo" port="20890"/>
    <!-- 暴露服务的bean-->
    <bean id="demoService" class="org.apache.dubbo.samples.basic.impl.DemoServiceImpl"/>
    <!-- 需要暴露的服务 -->
    <dubbo:service interface="org.apache.dubbo.samples.basic.api.DemoService" ref="demoService"/>
</beans>

2.服务消费者consumer.xml 示例

<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
       <!-- 服务名称 -->
    <dubbo:application name="demo-consumer"/>
    <!-- 注册中心 -->
    <dubbo:registry group="aaa" address="zookeeper://127.0.0.1:2181"/>
    <!-- 需要消费的服务 -->
    <dubbo:reference id="demoService" check="false" interface="org.apache.dubbo.samples.basic.api.DemoService"/>
</beans>
```


```
配置标签明细列表：
标签	                 用途	        解释
<dubbo:service/>	服务配置	用于暴露一个服务，定义服务的元信息，一个服务可以用多个协议暴露，一个服务也可以注册到多个注册中心
<dubbo:reference/> 引用配置	用于创建一个远程服务代理，一个引用可以指向多个注册中心
<dubbo:protocol/>	协议配置	用于配置提供服务的协议信息，协议由提供方指定，消费方被动接受
<dubbo:application/>	应用配置	用于配置当前应用信息，不管该应用是提供者还是消费者
<dubbo:module/>	模块配置	用于配置当前模块信息，可选
<dubbo:registry/>	注册中心配置	用于配置连接注册中心相关信息
<dubbo:monitor/>	监控中心配置	用于配置连接监控中心相关信息，可选
<dubbo:provider/>	提供方配置	当 ProtocolConfig 和 ServiceConfig 某属性没有配置时，采用此缺省值，可选
<dubbo:consumer/>	消费方配置	当 ReferenceConfig 某属性没有配置时，采用此缺省值，可选
<dubbo:method/>	方法配置	用于 ServiceConfig 和 ReferenceConfig 指定方法级的配置信息
<dubbo:argument/>	参数配置	用于指定方法参数配置
```


```
配置优先级：
方法级优先，接口级次之，全局配置再次之。
如果级别一样，则消费方优先，提供方次之
其中，服务提供方配置，通过 URL 经由注册中心传递给消费方。

以timeout配置为例，下面是查找匹配timeout属性的顺序
1.服务消费方内方法
<dubbo:reference interface="xxx.xxx.xxx.demoService">
    <dubbo:method name="demo" timeout=10000 />
<dubbo:reference/>
  
2.服务提供方内方法  
<dubbo:service interface="xxx.xxx.xxx.demoService">
     <dubbo:method name="demo" timeout=10000 />
<dubbo:service/>

3.服务消费接口
<dubbo:reference interface="xxx.xxx.xxx.demoService" timeout=1000/>

4.服务提供方接口
<dubbo:service interface="xxx.xxx.xxx.demoService" timeout=1000/>

5.服务消费方
<dubbo:consumer timeout=1000/>

6.服务提供方
<dubbo:provider timeout=1000/>
```

* `属性配置` 如果应用足够简单，例如，不需要多注册中心或多协议，并且需要在spring容器中共享配置，那么，可以直接使用 dubbo.properties作为默认配置，Dubbo会自动加载classpath根目录下的dubbo.properties文件，但同样可以使用JVM参数来指定路径：-Ddubbo.properties.file=xxx.properties。

```
规则：将xml的tag名和属性名组合起来，用‘.’分隔。每行一个属性
示例:
dubbo.application.name=应用名称 等同于标签<dubbo:application name="应用名称" />
dubbo.registry.addreess=zookeeper://127.0.0.1:2181 等同于标签<dubbo:registry address="zookeeper://127.0.0.1:2181"/>
duubo.service.interface=xxx.xxx.xxx.demoService 等同于标签<dubbo:service interface="org.apache.dubbo.samples.basic.api.DemoService" ref="demoService"/>
.....
具体配置明细参照xml配置即可

注：
如果配置中有超过一个的tag，可以使用‘id’进行区分。如果不指定id，它将作用于所有tag
比如：dubbo.service.one.interface=xxx.xxx.xxx.demoService 此配置的id就是one，等同于标签<dubbo:service id="one" interface="xxx.xxx.xxx.demoService"/>

优先级：
JVM：-D参数
XML：XML中的当前配置会重写dubbo.properties中的；
Properties：默认配置，仅仅作用于以上两者没有配置时。
```

* `api配置` API 属性与配置项一对一，各属性含义，请参考上述文档


```
示例：
// 当前应用配置
ApplicationConfig application = new ApplicationConfig();
application.setName("应用名称");

// 连接注册中心配置
RegistryConfig registry = new RegistryConfig();
registry.setAddress("10.20.130.230:9090");
registry.setUsername("aaa");
registry.setPassword("bbb");

// 服务提供者协议配置
ProtocolConfig protocol = new ProtocolConfig();
protocol.setName("dubbo");
protocol.setPort(12345);
protocol.setThreads(200);

// 注意：ServiceConfig为重对象，内部封装了与注册中心的连接，以及开启服务端口
 
// 方法级配置
List<MethodConfig> methods = new ArrayList<MethodConfig>();
MethodConfig method = new MethodConfig();
method.setName("createXxx");
method.setTimeout(10000);
method.setRetries(0);
methods.add(method);

// 服务提供者暴露服务配置
ServiceConfig<XxxService> service = new ServiceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接，请自行缓存，否则可能造成内存和连接泄漏
service.setApplication(application);
service.setRegistry(registry); // 多个注册中心可以用setRegistries()
service.setProtocol(protocol); // 多个协议可以用setProtocols()
service.setInterface(XxxService.class);
service.setRef(xxxService);
service.setVersion("1.0.0");
reference.setMethods(methods); // 设置方法级配置

// 暴露及注册服务
service.export();
......
消费者配置同理
reference.get(); //引用服务

注：以上各个bean可交给spring托管，即可不用自行暴露服务和引用服务
不推荐使用api配置，太重。
```

* `注解配置` 需要 2.6.3 及以上版本


```
示例：
服务提供方暴露服务使用@Service即可 
注：dubbo@Service注解与spring@Service注解名称一致，切勿混淆

//注解上也可以配置版本、超时时间等等
@Service(version="1.0.0" timeout=1000 ....)
public class UserServiceImpl implements UserService {
    @Override
    public String hello(String name) {
        return "hello";
    }
}

//服务提供方指定spring扫描路径
@Configuration
@EnableDubbo(scanBasePackages = "org.apache.dubbo.samples.simple.annotation.impl")
//dubbo的属性配置文件，一些配置还是需要配置在此文件里
@ImportResource("classpath:/spring/dubbo-provider.properties")
public class ProviderConfiguration {
       
}

//服务消费方引用服务使用@Reference即可
@Component
public class AnnotationAction {
    @Reference
    private UserService userService;
    
    public String hello(String name) {
        return userService.hello(name);
    }
}

//服务消费方指定spring扫描路径
@Configuration
@EnableDubbo(scanBasePackages = "org.apache.dubbo.samples.simple.annotation.action")
@PropertySource("classpath:/spring/dubbo-consumer.properties")
@ComponentScan(value = {"org.apache.dubbo.samples.simple.annotation.action"})
static public class ConsumerConfiguration {

}
```

###整合Springboot

```
1.导入duubo依赖 注意：springboot版本低于2.0需要使用dubbo-spring-boot-starter的0.1.0版本
       
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>0.2.0</version>
</dependency>

<!-- zookeeper客户端 -->
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.7</version>
</dependency>


2.dubbo配置 application.properties 或者 application.yml
示例：
dubbo.application.name=customer_name
dubbo.registry.protocol=zookeeper
dubbo.registry.address=127.0.0.1:2181
dubbo.protocol.name=dubbo
dubbo.protocol.prort=20880
dubbo.monitor.protocol=registry

3.使用
使用@Service暴露服务
@Service  //注意此@Service注解与spring的@Service名称一致，切勿混淆
@Component
public class DemoServiceImpl implements DemoService{
    @Override
    public String getDemo(String s) {
        return "demo "+s;
    }
}
使用@Reference引用服务
@Component
public class OrderServiceImpl implements OrderService{
    @Reference
    private DemoService demoService
    @Override
    public String getDemo(String s) {
        return demoService.getDemo(s);
    }
}
使用@EnableDubbo启用dubbo
@SpringBootApplication
@EnableDubbo
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(OrderApp.class, args);
    }
}

也可以使用@DubboComponentScan(basePackages = "需要扫描的包绝对路径")整合
整合xml配置步骤:
    1.配置服务提供者及服务消费者xml，详细配置见文档中的“配置”介绍
    2.应用启动类中新增：@ImportResource("dubbo.xml")
```

###原理
* `加载xml配置` 

```
1.DubboBeanDefinitionParser
实现BeanDefinitionParser，实现接口parse(Element element, ParserContext parserContext)

此类主要功能为：加载dubbo的xml配置的文件，解析各个标签并设置属性值，封装为RootBeanDefinition，交给spring容器托管

具体细节可自行查阅源码。
其中最特殊的配置为:ServiceBean

2.ServiceBean
实现了InitializingBean...等等接口，这里只阐述InitializingBean
spring在bean初始化的时候，会调用InitializingBean的afterPropertiesSet()方法，这里为ServiceBean实现的afterPropertiesSet()方法。
此方法主要是设置dubbo的各种配置，如：ApplicationConfig，ProtocolConfig，MonitorConfig，RegistryConfig...等等，设置完成后会调用export()方法暴露服务。
export():
    最终实际调用doExportUrlsFor1Protocol()暴露服务，其内部最重要的代码如下：
    
    //通过代理工厂获取接口实现类的代理对象
    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
    
    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
    //暴露服务
    Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
```

* `加载properties配置文件` @EnableDubbo方式


```
1.加载配置
@EnableDubboConfig，此注解import了DubboConfigConfigurationSelector类
实现如下：
@Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {

        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                importingClassMetadata.getAnnotationAttributes(EnableDubboConfig.class.getName()));

        boolean multiple = attributes.getBoolean("multiple");
        //是否多个配置
        if (multiple) {
            return of(DubboConfigConfiguration.Multiple.class.getName());
        } else {
            return of(DubboConfigConfiguration.Single.class.getName());
        }
    }

//获取配置的属性并且给对象的config bean赋值
/**
     * 单个配置
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
     * 多个配置
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
//解析配置及绑定
@EnableDubboConfigBindings：
    导入了DubboConfigBindingRegistrar类
    protected void registerBeanDefinitions(AnnotationAttributes attributes, BeanDefinitionRegistry registry) {
        //获取属性名，如:dubbo.application
        String prefix = environment.resolvePlaceholders(attributes.getString("prefix"));
        //获取属性对应的config
        Class<? extends AbstractConfig> configClass = attributes.getClass("type");
        //是否多个配置
        boolean multiple = attributes.getBoolean("multiple");
        //注册配置bean，具体实现可自行查阅源码
        registerDubboConfigBeans(prefix, configClass, multiple, registry);
    }


2.@DubboComponentScan
    import了DubboComponentScanRegistrar类
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        //需要扫描的路径
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
        //注册处理暴露服务的bean：ServiceAnnotationBeanPostProcessor，下面有解释         
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);
        //注册处理引用服务的bean：ReferenceAnnotationBeanPostProcessor 下面有解释
        registerReferenceAnnotationBeanPostProcessor(registry);
    }
    
    
此外DubboAutoConfig也做了相同的事，其内部也调用了@EnableDubboConfig，并且还新增了对@Service（暴露服务ServiceAnnotationBeanPostProcessor）和@Reference（引用服务ReferenceAnnotationBeanPostProcessor）的处理
DubboAutoConfig在使用了@DubboComponentScan后不生效，因为@DubboComponentScan也做了相同的事
    1>.ServiceAnnotationBeanPostProcessor
        //根据扫描的路径，注册暴露的服务bean
        @Override
        public voidpostProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
            Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);
            if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
                //注册暴露的servicebean，其内部实现可自行查阅源码
                registerServiceBeans(resolvedPackagesToScan, registry);
            } 

    2>.ReferenceAnnotationBeanPostProcessor
        @Override
        public PropertyValues postProcessPropertyValues(
                PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
            //获取需要注入的类（引用的服务代理类）
            //具体实现可查阅源码
            InjectionMetadata metadata = findReferenceMetadata(beanName, bean.getClass(), pvs);
            try { 
                //依赖注入
                metadata.inject(bean, beanName, pvs);
            } catch (BeanCreationException ex) {
                throw ex;
            } catch (Throwable ex) {
                throw new BeanCreationException(beanName, "Injection of @Reference dependencies failed", ex);
            }
            return pvs;
        }
```