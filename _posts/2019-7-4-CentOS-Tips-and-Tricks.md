---
layout: post
title: CentOS小贴士
---

CentOS的温馨小提示，长期更新中……

1. 创建符号链接

    ```text
    ln -s /var/www/test test
    ```
    
2. 查看软件包运行状态

    ```text
    // 查看Tomcat运行状态
    ps -ef|grep tomcat
    ```

3. 检查软件包是否安装

    ```text
    // 检查cifs-utils是否安装
    rpm -q cifs-utils
     
    // 列出cifs-utils安装路径
    rpm -ql cifs-utils
     
    // 查看cifs-utils详细信息
    rpm -qi cifs-utils
    ```