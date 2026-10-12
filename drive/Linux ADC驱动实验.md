# 第七十六章 Linux ADC 驱动实验

上一章我们讲解了如何给 ICM20608 编写 IIO 驱动，ICM20608 本质就是 ADC，因此纯粹的 ADC 驱动也是 IIO 驱动框架的。本章我们就来学习一下如何使用 I.MX6ULL 内部的 ADC，并且在学习巩固一下 IIO 驱动。

## 76.1 ADC 简介

关于 ADC 的介绍，我们已经在 ADC 裸机章节已经详细讲解过了，请大家参考《第 C1 章 ADC 实验》。

## 76.2 ADC 驱动源码简析

### 76.2.1 设备树下的 ADC 节点

I.MX6ULL 有 2 个 ADC，但是对应一个 ADC 控制器。本章实验我们使用 GPIO1_IO01 这个引脚来完成 ADC 实验，而 GPIO1_IO01 就是 ADC1 的通道 1 引脚，所以这里我们就以 ADC1 为例进行讲解，imx6ull.dtsi 文件中的 adc1 节点信息如下：

**示例代码 76.2.1.1 adc1 节点内容**

```dts
adc1: adc@02198000 {
	compatible = "fsl,imx6ul-adc", "fsl,vf610-adc";
	reg = <0x02198000 0x4000>;
	interrupts = <GIC_SPI 100 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&clks IMX6UL_CLK_ADC1>;
	num-channels = <2>;
	clock-names = "adc";
	status = "disabled";
};
```

第 2 行，compatible 属性值为"fsl,imx6ul-adc"和"fsl,vf610-adc"，所以在整个 Linux 源码里面搜索这个两个字符串即可找到 I.MX6ULL 的 ADC 驱动核心文件，这个文件就是 `drivers/iio/adc/vf610_adc.c`。

关于 I.MX6ULL 的 ADC 节点更为详细的信息请参考对应的绑定文档：`Documentation/devicetree/bindings/iio/adc/vf610-adc.txt`。接下来我们简单分析一下绑定文档，后面我们需要根据绑定文档修改设备树，使能 ADC 对应的通道。

ADC 相关属性有：
- compatible：兼容性属性，必须的，可以设置为"fsl,vf610-adc"。
- reg：ADC 控制器寄存器信息。
- interrupts：中断属性，ADC1 和 ADC2 各对应一个中断信息。
- clocks：时钟属性。
- clock-names：时钟名字，可选"adc"。
- vref-supply：此属性对应 vref 参考电压句柄。

可以看出，ADC 节点的属性还是比较少的。

### 76.2.2 ADC 驱动源码分析

I.MX6ULL 的 ADC 驱动文件就一个 vf610_adc.c，vf610_adc.c 主体框架是 platform，配合 IIO 驱动框架实现 ADC 驱动。

#### 1、vf610_adc 结构体

NXP 自己将 ADC 外设抽象成了结构体 vf610_adc，vf610_adc 就相当于自定义的设备结构体。vf610_adc 结构体贯穿于整个驱动文件，结构体内容如下：

**示例代码 76.2.2.1 vf610_adc 结构体**

```c
struct vf610_adc {
	struct device *dev;
	void __iomem *regs;
	struct clk *clk;

	u32 vref_uv;
	u32 value;
	struct regulator *vref;
	struct vf610_adc_feature adc_feature;

	u32 sample_freq_avail[5];

	struct completion completion;
};
```

#### 2、vf610_adc_probe 函数

接下来看一下 vf610_adc_probe 函数，内容如下(有省略)：

**示例代码 76.2.2.2 vf610_adc_probe 函数**

