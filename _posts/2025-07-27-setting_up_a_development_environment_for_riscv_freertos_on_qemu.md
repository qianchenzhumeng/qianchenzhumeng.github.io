---
layout: post
title:  "在 QEMU 上搭建 RISC-V 和 FreeROTS 的开发环境"
date:   2025-07-27 22:56:00 +0800
categories: [Embedded]
tags: [FreeRTOS, RISC-V]
excerpt: 这篇文章描述了如何在 QEMU 上搭建 RISC-V 和 FreeRTOS 的开发调试环境。本文的内容几乎全部来自这篇博文：[Setting up a development environment for RISCV-FreeRTOS on QEMU](https://embeddedinn.com/articles/tutorial/RISCV-FreeRTOS-QEMU/)
---

## 1. 背景

F在这篇博客中，我们在 Ubuntu 20.04.6 LTS 上从源码编译 RISC-V 工具链以及 QEMU，建立模拟开发环境，然后在这个基础上使用 VSCode 调试 FreeRTOS 的示例代码。实际上，从源码编译工具链很耗时而且从 github 克隆代码还会遇到网络问题，因此首选下载别人编译好的工具链，免去编译的繁琐工作。写这篇博客时，编译环境是 Ubuntu 20.04.6 LTS 的虚拟机，分配了 3 个 核，10G 内存，从下代码到最终编译成功，花了 1 天时间。

## 2. 安装工具链

先安装依赖：

```bash
sudo apt-get install autoconf automake autotools-dev curl python3 python3-pip libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake libglib2.0-dev libslirp-dev
```

编译过程比较耗时。首先，需要从 github 克隆源码，并且编译期间也会从 github 克隆源码，体积比较大，需要处理好网络问题。此外，如果是虚拟机，内存分配大一点，内存小的话，中途会出现退出的情况（报错 collect2: fatal error: ld terminated with signal 9 [Killed]）。一开始我应该只给虚拟机分配了 3G 或者 5G 内存，会报这个错误，后来增加到 10G 内存后就可以成功编译了。

为工具链准备安装目录：

```bash
sudo mkdir -p /opt/riscv
sudo chown $USER:$USER /opt/riscv
```

克隆 RISC-V 工具链：

```bash
git clone https://github.com/riscv/riscv-gnu-toolchain
```

编译并安装：
```bash
cd riscv-gnu-toolchain
./configure --prefix=/opt/riscv --enable-multilib
make -j$(nproc)
```

结束后可以在 `/opt/riscv 中看到编译好的工具链。

将工具链路径加入 PATH 环境变量：

```bash
echo 'export PATH=$PATH:/opt/riscv/bin' >> ~/.bashrc
source ~/.bashrc
```

检查是否安装成功：

```bash
$ riscv64-unknown-elf-gcc --version
riscv64-unknown-elf-gcc (g1b306039a) 15.1.0
Copyright (C) 2025 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

##  3. 安装 QEMU

可以直接从软件源安装，也可以从源码进行编译、安装。还是那句话，编译很耗时，能用现成的就用现成的。

查看一下软件源中的 QEMU 的版本，如果版本合适，直接从软件源安装即可：

```bash
$ apt-cache madison qemu
      qemu | 1:4.2-3ubuntu6.30 | http://cn.archive.ubuntu.com/ubuntu focal-updates/main amd64 Packages
      qemu | 1:4.2-3ubuntu6.30 | http://security.ubuntu.com/ubuntu focal-security/main amd64 Packages
      qemu | 1:4.2-3ubuntu6 | http://cn.archive.ubuntu.com/ubuntu focal/main amd64 Packages
$ sudo apt-get install qemu
```

不过我也不知道合适不合适，所以选择直接从源码编译。

先安装依赖：

```bash
sudo apt-get install libglib2.0-dev libpixman-1-dev
```

为 QEMU 安装目录：

```bash
sudo mkdir -p /opt/qemu
sudo chown $USER:$USER /opt/qemu
```

克隆源码：

```bash
git clone https://git.qemu.org/git/qemu.git
$ git log -n 1
commit 9e601684dc24a521bb1d23215a63e5c6e79ea0bb (HEAD -> master, tag: v10.1.0-rc0, origin/master, origin/HEAD)
Author: Stefan Hajnoczi <stefanha@redhat.com>
Date:   Tue Jul 22 15:48:48 2025 -0400

    Update version for the v10.1.0-rc0 release
    
    Signed-off-by: Stefan Hajnoczi <stefanha@redhat.com>
```

编译、安装：
```bash
cd qemu
./configure --prefix=/opt/qemu
make -j$(nproc)
make install
```

在 configure  的时候报错了，要求 Python >= 3.9，但我的操作系统是 Ubuntu 20.04，软件源中的 python 版本是 3.8.10，不满足要求，实际上不光是 Python，有对其他依赖也有版本要求，因此选择给 QEMU 降版本，挨个试大版本的最后一个小版本，最终切换到 v9.2.4 时可以顺利完成编译安装（但是 configure 时有些项是 NO，在这里没去管它）。

检出 9.2.4 版本的源码后重新开始配置、编译、安装：

```
git checkout v9.2.4
./configure --prefix=/opt/qemu
make -j$(nproc)
make install
```

结束后可以在 `/opt/qemu` 中看到编译好的工具链。

完成后将 QEMU 可执行文件路径加入 PATH 环境变量：

```bash
echo 'export PATH=$PATH:/opt/qemu/bin' >> ~/.bashrc
source ~/.bashrc
```

检查是否安装成功：

```bash
$ qemu-system-riscv64 --version
QEMU emulator version 8.2.9 (v8.2.9)
Copyright (c) 2003-2023 Fabrice Bellard and the QEMU Project developers
```

## 4. FreeRTOS

克隆 FreeRTOS 源码：

```bash
git clone https://github.com/FreeRTOS/FreeRTOS.git --recurse-submodules
```

所在节点：

```
commit 5cf13754a5547789e4f9a0a14f130a06642cc3f2 (HEAD -> main, origin/main, origin/HEAD)
```

编译示例代码：

```bash
cd FreeRTOS/Demo/RISC-V_RV32_QEMU_VIRT_GCC
make -C build/gcc DEBUG=1
```

使用 QEMU 执行：

```bash
~/Source/FreeRTOS/FreeRTOS/Demo/RISC-V_RV32_QEMU_VIRT_GCC$ qemu-system-riscv32 -nographic -machine virt -net none \
 -chardev stdio,id=con,mux=on -serial chardev:con \
 -mon chardev=con,mode=readline -bios none \
 -smp 4 -kernel ./build/gcc/output/RTOSDemo.elf
```

输出如下结果：
```
FreeRTOS Demo Start
FreeRTOS Demo SUCCESS: : 5037
FreeRTOS Demo ERROR: xAreStreamBufferTasksStillRunning() returned false : 10035
FreeRTOS Demo ERROR: xAreAbortDelayTestTasksStillRunning() returned false : 15036
FreeRTOS Demo ERROR: xAreBlockTimeTestTasksStillRunning() returned false : 20034
FreeRTOS Demo ERROR: xAreBlockTimeTestTasksStillRunning() returned false : 25034
FreeRTOS Demo ERROR: xAreBlockTimeTestTasksStillRunning() returned false : 30035
FreeRTOS Demo ERROR: xAreBlockTimeTestTasksStillRunning() returned false : 35035
FreeRTOS Demo ERROR: xAreBlockTimeTestTasksStillRunning() returned false : 40036
FreeRTOS Demo ERROR: xAreBlockTimeTestTasksStillRunning() returned false : 45038
FreeRTOS Demo ERROR: xAreBlockTimeTestTasksStillRunning() returned false : 50037
FreeRTOS Demo ERROR: xAreBlockTimeTestTasksStillRunning() returned false : 55038
ASSERT! Line 929, file ./../../../../Demo/Common/Minimal/TimerDemo.c
```

看起来有问题，正好，接下来开始调试。

## 5. 调试

我们利用 QEMU 带着的 GDB 服务器来调试程序，因此，在启动 QEMU 时，需要添加 `-s` 和 `-S` 选项。`-s` 选项是使能 GDB 服务器，默认监听 1234 端口，`-S` 选项是让 QEMU 加载可执行文件后暂停运行，等待 GDB 连接。

```bash
~/Source/FreeRTOS/FreeRTOS/Demo/RISC-V_RV32_QEMU_VIRT_GCC$ qemu-system-riscv32 -nographic -machine virt -net none \
 -chardev stdio,id=con,mux=on -serial chardev:con \
 -mon chardev=con,mode=readline -bios none \
 -smp 4 -kernel ./build/gcc/output/RTOSDemo.elf -s -S
```

使用 netstat 可以看到，可以看到 QEMU 在监听 1234 端口：

```
~$ netstat -tulnp | grep qemu
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:1234            0.0.0.0:*               LISTEN      178181/qemu-system- 
tcp6       0      0 :::1234                 :::*                    LISTEN      178181/qemu-system-
```

接下来就可以启动 GDB 来调试了。

### (1) 命令行中调试

打开另一个终端，进入 RISC-V_RV32_QEMU_VIRT_GCC 目录，使用 riscv64-unknown-elf-gdb 连接 QEMU 的 GDB 服务器来调试：

```bash
~/Source/FreeRTOS/FreeRTOS/Demo/RISC-V_RV32_QEMU_VIRT_GCC$ riscv64-unknown-elf-gdb ./build/gcc/output/RTOSDemo.elf -ex "target remote :1234"
GNU gdb (GDB) 16.3.90.20250610-git
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-pc-linux-gnu --target=riscv64-unknown-elf".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./build/gcc/output/RTOSDemo.elf...
Remote debugging using :1234
0x00001000 in ?? ()
```

列出源码：

```
(gdb) list
119	    {
120	        __asm__ volatile ( "csrw mtvec, %0" : : "r" ( freertos_risc_v_trap_handler ) );
121	    }
122	    #else
123	    {
124	        __asm__ volatile ( "csrw mtvec, %0" : : "r" ( ( uintptr_t ) freertos_vector_table | 0x1 ) );
125	    }
126	    #endif
127	
128	    /* The mainCREATE_SIMPLE_BLINKY_DEMO_ONLY setting is described at the top
(gdb) 
129	     * of this file. */
130	    #if ( mainCREATE_SIMPLE_BLINKY_DEMO_ONLY == 1 )
131	    {
132	        main_blinky();
133	    }
134	    #else
135	    {
136	        main_full();
137	    }
138	    #endif
(gdb)
```

在 136 行打断点：

```
(gdb) break main.c:136
Breakpoint 1 at 0x80000110: file ../../../../Demo/RISC-V_RV32_QEMU_VIRT_GCC/main.c, line 136.
```

继续运行：

```
(gdb) continue
Continuing.

Thread 1 hit Breakpoint 1, main () at ../../../../Demo/RISC-V_RV32_QEMU_VIRT_GCC/main.c:136
136	        main_full();
(gdb) 
```

可以看到，程序停在了断点的位置。

### (2) VSCode 中调试

先安装 native-Debug 插件。

添加调试配置，选择 “{} GDB: Connect to gdbserver”，`lunch.json` 文件中会自动添加如下内容：
```json
"configurations": [
        {
            "type": "gdb",
            "request": "attach",
            "name": "Attach to gdbserver",
            "executable": "./bin/executable",
            "target": ":2345",
            "remote": true,
            "cwd": "${workspaceRoot}",
            "valuesFormatting": "parseText"
        },
]
```

将其修改为：

```json
"configurations": [
        {
            "type": "gdb",
            "request": "attach",
            "name": "Attach to gdbserver",
            "executable": "${workspaceFolder}/build/gcc/output/RTOSDemo.elf",
            "target": ":1234",
            "remote": true,
            "cwd": "${workspaceRoot}",
            "gdbpath": "/opt/riscv/bin/riscv64-unknown-elf-gdb",
            "valuesFormatting": "parseText"
        },
]
```

重新启动 QEMU ，确保其停在开始状态以等待 GDB 的连接，然后就可以在 VSCode 中打断点、调试了。

## 6. 可能遇到的问题

编译时如果意外退出，并报如下错误，那么需要增加内存或者交换区域：

> collect2: fatal error: ld terminated with signal 9 [Killed]

VSCode 中，启动调试时如果遇到下述错误，检查 QEMU 及其 GDB 服务器是否在运行，并且关闭其他连接到该 GDB 服务器的进程：

> Failed to attach: Remote replied unexpectedly to 'vMustReplyEmpty': timeout (from target-select remote :1234)

## 参考

[1] [Setting up a development environment for RISCV-FreeRTOS on QEMU](https://embeddedinn.com/articles/tutorial/RISCV-FreeRTOS-QEMU/)

[2] [安装 RISC-V 交叉编译工具链](https://soc.ustc.edu.cn/CECS/lab0/riscv/)