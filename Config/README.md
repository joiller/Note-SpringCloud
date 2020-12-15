# 什么是config
在分布式中，我们有许多的配置文件，不好修改。\
spring cloud 提供了统一修改 配置文件的方案 Config

# 原理

让`application(config-client)`都指向`config-server`,`config-server`指向
git仓库。

# 实现方法
## git仓库
所以我们要先有个git仓库交给`config-server`去引用\
这个git里面需要有`config-dev.yml`,`config-prod.yml`,`config-test.yml`
等文件就行

## config-server
首先添加依赖config-server

然后在springboot上注册，`@EnableConfigServer`

配置yml：

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: git@github.com:joiller/spring-cloud-config.git
          search-paths:
            - spring-cloud-config
          passphrase: passphrase

      label: master
  application:
    name: spring-cloud-config7007
```

 我们可以像这样请求资源：
  - /{application}/{profile}[/{label}]
  - /{application}-{profile}.yml
  - /{label}/{application}-{profile}.yml
  - /{application}-{profile}.properties
  - /{label}/{application}-{profile}.properties
> 请求的是git中label分支的`application-profile.properties/yml`资源
> 请求到的application会作为`spring.config.name`进入到server的配置中


## config-client
首先添加依赖config-client

### bootstrap.yml
`application.yml`是用户级的资源配置\
`bootstrap.yml`是系统级的资源配置，优先级更高，优先加载他

springcloud会创建一个`bootstrap context`，作为`application context`的
`parent context`。`bootstrap context`负责从外部加载并解析配置。这两个`context`
共享同一个从外部得到的`Environment`

bootstrap.yml:
```yaml
spring:
  cloud:
    config:
      name: config
      profile: dev
      label: master
      uri: http://localhost:7007
  application:
    name: config-client
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
  instance:
    instance-id: config-client9011
server:
  port: 9011
```
> 注意这里`spring.cloud.config.uri`不能用eureka的负载均衡动态加载。要写静态uri
> `uri`表示我们要从哪个`config-server`去拿来配置
> `name`, `profile`, `label`都是表示config-server的具体位置
>  `name`就是`config-server`在获取git配置的时候的`application`，会自动加入到`config-server`的`spring.config.name`中

### 动态更新配置
我们要求在GitHub上更新了配置后，我们的config-client不需要重启就能更新配置

我们需要引入`spring-boot-starter-actuator`依赖，这个依赖是用来监控的，
可以使自己的变化被监控到

然后在配置文件上添加：
```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

然后我们要在`controller`上添加注解：`@RefreshScope`

接下来就可以通过对`ClientHost:Port/actuator/refresh`发送post请求，来实现更新了

# bug
主要是git连接问题：\
config用的是`JGit`来连接git。
## invalid private key
在用SSH的方式连接git的时候，需要`private key`

如果没有指定的话，就会查找`~/.ssh/known_hosts`和`/etc/ssh/ssh_config`来找
配置。

通过`ignoreLocalSshSettings: true`来取消默认查找配置。`private key`就是
`~/.ssh/id_rsa`里的内容。

然后我们自己指定`private key（在yml中）`的话，那就是我们复制的`private key`有问题

`private key`的格式在官网有，要`-----BEGIN RSA PRIVATE KEY-----`， 
如果是`-----BEGIN OPENSSH PRIVATE KEY-----`的话，是不会被导入的，就会报错。

我们的`~/.ssh/id_rsa`里面如果是`-----BEGIN OPENSSH PRIVATE KEY-----`，
我们要通过`ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`来
将他改成`-----BEGIN RSA PRIVATE KEY-----`，才能被`JGit`识别。

## USERAUTH fail
这个一般就是`passphrase`错误或者没写。