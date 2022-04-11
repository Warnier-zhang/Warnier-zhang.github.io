---
layout: post
title: Logback配置
---

Logback配置文件（`logback.xml`或`logback-spring.xml`）的结构如下：

![配置文件结构](../images/2022/4/11/1.png)

配置文件的根元素是`<configuration>`，`<configuration>`元素下有一个或多个`<appender>`、 `<logger>`元素，除此之外，有且仅有一个`<root>`元素。

Logback由3个核心组件**Logger**、**Appender**、**Layout**组成。

## Logger

在配置文件（`logback-spring.xml`）中，Logger（日志收集器）对应`<logger>`元素。`<logger>`元素有3个属性：

1. name：名称；
2. level：日志级别，可选值有`TRACE`、`DEBUG`、`INFO`、`WARN`、`ERROR`、`ALL`等等，大小写不敏感；
3. additivity：是否输出日志到<u>rootLogger配置的Appender</u>中，可选值有`true`、`false`，默认值为`true`。若值设为`false`，则只输出日志到<u>当前的`<logger>`元素配置的Appender</u>中。

`<logger>`元素下有一个或多个`<appender-ref ref="XXX">`元素，`<appender-ref ref="XXX">`元素用来匹配Logger的日志输出源。

### rootLogger

rootLogger（根日志输出器）是一个特殊的Logger，用来配置公共的日志级别、日志输出源。对应`<root>`元素，配置和`<logger>`元素大同小异，相同的地方是有一个或多个`<appender-ref ref="XXX">`子元素，不同的地方是只有level属性，没有name、additivity属性。

## Appender

`<appender>`元素用来配置日志输出源，常见的日志输出源有：

- 终端；
- 文件；
- 数据库，如：MySQL、PostgreSQL、Oracle等；
- 消息队列；
- UNIX/Linux系统日志；

![Appender配置元素结构](../images/2022/4/11/2.png)

`<appender>`元素有2个属性：

1. name：名称，一般大写；
2. class：日志输出源实现类名（含包名）；

`<appender>`元素下有一个或多个~~`<layout>`~~、`<encoder>`、`<filter>`等元素。

~~`<layout>`~~、`<encoder>`元素用来配置日志输出的格式、内容。

`<filter>`元素用来过滤日志内容，不常用，本篇文章暂不涉及。

## Encoder、Layout

Encoder、Layout用来定义日志输出的格式、内容，对应如上所述`<layout>`、`<encoder>`元素，它们都只有1个属性——`class`，用来配置具体的实现类。

<u>老版本推荐使用`<layout>`元素，新版本推荐使用`<encoder>`元素</u>。在新版本中，如果要使用`<layout>`元素，一般是把`<layout>`作为`<encoder>`的子元素，即：

```
<encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
    <layout class="ch.qos.logback.classic.PatternLayout">
    	<pattern>XXX</pattern>
    </layout>
</encoder>
```

`<encoder>`元素`class`属性的默认值是`ch.qos.logback.classic.encoder.PatternLayoutEncoder`，**PatternLayoutEncoder**扩展了**LayoutWrappingEncoder**，内部直接封装了一个PatternLayout实例。因为**PatternLayout** 是最常用的一种Layout，所以<u>PatternLayoutEncoder成为了最常见、最实用的一个Encoder</u>。

上述代码从而可以简写成：

```
<encoder>
	<pattern>XXX</pattern>
</encoder>
```

`<pattern>`元素用来配置PatternLayout的转换规则，详细的内容请参考[官方的使用说明](https://logback.qos.ch/manual/layouts.html#conversionWord)，下面给出一个例子：

```
%date{yyyy-MM-dd HH:mm:ss.SSS} %level ${APPLICATION_NAME:-} ${PID:-} %thread %file %line %logger %class %method : %msg %n %exception{full} %n
```

