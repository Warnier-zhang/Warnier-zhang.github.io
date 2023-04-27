---
layout: post
title: 微服务架构升级一：配置中心
---

在Spring Cloud微服务开发的体系下，主推的配置中心是Spring Cloud Config。

Spring Cloud Config支持使用文件系统、Git等等来保存`application.yml`（或`application.properties`）中的参数，根据`spring.profile.active`参数值（`default/dev/test/prod`）的不同来区分开发、测试、生产环境。

一开始，我们也是用Spring Cloud Config + 文件系统，具体的方案是：

> 在测试环境、生产环境中，分别部署一个ConfigServer，`application.yml`以文件的形式保存在CephFS中，通过Kubernetes PV挂载到Pod中供ConfigServer访问。`spring.profile.active`参数的值都是`default`。
>
> 此外，本地开发环境和测试环境共用一个ConfigServer，因此，测试环境额外有`XXX-dev.yml`等文件。

上述方案有如下的缺点：

1. 存在多个副本。

   一处变动需要修改多个地方，可能造成开发、测试、生产环境的`application.yml`不一致。

2. 无法在线编辑。

   如果要修改`application.yml`，首先得在本地编辑好，然后上传到测试、生产环境。

3. 没有用上`spring.profile.active`参数；

4. 占用系统资源。

   部署了多个ConfigServer。
   
4. 动态刷新复杂。

   如果启用Spring Cloud Config的动态刷新服务，还要再部署一个RabbitMQ或者Kafka。

当然，测试、生产环境可以通过共用一个ConfigServer来克服缺点1和缺点3。但是，缺点2、缺点4和缺点5是靠Spring Cloud Config自己解决不了的。

为了克服上述5个缺点，同时，考虑到运行环境是Kubenetes，决定用Spring Cloud Kubernetes + Kubenetes ConfigMap替换Spring Cloud Config。

ConfigMap是Kubernetes内置的一种API对象，无需安装部署，被用来保存Key-Value，底层的技术是etcd。通过Lens、Kuboard等等图形化管理工具可以在线编辑ConfigMap的属性和值。Spring Cloud Kubernetes提供**Reload**机制来跟踪ConfigMap属性值的变化。

### 配置Spring Boot

在`classpath`下新增`bootstrap.yml`文件：

```
spring:
  application:
    name: XXX

  cloud:
    kubernetes:
      config:
        namespace: default
        sources:
          - name: application
          - name: ${spring.application.name}
```

`spring.cloud.kubernetes.config.name`和`spring.cloud.kubernetes.config.namespace`设置该应用配置参数默认的名称和命名空间，名称和ConfigMap的`metadata.name`属性一一对应。

在Spring Cloud Config中，默认的、共用的参数都被保存在XXX[-default].yml（XXX是`application`和`spring.application.name`）中。举个例子，如果`spring.profile.active`的值是`dev`，就会依次读取`application.yml`、`application-dev.yml`、`XXX.yml`、`XXX-dev.yml`文件。如果出现同名的参数，就会后面的覆盖前面的。

在Spring Cloud Kubernetes中，可以通过`spring.cloud.kubernetes.config.sources`参数来配置多个ConfigMap实现类似的效果。

### 部署ConfigMap

对于默认、开发、测试、生产环境，就有`XXX.yml`、`XXX-dev.yml`、`XXX-test.yml`、`XXX-prod.yml`等4套参数，可以都保存在同一个ConfigMap中，使用`spring.profiles: default/dev/test/prod`区分，

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: XXX
data:
  application.yml: |-
    msg: "Hello, World"
    ---
    spring:
      profiles: dev
    msg: "你好，世界"
    ...
```

也可以分别保存在不同的ConfigMap中，

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: XXX
data:
  XXX.yml: |-
    msg: "Hello, World"
```

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: XXX-dev
data:
  XXX-dev.yml: |-
    spring:
      profiles: dev
    msg: "你好，世界"
```

……

推荐后面这种方式！可以先把参数写到一个文件中，然后使用如下脚本创建ConfigMap：

```
kubectl create configmap XXX --from-file=~/config/XXX.yaml
```

为了让应用能在运行时确定使用哪套参数，需要在Kubernetes Deployment文件中声明环境变量`SPRING_PROFILES_ACTIVE`。

### 动态刷新

设置`spring.cloud.kubernetes.reload.enabled=true`即可启用动态刷新服务：

```
spring:
  cloud:
    kubernetes:
      reload:
        enabled: true
        strategy: restart_context
        mode: polling
        period: 15000
```

`spring.cloud.kubernetes.reload.strategy`有3种策略：

- `refresh`：只替换应用上下文中被`@ConfigurationProperties`和`@RefreshScope`注解标注的Bean；

- `restart_context`：重启应用；

  如果使用`restart_context`策略，就要配上`spring-boot-starter-actuator`，并按照如下方式配置：

  ```
  management:
    endpoint:
      restart:
        enabled: true
    endpoints:
      web:
        exposure:
          include: restart
  ```

- `shutdown`：关闭应用；

`spring.cloud.kubernetes.reload.mode`有2种模式：

- `event`：事件；
- `polling`：定时作业；







