---
layout: post
title:  "Rust 串口编程"
date:   2021-07-22 20:59:00 +0800
categories: [Embedded, Other]
tags: [SerialPort]
excerpt: 这篇文章主要介绍了如何使用 Rust 编写 Linux 上的串口通信程序。
---

## 1. 背景

如果使用 C 来编写 Linux 上的串口通信程序，需要使用 termios，tldp 有详细的示例：[Serial-Programming-HOWTO](https://tldp.org/HOWTO/Serial-Programming-HOWTO/x56.html#AEN88)。

使用 Rust 编写串口通信程序，需要借助三个库：[serial](https://crates.io/crates/serial)、[ioctl-rs](https://crates.io/crates/ioctl-rs) 以及 [termios](https://crates.io/crates/termios)。serial 既支持 Linux，也支持 Windows，ioctl-rs 是对 Unix 上系统调用的部分 C 库的封装，termios 是对 Unix 上终端 I/O 的 C 库的封装，serial 依赖后面的两个库。

## 2. 编码

### (1) 配置工程

```bash
cargo new serial_port
```

在配置文件中添加依赖：

```toml
[dependencies]
serial = "0.4.0"
```

### (2) 示例程序

```rust
extern crate serial;

use std::time::Duration;
use std::thread;
use serial::prelude::*;
use std::io::prelude::*;

fn main() {
    const SETTINGS: serial::PortSettings = serial::PortSettings {
        baud_rate: serial::Baud115200,
        char_size: serial::Bits8,
        parity: serial::ParityNone,
        stop_bits: serial::Stop1,
        flow_control: serial::FlowNone,
    };

    let mut port = serial::open("/dev/ttyS9").unwrap();
    port.configure(&SETTINGS).unwrap();
    port.set_timeout(Duration::from_millis(1000)).unwrap();
    let mut buf: Vec<u8> = (0..255).collect();
    loop {
        if let Ok(n) = port.read(&mut buf[..]) {
            for i in 0..n {
                print!("{:02x }", buf[i]);
            }
        };
        thread::sleep(Duration::from_millis(10));
    }
}
```

在串口上接上诸如 Arduino 之类的设备，让其按上面的串口配置向上位机发送数据，然后运行上述示例程序，不出意外的话就可以看到输出。

### (3) 为本地机器生成绑定

实际使用过程中，可能会遇到一些问题，例如：

> thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Error { kind: Io(Other), description: "Inappropriate ioctl for device" }', src/main.rs:17:47

错误信息显示下发给该设备的系统调用不正确。这个其实是之前提到的 ioctl-rs 库的问题。不同的 Linux 版本，ioctl 的命令很有可能是不同的，即 C 库头文件中各命令对应的宏的数值可能不一样。ioctl-rs 是系统调用相关的 C 库的封装，serial 会直接使用 [crates.io](https://crates.io/) 上的 ioctl-rs 库，本地的机器上和生成 ioctl-rs 库的机器，ioctl 命令值有差异时，就会出现上面的问题。

需要为本地机器重新生成相应的绑定。

生成绑定之前，将需要库下载到工程目录，并修改相应库的路径：

```
git clone https://github.com/dcuddeback/serial-rs.git
git clone https://github.com/dcuddeback/ioctl-rs.git
git clone https://github.com/dcuddeback/termios-rs.git
```

修改 serial-rs/serial-unix/Cargo.toml 文件内的依赖项：

```toml
termios = { version = "0.3", path = "../../termios-rs" }
ioctl-rs = { path = "../../ioctl-rs" }
```

使用 ioctl_list 工具生成绑定：

```bash
cd ioctl_list/
./autogen.sh && ./configure && make
./ioctl_list
```

使用屏幕上输出的内容替换 ioctl-rs/src/os/linux.rs 文件内相应的内容即可。

ioctl-rs 作者提到，该工具打印的常量，其类型可能会不准确[[2]](https://github.com/dcuddeback/ioctl-rs/tree/master/ioctl_list)，编译过程中可能需要把部分常量的类型由 c_int 改为 c_uint。

## 3. 故障排查

使用最新的库可能会有问题，例如，在 wsl2 上可能还会报系统调用错误的问题，大部分还是命令值的问题，可能还得手工矫正，必要的话，可以试一试这几个矫正过的：

```
https://github.com/qianchenzhumeng/serial-rs.git
https://github.com/qianchenzhumeng/ioctl-rs.git
https://github.com/qianchenzhumeng/termios-rs.git
```

切换到分支 pi。

## 参考

[1] [https://tldp.org/HOWTO/Serial-Programming-HOWTO/x56.html#AEN88](https://tldp.org/HOWTO/Serial-Programming-HOWTO/x56.html#AEN88)

[2] [https://github.com/dcuddeback/ioctl-rs/tree/master/ioctl_list](https://github.com/dcuddeback/ioctl-rs/tree/master/ioctl_list)