---
layout: post
title:  "使用 libusb 发送控制传输"
date:   2021-11-16 20:45:00 +0800
categories: [Embedded, USB]
tags: [USB]
excerpt: 本文以通过控制传输读取 HUB 的描述符为例，阐述如何为 HUB 添加 API，顺带对 libusb 中与 Windows 有关的源码进行剖析，也可以用在 HUB 以外的 USB 设备上。
---

## 1. 背景

最近要为 USB HUB 开发一个测试工具，基本功能包括获取描述符、发送 USB 标准请求等，选择从 libusb-1.0.24 着手进行开发。

## 2. libusb 代码结构

总体看来，大致分为两层，第一层操作 `usbi_backend`，对外屏蔽操作系统的差异，下面还有一层后端，跟操作系统有关系，Windows 上可以选择 WinUSB 或 UsbDk。WinUSB 是 Windows 自带的驱动（`C:\Windows\System32\drivers\winusb.sys`），运行时库位为 `C:\Windows\System32\winusb.dll`，[官网](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/usbcon/winusb) 有详细的 API 使用文档。UsbDk 需要额外安装。

下图是大致阅读完代码后，整理出的一个结构图，可能会有因理解不到位造成的错误，仅供参考。

![libusb](/assets/img/2021-11-16-send_a_usb_control_transfer_to_hub_using_libusb_on_windows.assets/libusb.png)

### (1) Windows

Windows 上，最底层默认为 `winusb_backend`，可以在 `windows_init` 中看到：

```c
// By default, new contexts will use the WinUSB backend
priv->backend = &winusb_backend;
```

可以调用 `libusb_set_option` 函数切换后端，将其切换为 UsbDk。

```c
int API_EXPORTED libusb_set_option(libusb_context *ctx, enum libusb_option option, ...)
```

其实还可以选择 libusbK，不过使用的后端变量还是 `winusb_backend`，详细情况可以参考 `winusbx_init` 函数的实现。

### (2) Linux

Linux 上，`usbi_backend` 封装了 usbfs 的接口。

扫描设备时，通过预编译宏 `HAVE_LIBUDEV` 决定使用 udev 还是 usbfs，该宏看起来是由 `configure` 控制的。

## 3. 获取描述符

### (1) 设备描述符

- 使用 `usbdk_backend`
  - 在 `usbdk_device_init` 中，从 `usbdk_get_device_list` 传递进来的设备信息中复制设备描述符，后者调用 UsbDk 的接口 `UsbDk_GetDevicesList` 获取设备信息。
- 使用 `winusb_backend`(默认)
  - `init_device` 中，通过调用操作系统的接口 `DeviceIoControl` 获取设备信息，然后从获取的设备信息中拷贝设备描述符。

### (2) 配置描述符

`get_config_descriptor` 中调用 `usbi_backend` 的函数指针获取

- 使用 `usbdk_backend`
  - 最终通过 `UsbDk_GetConfigurationDescriptor` 获取配置描述符。
- 使用 `winusb_backend`(默认)
  - `winusb_get_config_descriptor` 中复制之前已经缓存的配置描述符（`init_device` 中缓存的）

### (3) 接口描述符

接口描述符是配置描述符内容的一部分，从配置描述符的结构体中取即可。

### (4) 端点描述符

是接口描述符内容的一部分，从接口描述符的结构体中取即可。

### (5)超高速端点伴侣描述符

调用 `libusb_get_ss_endpoint_companion_descriptor` 获取，该函数在端点描述符的基础上进行一系列偏移后得到超高速端点伴侣描述符。

### (6)获取 BOS 描述符

获取 BOS 描述符的接口与获取设备描述符的接口不同，需要先打开设备，获得设备句柄，然后通过控制传输获取。

### (7) 获取字符串描述符

和获取 BOS 描述符的操作类似，需要先打开设备，获得设备句柄，然后通过控制传输获取。

## 4. 如何获取 HUB 的描述符

不管选用哪种后端，都可以获取设备描述符、配置描述符、接口描述符、端点描述符、超高速端点伴侣描述符，这些描述符都是从操作系统缓存好的信息中拿出来的，无需额外发起控制传输。像 BOS 描述符、字符串描述符、HUB 类描述符都需要额外发起控制传输才能获取，而发起控制传输的必要条件是要获得设备句柄以及接口句柄。如前面讲的，`usb_api_backend` 中并未给 HUB 定义相关的 API。

尝试切换 `usbdk_backend` 后端：

```c
libusb_set_option(NULL, LIBUSB_OPTION_USE_USBDK);
```

提示不支持：

