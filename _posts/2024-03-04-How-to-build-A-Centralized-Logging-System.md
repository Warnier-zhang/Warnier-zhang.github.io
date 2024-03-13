---
layout: post
title: 如何构建一个中心化的日志系统
---

就像农夫山泉广告词说的那样，“我们不生产水，我们只是大自然的搬运工”，日志系统本身不产生日志，只是把各个源系统产生的日志收集起来，建立一套统一的、集中的可视化、检索和分析中心。

**定位**：平台、日志仓库、集中的检索、可视化和分析中心。

## 架构设计

以**Fluentd**为中心。

特点：

- 屏蔽技术细节；
- 无感式设计；
- 类似slf4j、log4j的用法；
- 多语言、多终端、多平台支持；
- 可扩展；
- 易于迁移历史数据；

![架构图](..\images\2024\3\4\架构图.jpg)

### 源系统

- Java
- .NET
- Vue 、React等纯前端；
- ……

### <u>日志收集器</u>

中间件——**Fluentd**，辅以开源的、官方的插件，支持如下的数据收集形式：

- File；
- SQL；
- Logback ；
- HTTP REST API；
- Kafka；
- ……

### 存储

- RDBMS，如：MySQL、Oracle等；
- MongoDB；
- Elasticsearch；
- Kafka；

### 日志可视化和分析

- RDBMS → Grafana；
- MongoDB → MongoDB Compass；
- Elasticsearch → Kibana；
- Kafka → UI for Apache Kafka；
- 定制化；

## 解决方案

> **定制Fluentd Docker镜像**
>
> - `fluentd:v1.14.5-debian-1.0-mysql2mysql`
>
>   安装`mysql2`、`fluent-plugin-sql`等插件，支持读取MySQL、文件等LOG记录。
>
> - `fluentd:v1.14.5-debian-1.0-rdbms2mysql`
>
>   在`fluentd:v1.14.5-debian-1.0-mysql2mysql`的基础上，安装`Oracle Instant Client`、`activerecord-oracle_enhanced-adapter`、`ruby-oci8`等插件，支持读取MySQL、文件、Oracle等LOG记录。

### 方案一

**SQL + Fluentd + MySQL + Grafana**：

> 注意：方案一特别适合**<u>没有操作日志</u>**或者操作日志保存在**<u>数据库表</u>**中的源系统！！！

1. 源系统（可能是Java、.NET等等）将操作日志写入LOG表；

   > 注意：源系统数据库支持MySQL、Oracle、PostgreSQL等关系型数据库。目前，仅针对**<u>MySQL</u>**、**<u>Oracle</u>**进行过测试！

   如果源系统没有操作日志，可以参考如下脚本自行创建LOG表：

   **MySQL**：
   
   ```
   CREATE TABLE `t_log` (
       `log_id` int NOT NULL AUTO_INCREMENT COMMENT 'ID',
       `log_app_id` varchar(255) NOT NULL COMMENT 'APP ID',
       `log_module` varchar(255) NOT NULL COMMENT '功能模块',
       `log_type` varchar(255) NOT NULL COMMENT '操作类型',
       `log_content` varchar(2048) NOT NULL COMMENT '操作内容',
       `log_operator` varchar(255) NOT NULL COMMENT '操作人',
       `log_created` timestamp NOT NULL COMMENT '创建时间',
       `log_updated` timestamp NOT NULL COMMENT '更新时间',
       PRIMARY KEY (`log_id`)
   ) COMMENT='LOG表';
   ```
   
   **Oracle**：
   
   ```
   CREATE TABLE "T_LOG" 
   (	
       "LOG_ID" NUMBER(38,0) NOT NULL ENABLE, 
       "LOG_APP_ID" VARCHAR2(500 BYTE) NOT NULL ENABLE, 
       "LOG_MODULE" VARCHAR2(500 BYTE) NOT NULL ENABLE, 
       "LOG_TYPE" VARCHAR2(500 BYTE) NOT NULL ENABLE, 
       "LOG_CONTENT" VARCHAR2(500 BYTE) NOT NULL ENABLE, 
       "LOG_OPERATOR" VARCHAR2(500 BYTE) NOT NULL ENABLE, 
       "LOG_CREATED" TIMESTAMP NOT NULL ENABLE, 
       "LOG_UPDATED" TIMESTAMP, 
       CONSTRAINT "LOG_PK" PRIMARY KEY ("LOG_ID")
   );
   COMMENT ON COLUMN "T_LOG"."LOG_ID" IS 'ID';
   COMMENT ON COLUMN "T_LOG"."LOG_APP_ID" IS 'APP ID';
   COMMENT ON COLUMN "T_LOG"."LOG_MODULE" IS '功能模块';
   COMMENT ON COLUMN "T_LOG"."LOG_TYPE" IS '操作类型';
   COMMENT ON COLUMN "T_LOG"."LOG_CONTENT" IS '操作内容';
   COMMENT ON COLUMN "T_LOG"."LOG_OPERATOR" IS '操作人';
   COMMENT ON COLUMN "T_LOG"."LOG_CREATED" IS '创建时间';
   COMMENT ON COLUMN "T_LOG"."LOG_UPDATED" IS '更新时间';
   COMMENT ON TABLE "T_LOG"  IS 'LOG表';
   ```
   
