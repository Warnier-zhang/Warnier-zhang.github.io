---
layout: post
title: How to install Homebrew in the Third-party way
---

Homebrew，是一款macOS系统必备的软件安装工具，是一种最简单、最灵活的安装UNIX命令行程序的方法。（Homebrew is the missing package manager for macOS. Homebrew is the easiest and most flexible way to install the UNIX tools Apple didn’t include with macOS.）

Homebrew的安装方法只需一步，即拷贝`/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`到终端中执行。

在通常的情况下，如果网速够给力，就能成功安装好。但是，如果照着上面的方法安装失败，并且反复尝试了都不行，就不要在一棵树上吊死了，可以试一下本文接下来介绍的第三方安装方法。

## 下载安装脚本

打开终端执行`cd Downloads && curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install >> install`命令。（意思是：切换到**下载**目录，下载Hombrew安装脚本，并重命名为`install`。）

## 替换为国内源

常见的Homebrew的国内镜像有如下这些，这些镜像没有好坏之分，挑个看得顺眼的就行了。推荐第一个由中科大维护的镜像：

* https://mirrors.ustc.edu.cn/brew.git；
* https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git；
* https://git.coding.net/homebrew/homebrew.git；

参考下面的代码编辑`install`文件：

```text
...
# 搜索到这一行，替换为国内源；
# BREW_REPO = "https://github.com/Homebrew/brew".freeze
BREW_REPO = "https://mirrors.ustc.edu.cn/brew.git".freeze
...
```

## 执行安装脚本

执行`cd Downloads && /usr/bin/ruby ./install`命令。当遇到`Downloading and installing Homebrew...`提示时，直接关闭终端中断脚本执行。当执行到这一步时，安装程序将绑定**Core Tap**，按照官方的安装方法操作，往往是卡在这一步导致安装失败！

## 手动绑定Core Tap

Core Tap是UNIX命令行程序仓库。

执行`brew tap homebrew/core https://mirrors.ustc.edu.cn/homebrew-core.git --full`命令来绑定Core Tap。

## 手动绑定Cask Tap
Cask Tap是macOS App仓库。

执行`brew tap homebrew/cask https://mirrors.ustc.edu.cn/homebrew-cask.git --full`命令来绑定Cask Tap。

## 配置环境变量

执行`echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.bash_profile && source ~/.bash_profile`命令来替换预编译的二进制包镜像。

到此，Homebrew的第三方安装方法介绍完毕！

## FAQ

1. 执行`brew install`命令一直卡在`Updating Homebrew...`怎么办？
    
    答案：配置`export HOMEBREW_NO_AUTO_UPDATE=true`环境变量来关闭自动更新程序。


