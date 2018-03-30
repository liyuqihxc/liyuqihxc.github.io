---
title: 在CentOS7上编译OpenCVSharp
author: 石榴骑士
layout: post
tags: cpp dotnet
feature-image: 2015-04-22-Github_feature.webp
---

### 目录

* [编译库](#编译库)
* [遇到的坑](#遇到的坑)

&emsp;&emsp;公司一个项目的要求有点奇葩，要用OpenCV做运动检测，不能用Python和C++且必须部署在Linux系统上。由于公司没人做Java于是只能寄希望于.net core了。

&emsp;&emsp;GitHub上有一个OpenCV的Wraper项目[OpenCvSharp](https://github.com/shimat/opencvsharp)支持.net core和.net standard，950+ star似乎挺可靠，于是果断添加它的包 OpenCvSharp3-AnyCPU。

&emsp;&emsp;仔细看了下，整个包有3个主要库文件：OpenCvSharp.dll为OpenCV的.net包装，OpenCvSharpExtern.dll和opencv_ffmpeg341.dll为NativeDll，没有提供Linux对应的库。要想在Linux下面使用这个包，关键是要重新编译OpenCvSharpExtern.dll。

### 编译库

* 编译CMake

下载、编译CMake的过程

    curl https://cmake.org/files/v3.11/cmake-3.11.0.tar.gz -o 
    cmake-3.11.0.tar.gz --progress
    tar zxf cmake-3.11.0-Linux-x86_64.tar.gz
    cd cmake-3.11.0
    makdir bin
    cd bin
    ./bootstrap
    make

* 编译OpenCV

下载、编译OpenCV的过程

    git clone https://github.com/opencv/opencv.git
    cd opencv
    git checkout 3.4.1
    mkdir bin
    cd bin
    cmake  -DCMAKE_BUILD_TYPE=Release ..
    make

* 编译OpenCvSharp

下载、编译OpenCvSharp的过程

    git clone https://github.com/shimat/opencvsharp.git
    cd opencvsharp
    git checkout 3.4.1.20180320
    mkdir bin
    cd bin
    cmake  -DCMAKE_BUILD_TYPE=Release ..

### 遇到的坑

&emsp;&emsp;由于缺少一些opencvsharp自带的文件，编译时需要将opencvsharp/opencv_files_*/include/opencv2文件夹拷贝到/usr/local/include文件夹中，不需要覆盖已存在的文件。

&emsp;&emsp;编译完成后生成的libOpenCvSharpExtern.so不能放到类似/usr/local/lib这样的库文件夹而应该跟Nuget包OpenCvSharp3-AnyCPU放在一起：lib\netcoreapp1.0\OpenCvSharp.dll所在的目录。否则无法被.net core找到并加载。