```c
static int vf610_adc_probe(struct platform_device *pdev)
{
	struct vf610_adc *info;
	struct iio_dev *indio_dev;
	struct resource *mem;
	int irq;
	int ret;
	u32 channels;

	indio_dev = devm_iio_device_alloc(&pdev->dev,
	sizeof(struct vf610_adc));
	if (!indio_dev) {
		dev_err(&pdev->dev, "Failed allocating iio device\n");
		return -ENOMEM;
	}

	info = iio_priv(indio_dev);
	info->dev = &pdev->dev;

	mem = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	info->regs = devm_ioremap_resource(&pdev->dev, mem);
	if (IS_ERR(info->regs))
		return PTR_ERR(info->regs);

	irq = platform_get_irq(pdev, 0);
	if (irq < 0) {
		dev_err(&pdev->dev, "no irq resource?\n");
		return irq;
	}

	ret = devm_request_irq(info->dev, irq,
	vf610_adc_isr, 0,
	dev_name(&pdev->dev), info);
	if (ret < 0) {
		dev_err(&pdev->dev, "failed requesting irq, irq = %d\n", irq);
		return ret;
	}

	info->clk = devm_clk_get(&pdev->dev, "adc");
	if (IS_ERR(info->clk)) {
		dev_err(&pdev->dev, "failed getting clock, err = %ld\n",
		PTR_ERR(info->clk));
		return PTR_ERR(info->clk);
	}

	info->vref = devm_regulator_get(&pdev->dev, "vref");
	if (IS_ERR(info->vref))
		return PTR_ERR(info->vref);

	ret = regulator_enable(info->vref);
	if (ret)
		return ret;

	info->vref_uv = regulator_get_voltage(info->vref);

	platform_set_drvdata(pdev, indio_dev);

	init_completion(&info->completion);

	ret = of_property_read_u32(pdev->dev.of_node,
	"num-channels", &channels);
	if (ret)
		channels = ARRAY_SIZE(vf610_adc_iio_channels);

	indio_dev->name = dev_name(&pdev->dev);
	indio_dev->dev.parent = &pdev->dev;
	indio_dev->dev.of_node = pdev->dev.of_node;
	indio_dev->info = &vf610_adc_iio_info;
	indio_dev->modes = INDIO_DIRECT_MODE;
	indio_dev->channels = vf610_adc_iio_channels;
	indio_dev->num_channels = (int)channels;

	ret = clk_prepare_enable(info->clk);
	if (ret) {
		dev_err(&pdev->dev,
		"Could not prepare or enable the clock.\n");
		goto error_adc_clk_enable;
	}

	vf610_adc_cfg_init(info);
	vf610_adc_hw_init(info);

	ret = iio_device_register(indio_dev);
	if (ret) {
		dev_err(&pdev->dev, "Couldn't register the device.\n");
		goto error_iio_device_register;
	}

	return 0;
......
	return ret;
}
```

第 10 行，调用 `devm_iio_device_alloc` 函数申请 iio_dev，这里也连 vf610_adc 内存一起申请了。 第 16 行，调用 `iio_priv` 函数从 iio_dev 里面得到 vf610_adc 首地址。 第 24 行，调用 `platform_get_irq` 获取中断号。 第 30 行，调用 `devm_request_irq` 函数申请中断，中断服务函数为 vf610_adc_isr。 第 64~70 行，初始化 iio_dev，重点是第 67 行的 `vf610_adc_iio_info`，因为用户空间读取 ADC 数据最终就是由 `vf610_adc_iio_info` 来完成的。 第 79 行，调用 `vf610_adc_cfg_init` 函数完成 ADC 的配置初始化。 第 80 行，调用 `vf610_adc_hw_init` 函数来初始化 ADC 硬件。 第 82 行，调用 `iio_device_register` 函数向内核注册 iio_dev。

可以看出 vf610_adc_probe 函数核心就是初始化 ADC 控制器，然后建立 ADC 的 IIO 驱动框架。

#### 3、vf610_adc_iio_info 结构体

vf610_adc_iio_info 结构体内容如下所示：

**示例代码 76.2.2.3 vf610_adc_iio_info 结构体**

```c
static const struct iio_info vf610_adc_iio_info = {
	.driver_module = THIS_MODULE,
	.read_raw = &vf610_read_raw,
	.write_raw = &vf610_write_raw,
	.debugfs_reg_access = &vf610_adc_reg_access,
	.attrs = &vf610_attribute_group,
};
```

