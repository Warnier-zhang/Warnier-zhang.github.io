---
layout: post
title: Jenkins迁移
---

因为老的服务器要进行升级维护，所以要把在上面运行的Jenkins迁移到一台新的服务器上。本文将介绍Jenkins迁移的过程，以及遇到的各种问题。

#### 老的服务器

老的服务器上的Jenkins是以`war`形式安装的，版本是**2.332.1**。打包步骤如下：

- 执行`systemctl stop jenkins`命令停机；

- 删除冗余、无用的构建历史、源码等文件：

  可以先用`du -h --max-depth=1`命令查看大文件所在目录，再有针对性地删除冗余、无用的构建历史、源码等文件。以Maven项目**Job** user-service为例：

  ```
  rm -rf /var/lib/jenkins/jobs/user-service/builds/*
  rm -rf /var/lib/jenkins/jobs/user-service/modules/[groupId]$[artifactId]/builds/*
  rm -rf /var/lib/jenkins/workspace/*
  ```

- 打包`$JENKINS_HOME`（`/var/lib/jenkins`）下的全部文件：

  ```
  tar -czvf jenkins.tar.gz /var/lib/jenkins/
  ```

#### 新的服务器

为了方便以后再次迁移，决定在新的服务器上以`Docker`形式来安装Jenkins，为了能够**<u>无缝衔接老的Jenkins</u>**，版本**<u>保持和老的Jenkins一致</u>**。

> 注意：
>
> Jenkins `2.332.1-lts-jdk11`镜像使用的GLIBC版本为**2.31**。而，从v20.10.13开始，Docker就要求GLIBC的版本最低为**2.32**、**2.34**，因此，为了能够无缝衔接老的Jenkins，最好使用v20.10.12及以下版本的Docker，否则会出现`docker: /lib/x86_64-linux-gnu/libc.so.6: version 'GLIBC_2.32' not found (required by docker)` 等问题。

首先，新建`/docker-data/jenkins`目录，然后执行 `tar -xvzf jenkins.tar.gz -C /docker-data/jenkins/`命令把上述打包的所有Jenkins文件解压到该目录中。

接下来，执行如下命令启动Jenkins：

```
jenkins:
  image: jenkins/jenkins:2.332.1-lts-jdk11
  container_name: jenkins-ci
  privileged: true
  user: root
  volumes:
    - /docker-data/jenkins:/jenkins_home
    - /var/run/docker.sock:/var/run/docker.sock
    - /usr/bin/docker:/usr/bin/docker
  ports:
    - 8080:8080
    - 50000:50000
```

**本来到此处，就能结束了！**然而，并没有。

上述启动的Jenkins，在编译Maven项目user-service时，一直报错：`/var/lib/jenkins/tools/apache-maven-3.3.9 not found`。即使在**全局工具配置**中，把`/var/lib/jenkins`改成`/jenkins_home`，问题也存在。百思不得其解。

考虑到`/jenkins_home`是由Jenkins镜像中的环境变量`$JENKINS_HOME`设置的，自然而然地就想到另一种处理方式——**<u>把`$JENKINS_HOME`的值改成`/var/lib/jenkins`</u>**。结果，问题迎刃而解。

完整的`docker-compose.yaml`如下：

```
version: '3'
services:
  jenkins:
    image: jenkins/jenkins:2.332.1-lts-jdk11
    container_name: jenkins-ci
    privileged: true
    user: root
    environment:
      JENKINS_HOME: "/var/lib/jenkins"
    volumes:
      - /docker-data/jenkins:/var/lib/jenkins
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    ports:
      - 8080:8080
      - 50000:50000
```
