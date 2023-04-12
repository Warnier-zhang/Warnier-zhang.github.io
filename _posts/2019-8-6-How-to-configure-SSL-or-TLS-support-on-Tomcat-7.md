---
layout: post
title: Tomcat 7配置HTTPS
---

HTTPS，是一种安全版本的HTTP协议。HTTPS的安全基础是建立在SSL/TLS协议之上，因此，HTTPS又被称作“HTTP over SSL”。为了保护网站免受黑客骚扰，升级HTTPS得到了大家的共识。本文将介绍Tomcat 7启用SSL/TLS支持的方法。

## 申请SSL证书

升级到HTTPS的第一步就是申请、购买SSL证书。SSL证书由权威的CA机构授权颁发。可以到[阿里云证书服务](https://www.aliyun.com/product/cas)申请购买。

当然，如果只是为了测试HTTPS，就没必要花钱买正式证书，用JDK自带的`keytool.exe`签发个人证书就可以了。

## 启用Tomcat 7 SSL支持

编辑`CATALINA_HOME/conf/server.xml`，搜索到`<Connector>`元素，参考如下代码进行配置：

```xml
<!-- HTTP -->
<Connector port="80" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="443" />

<!-- HTTPS -->
<Connector port="443" protocol="org.apache.coyote.http11.Http11Protocol"
               maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" keystoreFile="*.pfx" keystoreType="PKCS12" keystorePass="#"/>

<Connector port="8009" protocol="AJP/1.3" redirectPort="443" />
```

其中，

* keystoreFile：证书文件路径；
* keystoreType：证书格式，可选值有`JKS`、`PKCS11`、`PKCS12`，`JKS`，即“Java KeyStore”，由`keytool.exe`签发的证书格式；
* keystorePass：证书私钥密码；

## 默认将HTTP流量重定向到HTTPS

编辑`CATALINA_HOME/conf/web.xml`，在文件末尾（`<welcome-file-list>`元素之后）加上如下内容：

```xml
<login-config>
    <auth-method>CLIENT-CERT</auth-method>
    <realm-name>Client Cert Users-only Area</realm-name>
</login-config>
<security-constraint>
    <web-resource-collection>
        <web-resource-name >SSL</web-resource-name>
        <url-pattern>/*</url-pattern>
    </web-resource-collection>
    <user-data-constraint>
        <transport-guarantee>CONFIDENTIAL</transport-guarantee>
    </user-data-constraint>
</security-constraint>
```

## 参考资料

1. [Apache Tomcat 7 SSL/TLS Configuration HOW-TO][1]
2. [HTTPS 升级指南][2]

[1]: https://tomcat.apache.org/tomcat-7.0-doc/ssl-howto.html
[2]: http://www.ruanyifeng.com/blog/2016/08/migrate-from-http-to-https.html
