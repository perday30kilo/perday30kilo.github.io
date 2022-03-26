# Sentinel


# 1 Sentinel是什么
主要特性  
![alt 主要特性](https://perday30kilo.github.io/sentinel1.png)   
开源生态  
![alt 开源生态](https://perday30kilo.github.io/sentinel2.png)    
Sentinel 分为两个部分:

核心库（Java 客户端）不依赖任何框架/库，能够运行于所有 Java 运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持。
控制台（Dashboard）基于 Spring Boot 开发，打包后可以直接运行，不需要额外的 Tomcat 等应用容器。

# 2 Quick Start

## 2.1 Sentinel Dashboard

```
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar
```
从 Sentinel 1.6.0 起，Sentinel 控制台引入基本的登录功能，默认用户名和密码都是 sentinel


## 2.2 如何使用
我们说的资源，可以是任何东西，服务，服务里的方法，甚至是一段代码。使用 Sentinel 来进行资源保护，主要分为几个步骤:

1 定义资源  
2 定义规则   
3 检验规则是否生效  

先把可能需要保护的资源定义好（埋点），之后再配置规则。也可以理解为，只要有了资源，我们就可以在任何时候灵活地定义各种流量控制规则。在编码的时候，只需要考虑这个代码是否需要保护，如果需要保护，就将之定义为一个资源。


返回布尔值方式定义资源



