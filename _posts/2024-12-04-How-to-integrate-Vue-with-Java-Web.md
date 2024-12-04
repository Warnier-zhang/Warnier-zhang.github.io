---
layout: post
title: 如何在Java Web应用中集成Vue
---

在回答文章标题提出的问题之前，先通过下面这个表格，来比较一下传统的**Java Web**和新兴的**Vue**之间有什么不同：

|          | Java Web                                        | Vue                                 |
| -------- | ----------------------------------------------- | ----------------------------------- |
| 编程语言 | Java                                            | JavaScript / TypeScript             |
| 开发框架 | Struts、Spring Boot、Spring、MyBatis、Hibernate | Vue Router、Pinia、Vuetify、Element |
| 构建工具 | JDK、Maven / Gradle                             | Vite / Vue CLI / Webpack            |
| 部署文件 | WAR（JSP、Class、XML）                          | HTML、JS、CSS                       |
| 运行环境 | Tomcat（默认端口：8080）                        | Nginx（默认端口：80）               |

可以看出来，从开发、构建、到部署的各个环节，Java Web和Vue都不一样，就像是两条并行的河流，永不相交，最终流向各自的海洋。

即使抛开开发、构建等环节不谈，想要把上述两个不同类型的应用整合到一起，看似也是一个不可能完成的任务。

考虑到，**Java Web由JSP前端和Java后端组成，Vue是纯前端的**。因此，拿Vue和Java Web比是不公平的，Java Web是一个更大的概念，Vue和JSP是同等水平的对手。问题“**如何在Java Web应用中集成Vue？**”自然而然就演变成“**如何使用Vue替代JSP？**”。

在Java Web中，JSP前端联系Java后端的“桥梁”、“纽带”是**会话**（Session），会话用来判断用户是否登录、是否被授权等等。如果能在Vue前端和Java后端之间建立和JSP前端相同的会话，或者说**共享会话**，就能达到使用Vue替代JSP的目的。

以**本地的Java Web和Vue应用**为例：

> 即，Java Web应用部署在http://localhost:8080，Vue应用部署在http://localhost。

Java Web应用部署在中间件**Tomcat**中，Tomcat会把**会话ID**以**Cookie**的形式发送给浏览器。默认情况下，如果**应用的名称是xxx**，Cookie的内容就是`name="JSESSIONID", value="[会话ID]", domain="localhost", path="/xxx"`等等。当JSP前端发送HTTP请求给Java后端时，请求头会携带这个Cookie，这样，Java后端就能识别JSP前端，做出正确的处理和响应。

Vue应用每次发起HTTP请求时，请求头都会携带<u>**所有**</u>`domain="localhost", path="/"`的Cookie。但是，这些Cookie中并不包含`path="/xxx"`的Cookie。因此，如果Vue前端发送HTTP请求给Java后端，Java后端就无法识别Vue前端。

能够让Vue前端发送的HTTP请求携带`path="/xxx"`的Cookie，或者让Java后端创建`path="/"`的Cookie吗？

当然！！！

后者可通过配置`CATALINA_HOME/conf/context.xml`来实现：

```
<Context path="" sessionCookiePath="/">
	...
</Context>
```

这下，不论Java Web应用的名称是什么，都将创建的`path="/"`的Cookie。在Vue前端发送HTTP请求给Java后端时，`path="/"`的Cookie会被一同发送给Java后端，Java后端就能像对待JSP前端一样对待Vue前端。
