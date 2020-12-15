# 现状
慢慢淘汰中，但是仍然在很广泛应用中。将来会被springcloud的负载均衡代替。
# 负载均衡
有两种负载均衡，一种是外部的，一种是内部的
Ribbon属于内部的。
以eureka为例，consumer上的eureka会根据负载均衡，选择不同的server，然后再根据负载均衡，选择不同的实例

# 使用
Ribbon的使用通常要配合`restTemplate`

# 负载均衡规则
自带有个`IRule`接口，有多个规则实现类。

> 卧槽烦死了，Ribbon已经不更新了，我现在的项目都是用的最新版本，我不会实现Ribbon，所以就算了吧。最后在写一个老版本的有Ribbon的项目好了

## 自定义规则
我们的自定义规则不能放在springboot启动类的相同包下，因为`@SpringBootApplication`注解里面有个`@ComponentScan`，会扫描包下的所有类，作为组件。
这样子我们的规则类会被所有的`@RibbonClients`共享

## 自己实现规则
### 实现轮询规则
原理：
根据service获得service instance列表 `instances`\
instance列表长度为 `total`\
记录请求次数 `count`\
`count % total`作为`index`，\
`instances.get(index)`就是这一次请求所请求的服务实例\
代码：
```java
package com.example.consulconsumer9021.lb;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * @author jianghuilai
 * @create 2020-12-07 19:30
 **/
@Component
public class MyLoadBalancerImpl implements MyLoadBalancer {

    private AtomicInteger atomicInteger;  // 请求总次数

    public MyLoadBalancerImpl() {
        this.atomicInteger = new AtomicInteger(0);
    }

    /**
     * 利用到了CAS自旋锁，保证能够让atomicInteger成功+1
     * cas,就是与current进行compare，如果相等则set为next，并且返回true
     * 如果不相等则返回false
     **/
    public final int getAndIncrement(){
        int current;
        int next;
        do {
            current = this.atomicInteger.get();
            next = current+1;
        }while (!this.atomicInteger.compareAndSet(current, next));
        return current;
    }

    @Override
    public ServiceInstance choose(List<ServiceInstance> serviceInstances) {
        int index = this.getAndIncrement();
        return serviceInstances.get(index%serviceInstances.size());
    }
}

```