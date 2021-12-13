---
layout: post
title: Seata 简明教程（Part 1）

---

毫无疑问，分布式事务是微服务开发中的一个老大难，最好的解决办法就是绕过它，实在是绕不过去了，就要找一个称手的分布式事务框架。我挑的是Seata。

Seata，是一款由Alibaba背书、开源的分布式事务框架，作为Spring Cloud Alibaba的一部分被整合到了Spring Cloud体系当中，支持AT、TCC、SAGA、XA等多种事务处理模式。

与其说Seata是一个框架，倒不如说它是一个平台！Seata分为Server和Client两部分。Seata Server可以作为一个独立的程序运行，也可以作为一个微服务注册到Eureka、Nacos等注册中心。Seata Client，即依赖`com.alibaba.cloud:spring-cloud-starter-alibaba-seata`，用于业务代码集成Seata。

**Spring Cloud Alibaba的开发进度严重落后于Spring Cloud。**Spring Cloud的最新版本是**2021.0.0**，常用的版本有**2020.0.X**，然而，Spring Cloud Alibaba的最新版本**2.2.7.RELEASE**仅支持到**Spring Cloud Hoxton.SR12**，要知道Hoxton可是2年前（~~2019年~~）发布的。唯一支持Spring Cloud 2020.0.X的版本是**2021.1**，发布之后就没有更新过，bug修复之类的就别指望了。关于Spring Cloud和Seata之间的一一对应关系，请参考由官方提供的[版本说明](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)！快速上手的话，直接使用官方推荐的组合`Spring Cloud 2020.0.2 + Spring Cloud Alibaba 2021.1 + Spring Boot 2.4.2`即可。Seata Server和Client的版本最好保持一致!

> 经过试验，Spring Cloud Alibaba 2021.1可以用于最新的Spring Cloud 2020.0.4和Spring Boot 2.5.7！

下面用一个完整的例子来演示**AT模式**下的Seata用法。

首先，按照官网上的说明来安装Seata Server。

### 安装Seata Server

作为一个Spring Cloud的忠实用户，自然而然地就采用第2种安装方式，即把Seata Server作为一个微服务注册到Eureka，然后，Seata Client通过Eureka连上Seata Server。**找到并修改`registry.conf`**：

```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "eureka"

  ...
  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
  ...
  
}
```

官方文档明确说这种方式可以的，然并卵，这样一同操作猛如虎下来，结果报错：

```
java.lang.NoClassDefFoundError: com/netflix/config/ConfigurationManager
```

出错的原因可能是Spring Cloud Alibaba 2021.1与Eureka（`org.springframework.cloud:spring-cloud-starter-netflix-eureka-client:3.0.4`）不兼容。

没办法只能回头采用第1种方式，即把Seata Server作为一个独立的程序运行，Seata Client通过IP直接连接Seata Server。**无需修改配置文件`file.conf`和`registry.conf`，保持默认值，直接运行`seata-server.bat`即可。**

接下来，就是开发2个微服务，一个是服务提供者`service-a`，另一个是服务消费者`service-b`。数据库使用MySQL，**不分库分表**，ORM框架使用MyBatis。

### 服务提供者service-a

#### 新建回滚表

