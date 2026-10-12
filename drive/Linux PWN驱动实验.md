# 第七十三章 Linux PWM 驱动实验

在裸机篇我们已经学习过了如何使用 I.MX6ULL 的 PWM 外设来实现 LCD 的背光调节，其实在 Linux 的 LCD 驱动实验我们也提到过 I.MX6ULL 的 PWM 背光调节，但是并没有专门的去讲解 PWM 部分，本章我们就来学习一下 Linux 下的 PWM 驱动开发。

## 73.1 PWM 驱动简析

关于 PWM 原理以及 I.MX6ULL 的 PWM 外设已经在裸机篇进行了详细的讲解，这里就不再赘述了，我们重点来看一下 NXP 原厂提供的 Linux 内核自带的 PWM 驱动。

### 73.1.1 设备树下的 PWM 控制器节点

I.MX6ULL 有 8 路 PWM 输出，因此对应 8 个 PWM 控制器，所有在设备树下就有 8 个 PWM 控制器节点。这 8 路 PWM 都属于 I.MX6ULL 的 AIPS-1 域，但是在设备树 imx6ull.dtsi 中分为了两部分，PWM1PWM4 在一起，PWM5PWM8 在一起，这 8 路 PWM 并没有全部放到一起，这一点一定要注意，不要以为 imx6ull.dtsi 没有写完整。这 8 路 PWM 的设备树节点内容都是一样的，除了 reg 属性不同(毕竟不同的控制器，其地址范围不同)。本章实验我们使用 GPIO1_IO04 这个引脚来完成 PWM 实验，而 GPIO1_IO04 就是 PWM3 的输出引脚，所以这里我们就以 PWM3 为例进行讲解，imx6ull.dtsi 文件中的 pwm3 节点信息如下：

**示例代码 73.1.1.1 pwm3 节点内容**

```dts
pwm3: pwm@02088000 {
	compatible = "fsl,imx6ul-pwm", "fsl,imx27-pwm";
	reg = <0x02088000 0x4000>;
	interrupts = <GIC_SPI 85 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&clks IMX6UL_CLK_PWM3>,
			 <&clks IMX6UL_CLK_PWM3>;
	clock-names = "ipg", "per";
	#pwm-cells = <2>;
};
```

第 2 行，compatible 属性值有两个"fsl,imx6ul-pwm"和"fsl,imx27-pwm"，所以在整个 Linux 源码里面搜索这两个字符串即可找到 I.MX6ULL 的 PWM 驱动文件，这个文件就是 drivers/pwm/pwm-imx.c。

关于 I.MX6ULL 的 PWM 节点更为详细的信息请参考对应的绑定文档：Documentation/devicetree/bindings/pwm/imx-pwm.txt，这里就不去分析了。

### 73.1.2 PWM 子系统

Linux 内核提供了个 PWM 子系统框架，编写 PWM 驱动的时候一定要符合这个框架。PWM 子系统的核心是 pwm_chip 结构体，定义在文件 include/linux/pwm.h 中，定义如下：

**示例代码 73.1.2.1 pwm_chip 结构体**

```c
struct pwm_chip {
	struct device		*dev;
	struct list_head	list;
	const struct pwm_ops *ops;
	int			base;
	unsigned int		npwm;
	struct pwm_device	*pwms;
	struct pwm_device	* (*of_xlate)(struct pwm_chip *pc,
		const struct of_phandle_args *args);
	unsigned int		of_pwm_n_cells;
	bool			can_sleep;
};
```

第 4 行，pwm_ops 结构体就是 PWM 外设的各种操作函数集合，编写 PWM 外设驱动的时候需要开发人员实现。pwm_ops 结构体也定义在 pwm.h 头文件中，定义如下：

**示例代码 73.1.2.2 pwm_ops 结构体**

```c
struct pwm_ops {
	int	(*request)(struct pwm_chip *chip,	//请求 PWM
			   struct pwm_device *pwm);
	void (*free)(struct pwm_chip *chip,		//释放 PWM
		     struct pwm_device *pwm);
	int (*config)(struct pwm_chip *chip,	//配置 PWM 周期和占空比
		     struct pwm_device *pwm,
		     int duty_ns, int period_ns);
	int (*set_polarity)(struct pwm_chip *chip, //设置 PWM 极性
			    struct pwm_device *pwm,
			    enum pwm_polarity polarity);
	int (*enable)(struct pwm_chip *chip,	//使能 PWM
		      struct pwm_device *pwm);
	void (*disable)(struct pwm_chip *chip,	//关闭 PWM
			struct pwm_device *pwm);
	struct module *owner;
};
```