我们重点来看一下第 3 行的 `vf610_read_raw` 函数，因为此函数才是最终向用户空间发送 ADC 原始数据的，函数内容如下：

**示例代码 76.2.2.4 vf610_read_raw 函数**

```c
static int vf610_read_raw(struct iio_dev *indio_dev,
struct iio_chan_spec const *chan,
int *val,
int *val2,
long mask)
{
	struct vf610_adc *info = iio_priv(indio_dev);
	unsigned int hc_cfg;
	long ret;

	switch (mask) {
	case IIO_CHAN_INFO_RAW:
	case IIO_CHAN_INFO_PROCESSED:
		mutex_lock(&indio_dev->mlock);
		reinit_completion(&info->completion);

		hc_cfg = VF610_ADC_ADCHC(chan->channel);
		hc_cfg |= VF610_ADC_AIEN;
		writel(hc_cfg, info->regs + VF610_REG_ADC_HC0);
		ret = wait_for_completion_interruptible_timeout
		(&info->completion, VF610_ADC_TIMEOUT);
		if (ret == 0) {
			mutex_unlock(&indio_dev->mlock);
			return -ETIMEDOUT;
		}
		if (ret < 0) {
			mutex_unlock(&indio_dev->mlock);
			return ret;
		}

		switch (chan->type) {
		case IIO_VOLTAGE:
			*val = info->value;
			break;
		case IIO_TEMP:
			/*
			 * Calculate in degree Celsius times 1000
			 * Using sensor slope of 1.84 mV/°C and
			 * V at 25°C of 696 mV
			 */
			*val = 25000 - ((int)info->value - 864) * 1000000 /
			1840;
			break;
		default:
			mutex_unlock(&indio_dev->mlock);
			return -EINVAL;
		}

		mutex_unlock(&indio_dev->mlock);
		return IIO_VAL_INT;

	case IIO_CHAN_INFO_SCALE:
		*val = info->vref_uv / 1000;
		*val2 = info->adc_feature.res_mode;
		return IIO_VAL_FRACTIONAL_LOG2;

	case IIO_CHAN_INFO_SAMP_FREQ:
		*val = info->sample_freq_avail[info->
		adc_feature.sample_rate];
		*val2 = 0;
		return IIO_VAL_INT;

	default:
		break;
	}

	return -EINVAL;
}
```

第 12~49 行，读取 ADC 原始数据值，第 32 行 type 值为 `IIO_VOLTAGE`，也就是读取电压值。这里直接读取 vf610_adc 的 value 成员变量得到 ADC 转换结果，并没有看到读取 ADC 数据寄存器的过程。这是因为真正的ADC数据读取过程是在中断服务函数 `vf610_adc_isr`中完成。 第 51~54 行，返回 ADC 对应的分辨率。

#### 4、vf610_adc_isr 函数

函数内容如下：

**示例代码 76.2.2.5 vf610_adc_isr 函数**

```c
static irqreturn_t vf610_adc_isr(int irq, void *dev_id)
{
	struct vf610_adc *info = (struct vf610_adc *)dev_id;
	int coco;

	coco = readl(info->regs + VF610_REG_ADC_HS);
	if (coco & VF610_ADC_HS_COCO0) {
		info->value = vf610_adc_read_data(info);
		complete(&info->completion);
	}

	return IRQ_HANDLED;
}
```

可以看出，vf610_adc_isr 函数很简单，重点就是在第 8 行通过调用 `vf610_adc_read_data` 函数来读取 ADC 原始值，然后将 ADC 的原始值保存在 vf610_adc 的 value 成员变量里面。

## 76.3 硬件原理图分析

本试验用到的资源如下： 1）、指示灯 LED0。 2）、RGB LCD 接口。 3）、GPIO1_IO01 引脚

本实验主要用到 I.MX6ULL 的 GPIO1_IO01 引脚，将其作为 ADC1 的通道 1 引脚，正点原子 ALPHA 开发板上引出了 GPIO1_IO01 引脚，如图1-1所示，GPIO1 就是 GPIO1_IO01 引脚，此引脚作为 ADC1_CH1，我们可以使用杜邦线在此引脚上引入一个 0~3.3V 的电压，然后使用内部 ADC 进行测量。

