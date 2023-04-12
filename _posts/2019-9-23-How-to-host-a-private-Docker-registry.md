---
layout: post
title: 如何搭建Docker私服
---

Docker registry是一款用来托管和分发Docker镜像的应用程序。它是由Docker官方提供的，并以Apache license 2开源。Docker镜像仓库是CI / CD中最重要的基础设施之一。

## Docker registry安装方法

安装Docker registry的最简单的方法就是部署一个registry容器：

```text
docker container run -d -p 5000:5000 --restart=always -v /self/docker/registry:/var/lib/registry --name Dockerhub registry:latest
```

打开浏览器，在地址栏输入`http://192.168.99.100:5000/v2/`，应该可以看到下面的结果：

![Docker registry安装成功][1]

## Docker registry使用教程

上一个步骤安装好了Docker registry，接下来尝试本地打包、推送Docker镜像到仓库中。

首先，克隆[测试代码][2]到本地，并切换到项目根目录。执行如下命令：

```text
mvn clean package
docker image build -t 192.168.99.100:5000/self/test:1.0.0 .
docker image push 192.168.99.100:5000/self/test:1.0.0
```

Docker镜像的命名规则是：**[Docker registry地址]:[端口]/[Docker镜像名称]:[Docker镜像标签]**，例如：`hello-world`镜像完整的唯一标识是`docker.io/library/hello-world:latest`，其中`docker.io`是Docker仓库地址，`library/hello-world`是Docker镜像名称，`latest`是Docker镜像标签。同理，test镜像的唯一标识是`192.168.99.100:5000/self/test:1.0.0`。

`docker image build`根据`Dockerfile`文件本地编译、打包test镜像。

`docker image push`推送test镜像到仓库中。

>注：执行`docker image push`命令时，可能会遇到如下错误：
>
>```text
>Get https://192.168.99.100:5000/v2/: http: server gave HTTP response to HTTPS client
>```
>
>出现这个问题的原因是Docker registry不支持HTTPS，解决方法是配置`insecure-registries`参数，不同方式安装的Docker程序配置方法有所不同：
>
>### Docker for Linux
>```text
>vi /etc/docker/daemon.json
>{
>  "insecure-registries" : ["192.168.99.100:5000"]
>}
>```
>### Docker Toolbox
>```text
>docker-machine ssh default
>sudo -i
>vi /etc/docker/daemon.json
>{
>  "insecure-registries" : ["192.168.99.100:5000"]
>}
>```
>### Docker Desktop for Mac / Windows
>
>点击**Preferences > Daemon**标签，选中**Experimental features**，在**Insecure registries**区域中添加`192.168.99.100:5000`后点击**Apply && Restart**。
>
>![Docker Desktop for Mac配置insecure-registries参数][3]

## Docker registry查询接口

Docker registry内置了一套HTTP API用来查看、管理仓库中的镜像，比较常用的有下面这些：

1. 列出Docker仓库中的所有镜像：

    ```text
    http://192.168.99.100:5000/v2/_catalog
    ```
    ![列出Docker仓库中的所有镜像][4]

2. 列出某个Docker镜像的所有标签：
    ```text
    http://192.168.99.100:5000/v2/[Docker镜像名称]/tags/list
    例如：http://192.168.99.100:5000/v2/self/test/tags/list
    ```
    ![列出某个Docker镜像的所有标签][5]

[1]: ../images/2019/9/23/1.png
[2]: https://github.com/Warnier-zhang/Test
[3]: ../images/2019/9/23/2.png
[4]: ../images/2019/9/23/3.png
[5]: ../images/2019/9/23/4.png
