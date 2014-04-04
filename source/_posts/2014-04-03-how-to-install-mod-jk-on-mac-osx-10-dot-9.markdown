---
layout: post
title: "How to install mod_jk on mac osx Mavericks(1.9)"
date: 2014-04-03 16:53
comments: true
keywords: mod_jk,osx mavericks
categories:
- 技术 
--- 

{% img center /images/2014/04/Mavericks.jpg %}


最近要在mac osx 10.9上集成apache 和 Tomcat，采用mod_jk的方式进行集成，但是在过程中遇到了一些在10.8上面没有遇到的蛋疼的问题，本篇文章就做一下总结。

#第一步 下载mod_jk源代码
首先去[官方网站](http://tomcat.apache.org/connectors-doc/)下载mod_jk的代码,在笔者写本篇总结的时候，最新版本的是1.2.39，请根据对应的apache版本选择相应版本的mod_jk下载。

#第二步 安装mod_jk

笔者将源代码放在~/Download目录下面。接下来我们就来看看如何安装mod_jk.  

```bash
cd ~/Downloads/tomcat-connectors-1.2.39-src/native

./configure CFLAGS='-arch x86_64' APXSLDFLAGS='-arch x86_64' --with-apxs=/usr/sbin/apxs
```

执行上面的命令结果出现了一个错误:

```bash
configure: error: could not detect a 32-bit integer type
```
<!-- more -->
上面错误的描述是无法发现一个32位的整型。

通过查看configure，我们发现configure文件中有如下的语句：  

```bash
  #ifdef HAVE_INTTYPES_H
  # include <inttypes.h>
  #endif
  #ifdef HAVE_STDINT_H
  # include <stdint.h>
  #endif
```

初步判断应该在这两个文件中定义了32位的整形。而在mac osx 中的头文件是放在/usr/include目录下的，我们查看此目录发现并没有此头文件，笔者google一下，发现了如下这篇文章：[MAC OSX下include头文件缺失引起的问题及解决办法, command line tools 安装](http://marchtea.com/?p=104)       

上面的文章提到可以用如下的命令安装命令行工具，此命令会安装上文提到的stdint.h文件。  

```bash
xcode-select --install
```
运行上面命令以后，我们会发现/usr/include 目录中出现了stdint.h文件。再接着运行一下configure，发现没有任何错误。然后运行`make install`，安装完成后会在/usr/libexec/apache2/ 生成mod_jk.so文件。

#第三步 配置apache
修改apache配置文件httpd.conf(在osx 10.9上面，httpd.conf位于/etc/apache2 目录)，增加如下语句：  
`LoadModule jk_module libexec/apache2/mod_jk.so`


Over.Have fun,gays!



