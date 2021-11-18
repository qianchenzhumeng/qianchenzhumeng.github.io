---
layout: post
title:  "Rust 笔记"
date:   2021-07-23 20:49:00 +0800
categories: [Rust]
tags: [Rust]
excerpt: 记录、总结使用 Rust 的过程中遇到的问题。
---

## 1. 设置代理

Windows 命令行代理设置

```bat
set http_proxy=http://example.com:80
set https_proxy=http://example.com:80
:: 查看
set http_proxy
set https_proxy
```

Windows PowerShell 代理设置

```powershell
$env:http_proxy="http://example.com:80"
$env:https_proxy="http://example.com:80"
#　查看
ls env:http_proxy
ls env:https_proxy
```

Linux 终端代理设置：

```bash
export http_proxy=example.com:80
export https_proxy=example.com:80
#　查看
echo $http_proxy
echo $https_proxy
```

也可以修改 cargo 的配置文件 `~/.cargo/config`：

```toml
[http]
proxy = "http://example.com:80"

[https]
proxy = "http://example.com:80"
```

## 2. 安装

### 2.1 从源码编译安装

以 CORTEX-A53 大端 CPU 为例:

- CPU: CORTEX-A53(ARMv8)
- toolchain: armeb-unknown-linux-gnueabi-
- endian: big
- 用户态程序是 32 位的

步骤：

（1） 添加目标板文件

仿照 `src/librustc_target/spec/arm_unknown_linux_gnueabi.rs` 创建目标板的描述文件（修改 max_atomic_width，target_endian，data_layout，features）

```
$ cat src/librustc_target/spec/armeb_unknown_linux_gnueabi.rs
use crate::spec::{LinkerFlavor, Target, TargetOptions, TargetResult};

pub fn target() -> TargetResult {
    let mut base = super::linux_base::opts();
    base.max_atomic_width = Some(32);
    Ok(Target {
        llvm_target: "armeb-unknown-linux-gnueabi".to_string(),
        target_endian: "big".to_string(),
        target_pointer_width: "32".to_string(),
        target_c_int_width: "32".to_string(),
        data_layout: "E-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64".to_string(),
        arch: "arm".to_string(),
        target_os: "linux".to_string(),
        target_env: "gnu".to_string(),
        target_vendor: "unknown".to_string(),
        linker_flavor: LinkerFlavor::Gcc,

        options: TargetOptions {
            features: "+strict-align,+v8".to_string(),
            abi_blacklist: super::arm_base::abi_blacklist(),
            target_mcount: "\u{1}__gnu_mcount_nc".to_string(),
            ..base
        },
    })
}
```

（2）添加 target triple：

```
$ git diff src/librustc_target/spec/mod.rs
diff --git a/src/librustc_target/spec/mod.rs b/src/librustc_target/spec/mod.rs
index 8f3097a..bd4cb84 100644
--- a/src/librustc_target/spec/mod.rs
+++ b/src/librustc_target/spec/mod.rs
@@ -348,6 +348,9 @@ supported_targets! {
     ("sparc-unknown-linux-gnu", sparc_unknown_linux_gnu),
     ("sparc64-unknown-linux-gnu", sparc64_unknown_linux_gnu),
     ("arm-unknown-linux-gnueabi", arm_unknown_linux_gnueabi),
+
+    ("armeb-unknown-linux-gnueabi", armeb_unknown_linux_gnueabi),
+
     ("arm-unknown-linux-gnueabihf", arm_unknown_linux_gnueabihf),
     ("arm-unknown-linux-musleabi", arm_unknown_linux_musleabi),
     ("arm-unknown-linux-musleabihf", arm_unknown_linux_musleabihf),
```

（3）修改 config.toml 文件，包括增加 [target.armeb-unknown-linux-gnueabi] 的配置，将 target 指定为 ["armeb-unknown-linux-gnueabi"]等。

```
target = ["armeb-unknown-linux-gnueabi"]
docs = false
compiler-docs = false
extended = true
tools = ["cargo", "rls", "clippy", "rustfmt", "analysis", "src"]
prefix = "usr"
sysconfdir = "etc"
docdir = "share/doc/rust"
bindir = "bin"
libdir = "lib"
mandir = "share/man"
datadir = "share"
infodir = "share/info"
localstatedir = "var/lib"
channel = "stable"
[target.armeb-unknown-linux-gnueabi]
cc = "armeb-unknown-linux-gnueabi-gcc"
ar = "armeb-unknown-linux-gnueabi-ar"
linker = "armeb-unknown-linux-gnueabi-gcc"
```

