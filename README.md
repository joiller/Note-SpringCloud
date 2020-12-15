# Note-SpringCloud
# 预览
## 注意版本
springCloud对于springBoot有版本要求，具体版本要求可以在官网查看\
这里用的是`springCloud Hoxton.SR8`, 对应的`springboot`版本是`2.3.5.RELEASE`\
不过直接用springbootinitializer创建就好了，选择版本的时候会自动匹配的

添加好依赖，注意`dependencies`和`dependencyManagement`的区别

这里我遇到了个bug，是版本问题，同时找到了两个版本的xxx.class，并且使用了其中一个，结果并不行。\
是因为继承问题，就如maven官网所说的：
> 一个项目引入了`dep1`和`dep2`,`dep1`又依赖于`dep2`，
>我们给的`dep2`版本和`dep1`所需要的版本不符合，`dep1`当然只能选择我们给的版本，然后就会出错

在想想，看报错信息是说`servlet-api-2.5.jar`里面有个
`javax.servlet.ServletContext.class`,\
`tomcat-embed-core-9.0.33.jar`\里面也有`javax.servlet.ServletContext.class`，这两个冲突了。\
根据maven的tree来查看，发现`tomcat-embed-core-9.0.33.jar`是optional的，
也就是不调用他的时候不会被加载。\
根据maven mediation规则，会优先选用层级靠前的（依赖层级），如果层级一样，就会选用首先声明的。\
但是这里由于是`optional`，所以他要被优先声明才会被调用，否则不会被调用。\
`servlet-api-2.5.jar`是在
`spring-cloud-starter-netflix-eureka-server:jar`，\
`tomcat-embed-core-9.0.33.jar`是在`spring-boot-starter-web:jar`，\
所以我把`spring-boot-starter-web:jar`优先声明，
就会调用正确的`javax.servlet.ServletContext.class`。\
当然也可以在`spring-cloud-starter-netflix-eureka-server:jar`
里exclude掉`servlet-api`，也不会冲突
