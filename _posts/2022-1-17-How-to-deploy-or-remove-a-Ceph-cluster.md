---
layout: post
title: 如何部署、删除Ceph存储集群
---

Ceph **v15.2.0**（Octopus）**及以上版本**推荐使用`cephadm`工具来部署和管理Ceph存储集群。如果要部署老版本的Ceph存储集群，就使用`ceph-deploy`工具。

## 部署Ceph存储集群

1. 安装`cephadm`；

2. 安装Ceph CLI：

   ```
   cephadm install
   cephadm install ceph-common
   ```

3. 在Ceph存储集群的第1台机器上，执行`cephadm bootstrap`命令，创建一个新的Ceph存储集群，并且启动Monitor进程：

   ```
   cephadm bootstrap --mon-ip <本机IP>
   ```
   > 执行`cephadm ls`或者`ceph -s`命令查询Ceph集群是否创建成功。

4. 添加Ceph存储集群节点：

   1. 配置免密登录：

      ```
      ceph cephadm get-pub-key > ~/ceph.pub
      ssh-copy-id -f -i ~/ceph.pub root@node02
      ```

   2. 添加节点：

      ```
      ceph orch host add node02
      ```
      > 执行`ceph orch host ls`查询节点信息。

5. 添加可用的OSD：

   ```
   ceph orch apply osd --all-available-devices
   ```
   > 执行`ceph osd tree`命令查看对象存储设备信息；
## 删除Ceph存储集群

在Ceph存储集群**运行Monitor进程**的节点上：

1. 停掉、并且删除所有的OSD：

   ```
   for i in $(seq <OSD起始ID> <OSD结束ID>);do ceph osd stop $i;done
   for i in $(seq 1 11);do ceph osd rm $i;done
   ```

2. 删除Ceph存储集群：

   ```
   cephadm rm-cluster --fsid <fsid> --force
   ```

3. 删除包含`ceph-`前缀的虚拟卷、物理卷：

   ```
   lsblk
   
   vgs
   vgremove ceph-24390f48-52fd-4b5a-9a81-f0569f499ddf
   
   pvs
   pvremove /dev/vdb
   ```
   

在Ceph存储集群的**其他节点**上**重复2、3两个步骤**；