（4）将 armeb-unknown-linux-gnueabi 工具链路径添加到环境变量中

（5）编译 rust 并安装（时间可能长达数小时）

```
$ ./x.py build && ./x.py install
```

（6）将 bin 和 lib 添加到环境变量中：

```
export PATH=$PATH:<path/to/rust/usr/bin>
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:<path/to/rust/usr/lib>
```

编译 cc crate 时，需要指定交叉编译时用到的编译器，否则会出现文件格式无法使别的情况：

```
export CC_armeb_unknown_linux_gnueabi=armeb-unknown-linux-gnueabi-gcc
```

### 2.2 使用 rustup 安装

安装：

```
rustup install 1.44.0-x86_64-unknown-linux-gnu
# 切换
rustup default 1.44.0
```

卸载：

```
rustup uninstall 1.44.0
```

### 2.3 创建 rustup 工具链

添加自己编译的工具链（在 `.rustup` 下创建软链接）：

```bash
# 参考[3]
rustup toolchain link my_toolchain <path/to/rust/usr/local>
```

添加之后可以看到对应的工具链：

```bash
[~]$ rustup show
Default host: x86_64-unknown-linux-gnu
rustup home:  /home/wsl/.rustup

installed toolchains
--------------------

stable-x86_64-unknown-linux-gnu
my_toolchain (default)

active toolchain
----------------

my_toolchain (default)
rustc 1.43.0 (4fb7144ed 2020-04-20)
[~]$ ls .rustup/toolchains/
my_toolchain  stable-x86_64-unknown-linux-gnu
```

cargo 和 rustc 版本不匹配时，可能存在问题（新增的选项无法被 rustc 识别）。例如，cargo 版本为 1.5，rustc 版本为 1.43.0，传给 rustc 的选项可能无法被 rustc 识别：

```bash
[~/Code/test/hello_world]$ cargo build --target=mips-unknown-linux-uclibc --verbose
   Compiling hello_world v0.1.0 (/media/B/Code/test/hello_world)
     Running `rustc --crate-name hello_world --edition=2018 src/main.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=e4b5d2b0d82a47b5 -C extra-filename=-e4b5d2b0d82a47b5 --out-dir /media/B/Code/test/hello_world/target/mips-unknown-linux-uclibc/debug/deps --target mips-unknown-linux-uclibc -C linker=mips-openwrt-linux-uclibc-gcc -C incremental=/media/B/Code/test/hello_world/target/mips-unknown-linux-uclibc/debug/incremental -L dependency=/media/B/Code/test/hello_world/target/mips-unknown-linux-uclibc/debug/deps -L dependency=/media/B/Code/test/hello_world/target/debug/deps`
error: unknown codegen option: `embed-bitcode`
```

安装与 rustc 版本号相同的 cargo 即可，比如，使用编译好的 cargo 替换 `~/.cargo/bin/cargo`。

## 3. 所有权

(1) 借用规则

```
frame: T,
frames: std::collections::VecDeque<T>,
pub fn send_frame(&mut self, id: u8, payload: &[u8], len: u8) -> Result<u8, Error> {...}
```

`send_frame` 中的 `self` 是一个结构体 S，有一个类型为 `std::collections::VecDeque<T>` 的成员，即 `frames`，类型 `T` 即 `frame` 的真实类型，也是一个结构体。

```rust
// 这个地方有点疑惑，为什么必须是 `&mut frame`，去掉 `&mut` 会因两次可变借用而编译失败，进一步改为 `get` 后，会因可变借用和不可变借用同时发生而编译失败
if let Some(&mut frame) = self.transport.frames.get_mut(window_size as usize) {
    self.send_frame(frame.min_id, &frame.payload[0..frame.payload_len as usize], frame.payload_len).unwrap_or(0);
}
```

去掉 `&mut` 会因两次可变借用而编译失败：

```rust
if let Some(frame) = self.transport.frames.get_mut(window_size as usize) {
                     --------------------- first mutable borrow occurs here
    self.send_frame(frame.min_id, &frame.payload[0..frame.payload_len as usize], frame.payload_len).unwrap_or(0);
    ^^^^            ------------ first borrow later used here
    |
    second mutable borrow occurs here
```

进一步改为 `get` 后，会因可变借用和不可变借用同时发生而编译失败：

