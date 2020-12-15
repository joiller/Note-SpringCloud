@[toc](笔记)
# 预览
# 普通结构
- parent
    - eureka-server
    - service-provider
    - service-consumer

server的配置
```yaml
server:
  port: 7001
eureka:
  client:
    fetch-registry: false         # 不去检索服务
    register-with-eureka: false   # 不注册进eureka，因为自己就是注册中心
    service-url:                  
      defaultZone: http://localhost:7001/eureka # 不要漏掉/eureka
spring:
  application:
    name: server7001
```
> 也就是将自己的

Provider的配置
```yaml
server:
  port: 8001
eureka:
  instance:
    instance-id: provider8001   # 可以在eureka服务主页上看到的具体实例的id
  client:
    service-url:
      defaultZone: http://locahost:7001/eureka  # 不要漏掉/eureka
mybatis:
  type-aliases-package: entity  # 就相当于mapper的配置文件中的typealias的package属性
  mapper-locations: classpath:mapper/*.xml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ex5
    username: root
    type: com.alibaba.druid.pool.DruidDataSource
  application:
    name: provider

```

Consumer的配置
```yaml
server:
  port: 9001

spring:
  application:
    name: consumer
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:7001/eureka # 不要漏掉/eureka
  instance:
    instance-id: consumer9001
```
在每个程序启动类上面添加`@EnableEurekaServer`或`@EnableEurekaClient`注解

使用Consumer通过eureka访问到Provider，我们需要利用一个`RestTemplate`类。
域名是eureka主页里面的`Application`下面的名字，也可以自己在yml中设置：`eureka.client.instance.hostname`，eureka就会根据负载均衡的原则，自动选择该application下的任一instance去执行
域名也可以是具体的instance在eureka下所对应的域名，那就没有负载均衡了
# 集群
多个eureka-server
这里面多个server之间互相注册
```yaml
server
  port: 7001
eureka:
  client:
    service-url:
      defaultZone: http://locahost:7002/eureka
```
```yaml
server
  port: 7002
eureka:
  client:
    service-url:
      defaultZone: http://locahost:7001/eureka
```

在client中同时注册多个server
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://locahost:7001/eureka,http://localhost:7002/eureka
```
# 负载均衡
将多个client注册进入eureka server
要求`spring.application.name`相同
需要请求的路径就不能是具体的instance在eureka下对应的域名了，而是eureka的application的名字
# 自我保护机制
默认是开启自我保护机制的，如果关闭的话，就会：每隔一段时间确认服务的存活，如果在一定时间内没有受到实例的心跳（默认90s），则会移除掉该实例
但是服务没有回应有可能是网络波动，这种情况下我们不希望eureka移除掉实例
自我保护就是在网络波动的情况下，保护整个eureka集群不产生不希望的变化（如移除实例就是不希望的变化）
## 启动自我保护机制
`eureka.server.enable-self-preservation=true`，默认就是true。
启动后，如果15分钟里面，超过85%的实例都没有正常的心跳，就认为出现了网络问题，进入自我保护。
> 1，不会移除没有心跳的实例
> 2，仍然会接收新服务的注册和查询请求，但是不会同步到其他节点上
> 3，网络稳定时，就会同步

建议在开发环境中关闭自我保护机制，修改检查失效服务的时间间隔：
```yaml
eureka:
  server:
     enable-self-preservation: false
     eviction-interval-timer-in-ms: 3000
```
建议在生产环境中打开
```yaml
eureka:
  server:
     enable-self-preservation: true
```