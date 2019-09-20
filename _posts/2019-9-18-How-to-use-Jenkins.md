---
layout: post
title: How to use Jenkins
---

Jenkins是一款基于Java开发、以MIT License开源的持续集成（CI）、持续部署（CD）工具，用于自动化地完成编译、打包、测试、部署等任务，从而把开发者从这些繁琐的工作中解放出来，将更多的时间和精力花在理解、实现业务上。

Jenkins发布于2011年2月2日，前身是Hudson，在由谁主导Hudson的问题上与Oracle发生争执后，从Hudson项目中脱胎而出。

![Jenkins CI/CD流程][1]

本文以一个完整例子的形式，向读者演示Jenkins的安装、使用方法，主要的内容会涉及到如下几个方面：

* 拉取Github代码；
* 编译Maven项目；
* 本地打包、推送Docker镜像到Docker registry；
* 远程部署Docker容器

## Jenkins安装方法

### 新建Jenkins数据卷

执行如下命令新建Jenkins数据卷，或者查看Jenkins数据卷在宿主机中对应的存储位置。

```text
docker volume create jenkins
docker volume inspect jenkins
```

![Jenkins数据卷][2]

### 启动Jenkins容器

```text
docker container run \
-d \
-p 8080:8080 \
-p 50000:50000 \
-v /var/run/docker.sock:/var/run/docker.sock \
-v $(which docker):/usr/bin/docker \
-v jenkins:/var/jenkins_home \
--name Jenkins \
jenkins/jenkins:lts
```

虽然需要Jenkins把编译后的代码打包成Docker镜像并推送到Docker registry，但是本文没有采用[**Docker in Docker**][26]的方式在Jenkins容器中安装一个**子Docker**，而是把**宿主机中的Docker**的套接字、命令行的访问路径映射到Jenkins容器中，这样Jenkins就能将打包、推送Docker镜像的任务委托给**宿主机中的Docker**执行。

>注：`-v jenkins:/var/jenkins_home`是把前面新建的Jenkins数据卷映射到容器中的`/var/jenkins_home`目录（即Jenkins的工作目录）。
>
>尽量避免把宿主机中的某个目录直接映射到`/var/jenkins_home`，否则，当容器中的用户使用这些文件时，可能会出现读写权限不够的错误。

### 配置Jenkins

在浏览器地址栏中输入`http://192.168.99.100:8080/`，进入Jenkins工作台。

>注：本文中，Docker是用Docker Toolbox安装的，以其他方式安装的，请输入`http://localhost:8080/`。

#### 输入默认的密码；

![Jenkins默认密码][3]

执行`docker container logs [Jenkins containerID]`命令查看默认的密码：

![Jenkins默认密码][4]

>注：也可以在`/var/jenkins_home/secrets/initialAdminPassword`文件中找到Jenkins默认密码。

#### 安装社区推荐的插件

![安装社区推荐的插件][5]
![安装社区推荐的插件][6]

#### 修改系统管理员密码

![修改系统管理员密码][7]

#### 安装额外的Locale、Maven、SSH插件

![Locale插件][8]
![Maven插件][9]
![SSH插件][10]

#### 设置Jenkins界面显示语言

Jenkins界面显示语言的中文翻译不全，如果中、英文混在一起看着不习惯，可以点击**Manage Jenkins** > **Configure System**选项，参考下图把显示语言切换为英文：

![设置Jenkins界面显示语言][11]

#### 添加远程主机

点击**Manage Jenkins** > **Configure System**选项，参考如下所示添加一台远程主机：

![添加远程主机][12]

#### 配置JDK、Maven等Jenkins全局工具

首先，请自行下载、拷贝`jdk-8u202-linux-x64.tar.gz`、`apache-maven-3.3.9-bin.tar.gz`、`settings.xml`到`/var/jenkins_home/tools`（对应宿主机中的`/mnt/sda1/var/lib/docker/volumes/jenkins/_data/tools`）目录中并解压。然后，请参考如下所示填写相应的栏目：

![配置Maven settings.xml][13]
![配置JDK 8][14]
![配置Apache Maven 3][15]

## Jenkins使用方法

点击**New Item**来新增一个作业，填写完名称、选中Maven Project等栏目后保存。

![新增Jenkins作业][16]

### 配置Source Code Management