```
[ 0.002677] [000042c0] libusb: debug [windows_iocp_thread] I/O completion thread started
[ 0.002770] [000011a4] libusb: error [windows_set_option] UsbDk backend not available
```

需要根据这篇文章安装 [UsbDk](https://cgit.freedesktop.org/spice/win32/usbdk)：[https://github.com/libusb/libusb/wiki/Windows#Driver_Installation](https://github.com/libusb/libusb/wiki/Windows#Driver_Installation)。但是安装好后也有问题：在 64 位机器上使用最新版本（v1.00-22），访问 HUB 时 Windows 会崩溃，出现蓝屏错误。v1.00-17 和 v1.00-16 更离谱，一旦安装，USB 设备就无法用了（**不建议安装 UsbDk，风险有点大。多亏之前允许过远程连接，鼠标、键盘没法用的话，还可以通过远程连接登录上去，把 usbDk 卸载掉**）。 

如果选择 `winusb_backend` 后端，还需要在 `usb_api_backend` 为 HUB 添加对应的 API，可以仿照 `USB_API_WINUSBX` 的接口为 HUB 封装必要的接口。而且还需要将 HUB 的驱动更换为 WinUSB，才能通过 WinUSB 接口访问 HUB（默认的驱动是 USBHUB3.SYS，不对外开放，没有 API 使用说明），可以借助 [Zadig](https://zadig.akeo.ie/) 更换 HUB 的驱动。需要注意的是，将 HUB 的驱动切换成 WinUSB 后，虽然可以通过控制传输向 HUB 发送命令，但是 HUB 会失去 HUB 的功能，例如，下行口接上设备时，电脑并不能识别到这个设备。从抓包分析的结果来看，换用 WinUSB 驱动后，PC 仅会完成 HUB 的枚举过程，后续的端口上电、使能、复位流程都没有。

使用 Zadig 为 HUB 换驱动时，需要在 `Options` 菜单中勾选 `List All Devices`，并且取消对 `Ignore Hubs or Composite Parents` 的勾选，否则无法看到 HUB。

## 5. 获取设备句柄以及接口句柄

获取句柄的过程：调用 `usbi_backend` 的 `open` 函数指针，该指针指向 `windows_common.c` 中的 `windows_open` 函数，该函数内，会调用第二层后端的 `open` 函数。

### (1) winusb_backend

该后端使用 `usb_api_backend` 中的回调函数。`usb_api_backend` 定义了 5 组 API（在 `windows_winusb.c` 中）：

- USB_API_UNSUPPORTED
- USB_API_HUB
- USB_API_COMPOSITE
- USB_API_WINUSBX
- USB_API_HID

只有后三种有 `open` 函数，即前两种不支持获取设备句柄。

使用 `winusb_backend` 后端时，会提示不支持 HUB 的 `open` 操作：

```
Dev (bus 1, device 4): 05E3 - 0610 speed: 480M						// 2.0 HUB
[ 0.018514] [00003c5c] libusb: debug [libusb_open] open 1.4
[ 0.018586] [00003c5c] libusb: debug [winusb_open] unsupported API call for 'open' (unrecognized device driver)
[ 0.018676] [00003c5c] libusb: debug [libusb_open] open 1.4 returns -12
[ 0.018751] [00003c5c] libusb: debug [libusb_get_device_descriptor]
Dev (bus 1, device 3): 05E3 - 0626 speed: 5G						// 3.0 HUB
[ 0.018890] [00003c5c] libusb: debug [libusb_open] open 1.3
[ 0.018961] [00003c5c] libusb: debug [winusb_open] unsupported API call for 'open' (unrecognized device driver)
[ 0.019051] [00003c5c] libusb: debug [libusb_open] open 1.3 returns -12
[ 0.019125] [00003c5c] libusb: debug [libusb_get_device_descriptor]
Dev (bus 1, device 0): 8086 - A3AF speed: 5G						// 3.0 根集线器
[ 0.019265] [00003c5c] libusb: debug [libusb_open] open 1.0
[ 0.019334] [00003c5c] libusb: debug [winusb_open] unsupported API call for 'open' (unrecognized device driver)
[ 0.019427] [00003c5c] libusb: debug [libusb_open] open 1.0 returns -12
[ 0.019504] [00003c5c] libusb: debug [libusb_get_device_descriptor]
```

正如前文所述，需要给 HUB 添加 API，获取到设备句柄以及接口句柄，涉及三个函数，初始化、打开、关闭，对应于 `windows_usb_api_backend` 的如下函数指针：

```
bool (*init)(struct libusb_context *ctx);
void (*exit)(void);
int (*open)(int sub_api, struct libusb_device_handle *dev_handle);
```

`init` 函数要完成加载 `WinUSB` 运行时库，查找并储存相关函数地址的工作，可以参照 `winusbx_init` 编写。

`exit` 函数要完成清理工作，可以参照 `winusbx_exit` 编写。

`open` 函数比较重要，我们要在该函数中获取到 HUB 的设备句柄和接口句柄，并且，还要建立接口句柄和后文提到的 I/O completion port 的关联关系。可以参照 `winusbx_open` 以及 `windows_open` 编写。

需要注意，HUB 的驱动要和使用的后端匹配，比如，使用 WinUSB，那么 HUB 的驱动就要改成 WinUSB，使用 libusbK，那么 HUB 的驱动就要改成 libusbK，否则，会出现接口句柄获取失败或控制传输下发失败的问题（函数会返回成功，但是实际上没有发出去）的情况。

### (2) usbdk_backend

通过 `usbdk_helper` 获取设备句柄，`usbdk_helper` 的回调函数地址都是从系统的 `UsbDkHelper.dll` 动态库中获取的。

## 6. 下发 USB 请求

USB 请求是通过控制传输下发的。`windows_usb_api_backend` 结构体中，与控制传输相关的函数指针有三个：

```
int (*submit_control_transfer)(int sub_api, struct usbi_transfer *itransfer);
int (*cancel_transfer)(int sub_api, struct usbi_transfer *itransfer);
enum libusb_transfer_status (*copy_transfer_data)(int sub_api, struct usbi_transfer *itransfer, DWORD length);
```

`submit_control_transfer` 发起控制传输，可以仿照 `winusbx_submit_control_transfer` 编写。

`cancel_transfer` 取消控制传输，可以仿照 `winusbx_cancel_transfer` 编写。

`copy_transfer_data` 用于获取 IN 传输的结果，可以仿照 `winusbx_copy_transfer_data` 编写。

### (1) 标准请求

标准请求可以通过 `libusb_control_transfer` 下发，使用方法可以参考 `testlibusb.c` 示例，追踪获取字符串描述符的函数 `libusb_get_string_descriptor_ascii`，该函数会调用 `libusb_control_transfer`。

### (2) 供应商指令

供应商指令也是通过控制传输下发的，使用方法可以参考源码中的示例 `ezusb.c`。

## 7. 事件处理

本节要解决的问题：分析控制传输相关的事件源于哪里？处理流程是什么样的？

### (1) 事件源

时间源是在 `usbi_io_init` 中设置的。

```
usbi_io_init
-> usbi_create_event
   -> CreateEvent	# 操作系统接口
-> usbi_add_event_source # 将事件源加入事件源链表
   -> 将事件源加入事件源链表 ctx->event_sources（事件源的数据为 USBI_EVENT_POLL_EVENTS，值为 0）
   -> usbi_event_source_notification
      -> ctx->event_flags |= USBI_EVENT_EVENT_SOURCES_MODIFIED;
      -> usbi_signal_event	# 如果尚未设置 event_flags 为 0 则
         -> SetEvent	# 将事件对象设置为 signaled 状态，目的应该是处理上面的 USBI_EVENT_EVENT_SOURCES_MODIFIED 事件。
```

这个地方其实仅仅只是创建了事件对象，还没有具体的意义。

### (2) 等待

在 `libusb_control_transfer` 函数中发起控制传输后，程序会进入等待传输完成流程：

````c
sync_transfer_wait_for_completion(transfer)
````

该函数中，会以死循环的形式等待传输完成标志 `transfer->user_data` 变为非 0 值。

有如下调用链：

```
sync_transfer_wait_for_completion
-> libusb_handle_events_completed
   -> libusb_handle_events_completed # 设置超时时间（60s）
      -> libusb_handle_events_timeout_completed
         -> handle_events
            -> usbi_alloc_event_data # 处理 USBI_EVENT_EVENT_SOURCES_MODIFIED 事件
            -> usbi_wait_for_events	# 等待事件，设置事件触发标志
               -> WaitForMultipleObjects # 操作系统的接口，等待的对象是由 usbi_alloc_event_data 中分配的句柄来指定的
            -> handle_event_trigger # 根据 ctx->event_flags 处理不同的事件
               -> usbi_backend.handle_transfer_completion	# 指向 windows_handle_transfer_completion
                  -> GetOverlappedResult # 操作系统的接口，获取 overlapped operation 的结果
                  -> backend->copy_transfer_data 
```

`WaitForMultipleObjects` 是操作系统的接口，参见 [WaitForMultipleObjects function (synchapi.h)](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects)，等待实际上就是在这个位置进行的。

`GetOverlappedResult` 是操作系统的接口，获取针对某个文件（通过入参指定）进行 overlapped operation 的结果，参见 [GetOverlappedResult function (ioapiset.h)](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getoverlappedresult)。这里的文件句柄是 HUB 设备对应的句柄。

### (3) 触发

通过上面的调用链可以看到，event_data 所需的内存是在 `usbi_alloc_event_data` 中分配的，那么，`usbi_alloc_event_data` 中的事件和要等待的控制传输完成事件是如何关联的，换句话说，就是事件是如何触发的？

实际上，`usbi_alloc_event_data` 不涉及设定关联关系，仅用来处理 `USBI_EVENT_EVENT_SOURCES_MODIFIED` 事件，根据事件源列表中的事件源数量（最多限制为 2 个，一个为控制传输完成事件，另一个为定时器超时时间），为 `ctx->event_data` 分配内存空间。

事件触发有两种方式，一种是主动强行触发，另一种是等待操作系统的信号。

关联关系是通过事件对象的使用方式建立的。从微软网站上的文档来看，事件的使用方式是这样的：A 函数监听该事件，要触发该事件的话，需要在另外的地方将事件对象设置为 signaled 状态，这样的话，监听该事件的 A 函数会被释放，完成后续的处理。

设置事件对象为 signaled 状态的地方有 6 处：

1. `libusb_close`
2. `usbi_signal_transfer_completion`
3. `libusb_interrupt_event_handler`
4. `usbi_event_source_notification`
5. `usbi_hotplug_notification`
6. `libusb_hotplug_deregister_callback`

看起来仅有 2 和 4 跟这里的分析有关系，跟控制传输完成事件相关的仅有 2：

`usbi_signal_transfer_completion` 有两处调用：

- `windows_force_sync_completion`
- `windows_iocp_thread`

`usbi_event_source_notification` 有两处调用：

- `usbi_add_event_source`
- `usbi_remove_event_source`

`usbi_signal_transfer_completion` 的两处调用，对应着前面说的两种触发方式。

一是发起控制传输后，通过 `windows_force_sync_completion` 强行触发，有如下调用链：

```
windows_force_sync_completion
-> usbi_signal_transfer_completion # 设置 ctx 中的事件标志，设置为已完成
   -> usbi_signal_event
      -> SetEvent
```

这个 `SetEvent` 是操作系统的接口，作用就是前面讲的将指定的事件对象设置为 signaled 状态。详见 [SetEvent function (synchapi.h)](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setevent)。

简单来说，强行触发是将对应的事件对象设置为 signaled 状态，这样的话，等待该事件对象的函数就可以被触发了。`winusbx_submit_control_transfer` 函数中，发起 `Set_Configuration` 请求后，会以强行触发的方式通知等待函数控制传输已完成。

另外一种呢？`usbi_signal_transfer_completion` 被调用的另一处地方在 `windows_iocp_thread` 函数内。

`libusb_init` 调用的 `windows_init` 函数会创建一个等待 I/O 完成的线程，该线程的入口函数就是 `windows_iocp_thread`。

`windows_iocp_thread` 函数在死循环中有如下调用链：

```
windows_iocp_thread
-> GetQueuedCompletionStatus # 会一直阻塞在这个地方，直到有完成或者退出时才会往下进行
   -> usbi_signal_transfer_completion
```

这种就是第二种情况，即等待操作系统完成事件的情况。

```
GetQueuedCompletionStatus(iocp, &num_bytes, &completion_key, &overlapped, INFINITE)
```

iocp 是 `windows_init` 中通过 `CreateIoCompletionPort` 创建的 I/O completion port，创建的时候未和任何文件句柄关联，存放在 ctx 的 `priv->completion_port` 中。

在 `windows_winusb.c` 文件的 `windows_open` 函数中，ctx 的 `priv->completion_port` 和设备的接口的 path 关联到了一起。前文提到事件对象和控制传输完成这一物理事件的对应关系就是在这个地方建立的。

### (4) 小结

libusb 使用操作系统提供的事件对象来获知控制传输是否完成。等待方需要通过 `WaitForMultipleObjects` 等待事件。触发该事件的方式有两种，一种是强行触发，即程序主动将事件对象设置为 signaled 状态，另一种是通过 `GetQueuedCompletionStatus` 等待 overlapped 操作完成，完成后将事件对象设置为 signaled 状态。事件对象的状态变为 signaled 状态时，等待该事件的函数会被释放。

## 8. 调试

libusb 有非常丰富的调试信息，打开调试输出的方式有两种，一种是调用 `libusb_set_option` 函数，设置输出级别；另一种是设置环境变量 `LIBUSB_DEBUG` 来设置输出级别。

例如，cmd 上：

```
set LIBUSB_DEBUG=4
```