回滚表的DDL可以从[Seata Github源码仓库](https://github.com/seata/seata/blob/v1.3.0/script/client/at/db/mysql.sql)下载。

```
CREATE TABLE IF NOT EXISTS `undo_log_a`
(
    `branch_id`     BIGINT(20)   NOT NULL COMMENT 'branch transaction id',
    `xid`           VARCHAR(100) NOT NULL COMMENT 'global transaction id',
    `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
    `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
    `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8 COMMENT ='AT transaction mode undo table';
```

#### 添加依赖

```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

#### 修改配置文件`application.yml`

一个完整的配置示例可以在[Seata Github源码仓库](https://github.com/seata/seata/blob/v1.3.0/script/client/spring/application.yml)中找到。

```
spring:
  application:
    name: service-a
  cloud:
    alibaba:
      seata:
        tx-service-group: service-a-group     
seata:
  enabled: true
  service:
    vgroup-mapping:
      service-a-group: default
    grouplist:
      default: 127.0.0.1:8091
    disable-global-transaction: false
  client:
    undo:
      log-table: undo_log_a
```

在上述`application.yml`中省略了Eureka、Spring Cloud Config、日志和Tomcat端口的配置参数！！！

#### 编写业务代码

##### 控制器

```
@RestController
@RequestMapping("/a")
public class AController {
    @Autowired
    private AService aService;

    @GetMapping("/addA")
    public String addA(String aaa) {
        A a = new A();
        a.setAaa(aaa);
        aService.addA(a);
        return "OK";
    }

    @GetMapping("/updateA")
    public String updateA(Integer id, String aaa) {
        A a = new A();
        a.setId(id);
        a.setAaa(aaa);
        aService.updateA(a);
        return "OK";
    }
}
```

##### 服务

```
@Service
public class AService {
    @Autowired
    private ADao aDao;

    @Transactional
    public void addA(A a) {
        aDao.insertA(a);
    }

    @Transactional
    public void updateA(A a) {
        aDao.updateA(a);
    }
}
```

##### 数据库访问对象

```
@Mapper
public interface ADao {
    void insertA(A a);

    void updateA(A a);
}
```

### 服务消费者service-b

**service-b**和**service-a**大同小异。

#### 新建回滚表

```
CREATE TABLE IF NOT EXISTS `undo_log_b`
(
    `branch_id`     BIGINT(20)   NOT NULL COMMENT 'branch transaction id',
    `xid`           VARCHAR(100) NOT NULL COMMENT 'global transaction id',
    `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
    `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
    `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8 COMMENT ='AT transaction mode undo table';
```

#### 添加依赖

```
<!-- OpenFeign -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<!-- Spring Cloud Circuit Breaker-->
<!--
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

<!-- Seata -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

与service-a的依赖相比，除了Seata之外，增加了Circuit Breaker（熔断）。Spring Cloud 2020.0.X移除了Hystrix，本来打算使用Resilience4j替代，但是一番尝试下来，发现Seata和Resilience4j不兼容（**可能是Spring Cloud Alibaba 2021.1的bug**），只能退而求其次，选择用Sentinel（感觉是Alibaba在强推自己家的框架？）。

#### 修改配置文件`application.yml`

```
spring:
  application:
    name: service-b
  cloud:
    alibaba:
      seata:
        tx-service-group: service-b-group
#feign:
#  circuitbreaker:
#    enabled: true
feign:
  sentinel:
    enabled: true
seata:
  enabled: true
  service:
    vgroup-mapping:
      service-b-group: default
    grouplist:
      default: 127.0.0.1:8091
    disable-global-transaction: false
  client:
    undo:
      log-table: undo_log_b
```

#### 编写业务代码

##### 控制器

```
@RestController
@RequestMapping("/b")
public class BController {
    @Autowired
    private BService bService;

    @GetMapping("/addB")
    public String addB(String bbb) {
        B b = new B();
        b.setBbb(bbb);
        bService.addB(b);
        return "OK";
    }

    @GetMapping("/updateB")
    public String updateB(Integer id, String bbb) {
        B b = new B();
        b.setId(id);
        b.setBbb(bbb);
        bService.updateB(b);
        return "OK";
    }

    @GetMapping("/updateAAndB")
    public String updateAAndB(Integer aId, String aaa, Integer bId, String bbb) {
        A a = new A();
        a.setId(aId);
        a.setAaa(aaa);

        B b = new B();
        b.setId(bId);
        b.setBbb(bbb);
        bService.updateAAndB(a, b);
        return "OK";
    }
}
```

##### 服务

```
@Service
public class BService {
    @Autowired
    private BDao bDao;

    @Autowired
    private AClient aClient;

    @Transactional
    public void addB(B b) {
        bDao.insertB(b);
    }

    @Transactional
    public void updateB(B b) {
        bDao.updateB(b);
    }

    @GlobalTransactional
    public void updateAAndB(A a, B b) {
        aClient.updateA(a.getId(), a.getAaa());
        bDao.updateB(b);
    }
}
```

##### service-a客户端

```
@FeignClient(name = "SERVICE-A", path = "a")
public interface AClient {
    @GetMapping("/updateA")
    public String updateA(@RequestParam("id") Integer id, @RequestParam("aaa") String aaa);
}
```

##### 数据库访问对象

```
@Mapper
public interface BDao {
    void insertB(B b);

    void updateB(B b);
}
```
最后，访问测试地址：http://localhost:8080/b/updateAAndB?bbb=abc&bId=1&aaa=123&aId=1，根据自己的实际情况，构建合适的测试用例。