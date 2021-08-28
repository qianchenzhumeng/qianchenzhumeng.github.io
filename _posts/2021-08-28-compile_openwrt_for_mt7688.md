---
layout: post
title:  "OpenWrt 编译"
date:   2021-08-28 08:00:00 +0800
categories: [Embedded, OpenWrt]
tags: [OpenWrt]
excerpt: 这篇文章以openwrt-19.07.2为例，介绍如何编译 OpenWrt。
---

## 1. 背景

- 处理器：MT7688
- OpenWrt版本：19.07.2
- 编译环境：wsl（Ubuntu 20.04.1 LTS）

## 2. 下载源码

源码地址：[https://github.com/openwrt/openwrt](https://github.com/openwrt/openwrt)

可以通过 [gitee](https://gitee.com/) 加速代码下载，具体做法是在 gitee 上通过“从GitHub/GitLab导入仓库”的形式创建仓库，等创建好后，从 gitee 克隆该仓库。

如果需要对源码做些修改，并且希望可以保存下来，建议自己新建仓库，后续将更改提交到该仓库；如果不需要保存，直接克隆 gitee 上已有的 OpenWrt 的源码克隆仓库就可以了：

```bash
git clone git@gitee.com:openwrt-mirror/openwrt.git
```

如果想提交到 GitHub 上，现在 GitHub 上 fork 仓库，然后按上面的方法将仓库克隆到本地，之后修改下仓库的 `.git/config` 文件，将远端 url 改成 GitHub 上自己的仓库对应的 ssh 地址。

检出对应的版本：

```bash
git checkout v19.07.2
```

检出后 HEAD 是游离状态，可以以该标签为基础创建一个新分支：

```bash
git checkout -b 19.07.2
```

后续所有的修改动作都在这个新分支上进行。

## 3. 编译

### (1) 更新 feeds 源

编译过程中，OpenWrt 还会下载很多东西，速度也比较慢，可以按照上面介绍的使用 gitee 加速的方法，将源码目录下 feeds.conf.default 文件内使用的到仓库克隆到 gitee，然后使用 gtiee 上的地址替换 feeds.conf.default 文件内的地址，来加速编译过程中的下载动作。注意，修改地址即可，不要修改 commit 号[[1]](https://www.cnblogs.com/thammer/p/13531058.html)。

接下来更新 feeds：

```bash
./scripts/feeds update -a
```

使下载的包能够在 `make menuconfig` 中可用：

```bash
./scripts/feeds install -a
```

### (2) 安装第三方 luci 应用

例如安装通过 web 管理页面给 arduino 烧录程序的应用 [luci-app-avrdude](https://github.com/qianchenzhumeng/luci-app-avrdude)：

```bash
cd feeds/luci/applications
git clone https://github.com/qianchenzhumeng/luci-app-avrdude.git
```

安装 feeds：

```bash
cd ../../../
./scripts/feeds install -a -p luci
```

### (3) 增加目标板配置

可以找卖家咨询一下，开发板是否需要特定的配置，如设备树配置文件、makefile配置等，如果有，需要在配置源码（`make menuconfig`）之前完成，后续在配置源码的时候需要选择该配置。

比如，我手中的开发板是需要特定的设备树配置文件的。卖家提供了设备树配置文件以及修改方法：

- 将设备树配置文件 [BL_SMGW.dts](https://github.com/qianchenzhumeng/bliot_config/blob/master/BL_SMGW.dts) 放到 target/linux/ramips/dts 目录下
- 使用 [mt76x8.mk](https://github.com/qianchenzhumeng/bliot_config/blob/master/mt76x8.mk) 替换 `target/linux/ramips/image/mt76x8.mk`

链接中的 `mt76x8.mk` 在原来的基础上增加了如下内容：

```makefile
define Device/bliot_smart-gateway
  DTS := BL_SMGW
  IMAGE_SIZE := 16064k
  DEVICE_TITLE := BLIOT SmartGateway
  DEVICE_PACKAGES := kmod-usb2 kmod-usb-ohci
endef
TARGET_DEVICES += bliot_smart-gateway
```

### (4) 配置源码

不熟悉自己的硬件的话，可以找卖家要该开发板的 `.config` 文件，放到源码根目录，编译即可。

####  1) 配置开发板信息

熟悉的话，可以运行 `make menuconfig` 进行配置，例如：

- Target System (MediaTek Ralink MIPS)
- Subtarget (MT76x8 based boards)
- Target Profile (BLIOT SmartGateway)

上面的 `Target Profile` 就是之前新增的配置。

####  2) 配置生成 SDK

如果后续需要在此版本上开发应用程序，那么建议打开生成 SDK 的选项（Build the OpenWrt SDK）。如果配置了该选项，编译过程中会生成对应的交叉编译工具链。

####  3) 添加应用

以刚才安装的 luci 应用 [luci-app-avrdude](https://github.com/qianchenzhumeng/luci-app-avrdude) 为例，按如下层级选中：

```
-> LuCI
    -> 3. Applications
       -> luci-app-avrdude
```

配置好之后保存、退出，源码根目录下会生成对应的 `.config` 文件。

### (5) 编译

接下来编译。如果是 WSL，编译过程可能会应为目录权限问题失败，需要将源码目录移动到 WSL 的根目录下，再编译。

```bash
make V=99
```

首次编译可能会持续数小时，大部分时间会耗在文件下载上面。

编译后的产物位于 bin 目录下：

```
bin
├── packages
│   └── mipsel_24kc
│       ├── base
│       ├── luci
│       ├── packages
│       ├── routing
│       └── telephony
└── targets
    └── ramips
        └── mt76x8
```

其中，固件及 SDK 位于 `bin/targets/ramips/mt76x8` 目录下：

- openwrt-ramips-mt76x8-bliot_smart-gateway-squashfs-sysupgrade.bin
- openwrt-sdk-ramips-mt76x8_gcc-7.5.0_musl.Linux-x86_64.tar.xz

## 4. 故障解决

```
mv: cannot move 'bin/aclocal.tmp2' to 'bin/aclocal.tmp': Permission denied
```

WSL 操作 windows 中的目录权限存在问题，将源码目录移动到根目录下（见[[4]](https://www.right.com.cn/forum/thread-1388297-1-1.html)）。如果没有改动 WSL 根目录的话，需要确保 C 盘有足够的空间。

## 参考

[1] [加速openwrt编译过程中的下载动作](https://www.cnblogs.com/thammer/p/13531058.html)

[2] [https://www.jianshu.com/p/00bb57319302](https://www.jianshu.com/p/00bb57319302)

[3] [https://docs.labs.mediatek.com/resource/linkit-smart-7688/zh_cn/tutorials/firmware-and-bootloader/build-the-firmware-from-source-codes](https://docs.labs.mediatek.com/resource/linkit-smart-7688/zh_cn/tutorials/firmware-and-bootloader/build-the-firmware-from-source-codes)

[4] [https://www.right.com.cn/forum/thread-1388297-1-1.html](https://www.right.com.cn/forum/thread-1388297-1-1.html)

[5] [使用 GitHub Actions 云编译 OpenWrt](https://www.cnblogs.com/lsgxeva/p/13742939.html) 

[6] [openwrt目录结构之staging_dir](https://blog.csdn.net/lixiaofeng0/article/details/109901620)

[7] [LuCI 界面开发](https://blog.csdn.net/qq_28812525/article/details/103870169)

[8] [Openwrt Image Builder/SDK 初探](https://www.cnblogs.com/tanhangbo/p/4559168.html)

[9] [OpenWRT添加自定义LUCI页面示例](https://blog.csdn.net/kakabuqinuo/article/details/103364029)

[10] [使用OpenWRT路由远程给Arduino下载程序](https://www.geek-workshop.com/thread-5816-1-1.html)

