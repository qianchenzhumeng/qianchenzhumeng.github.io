---
layout: post
title:  "SDCC for Contiki"
date:   2016-12-23 09:41:00 +0800
categories: [Embedded, MCU]
tags: [Contiki]
---

## 1. 背景

平台：CC2530F256

系统环境：InstantContiki2.7 Ubuntu 12.04 32bit

## 2. 安装前的准备

```
$ sudo apt-get install libboost-graph-dev srecord
```

下载SDCC源码后，需要对源码进行一些修改：

编辑“device/lib/incl.mk”，找到103行的：

```
MODELS = small medium large
```

改为：

```
MODELS = small large huge
```

编辑“device/lib/Makefile.in”，找到187行的：

```
TARGETS += models small-mcs51-stack-auto
```

改为：

```
TARGETS += models model-mcs51-stack-auto
```

配置：

```
$ ./configure --disable-ds400-port --disable-hc08-port --disable-s08-port --disable-gbz80-port --disable-z80-port --disable-ds390-port  --disable-pic14-port --disable-pic16-port --disable-r2k-port --disable-r3ka-port --disable-stm8-port --disable-tlcs90-port --disable-z180-port --disable-sdcdb --disable-ucsim 
```

## 3. 编译安装

```
$ make
$ sudo make install
```

如果一切顺利，SDCC已经安装好了，默认安装目录为：“/usr/local/share/sdcc”。

把路径加入环境变量：

```
$ export PATH=$PATH:/usr/local/share/sdcc
```

## 4. 查看安装结果

运行以下命令可以查看SDCC的版本：

```
$ sdcc -v
```

输出的信息会像这样：

```
SDCC : mcs51 3.4.1 #9092 (Dec 26 2016) (Linux)
published under GNU General Public License (GPL)
```
## 5. 可能会遇到的问题

从 3.6.0 版本开始，SDCC 将 `putchar` 函数的原型从 `void putchar(char)` 修改为 `int putchar(int)`，但 `contiki/platform/cc2530dk/debug.h` 中的 `putchar` 函数原型仍为前者，因此，编译时会报错。
建议修改一下 `sdcc/include/stdio.h` 内 `putchar` 函数的原型，将其改回 `void putchar(char)`。

```
$ sudo vim /usr/local/share/sdcc/include/stdio.h
```

## 参考
[1] https://github.com/contiki-os/contiki/wiki/8051-Requirements#build-your-toolchain-sd

[2] http://sdcc.sourceforge.net/doc/sdccman.pdf