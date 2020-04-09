# Spring Framework

```
建议配合spring官网源码介绍阅读
此笔记依据4.3.14版本记录
```
## IOC

* `bean的生命周期`


```
bean是怎么被实例化的？
1.spring容器启动，初始化GenericApplicationContext构造方法，构建bean工厂：DefaultListableBeanFactory

2.扫描配置的config类，封装成BeanDefinition放到BeanDefinitionMap

3.调用invokeBeanFactoryPostProcessors，扫描有@Component、@Service、@Bean等注解的类封装成BeanDefinition并放到BeanDefinitionMap中

4.如果有自定义的BeanFactoryPostProcessor(后置处理器)实现类，则执行实现的自定义逻辑

5.调用finishBeanFactoryInitialization，实例化bean，并放到单例池(map)中
    5.1 先拿到所有可能需要实例化的beanNames
    5.2 如果scope!=单例或者lazy=true或者是抽象类则不会被实例化，这种bean只有被用到的时候才会被实例化
    5.3 判断是否factorybean（特殊的bean，后面介绍）
    5.4 判断是否允许循环依赖
    5.5 普通bean，直接调用getBean(name)验证bean是否被实例化过，getbean方法是从signleObjects(单例池，map<beanname,bean>)中获取bean
    5.6 调用createBean创建对象，填充属性、注入依赖等等
    5.7 执行构造方法
    5.8 执行BeanPostProcessor
    5.9 执行回调(init)
    5.10 放入单例池
    5.11 对象销毁执行destroy回调
```

* `spring如何解决循环依赖`


```
1.产生循环依赖的问题
比如有2个类，A和B，互相引用
首先A被创建，经过一系列流程，到自动注入时获取B这个bean(getBean(B))
但此时B这个bean并没有被实例化(signleObjects中没有)，于是就会创建B，B也经过一系列流程，到自动注入时获取A(getBean(A))，发现A也没有被实例化(signleObjects中没有)，于是A又去找B...死循环

2.解决循环依赖的方法
比如有2个类，A和B，互相引用
    2.1 首先A被创建，此时A会被放到singletonsCurrentlyInCreation这个set集合中，表示该对象是在创建过程中
    2.2 经过一系列流程，到自动注入时获取B(getBean(B)),此时B并没有被实例化(signleObjects中没有)，就会创建B
    3.4 B经过一系列流程，到自动注入时获取A(getBean(A))，发现A正在创建过程中(isSingletonCurrentlyInCreation(A)为ture)，于是从earlySingletonObjects(存放对象集合，map)中获取到A，因为实例化B并不需要A这个bean，只需要A对象
    2.3 此时B就能被实例化成bean，A接下来获取到B也能实例化
    
注：earlySingletonObjects是存放对象的，singletonObjects是存在bean的
```

* `bean与对象的区别`


```
bean：有一套完整的spring生命周期，spring会对其进行比如aop、初始化前、初始化后、销毁前、自动注入等等的操作
对象：就是一个普通的java对象
```

* `BeanDefinition`

```
BeanDefinition：用来记录装配bean的各种信息，是否需要懒加载isLazy、class、注入模型autoModel(byType,byName)、父类的名称parentName、是否抽象类isAbstract等、beanName、FactoryMethod等等
```

* `关闭是否允许循环依赖`


```
//默认为ture，设置为false关闭
AbstractAutowireCapableBeanfactory.allowcularReferences=false;
```

* `生命周期回调` Lifecycle Callbacks

```
1.bean实例化后初始化
  1.1 实现InitializingBean.afterPropertiesSet()
  1.2 <bean id='xxx' init='init'>
  1.3 使用@PostConstruct作用于方法上
  注：1.1与1.2是调用InvokeInitMethods()，1.3是在BeanPostProcessor实现
2.bean实例化后销毁
  2.1 实现DisposableBean.destroy()
  2.2 <bean id='xxx' destroy-method='destroy'>
  2.3 使用@PreDestroy作用于方法上
```


### aop

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
实现BeanPostProcessor, 在ioc doCreateBean() 方法中调用实现BeanPostProcessor的实现类，完成对bean的aop增强并返回增强的代理对象(bean)。

通过aopProxy 生成aop代理对象。
aopProxy 有2个实现类 JDKDynamicAopProxy 和  cglibAopProxy
通过bean，执行不同的代理方式
```

* `cglib`

```
```