pwm_ops 中的这些函数不一定全部实现，但是像 config、enable 和 disable 这些肯定是需要实现的，否则的话打开/关闭 PWM，设置 PWM 的占空比这些就没操作了。

PWM 子系统驱动的核心就是初始化 pwm_chip 结构体各成员变量，然后向内核注册初始化完成以后的 pwm_chip。这里就要用到 pwmchip_add 函数，此函数定义在 drivers/pwm/core.c 文件中，函数原型如下：

```c
int pwmchip_add(struct pwm_chip *chip)
```

函数参数和返回值含义如下：
- chip：要向内核注册的 pwm_chip。
- 返回值：0 成功；负数 失败。

卸载 PWM 驱动的时候需要将前面注册的 pwm_chip 从内核移除掉，这里要用到 pwmchip_remove 函数，函数原型如下：

```c
int pwmchip_remove(struct pwm_chip *chip)
```

函数参数和返回值含义如下：
- chip：要移除的 pwm_chip。
- 返回值：0 成功；负数 失败。

### 73.1.3 PWM 驱动源码分析

我们简单分析一下 Linux 内核自带的 I.MX6ULL PWM 驱动，驱动文件前面都说了，是 pwm-imx.c 这个文件。打开这个文件，可以看到，这是一个标准的平台设备驱动文件，如下所示：

**示例代码 73.1.3.1 I.MX6ULL PWM 平台驱动**

```c
static const struct of_device_id imx_pwm_dt_ids[] = {
	{ .compatible = "fsl,imx1-pwm", .data = &imx_pwm_data_v1, },
	{ .compatible = "fsl,imx27-pwm", .data = &imx_pwm_data_v2, },
	{ /* sentinel */ }
};

......

static struct platform_driver imx_pwm_driver = {
	.driver = {
		.name = "imx-pwm",
		.of_match_table = imx_pwm_dt_ids,
	},
	.probe = imx_pwm_probe,
	.remove = imx_pwm_remove,
};

module_platform_driver(imx_pwm_driver);
```

第 3 行，当设备树 PWM 节点的 compatible 属性值为"fsl,imx27-pwm"的话就会匹配此驱动，注意后面的.data 为 imx_pwm_data_v2，这是一个 imx_pwm_data 类型的结构体变量，内容如下：

**示例代码 73.1.3.2 imx_pwm_data_v2 结构体变量**

```c
static struct imx_pwm_data imx_pwm_data_v2 = {
	.config = imx_pwm_config_v2,
	.set_enable = imx_pwm_set_enable_v2,
};
```

imx_pwm_config_v2 函数就是最终操作 I.MX6ULL 的 PWM 外设寄存器，进行实际配置的函数。imx_pwm_set_enable_v2 就是具体使能 PWM 的函数。

第 14 行，当设备树节点和驱动匹配以后 imx_pwm_probe 函数就会执行。 imx_pwm_probe 函数如下(有缩减)：

**示例代码 73.1.3.3 imx_pwm_probe 函数**

```c
static int imx_pwm_probe(struct platform_device *pdev)
{
	const struct of_device_id *of_id =
		of_match_device(imx_pwm_dt_ids, &pdev->dev);
	const struct imx_pwm_data *data;
	struct imx_chip *imx;
	struct resource *r;
	int ret = 0;

	if (!of_id)
		return -ENODEV;

	imx = devm_kzalloc(&pdev->dev, sizeof(*imx), GFP_KERNEL);
	if (imx == NULL)
		return -ENOMEM;
......
	imx->chip.ops = &imx_pwm_ops;
	imx->chip.dev = &pdev->dev;
	imx->chip.base = -1;
	imx->chip.npwm = 1;
	imx->chip.can_sleep = true;

	r = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	imx->mmio_base = devm_ioremap_resource(&pdev->dev, r);
	if (IS_ERR(imx->mmio_base))
		return PTR_ERR(imx->mmio_base);

	data = of_id->data;
	imx->config = data->config;
	imx->set_enable = data->set_enable;

	ret = pwmchip_add(&imx->chip);
	if (ret < 0)
		return ret;

	platform_set_drvdata(pdev, imx);
	return 0;
}
```

