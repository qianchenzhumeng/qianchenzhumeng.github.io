---
layout: post
title:  "使用 cmocka 进行单元测试"
date:   2021-02-23 21:46:00 +0800
categories: [C, Unit Testing]
tags: [Unit Testing Framework]
---

## 1. cmocka 介绍

cmocka 是一款简洁的 C 单元测试框架，支持打桩。它只依赖 C 标准库，可以运行在多种平台上（包括嵌入式环境）。

## 2. 安装 cmocka

从 [cmocka.org](https://cmocka.org) 下载 cmocka 安装包或源码，例如，linux 上下载 [cmocka-1.1.3.tar.xz](https://cmocka.org/files/1.1/cmocka-1.1.3.tar.xz) 并解压：

```bash
wget https://cmocka.org/files/1.1/cmocka-1.1.3.tar.xz
tar -xvJf cmocka-1.1.3.tar.xz
```

按照源码包中的 `INSTALL.md` 内的指导进行编译安装，例如：

```bash
cd cmocka-1.1.3
mkdir build
cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Debug ..
make
sudo make install
```

## 3. 使用 cmocka

[cmocka.org](https://cmocka.org) 上有使用教程 [Unit testing and mocking with cmocka](https://cmocka.org/talks/cmocka_unit_testing_and_mocking.pdf)，里面介绍了如何用 cmocka 进行单元测试以及如何打桩（教程第 14 页里面有一处头文件包含错误，`stdint.h` 被误写为 `sdtint.h` ）。另外，源码包的 `example` 目录下有丰富的使用示例，可以进行参考。

编译时需要链接 cmocka 库，例如：

```bash
cd example
gcc simple_test.c -lcmocka
./a.out
```

## 4. 故障解决

### (1) 编译源码时 cmake 报错

> CMake Error: CMake was unable to find a build program corresponding to "Unix Makefiles".  CMAKE_MAKE_PROGRAM is not set.
>   You probably need to select a different build tool.
> CMake Error: CMAKE_C_COMPILER not set, after EnableLanguage

我的环境是 WSL（Windows Subsystem For Linux，Ubuntu 20.04.1 LTS），安装 `cmocka-1.1.5` 时，cmake会报错，尝试将 `CMAKE_MAKE_PROGRAM` 设置为 make 无济于事，换了 `cmocka-1.1.3` 后可以顺利编译。

### (2) 编译测试代码时找不到 cmocka 相关的符号

> simple_test.c:(.text+0x77): undefined reference to `_cmocka_run_group_tests'

链接时，需要使用选项 `-lcmocka` 链接 cmocka 库。