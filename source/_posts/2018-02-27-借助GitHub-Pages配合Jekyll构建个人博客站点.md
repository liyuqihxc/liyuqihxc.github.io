---
title: 借助GitHub-Pages配合Jekyll构建个人博客站点
date: 2018-02-27
updated: 2019-01-08 09:33:56
tags: [GitHub, Jekyll]
---

### 目录

* [Quick Setup](#quick-setup)
* [Fork 指南](#fork-指南)
* [经验和遇到的坑](#经验和遇到的坑)

------

### Quick Setup

&emsp;&emsp;Jekyll是基于Ruby的工具，首先需要安装Ruby，[Ruby的官方网站](https://www.ruby-lang.org/zh_cn/downloads/)可以下载。安装好后会自动弹出命令提示符窗口询问是否安装msys2，因为后面需要用 gem 安装Jekyll，所以这里选择安装 **msys2** 和 **MinGW dev toolchain**。之后是gem，gem可以在(rubygems.org)[https://rubygems.org/pages/download]下载，安装完后重新打开命令提示符，用 **gem install** 命令安装以下几个gem包：

<table style="text-align: center;">
  <tr>
    <td>Jekyll</td>
    <td>这个不用说了</td>
  </tr>
  <tr>
    <td>kramdown</td>
    <td>markdown转html</td>
  </tr>
  <tr>
    <td>liquid</td>
    <td>liquid Template</td>
  </tr>
    <tr>
    <td>github-pages</td>
    <td>for building Jekyll sites with any GitHub-hosted theme</td>
  </tr>
  <tr>
    <td>tzinfo</td>
    <td>Ruby Timezone Library</td>
  </tr>
  <tr>
    <td>tzinfo-data</td>
    <td>为tzinfo提供windows系统上的支持</td>
  </tr>
</table>

到这里，所有需要安装的东西就都结束了。

### Fork 指南

&emsp;&emsp;博客的模板问题有两种选择：直接fork其他人的现成项目，这个最省事基本不用修改；去[Jekyll Themes](http://jekyllthemes.org/)下载模板修改这个最麻烦，博主之前就是打算下载空的模板来修改，现学现卖liquid与Jekyll。弄到半路想想看其实部署个博客不就是来记录自己平常项目的经验经历嘛，这样造轮子完全背离了自己的初衷，干嘛不直接“拿来”个现成的项目呢？

&emsp;&emsp;在google上乱搜一阵之后发现了个[全是法文的小站](https://remidoolaeghe.github.io)，随便复制几段翻译发现主人是个自由职业的Java程序猿,整个页面的布局还挺符合我的需求，于是直接Fork修改。

<div class="well">
    <a href="https://github.com/remidoolaeghe/remidoolaeghe.github.io">https://github.com/remidoolaeghe/remidoolaeghe.github.io</a>
</div>

&emsp;&emsp;作者把所有的工作都做好了，基本上不需要做多少修改：页面里用国外CDN的js、css换成了国内的，去掉了自己不需要的页面，评论换成国内能访问的IntenseDebate（速度还是很慢，后面看看能不能换成gitalk），修改了文章页面右侧“相关博文”部分的查找逻辑。轻松完成~

### 经验和遇到的坑

#### 文档：

<table>
  <tr>
    <td>Jekyll</td>
    <td><a href="https://jekyllrb.com/docs/home/">https://jekyllrb.com/docs/home/</a></td>
  </tr>
  <tr>
    <td>liquid</td>
    <td><a href="https://shopify.github.io/liquid/">https://shopify.github.io/liquid/</a></td>
  </tr>
  <tr>
    <td>markdown语法</td>
    <td><a href="http://wowubuntu.com/markdown/index.html">http://wowubuntu.com/markdown/index.html</a></td>
  </tr>
</table>

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

实际上GitHub Pages现在已经转为使用Rouge来实现代码高亮了，不过因为Pygments和Rouge两者的样式通用，所以还是可以用Pygments的样式文件。Rouge的使用方法参照命令

    rougify help

本地调试的过程中遇到不少坑，不过都不难解决，只是因为不熟悉liquid模板，修改代码的时候花了一点时间。

* 本地调试

本地调试可以使用命令

    bundle exec jekyll serve

如果输出错误

    Liquid Exception: invalid byte sequence in GBK in *****

可以把控制台的代码页改为utf-8再重试

    chcp 65001

* tzinfo

Ruby的tzinfo库如果在windows系统上使用会报dependency error，bing+google之后找到这么个issue：[tzinfo dependency error](https://github.com/aron-bordin/neo-hpstr-jekyll-theme/issues/40)。

解决方法是安装tzinfo-data

    gem install tzinfo-data

再在站点根目录的Gemfile里添加一行

    gem 'tzinfo-data'

* GitHub Pages

页面生成很快，如果生成出现问题可以等GitHub系统发送邮件到你账户对应的邮箱来查看。博主之前没仔细看GitHub Pages的文档，不知道生成失败会发送邮件，于是做了点无用功在找Bug上。
