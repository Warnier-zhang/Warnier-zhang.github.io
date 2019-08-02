---
layout: post
title: How to use MyBatis Generator
---

MyBatis Generator，简称“MBG”，是一款MyBatis代码生成器，主要是用来生成数据库表对应的POJO类、SQL Map XML文件、DAO接口，大大减轻了开发者编写数据库层CRUD（Create，Retrieve，Update，Delete）代码的工作量。

## 使用方法

以下介绍的使用方法基于MBG Maven插件运行方式。

1. 安装Maven插件：

    ```xml
    <plugin>
        <groupId>org.mybatis.generator</groupId>
        <artifactId>mybatis-generator-maven-plugin</artifactId>
        <version>1.3.7</version>
        <configuration>
            <overwrite>true</overwrite>
        </configuration>
        <dependencies>
            <!-- Oracle JDBC Driver -->
            <dependency>
                <groupId>com.oracle</groupId>
                <artifactId>ojdbc6</artifactId>
                <version>11.2.0</version>
            </dependency>
        </dependencies>
    </plugin>
    ```
    其中，
    
    * overwrite 若值为true，则覆盖重名文件；
    * dependencies 声明JDBC驱动程序；
    
2. 编写配置文件**generatorConfig.xml**，主要的配置参数如下：
    
    * jdbcConnection 连接数据库；
    * javaModelGenerator 生成POJO类；
    * sqlMapGenerator 生成SQL Map XML文件；
    * javaClientGenerator （可选）生成DAO接口；
    * table 指定表；
    
    完整的配置例子请参考[此处][1]。 

3. 保存generatorConfig.xml到`src/main/resources`目录中。（或者配置Maven插件的`configurationFile`属性。）

4. 执行`mvn mybatis-generator:generate`命令。

## generatorConfig.xml

下面，我用一个完整的例子来说明怎么配置generatorConfig.xml，实现生成`Oracle Database 11g Release 2`数据库中表`CMS_LINK`对应的样板代码，表`CMS_LINK`的结构如下：

![`CMS_LINK`表结构][2]

首先，在`src/main/resources`目录中，新建一个文本文件`db.properties`，写入下面的内容：

```properties
oracle11gR2.driverClassName=oracle.jdbc.OracleDriver
oracle11gR2.url=#
oracle11gR2.username=#
oracle11gR2.password=#
```

然后，在`src/main/resources`目录中，再新建一个配置文件`generatorConfig.xml`，拷贝下面的代码写入其中：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <properties resource="db.properties"/>

    <context id="oracle11gR2Tables" targetRuntime="MyBatis3">
        <jdbcConnection driverClass="${oracle11gR2.driverClassName}"
                        connectionURL="${oracle11gR2.url}"
                        userId="${oracle11gR2.username}"
                        password="${oracle11gR2.password}">
        </jdbcConnection>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <javaModelGenerator targetPackage="org.warnier.zhang.notes.mybatis3.generator.model"
                            targetProject="./src/main/java">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <sqlMapGenerator targetPackage="org.warnier.zhang.notes.mybatis3.generator.dao"
                         targetProject="./src/main/resources">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <javaClientGenerator type="XMLMAPPER"
                             targetPackage="org.warnier.zhang.notes.mybatis3.generator.dao"
                             targetProject="./src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <table tableName="CMS_LINK"
               domainObjectName="Link">
            <property name="useActualColumnNames" value="false"/>
            <generatedKey column="LINK_ID"
                          sqlStatement="SELECT cms_link_seq_1.NEXTVAL FROM dual"
                          identity="false"/>
        </table>
    </context>
</generatorConfiguration>
```

上面代码的含义如下：

* context

    `context`元素用来设置生成一套样板代码的运行环境。一个`context`元素对应着一种数据库环境，`generatorConfig.xml`中可以有多个`context`元素。
    
    ****属性****
    
    * id： 运行环境ID;
    * targetRuntime：设置生成哪种版本的样板代码，若值为“MyBatis3”，则生成符合MyBatis 3规范的代码。
    
    ****子元素****
    
    * jdbcConnection
    
        连接数据库，本例子引用`db.properties`文件中的数据库连接参数。
    
    * javaTypeResolver：
        
        类型转换器。
        
        ****属性****
        
        * forceBigDecimals：若值为true，则把`DECIMAL`和`NUMERIC`类型转换成`java.math.BigDecimal`，否则转换成`Integer`或`Long`。
            
    * javaModelGenerator
    
        生成POJO类。
        
        ****属性****
        
        * targetPackage：设置包名。
        * targetProject：设置POJO类文件的存储路径。
        
    * sqlMapGenerator：
    
        生成SQL Map XML文件。
        
        ****属性****
        
        * targetPackage：设置包名。
        * targetProject：设置SQL Map XML文件的存储路径。
        
    * javaClientGenerator：
    
        （可选）生成DAO接口。
        
        ****属性****
          
        * type: 若值为“XMLMAPPER”时，则生成适用于MyBatis 3.x的Mapper接口，并与SQL Map XML文件进行绑定。
        * targetPackage：设置包名。
        * targetProject：设置DAO（也就是*Mapper）接口文件的存储路径。
    
    * table
        
        指定表。
        
        ****属性****
        
        * tableName：设置表名；
        * domainObjectName：设置POJO类名；
        * mapperName：设置DAO接口名；
        * useActualColumnNames：若值为true，则生成的样板代码中的属性名与列名相同；若值为false，则如果列名是以“_”分隔的，属性名就是列名的“Camel Case”格式的。
        
        ****子元素****
        
        * generatedKey 生成自增长的主键ID。

## 常用插件

* org.mybatis.generator.plugins.SerializablePlugin

    生成的POJO类实现了`java.io.Serializable`接口。
    
* org.mybatis.generator.plugins.RowBoundsPlugin
    
    在生成的DAO接口中增加一个`selectByExampleWithRowbounds`方法来实现分页查询功能。
    
本文到此结束，感谢大家阅读！
    
## FAQ

* 如果出现“...Exception getting JDBC Driver: oracle.jdbc.OracleDriver...”怎么办？

    答：配置MBG Maven插件的`dependencies`属性来声明JDBC驱动程序。

* 如果出现“...A required class was missing while executing org.mybatis.generator:mybatis-generator-maven-plugin:1.3.7:generate: java/time/temporal/TemporalAccessor...”怎么办？

    答：使用JDK 8。

## 参考资料

1. [MyBatis Generator指南][3]；

[1]: http://www.mybatis.org/generator/configreference/xmlconfig.html
[2]: ../images/2019/8/2/1.png
[3]: http://www.mybatis.org/generator/