---
layout: post
title: Ceph 入门
---

Ceph是一种分布式存储系统，有Redhat背书，特点是高性能、高可用（无单点故障）、高扩展（支持动态扩容），提供了三大功能：

1. RBD：块存储；
2. RGW：对象存储；
3. CephFS：文件存储、文件共享；

如果把Ceph和传统的文件存储系统比较，Ceph的组件就大概扮演着如下的角色：

| Filesystem | Ceph           |
| ---------- | -------------- |
| 硬盘、SSD  | OSD            |
| 分区       | Pool（存储池） |
| 卷         | Image（块）    |
| 文件夹     | PG（归置组）   |
| 文件       | 数据           |

- **OSD**：Object Storage Device，存储数据；
- **MON**：Monitor，监控Ceph集群的状态；
- **MDS**：MetaData Server，存储CephFS元数据；
- **Pool**：存储池，是一个抽象、逻辑概念。所有的对象都必须存储在Pool中。
- **PG**：Placement Group，归置组，是一个抽象、逻辑概念，作用类似于文件夹。在创建Pool的时候，需要定义PG和PGP的数量，它们的值是相同的，都是2^n。
- **Client**：客户端，也就是用户，命名规则是`client.xxx`，在新增用户的时候，Ceph集群会自动生成该用户的密码，密码是一串包含40个字符的字符串。

## CephFS

CephFS是一种文件系统，非常类似NFS，能通过网络在多台机器之间共享存储。**一个Ceph集群仅能创建一个CephFS**。下面用一个实例来演示CephFS的用法：

1. 创建存储池（Pool）：

   ```
   ceph osd pool create fs_amos_metadata 64 64
   ceph osd pool create fs_amos_data 64 64
   ```

2. 创建文件系统（Filesystem）：

   ```
   ceph fs new fs-amos fs_amos_metadata fs_amos_data
   ```

3. 创建元数据服务器（MDS）：

   ```
   ceph orch apply mds fs-amos --placement="2 kube-master01 kube-node01"
   ```

4. 创建用户：

   ```
   ceph auth get-or-create client.fs-amos mon 'allow r' mds 'allow rw' osd 'allow rwx pool=fs_amos_data' -o ceph.client.fs-amos.keyring
   ```

5. 挂载到本地文件夹：

   ```
   mount -t ceph 172.16.35.151:6789:/ /mnt/ceph-fs-amos/ -o name=fs-amos,secret=AQBEgJxhPN6DFBAAL/TMvLMFFIRdJ6ygT6ySzA==
   ```

## RBD

RBD是一种块设备，就像是裸盘，如果要使用RBD块，就得先把它格式化，再挂载到本地文件系统。**需要注意的是不要把同一个RBD块挂载到多台不同的机器上**。RBD块的使用方法如下：

1. 创建存储池（Pool）：

   ```
   创建Pool
   ceph osd pool create amos 256 256
   
   查看Pool
   ceph osd lspools 或者 ceph osd pool ls
   
   查看Ceph集群存储空间使用情况
   ceph df
   ```

   ![列出Ceph集群中的Pool][1]

2. 设置存储池的应用对象（Application）:

   存储池应用对象的可选值有`rbd`、`cephfs`和`rgw`。

   ```
   设置Pool的存储类型
   ceph osd pool application enable amos rbd
   
   查看Pool的存储类型
   ceph osd pool application get amos
   ```

   ![设置、查看Pool的存储类型][2]

3. 创建块（Image）：

   ```
   创建一个名称为amos，大小为200G的块，并绑定到名称为amos的池上
   rbd create -p amos --image amos --size 200G
   或
   rbd create amos/amos --size 200G
   
   列出某个Pool下的所有块
   rbd ls -p amos
   
   查看块的详情
   rbd info amos -p amos
   
   删除块
   rbd rm amos -p amos
   ```

   ![管理块][3]

4. 映射成本地块设备（Block Device）：

   `rbd`提供了一个`map`命令，用来把RBD块映射成一个本地的Block设备。

   ```
   挂载RBD块
   rbd device unmap -p amos --image amos 或 rbd map -p amos --image amos 或 rbd map amos/amos
   
   列出RBD块的挂载情况
   rbd device list
   
   卸载RBD块
   rbd device unmap -p amos --image amos 或 rbd unmap -p amos --image amos 或 rbd unmap amos/amos
   ```

   ![映射RBD块][4]

5. 挂载到本地文件夹：

   最后就是格式化磁盘`/dev/rbd2`，并把它挂载到本地文件系统中。

   ```
   格式化RBD模拟盘
   mkfs.ext4 /dev/rbd2 或 mkfs.xfs /dev/rbd2
   
   查看格式化结果
   blkid /dev/rbd2
   
   挂载RBD模拟盘到文件系统中
   mkdir /mnt/ceph-amos
   mount /dev/rbd2 /mnt/ceph-amos/
   
   从文件系统中卸载RBD模拟盘
   umount /dev/rbd2
   ```

   ![格式化、挂载RBD块][5]

之后，读写文件就和在本地文件夹中操作没什么两样了。

[1]: ../images/2021/11/17/1.png
[2]: ../images/2021/11/17/2.png
[3]: ../images/2021/11/17/3.png
[4]: ../images/2021/11/17/4.png
[5]: ../images/2021/11/17/5.png



