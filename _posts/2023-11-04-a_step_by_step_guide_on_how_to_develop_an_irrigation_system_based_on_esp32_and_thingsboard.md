---
layout: post
title:  "基于 ESP32 和 ThingsBoard 从零开始搭建远程浇花系统（陆续更新）"
date:   2023-11-04 21:44:00 +0800
categories: [IoT, Irrigation]
tags: [IoT]
excerpt: 分享如何使用 ESP32 和 ThingsBoard 搭建一个可以进行远程浇花的物联网系统。
---

### 1. 背景

不得不说，这是个激动人心的时刻。从十年前在大学接触 C 语言开始，就有了一个宏大的目标，不断学习相关知识，希望有朝一日，能够搭建一个物联网系统，逐步实现农业生产的自动化，先是远程监测，再是远程控制，最后是自动控制。当时就已经明白，对于自己来说，这个目标过于宏大，至于何时能够实现，心里也没有底。但是，凭借一腔热血，就这么一路学了过来。终于，不管是内部条件还是外部条件，都能支撑我迈上一个新的台阶了。

两年前做了一个基于 Lora 的温湿度监控系统，可以说将习得的大部分知识遍历了一遍，包括：设计温湿度终端的外形及内部结构，通过 3D 打印让其从数字世界进入现实世界；开发温湿度终端的程序、设计 PCB，使其在使用 2000mAh 锂电池、按 5min 一次的频率将数据发送到 1km 外的网关的情况下工作 1 个月；基于成品的开发板开发网关软件，将来自终端的数据发送到服务器上；自己开发服务器软件（急流勇退了，鉴权、加密等，不是一个人能完成的，完成了一些简单的功能后决定尝试开源的物联网平台）；筛选合适的开源物联网平台（过程也比较艰难，在电脑上挨个搭建试用，最终选了 ThingsBoard）；买服务器、买域名、域名备案等等。当时，每个环节都举步维艰。博客中的好多博文都是跟这个有关的。

那个时候还没有集成 Lore 的 MCU，需要使用 Lora 模块加 MCU 的方案。终端最先用的是安信可的 Ra-02 模块和一款 STM8L 系列的单片机，网关买的联粹科技的 LG-01P。期间遇到了不少问题，信号传输距离达不到预期、功耗达不到预期、所选 STM8L 的 Flash 空间太小导致程序放不下、终端和网关无法通信等，凡此种种，更要命的是，终端和网关都在我老家，好多测试只能在放假或者请假回家的时候做，于是，每次回家我都带着电脑、调试器、焊接工具等。还有网关，先是用的厂家的示例代码，Linux 侧和 MCU 侧通信用的是 Arduino 的 Bridge 方案，一方面，Linux 和 MCU 只能单向通信，另一方面，MCU 上的内存资源消耗也比较严重，时长出现增加功能导致栈溢出的问题。后来废弃了这个方案，自己写 Linux 侧的程序，和 MCU 使用串口通信，先是用 C 语言开发 Linux 侧的网关程序，效率低不说，BUG 也层出不穷，后来为此花了半年时间学习 Rust，但是不幸的是，LG-01P 用的 OpenWrt 版本比较老，使用的工具链是 mips-unknown-linux-uclibc，好巧不巧，Rust 没有 uclibc 的版本，没办法，只能硬着头皮为其编译 Rust 工具链，真是艰难呐，工具链编译及其耗时间，需要七八个小时，而且最初的时候，编译总出错，夜没少熬。后来的路依旧艰难，uclibc 的 ioctl 命令和 gcc 的不太一样，需要修改 Rust 的串口库，解决 ioctl 命令不匹配的问题，uclibc 的工具链里面没有 openssl，需要从源码编译 openssl 以支持 MQTT。不过功夫不负有心人，最终还是做成了。用了 Rust，后面不管是效率还是质量，都比较高了。这个系统已经稳定工作了两年多了，远程监测算是实现了。

现在，得益于 ESP32 的存在，总算实现了远程控制。为什么不用前面温湿度终端的方案呢？因为，远程控制就涉及到安全问题了，要加密、要防重放攻击等等，虽然网关和服务器实现了 MQTTS 双向认证，但是，终端和网关没有可靠地加密机制与防重放攻击的机制，在自定义的协议基础上做这个事情，得不偿失。但是，可不可以选用 LoraWAN 呢？当时考虑过，一方面，LoraWAN 网关不便宜，另一方面，没有开源的 LoraWAN 代码可用，再一方面，LoraWAN 需要服务器支持，当时的 ThingsBoard 还不支持。因此，这显然不是我一个人能搞定的事情。为什么 ESP32 可以呢？因为它的资源丰富，足够存得下 X.509 证书，这样就可以使用 MQTTS 协议以及 HTTPS 协议了，而且使用 MQTTS 时可以进行双向认证，于是，安全问题就解决了。不过，最终还是要走上使用 LoraWAN 的道路，毕竟，Wi-Fi 没有那么大的覆盖范围。

接下来进入正题，介绍如何使用 ESP32 和 ThingsBoard 搭建一个可以进行远程浇花的物联网系统，需要有一些知识或技能储备，包括：会搭建 ESP-IDF 开发环境，会使用 Linux 操作系统，了解非对称加密，了解什么是域名以及二级域名，了解 MQTT 协议，还有一点点动手能力，当然，还要有足够的耐心，毕竟，环节越多，涉及的知识点越多，遇到困难的可能性也越大。

先来看看最终的系统长啥样：

![irrigation](/assets/img/2023-11-04-a_step_by_step_guide_on_how_to_develop_an_irrigation_system_based_on_esp32_and_thingsboard.assets/irrigation.jpg)

### 2. 系统架构及功能描述

该系统由三部分构成，第一部分是主控为 ESP32 的终端，第二部分是自建的 ThingsBoard 物联网平台，第三部分是 ThingsBoard 手机 App。终端通过 Wi-Fi 接入互联网，通过 MQTTS 协议接入自建的 ThingsBoard 物联网平台，通过 HTTPS 协议进行 OTA 升级，我们可以使用电脑或手机远程控制终端来浇花。

（持续更新中）