第 13 行，imx 是一个 imx_chip 类型的结构体指针变量，这里为其申请内存。imx_chip 结构体有个重要的成员变量 chip，chip 是 pwm_chip 类型的。所以这一行就引出了 PWM 子系统核心部件 pwm_chip，稍后的重点就是初始化 chip。

第 31~35 行，初始化 imx 的 chip 成员变量，也就是初始化 pwm_chip！第 31 行设置 pwm_chip 的 ops 操作集为 imx_pwm_ops，imx_pwm_ops 定义如下：

**示例代码 73.1.3.4 imx_pwm_ops 操作集合**

```c
static struct pwm_ops imx_pwm_ops = {
	.enable = imx_pwm_enable,
	.disable = imx_pwm_disable,
	.config = imx_pwm_config,
	.owner = THIS_MODULE,
};
```

imx_pwm_enable、imx_pwm_disable 和 imx_pwm_config 这三个函数就是使能、关闭和配置 PWM 的函数。

继续回到示例代码 73.1.3.3 中的 37 和 38 行，从设备树中获取 PWM 节点中关于 PWM 控制器的地址信息，然后再进行内存映射，这样我们就得到了 PWM 控制器的基地址。

第 43 和 44 行，这两行设置 imx 的 config 和 set_enable 这两个成员变量为 data->config 和 data->set_enable，也就是示例代码 73.1.3.2 中的 imx_pwm_config_v2 和 imx_pwm_set_enable_v2 这两个函数。imx_pwm_enable、imx_pwm_disable 和 imx_pwm_config 这三个函数最终调用就是 imx_pwm_config_v2 和 imx_pwm_set_enable_v2。

整个 pwm-imx.c 文件里面，最终和 I.MX6ULL 的 PWM 寄存器打交道的就是 imx_pwm_config_v2 和 imx_pwm_set_enable_v2 这两个函数，我们先来看一下 imx_pwm_set_enable_v2 函数，此函数用于打开或关闭对应的 PWM，函数内容如下：

**示例代码 73.1.3.5 imx_pwm_set_enable_v2 函数**

```c
static void imx_pwm_set_enable_v2(struct pwm_chip *chip, bool enable)
{
	struct imx_chip *imx = to_imx_chip(chip);
	u32 val;

	val = readl(imx->mmio_base + MX3_PWMCR);

	if (enable)
		val |= MX3_PWMCR_EN;
	else
		val &= ~MX3_PWMCR_EN;

	writel(val, imx->mmio_base + MX3_PWMCR);
}
```

第 6 行，读取 PWMCR 寄存器的值。 第 9 行，如果 enable 为真，表示使能 PWM，将 PWMCR 寄存器的 bit0 置 1 即可，宏 MX3_PWMCR_EN 为(1<<0)。 第 11 行，如果 enable 不为真，表示关闭 PWM，将 PWMCR 寄存器的 bit0 清 0 即可。 第 13 行，将新的 val 值写入到 PWMCR 寄存器中。

imx_pwm_config_v2 函数用于设置 PWM 的频率和占空比，相关操作如下：

**示例代码 73.1.3.6 imx_pwm_config_v2 函数**

