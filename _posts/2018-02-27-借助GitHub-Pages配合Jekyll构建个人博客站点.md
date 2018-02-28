---
title: 借助GitHub Pages配合Jekyll构建个人博客站点
author: 石榴骑士
layout: post
icon: fa-lightbulb-o
---

### 目录

* [Quick Setup](#quick-setup)
* [Fork 指南](#fork-指南)
* [经验和遇到的坑](#经验和遇到的坑)

------

### Quick Setup

&emsp;&emsp;Jekyll是基于Ruby的工具，首先需要安装Ruby，[Ruby的官方网站](https://www.ruby-lang.org/zh_cn/downloads/)可以下载。安装好后会自动弹出命令提示符窗口询问是否安装msys2，因为后面需要用 gem 安装Jekyll<!--excerpt-->，所以这里选择安装 **msys2** 和 **MinGW dev toolchain**。之后是gem，gem可以在(rubygems.org)[https://rubygems.org/pages/download]下载，安装完后重新打开命令提示符，用 **gem install** 命令安装以下几个gem包：

|Jekyll|这个不用说了|
|kramdown|markdown转html|
|liquid|liquid Template|
|tzinfo|Ruby Timezone Library|
|tzinfo-data|为tzinfo提供windows系统上的支持|

到这里，所有需要安装的东西就都结束了。

&emsp;&emsp;博客的模板问题有两种选择：直接fork其他人的项目，这个最省事；去[Jekyll Themes](http://jekyllthemes.org/)下载模板修改。直接fork其他人的项目只需要修改仓库名称，如果打算下载模板来修改的话，需要先在[GitHub](https://github.com)里新建一个名称如 **username.github.io** 的仓库再clone到本地，把模板放到本地仓库根目录修改调试好后commit。

### Fork 指南

暂空

### 经验和遇到的坑

#### 文档：

|Jekyll|[https://jekyllrb.com/docs/home/](https://jekyllrb.com/docs/home/)|
|liquid|[https://shopify.github.io/liquid/](https://shopify.github.io/liquid/)|
|markdown语法|[http://wowubuntu.com/markdown/index.html](http://wowubuntu.com/markdown/index.html)|

Jekyll有一个本地的服务器可以用来调试、预览整个博客系统，在命令提示符输入以下命令可以查看jekyll的所有命令选项

    jekyll help

pygments所使用的css文件可以用以下命令来生成

    pygmentize -f html -a .highlight -S vs > pygments.css

* -a .highlight指所有css类都具有.highlight这一祖先
* -S vs 指定所需要的样式

所有的样式可以在Python中使用以下代码查看

```python
from pygments.styles import STYLE_MAP
STYLE_MAP.keys()
```

整个过程中遇到不少坑，因为不熟悉liquid模板的语法，修改的时候花了一点时间。其它的坑，比较深的是下来两个：

* tzinfo

Ruby的tzinfo库如果在windows系统上使用会报dependency error，bing+google之后找到这么个issue：[tzinfo dependency error](https://github.com/aron-bordin/neo-hpstr-jekyll-theme/issues/40)。

解决方法是安装tzinfo-data

    gem install tzinfo-data

再在站点根目录的Gemfile里添加一行

    gem 'tzinfo-data'

* GitHub缓存更新很慢

网站代码commit之后等了好久都无法打开，于是一直怀疑Jekyll本地服务器与GitHub的是不是有什么区别导致我配置写错了。在Jekyll的官方文档当中找了很多遍确定自己没错，然后等了好久终于能打开了...