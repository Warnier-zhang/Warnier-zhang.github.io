---
layout: post
title: 如何安装Jenkins 2.504.3 LTS
---

前段时间刚刚按照[Jenkins迁移](https://warnier-zhang.github.io/How-to-Migrate-a-Jenkins-instance/)中描述的方法，把**旧版本**的Jenkins（**2.332.1**）从老服务器迁移到一台新服务器上。本以为大功告成、可以高枕无忧了，结果，漏洞扫描发现一堆中高风险，立马被下了死亡通知。不得不考虑升级到最新的**2.504.3 LTS**。下面介绍**快速安装Jenkins 2.504.3 LTS**的详细步骤：

1. 按照如下内容编写`docker-compose.yml`文件：

   ```
   version: '3'
   services:
     jenkins:
       image: jenkins/jenkins:2.504.3-lts-jdk21
       privileged: true
       user: root
       volumes:
         - /docker-data/jenkins:/var/jenkins_home
         - /var/run/docker.sock:/var/run/docker.sock
         - /usr/bin/docker:/usr/bin/docker
       ports:
         - 8080:8080
         - 50000:50000
   ```

   > 注意：本文采用把宿主机上的Docker映射到Jenkins容器中的方法。

2. 执行`docker-compose up -d`命令，之后，访问http://localhost:8080即可。

3. **加速**方法：

   当Jenkins**初始化**的时候，会安装社区推荐的插件。默认情况下，Jenkins从https://updates.jenkins.io/download/plugins处下载插件，该地址在**国内访问受到限制**，龟速，推荐切换到国内的镜像源，例如：[中科大](https://mirrors.ustc.edu.cn/jenkins/)。详细的操作步骤如下：

   - 执行`docker container stop [容器ID]`命令，停止**步骤2**创建的Jenkins容器；

   - 【可选】编辑`hudson.model.UpdateCenter.xml`文件：

     ```
     <?xml version='1.1' encoding='UTF-8'?>
     <sites>
       <site>
         <id>default</id>
         <!-- <url>http://updates.jenkins.io/update-center.json</url> -->
         <url>https://mirrors.huaweicloud.com/jenkins/updates/update-center.json</url>
       </site>
     </sites>
     ```

   - 编辑`updates/default.json`文件，把文件中所有的https://updates.jenkins.io/download/plugins替换为https://mirrors.ustc.edu.cn/jenkins/plugins；

   - 执行`docker container start [容器ID]`命令，重启**步骤2**创建的Jenkins容器；
