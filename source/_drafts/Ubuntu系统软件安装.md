---
title: Ubuntu系统软件安装
categories:
  - [操作系统]
tags: [Ubuntu]
---

### Shadowsocks

https://blog.csdn.net/qq_24406903/article/details/85011090

* 安装 Shadowsocks

```bash
pip install --upgrade git+https://github.com/shadowsocks/shadowsocks.git@master

# local address 127.0.0.1:18650
sslocal -c ~/shadowsocks.json -d start
```

* 安装genpac、下载gfwlist文件

```bash
sudo pip install genpac
pip install --upgrade genpac
# 下载gfwlist文件
genpac --pac-proxy "SOCKS5 127.0.0.1:18650" --gfwlist-proxy="SOCKS5 127.0.0.1:18650" --gfwlist-url=https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt --output="autoproxy.pac"
```

修改系统网络代理为自动，代理脚本路径类似：

```
file:///home/username/autoproxy.pac
```

### QQ与Wechat

https://www.lulinux.com/archives/1319

https://github.com/wszqkzqk/deepin-wine-ubuntu

https://www.cnblogs.com/dunitian/p/9124806.html
