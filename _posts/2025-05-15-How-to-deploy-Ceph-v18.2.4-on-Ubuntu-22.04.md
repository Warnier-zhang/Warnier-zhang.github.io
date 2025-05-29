---
layout: post
title: Ubuntu 22.04部署Ceph v18.2.4集群
---

本篇将逐一介绍Ubuntu 22.04 使用cephadm部署Ceph v18.2.4 集群的详细步骤。

#### 1、Ubuntu系统准备

- 修改主机名

  ```
 hostnamectl set-hostname ceph1
  ```

- 编辑`/etc/hosts`文件，新增如下内容：

  ```
  192.168.1.1 ceph1
  192.168.1.2 ceph2
  192.168.1.3 ceph3
  ```
  
- 设置中文时区

  ```
  timedatectl set-timezone Asia/Shanghai
  ```

- 安装Ubuntu系统缺失的依赖：

  ```
  apt install chrony openssh-client -y
  ```

- 同步系统时间

  按如下方式编辑 `/etc/chrony/chrony.conf`配置文件：

  ```
  ...
  pool ntp1.aliyun.com iburst
  ...
  ```

  重启chromy即可。可通过`chronyc tracking`命令查看本机时间同步状态。

#### 2、安装Docker

提前下载好[最新的deb](https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/dists/jammy/pool/stable/amd64/)包，并执行如下命令：

```
dpkg -i ./containerd.io_1.7.27-1_amd64.deb \
./docker-ce_28.0.4-1~ubuntu.22.04~jammy_amd64.deb \
./docker-ce-cli_28.0.4-1~ubuntu.22.04~jammy_amd64.deb \
./docker-buildx-plugin_0.22.0-1~ubuntu.22.04~jammy_amd64.deb \
./docker-compose-plugin_2.34.0-1~ubuntu.22.04~jammy_amd64.deb \
```

##### 安装docker-compose

提前下载好`docker-compose-linux-x86_64`文件，并执行如下命令：

```
mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

#### 3、安装cephadm

提前下载好[cephadm](https://download.ceph.com/rpm-18.2.4/el9/noarch/cephadm)文件，并执行如下命令：

```
mv cephadm /usr/local/bin/cephadm
chmod +x /usr/local/bin/cephadm

# cephadm add-repo --release reef
# 使用阿里云镜像
wget -q -O- 'https://mirrors.aliyun.com/ceph/keys/release.asc' | sudo apt-key add -
apt-add-repository 'deb https://mirrors.aliyun.com/ceph/debian-18.2.4/ jammy main'
apt update

cephadm install

# 安装Ceph CLI
cephadm install ceph-common
```

#### 4、初始化Ceph集群

Ceph镜像从**<u>v17</u>**开始就是基于CentOS Stream 9构建的。CentOS Stream 9的最低要求包括：`AMD and Intel 64-bit architectures (x86-64-v2)`，如果CPU不支持**<u>x86-64-v2</u>**指令集（主要是**<u>国产</u>**的CPU，如：**<u>海光</u>**），就会出现`Fatal glibc error: CPU does not support x86-64-v2`问题。

解决方法有2种：

- 如果Ceph节点是一台虚机（如：**深信服超融合云主机**），就**<u>启用Host CPU</u>**模式运行；
- 寻找**<u>[基于Ubuntu构建的Ceph镜像](https://github.com/canonical/ceph-containers/pkgs/container/ceph)</u>**作为替代品；

本文采用第2种方法：

```
cephadm --docker bootstrap --mon-ip 192.168.1.1

# cephadm --docker --image ghcr.io/canonical/ceph:18.2.4 bootstrap --mon-ip 192.168.1.1
cephadm --docker --image ghcr.nju.edu.cn/canonical/ceph:18.2.4 bootstrap --mon-ip 192.168.1.1
```

#### 5、配置Ceph Dashboard

在上述步骤初始化完成之后，会出现这样一段提示：

```
Generating a dashboard self-signed certificate...
Creating initial admin user...
Fetching dashboard port number...
Ceph Dashboard is now available at:

             URL: https://ceph1:8443/
            User: admin
        Password: f39x48br94

Enabling client.admin keyring and conf on hosts with "admin" label
Saving cluster configuration to /var/lib/ceph/cac07d38-36a5-11f0-971c-cff4e8befd42/config directory
Enabling autotune for osd_memory_target
You can access the Ceph CLI as following in case of multi-cluster or non-default config:

        sudo /usr/local/bin/cephadm shell --fsid cac07d38-36a5-11f0-971c-cff4e8befd42 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only running a single cluster on this host:

        sudo /usr/local/bin/cephadm shell

Please consider enabling telemetry to help improve Ceph:

        ceph telemetry on

For more information see:

        https://docs.ceph.com/en/latest/mgr/telemetry/

Bootstrap complete.
```

此时，访问https://192.168.250.1:8443即可。

#### 6、添加、删除Ceph节点

```
ceph orch host ls
ssh-copy-id -f -i /etc/ceph/ceph.pub ceph2
ssh-copy-id -f -i /etc/ceph/ceph.pub ceph3

# 添加节点
ceph orch host add ceph2 --labels _admin
ceph orch host add ceph3 --labels _admin

# 删除节点
ceph orch host rm ceph2
```

#### 7、添加裸盘到Ceph集群

如下命令可以一键添加Ceph集群内的所有裸盘：

```
ceph orch apply osd --all-available-devices
```

或者手动添加某个节点上的裸盘：

```
# 添加裸盘
ceph orch daemon add osd ceph2:/dev/vdb

ceph osd tree
```

#### 8、测试用例

以在Ceph集群中读写一个`test.txt`文件为例：

```
# 本地创建test.txt文件
echo 'Hello, World!' > test.txt

# 创建Pool
ceph osd pool create test 16 16

# 设置Pool的存储类型
ceph osd pool application enable test rbd

# 上传test.txt文件到Pool
rados put test.txt ./test.txt  -p test

# 查看Pool下的内容
rados ls -p test

# 从Pool中下载test.txt对象到本地
rados get -p test test.txt ./test2.txt
```

#### 9、卸载Ceph集群

```
cephadm reset
```

#### 10、FAQ

#### 11、常用的Ceph命令

```
# 查看集群状态
ceph -s

# 查看硬盘状态
lsblk

# 部署、删除服务
ceph orch apply ceph-exporter
ceph orch rm ceph-exporter
```

