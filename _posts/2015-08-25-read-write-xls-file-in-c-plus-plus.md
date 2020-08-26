---
layout: post
title:  "C++ 读写 xls 文件"
date:   2015-08-25 00:11:26 +0800
categories: Other
---
## 下载源码

下载：[xlslib-package-2.5.0.zip](http://jaist.dl.sourceforge.net/project/xlslib/xlslib-package-2.5.0.zip)

平台： Ubuntu 14.04 64bits

g++， version 4.6-4.8无法编译xlslib

```
sudo apt-get install g++-4.4
```

## 编译安装

```
export CC=gcc-4.4 CXX=g++-4.4 
```

```
./configure 
```

```
make
```

```
sudo make install
```

## 使用

```
g++ main.cpp -I/usr/local/include/xlslib -lxls
```