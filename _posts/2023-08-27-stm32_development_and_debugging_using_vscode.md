---
layout: post
title:  "VSCode STM32 开发环境配置"
date:   2023-08-27 21:44:00 +0800
categories: [Embedded, MCU]
tags: [MCU]
excerpt: 分享使用 VSCode 进行 STM32 开发的过程。
---

## 1. VSCode + Keil

在 Keil 里面编码不太方便，如果直接用 vscode 打开源码目录，会有很多告警（使用 C/C++ 扩展），这些告警会对跳转、自动补全产生影响。告警的原因是 vscode 自动生成的 C/C++ 扩展配置是自动检测电脑编译链之后的结果，例如，`windows-msvc-x64`，这样的话，vscode 找不到正确的头文件，进而，也找不到 ARMCC 相关的宏定义，于是导致上面的问题发生。

可以借用 Keil Assistant 扩展生成 C/C++ 扩展配置文件，解决告警的问题。

进行如下配置：

- 安装 Keil Assistant 扩展。

- 打开 Keil Assistant 配置页面，设置好 `UV4.exe` 的路径。

- 安装好 Keil Assistant 插件后，vscode 资源管理器的下方会出现 `KEIL UVISION PROJECT`，光标移动到上面上，旁边会出现加号图标，点击该图标选择对应的 Keil 工程。

- 根据提示，切换工作空间（打开的目录为 Keil 工程文件所在的目录）。
- 使用 Keil 工程文件所在目录下自动生成的 `.vscode/c_cpp_properties.json` 文件替换原工作空间（代码所在目录）中的 `.vscode/c_cpp_properties.json` 文件。
- 重新打开源代码所在目录。

尽管使用 Keil Assistant 打开工程文件后，可以直接在 vscode 中进行编译、烧录，但是由于此时的工作空间目录是 Keil 工程文件所在目录，快捷键 `Ctrl+P`、`Ctrl+Shift+P`、`Ctrl+Shift+O` 等无法搜索不在该目录内的源码文件，不方便。

## 2. VSCode + GNU Arm Embedded Toolchain + OpenOCD

在这套方案下，编码、编译、调试都可以在 VSCode 中完成。

### 2.1 安装开发环境

#### (1) 安装 Windows Terminal 和 PowerShell 

在 Windows 上进行命令行操作时，强烈推荐使用 Windows Terminal 和 PowerShell 的组合，例如使用 wsl、Rust 或者手动调用 gcc 的时候，比系统自带的 CMD 和 Windows PowerShell 工具好用很多。

