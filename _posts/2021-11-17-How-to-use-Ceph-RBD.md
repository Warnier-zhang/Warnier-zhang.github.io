---
layout: post
title: Ceph RBD入门
---

Ceph支持3种存储类型：

1. RBD（RADOS Block Device，块，Image）；
2. RGW（RADOS Gateway，Object Storage，对象）；
3. CephFS（文件系统）；

Ceph RBD的特点如下：

| Filesystem | Ceph RBD                     |
| ---------- | ---------------------------- |
| 硬盘、SSD  | OSD（Object Storage Device） |
|            | 池（Pool）                   |
| 卷         | 块（Image）                  |
| 文件夹     | 归置档（PG）                 |
| 文件       | 数据                         |

池（Pool）是一个抽象、逻辑概念。在创建池（Pool）的时候，要指定PG（Placement Group，归置档）和PGP的数量，PG也是一个逻辑概念，作用相当于文件夹，它的值是2^n。

```shell
创建Pool
ceph osd pool create amos 256 256

查看Pool
ceph osd lspools 或者 ceph osd pool ls

查看Ceph集群存储空间使用情况
ceph df
```

![列出Ceph集群中的Pool][1]

设置Pool的存储类型，可选的值有`rbd`、`cephfs`和`rgw`：

```shell
设置Pool的存储类型
ceph osd pool application enable amos rbd

查看Pool的存储类型
ceph osd pool application get amos
```

![设置、查看Pool的存储类型][2]

RBD，是Ceph集群的一种存储类型——块存储。块在Ceph集群叫`image`，使用`rbd`命令来管理块。

```shell
创建一个名称为amos，大小为200G的块，并绑定到名称为amos的池上
rbd create -p amos --image amos --size 200G
或
rbd create amos/amos --size 200G

列出某个Pool下的所有块
rbd ls -p amos

查看块的详情
rbd info amos -p amos
```

![管理块][3]

有了RBD块怎么用呢？`rbd`提供了一个`map`命令，用来把一个RBD块挂载到Linux系统成为一个Block设备。

```shell
挂载RBD块
rbd device unmap -p amos --image amos 或 rbd map -p amos --image amos 或 rbd map amos/amos

列出RBD块的挂载情况
rbd device list

卸载RBD块
rbd device unmap -p amos --image amos 或 rbd unmap -p amos --image amos 或 rbd unmap amos/amos
```

![挂载RBD块][4]

接下来就是格式化磁盘`/dev/rbd2`，并挂载磁盘到文件系统中：

```shell
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

![格式化、挂载RBD模拟盘][5]

下面的文件读写就和使用本地文件夹没什么区别了。

[1]: ../images/2021/11/17/1.png
[2]: ../images/2021/11/17/2.png
[3]: ../images/2021/11/17/3.png
[4]: ../images/2021/11/17/4.png
[5]: ../images/2021/11/17/5.png



