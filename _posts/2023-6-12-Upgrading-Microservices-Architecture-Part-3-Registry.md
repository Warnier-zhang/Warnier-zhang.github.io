---
layout: post
title: 微服务架构升级三：注册中心
---

有了前面两篇文章：

- [微服务架构升级一：配置中心](https://warnier-zhang.github.io/Upgrading-Microservices-Architecture-Part-1-Config/)；
- [微服务架构升级二：API网关](https://warnier-zhang.github.io/Upgrading-Microservices-Architecture-Part-2-Gateway/)；

作为引子，现在引出**微服务架构升级最后一篇——注册中心**；注册中心、配置中心、API网关是微服务开发三大核心组件，既然后两者都被成功改造了，当然也不能忽略掉前者。

![Eureka架构](../images/2023/6/12/1.png)

如图所示，在使用Eureka作为注册中心的时候，服务提供和消费的过程如下：

1. <u>服务提供者</u>把自身注册到<u>Eureka服务器</u>；
2. <u>服务消费者</u>通过<u>Eureka客户端</u>获取<u>Eureka服务器</u>上的服务列表；
3. <u>服务消费者</u>通过<u>Feign客户端</u>调用某个服务；

简而言之，就是服务注册、服务发现、服务消费，改造注册中心的关键是处理好这3个问题，答案就是**使用Kubernetes Service替换Eureka**！

为了使外部用户能够访问集群中的Pod，Kubernetes引入了一个新概念——**Service**，Service是一种抽象的、逻辑的概念，一个Service通过**<u>标签</u>**和**<u>端口</u>**绑定了一组Pod，起到负载均衡的作用。Service和Pod之间是松耦合的，即使Pod被销毁或重新创建，Service也保持不变。当某个Service被创建之后，可通过<u>**名称**</u>（完整路径格式：`服务名称.命名空间名称.svc.集群名称.local`）访问该Service。

### 服务注册

当一个API服务被部署到Kubernetes集群之后，该API服务就会被自动注册到API Server了。因此，无需再注册。

### 服务发现

执行`kubectl get svc -n XXX`命令可以查看Kubernetes集群中已部署的服务列表；

Spring Cloud Kubernetes提供一个适配Kubernetes的`DiscoveryClient`类，可以通过`getServices()`方法获取服务列表，还可以通过`getInstance()`方法获取服务详情。

### 服务消费

通常，我们使用`Feign客户端`调用微服务，也可以用`RestTemplate`等等。

当使用Eureka作为注册中心时，Feign客户端的配置如下：

```
@FeignClient(name = "A-SERVICE", path = "/a")
public interface AClient {
    @GetMapping("/hello")
    String sayHello(@RequestParam("name") String name);
}
```
其中，`name`属性的值设置为`微服务名称`，借助Ribbon等实现负载均衡。

在使用Kubernetes Service替换Eureka之后，必须把Feign客户端的`name`和`url`属性的值均设置为`http://Service名称:端口号`：

```
@FeignClient(name = "http://a-service:8080", url = "http://a-service:8080", path = "/a")
public interface AClient {
    @GetMapping("/hello")
    String sayHello(@RequestParam("name") String name);
}
```

当然，若使用`Spring Cloud Kubernetes LoadBalancer`，则`name`属性的值可简化为`Service名称`；
```
@FeignClient(name = "a-service-svc", path = "/a")
public interface AClient {
    @GetMapping("/hello")
    String sayHello(@RequestParam("name") String name);
}
```