```c
static int imx_pwm_config_v2(struct pwm_chip *chip,
	struct pwm_device *pwm, int duty_ns, int period_ns)
{
	struct imx_chip *imx = to_imx_chip(chip);
	struct device *dev = chip->dev;
	unsigned long long c;
	unsigned long period_cycles, duty_cycles, prescale;
	unsigned int period_ms;
	bool enable = test_bit(PWMF_ENABLED, &pwm->flags);
	int wait_count = 0, fifoav;
	u32 cr, sr;
......

	c = clk_get_rate(imx->clk_per);
	c = c * period_ns;
	do_div(c, 1000000000);
	period_cycles = c;

	prescale = period_cycles / 0x10000 + 1;

	period_cycles /= prescale;
	c = (unsigned long long)period_cycles * duty_ns;
	do_div(c, period_ns);
	duty_cycles = c;

 /*
 * according to imx pwm RM, the real period value should be
 * PERIOD value in PWMPR plus 2.
 */
	if (period_cycles > 2)
		period_cycles -= 2;
	else
		period_cycles = 0;

	writel(duty_cycles, imx->mmio_base + MX3_PWMSAR);
	writel(period_cycles, imx->mmio_base + MX3_PWMPR);

	cr = MX3_PWMCR_PRESCALER(prescale) |
		MX3_PWMCR_DOZEEN | MX3_PWMCR_WAITEN |
		MX3_PWMCR_DBGEN | MX3_PWMCR_CLKSRC_IPG_HIGH;

	if (enable)
		cr |= MX3_PWMCR_EN;

	writel(cr, imx->mmio_base + MX3_PWMCR);

	return 0;
}
```

第 43~62 行，根据参数 duty_ns 和 period_ns 来计算出应该写入到寄存器里面的值 duty_cycles 和 period_cycles。 第 64 行，将计算得到的 duty_cycles 写入到 PWMSAR 寄存器中，设置 PWM 的占空比。 第 65 行，将计算得到的 period_cycles 写入到 PWMPR 寄存器中，设置 PWM 的频率。

至此，I.MX6ULL 的 PWM 驱动我们就分析完了。

## 73.2 PWM 驱动编写

### 73.2.1 修改设备树

PWM 驱动就不需要我们再编写了，NXP 已经写好了，前面我们也已经详细的分析过这个驱动源码了。我们在实际使用的时候只需要修改设备树即可，ALPHA 开发板上的 JP2 排针引出了 GPIO1_IO04 这个引脚。 GPIO1_IO04 可以作为 PWM3 的输出引脚，所以我们需要在设备树里面添加 GPIO1_IO04 的引脚信息以及 PWM3 控制器对应的节点信息。

#### 1、添加 GPIO1_IO04 引脚信息
打开 imx6ull-alientek-emmc.dts 文件，在 iomuxc 节点下添加 GPIO1_IO04 的引脚信息，如下所示：

**示例代码 73.2.1.1 GPIO1_IO04 引脚信息**

```dts
pinctrl_pwm3: pwm3grp {
	fsl,pins = <
		MX6UL_PAD_GPIO1_IO04__PWM3_OUT 0x110b0
	>;
};
```

#### 2、向 pwm3 节点追加信息
前面已经讲过了，imx6ull.dtsi 文件中已经有了"pwm3"节点，但是还不能直接使用，需要在 imx6ull-alientek-emmc.dts 文件中向 pwm3 节点追加一些内容，在 imx6ull-alientek-emmc.dts 文件中加入如下所示内容：

**示例代码 73.2.1.2 向 pwm3 添加的内容**

```dts
&pwm3 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_pwm3>;
	clocks = <&clks IMX6UL_CLK_PWM3>,
			 <&clks IMX6UL_CLK_PWM3>;
	status = "okay";
};
```

第 3 行，pinctrl-0 属性指定 PWM3 所使用的输出引脚对应的 pinctrl 节点，这里设置为示例代码 73.2.1.1 中的 pinctrl_pwm3。 第 4 和 5 行，设置时钟，第 4 行设置 ipg 时钟，第 5 行设置 per 时钟。有些 pwm 节点默认时钟源是 IMX6UL_CLK_DUMMY，这里我们需要将其改为对应的时钟，比如这里设置为 IMX6UL_CLK_PWM3。PWM1~PWM8 分别对应 IMX6UL_CLK_PWM1 ~ IMX6UL_CLK_PWM8。

#### 3、屏蔽掉其他复用的 IO
检查一下设备树中有没有其他外设用到 GPIO1_IO04，如果有的话需要屏蔽掉！注意，不能只屏蔽掉 GPIO1_IO04 的 pinctrl 配置信息，也要搜索一下"gpio1 4"，看看有没有哪里用到，用到的话也要屏蔽掉。

设备树修改完成以后重新编译设备树，然后使用新的设备树启动系统。

### 73.2.2 使能 PWM 驱动