1. 日志收集器——Fluentd，轮询读取LOG表记录；

   配置`fluent.conf`：

   ```
   <source>
     @type sql
   
     host [地址]
     port [端口]
     database [数据库名]
     adapter mysql2 / oracle_enhanced
     username [账号]
     password [密码]
   
     tag_prefix [APP ID]
   
     select_interval 60s
     select_limit 500
   
     state_file /app/sql_state_[APP ID]
   
     <table>
       table [LOG表名]
       tag log
       update_column [LOG表主键]
     </table>
   </source>
   ```
   
3. Fluentd把LOG表记录存到MySQL；

   配置`fluent.conf`：

   ```
   <match *.log>
     @type copy
   
     <store>
       @type sql
       host [地址]
       port [端口]
       database [数据库名]
       adapter mysql2
       username [账号]
       password [密码]
       # remove_tag_prefix [APP ID]
   
       flush_interval 1s
   
       <table>
         table t_log
         column_mapping 'log_app_id,log_module,log_type,log_content,log_operator,log_created,log_updated'
       </table>
   
       <table *.log>
         table t_log
         column_mapping 'log_app_id,log_module,log_type,log_content,log_operator,log_created,log_updated'
       </table>
     </store>
   
     <store>
       @type stdout
     </store>
   </match>
   ```
   
4. 可视化工具——Grafana，从MySQL中检索操作日志，并展示给最终用户；

   1. 使用docker-compose安装Grafana：

      ```
      version: '3.8'
      services:
        grafana:
          image: grafana/grafana-oss
          container_name: grafana
          user: '0'
          ports:
            - '3000:3000'
          volumes:
            - '$PWD/data:/var/lib/grafana'
      ```

   2. 操作日志可视化：

      ![日志系统可视化](..\images\2024\3\4\日志系统可视化.png)

### 方案二

**File + Fluentd + MySQL + Grafana**：

> 注意：方案二特别适合已经有操作日志，并且操作日志保存在**<u>文件</u>**中的源系统！！！

和方案一相比，步骤3、4都是一样的，只是步骤1、2收集操作日志的方式略有不同，以日志文件`sipgbp.log`为例：

```
SIPGBP [2024-02-23 10:40:29] INFO myScheduler-4 com.bpw.action.EbsDataSyncTimerTask - "调用埃森哲接口开始， 开始时间：2024-02-23 10:40:29"
SIPGBP [2024-02-23 10:41:29] WARN myScheduler-5 com.bpw.action.MailSenderTask - "发送邮件开始， 开始时间： 2024-02-23 10:40:39"
SIPGBP [2024-02-23 10:42:29] WARN myScheduler-5 com.bpw.action.MailSenderTask - "发送邮件结束， 结束时间： 2024-02-23 10:40:39"
SIPGBP [2024-02-23 10:42:29] WARN myScheduler-5 com.bpw.action.MailSenderTask - "发送邮件结束， 结束时间： 2024-02-23 10:40:39"
...
```

配置`fluent.conf`：

```
<source>
    @type tail
    tag sipgbp.log
    path /app/logs/sipgbp.log
    pos_file /app/sipgbp.log.pos
    read_from_head true
    <parse>
      @type regexp
      expression ^(?<log_app_id>[^ ]*) \[(?<log_created>[^\]]*)\] (?<log_type>[^ ]*) (?<log_module>[^ ]*) (?<log_operator>[^ ]*) [^ ]* "(?<log_content>[^\"]*)"$
    </parse>
</source>
```

### 方案三

**SQL + EFK**：

步骤1、2和方案一相同，LOG表记录的收集、存储、检索及可视化交由EFK负责，不再冗述。

### 方案四

**HTTP REST API + Fluentd + 同上**：

针对Vue、React一类无后台、纯前端的应用，可以通过调用HTTP REST API，将操作日志发送给Fluentd，后续的存储、检索及可视化沿用方案一或方案三。

## 常见问题

