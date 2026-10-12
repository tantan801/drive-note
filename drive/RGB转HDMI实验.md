# 第七十二章 RGB 转 HDMI 实验

目前大多数的显示器都提供了 HDMI 接口，HDMI 的应用范围也越来越广，但是 I.MX6ULL 这颗芯片原生并不支持 HDMI 显示，但是我们可以通过 RGB 转 HDMI 芯片将 RGB 信号转为 HDMI 信号，这样就可以连接 HDMI 显示器了。本章我们就来学习一下如何在正点原子的 I.MX6U-ALPHA 开发板上实现 RGB 转 HDMI。

## 72.1 RGB 转 HDMI 简介

I.MX6ULL 这颗 SOC 是没有 HDMI 外设，只有 RGB 屏幕接口，因此只能通过 RGB 转 HDMI 的芯片来实现 HDMI 连接。效果肯定是没法和原生支持 HDMI 接口的 SOC 比，当个玩具来玩一下还是可以的。因此本质上来讲还是 RGB 驱动，并非原生的 HDMI 驱动。

正点原子为 I.MX6U-ALPHA 开发板提供了 RGB 转 HDMI 模块，通过 40Pin 的 FPC 排线将模块与 ALPHA 开发板的 RGB 接口连接在一起，另一端通过 HDMI 线将模块与显示器连接在一起。

模块使用 Sii9022A 这颗芯片来完成 RGB 转 HDMI，Sii9022A 以前是 Silicon Image 公司出品的，但是 Silicon Image 后来被 Lattice 收购了。

SiI9022A 是一款 HDMI 传输芯片，适用于高清便携相机、数字相机和个人移动设备，可以灵活的将其他音视频接口转换为 HDMI 或者 DVI 格式。SiI9022A 支持预编程 HDCP 键码，可以完全自动进行 HDCP 检测和鉴定。SiI9022A 是一个视频转换芯片，支持输入视频格式有：xvYCC、BTA-T1004、ITU-R.656，内置 DE 发生器支持 SYNC 格式(RGB 格式)。输出格式支持：HDMI、HDCP 和 DVI、最高支持 1080P 视频输出、支持 HDMI A、HDMI C 和 Micro-D 连接器。SiI9022A 功能非常多，具体使用什么功能需要进行配置，因此 SiI9022A 提供了一个 I2C 接口用于配置。

## 72.2 硬件原理图分析

由于 RGB 转 HDMI 模块详细的原理图比较大，这里就不贴出来了，大家自行查看相应的原理图，RGB 转 HDMI 模块的原理图已经放到了开发板光盘中，路径为：2、开发板原理图->正点原子其他模块原理图->ATK-RGB2HDMI 模块 V1.3.pdf。这里我们重点来看一下模块原理图中与 ALPHA 开发板的 FPC 连接口，如图1-1所示

![1-1 RGB转HDMI模块FPC接口](photo/RGB转HDMI实验/1-1%20RGB转HDMI模块FPC接口.png)

上图就是 RGB 转 HDMI 模块的 40P FPC 接口，使用一根 40P 反向 FPC 连接线与 ALPHA 开发板的 RGB 接口连接，FPC 线一定要短，推荐使用 5CM 长的。从图 72.2.1 可以看出，主要是 RGB 接口引脚，右上角一路 I2C 接口，这路 I2C 最终连接到了 ALPHA 开发板的触摸屏 I2C 接口上，也就是 I.MX6ULL 的 I2C2 接口。另外还有一个中断 INT 和一个复位 HDMI_RESET 引脚。

## 72.3 实验驱动编写

### 72.3.1 修改设备树

#### 1、HDMI 模块 IO 节点信息添加
HDMI模块包含RGB接口与I2C2接口；I2C2的IO配置参考《第六十四章 Linux多点电容触摸屏实验》64.3.1小节，本节重点配置RGB相关引脚。 打开imx6ull-alientek-emmc.dts，在iomuxc节点的imx6ulevk子节点添加如下内容：

