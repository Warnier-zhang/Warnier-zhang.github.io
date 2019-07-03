---
layout: post
title: How to install JDK 8 on CentOS 7
---

Java是一门功能强大、简单易用的面向对象编程语言，被广泛地用于开发Web后端、移动App等应用程序。使用Java的先决条件是安装Java Development Kit（即JDK），JDK包含了虚拟机、编译器、调试器等用来开发Java应用程序的工具。当前，JDK的最新版本是JDK 12，然而，根据[市场调查][1]的结果，JDK 8仍然是最受欢迎、使用最多的。因此，本文介绍2种CentOS 7安装JDK 8的方法。

## 方法一

打开终端，输入并运行命令：

```text
yum install java-1.8.0-openjdk-devel
```

只要一路输入`Y`，并且敲击回车，就能安装成功！

## 方法二

除了`yum`命令这种安装方法之外，还可以下载安装包来手动安装。而且，相比于前者，后者更灵活，能够确保安装最新版本JDK 8，自定义JDK 8的安装目录。

### 下载安装文件

![JDK 8安装包][3]

从官方提供的[下载页面][2]处下载最新版本的JDK 8安装包。

### 解压缩安装包

拷贝安装包到JDK安装目录，例如：`/usr/local/java`，并且解压文件到当前目录。

```text
tar zxvf jdk-8u202-linux-x64.tar.gz
```
### 配置环境变量

编辑`/etc/profile`文件，在末尾加上如下5行代码：

```text
JAVA_HOME=/usr/local/java/jdk1.8.0_202
JRE_HOME=$JAVA_HOME/jre
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME JRE_HOME CLASSPATH PATH
```

## 查看JDK版本号

在安装完成之后，打开终端，运行下面的命令：

```text
java -version
```

若出现如下截图，则表明安装成功！

![JDK 8版本号][4]

[1]: https://www.jetbrains.com/zh-cn/lp/devecosystem-2019/java/
[2]: https://www.oracle.com/technetwork/java/javase/downloads/java-archive-javase8-2177648.html
[3]: ../images/2019/5/30/1.png
[4]: ../images/2019/5/30/2.png