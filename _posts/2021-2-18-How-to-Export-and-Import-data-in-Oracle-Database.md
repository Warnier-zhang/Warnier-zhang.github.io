---
layout: post
title: How to Export and Import data in Oracle Database
---

如何迁移或者备份Oracle数据库？

## 方法一

直接上**Oracle SQL Developer**，注意，不是**PL/SQL Developer**，这是我的首选方法，“快糙猛”，

![数据库复制菜单][1]

![数据库复制弹窗][2]

操作方法，首先，选择**工具**菜单，在菜单项中选中**数据库复制**；然后，在数据库复制向导弹窗中，编辑合适的**源连接**、**目标连接**；接下来，一路傻瓜地点击**下一步(N)**即可。

当然，**方法一**不是万能的，否则没必要写**方法二**了，有下面的几点限制：

- 源连接、目标连接所在网络必须是互通的；
- 源连接、目标连接对应的Oracle数据库**大版本号**尽可能保持一致，比如都是11.2的；

> **注意**：如果复制出现了异常，请仔细阅读操作日志，对于大多数的常见问题，都能百度到解决方法！！！

## 方法二

类比在电脑上安装App，如果方法一算是**一键安装**，方法二就相当于**自定义安装**，分成先导出、再导入等2个步骤。

1. **从旧数据库中导出数据**

   打开命令提示符（CMD），输入并执行如下命令：

   ```shell
   exp test/123456@localhost:1521/xe file=D:\xe.dump log=D:\xe.log
   ```

   默认是导出**当前用户**的数据。

   * 如果要导出全部用户的数据，就在命令的末尾加上`full=y`参数：

     ```shell
     exp test/123456@localhost:1521/xe file=D:\xe.dump log=D:\xe.log full=y
     ```

     

   * 如果只要导出表结构（DDL），就在命令的末尾加上`rows=n`参数：

     ```shell
     exp test/123456@localhost:1521/xe file=D:\xe.dump log=D:\xe.log rows=n
     ```

   > **注意**：在执行导出数据命令的过程中，如果出现了`EXP-00003: 未找到段 (0,0) 的存储定义`，就要小心了，错误提示中提到的这些表，因为没有数据，是空表，Oracle数据库没有给它们分配存储空间，所以无法导出这些表的结构和数据。如果要解决这个问题，就照下面的步骤处理。
>
   > 1. 查询所有的空表
>
   >    ```sql
   >    SELECT
   >        table_name
   >    FROM
   >        user_tables
   >    WHERE
   >        segment_created = 'NO';
   >    ```
>
   > 2. 手动给空表分配存储空间
>
   >    ```sql
   >    ALTER TABLE [table_name] ALLOCATE EXTENT;
   >    ```
>
   > 然后，重新执行上面的导出数据库命令即可。

2. **把数据导入到新数据库**

   打开命令提示符（CMD），输入并执行如下命令：

   ```shell
   imp test2/123456@localhost:1521/xe2 file=D:\xe.dump log=D:\xe2.log full=y commit=y ignore=y
   ```

   在一切正常的情况下，到这里就应该结束了。当然，这是非常理想的，实际上完全不是那么回事儿。

   > **注意**：在执行导入数据命令的过程中，如果出现了`ORA-00959: 表空间 'XXX' 不存在`，就说明新数据库的表空间与旧数据库冲突，需要**创建新的表空间**或者**重命名当前的表空间**。
   >
   > 如果当前用户所属的表空间不是`SYSTEM`等默认的表空间，最简单的方法就是把它重命名为旧数据库对应的表空间，完整的命令如下：
   >
   > ```sql
   > ALTER TABLESPACE [old name] RENAME TO [new name];
   > ```
   >
   > 否则，就手动创建新的表空间：
   >
   > ```sql
   > CREATE TEMPORARY TABLESPACE ts_test2_temp 
   > TEMPFILE 'D:\app\oradata\xe2\ts_test2_temp.dbf'
   > SIZE 100M 
   > AUTOEXTEND ON 
   > NEXT 32M [MAXSIZE 500M 
   > EXTENT MANAGEMENT LOCAL];
   > 
   > CREATE TABLESPACE ts_test2_data 
   > DATAFILE 'D:\app\oradata\xe2\ts_test2_data.dbf' 
   > SIZE 100M 
   > AUTOEXTEND ON 
   > NEXT 32M [MAXSIZE 500M 
   > EXTENT MANAGEMENT LOCAL];
   > ```
[1]: ../images/2021/2/18/1.png
[2]: ../images/2021/2/18/2.png