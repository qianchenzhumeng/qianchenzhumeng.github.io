---
layout: post
title:  "自由使用 Dragino 网关 MCU 串口"
date:   2020-08-20 23:41:00 +0800
categories: [Embedded, OpenWrt]
tags: [OpenWrt]
---

## 网关

Dragino LG01-P LoRa Gateway

![LG01-P](/assets/img/2020-08-20-自由使用Dragino网关MCU串口.assets/LG01-P.png)

Linux Side:

- Processor: AR9331

- Frequency: 400MHz

- Flash: 16MB

- RAM: 64MB

MCU/LoRa Side:

- MCU: ATMega328P

- Flash: 32KB

- RAM: 2KB

- LoRa Chip: SX2176/78

网关设计应该借鉴了 Arduino Yun，运行 OpenWRT 系统。

## 串口

和 MCU 相连的串口 /dev/ttyATH0 和 /dev/console 应该有关系，从测试情况看二者都可以收到从 MCU 串口过来的数据。

```
root@dragino-16e828:~# ls -l /dev
crw-r--r-- 1 root root 5, 1 Jan 1 1970 console
crw-r--r-- 1 root root 5, 0 Jan 1 1970 tty
crw-r--r-- 1 root root 253, 0 Jan 1 1970 ttyATH0
crw-r--r-- 1 root root 4, 64 Jan 1 1970 ttyS0
```

即使在配置 PowerUART 时禁用了 Linux 控制台，应用程序也无法从上述两个设备文件中读取到完整的数据（会有部分片段丢失）。

连续输入 lsof 命令，可以看到程序 askfirst（/sbin/askfirst） 在访问 /dev/ttyATH0：

```
root@dragino-16e828:~# lsof /dev/ttyATH0
COMMAND PID USER FD TYPE DEVICE SIZE/OFF NODE NAME
askfirst 29587 root 0u CHR 253,0 0t0 215 /dev/ttyATH0
askfirst 29587 root 1u CHR 253,0 0t0 215 /dev/ttyATH0
askfirst 29587 root 2u CHR 253,0 0t0 215 /dev/ttyATH0
```

杀掉对应的进程后，程序仍然会再次运行，先重命名该程序再杀掉对应的进程可以阻止 askfirst 继续运行。

该程序不断运行的原因是 /etc/inittab 中有对应的项：

```
root@dragino-16e828:~# cat /etc/inittab
::sysinit:/etc/init.d/rcS S boot
::shutdown:/etc/init.d/rcS K shutdown
::askconsole:/bin/ash --login
```

注释掉 /etc/inittab 中有 askconsole 的项后重启网关即可自由使用 `/dev/ttyATH0`：

```
root@dragino-16e828:~# cat /etc/inittab
::sysinit:/etc/init.d/rcS S boot
::shutdown:/etc/init.d/rcS K shutdown
#::askconsole:/bin/ash --login
```

## 参考

[1] [http://www.dragino.com/products/lora-lorawan-gateway/item/117-lg01-p.html](http://www.dragino.com/products/lora-lorawan-gateway/item/117-lg01-p.html)