NXP 官方的 Linux 内核已经默认使能了 PWM 驱动，所以不需要我们修改，但是为了学习，我们还是需要知道怎么使能。打开 Linux 内核配置界面，按照如下路径找到配置项：
```
-> Device Drivers
	-> Pulse‑Width Modulation (PWM) Support
		-> <*> i.MX PWM support
```
配置如图1-3。

## 73.3 PWM 驱动测试

使用新的设备树启动系统，然后将开发板 JP2 排针上的 GPIO_4(GPIO1_IO04)引脚连接到示波器上，通过示波器来查看 PWM 波形图。我们可以直接在用户层来配置 PWM，进入目录/sys/class/pwm中，pwmchip0~pwmchip7 对应 I.MX6ULL 的 PWM1~PWM8，所以我们需要用到 pwmchip2。

#### 1、调出 pwmchip2 的 pwm0 子目录
首先需要调出 pwmchip2 下的 pwm0 目录，否则后续就没法操作了，输入如下命令：
```bash
echo 0 > /sys/class/pwm/pwmchip2/export
```
执行完成会在 pwmchip2 目录下生成一个名为"pwm0"的子目录。

#### 2、使能 PWM3
输入如下命令使能 PWM3：
```bash
echo 1 > /sys/class/pwm/pwmchip2/pwm0/enable
```

#### 3、设置 PWM3 的频率
注意，这里设置的是周期值，单位为 ns，比如 20KHz 频率的周期就是 50000ns，输入如下命令：
```bash
echo 50000 > /sys/class/pwm/pwmchip2/pwm0/period
```

#### 4、设置 PWM3 的占空比
这里不能直接设置占空比，而是设置的一个周期的 ON 时间，也就是高电平时间，比如 20KHz 频率下 20%占空比的 ON 时间就是 10000，输入如下命令：
```bash
echo 10000 > /sys/class/pwm/pwmchip2/pwm0/duty_cycle
```

## 73.4 PWM 背光设置

有时候我们需要在某个外设上添加 PWM 功能，比如，LCD 的背光控制就是 PWM 来完成的，本小节我们就以 PWM 背光控制为例，学习一下如何在其他外设上添加 PWM 功能。首先肯定是设备树描述，直接看 linux 内核里面关于 backlight(背光)的绑定文档，路径为 Documentation/devicetree/bindings/video/backlight/pwm-backlight.txt，此文档描述了如何创建 backlight 节点来使用 linux 内核自带的 pwm 背光驱动。

必要的属性如下：
- compatible：内容必须为"pwm-backlight"，通过这个可以匹配到内核自带的 PWM 背光驱动，驱动文件为 drivers/video/backlight/pwm_bl.c，这里就不去分析驱动源码了。
- pwms：此属性指定背光使用哪一路 PWM，以及 PWM 相关的属性。
- brightness-levels：背光等级数组，范围 0~255，对应占空比为 0%~100%。数组内的值必须从 0 开始，也就是 0%占空比，最后一个值必须是 255，也就是 100%占空比。数组中间值的个数以及值大小可以自行定义。
- default-brightness-level：默认的背光等级，也就是 brightness-levels 属性中第几个值，注意这里是数索引编号，不是具体的数值！
- power-supply：支持的电压，此属性可以不需要。

以正点原子 ALPHA 开发板为例，看一下 PWM 背光节点是如何设置的，打开 imx6ull-alientek-emmc.dts，找到如下所示节点内容：

**示例代码 73.4.1.1 pwm 背光节点**

```dts
backlight {
	compatible = "pwm-backlight";
	pwms = <&pwm1 0 5000000>;
	brightness-levels = <0 4 8 16 32 64 128 255>;
	default-brightness-level = <7>;
	status = "okay";
};
```

第 2 行，compatible 属性必须为"pwm-backlight"。 第 3 行，pwms 属性指定背光所使用的 pwm 通道，第一个参数指定使用 pwm1，由于 I.MX6ULL 的 PWM 只有一个通道，因此这里为 0。最后一个参数是 PWM 周期，单位为 ns，这里 PWM 周期为 5000000ns，频率为 200Hz。 第 4 行，背光等级数组，一共 8 个等级，索引编号从 0 到 7。 第 5 行，背光默认处于第 7 等级，也就是 255，为 100%占空比。

关于 pwm 做背光控制就讲解到这里，pwm 做其他外设某个功能的时候要具体问题具体分析。