**示例代码 72.3.1.1 iomuxc 子节点 imx6ulevk 中添加的 RGB 引脚信息**

```dts
pinctrl_hdmi_dat: hdmidatgrp {
	fsl,pins = <
		MX6UL_PAD_LCD_DATA00__LCDIF_DATA00 0x49
		MX6UL_PAD_LCD_DATA01__LCDIF_DATA01 0x49
		MX6UL_PAD_LCD_DATA02__LCDIF_DATA02 0x49
		MX6UL_PAD_LCD_DATA03__LCDIF_DATA03 0x49
		MX6UL_PAD_LCD_DATA04__LCDIF_DATA04 0x49
		MX6UL_PAD_LCD_DATA05__LCDIF_DATA05 0x49
		MX6UL_PAD_LCD_DATA06__LCDIF_DATA06 0x49
		MX6UL_PAD_LCD_DATA07__LCDIF_DATA07 0x49
		MX6UL_PAD_LCD_DATA08__LCDIF_DATA08 0x49
		MX6UL_PAD_LCD_DATA09__LCDIF_DATA09 0x49
		MX6UL_PAD_LCD_DATA10__LCDIF_DATA10 0x49
		MX6UL_PAD_LCD_DATA11__LCDIF_DATA11 0x49
		MX6UL_PAD_LCD_DATA12__LCDIF_DATA12 0x49
		MX6UL_PAD_LCD_DATA13__LCDIF_DATA13 0x49
		MX6UL_PAD_LCD_DATA14__LCDIF_DATA14 0x49
		MX6UL_PAD_LCD_DATA15__LCDIF_DATA15 0x51
		MX6UL_PAD_LCD_DATA16__LCDIF_DATA16 0x49
		MX6UL_PAD_LCD_DATA17__LCDIF_DATA17 0x49
		MX6UL_PAD_LCD_DATA18__LCDIF_DATA18 0x49
		MX6UL_PAD_LCD_DATA19__LCDIF_DATA19 0x49
		MX6UL_PAD_LCD_DATA20__LCDIF_DATA20 0x49
		MX6UL_PAD_LCD_DATA21__LCDIF_DATA21 0x49
		MX6UL_PAD_LCD_DATA22__LCDIF_DATA22 0x49
		MX6UL_PAD_LCD_DATA23__LCDIF_DATA23 0x49
	>;
};

/* zuozhongkai HDMI RGB */
pinctrl_hdmi_ctrl: hdmictrlgrp {
	fsl,pins = <
		MX6UL_PAD_LCD_CLK__LCDIF_CLK 0x49
		MX6UL_PAD_LCD_ENABLE__LCDIF_ENABLE 0x49
		MX6UL_PAD_LCD_HSYNC__LCDIF_HSYNC 0x49
		MX6UL_PAD_LCD_VSYNC__LCDIF_VSYNC 0x49
	>;
};

/* zuozhongkai SII902X INT*/
pinctrl_sii902x: hdmigrp-1 {
	fsl,pins = <
		MX6UL_PAD_GPIO1_IO09__GPIO1_IO09 0x11
	>;
};
```

在iomuxc_snvs节点的imx6ulevk子节点，添加模块复位引脚：

**示例代码 72.3.1.2 iomuxc_snvs 子节点 imx6ulevk 中的复位引脚信息**

```dts
ts_reset_hdmi_pin: ts_reset_hdmi_mux {
	fsl,pins = <
		MX6ULL_PAD_SNVS_TAMPER9__GPIO5_IO09 0x49
	>;
};
```

#### 2、sii902x 节点创建
Sii902x通过I2C配置，硬件接I2C2，在dts的i2c2节点下增加sii902x子节点。

**示例代码 72.3.1.3 i2c2 节点下添加 sii902x 子节点**