![图1-1 GPIO1_IO01引脚](photo/Linux%20ADC驱动实验/1-1%20GPIO1_IO01引脚.png)

## 76.4 ADC 驱动编写

### 76.4.1 修改设备树

ADC 驱动 NXP 已经编写好了，我们只需要修改设备树即可。首先在 imx6ull-alientek-emmc.dts 文件中添加 ADC 使用的 GPIO1_IO01 引脚配置信息：

**示例代码 76.4.1.1 引脚配置信息**

```dts
pinctrl_adc1: adc1grp {
	fsl,pins = <
		MX6UL_PAD_GPIO1_IO01__GPIO1_IO01 0xb0
	>;
};
```

接下来在 imx6ull-alientek-emmc.dts 文件中的在 regulators 节点下添加参考电源子节点，内容如下：

**示例代码 76.4.1.2 reg_vref_adc 子节点**

```dts
reg_vref_adc: regulator@2 {
	compatible = "regulator-fixed";
	regulator-name = "VREF_3V3";
	regulator-min-microvolt = <3300000>;
	regulator-max-microvolt = <3300000>;
};
```

最后在 imx6ull-alientek-emmc.dts 文件中向 adc1 节点追加一些内容，内容如下：

**示例代码 76.4.1.3 adc1 节点**

```dts
&adc1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_adc1>;
	num-channels = <2>;
	vref-supply = <&reg_vref_adc>;
	status = "okay";
};
```

### 76.4.2 使能 ADC 驱动

使能内核里面自带的 I.MX6ULL ADC 驱动，打开 Linux 内核配置界面，配置路径如下：
```
-> Device Drivers
	-> Analog-to-Digital Converter (ADC) Support
		-> <*> i.MX ADC support
```

### 76.4.3 编写测试 APP

编译修改后的设备树，然后使用新的设备树启动系统。进入 `/sys/bus/iio/devices` 目录下，此目录下就有 ADC 对应的 iio 设备：iio:deviceX

"iio:device0"就是 ADC 设备，因为此时并没有加载其他的 IIO 设备驱动，只有一个 ADC。如果大家还加载了其他 IIO 设备驱动，那么就要依次进入 iio 设备目录，查看一下都对应的是什么设备。进入"iio:device0"目录，标准的 IIO 设备文件目录，我们只关心三个文件：
- in_voltage1_raw：ADC1 通道 1 原始值文件。
- in_voltage_scale：ADC1 比例文件(分辨率)，单位为 mV。实际电压值(mV)=in_voltage1_raw * in_voltage_scale

经过计算，图 76.4.3.3 中实际电压：991*0.805664062≈798.4mV，也就是 0.7984V。

接下来就编写测试 APP，新建 adcApp.c 文件，然后在里面输入如下所示内容：

**示例代码 76.4.3.1 adcApp.c 文件内容**