```rust
if let Some(frame) = self.transport.frames.get(window_size as usize) {
                     --------------------- immutable borrow occurs here
    self.send_frame(frame.min_id, &frame.payload[0..frame.payload_len as usize], frame.payload_len).unwrap_or(0);
    ^^^^^----------^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |    |
    |    immutable borrow later used by call
    mutable borrow occurs here
```

## 4. 格式化输出

有时候使用 `print!` 进行格式化输出时，屏幕上没有任何输出信息，原因是 stdout 默认是 line-bffered 的，即行缓冲，遇到换行符或者缓冲区满的时候才会冲刷缓冲区。为了让其立马可以显示，需要在输出语句后面主动冲刷缓冲区：

```rust
use std::io::{self, Write};

print!("Hello");
io::stdout().flush().unwrap();
```

## 5. 字符串处理

`[u8]` 转 `String`：

```
let buf: [u8; 5] = [0x48, 0x65, 0x6c, 0x6c, 0x6f];	// "Hello"
let hello = String::from_utf8(buf.to_vec()).unwrap();
assert_eq!(hello, String::from("Hello"));
```

`String` 转 `[u8]`：

```rust
let hello = String::from("Hello");
let buf = hello.as_bytes();
assert_eq!(buf, [0x48, 0x65, 0x6c, 0x6c, 0x6f]);
```

移除换行符：

```
let mut recv = String::new();
io::stdin()
    .read_line(&mut recv)
    .expect("Failed to read line");
if recv.ends_with('\n') {
    recv.pop();
    if recv.ends_with('\r') {
        recv.pop();
    }
}
```

## 6. 链接问题记录

```
= note: /mnt/f/wsl/project/iot_gw/target/arm-unknown-linux-gnueabihf/release/deps/liblibsqlite3_sys-6e13660e481f9b38.rlib(sqlite3.o): In function `unixDlOpen':
    sqlite3.c:(.text.unixDlOpen+0x8): warning: Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
      /mnt/f/wsl/tool/raspberrypi-tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/bin/../lib/gcc/arm-linux-gnueabihf/4.9.3/../../../../arm-linux-gnueabihf/bin/ld: BFD (crosstool-NG crosstool-ng-1.22.0-88-g8460611) 2.25.1 assertion fail /home/dom/projects/crosstool-ng/install/bin/.build/src/binutils-2.25.1/bfd/elflink.c:2508
      /mnt/f/wsl/tool/raspberrypi-tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/bin/../lib/gcc/arm-linux-gnueabihf/4.9.3/../../../../arm-linux-gnueabihf/bin/ld: BFD (crosstool-NG crosstool-ng-1.22.0-88-g8460611) 2.25.1 assertion fail /home/dom/projects/crosstool-ng/install/bin/.build/src/binutils-2.25.1/bfd/elflink.c:2510
      /mnt/f/wsl/tool/raspberrypi-tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/bin/../lib/gcc/arm-linux-gnueabihf/4.9.3/../../../../arm-linux-gnueabihf/bin/ld: BFD (crosstool-NG crosstool-ng-1.22.0-88-g8460611) 2.25.1 assertion fail /home/dom/projects/crosstool-ng/install/bin/.build/src/binutils-2.25.1/bfd/elflink.c:2508
      /mnt/f/wsl/tool/raspberrypi-tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/bin/../lib/gcc/arm-linux-gnueabihf/4.9.3/../../../../arm-linux-gnueabihf/bin/ld: BFD (crosstool-NG crosstool-ng-1.22.0-88-g8460611) 2.25.1 assertion fail /home/dom/projects/crosstool-ng/install/bin/.build/src/binutils-2.25.1/bfd/elflink.c:2510
      collect2: error: ld returned 1 exit status
