---
layout: post
title: "Mac 利用 brew 和 brew cask 安装应用程序 "
date: 2015-01-20 16:49
comments: true
categories: 
- 技术
---

![Homebrew](/images/2015/01/HomeBrew.png)

`brew`是通过命令行来安装和管理程序的工具，brew 是下载源代码，解压，然后./configure&&make install，同时会一并下载和安装依赖的库。这对于程序猿们来说简直太幸福了。

`brew cask`是直接下载已经compile过的应用安装包，通过此命令用户省去了下载，解压，拖曳等步骤，直接通过一行命令即可安装应用程序。此命令可以用来安装Mac下常见的GUI程序，比如qq,chrome等等。

<!-- more -->

接下来就简单总结下这两个工具的安装以及使用方法：

#安装
##安装homebrew
``` ruby 
ruby -e "$(curl -fsSL https://raw.github.com/Homebrew/homebrew/go/install)"

``` 

##安装cask
```
brew tap phinze/cask
brew install brew-cask
```

#使用
##brew使用
如果使用brew 安装应用，则通过如下命令：
```
brew install appname
```

如果要查看使用brew 安装了哪些库可以通过如下命令:

```
brew list
```


##brew cask使用
如果使用brew cask安装应用，则通过如下命令：
```
brew cask install appname
```

如果要查看使用brew cask安装了哪些应用可以通过如下命令：
```
brew cask list
```
