---
layout: post
title:  "OpenWrt GPIO（MT7688AN）"
date:   2021-03-20 10:00:00 +0800
categories: [Embedded, OpenWrt]
tags: [OpenWrt]
---

## 1. 设备树

在开发板上查看引脚使用情况：

```bash
root@OpenWrt:~# cat /sys/kernel/debug/gpio
gpiochip0: GPIOs 0-31, parent: platform/10000600.gpio, 10000600.gpio:
 gpio-6   (                    |reboot              ) in  hi

gpiochip1: GPIOs 32-63, parent: platform/10000600.gpio, 10000600.gpio:
 gpio-38  (                    |reset               ) in  hi
 gpio-41  (                    |skw92a:green:wps    ) out hi
 gpio-42  (                    |skw92a:green:wlan   ) out hi

gpiochip2: GPIOs 64-95, parent: platform/10000600.gpio, 10000600.gpio:
root@OpenWrt:~#
```

设备树位于 “openwrt-19.07.2/target/linux/ramips/dts” 目录下，手中的开发板对应的设备树文件中有如下内容：

```
gpio-leds {
	compatible = "gpio-leds";

	led_power: wps {
		label = "skw92a:green:wps";
		gpios = <&gpio1 9 GPIO_ACTIVE_LOW>;
	};

	wlan {
		label = "skw92a:green:wlan";
		gpios = <&gpio1 10 GPIO_ACTIVE_LOW>;
	};
};
```

可以看出设备树文件中 `led_power: wps` 对应开发板上的 `gpio-41`，`wlan` 对应开发板上的 `gpio-42`。

设备树文件中的 `gpios = <&gpio1 9 GPIO_ACTIVE_LOW>` 是该 gpio 口信息的属性值，格式为 `<&gpio控制器节点名 具体gpio口的标识符>`，其中，gpio 的标识符是由多个数字组成, 数字的个数由所用的 gpio 控制器节点里的 `#gpio-cells` 属性值指定的，该属性值可以在开发板所包含的设备树文件 `mt7628an.dtsi` 中找到。

“openwrt-19.07.2/target/linux/ramips/dts/mt7628an.dtsi” 里关于gpio控制器的节点信息如下：

```
gpio@600 {
	#address-cells = <1>;
	#size-cells = <0>;

	compatible = "mtk,mt7628-gpio", "mtk,mt7621-gpio";
	reg = <0x600 0x100>;

	interrupt-parent = <&intc>;
	interrupts = <6>;

	gpio0: bank@0 {
		reg = <0>;
		compatible = "mtk,mt7621-gpio-bank";
		gpio-controller;
		#gpio-cells = <2>; 
	};

	gpio1: bank@1 {
		reg = <1>;
		compatible = "mtk,mt7621-gpio-bank";
		gpio-controller;
		#gpio-cells = <2>;
	};

	gpio2: bank@2 {
		reg = <2>;
		compatible = "mtk,mt7621-gpio-bank";
		gpio-controller;
		#gpio-cells = <2>;
	};
};
```

其中，`#gpio-cells = <2>` 表明每个 gpio 口的标识符由 2 个数字来识别。从下面的帮助文档可以得知，第 1 个数字表示引脚在所属 gpio controller 内的编号, 第 2 个数字是标志位（高电平或低电平）。其中，高低电平定义在 “openwrt-19.07.2/build_dir/target-mipsel_24kc_musl/linux-ramips_mt76x8/linux-4.14.171/include/dt-bindings/gpio/gpio.h”中。

帮助文档：

- openwrt-19.07.2/build_dir/target-mipsel_24kc_musl/linux-ramips_mt76x8/linux-4.14.171/Documentation/devicetree/bindings/gpio/gpio.txt

文档中有如下内容：

> Every GPIO controller node must contain both an empty "gpio-controller"
> property, and a #gpio-cells integer property, which indicates the number of
> cells in a gpio-specifier.
>
> Some system-on-chips (SoCs) use the concept of GPIO banks. A GPIO bank is an
> instance of a hardware IP core on a silicon die, usually exposed to the
> programmer as a coherent range of I/O addresses. Usually each such bank is
> exposed in the device tree as an individual gpio-controller node, reflecting
> the fact that the hardware was synthesized by reusing the same IP block a
> few times over.
>
> Optionally, a GPIO controller may have a "ngpios" property. This property
> indicates the number of in-use slots of available slots for GPIOs. The
> typical example is something like this: the hardware register is 32 bits
> wide, but only 18 of the bits have a physical counterpart. The driver is
> generally written so that all 32 bits can be used, but the IP block is reused
> in a lot of designs, some using all 32 bits, some using 18 and some using 12.
> In this case, setting "ngpios = <18>;" informs the driver that only the
> first 18 GPIOs, at local offset 0 .. 17, are in use.

结合 [MT7688_Datasheet_v1_4.pdf](https://labs.mediatek.com/en/download/50WkbgbH) 5.8.4 一节可以看出，MT7688 有 3 个 gpio 控制器，每个控制器的 `ngpios` 属性是 32：

![gpio_register](/assets/img/2021-03-20-openwrt_gpio_of_mt7688an.assets/gpio_register.png)

引脚的编号和设备树中的标识符关系如下：

引脚编号 = 所属控制器基数 * ngpios + 在所属控制寄存器中对应的位数

以前面的设备树文件中的 gpio 标识符 `<&gpio1 9 GPIO_ACTIVE_LOW>` 来举例，对应的引脚为 `1 * 32 + 9`，即 gpio-41。

## 2. 使用 GPIO

引脚的复用功能定义在 openwrt-19.07.2/build_dir/target-mipsel_24kc_musl/linux-ramips_mt76x8/linux-4.14.171/arch/mips/ralink/mt7620.c 文件中，例如：

```c
static struct rt2880_pmx_func gpio_grp_mt7628[] = {
	FUNC("pcie", 3, 11, 1),
	FUNC("refclk", 2, 11, 1),
	FUNC("gpio", 1, 11, 1),
	FUNC("gpio", 0, 11, 1),	/* "gpio" 使用的 gpio 从 GPIO#11 开始，共 1 个引脚 */
};
```

数组 `mt7628an_pinmux_data` 中包含了所有的组。

如果想使用开发板上的 `GPIO0` 引脚，在 [MT7688_Datasheet_v1_4.pdf](https://labs.mediatek.com/en/download/50WkbgbH) 中可以看到 `GPIO0` 对应的引脚是 `GPIO#11`：

![GPIO0](/assets/img/2021-03-20-openwrt_gpio_of_mt7688an.assets/gpio0.png)

```bash
# 使用 GPIO#11
echo "11" > /sys/class/gpio/export
# 设置方向为输出
echo "out" > /sys/class/gpio/gpio11/direction
# 输出高电平
echo "1" > /sys/class/gpio/gpio11/value
# 输出低电平
echo "0" > /sys/class/gpio/gpio11/value
```

如果想使用其他引脚，可能需要通过设备树文件修改端口复用功能，参考 [[4]](https://blog.csdn.net/jazzsoldier/article/details/65438903)。

## 参考

[1] [OpenWRT GPIO 口配置](https://www.jianshu.com/p/5e840ca0ee19)

[2] [OpenWrt之GPIO操作](https://blog.csdn.net/hzlarm/article/details/103120139)

[3] [Linux驱动之GPIO子系统和pinctrl子系统](https://www.cnblogs.com/UnfriendlyARM/p/13674502.html)

[4] [openwrt gpio控制与使用](https://blog.csdn.net/jazzsoldier/article/details/65438903)