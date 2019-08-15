---
title: Ubuntu系统软件安装
categories:
  - [操作系统]
tags: [Ubuntu]
---

### Shadowsocks

https://blog.csdn.net/qq_24406903/article/details/85011090

- 安装 Shadowsocks

```bash
$ pip install --upgrade git+https://github.com/shadowsocks/shadowsocks.git@master

# local address 127.0.0.1:18650
$ sslocal -c ~/shadowsocks.json -d start
```

- 安装genpac、下载gfwlist文件

```bash
$ sudo pip install genpac
$ pip install --upgrade genpac
# 下载gfwlist文件
$ genpac --pac-proxy "SOCKS5 127.0.0.1:18650" --gfwlist-proxy="SOCKS5 127.0.0.1:18650" --gfwlist-url=https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt --output="autoproxy.pac"
```

修改系统网络代理为自动，代理脚本路径类似：

```
file:///home/username/autoproxy.pac
```

### QQ与Wechat

https://www.lulinux.com/archives/1319

https://github.com/wszqkzqk/deepin-wine-ubuntu

https://www.cnblogs.com/dunitian/p/9124806.html


### Git设置

生成新ssh key或是复制已有key到 **～/.ssh** ，确保文件权限为400。

```bash
$ chmod 400 ~/.ssh/id_rsa
```

启动ssh agent并添加ssh key

```bash
$ eval "$(ssh-agent -s)"
> Agent pid 59566

$ ssh-add ~/.ssh/id_rsa
```

检查连接

```bash
$ ssh -T git@github.com
```
### node.js filesystem watcher报错ENOSPEC

```bash
# https://github.com/facebook/jest/issues/3254#issuecomment-297214395
$ echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
```

### Nodejs切换版本

```bash
sudo yarn global add n
sudo n lts # 也可以用版本号10.16.0
node -v

# 如过版本号没有变
export NODE_HOME=/usr/local
export PATH=$NODE_HOME/bin:$PATH
export NODE_PATH=$NODE_HOME/lib/node_modules:$PATH
sudo n
```



### 网易云音乐

https://www.zhihu.com/question/277330447/answer/478510195

### 程序自启动

- 普通权限

按Alt+F2输入gnome-session-properties回车，打开 **启动应用程序首选项** 设置启动程序的命令。

- Root权限

创建文件 **/etc/rc.local** ，加上执行权限(如果需要停用自启动则去掉执行权限)。

```shell
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

sslocal -c /home/user/shadowsocks.json -d start &
exit 0
```

```bash
$ chmod +x /etc/rc.local
```

### OpenJDK 11

https://blog.csdn.net/qwfys200/article/details/85740990
https://dzone.com/articles/installing-openjdk-11-on-ubuntu-1804-for-real
https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-on-ubuntu-18-04
https://www.linode.com/docs/development/java/install-java-on-ubuntu-18-04/

- 解决Java桌面程序字体反锯齿问题

在文件 **/etc/environment** 中加上

```
_JAVA_OPTIONS='-Dawt.useSystemAAFontSettings=on -Dswing.aatext=true'
```