点击**Source Code Management**来配置源码仓库地址，常见的版本控制系统有Git、SVN等，本文的[测试代码][29]使用Git并托管在Github。

![配置Source Code Management][17]

### 配置Build Triggers

点击**Build Triggers**来配置Jenkins CI作业自动触发条件。Jenkins支持类似**Cron**的定时轮询机制，下图中配置每5分钟检查一次Git仓库的代码变化。

![配置Build Triggers][18]

### 配置Build

![配置Build][19]

### 配置Post Steps

**Post Steps**紧跟着**Build**执行。本地打包、推送Docker镜像，以及远程部署Docker容器的动作都是发生在这个阶段。

#### 本地执行Shell命令打包、推送Docker镜像

![本地打包、推送Docker镜像][20]

点击**Add post-build step**，选中**Execute shell**。在新增的窗口中，把下面的代码拷贝、粘贴到**Command**栏目中。

```text
IMAGE_NAME=[Docker registry address]/[repository]:[tag]

docker image rm -f $(docker image ls -q $IMAGE_NAME) | true
docker image build -t $IMAGE_NAME .
docker image push $IMAGE_NAME
```

上述代码的意思是：首先，删除本地旧的、同名的Docker镜像；然后，编译、打包新的Docker镜像；最后，把Docker镜像推送给Docker registry。

>注：
>
>![/var/run/docker.sock文件权限不足][21]
>
>如果出现如上图所示的错误，是因为`jenkins`用户没有权限读写`docker.sock`，请执行如下命令分配权限给`jenkins`。
>
>```text
>docker container exec -it -u root  [containerID] /bin/bash
>usermod -a -G [root所属用户组] jenkins
>```

#### 使用SSH远程执行Shell命令部署Docker容器

![远程部署Docker容器][22]

点击**Add post-build step**，选中**Execute shell script on remote host using ssh**。在新增的窗口中，选择上面添加好的远程主机，并把下面的代码拷贝、粘贴到**Command**栏目中。

```text
IMAGE_NAME=[Docker registry address]/[repository]:[tag]

docker container rm -f $(docker container ls -q -f ancestor=$IMAGE_NAME) | true
docker image rm -f $(docker image ls -q $IMAGE_NAME) | true
docker image pull $IMAGE_NAME
docker container run -d -p 8080:8080 --name test $IMAGE_NAME
```

这段代码的意思是：首先，删除由旧的、同名的Docker镜像以及以它为蓝本创建的容器；然后，从Docker registry中拉取新的Docker镜像；最后，部署新的Docker容器，暴露8080端口以供访问。

最后，打开浏览器访问`http://119.3.227.133:8080/test/md5?text=123456`，应该可以看到如下图所示的结果：

![测试结果][23]

## 参考资料

1. [Jenkins Wiki][24]
2. [Official Jenkins Docker image usage][25]
3. [Using Docker-in-Docker for your CI or testing environment? Think twice.][26]
4. [Use volumes][27]
5. [Use bind mounts][28]

[1]: ../images/2019/9/18/1.png
[2]: ../images/2019/9/18/2.png
[3]: ../images/2019/9/18/3.png
[4]: ../images/2019/9/18/4.png
[5]: ../images/2019/9/18/5.png
[6]: ../images/2019/9/18/6.png
[7]: ../images/2019/9/18/7.png
[8]: ../images/2019/9/18/8.png
[9]: ../images/2019/9/18/9.png
[10]: ../images/2019/9/18/10.png
[11]: ../images/2019/9/18/11.png
[12]: ../images/2019/9/18/12.png
[13]: ../images/2019/9/18/13.png
[14]: ../images/2019/9/18/14.png
[15]: ../images/2019/9/18/15.png
[16]: ../images/2019/9/18/16.png
[17]: ../images/2019/9/18/17.png
[18]: ../images/2019/9/18/18.png
[19]: ../images/2019/9/18/19.png
[20]: ../images/2019/9/18/20.png
[21]: ../images/2019/9/18/21.png
[22]: ../images/2019/9/18/22.png
[23]: ../images/2019/9/18/23.png
[24]: https://en.wikipedia.org/wiki/Jenkins_(software)
[25]: https://github.com/jenkinsci/docker/blob/master/README.md
[26]: https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/
[27]: https://docs.docker.com/storage/volumes/
[28]: https://docs.docker.com/storage/bind-mounts/
[29]: https://github.com/Warnier-zhang/Test
