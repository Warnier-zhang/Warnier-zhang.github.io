---
layout: post
title: Fluentd入门
---

## 介绍

Fluentd是一款由Treasure Data, Inc开发并开源的，面向**统一日志记录层（Unified Logging Layer）**的数据收集程序。

Fluentd是用C和Ruby语言开发的。

### 架构

![Fluentd架构](../images/2022/4/12/1.png)

**事件处理流：Input -> filter 1 -> ... -> filter N -> Output。**

首先，**Input **捕获**外部输入的数据**，并把它封装成Fluentd特有的**Event**；然后，**Fluentd engine**对这个**Event**应用一系列的**Filter（过滤器）**，**Filter**根据预定义好的规则决定是接受，还是抛弃这个**Event**；如果最终的结果是接受这个**Event**，接下来，**Fluentd engine**就会把这个**Event**传递给**Output**，**Output**负责（使用**Buffer（缓存）**）把**Event**的内容输出到给定的输出源；

核心组件：

- Event：事件，包括`tag`、`time`、`record`等3个属性，`tag`属性是`.`分隔的字符串，`record`属性对应着外部输入的数据；
- Input：对应`<source>`元素；
- Filter：根据Input的`tag`属性，匹配对应的Filter；
- Match：根据Input的`tag`属性，匹配对应的Output；
  - Output；
    - Buffer；
- Label：对应Input的`@label`属性和`<label>`元素，`<label>`元素用来把Filter和Output组合起来。Label的作用类似于GOTO语句，如果Input配置了`@label`属性，Fluentd engine就会**忽略公共的Filters**，**直接把Event传递给对应的`<label>`元素**；

## 安装

推荐使用Docker安装，**默认的镜像没有Elasticsearch Output插件**，需要自行安装，Dockerfile如下：

```
FROM fluent/fluentd:v1.14.5-debian-1.0
USER root
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-document", "--version", "5.2.1"]
USER fluent
```

然后，编写docker-compose.yml：

```
version: "3"
services:
  fluentd:
    build:
      context: ./fluentd
    container_name: fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
    links:
      - "elasticsearch"
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    depends_on:
      - elasticsearch
```

最后，运行命令：

```
docker-compose up -d
```

## 配置

Fluentd允许我们自定义输入源（Input）、过滤器（Filter）、输出源（Output），以及事件（Event）在这些组件之间传递的路由规则。

如果是使用Docker安装Fluentd，配置文件的路径就是`/fluentd/etc/fluent.conf`。

### 输入源（Input）

Input对应`<source>`元素，常见的Input插件有：

- in_forward

  in_forward插件监听一个TCP（/UDP）端口，用来接收Fluentd客户端（如：fluent-logger-java）或者其他Fluentd实例发送的数据；

- in_tail

- in_http

### 过滤器（Filter）

Input对应`<filter XXX>`元素，`XXX`用来匹配Event的`tag`属性。

### 输出源（Output）

`<match XXX>`元素用来配置输出源，，`XXX`用来匹配Event的`tag`属性，`@type`参数用来指定输出源类型，如：终端，

```
<match XXX>
  @type stdout
</match>
```

常见的输出源如下：

- out_copy：用来把Event同时输出到多个不同的地方。`<store>`元素是`<match>`的子元素，当且仅当 `@type copy`时生效，`<store>`元素的配置方式和`<match>`元素相同；
- out_stdout：终端；
- out_file：文件；
- out_kafka2：Apache Kafka；
- out_elasticsearch：Elasticsearch（EFK技术栈）；
- out_mongo：MongoDB；

**Event的`tag`属性的匹配规则**：

- 完全匹配；
- **\***：匹配单个字符串，如：a.*，能匹配到a.b、a.bc、不能匹配a.b.c；
- **\****：匹配一个或多个`.`号分隔的字符串，如：a.**，能匹配a、a.b、a.b.c等；
- {a,b,c}：匹配a、b、c中的任意一个；
- /regex/：匹配正则表达式；

在同一个配置文件（`fluent.conf`）中，可以配置一个或多个`<match>`元素。**如果一个Event匹配到2个及以上的`<match>`元素，那么就按照`<match>`元素在配置文件中定义的顺序，排在前面的生效！**（这也意味着排在后面的永远不会生效！）

## 实例

一个匹配所有的Event，并且把Event同时输出到Elasticsearch和终端的例子如下：

```
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<match **>
  @type copy
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    user elastic
    password 123456
    scheme https
    ssl_verify false
    logstash_format true
    <buffer>
      flush_interval 5s
    </buffer>
  </store>
  <store>
    @type stdout
  </store>
</match>
```

