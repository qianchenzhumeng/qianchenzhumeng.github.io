---
layout: post
title:  "Rust USB 开发"
date:   2021-11-03 21:13:00 +0800
categories: [Rust, Other]
tags: [Rust]
excerpt: 写这篇文章的目的是介绍一下如何使用Rust和libusb在windows上进行USB软件开发。
---

## 1. 背景

最近在参与 USB 相关的项目，需要开发一个测试工具，用于最终产品的量产配置、测试，预计运行在 Windows 上。我从微软的网站上找到了为 USB 设备开发桌面应用的两篇文章：《[为 USB 设备开发 Windows 应用程序的概述](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/usbcon/developing-windows-applications-that-communicate-with-a-usb-device)》《[USB 设备的 Windows 桌面应用](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/usbcon/windows-desktop-app-for-a-usb-device)》，文章提到了两种实现方式，一是使用 Visual Studio 创建包含 WindUSB 的模板，基于这个模板进行开发，二是使用 WinUSB 函数访问 USB 设备，后者给的教程也是基于 WindUSB 模板创建主干应用......使用 Visual Studio 有点麻烦，想试试不用 Visual Studio 的方法。

在 [crates.io](https://crates.io/) 上检索 USB 时，找到了 [libusb-rs](https://crates.io/crates/libusb)，这个库实现了对 [libusb](https://libusb.info/) 的 Rust 封装，而 [libusb](https://libusb.info/) 是 C 语言实现的跨平台 USB 设备通用访问库。

## 2. 搭建开发环境

### (1) Rust 开发环境

手头有搭建好的 Windows 上的 Rust 开发环境，工具链为 stable-x86_64-pc-windows-gnu，后端用的是 [MinGW](https://sourceforge.net/projects/mingw/)。

### (2) 安装 pkg-config

按照 [libusb-rs](https://crates.io/crates/libusb) 的要求，需要安装 pkg-config，可以按照 Stack Overflow 上的教程进行安装：[How to install pkg config in windows?](https://stackoverflow.com/questions/1710922/how-to-install-pkg-config-in-windows)

1. go to http://ftp.gnome.org/pub/gnome/binaries/win32/dependencies/
2. download the file [pkg-config_0.26-1_win32.zip](http://ftp.gnome.org/pub/gnome/binaries/win32/dependencies/pkg-config_0.26-1_win32.zip)
3. extract the file bin/pkg-config.exe to C:\MinGW\bin
4. download the file [gettext-runtime_0.18.1.1-2_win32.zip](http://ftp.gnome.org/pub/gnome/binaries/win32/dependencies/gettext-runtime_0.18.1.1-2_win32.zip)
5. extract the file bin/intl.dll to C:\MinGW\bin
6. go to http://ftp.gnome.org/pub/gnome/binaries/win32/glib/2.28
7. download the file [glib_2.28.8-1_win32.zip](http://ftp.acc.umu.se/pub/gnome/binaries/win32/glib/2.28/glib_2.28.8-1_win32.zip)
8. extract the file bin/libglib-2.0-0.dll to C:\MinGW\bin

### (3) 安装 libusb

下载 libusb 的源码，根据 README.txt 中的步骤将源码复制到对应的位置。需要特别注意一下链接库的选择，如果后续程序要静态链接 libusb，需要使用 `MinGW64\static\` 下的 `libusb-1.0.a`，如果选择动态链接，需要使用 `MinGW64\dll\` 下的 `libusb-1.0.dll.a`，并且后续运行编译好的二进制文件时，需要把运行时库 `libusb-1.0.dll` 放到可执行文件的同级目录下。后面会提到，选择哪种库可通过 pkg-config 的配置文件指定。

可按如下完整流程进行安装：

- 将 libusb 源码中 `include` 目录下的 `libusb-1.0` 目录复制到工具链的默认头文件目录下，例如`C:\MinGW\include`
- 在工具链的库目录下创建 `libusb-1.0` 目录
- 将 libusb 源码中 `MinGW64` 目录下的 `static` 目录和 `dll` 目录复制到上一步创建的 `libusb-1.0` 目录中

#### a. 动态链接 libusb

在工具链的库目录新建目录 `pkgconfig`，例如 `C:\MinGW\lib\pkgconfig` ，目录中添加文件 `libusb-1.0.pc`：

```
prefix=C:\MinGW
exec_prefix=${prefix}
includedir=${prefix}/include/libusb-1.0
libdir=${exec_prefix}/lib/libusb-1.0/dll	# 动态链接库所在目录

Name: libusb
Description: libusb
Version: 1.0
Cflags: -I${includedir}
Libs: -L${libdir} -llibusb-1.0	# 使用动态链接库
```

设置环境变量 `PKG_CONFIG_PATH`，指向刚才创建的 `pkgconfig` 目录。

```powershell
$env:PKG_CONFIG_PATH="C:\MinGW\lib\pkgconfig"
```

可以用如下命令验证下配置结果：

```powershell
pkg-config --libs --cflags libusb-1.0
```

没有问题的话会输出如下信息：

> -IC:/MinGW/include/libusb-1.0  -LC:/MinGW/lib/libusb-1.0/static -lusb-1.0

#### b. 静态链接 libusb

对之前创建的 `libusb-1.0.pc` 文件进行修改，将库所在的目录改为静态链接库所在的目录，将最后一行的链接库由 `-llibusb-1.0` 改为 `-lusb-1.0`：

```
prefix=C:\MinGW
exec_prefix=${prefix}
includedir=${prefix}/include/libusb-1.0
libdir=${exec_prefix}/lib/libusb-1.0/static	# 静态链接库所在目录

Name: libusb
Description: libusb
Version: 1.0
Cflags: -I${includedir}
Libs: -L${libdir} -lusb-1.0	# 使用静态链接库
```

按照同样的步骤，指定环境变量，没问题的话，验证会有如下输出：

> -IC:/MinGW/include/libusb-1.0  -LC:/MinGW/lib/libusb-1.0/static -lusb-1.0

## 3. 运行示例程序

新建项目，例如 `usb_test`：

```powershell
cargo new usb_test
```

在项目配置文件 `Cargo.toml` 中添加如下依赖：

```toml
libusb = "0.3.0"
```

在 `main.rs` 中创建如下内容：

```rust
extern crate libusb;

fn main() {
    let context = libusb::Context::new().unwrap();
    for device in context.devices().unwrap().iter() {
        let device_desc = device.device_descriptor().unwrap();

        println!("Bus {:03} Device {:03} ID {:04x}:{:04x}",
            device.bus_number(),
            device.address(),
            device_desc.vendor_id(),
            device_desc.product_id());
    }
}
```

编译：

```cargo
cargo build
```

运行时需要注意，如前面所说，如果选择了静态链接的方式，直接运行就可以，如果选择了动态链接 libusb 的方式，运行可执行文件前要将运行时库 `libusb-1.0.dll` 复制到可执行文件的同级目录下，否则会报如下错误：

> error: process didn't exit successfully: `target\debug\usb.exe` (exit code: 0xc0000135, STATUS_DLL_NOT_FOUND)

例如，默认情况下，编译生成的可执行文件在 `target\debug` 目录内，故需将 `libusb-1.0.dll` 复制到 `target\debug` 目录下内，如果生成的是 release 版本，甚至是交叉编译版本，都需要放到对应的目录去，执行 `cargo clean` 后得重新拷贝，比较麻烦。

运行：

```rust
cargo run
```

## 参考

[1] [How to install pkg config in windows?](https://stackoverflow.com/questions/1710922/how-to-install-pkg-config-in-windows)