可以快速浏览官网介绍，了解 Windows Terminal 的特性：[什么是 Windows 终端？](https://learn.microsoft.com/zh-cn/windows/terminal/)

Windows Terminal 在 Microsoft Store 中安装，PowerShell 要从 GitHub 仓库下载：

```
https://github.com/PowerShell/PowerShell/releases
```

安装完成后，在 Windows Terminal 中新建一个 PowerShell 的配置文件，并将其设置为 Windows Terminal 的默认配置文件，这样每次启动 Windows Terminal 时，默认打开的就是 PowerShell 了。

#### (2) 安装 arm-none-eabi 工具链

从 ARM 官网下载工具链，例如 [gcc-arm-none-eabi-10.3-2021.10-win32.exe](https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-rm/10.3-2021.10/gcc-arm-none-eabi-10.3-2021.10-win32.exe)，下载完成后安装，安装完成后将工具链路径添加到环境变量中，例如：

```
D:\Program Files (x86)\GNU Arm Embedded Toolchain\10 2021.10\bin
```

之后在 `PowerShell` 中测试一下工具链是否能被调用：

```
gcm arm-none-eabi-gcc
gcm arm-none-eabi-gdb

arm-none-eabi-gcc.exe -v
arm-none-eabi-gdb.exe -v
```

#### (3) 安装 make

借助 [MSYS2](https://www.msys2.org/) 安装 make 工具（按照官网首页的教程安装 `mingw-w64 GCC` 即可），安装完成之后，将 `make` 所在目录添加到环境变量中，例如：

```
C:\msys64\usr\bin
```

之后在 `PowerShell` 中测试一下 `make` 是否能被调用：

```
gcm make

make -v
```

#### (4) VSCode 安装 Cortex-Debug 插件

搜索安装即可。

#### (5) 安装 OpenOCD

打开刚才安装好的 MSYS2 命令行，安装 OpenOCD：

```
pacman -S --needed mingw-w64-x86_64-openocd
```

如果遇到安装包签名问题，可以尝试执行如下命令解决：

```
rm -rf /etc/packman.d/gnupg
pacman-key --init
pacman -S msys2-keyring
pacman-key --populate msys2
pacman -Syu
```

`OpenCDC` 会被安装到 MSYS2 的 `usr\bin` 目录下（和 `make` 目录相同）。同样，在命令行中测试一下是否可用：

```
openocd -v
```

### 2.2 生成示例工程并在命令行中编译调试

使用 `STM32CubeMX` 来生成 `make` 工程，配置好外设、时钟之后，在 `Project Manger` -> `Project` -> `Toolchain/IDE` 中选择 `Makefile`，生成 `make` 工程，例如为 `NUCLEO-L452RE` 生成的名为 `base` 的工程：

```
├─build
├─Core
│  ├─Inc
│  └─Src
├─Drivers
│   ├─CMSIS
│   └─STM32L4xx_HAL_Driver
├─Makefile
├─startup_stm32l452xx.s
├─STM32L452RETx_FLASH.ld
└─base.ioc
```

然后打开 Windows Terminal，进入工程目录，执行 `make` 编译工程：

```
make
```

编译完成后，连接 `NUCLEO-L452RE` 开发板，在 Windows Terminal 中启动 `OpenOCD` 进行调试：

```
base> openocd -f interface/stlink-V2.cfg -f target/stm32l4x.cfg
Open On-Chip Debugger 0.12.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
WARNING: interface/stlink-v2.cfg is deprecated, please switch to interface/stlink.cfg
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : clock speed 500 kHz
Info : STLINK V2J38M27 (API v2) VID:PID 0483:374B
Info : Target voltage: 3.263559
Info : [stm32l4x.cpu] Cortex-M4 r0p1 processor detected
Info : [stm32l4x.cpu] target has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32l4x.cpu on 3333
Info : Listening on port 3333 for gdb connections
[stm32l4x.cpu] halted due to breakpoint, current mode: Thread
xPSR: 0x01000000 pc: 0x08000874 msp: 0x20027ff8
```

再打开一个中断，启动 gdb：

```
base> arm-none-eabi-gdb.exe -q .\build\base.elf
D:\Program Files (x86)\GNU Arm Embedded Toolchain\10 2021.10\bin\arm-none-eabi-gdb.exe: warning: Couldn't determine a path for the index cache directory.
Reading symbols from .\build\base.elf...
(gdb)
```

连接 `OpenOCD`：

```
(gdb) target remote: 3333
Remote debugging using : 3333
main () at Core/Src/main.c:100
100       while (1)
(gdb)
```

`OpenOCD` 会有如下输出：

```
Info : accepting 'gdb' connection on tcp/3333
Info : device idcode = 0x20016462 (STM32L45/L46xx - Rev Y : 0x2001)
Info : RDP level 0 (0xAA)
Info : flash size = 512 KiB
Info : flash mode : single-bank
Info : device idcode = 0x20016462 (STM32L45/L46xx - Rev Y : 0x2001)
Info : RDP level 0 (0xAA)
Info : OTP size is 1024 bytes, base address is 0x1fff7000
Warn : Prefer GDB command "target extended-remote :3333" instead of "target remote :3333"
```

在 gdb 中增加断点。首先，使用 `list main` 命令查看 `main` 函数源码：

```
(gdb) list main
63      /**
64        * @brief  The application entry point.
65        * @retval int
66        */
67      int main(void)
68      {
69        /* USER CODE BEGIN 1 */
70 
71        /* USER CODE END 1 */
72
(gdb)
73        /* MCU Configuration--------------------------------------------------------*/
74
75        /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
76        HAL_Init();
77
78        /* USER CODE BEGIN Init */
79
80        /* USER CODE END Init */
81
82        /* Configure the system clock */
(gdb)
83        SystemClock_Config();
84
85        /* USER CODE BEGIN SysInit */
86
87        /* USER CODE END SysInit */
88
89        /* Initialize all configured peripherals */
90        MX_GPIO_Init();
91        MX_USART2_UART_Init();
```

然后在添加断点，比如在 `main` 函数入口以及 `MX_GPIO_Init()` 所在的第 90 行添加断点：

``` 
(gdb) break main
Breakpoint 1 at 0x800085a: file Core/Src/main.c, line 76.
Note: automatically using hardware breakpoints for read-only addresses.
(gdb) break 90
Breakpoint 2 at 0x8000862: file Core/Src/main.c, line 90.
(gdb)
```

查看已经添加的断点：

```
(gdb) info break
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x0800085a in main at Core/Src/main.c:76
2       breakpoint     keep y   0x08000862 in main at Core/Src/main.c:90
(gdb)
```

装载：

```
(gdb) load
Loading section .isr_vector, size 0x198 lma 0x8000000
Loading section .text, size 0x3e68 lma 0x80001c0
Loading section .rodata, size 0x60 lma 0x8004028
Loading section .ARM, size 0x8 lma 0x8004088
Loading section .init_array, size 0x8 lma 0x8004090
Loading section .fini_array, size 0x8 lma 0x8004098
Loading section .data, size 0x850 lma 0x80040a0
Start address 0x08002204, load size 18632
Transfer rate: 22 KB/sec, 2661 bytes/write.
(gdb)
```

运行：

```
(gdb) c
Continuing.

Breakpoint 1, main () at Core/Src/main.c:76
76        HAL_Init();
(gdb)
```

遇到了第一个断点，停住了。继续执行：

```
(gdb) c
Continuing.
halted: PC: 0x08000fc0

Breakpoint 2, main () at Core/Src/main.c:90
90        MX_GPIO_Init();
(gdb)
```

遇到了第二个断点，停住了。

截止到现在，已经可以在终端中编译、调试 STM32 程序了。接下来看一下如何在 VS Code 中编译、调试程序。

### 2.3 配置 VS Code 并在 VS Code 中开发调试

使用 VSCode 打开刚才生成的工程目录，新增 `make` 任务（`.vscode/tasks.json`）：

```
{

    "version": "2.0.0",
    "tasks": [
        {
            "label": "make",
            "type": "shell",
            "command": "make",
            "group": "build",
            "problemMatcher": [],
            "detail": "target build task"
        }
    ]
}
```

配置调试参数（`.vscode/launch.json`）：

```
{
    "version": "0.2.0",
    "configurations": [
        {
            /* Configuration for the NUCLEO-L452RE board */
            "type": "cortex-debug",
            "request": "launch",
            "name": "Debug (OpenOCD)",
            "servertype": "openocd",
            "cwd": "${workspaceRoot}",
            "preLaunchTask": "make",
            "runToEntryPoint": "main",
            "executable": "./build/base.elf",
            "device": "STM32L452RE",
            "configFiles": [
                "interface/stlink.cfg",
                "target/stm32l4x.cfg"
            ]
        }
    ]
}
```

完整目录如下：

```
├─.vscode
│  ├─launch.json
│  └─tasks.json
├─build
├─Core
│  ├─Inc
│  └─Src
├─Drivers
│  ├─CMSIS
│  └─STM32L4xx_HAL_Driver
├─Makefile
├─startup_stm32l452xx.s
├─STM32L452RETx_FLASH.ld
└─base.ioc
```

关闭之前打开的 `OpenOCD` 和 `gdb` 终端（否则 VS Code 中运行调试会失败），在 VS Code 中点击 `运行和调试`，即可编译、调试程序。

### 2.4 使用“半托管”特性

借助 `semihosting`，可以将一些调试信息输出到 `OpenOCD` 的控制台上。需要修改 Makefile，连接 `rdimon` 库，并在 `main` 函数中调用 `initialise_monitor_handles()`：

```
# LIBS = -lc -lm -lnosys
LIBS = -lc -lm -lrdimon

# LDFLAGS = $(MCU) -specs=nano.specs -T$(LDSCRIPT) $(LIBDIR) $(LIBS) -Wl,-Map=$(BUILD_DIR)/$(TARGET).map,--cref -Wl,--gc-sections
LDFLAGS = $(MCU) --specs=rdimon.specs -T$(LDSCRIPT) $(LIBDIR) $(LIBS) -Wl,-Map=$(BUILD_DIR)/$(TARGET).map,--cref -Wl,--gc-sections
```

代码：

```
/* ... */
#include <stdio.h>
/* ... */
int main(void)
{
  	/* ... */
    initialise_monitor_handles();
    /* ... */
    printf("Hello World.\n");
    /* ... */
}
```

清除编译工程，重新编译并调试：可以在 `OpenOCD` 的控制台输出中看到打印信息：

```
make clean
make
```

在 `gdb` 的控制台使能 `semihosting`（输入 `monitor arm semihosting enable`）：

```
(gdb) target remote: 3333
(gdb) monitor arm semihosting enable
(gdb) list main
(gdb)
93
94          printf("Hello World.\n");
95
(gdb) break 94
(gdb) load
Loading section .isr_vector, size 0x198 lma 0x8000000
Loading section .text, size 0x41d8 lma 0x80001c0
Loading section .rodata, size 0x80 lma 0x8004398
Loading section .ARM, size 0x8 lma 0x8004418
Loading section .init_array, size 0x8 lma 0x8004420
Loading section .fini_array, size 0x8 lma 0x8004428
Loading section .data, size 0x858 lma 0x8004430
Start address 0x08002208, load size 19552
Transfer rate: 23 KB/sec, 2444 bytes/write.
(gdb) c
Continuing.

Breakpoint 1, main () at Core/Src/main.c:94
94          printf("Hello World.\n");
(gdb)c
```

`OpenOCD` 控制台有如下输出：

```
[stm32l4x.cpu] halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x08002208 msp: 0x20028000
Info : halted: PC: 0x08000870
Hello World.
```

## 参考

[1] [手把手教你 VSCode搭建STM32开发环境](https://zhuanlan.zhihu.com/p/385497405)

[2] [ARM Semihosting - native but slow Debugging](https://www.codeinsideout.com/blog/stm32/semihosting/#debugging)

[3] [Blink - say Hello to the World](https://www.codeinsideout.com/blog/stm32/blink/#step-5-check-the-compilation-settings)
