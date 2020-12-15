# 什么是Hystrix
## 分布式系统的问题
我们会出现服务雪崩的问题：假设A调用B，B调用C,C调用D，这就是所谓的`扇出`，
如果其中一个出现了问题，响应时间过长或无法使用。那么A就会占用很多的资源，
引起系统崩溃，就是雪崩效应
## 解决方案
Hystrix是一个处理分布式系统的`延迟`和`容错`的开源库。能够保证在一个服务出现问
题的情况下，不会导致整个系统的崩溃\
`断路器`本身是一种开关，当某个服务发生故障，给出一个解决方案（Fallback）
，而不是长时间的等待或者抛出无法处理的异常

# 能干嘛
- 服务降级
- 服务熔断
- 接近实时的监控
- 。。。

# 我们的要求
A调用B
- B自己有个时间要求，B的执行超时了，我们需要返回一个超时信息
- A对B有时间要求，B的执行超时了，我们需要返回一个超时信息
- B发生了错误，我们需要返回一个错误信息

# 实现
## 直接给一个方法定义fallback
在启动方法上添加或者`@EnableCircuitBreaker`，在controller方法上添加\
`@HystrixCommand(fallbackMethod = "waitingFallback",commandProperties = {@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000")})`\
这里表示`fallbackMethod`就是发生问题时调用的方法，`HystrixProperty`表示如果超时3000ms则表示出现了问题，需要调用`fallbackMethod`\
当然如果发生异常，也会调用`fallbackMethod`

## 给方法定义默认fallback
我们如果都这样子写，就是每个方法都对应一个fallback，显得冗余。
这个`@HystrixCommand`里面有个属性叫`defaultFallback`，表示如果我们不指定
`fallbackMethod`，就会使用`defaultFallback`，我们可以在类上面添加一个
`@DefaultProperties(defaultFallback = "MyDefaultFallback")`，
应用于这个类里面的带有`@HystrixCommand`注解的所有方法。\
也就是说，如果方法上添加有`@HystrixCommand`，并且没有指定`fallbackMethod`，
就会使用这个方法，如果指定了，就会使用指定的。如果没有`@HystrixCommand`，
就无关。

## 通过实现feign接口给方法定义fallback
这是在consumer端的方法，我们在consumer端通过接口和feign来调用Provider端
的方法。我们给feign注解一个值\
`@FeignClient(value = "EUREKA-PROVIDER",fallback = ProviderServiceImpl.class)`\
表示feign接口如果调用出错，就会选择fallback类来执行。
```java
@Component
public class ProviderServiceImpl implements ProviderService {

    @Override
    public String waiting() {
        return "waiting Implementation failed";
    }

    @Override
    public String ok() {
        return "ok implementation failed";
    }
}
```

# 熔断
是在微服务链路中，当扇出链路的某个微服务出错或响应时间太长，会进行服务降级，\
进而熔断该微服务，快速返回错误信息。\
当该微服务恢复正常，则会恢复调用链路。

通过Hystrix实现，Hystrix会监控微服务的调用状况，当失败的调达到一定次数，
缺省是5秒内20次失败，就会启用熔断机制，熔断机制的注解是`@HystrixCommand`

## 熔断的表现
理解为`circuit`电路，电路为开的时候，就表示电路断了，电路为关闭的时候表示电路通了。\
电路有`close`，`Open`，`Half Open`，的情况。\
平时为`close`，\
触发了熔断后，为`Open`，此时对该URI的所有请求失败，会导向fallback，\
时间到了后，为`Half Open`，此时如果有请求成功了，则变为`Close`，\
否则变回`Open`
## 使用
我们在Provider使用：
```java
@RestController
public class ProviderController {

    @GetMapping("/positive/{num}")
    @HystrixCommand(fallbackMethod = "positiveFallback",
            commandProperties = {
                    @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),
                    @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
                    @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "15000"),
                    @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60"),

            })
    String positive(@PathVariable("num") Integer num) throws Exception {
        if (num < 0) {
            throw new Exception("num is negative");
        } else {
            return "your num is " + num + "!!!";
        }
    }

    String positiveFallback(Integer num) {
        return "you num is negative : " + num;
    }
}

```
> 以上属性：`commandProperties`接收一个数组，`@HystrixProperty`表示
> 一个属性
> circuitBreaker.enabled    表示是否开启熔断， 默认为false
> circuitBreaker.requestVolumeThreshold     表示多少个请求为基准
> circuitBreaker.sleepWindowInMilliseconds  表示要在该时间段内的这么多个请求
> circuitBreaker.errorThresholdPercentage   表示失败率达到多少时触发熔断


> 如果在最近15秒内，有10次请求，有6次是失败的（达到了阈值），就会触发熔断，15秒都是fallback
> 15秒后会变为`Half Open`状态，如果访问时没有出错，就是`Close`了，恢复正常

然后在Consumer中使用：
```java
@RestController
@DefaultProperties(defaultFallback = "MyDefaultFallback")
public class ConsumerController {
    @Resource
    private ProviderService providerService;

    String MyDefaultFallback(){
        return "MyDefaultFallback ------ failed";
    }

    @GetMapping("/consumer/positive/{num}")
    @HystrixCommand(commandProperties = {
            @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "15000"),
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60"),

    })
    String positive(@PathVariable("num") Integer num){
        return providerService.positive(num);
    }
}

```
> 原理与Provider上的一样
> 需要注意的是，如果Provider已经有了fallback，
> 而且我们客户端认为该fallback不算错误，那就不会触发Consumer中的fallback，
> 更不会触发Consumer的熔断。
> 触发的是Provider的fallback和熔断


# DashBoard
这是可视化工具\
我们新建一个项目，用来监控其他Hystrix项目的流量，\
需要添加`<artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>`和
`<artifactId>spring-boot-starter-actuator</artifactId>`依赖\
需要将被监控项目的host添加到
`application`中的`hystrix.dashboard.proxy-stream-allow-list`属性里。
然后当前版本有个bug：\
原本我们只需在可视化界面中输入`http://host:port/actuator/hystrix.stream`就可以了，
但是现在他会找不到，我们需要在被监控的程序中添加一个配置：
```java
@Bean
public ServletRegistrationBean getBean(){
    HystrixMetricsStreamServlet hystrixMetricsStreamServlet = new HystrixMetricsStreamServlet();
    ServletRegistrationBean servletRegistrationBean = new ServletRegistrationBean(hystrixMetricsStreamServlet);
    servletRegistrationBean.setLoadOnStartup(1);
    servletRegistrationBean.addUrlMappings("/actuator/hystrix.stream");
    servletRegistrationBean.setName("HystrixMetricsStreamServlet");
    return servletRegistrationBean;
}
```