```

GCC 静态链接的程序不能调用 dl 库。这个问题是 libsqlite3-sys 会调用 dl 库、项目的 cargo 配置文件设置了静态链接标志导致的，最后的解决办法是删除静态链接标志。

```toml
[target.arm-unknown-linux-gnueabihf]
linker = "arm-linux-gnueabihf-gcc"
ar = "arm-linux-gnueabihf-ar"
# rustflags = ["-C", "target-feature=+crt-static"]
```

## 7. FFI(C)

### (1) 0 长度数组的处理

C 语言中结构体中长度为 0 的数组，在 Rust 中表示时可以将其定义为 `*const *const` 类型的指针，原因是对该内存区域内数据的解释方式要一致，定义为其他类型会造成解释方式不一致，引发乱七八糟的问题。关于 C 语言中 0 长度数组的解释，可以参照《[浅析长度为0的数组](https://www.cnblogs.com/felove2013/articles/4050226.html)》。

例如 libusb-1.0.24 中 BOS 描述符的结构体，由于能力描述符的大小无法在编译时确定，从而整个结构体的大小也无法在编译时确定，因此使用了长度为 0 的数组占位，后续使用中根据 `bNumDeviceCaps` 来分配内存空间，后面的内存是当指针数据来使用的：

```c
// BOS 描述符结构体
struct libusb_bos_descriptor {
	uint8_t  bLength;
	uint8_t  bDescriptorType;
	uint16_t wTotalLength;
	uint8_t  bNumDeviceCaps;
	struct libusb_bos_dev_capability_descriptor *dev_capability[ZERO_SIZED_ARRAY];
};

// 代码片段
static int parse_bos(struct libusb_context *ctx,
	struct libusb_bos_descriptor **bos,
	const uint8_t *buffer, int size)
{
	struct libusb_bos_descriptor *_bos;
	/* ... */
	/* 这个地方根据描述符中的设备能力数来为 BOS 描述符分配内存空间 */
	_bos = calloc(1, sizeof(*_bos) + bos_desc->bNumDeviceCaps * sizeof(void *));
	if (!_bos)
		return LIBUSB_ERROR_NO_MEM;
	/* ... */
    /* dev_capability 数组保存每个能力描述符所在内存的地址 */
    _bos->dev_capability[i] = malloc(header->bLength);
}
```

libusb 的 Rust 绑定，[libusb-sys v0.2.3](https://github.com/dcuddeback/libusb-sys/tree/v0.2.3) 放弃了对能力描述符的解释，能力描述符中的 `Capability-Dependent` 字段的数据长度也一并放弃了：

```rust
#[allow(non_snake_case)]
#[repr(C)]
pub struct libusb_bos_dev_capability_descriptor {
    pub bLength: u8,
    pub bDescriptorType: u8,
    pub bDevCapabilityType: u8,
    // 这个位置应该是 Capability-Dependent 字段的数据
}

#[allow(non_snake_case)]
#[repr(C)]
pub struct libusb_bos_descriptor {
    pub bLength: u8,
    pub bDescriptorType: u8,
    pub wTotalLength: u16,
    pub bNumDeviceCaps: u8,
    // 这个位置应该是说有的能力描述符开始的内存区域
}
```

按照前面说的，在 Rust 中可以将 C 中 0 长度数组的结构体成员定义为 `*const *const` 类型：

```rust
#[allow(non_snake_case)]
#[repr(C)]
pub struct libusb_bos_dev_capability_descriptor {
    pub bLength: u8,
    pub bDescriptorType: u8,
    pub bDevCapabilityType: u8,
    pub dev_capability_data: *const *const u8,
}

