---
layout: post
title: How to install Tomcat 7 on CentOS 7
---

Tomcat是一款免费、开源的Web服务器，由Apache软件基金会负责开发、维护，主要是实现了Servlet、JSP等Java EE规范。本文的目的是演示CentOS 7安装Tomcat 7的方法。

## 安装方法

1. 打开Tomcat的[下载页面][1]下载最新的安装包。

2. 拷贝文件到Tomcat安装目录，例如：`/usr/local/tomcat`，然后解压缩到当前目录中。

    ```text
    tar zxvf apache-tomcat-7.0.42.tar.gz
    ```
    
3. 编辑`/etc/profile`文件，配置`CATALINA_HOME`环境变量。
    
    ```text
    CATALINA_HOME=/usr/local/tomcat/apache-tomcat-7.0.42
    export CATALINA_HOME
    ```
    
4. 启动、关闭Tomcat。

    ```text
    // 启动；
    $CATALINA_HOME/bin/startup.sh
    // 关闭；
    $CATALINA_HOME/bin/shutdown.sh
    ```

若出现如下截图，则表明安装成功！

![Tomcat效果图][2]
    
## 自定义JAVA_HOME、JAVA_OPTS等参数

在`$CATALINA_HOME/bin`目录中，新增`setenv.sh`文件，写入如下的内容：

```text
export JAVA_HOME=/usr/local/java/jdk1.7.0_75
export JAVA_OPTS="-server -Xms512M -Xmx512M -XX:PermSize=128M -XX:MaxPermSize=256M"
```

当启动Tomcat时，`catalina.sh`会执行`setenv.sh`脚本，从而，覆盖掉`JAVA_HOME`，`JAVA_OPTS`等参数的默认值。Tomcat运行时的参数如下：

![Tomcat运行时参数][3]

## 参考资料

1. ![Tomcat configuration recommendations][4]
2. ![Tomcat 8 setenv.sh][5]

[1]: https://tomcat.apache.org/download-70.cgi
[2]: ../images/2019/5/31/1.png
[3]: ../images/2019/5/31/2.png
[4]: https://docs.oracle.com/cd/E40518_01/integrator.311/integrator_install/src/cli_ldi_server_config.html
[5]: https://gist.github.com/patmandenver/cadb5f3eb567a439ec01