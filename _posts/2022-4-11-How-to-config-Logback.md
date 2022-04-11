---
layout: post
title: 如何本地调试Kubernetes集群上的Spring Cloud微服务
---

一种常见的方法——在本机上搭建一套开发环境，哪怕只是其中的一个微服务出现问题了，也要把整个系统完整地跑起来。以`a-service`为例，如果要调试、修复`a-service`的bug，除了要启动`a-service`本身之外，还要启动Eureka、Config、Gateway等等。

费事！费时！费力！

能不能不启动Eureka、Config这些中间件，直接使用Kubernetes集群里运行好的呢？答案当然是可以的，具体的方法如下。

## 配置Eureka

在Kubernetes集群里，`a-service`通过域名`k8s-eureka`（假设）访问Eureka实例。域名`k8s-eureka`只在集群内生效，在集群外无法使用。在本地开发环境中，如果要访问集群内的Eureka实例，就要把Eureka的IP、端口暴露出来，方法是创建一个`NodePort`或者`LoadBalancer`类型的`Service`。假设外部访问地址为`http://my-eureka:8761`，则按如下方式修改`application.yml`。

```
eureka:
  instance:
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://my-eureka:8761/eureka/
```

## 配置Config

一般情况下，`a-service`是通过Eureka来连接Config，即：

```
spring:
  config:
    import: "optional:configserver:"
  cloud:
    config:
      discovery:
        enabled: true
        service-id: SPRING-CLOUD-CONFIG
      label: master
      name: a-service
      profile: default
```

要知道，Eureka和Config都是运行在Kubernetes集群里面，**通过Eureka获得的Config的IP、端口**，都是只在集群内生效，在集群外无法使用。因此，在本地开发环境中，如果要访问集群内的Config实例，就不能通过Eureka来访问。只能是先把Config实例的IP、端口暴露出来，然后通过这个IP、地址（假设为：`http://my-config:8888`）直接连过去，即：

```
spring:
  config:
    import: "optional:configserver:http://my-config:8888"
  cloud:
    config:
      label: master
      name: a-service
      profile: dev
```

## 修改`a-service`的服务名

为什么要把服务名`a-service`重命名为`a-service-dev`？原因是本地的`a-service`与集群的`a-service`最终都是注册到同一个Eureka上，如果不区分两者的服务名，就无法确定实际调用的是哪一个`a-service`。代码如下：

```
spring:
  application:
    name: a-service-dev
```

## 修改`a-service`的客户端

修改`a-service`的客户端的原因同上，代码如下：

```
@FeignClient(name = "A-SERVICE-DEV", path = "/a")
public interface AClient {
...
}
```