#[allow(non_snake_case)]
#[repr(C)]
pub struct libusb_bos_descriptor {
    pub bLength: u8,
    pub bDescriptorType: u8,
    pub wTotalLength: u16,
    pub bNumDeviceCaps: u8,
    pub dev_capability: *const *const libusb_bos_dev_capability_descriptor,
}
```

在调用该结构体的地方，先使用 `std::ptr::addr_of!` 或 `std::ptr::addr_of_mut!` 宏将定义的 `*const *const T` 类型的成员转换为指针，然后在进行两次解引用。对于数组的处理方式，可以调用指针的 `.offset()` 方法完成类似数组索引的操作。例如：

```rust
pub fn dev_capability(&self) -> Vec<BosDevCapabilityDescriptor> {
    unsafe {
        let mut v: Vec<BosDevCapabilityDescriptor> = Vec::new();
        for i in 0..self.num_device_caps() {
            // 先转换成指针
            let point = std::ptr::addr_of!((*self.descriptor).dev_capability).offset(i as _);
            // 在将指针转换为 *const *const libusb_bos_dev_capability_descriptor，然后解引用两次
            let dev_cap = &(*(*(point as * const *const libusb_bos_dev_capability_descriptor)));
            let cap = BosDevCapabilityDescriptor {
                addr: point as *const *const u8,
                bLength: dev_cap.bLength,
                bDescriptorType: dev_cap.bDescriptorType,
                bDevCapabilityType: dev_cap.bDevCapabilityType,
            };
            v.push(cap);
        }
        v
    }
}
```

要确保两边的地址是一样的，因此，添加打印的方式很有用。Rust 打印地址的控制符是 `{:p}`，和 C 的 `%p` 类似，打印指针的内容，即指针所指向的区域的地址，可以参考《[rust:打印变量地址](https://blog.csdn.net/varding/article/details/48104893)》。

Rust 中指针相关的操作，可以参考《[Interacting with data from FFI in Rust](https://blog.guillaume-gomez.fr/articles/2021-07-29+Interacting+with+data+from+FFI+in+Rust)》。

关于，Rust FFI(C) 中复杂类型数据的处理，可以参考《[Rust与C交互(FFI)中复杂类型的处理](https://blog.csdn.net/guiqulaxi920/article/details/78653054)》。

改用 `[*const libusb_bos_dev_capability_descriptor]` 类型的话，Windows 上会引发 `STATUS_HEAP_CORRUPTION` 或 `STATUS_ACCESS_VIOLATION` 错误，原因可能是 Rust 的数组和 C 中的数组在内存的使用上有区别，导致对同一块内存数据的解读方式不同。

例如，如下程序，能取到正确的数据，但是程序会崩溃，错误码为(exit code: 0xc0000374, STATUS_HEAP_CORRUPTION)：

```rust
pub struct libusb_bos_descriptor {
    pub bLength: u8,
    pub bDescriptorType: u8,
    pub wTotalLength: u16,
    pub bNumDeviceCaps: u8,
    // 错误是这一行引起的
    pub dev_capability: [*const libusb_bos_dev_capability_descriptor],
}

pub fn dev_capability(&self) -> Vec<libusb_bos_dev_capability_descriptor> {
    unsafe {
        let mut v: Vec<libusb_bos_dev_capability_descriptor> = Vec::new();
        for i in 0..self.num_device_caps() {
            println!("&dev_capability[{}]: {:p}", i, &(*self.descriptor).dev_capability[i as usize]);
            println!("dev_capability[{}]: {:p}", i, &(*((*self.descriptor).dev_capability)[i as usize]));
            let dev_cap = &(*(*self.descriptor).dev_capability[i as usize]);

            let cap = libusb_bos_dev_capability_descriptor {
                bLength: dev_cap.bLength,
                bDescriptorType: dev_cap.bDescriptorType,
                bDevCapabilityType: dev_cap.bDevCapabilityType,
                //dev_capability_data: dev_cap.dev_capability_data,
            };
            v.push(cap);
        }
        v
    }
}

for cap in bos_desc.dev_capability() {
    println!("      bLength:                 {}", cap.bLength);
    println!("      bDescriptorType:         {}", cap.bDescriptorType);
    println!("      bDevCapabilityType:      {}", cap.bDevCapabilityType);
    //println!("      dev_capability_data:     {}", cap.dev_capability_data);
}
```

下面的错误码是 (exit code: 0xc0000005, STATUS_ACCESS_VIOLATION)：

```rust
pub struct libusb_bos_descriptor {
    pub bLength: u8,
    pub bDescriptorType: u8,
    pub wTotalLength: u16,
    pub bNumDeviceCaps: u8,
    // 错误是这一行引起的
    pub dev_capability: [*const libusb_bos_dev_capability_descriptor],
}

pub fn dev_capability(&self) -> &[*const libusb_bos_dev_capability_descriptor] {
    unsafe {
        &(*self.descriptor).dev_capability
    }
}

for cap in bos_desc.dev_capability() {
    unsafe {
        println!("      bLength:                 {}", (*(*cap)).bLength);
        println!("      bDescriptorType:         {}", (*(*cap)).bDescriptorType);
        println!("      bDevCapabilityType:      {}", (*(*cap)).bDevCapabilityType);
        //println!("      dev_capability_data:     {}", cap.dev_capability_data);
    }
}
```

## 参考

[1] [https://rust-embedded.github.io/embedonomicon/custom-target.html#use-the-target-file](https://rust-embedded.github.io/embedonomicon/custom-target.html#use-the-target-file)

[2] [https://docs.rust-embedded.org/faq.html#compilation-target-support](https://docs.rust-embedded.org/faq.html#compilation-target-support)

[3] [https://rustc-dev-guide.rust-lang.org/building/build-install-distribution-artifacts.html](https://rustc-dev-guide.rust-lang.org/building/build-install-distribution-artifacts.html)
