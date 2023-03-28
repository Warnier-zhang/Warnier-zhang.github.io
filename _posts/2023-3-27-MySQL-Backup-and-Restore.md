---
layout: post
title: MySQL备份和恢复
---

## 备份

1. 使用`mysqldump`导出数据：

   ```
   mysqldump -uroot -p123456 -hlocalhost --routines --triggers --events --single-transaction --master-data=2 --hex-blob --default-character-set=utf8mb4 --flush-logs --quick --databases test > test.sql
   ```

   参数说明：

   - `--user`或者`-u`：MySQL用户名；

   - `--password`或者`-p`：MySQL密码；

   - `--host`或者`-h`：MySQL服务器地址；
   - `--port`或者`-P`：MySQL服务器端口；

   - `--all-databases`：导出全部数据库；
   - `--databases [数据库名1] [数据库名2]`：导出部分数据库；
   - `--tables [表名1] [表名2]`：导出部分表；
   - `--routines`或者`-R`：导出存储过程和函数；
   - `--triggers`：导出触发器；
   - `--events`：导出事件；
   - `--comments`：导出注释；

   - `--single-transaction`：保证导出时数据库的一致性状态；

   - `--master-data=2`：将当前服务器的`binlog`**文件名**和**位置**追加到导出文件；

     可以使用`SHOW MASTER STATUS`命令查看；

   - `--hex-blob`：使用十六进制格式的字符串导出`BINARY`、`VARBINARY`、`BLOB`等二进制数据；

   - `--default-character-set=utf8mb4`：设置默认的字符集；

   - `--flush-logs`或者`-F`：在开始导出之前刷新日志；

   - `--quick`：不缓存记录，逐行输出；
   - `--no-data`：只导出表结构；

2. 复制`binlog`日志文件：

   ```
   cp -f /var/lib/mysql/binlog.* /mysql-backups/
   ```
   
3. 复制MySQL配置文件：

   ```
   cp -f /etc/mysql/* /mysql-backups/config/
   ```

## 恢复

1. 在`/etc/mysql/my.cnf`中，开启`skip_networking`，重启MySQL服务，禁止外部用户在恢复期间访问数据库；

2. 根据`binlog`日志文件，生成增量数据导入文件：

   1. 根据`CHANGE MASTER TO MASTER_LOG_FILE='binlog.000011', MASTER_LOG_POS=100;` 确定`binlog`起始文件、位置，并执行如下命令：

      ```
      mysqlbinlog --database=test --start-position=100 binlog.000011 > test-binlog.sql
      ```

   2. 按顺序把剩下的`binlog`日志文件追加到上述导出文件；

      ```
      mysqlbinlog --database=test binlog.000012 >> test-binlog.sql
      mysqlbinlog --database=test binlog.000013 >> test-binlog.sql
      ```
      > 一般情况下，是先有备份，再出现误操作，然后借助备份恢复！！！如果还要恢复备份时间点之后的增量数据，就需要结合最新的`binlog`日志文件（例如：`/var/lib/mysql/binlog.000014`），在`binlog`日志文件中，默认对`INSERT`、`UPDATE`、`DELETE`语句部分进行Base64编码，不方便查找、阅读，可用`mysqlbinlog --database=test --base64-output=decode-rows --verbose binlog.000014 | grep -i 'XXX'` 命令确定还原点。然后执行如下命令把最新数据追加到导入文件：
      >
      > ```
      > mysqlbinlog --database=test --stop-position=200 binlog.000014 >> test-binlog.sql
      > ```
      > 
      > 需要手动删除冗余的数据！！！
   
3. 导入数据文件；

   > 删除首行的`WARNING: --master-data is deprecated and will be removed in a future version. Use --source-data instead.`！
   >
   > 事先重新创建库test！

   ```
   mysql -uroot -p123456 test < test.sql
   ```

4. 导入`binlog`增量数据；

   ```
   mysql -uroot -p123456 test < test-binlog.sql
   ```

5. 关闭`skip_networking`，重启MySQL服务；

