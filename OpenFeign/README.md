# OpenFeign是什么
`OpenFeign`是个声明式的web service 客户端。
创建一个接口，并且使用feign去注解。就可以更加简单的使用网络服务
## 举个例子
我们之前使用`Ribbon`+`RestTemplate`去实现负载均衡的网络请求。现在我们可以使用feign实现它
# 如何使用
我们创建一个关于controller的接口，然后在该接口上添加feign注解
```java
@Component
@FeignClient("PROVIDER")  // 表示我们请求的是PROVIDER这个service
public interface OrderService {
    @GetMapping("/payment/get/{id}")
    CommonResult<Payment> get(@PathVariable("id") Long id);

    @GetMapping("/wait")
    CommonResult wait() throws InterruptedException;
}
```

在Provider中对应这个接口的方法是：
```java
@RestController
@Log4j2
public class PaymentController {
    @Value("${server.port}")
    private String port;

    @Resource
    private PaymentService paymentService;

    @GetMapping("/payment/get/{id}")
    CommonResult<Payment> selectById(@PathVariable("id") Long id){
        Payment payment = paymentService.selectById(id);
        if (payment == null){
            return new CommonResult<>(404, "get failed from "+port);

        }
        return new CommonResult<>(200, "get ok from "+port, payment);
    }

    @GetMapping("/wait")
    CommonResult wait() throws InterruptedException {
        TimeUnit.SECONDS.sleep(3);
        return new CommonResult();
    }
}

```
> 这个接口的各个方法就类似于Dao和mapper的关系，
> 这个接口就类似于Dao， 在Provider中的controller就类似于mapper
> 就是说feign会通过这个接口去找到Provider中的对应方法
> 这个接口里的`@GetMapping`注解对应的是Provider的`@GetMapping`

然后我们可以通过这个接口实现负载均衡了

我们使用这个接口:
```java
@RestController
public class OrderController {

    @Resource
    private OrderService orderService;

    @GetMapping("/consumer/payment/get/{id}")
    public CommonResult<Payment> get(Long id) {
        return orderService.get(id);
    }

    @GetMapping("/consumer/wait")
    public CommonResult wait() throws InterruptedException {
        return orderService.wwait();
    }
}
```
# 自定义配置
主要在`proerties`/`yml`中修改：
```yaml
feign:
  client:
    config:
      feignName:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full
        errorDecoder: com.example.SimpleErrorDecoder
        retryer: com.example.SimpleRetryer
        requestInterceptors:
          - com.example.FooRequestInterceptor
          - com.example.BarRequestInterceptor
        decode404: false
        encoder: com.example.SimpleEncoder
        decoder: com.example.SimpleDecoder
        contract: com.example.SimpleContract
```
> `feignName`就是`@FeignClient("PROVIDER")`中指定的name属性，如果没有name属性，那就是value属性

# logging
我们在application中设置logging.level:
```yaml
logging:
  level:
    com.example.eurekafeignconsumer9031.service.OrderService: debug
```

然后我们指定`feign.Logger.Level`
这个`Logger.Level`表示的是我们要log的信息量：
- NONE, No logging (DEFAULT).
  
- BASIC, Log only the request method and URL and the response status code and execution time.
  
- HEADERS, Log the basic information along with request and response headers.
  
- FULL, Log the headers, body, and metadata for both requests and responses.

```java
@Configuration
public class FooConfiguration {
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```