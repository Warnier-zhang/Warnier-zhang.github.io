---
layout: post
title: 如何升级Spring Boot 2.5.x至4.0.x
---

先给出升级前后各个核心框架版本对比、以及注意点：

| 组件                        | 旧版本                      | 新版本                  | 注意点                              |
| --------------------------- | --------------------------- | ----------------------- | ----------------------------------- |
| JDK / 发布时间              | 1.8 / 14年3月               | 21 / 23年9月            | 移除了过时的API                     |
| Spring Boot / 发布时间      | 2.5.14 / 21年5月            | 4.0.5 / 25年11月        | Starter拆分、显式引入               |
| Spring Framework            | 5.3.20                      | 7.0.6                   | 无缝升级                            |
| Spring Security             | 5.5.3                       | 7.0.4                   | 无缝升级                            |
| Jackson                     | 2.12.6                      | 3.1.0                   | 包名、类、语法变化大                |
| Java EE / Jakarta EE        | javax.*                     | jakarta.*               | Java EE相关的`javax.*`API可全局替换 |
| MyBatis Spring Boot Starter | 2.3.1                       | 4.0.1                   | 无缝升级                            |
| MyBatis（<u>更新缓慢</u>）  | 3.5.13                      | 3.5.19                  | 无缝升级                            |
| MySQL Connector/J           | mysql-connector-java:8.0.29 | mysql-connector-j:9.6.0 | Maven构件的`artifactId`重命名       |
| Tomcat                      | 9.0.63                      | 11.0.20                 | 无缝升级                            |

下面，再逐一介绍各个核心框架的升级内容：

#### JDK

各个LTS版本的主要特性：

| 版本 | 特性                                                         |
| ---- | ------------------------------------------------------------ |
| 1.8  | Lambda表达式、函数式编程、Stream API、Optional、新日期时间API、接口默认方法 |
| 9    | 模块化                                                       |
| 11   | var局部变量类型推断、HTTP Client                             |
| 17   | Switch表达式、文本块、instanceof模式匹配、Record类、Sealed（密封）类 |
| 21   | 虚拟线程                                                     |

不再100%向下兼容！！！

从**JDK 9**起，已限制使用`com.sun.*`中的API，可用`java.*`和`javax.*`等API替换，例如：`com.sun.image.codec.jpeg.JPEGCodec`可替换为`javax.imageio.ImageIO`。

**JDK 17**移除了SecurityManager、Applet等。

#### Spring Boot

与JDK、Spring Framework、Spring Security等版本对应关系：

| Spring Boot | JDK  | Spring Framework          | Spring Security |
| ----------- | ---- | ------------------------- | --------------- |
| 2.5.x~2.7.x | 8~21 | 5.3                       | 5.5             |
| 3.0.x~3.5.x | 17~  | 6.x、Java EE → Jakarta EE | 6.x             |
| 4.0.x       | 17~  | 7.x                       | 7.x             |

##### 容器

Spring Boot 4默认仅支持**Tomcat**、**Jetty**，不支持**Undertow**！

##### spring-boot-starter-web

在**Spring Boot 4**中，被拆分成`spring-boot-starter-webmvc`、`spring-boot-starter-web-server-tomcat`、`spring-boot-starter-validation`等3个Starter，需要**显式**引入。

##### Spring Framework

各个版本的主要特性：

| 版本 | JDK  | 规范            | 特性                                                         |
| ---- | ---- | --------------- | ------------------------------------------------------------ |
| 5.x  | 8~17 | Java EE 7       | 响应式编程（WebFlux）、Kotlin支持、JUnit 5、WebClient        |
| 6.x  | 17~  | Jakarta EE 9/10 | Java EE → Jakarta EE、AOT、原生镜像（Native Image）          |
| 7.x  | 17~  | Jakarta EE 11   | 虚拟线程、内置Resilience、声明式HTTP客户端（@HttpExchange）、移除Undertow |

##### Spring Security

##### Jackson

和2.x相比，**Jackson 3.1**的包名、类、语法经历了翻天覆地的变化，以**POJO对象与字符串**的互相转换为例：

**2.x版本**如下：

```
...
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.IOException;

public class JSONUtils {
	private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();
	
    public static String stringify(Object object) {
        try {
            return OBJECT_MAPPER.writeValueAsString(object);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        return null;
    }

    public static <T> T parse(String text, Class<T> clazz) {
        try {
            return OBJECT_MAPPER.readValue(text, clazz);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

**3.1版本**如下：

```
...
import tools.jackson.databind.json.JsonMapper;

public class JSONUtils {
    private static final JsonMapper JSON_MAPPER = JsonMapper
            .builder()
            .build();

    public static String stringify(Object object) {
        return JSON_MAPPER.writeValueAsString(object);
    }

    public static <T> T parse(String text, Class<T> clazz) {
        return JSON_MAPPER.readValue(text, clazz);
    }
}

```

#### Java EE / Jakarta EE

**Java SE**仍然保留了部分`javax.*`API，因此，不能无脑地把`javax.*`全部替换为`jakarta.*`！下表给出了Java EE的核心模块与Jakarta EE的对应关系：

| 模块      | javax包               | jakarta包               |
| --------- | --------------------- | ----------------------- |
| 激活      | javax.activation-api  | jakarta.activation-api  |
| Servlet   | javax.servlet-api     | jakarta.servlet-api     |
| JSP       | javax.servlet.jsp-api | jakarta.servlet.jsp-api |
| 验证      | javax.validation-api  | jakarta.validation-api  |
| JPA       | javax.persistence-api | jakarta.persistence-api |
| 事务      | javax.transaction-api | jakarta.transaction-api |
| 注解      | javax.annotation-api  | jakarta.annotation-api  |
| 邮件      | javax.mail            | jakarta.mail-api        |
| WebSocket | javax.websocket-api   | jakarta.websocket-api   |
| JMS       | javax.jms-api         | jakarta.jms-api         |

#### MyBatis

MyBatis **3.5.19**是最后一个支持JDK 8的版本，后续的**3.6.0+（尚未正式发布）**仅支持JDK 11+。

##### MySQL Connector/J

在版本**8.0.31**之后，Maven构件的`artifactId`，从`mysql-connector-java`改成了`mysql-connector-j`。

#### Tomcat

各个版本的主要特性：

| 版本 | JDK最低 | 规范                 | 特性                               |
| ---- | ------- | -------------------- | ---------------------------------- |
| 7    | 6       | Servlet 3.0、JSP 2.2 |                                    |
| 8    | 7       | Servlet 3.1、JSP 2.3 | HTTP/2                             |
| 9    | 8       | Servlet 4.0、JSP 2.3 | TLS 1.3                            |
| 10.0 | 8       | Servlet 5.0、JSP 3.0 | Java EE → Jakarta EE、Jakarta EE 9 |
| 10.1 | 11      | Servlet 6.0、JSP 3.1 | Jakarta EE 10                      |
| 11   | 17      | Servlet 6.1、JSP 4.0 | Jakarta EE 11、HTTP/3、虚拟线程    |
