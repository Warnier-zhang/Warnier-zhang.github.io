---
layout: post
title: How to Allow Symbolic Links in Tomcat
---

当部署Java Web应用程序时，常常会碰到一些页面、`jar`包等资源被多个项目共享的问题，为了解决这个问题多是把公用的资源文件拷贝一份放到各个项目中。虽然做到了对症下药，但也带来了更新问题，稍微有一点小的改动，就得把所有的都替换。最好是能把公用的资源放到一处，各个项目以链接的方式引用这里。

毫无疑问，做得到的！

就如何创建链接这个问题来说，Windows和Linux的答案是不同的。

## Windows

```text
mklink/? fileupload D:\fileupload
```

其中，`?`的可选值有`D`（符号链接）、`H`（硬链接）、`J`（目录链接）。

## Linux

```text
ln -? /var/www/test test
```

其中，`?`的值可能是`d`（硬链接），或者`s`（符号链接，又称软链接）。

现在符号链接创建好了，就得看Tomcat是否支持？出于安全的考虑，在默认的情况下，Linux版本的Tomcat禁用符号链接。举个例子，如果把一个符号链接文件放到Web应用程序中，用户在浏览器中访问这个文件，浏览器会提示用户出错了：“404，文件未找到！”。

幸运的是，如果确实需要Tomcat支持符号链接，可以修改某些参数来启用这项特性。请按照如下的方式配置`Context`元素：

```text
<Context allowLinking="true">
...
</Context>
```

重启Tomcat即可生效！