```c
/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名	: adcApp.c
作者 	: 正点原子 Linux 团队
版本 	: V1.0
描述 	: adc 测试应用文件。
其他 	: 无
使用方法 ：./adcApp
论坛 	: www.openedv.com
日志 	: 初版 V1.0 2021/09/24 正点原子 Linux 团队创建
***************************************************************/
#include "stdio.h"
#include "unistd.h"
#include "sys/types.h"
#include "sys/stat.h"
#include "sys/ioctl.h"
#include "fcntl.h"
#include "stdlib.h"
#include "string.h"
#include <poll.h>
#include <sys/select.h>
#include <sys/time.h>
#include <signal.h>
#include <fcntl.h>
#include <errno.h>

/* 字符串转数字，将浮点小数字符串转换为浮点数数值 */
#define SENSOR_FLOAT_DATA_GET(ret, index, str, member)\
ret = file_data_read(file_path[index], str);\
dev->member = atof(str);\

/* 字符串转数字，将整数字符串转换为整数数值 */
#define SENSOR_INT_DATA_GET(ret, index, str, member)\
ret = file_data_read(file_path[index], str);\
dev->member = atoi(str);\


/* adc iio 框架对应的文件路径 */
static char *file_path[] = {
	"/sys/bus/iio/devices/iio:device0/in_voltage_scale",
	"/sys/bus/iio/devices/iio:device0/in_voltage1_raw",
};

/* 文件路径索引，要和 file_path 里面的文件顺序对应 */
enum path_index {
	IN_VOLTAGE_SCALE = 0,
	IN_VOLTAGE_RAW,
};

/*
 * ADC 数据设备结构体
 */
struct adc_dev{
	int raw;
	float scale;
	float act;
};

struct adc_dev imx6ulladc;

/*
 * @description : 读取指定文件内容
 * @param – filename : 要读取的文件路径
 * @param - str : 读取到的文件字符串
 * @return : 0 成功;其他 失败
 */
static int file_data_read(char *filename, char *str)
{
	int ret = 0;
	FILE *data_stream;

	data_stream = fopen(filename, "r"); /* 只读打开 */
	if(data_stream == NULL) {
		printf("can't open file %s\r\n", filename);
		return -1;
	}

	ret = fscanf(data_stream, "%s", str);
	if(!ret) {
		printf("file read error!\r\n");
	} else if(ret == EOF) {
		/* 读到文件末尾的话将文件指针重新调整到文件头 */
		fseek(data_stream, 0, SEEK_SET);
	}
	fclose(data_stream); /* 关闭文件 */
	return 0;
}

/*
 * @description : 获取 ADC 数据
 * @param - dev : 设备结构体
 * @return : 0 成功;其他 失败
 */
static int adc_read(struct adc_dev *dev)
{
	int ret = 0;
	char str[50];

	SENSOR_FLOAT_DATA_GET(ret, IN_VOLTAGE_SCALE, str, scale);
	SENSOR_INT_DATA_GET(ret, IN_VOLTAGE_RAW, str, raw);

	/* 转换得到实际电压值 mV */
	dev->act = (dev->scale * dev->raw)/1000.f;
	return ret;
}

/*
 * @description : main 主程序
 * @param – argc : argv 数组元素个数
 * @param - argv : 具体参数
 * @return : 0 成功;其他 失败
 */
int main(int argc, char *argv[])
{
	int ret = 0;

	if (argc != 1) {
		printf("Error Usage!\r\n");
		return -1;
	}

	while (1) {
		ret = adc_read(&imx6ulladc);
		if(ret == 0) { /* 数据读取成功 */
			printf("ADC 原始值：%d，电压值：%.3fV\r\n", imx6ulladc.raw, imx6ulladc.act);
		}
		usleep(100000); /*100ms */
	}
	return 0;
}
```

adcApp.c 就是在上一章的应用程序上修改而来的，由于只读取一路 ADC，因此内容反而更简单，这里就不做介绍了。

## 76.5 运行测试

### 76.5.1 编译驱动程序和测试 APP

由于不需要我们编写 ADC 驱动程序，因此也就不需要编译驱动程序。设备树前面已经编译过了，所以这里就只剩下编译测试 APP。由于 adcApp.c 用到了浮点运算，因此我们编译的时候要使能硬件浮点，输入如下编译 adcApp.c 这个测试程序：

```bash
arm-linux-gnueabihf-gcc -march=armv7-a -mfpu=neon -mfloat-abi=hard adcApp.c -o adcApp
```

编译成功以后就会生成 adcApp 这个应用程序。

### 76.5.2 运行测试

注意，在测试之前一定要先用杜邦线将 GPIO1_IO01 引脚与要测量的电压连接起来，注意待测电压不能超过 3.3V！否则会烧坏开发板！

输入如下命令，使用 adcApp 测试程序：

```bash
./adcApp
```

测试 APP 会不断的读取 ADC 值并输出到终端，我们可以用杜邦线将 GPIO1_IO01 分别接到开发板的 3.3V 和 GND 上。
