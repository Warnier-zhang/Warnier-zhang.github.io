---
layout: post
title: 内网安装Dify 1.4
---

离线安装，先要在某台服务器上**实际安装一次（包括插件）**，再**打包**必备的**Docker镜像**和用到的**插件**，然后**上传**到**内网**服务器上，最后执行常规的安装步骤。相当麻烦、冗余！条件允许的话没必要没苦硬吃。

本文设定是在**内网**环境下，而不是在**离线**环境下。是**把本地Windows主机**（IP：192.168.1.1）**作为远程Linux服务器的代理服务器**，允许内网服务器**临时访问外网**。如果不知道如何操作，请参考[上一篇](https://warnier-zhang.github.io/How-to-setup-A-Local-Proxy-Server-for-the-Remote-Server/)。

下面介绍内网安装**最新Dify 1.4.3**的详细步骤：

- 配置Docker代理：

  按如下内容编辑`/etc/systemd/system/docker.service.d/proxy.conf`文件，**该文件如果不存在，请自行创建**：

  ```
  [Service]
  Environment="HTTP_PROXY=http://192.168.1.1:7890"
  Environment="HTTPS_PROXY=http://192.168.1.1:7890"
  Environment="NO_PROXY=localhost,127.0.0.1"
  ```

- 先提前下载好[Dify 1.4.3 安装包](https://github.com/langgenius/dify/releases/tag/1.4.3)，再上传到内网服务器`/dify`目录，然后解压缩。

  - `cd /dify/docker`

  - `cp .env.example .env`，并编辑`.env`文件，在文件末尾追加上

    ```
    REMOTE_INSTALL_URL=http://${EXPOSE_PLUGIN_DEBUGGING_HOST:-localhost}:${EXPOSE_PLUGIN_DEBUGGING_PORT:-5003}
    ```

  - 按如下内容编辑`docker-compose.yaml`文件：

    - `api`、`worker`、`plugin_daemon`新增环境变量`REMOTE_INSTALL_URL`：

      ```
      REMOTE_INSTALL_URL: http://${EXPOSE_PLUGIN_DEBUGGING_HOST:-localhost}:${EXPOSE_PLUGIN_DEBUGGING_PORT:-5003}
      ```

      > 注意：
      >
      > 如果不新增环境变量**REMOTE_INSTALL_URL**，在模型供应商中**集成Ollama、GPU Stack等本地大模型**时，就会出现本地大模型不可用、**显示模型 -> 0个模型**等问题。

    - `plugin_daemon`新增如下**代理**相关的环境变量：

      ```
      http_proxy: http://192.168.1.1:7890
      https_proxy: http://192.168.1.1:7890
      HTTP_PROXY: http://192.168.1.1:7890
      HTTPS_PROXY: http://192.168.1.1:7890
      ```
      > 注意：
      >
      > 在临时访问外网，安装好所有的插件之后，需要去掉上述代理相关的环境变量，重新部署`plugin_daemon`，命令如下：
      >
      > ```
      > docker-compose -p dify down plugin_daemon
      > docker-compose -p dify up -d
      > ```
  - `docker-compose -p dify up -d`

- 初始化Dify；
