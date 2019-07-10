---
layout: post
title: How to mount Remote Linux or Windows Directory on Another Linux
---

在Linux主机之间共享文件有CIFS、NFS等2种常用方法。在Windows主机之间共享文件是由CIFS服务实现的。因此，本文主要介绍由CIFS方式实现的在一台Linux主机中挂载另一台Windows或Linux主机的文件夹。

## 挂载Windows共享文件夹

### 共享Windows文件夹
 
共享的方法是傻瓜式，鼠标点点点即可搞定。例如，共享`D:\test`文件夹：

1. 选中`D:\test`文件夹，单击鼠标右键，选中“**共享**”标签：
    
    ![`D:\test`文件夹属性弹窗][1]
 
2. 点击“**高级共享**”按钮，在“**高级共享**”弹窗中，选中“**共享此文件夹**”复选框，填写“**共享名**”栏目：
    
    ![`D:\test`文件夹高级共享弹窗][2]

### 挂载Windows文件夹

`mount`命令挂载CIFS文件系统，需要安装`cifs-utils`软件包，若已经安装了请忽略此步骤。

打开终端，执行命令：

```text
// 挂载
mount -t cifs -o username="administrator",password="Sipgadmin=24680" //172.16.35.56/test /tmp/test

// 取消挂载
umount //172.16.35.56/test
```
    
当执行`umount //172.16.35.56/test`命令时，如果出现“umount: /tmp/test：目标忙。 (有些情况下通过 lsof(8) 或 fuser(1) 可以找到有关使用该设备的进程的有用信息。)”错误提示，就试试切换到其他目录重新执行该命令。

## 挂载Linux共享文件夹

### 共享Linux文件夹

在Linux系统中，**Samba**是一种免费、开源的SMB/CIFS协议实现方案, 支持在Linux和Windows系统间之间共享文件、打印机。

1. 安装Samba：

    ```text
    yum install samba samba-client samba-common
    ```

2. 配置Samba：
    
    Samba的配置文件是`/etc/samba/smb.conf`，编辑该文件并写入如下内容：
    
    ```text
    [test]
            comment = Share test folder between CentOS machines
            path = /tmp/test
            writeable = no
            browseable = yes
            public = no
            valid users = root
    ```

3. 启动Samba：

    ```text
    // 启动Samba服务器
    systemctl start smb
 
    // 查看Samba服务器状态
    systemctl status smb
 
    // 关闭Samba服务器
    systemctl stop smb
    ```
    若启动Samba服务器成功，则会出现类似如下的截图：
    
    ![Samba服务器状态][3]
    
### 挂载Linux文件夹

```text
// 挂载
mount -t cifs -o username=root,password=123456 //172.16.35.55/test /tmp/test
```

[1]: ../images/2019/5/28/1.png
[2]: ../images/2019/5/28/2.png
[3]: ../images/2019/5/28/3.png
