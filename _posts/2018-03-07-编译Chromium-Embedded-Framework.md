---
title: VS2017编译Protobuf
author: 石榴骑士
layout: post
tags: cpp cef
feature-image: 2015-04-22-Github_feature.webp
---

https://www.cnblogs.com/caibirdy1985/p/7244961.html

https://chromium.googlesource.com/chromium/src/+/master/docs/windows_build_instructions.md

    git config --global core.autocrlf false
    git config --global core.filemode false
    git config --global branch.autosetuprebase always

* git的代理设置

用于前期下载depot_tools

    git config --global http.proxy http://127.0.0.1:18650
    git config --global https.proxy https://127.0.0.1:18650

* 配置镜像

用于下载Chromium源码

    git config --global url.https://beijing.source.codeaurora.org/mirrors/chromium.googlesource.com.insteadOf https://chromium.googlesource.com
    git config --global --unset url.https://beijing.source.codeaurora.org/mirrors/chromium.googlesource.com.insteadOf

* winhttp的代理设置

    netsh winhttp set proxy 127.0.0.1:18650

* cipd_client项目使用的是golang的net/http库访问http/https，可通过环境变量设置代理

    set HTTP_PROXY=http://127.0.0.1:18650
    set HTTPS_PROXY=https://127.0.0.1:18650

http://blog.csdn.net/qpshen/article/details/78559710

fatal error C1083: 无法打开包括文件: “windows.h”: No such file or directory
安装Windows Kits 10.0.15063.468
https://hk.saowen.com/a/708109e3ddb8513d4be0f914b2d51a00cdc14fc935dea0f271f114ddfcc9c664
