---
layout: post
title:  "在 Windows 上搭建 RISC-V 和 FreeROTS 的 QEMU 开发调试环境"
date:   2025-07-28 22:38:00 +0800
categories: [Embedded]
tags: [FreeRTOS, RISC-V]
excerpt: 这篇文章描述了如何在 Windows 上搭建 RISC-V 和 FreeRTOS 的 QEMU 开发调试环境。
---

## 1. 背景

上篇博文介绍了如何在 Ubuntu 20.04.6 LTS 上从源码编译 RISC-V 工具链以及 QEMU，建立模拟开发环境，过程很繁琐，而且在虚拟机中使用 VSCode 调试程序也很难受。所以今天尝试了一下直接 Windows 上使用 MSYS2 上建立这样的开发环境。

## 2. 安装 MSYS2

这个步骤很简单，从官网下载安装包安装即可。如果电脑上已经有了，建议先卸载掉，重新下载最新版本，否则接下来安装 RISC-V 工具链和 QEMU 时可能会遇到一大堆麻烦的依赖问题，主要原因是官方在不断更新软件包，如果要安装的软件包，其依赖项在电脑上已经安装了，但电脑上的依赖项版本比较旧，且被其他软件包依赖时，就会报错。与其解决这些依赖，不如直接从新开始安装，至少比从源码编译要快。

MSYS2 提供了多种开发环境，这里使用 MSYS2 MINGW64。

## 3. 安装软件包

安装 RISC-V 工具链：
```
pacman -S mingw-w64-x86_64-riscv64-unknown-elf-gcc
```

这个工具链里面是没有 riscv64-unknown-elf-gdb 的，且 MSYS2 也没有提供 riscv64-unknown-elf-gdb，不过可以用 gdb-multiarch 来调试，所以安装 gdb-multiarch 即可：

```
pacman -S mingw-w64-x86_64-gdb-multiarch
```

安装 make（实际使用时，make 的名称是 mingw32-make）：

```
pacman -S mingw-w64-x86_64-make 
```

安装 QEMU：

```
pacman -S mingw-w64-x86_64-qemu
```

## 4. FreeRTOS

上篇博文中克隆了 FreeRTOS 的全部源码，体积很大：

```bash
git clone https://github.com/FreeRTOS/FreeRTOS.git --recurse-submodules
```

所在节点：

```
commit 5cf13754a5547789e4f9a0a14f130a06642cc3f2 (HEAD -> main, origin/main, origin/HEAD)
```

实际上本文只需要这些，可以根据需要仅下载这部分代码：

```
FreeRTOS
 ├── Demo
 │   ├── Common
 │   └── RISC-V_RV32_QEMU_VIRT_GCC
 └── Source
```

## 5. 编译及运行

编译和运行都需要在 MSYS2 MINGW64 的环境中进行。

编译：

```
cd FreeRTOS/Demo/RISC-V_RV32_QEMU_VIRT_GCC
mingw32-make -C build/gcc DEBUG=1
```

在同样的路径下，使用 QEMU 运行：

```
qemu-system-riscv32 -nographic -machine virt -net none \
 -chardev stdio,id=con,mux=on -serial chardev:con \
 -mon chardev=con,mode=readline -bios none \
 -smp 4 -kernel ./build/gcc/output/RTOSDemo.elf
```

预期输出类似下面的结果：
```
FreeRTOS Demo Start
FreeRTOS Demo SUCCESS: : 5033
FreeRTOS Demo SUCCESS: : 10033
```

## 6. 调试

我们利用 QEMU 带着的 GDB 服务器来调试程序，因此，在启动 QEMU 时，需要添加 `-s` 和 `-S` 选项。`-s` 选项是使能 GDB 服务器，默认监听 1234 端口，`-S` 选项是让 QEMU 加载可执行文件后暂停运行，等待 GDB 连接。

```bash
qemu-system-riscv32 -nographic -machine virt -net none \
 -chardev stdio,id=con,mux=on -serial chardev:con \
 -mon chardev=con,mode=readline -bios none \
 -smp 4 -kernel ./build/gcc/output/RTOSDemo.elf -s -S
```

接下来就可以启动 GDB 来调试了。

### (1) 命令行中调试

打开另一个 MSYS2 MINGW64 终端，进入 RISC-V_RV32_QEMU_VIRT_GCC 目录，使用 gdb-multiarch 连接 QEMU 的 GDB 服务器来调试：

```bash
$ gdb-multiarch.exe ./build/gcc/output/RTOSDemo.elf -ex "target remote :1234" -ex "riscv:rv32"
GNU gdb (GDB) 16.3
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-w64-mingw32".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...

warning: A handler for the OS ABI "Windows" is not built into this configuration
of GDB.  Attempting to continue with the default riscv:rv32 settings.

Reading symbols from ./Demo/RISC-V_RV32_QEMU_VIRT_GCC/build/gcc/output/RTOSDemo.elf...
Remote debugging using :1234
warning: A handler for the OS ABI "Windows" is not built into this configuration
of GDB.  Attempting to continue with the default riscv:rv32 settings.

0x00001000 in ?? ()
Undefined command: "riscv".  Try "help".
(gdb)
```

列出源码：

```
(gdb) list
119         {
120             __asm__ volatile ( "csrw mtvec, %0" : : "r" ( freertos_risc_v_trap_handler ) );
121         }
122         #else
123         {
124             __asm__ volatile ( "csrw mtvec, %0" : : "r" ( ( uintptr_t ) freertos_vector_table | 0x1 ) );
125         }
126         #endif
127
128         /* The mainCREATE_SIMPLE_BLINKY_DEMO_ONLY setting is described at the top
(gdb)
129          * of this file. */
130         #if ( mainCREATE_SIMPLE_BLINKY_DEMO_ONLY == 1 )
131         {
132             main_blinky();
133         }
134         #else
135         {
136             main_full();
137         }
138         #endif
(gdb)
```

在 136 行打断点：

```
(gdb) break main.c:136
Breakpoint 1 at 0x80000110: file ../../../../Demo/RISC-V_RV32_QEMU_VIRT_GCC/main.c, line 136.
(gdb)
```

继续运行：

```
(gdb) continue
Continuing.

Thread 1 hit Breakpoint 1, main () at ../../../../Demo/RISC-V_RV32_QEMU_VIRT_GCC/main.c:136
136             main_full();
```

可以看到，程序停在了断点的位置。

### (2) VSCode 中调试

先安装 native-Debug 插件。

打开 FreeRTOS 目录，即下述结构中的顶层目录：

```
FreeRTOS
 ├── Demo
 │   ├── Common
 │   └── RISC-V_RV32_QEMU_VIRT_GCC
 └── Source
```

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

将其修改为（注意替换 `gbdpath`）：

```json
"configurations": [
        {
            "type": "gdb",
            "request": "attach",
            "name": "Attach to gdbserver",
            "executable": "${workspaceFolder}/Demo/RISC-V_RV32_QEMU_VIRT_GCC/build/gcc/output/RTOSDemo.elf",
            "target": ":1234",
            "remote": true,
            "cwd": "${workspaceRoot}",
            "gdbpath": "D:\\msys64\\mingw64\\bin\\gdb-multiarch.exe",
            "valuesFormatting": "parseText",
        },
]
```

重新启动 QEMU ，确保其停在开始状态以等待 GDB 的连接，然后就可以在 VSCode 中打断点、调试了。