---
layout: post
title: GoAccess入门
---

GoAccess是Linux系统环境下的一款开源的、实时的网络日志分析工具，用户可以通过浏览器或者命令行来查看、导出分析结果。

[GoAccess特性][1]

## 安装方法

**GoAccess不支持在Windows系统环境下直接安装，需要借助Cygwin、虚拟机、或者Docker来模拟。** 推荐使用Docker镜像方式安装，具体请访问官方的[安装手册][2]。**需要注意的是从1.3版本开始，GoAccess才正式支持中文语言。因此，请选择1.3及更新版本的GoAccess！**

## 使用用法

### Docker镜像文件

虽然GoAccess提供了官方的[Docker镜像][3]，但是这个镜像文件的默认语言与时区是英文，如果要生成中文版本的HTML分析报告，就得把语言与时区切换成中文。达到目的的最简单做法就是自定义Docker镜像：

```dockerfile
FROM allinurl/goaccess:latest

MAINTAINER Warnier-zhang <warnier.zhang@gmail.com>

# 设置中文时区
RUN apk add tzdata
ENV TIME_ZONE=Asia/Shanghai
RUN /bin/cp /usr/share/zoneinfo/$TIME_ZONE /etc/localtime && echo $TIME_ZONE > /etc/timezone

# 设置中文语言
ENV LANG zh_CN.UTF-8
```

执行`docker image build -t goaccess:1.3 .`命令来编译Docker镜像。

### goaccess.conf 

`goaccess.conf`，是GoAccess的配置文件，存储在`/srv/data`目录中，完整的参数列表请参考[GoAccess配置参数][4]。GoAccess的核心配置参数及其示例如下所示：

```text
time-format %H:%M:%S
date-format %d/%b/%Y
log-format %h %^ %^ [%d:%t %^] "%r" %s %b
port 7890
real-time-html true
ws-url ws://192.168.99.100:7890/
log-file /srv/logs/localhost_access_log.2019-02-26.txt
config-file /srv/data/goaccess.conf
```

其中，各个参数的意义如下：

* time-format 
  
    时间格式，请参考[strftime格式][5]。
    
* date-format 

    日期格式，请参考[strftime格式][5]。

* config-file 
  
    自定义的配置文件路径，如果设置这个参数，就会覆盖全局的默认配置参数。
    
* log-file 
  
    日志文件路径，Tomcat的日志文件默认输出到`$CATALINA_HOME/logs`目录中。
    
* log-format 

    日志文件格式，请参考[GoAccess配置参数][6]中的log-format部分。查看`$CATALINA_HOME/conf/server.xml`文件，可以得出Tomcat的日志输出格式是`%h %l %u %t &quot;%r&quot; %s %b`（具体含义参考[Tomcat日志格式][7]），GoAccess的日志解析格式必须与之一一对应。
    
* real-time-html 

    是否实时输出HTML报告。
    
* ws-url 
  
    WebSocket服务器地址。
    
* port 

    WebSocket服务器端口。

### 启动GoAccess

```text
docker run 
  --restart=always 
  -d 
  -p 7890:7890 
  -v "/docker/goaccess/data:/srv/data"         
  -v "/docker/goaccess/html:/srv/report"       
  -v "/docker/tomcat/logs:/srv/logs"           
  --name=goaccess 
  goaccess:1.3
```

执行上述命令启动GoAccess，其中，`/srv/data`是配置文件目录，`/srv/report`是分析结果目录，`/srv/logs`是日志文件目录，最终生成的HTML报告示例如下：

![GoAccess分析报告][8]

[1]: https://goaccess.io/features
[2]: https://goaccess.io/download#docker
[3]: https://hub.docker.com/r/allinurl/goaccess
[4]: https://raw.githubusercontent.com/allinurl/goaccess/master/config/goaccess.conf
[5]: https://www.php.net/manual/zh/function.strftime.php
[6]: https://goaccess.io/man#options
[7]: https://tomcat.apache.org/tomcat-7.0-doc/api/org/apache/catalina/valves/AccessLogValve.html
[8]: ../images/2019/4/3/1.png