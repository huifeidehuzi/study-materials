# Mybatis

```
本文主要是了解
1.mybatis的mapper是被实例化
2.整合Spring时托管给Spring的原理
版本：v3.4.6
```

* `mapper如何被实例化`

```
JDK动态代理

通过SqlSessionManager(实现SqlsessionFactory) 获取mapper对象
通过MapperProxyFactory.newInstance() 生成mapper代理对象
通过MapperProxy(实现动态代理)invoke() 执行sql
通过Executor(默认BaseExecutor) 执行sql
```


* `如何将mapper托管给Spring`

```

1.使用@MapperScan 或者 在xml配置扫描
    容器启动，MapperScannerRegistrar扫描，扫描到的mapper会注册为BeanDefinition，后续流程与IOC 注册bean的流程一致
    ImportBeanDefinitionRegistrar实现ImportBeanDefinitionRegistrar,ResourceLoaderAware
2.实现FactoryBean，在getObject中设置，mybatis使用的就是这种，比如@MapperScan中默认的就是MapperFactoryBean，也可以自定义
3.实现BeanFactory.registerSingletion()

```