---
layout: post
title: Get Started with GoAccess
---

GoAccess是**Unix-like**系统环境下的一款开源的、实时的网络日志分析工具，用户可以通过浏览器或者命令行来查看、导出分析结果。

## GoAccess特性

GoAccess会解析特定的日志文件，并对如下的方方面面进行统计、分析：

* 失败请求次数；
* 请求处理时延；
* 独立访客人数；
* 请求路径分布；
* 静态文件访问；
* 404（页面未找到）次数；
* 日志文件大小；
* 用户来源地址；
* 操作系统参数；
* 使用的浏览器；

## GoAccess安装

GoAccess的安装方法，请参考官方的[安装手册](https://goaccess.io/download)。

**注意：GoAccess不支持在Windows环境下直接安装，需要通过Cygwin、虚拟机、或者Docker。**

## GoAccess用法

下面，我们通过Docker来安装GoAccess、并监控Tomcat的Access Log文件，感受一下GoAccess。

虽然GoAccess提供了官方的[Docker镜像文件](https://hub.docker.com/r/allinurl/goaccess)，但是这个镜像文件的默认语言与时区是英文，如果需要生成中文版本的HTML分析报告，就要把语言与时区切换成中文，最简单的做法就是基于官方的镜像文件自定义。

### Docker镜像文件
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

运行`docker image build -t goaccess:1.3 .`命令生成自定义镜像文件。（**注意：从发布1.3版本开始，GoAccess正式支持中文。因此，请选择1.3及更新版本的GoAccess！**）

下一步就是配置GoAccess。

### GoAccess配置文件 

首先，分别新建`/docker/goaccess/data`、`/docker/goaccess/html`、`/docker/tomcat/logs`目录，在`/docker/goaccess/data`目录中新建一个文本文件`goaccess.conf`，然后，拷贝并写入[链接的内容](https://raw.githubusercontent.com/allinurl/goaccess/master/config/goaccess.conf)，接下来，修改如下的部分参数。
```text
time-format %H:%M:%S
date-format %d/%b/%Y

# Tomcat Log Format
log-format %h %^ %^ [%d:%t %^] "%r" %s %b

port 7890
real-time-html true
ws-url ws://192.168.99.100:7890/
log-file /srv/logs/localhost_access_log.2019-02-26.txt
config-file /srv/data/goaccess.conf
```

* time-format
* date-format
* config-file
* log-file
* log-format
* real-time-html
* ws-url
* port


### GoAccess运行方法
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

## 参考资料

1. [官方网站](https://goaccess.io/)；
2. [Github官方仓库](https://github.com/allinurl/goaccess)；
3. [Analyze Tomcat logs with GoAccess](http://www.fepede.net/blog/2016/05/analyze-tomcat-logs-goaccess/)；
4. [GoAccess中文界面显示配置](https://blog.51cto.com/linuxg/2335007)；
5. [Docker容器时区设置与中文字符支持](https://segmentfault.com/a/1190000005026503)；
6. [Using Docker containers as localhost on Mac/Windows](https://www.jhipster.tech/tips/020_tip_using_docker_containers_as_localhost_on_mac_and_windows.html)；

