---
layout: post
title: Jenkins流水线入门
---

#### 基础语法

Jenkins流水线分成**声明式**和**脚本式**两种语法，本文主要介绍的是**前者**。Jenkins流水线的配置文件是`Jenkinsfile`，可以通过UI创建，也可以从源码管理系统中检出。声明式的流水线由`pipeline`、`agent`、`stage`、`steps`等部分组成。

- `agent`：代理，为代理分配一个执行器，例如：在**本机**或者**Docker容器**中执行，可以在`pipeline`或者`stage`中声明；

- `tools`：工具，支持自定义`maven`、`jdk`、`gradle`等，可以在`pipeline`或者`stage`中声明；

  > 需要先在**<u>系统管理</u>** > **<u>全局工具配置</u>**功能中配置Maven、JDK、Gradle！

- `parameters`：定义用户参数；

- `triggers`：触发器，支持`cron`等；

- `stages`：

  - `stage`：阶段，例如：编译、测试、部署等等。`agent`也可以在`stage`中单独定义，作用范围是该`stage`；
    - `environment`：设置环境变量；
    - `env`：访问环境变量；
    - `params`：访问用户参数；
    - `steps`：步骤，每个`stage`执行的步骤；
      - `script`：Groovy脚本块；
      - `sh ''`：Shell脚本行；
      - `sh ''' '''` ：Shell脚本块；

- `post`：在执行完`stage`声明的`steps`之后，根据`stage`的完成情况（例如：`always`、`success`、`failure`等等），执行相应的步骤；

#### Jenkins流水线示例

##### 编译Maven项目

1. 配置`JAVA_HOME`、`M2_HOME`：

   ```
   tools { 
       maven 'Apache Maven 3.3.9' 
       jdk 'JDK 8' 
   }
   ```

2. 编译Maven项目：

   ```
   stages {
   	...
   	stage('Build') {
           steps {
           	sh "mvn clean install -s /var/jenkins_home/tools/settings.xml -Dmaven.test.skip=true"
           }
       }
   	...
   }
   ```

##### 打包Docker镜像

把上述Maven项目编译结果打包成Docker镜像，并推送到Docker私服（`192.168.1.1:5000`）。

为了能够从`pom.xml`文件中读取`groupId`、`artifactId`、`version`等参数，需要用到[Pipeline Utility Steps](https://plugins.jenkins.io/pipeline-utility-steps/)插件中的`readMavenPom`方法。

```
stages {
	...
	stage('Package') {
        steps {
            script {
                def pom = readMavenPom file: 'pom.xml'
                env.POM_ARTIFACTID = pom.artifactId;
                env.POM_VERSION = pom.version;
                env.DOCKER_IMAGE_NAME = "192.168.1.1:5000/test/${pom.artifactId}:${pom.version}"
            }

            sh '''
            docker image build -t ${DOCKER_IMAGE_NAME} .
            docker login 192.168.1.1:5000 -u admin -p123456
            docker image push ${DOCKER_IMAGE_NAME}
            '''
        }
    }
	...
}
```
> 注意：
>
> 由于使用**Groovy沙盒**，可能会出现`Scripts not permitted to use method org.apache.maven.model.Model getVersion. Administrators can decide whether to approve or reject this signature.`错误。
>
> 此时，需要在**Dashboard** > **系统管理** > **ScriptApproval**功能中配置如下内容：
>
> ```
> method org.apache.maven.model.Model getArtifactId
> method org.apache.maven.model.Model getGroupId
> method org.apache.maven.model.Model getVersion
> ```
