---
layout: post
title:  "Cross Compile Rust For OpenWRT"
date:   2020-08-09 00:57:29 +0000
categories: Rust
---
# 背景

OpenWRT SDK: OpenWrt-SDK-ar71xx-for-linux-x86_64-gcc-4.8-linaro_uClibc-0.9.33.2

工具链目录：staging_dir\toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2\bin

目录内的 "mips-openwrt-linux-" 工具是指向 "mips-openwrt-linux-uclibc-" 的软连接，即工具链使用的是 uclibc 的版本。

uClibc 和 glibc 有差异，有些应用使用 uClibc 可能无法编译。

Rust 目前的官方构建还不支持 mips-unknown-linux-uclibc[1]，从目前 Rust 支持的平台来看，uclibc 对应的 std 是支持的（但是稳定版中没有），target-list 中虽然可以显示出 mips-unknown-linux-uclibc，但是添加是会显示失败。 

```
$ rustc --print target-list | grep mips
mips-unknown-linux-gnu
mips-unknown-linux-musl
mips-unknown-linux-uclibc
```

添加 gnu 和 musl target 后，rust 安装目录下会出现对应的标准库：

```
$ ls -l ~/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/ | grep mips
-rw-rw-rw- 1 Dell Dell   1836 Jun 26 20:02 manifest-rust-std-mips-unknown-linux-gnu
-rw-rw-rw- 1 Dell Dell   2015 Jun 27 18:41 manifest-rust-std-mips-unknown-linux-musl
drwxrwxrwx 1 Dell Dell    512 Jun 26 20:02 mips-unknown-linux-gnu
drwxrwxrwx 1 Dell Dell    512 Jun 27 18:41 mips-unknown-linux-musl
```

# 交叉编译 Rust

从 config.toml.example 复制一份 config.toml 文件，修改 config.toml 文件，指定要编译的 target，使能工具编译（默认不编），指定安装目录（相对路径，如果是系统的路径，安装时没有权限，会失败）及交叉编译用到的工具链：

```
# 修改
target = ['mips-unknown-linux-uclibc']
extended = true
tools = ["cargo", "rls", "clippy", "rustfmt", "analysis", "src"]
prefix = 'usr/local'
sysconfdir = "etc"
docdir = "share/doc/rust"
bindir = "bin"
libdir = "lib"
mandir = "share/man"
# 添加（在 [target.x86_64-unknown-linux-gnu] 之后）
[target.mips-unknown-linux-uclibc]
cc = "mips-openwrt-linux-uclibc-gcc"
cxx = "mips-openwrt-linux-uclibc-c++"
ar = "mips-openwrt-linux-uclibc-ar"
linker = "mips-openwrt-linux-uclibc-gcc"
```

编译并安装：

```
~/wsl/rust$ ./x.py build && ./x.py install  
```

安装完成后库和可执行文件位于以下目录：

```
~/wsl/rustrust/usr/local/lib
~/wsl/rust/local/bin
```

将路径添加到环境变量（或者修改 ~/.profile 文件）：

```
~/wsl/rust$ export PATH=$PATH:~/wsl/rust/usr/local/bin
~/wsl/rust$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/wsl/rust/usr/local/lib
```

# 测试

编译 hello 工程：

```
$ cargo new hello 
```

添加 hello/.cargo/config 配置文件：

```
[target.mips-unknown-linux-uclibc]
linker = "mips-openwrt-linux-uclibc-gcc"
ar = "mips-openwrt-linux-uclibc-ar"
```

构建：

```
hello$ cargo build --target=mips-unknown-linux-uclibc
hello$ mips-openwrt-linux-strip target/mips-unknown-linux-uclibc/debug/hello
```

如果编译单个文件：

```
$ rustc hello.rs --target=mips-unknown-linux-uclibc -C linker=mips-openwrt-linux-uclibc-gcc -C ar=mips-openwrt-uclibc-ar
$ mips-openwrt-linux-strip hello
```

# 其他尝试

（1）探索将编译出的标准库打包，且能直接安装

无需额外步骤，执行安装步骤后，安装包会出现在 build/dist 目录下。

直接将 std 安装包内或者安装路径内 rustlib 下的 mips-unknown-linux-uclibc 复制到 {sysroot}/lib/rustlib 下即可，但是如果和系统安装的 rust 版本不符，无法编译程序，会提示 std 和 rustc 版本不符。即使从源码检出系统安装的 rust 版本对应的提交记录（rustc -Vv 查看），编译出的版本会带 -dev 后缀，编译程序时会显示 std 版本不符。尝试失败。

（2）尝试在 spec 目录下添加 mips-openwrt-linux-uclibc

# 参考

[1] https://forge.rust-lang.org/release/platform-support.html