```dts
sii902x: sii902x@39 {
	compatible = "SiI,sii902x";
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_sii902x>;
	interrupt-parent = <&gpio1>;
	interrupts = <9 IRQ_TYPE_EDGE_FALLING>;
	irq-gpios = <&gpio1 9 GPIO_ACTIVE_LOW>;
	mode_str = "1280x720M@60";
	bits-per-pixel = <16>;
	resets = <&sii902x_reset>;
	reg = <0x39>;
	status = "okay";
};
```
mode_str配置分辨率与帧率，此处为1280*720@60。

在设备树根节点/下创建复位子节点：

**示例代码 72.3.1.4 根节点"/"下添加 sii902x 复位子节点**

```dts
sii902x_reset: sii902x-reset {
	compatible = "gpio-reset";
	reset-gpios = <&gpio5 9 GPIO_ACTIVE_LOW>;
	reset-delay-us = <100000>;
	#reset-cells = <0>;
	status = "disabled";
};
```

#### 3、修改 lcdif 节点下的像素极性
切换为RGB转HDMI模块，修改lcdif的pixelclk-active属性为1，其余参数保持不变。

**示例代码 72.3.1.5 修改 lcdif 节点的 pixelclkactive 属性**

```dts
&lcdif {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hdmi_dat
				 &pinctrl_hdmi_ctrl>;
	display = <&display0>;
	status = "okay";

	display0: display {
		......
		pixelclk-active = <1>;
	};
};
```

#### 4、屏蔽其他复用的IO
模块复位引脚GPIO5_IO09、中断引脚GPIO1_IO09用作GPIO。检索整个设备树，屏蔽其他地方对gpio5 9、gpio1 9的使用，防止IO资源冲突造成驱动加载失败。

### 72.3.2 使能内核自带的 sii902x 驱动

内核配置开启SII9022驱动，配置路径：
```
-> Device Drivers
	-> Pulse‑Width Modulation (PWM) Support
		-> <*> i.MX PWM support
```

配置如图1-2所示，编译内核验证编译无报错。

![1-2 使能sii902x驱动](photo/RGB转HDMI实验/1-2%20使能sii902x驱动.png)

### 72.3.3 修改 sii902x 驱动

修改内核文件 `drivers/video/fbdev/mxc/mxsfb_sii902x.c`。

修改sii902x_poweron、sii902x_poweroff函数：

**示例代码 72.3.3.1 修改 mxsfb_sii902x.c 文件**

```c
static void sii902x_poweron(void)
{
	/* Turn on DVI or HDMI */
	i2c_smbus_write_byte_data(sii902x.client, 0x1A, 0x00);
	return;
}

static void sii902x_poweroff(void)
{
	/* disable tmds before changing resolution */
	i2c_smbus_write_byte_data(sii902x.client, 0x1A, 0x10);
	return;
}
```

修改mxsfb_get_of_property函数，增加中断GPIO读取配置，第6、22、23行为新增代码：

**示例代码 72.3.3.2 mxsfb_get_of_property 函数**

```c
static int mxsfb_get_of_property(void)
{
	struct device_node *np = sii902x.client->dev.of_node;
	const char *mode_str;
	int bits_per_pixel, ret;
	int irq_pin;

	ret = of_property_read_string(np, "mode_str", &mode_str);
	......
	sii902x.mode_str = mode_str;
	sii902x.bits_per_pixel = bits_per_pixel;

	irq_pin = of_get_named_gpio(np, "irq-gpios", 0);
	gpio_direction_output(irq_pin, 1);

	return ret;
}
```

文件头部添加头文件：

**示例代码 72.3.3.3 mxsfb_sii902x.c 文件添加头文件**

```c
#include <linux/of_gpio.h>
```

全部修改完成后重新编译Linux内核。

## 72.4 RGB 转 HDMI 测试

使用 FPC 线将 RGB 转 HDMI 模块连接到开发板上的 RGB 屏幕接口上，RGB 转 HDMI 模块通过 HDMI 线连接到显示器上。

使用新编译得到的 zImage 和 imx6ull-alientek-emmc.dtb 启动开发板，如果驱动工作正常的话显示器就会有显示。