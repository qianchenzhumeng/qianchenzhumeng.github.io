---
layout: post
title:  "为芒果派 MQ-R F133 编译 iot_gw 网关"
date:   2022-10-27 23:26:00 +0800
categories: [Embedded, OpenWrt]
tags: [OpenWrt]
excerpt: 每次拿到一块可以跑 linux 的开发板之后，都会产生为其编译 iot_gw 的冲动，这次也不例外。
---

前前后后花了大概半个月的时间，总算完成了两件事：

- 为 `iot_gw` 替换串口库，避免移植到新平台时适配串口的繁琐流程。
- 为芒果派 MQ-R F133 编译 `iot_gw` 网关。

RUST 和 RISC-V，都是比较新的东西，二者结合的时候，遇到的问题也比较多。这次编译主要卡在 `openssl`、`paho-mqtt-sys` 的编译上，概括来说，有以下几个方面：

- 老版本的 `openssl`，比如 `openssl-1.0.2l`，没有 `riscv` 的配置，使用 `riscv64-unknown-linux-gnu` 工具链编译时，最后编译出的库格式无法被链接器识别。
- `riscv64gc-unknown-linux-gnu` 的 `ssize_t` 是 4 个字节，和指针 8 个字节大小不同，为 C 代码生成 Rust 绑定时需要传入 `--no-size_t-is-usize` 来避免 `rust-bindgen` 报错。
- Rust 和 LLVM 为 RISC-V 定义的三元组名称不同，为 C 代码生成 Rust 绑定时，`rust-bindgen` 传给 `clang` 的三元组无法被 `clang` 识别，`clang` 报错。

具体问题的处理见 `iot_gw` 的 [`README`](https://github.com/qianchenzhumeng/iot_gw/tree/dev#readme)。