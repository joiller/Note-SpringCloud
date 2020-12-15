# 是什么
类似于eureka
有注册中心，有client

# 开启注册中心
与eureka不同的是，consul的注册中心不需要在java项目中启动。\
可以直接在命令行中启动
`consul agent -dev`

然后就可以在`http://localhost:8500`开启和eureka差不多的注册中心监控了

# Provider
添加consul依赖。

注册consul `@EnableDiscoveryClient`

配置：
```yaml
server:
  port: 8021
spring:
  application:
    name: consul-provider-app-name-9021
  cloud:
    consul:
      discovery:
        instance-id: consul-provider-instance-9021
```

然后再简单的写几个controller

启动后，服务会注册在注册中心里。


# Consumer
添加consul依赖

注册consul `@EnableDiscoveryClient`

```yaml
server:
  port: 9021
spring:
  application:
    name: consul-consumer-app-name-9021
  cloud:
    consul:
      discovery:
        instance-id: consul-consumer-instance-9021
```

添加几个controller

可以通过负载均衡去访问Provider的服务，用`ribbon`或者`openfeign`都是可以的