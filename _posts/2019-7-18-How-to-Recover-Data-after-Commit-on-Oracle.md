---
layout: post
title: How to Recover Data after Commit on Oracle
---

误删（或者更新）表中的部分数据，并且已经向数据库提交了更改，在没有备份这些数据的情况下，有没有办法恢复到原来的样子？答案是：Yes！Oracle提供了一项叫作**闪回查询**（**Flashback Query**）的特性来帮助我们查看和重建意外删除或更改的受损数据。

闪回查询用于查询过去某个时间点的数据。

## 语法规则

```sql
SELECT
    *
FROM
    test AS OF TIMESTAMP ( SYSDATE - INTERVAL '1' HOUR )

-- 或者
  
SELECT
    *
FROM
    test AS OF TIMESTAMP to_timestamp('2019-07-17 00:00:00','YYYY-MM-DD HH24:MI:SS')
```

## 详细示例

假设当前时间下，表`test`中的数据是：

![表`test`原数据][1]

接下来，请动手模拟如下的CRUD场景来感受一下闪回查询。

1. 更改`id=2`记录的`name`字段的值为`654321`；
2. 删除`id=2`记录；

执行如下SQL语句来查看5分钟之前的表`test`中的数据：

```sql
SELECT
    *
FROM
    test AS OF TIMESTAMP ( SYSDATE - INTERVAL '5' MINUTE )
ORDER BY
    id
```

查询结果如下：

![5分钟之前的表`test`中的数据][2]

根据查询结果，还原`id=2`的记录。删除`id=2`记录后的恢复方法，与之类似，不再冗述。

以上就是闪回查询的入门介绍，更详细的用法请参考[使用手册][4]。

## 参考资料

1. [How to Recover Data (Without a Backup!)][3]
2. [Using Oracle Flashback Query (SELECT AS OF)][4]

[1]: ../images/2019/7/18/1.png
[2]: ../images/2019/7/18/2.png
[3]: https://blogs.oracle.com/sql/how-to-recover-data-without-a-backup
[4]: https://docs.oracle.com/database/121/ADFNS/adfns_flashback.htm#ADFNS01003

