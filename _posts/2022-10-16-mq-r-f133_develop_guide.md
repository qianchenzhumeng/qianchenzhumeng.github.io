---
layout: post
title:  "芒果派 MQ-R F133 开发指南"
date:   2022-10-17 00:04:00 +0800
categories: [Embedded, OpenWrt]
tags: [OpenWrt]
excerpt: 最近入手了一块带有 WIFI 模块的芒果派 MQ-R F133 开发板，这个开发板很小巧，可以运行全志的 Tina-Linux，打算将之前用 Rust 开发的网关应用 iot_gw 移植到这个上面，这篇博客主要用于记录移植的过程。
---

## 1. 开发环境搭建

搭建开发环境比较简单，芒果派 [Tina-Linux](https://github.com/mangopi-sbc/Tina-Linux) 代码库的自述文件中有在 Ubuntu18.04 上搭建 F133 开发环境的的指导，唯一比较折磨人的问题是下载 SDK 的时间比较久。

如果是在 WSL 中搭建开发环境，需要注意以下两个问题：

1.注意不要将 Tina-Linux 随意放置在 C 盘、D 盘等位置上，实测这样做的时候，即使对应目录开启了大小写敏感，编译时也会发生莫名其妙的问题。需要放置在 WSL 用户的家目录下，编译才能正常进行。

2.确保 WSL 的版本是 2。编译打包的过程会调用一些工具，有一部分工具是 32 位的，WSL1 不支持运行 32 位程序，需要升级为 WSL2。

WSL1 升级到 WSL2：

```powershell
# 查看名称和版本号
wsl -l -v
```

比如：

```powershell
  NAME            STATE           VERSION
* Ubuntu          Running         1
  Ubuntu-18.04    Stopped         2
```

设置版本：

```powershell
wsl --set-version Ubuntu 2
```

等个几分钟就可以了。

## 2. 使能 uart1

默认只有 uart0 使能了，用作 linux 的控制台。开发板上电后，可以在 `/dev` 目录下看到：

```bash
root@TinaLinux:/# ls -l /dev/tty*
crw-rw-rw-    1 root     root        5,   0 Jan  1 00:00 /dev/tty
crw-------    1 root     root      249,   0 Jan  1 00:00 /dev/ttyS0
```

网关 [iot_gw](https://github.com/qianchenzhumeng/iot_gw) 需要使用串口和 Arduino 交互，因此，需要占用一个串口，计划使用 uart1。需要做两件事情：

1. 使能 uart1。
2. 确定 uart1 对应的引脚。

需要编辑设备树文件，使能 uart1：

```bash
~/Tina-Linux$ vim ./device/config/chips/f133/configs/mq_r/board.dts
```

找到 uart1 的配置：

```
&uart1 {
        pinctrl-names = "default", "sleep";
        pinctrl-0 = <&uart1_pins_a>;
        pinctrl-1 = <&uart1_pins_b>;
        status = "disabled";
};
```

修改状态：

```
&uart1 {
        pinctrl-names = "default", "sleep";
        pinctrl-0 = <&uart1_pins_a>;
        pinctrl-1 = <&uart1_pins_b>;
        status = "okay";
};
```

单看开发板原理图，可以看到很多引脚都可以做为 uart1 使用，具体是哪个引脚，需要查看设备树文件。

```
        uart1_pins_a: uart1_pins@0 {  /* For EVB1 board */
                pins = "PG6", "PG7", "PG8", "PG9";
                function = "uart1";
                drive-strength = <10>;
                bias-pull-up;
        };

        uart1_pins_b: uart1_pins {  /* For EVB1 board */
                pins = "PG6", "PG7", "PG8", "PG9";
                function = "gpio_in";
        };
```

从上面的内容可以看到，uart1 对应的引脚是 PG6、PG7、PG8、PG9，结合开发板原理图可知，UART1-TX 是 PG6，UART1_RX 是 PG7。

接下来，编译、打包、烧录即可。开发板运行起来后，就可以在 '/dev' 目录下看到 uart1

了：

```bash
root@TinaLinux:/# ls -l /dev/tty*
crw-rw-rw-    1 root     root        5,   0 Jan  1 00:00 /dev/tty
crw-------    1 root     root      249,   0 Jan  1 00:00 /dev/ttyS0
crw-------    1 root     root      249,   1 Jan  1 00:00 /dev/ttyS1
```

## 3. Rust 串口编程

网关 [iot_gw](https://github.com/qianchenzhumeng/iot_gw) 是用 Rust 写的，操作串口借助了 [serial](https://crates.io/crates/serial) 库，这个库使用了 [termios](https://crates.io/crates/termios) 库和 [ioctl-rs](https://crates.io/crates/ioctl-rs) 库（三个库的开发者是同一个人），之前在为 `mips-unknown-linux-uclibc` 开发这个网关程序时，发现不同的 linux 发行版，ioctl 接口同一命令号对应的命令含义并不相同，因此，编译网关应用前还得单独处理这三个库，才能确保应用能够正确操作串口，很麻烦。

计划试一试另一个串口库 [serialport](https://crates.io/crates/serialport)。这个库的介绍页面提到，工作在 Linux 上时，这个库提供的串口枚举功能是使用 `glibc` 实现的，借助了 `libudev`，如果使用这个特性，仍然可以枚举串口，但是可能无法获得比较丰富的信息，并且也可能返回一些物理上不存在的串口。这个特性在网关上用不到，不过，本着钻研的目的，编译网关应用时还是不要关闭该特性。

使用 [serialport](https://crates.io/crates/serialport) 提供的 `list_ports` 示例创建 rust 工程，为工程添加 `riscv64gc-unknown-linux-gnu` 目标编译配置：

```toml
# .cargo/config.toml
[target.riscv64gc-unknown-linux-gnu]
linker = "riscv64-unknown-linux-gnu-gcc"
ar = "riscv64gc-unknown-linux-gnu-ar"
```

创建编译脚本 `build_f133.sh`：

```bash
#!/bin/bash

# 注意，将 HOME 替换为实际的路径
HOME="/home/dell"
TARGET="riscv64gc-unknown-linux-gnu"

CC="riscv64-unknown-linux-gnu-gcc"
TOOLCHAIN=~/Tina-Linux/out/f133-mq_r/staging_dir/toolchain

export CC_riscv64gc_unknown_linux_gnu=$CC
export PATH=$PATH:$TOOLCHAIN/bin

cargo build --target=$TARGET -vv
```

接下来，赋予该脚本执行权限，并运行：

```bash
chmod +x build_f133.sh
./build_f133.sh
```

很遗憾，报错了：

> [libudev-sys 0.1.4] thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "pkg-config has not been configured to support cross-compilation.\n\nInstall a sysroot for the target platform and configure it via\nPKG_CONFIG_SYSROOT_DIR and PKG_CONFIG_PATH, or install a\ncross-compiling wrapper for pkg-config and set it via\nPKG_CONFIG environment variable."', /home/dell/.cargo/registry/src/mirrors.ustc.edu.cn-12df342d903acd47/libudev-sys-0.1.4/build.rs:38:41
> [libudev-sys 0.1.4] note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
> error: failed to run custom build command for `libudev-sys v0.1.4`

找不到 `libudev` 的 `pkg-config` 配置文件。

先在 SDK 中 `make menuconfig`， 使能 `eudev` ，编译出 `libudev` 看一看能不能产生 `pkg-config` 配置文件：

```
    Base system  --->
		<*> eudev................................ Dynamic device management subsystem  --->
```

编译完成之后，可以看到生成了 `libudev` 的 `pkg-config` 配置文件：

```bash
~/Tina-Linux$ ls ~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/pkgconfig
alsa.pc     ext2fs.pc     libcares.pc   libcurl.pc   libiptc.pc     libpng.pc    libudev.pc  openssl.pc    uuid.pc
blkid.pc    freetype2.pc  libconfig.pc  libip4tc.pc  libnghttp2.pc  libpng12.pc  lua.pc      smartcols.pc  xtables.pc
com_err.pc  json-c.pc     libcrypto.pc  libip6tc.pc  libnl-tiny.pc  libssl.pc    mount.pc    speexdsp.pc   zlib.pc
```

修改编译脚本，通过环境变量指定 `pkg-config` 配置文件目录，并使能 `pkg-config` 交叉编译：

```bash
#!/bin/bash

# 注意，将 HOME 替换为实际的路径
HOME="/home/dell"
TARGET="riscv64gc-unknown-linux-gnu"

CC="riscv64-unknown-linux-gnu-gcc"
TOOLCHAIN=~/Tina-Linux/out/f133-mq_r/staging_dir/toolchain
STRIP=riscv64-unknown-linux-gnu-strip

export PKG_CONFIG_LIBDIR=~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/pkgconfig
export PKG_CONFIG_ALLOW_CROSS=1
export CC_riscv64gc_unknown_linux_gnu=$CC
export PATH=$PATH:$TOOLCHAIN/bin

cargo build --target=$TARGET -vv
```

接下来还是报错，找不到头文件 `zlib.h`：

> cargo:warning=In file included from libgit2/src/attr_file.c:11:
>   cargo:warning=libgit2/src/filebuf.h:14:10: fatal error: zlib.h: No such file or directory
>   cargo:warning= #include <zlib.h>
>   cargo:warning=          ^~~~~~~~
>   cargo:warning=compilation terminated.

修改编译脚本，指定头文件搜索路径：

```bash
#!/bin/bash

# 注意，将 HOME 替换为实际的路径
HOME="/home/dell"
TARGET="riscv64gc-unknown-linux-gnu"

CC="riscv64-unknown-linux-gnu-gcc"
CFLAGS="-I$HOME/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/include"
TOOLCHAIN=~/Tina-Linux/out/f133-mq_r/staging_dir/toolchain
STRIP=riscv64-unknown-linux-gnu-strip

export PKG_CONFIG_LIBDIR=~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/pkgconfig
export PKG_CONFIG_ALLOW_CROSS=1
export CC_riscv64gc_unknown_linux_gnu=$CC
export CFLAGS_riscv64gc_unknown_linux_gnu=$CFLAGS
export PATH=$PATH:$TOOLCHAIN/bin

cargo build --target=$TARGET -vv
```

链接时报错，找不到 `-lz` 及 `-ludev`：

>   = note: /home/dell/Tina-Linux/prebuilt/gcc/linux-x86/riscv/toolchain-thead-glibc/riscv64-glibc-gcc-thead_20200702/bin/../lib/gcc/riscv64-unknown-linux-gnu/8.1.0/../../../../riscv64-unknown-linux-gnu/bin/ld: cannot find -lz
>           /home/dell/Tina-Linux/prebuilt/gcc/linux-x86/riscv/toolchain-thead-glibc/riscv64-glibc-gcc-thead_20200702/bin/../lib/gcc/riscv64-unknown-linux-gnu/8.1.0/../../../../riscv64-unknown-linux-gnu/bin/ld: cannot find -ludev
>           collect2: error: ld returned 1 exit status

修改编译脚本，指定 `ld` 的搜索路径：

```bash
#!/bin/bash

# 注意，将 HOME 替换为实际的路径
HOME="/home/dell"
TARGET="riscv64gc-unknown-linux-gnu"

CC="riscv64-unknown-linux-gnu-gcc"
CFLAGS="-I$HOME/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/include"
LDFLAGS="-L$HOME/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib"
TOOLCHAIN=~/Tina-Linux/out/f133-mq_r/staging_dir/toolchain
STRIP=riscv64-unknown-linux-gnu-strip

export PKG_CONFIG_LIBDIR=~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/pkgconfig
export PKG_CONFIG_ALLOW_CROSS=1
export CC_riscv64gc_unknown_linux_gnu=$CC
export CFLAGS_riscv64gc_unknown_linux_gnu=$CFLAGS
export LDFLAGS_riscv64gc_unknown_linux_gnu=$LDFLAGS
export PATH=$PATH:$TOOLCHAIN/bin

cargo build --target=$TARGET -vv
```

没起作用，仍旧报上述错误，但是编译输出信息中可以看到，上述操作生效了：

> "-L" " /home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib"

找不到的 `` 和 `` 在 `~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/` 是存在的：

```bash
ls ~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev*
/home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev.so
/home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev.so.1
/home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev.so.1.6.3
ls ~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libz.*
/home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libz.a
/home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libz.so
/home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libz.so.1
/home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libz.so.1.2.8
```

不知道为啥还是找不到，干脆为这两个库创建软链接（不知道为啥会调到 `prebuilt` 目录中的工具）：

```bash
# 创建 libudev.so 的软链接
ln -s ~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev.so ~/Tina-Linux/prebuilt/gcc/linux-x86/riscv/toolchain-thead-glibc/riscv64-glibc-gcc-thead_20200702/riscv64-unknown-linux-gnu/lib/libudev.so
# 创建 libz.so 的软链接
ln -s ~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libz.so ~/Tina-Linux/prebuilt/gcc/linux-x86/riscv/toolchain-thead-glibc/riscv64-glibc-gcc-thead_20200702/riscv64-unknown-linux-gnu/lib/libz.so
```

删掉刚才在编译脚本中添加的链接标志：

```bash
#!/bin/bash

# 注意，将 HOME 替换为实际的路径
HOME="/home/dell"
TARGET="riscv64gc-unknown-linux-gnu"

CC="riscv64-unknown-linux-gnu-gcc"
CFLAGS="-I$HOME/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/include"
TOOLCHAIN=~/Tina-Linux/out/f133-mq_r/staging_dir/toolchain
STRIP=riscv64-unknown-linux-gnu-strip

export PKG_CONFIG_LIBDIR=~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/pkgconfig
export PKG_CONFIG_ALLOW_CROSS=1
export CC_riscv64gc_unknown_linux_gnu=$CC
export CFLAGS_riscv64gc_unknown_linux_gnu=$CFLAGS
export PATH=$PATH:$TOOLCHAIN/bin

cargo build --target=$TARGET -vv
```

再编译就顺利通过了。

将编译出的二进制文件放到开发板上运行时，会报找不到 `libudev`，需要将 `~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib` 目录下的 `libudev` 上传到开发板上去。总共有三个文件：

```bash
ls -l ~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev.so*
lrwxrwxrwx 1 dell dell     16 Oct 16 14:36 /home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev.so -> libudev.so.1.6.3
lrwxrwxrwx 1 dell dell     16 Oct 16 14:36 /home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev.so.1 -> libudev.so.1.6.3
-rwxr-xr-x 1 dell dell 145904 Oct 16 14:36 /home/dell/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev.so.1.6.3
```

可以看到，前两个实际上是软链接，而且，无法使用 `adb` 工具将前两个文件传到开发板上，因此，仅将 `libudev.so.1.6.3` 传到开发板的 `/lib` 目录下，并为其创建软链接。

或者重新打包，将 `libudev` 的库打包到镜像中，重新烧录：

```bash
# 打包前将 libudev 的库复制到开发板的 rootfs 目录中
cp ~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/libudev.* ~/Tina-Linux/out/f133-mq_r/compile_dir/target/rootfs/lib
```

之后，编译出的二进制文件就可以在开发板上运行了。