---
layout: post
title: 如何在Logback配置文件中处理条件判断？
---

在关闭`EFK`之后，代码会在控制台输出如下错误日志：

```
o.fluentd.logger.sender.RawSocketSender  : org.fluentd.logger.sender.RawSocketSender java.net.ConnectException: 拒绝连接 (Connection refused)
```

虽然没有影响到系统正常运行，但是给查错带来了很大的不便。

有没有一种方式能根据环境变量、运行参数、`application.yml`等，来决定是否启用`EFK`相关的`Appender`？

**Logback**官方文档[Conditional processing of configuration files](https://logback.qos.ch/manual/configuration.html#conditional)一节给出了答案！

可以在`<configuration>`元素下，使用`<if>`、`<then>`、`<else>`等来处理条件判断：

```
<if condition="">
    <then>
      ...
    </then>
    <else>
      ...
    </else>
</if>
```

`condition`中是一个返回`String`对象的表达式，可以借助内置的`property()`或者`p()`方法读取上下文参数、系统参数，如果参数不存在，就返回空字符串，而不是`null`。`isDefined()`用来判断是否定义了某个变量，`isNull()`用来判断某个变量的值是否为null。

读取环境变量

读取上下文参数

读取application.yml参数





