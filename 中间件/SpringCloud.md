# SpringCloud

## `负载均衡`

* `Ribbon`

```
基于Netfix Riboon实现的一套客户端负载均衡工具
配置：
依赖spring-cloud-starter-ribbon
在连接对象加@LoadBalanced，比如RestTemplate
更换策略:
定义对应的策略，@Bean即可

负载：
继承 AbstractLoadBalancerRule implements IRule
最主要的是public Server choose(Object key) 方法，自定义负载均衡就是实现这个方法。

RoundRobinRule：轮询
RandomRule：随机
AvailabilityFilteringRule：过滤掉熔断、并发连接数超过阈值的服务，在剩下的服务中轮询
WeightedReponseTimeRule：根据响应时间速度计算权重，响应时间越快权重越高，如果服务刚启动，权重计算值不足，则按RoundRobinRule轮询，等信息够了再切换成WeightedReponseTimeRule
ReTryRule：先用RoundRobinRule，如果服务获取失败则在指定时间内重试，获取可用服务
BestAvailableRule：过滤掉熔断的服务，然后选择并发量最小的服务
ZoneAvoidanceRule：默认规则，复合判断Server所在区域的性能和Server的可用性服务

自定义负载均衡策略：
启动类添加：@RibbonClient(name="xxxxx",configuration=自定义Rule.class)
目的是服务启动时，不加载默认负载均衡策略，加载自定义负载均衡策略

自定义Rule类，需添加@Configuration
1.定义实现类
public class MyRule extends AbstractLoadBalancerRule{
    public Server choose(Object key) {
        //doSomething
    }
}
2.设置负载策略
@Configuration
public class MyRule2{
    @Bean
    public IRule myRule(){
        return new MyRule();
    }
}
注：自定义的Rule类，不能与包含@ComponentScan的类在同个包或者其子包下


配置：
ribbon.ReadTimeout=1000 //处理请求的超时时间，默认为1秒
ribbon.ConnectTimeout=1000 //连接建立的超时时长，默认1秒
ribbon.MaxAutoRetries=1 //同一台实例的最大重试次数，但是不包括首次调用，默认为1次
ribbon.MaxAutoRetriesNextServer=0 //重试负载均衡其他实例的最大重试次数，不包括首次调用，默认为0次
ribbon.OkToRetryOnAllOperations=false //是否对所有操作都重试，默认false
ribbon.MaxTotalConnections=500 //最大连接数
ribbon.MaxConnectionsPerHost=500 //每个host最大连接数
ribbon.retryableStatusCodes=500,404,502 //对Http响应码进行重试

也可以为每个Ribbon客户端设置不同的超时时间, 通过服务名称进行指定：
服务名.ribbon.ConnectTimeout=2000
服务名.ribbon.ReadTimeout=5000
服务名为RibbonClient的name属性值
```

* `Feign`

```
面向接口编程
启动类添加：@EnableFeignClients
依赖：spring-cloud-starter-feign
```

## 网关路由

* `Zuul`

```
启动类添加：@EnableZuulProxy
依赖：spring-cloud-starter-zuul
路由配置
zuul:
  prefix: /v1   //前缀，访问任何url都会在域名后增加此配置
  retryable: true //网关重试
  routes:
　　user-service:
　　　　path: /user/**
　　　　serviceId: user-service
　　　　url: http://localhost:8080 //路由到物理地址，与ServiceId两者取其一使用
　　　　strip-prefix: false  //去除路由前缀，默认为true，改为false即使user-serivce这条路由的/user路径不做去除，发送给真实路径,则访问地址为127.0.0.1:10010/user/16，因为匹配路径的/user/会发送给真实路径，但使用此功能无法简化路由条目配置

使用url配置，路径为：http://localhost:8080/user/**
使用serviceId，路径为：：http://localhost:8080/user-service/user/**
```

## 熔断
* `Hystrix`

```
使用：
1.@HystrixCommand 在方法上添加注解，指定回滚方法，可配置隔离策略
2.搭配feign使用，指定feign的熔断方法

两种隔离策略：线程池隔离和信号量隔离。默认线程池隔离
线程池隔离：不同服务通过使用不同线程池，彼此间将不受影响，达到隔离效果

信号量隔离：当服务的并发数大于信号量阈值时将进入fallback。

开启熔断
feign:
  hystrix:
    enabled: true
    
熔断配置超时时间：
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: true //是否开启超时，默认true
        isolation:
          thread:
            timeoutInMilliseconds: 10000 //设置超时时间
  threadpool:
      default:
        coreSize: 20 //最大线程数量，默认10
        queueSizeRejectionThreshold: 10 //允许在队列中的等待的任务数量，默认5
        

//单个服务并发请求数（信号量）
服务名.execution.isolation.semaphore.maxConcurrentRequests:100
```