---
layout: post
title: 如何在Logback配置文件中处理条件判断？
---

Spring Boot应用可以通过配置Logback，借助`ch.qos.logback.more.appenders.DataFluentAppender`，把系统在运行过程中产生的日志输出到`EFK`。`EFK`一切正常的话，万事大吉。如果`EFK`出现故障了，代码就会不停地在控制台输出如下错误日志：

```
o.fluentd.logger.sender.RawSocketSender  : org.fluentd.logger.sender.RawSocketSender java.net.ConnectException: 拒绝连接 (Connection refused)
```

虽然不会影响系统正常运行，但是给调试带来了很大不便。

有没有一种方式能根据**环境变量**、**启动参数**、**`application.yml`**等中定义的变量，来决定是否启用`EFK`相关的`Appender`呢？

Logback官方文档[Conditional processing of configuration files](https://logback.qos.ch/manual/configuration.html#conditional)一节给出了答案，即在Logback配置文件中的根元素`<configuration>`元素下，可以使用`<if>`、`<then>`、`<else>`等子元素来处理条件判断：

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

其中，`condition`中是一个返回`String`对象的表达式。表达式使用`property()`（简写：`p()`）方法读来取上下文变量、系统变量。读取在`application.yml`中定义的变量要复杂一点，首先需要用`<springProperty>`绑定该变量，然后`property()`方法才可以引用该变量。如果上述的变量不存在，`property()`方法就返回空字符串，而不是`null`。

Logback还提供了`isDefined()`和`isNull()`方法。`isDefined()`用来判断是否定义了某个变量。`isNull()`用来判断某个变量的值是否为`null`。

下面给出了完整的使用步骤：

1. 引入相关的依赖：

   ```
   <dependency>
       <groupId>org.codehaus.janino</groupId>
       <artifactId>janino</artifactId>
       <version>3.1.9</version>
   </dependency>
   ```

2. 配置`Logback`：

   ```
   <?xml version="1.0" encoding="UTF-8"?>
   <included>
       <include resource="org/springframework/boot/logging/logback/base.xml"/>
   
       <springProperty scope="context" name="enableEFK" source="test.logging.efk.enable" defaultValue="false"/>
       
       ...
       
       <appender name="FLUENTD" class="ch.qos.logback.more.appenders.DataFluentAppender">
           ...
       </appender>
   
       <root level="INFO">
           <if condition='property("enableEFK").equals("true")'>
               <then>
                   <appender-ref ref="FLUENTD"/>
               </then>
               <else>
                   <appender-ref ref="CONSOLE"/>
                   <appender-ref ref="FILE"/>
               </else>
           </if>
       </root>
   </included>
   ```

3. 配置`application.yml`：

   ```
   ...
   test:
     logging:
       efk:
         enable: [true/false]
   ...
   ```

   

4. 测试；
