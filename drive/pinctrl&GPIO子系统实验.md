# 第 45 章 pinctrl & GPIO 子系统

pinctrl 子系统负责 PIN 引脚复用 + 电气属性；GPIO 子系统负责 GPIO 方向、电平读写。把引脚硬件配置交给内核标准子系统，驱动代码不再直接操作 IOMUX 寄存器，实现驱动分层分离。

---

## 45.1 pinctrl 子系统

### 45.1.1 pinctrl 子系统简介

**传统方式（上一章 dtsled）操作 GPIO 的两步：**

1. 配置 PIN：复用功能、上下拉、驱动能力、速度（IOMUX 寄存器）
2. 配置 GPIO：方向（输入/输出）、高低电平（GPIO 模块寄存器）

几乎所有 SOC 都是这个套路：PIN 配置和 GPIO 配置是两套独立寄存器。STM32 同理：先配置 PIN 复用/上下拉，再配置 GPIO 口方向。

**对驱动开发者：** 只需要在 DTS 写 pin 配置，内核 pinctrl 驱动帮你完成寄存器初始化，驱动代码不用操作 IOMUX/PAD 寄存器。源码路径：`drivers/pinctrl`。

**区分两个概念：**

| 概念 | 负责内容 | 管理子系统 |
|------|----------|------------|
| **PIN（引脚）** | 芯片物理引脚，比如 GPIO1_IO03；PIN 的复用、电气属性 | pinctrl 子系统 |
| **GPIO** | GPIO 控制器，用来设置方向、读/写引脚电平 | gpio 子系统 |

> pinctrl 只管引脚复用和电气特性；不管 GPIO 输入输出、电平。pinctrl 把 PIN 复成为 GPIO 之后，才交给 GPIO 子系统接管。

**44 章 dtsled vs 45 章 pinctrl+gpio 子系统驱动对比：**

| 项目 | 44 章 dtsled（裸机式设备树驱动） | 45 章 pinctrl+gpio 子系统驱动 |
|------|--------------------------------|-------------------------------|
| PIN 复用/PAD 配置 | 驱动代码中 `of_iomap` 拿到 IOMUX 寄存器，`writel` 直接写寄存器 | DTS 中写 pinctrl 节点，内核 pinctrl 子系统自动完成 |
| GPIO 方向、电平 | 驱动代码操作 GPIO1_DR/GDIR 寄存器 | 使用 GPIO 子系统标准 API：`gpio_request`、`gpio_direction_output`、`gpio_set_value` |
| 寄存器操作 | 驱动大量直接读写寄存器 | 驱动不再操作 IOMUX、PAD 寄存器，只调用内核 GPIO 标准接口 |
| 优点 | 简单好理解，贴近裸机 | 代码简洁、标准化、防止引脚冲突，符合 Linux 驱动规范 |
| 缺点 | 代码耦合硬件寄存器，不规范 | 需要学会 pinctrl 设备树语法、GPIO 子系统 API |

### 45.1.2 I.MX6ULL 的 pinctrl 子系统驱动

#### 1）PIN 配置信息详解

**① iomuxc 根节点（imx6ull.dtsi 中）**

```c
iomuxc: iomuxc@020e0000 {
    compatible = "fsl,imx6ul-iomuxc";
    reg = <0x020e0000 0x4000>;
};
```

- `iomuxc`：IOMUXC 外设的设备树根节点，定义在 dtsi（SOC 公共头文件）
- `compatible = "fsl,imx6ul-iomuxc"`：匹配内核 pinctrl 驱动的关键字
- `reg`：IOMUXC 寄存器物理基地址 `0x020e0000`，寄存器区域长度 `0x4000`

> 这个根节点只描述 IOMUXC 硬件本身，不包含任何引脚配置；引脚分组配置写在板级 dts 中，使用 `&iomuxc` 追加子节点。

**② 板级 dts 追加引脚组（imx6ull-alientek-emmc.dts）**

```c
&iomuxc {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_hog_1>;
    imx6ul-evk {
        pinctrl_hog_1: hoggrp-1 {
            fsl,pins = <
                MX6UL_PAD_UART1_RTS_B__GPIO1_IO19 0x17059
                MX6UL_PAD_GPIO1_IO05__USDHC1_VSELECT 0x17059
                MX6UL_PAD_GPIO1_IO09__GPIO1_IO09 0x17059
                MX6UL_PAD_GPIO1_IO00__ANATOP_OTG1_ID 0x13058
            >;
        };
        pinctrl_flexcan1: flexcan1grp {
            fsl,pins = <
                MX6UL_PAD_UART3_RTS_B__FLEXCAN1_RX 0x1b020
                MX6UL_PAD_UART3_CTS_B__FLEXCAN1_TX 0x1b020
            >;
        };
        pinctrl_wdog: wdoggrp {
            fsl,pins = <
                MX6UL_PAD_LCD_RESET__WDOG1_WDOG_ANY 0x30b0
            >;
        };
    };
};
```

- `&iomuxc`：引用 dtsi 里面的 iomuxc 节点，追加板上外设引脚配置，不覆盖原有内容
- 大括号内的 `imx6ul-evk` 是子容器，里面可以新建多个 pin 组子节点
- 每个 pin 组子节点对应一个外设用到的一组引脚：
  - `pinctrl_hog_1`：热插拔相关引脚（SD 卡检测、OTG ID）
  - `pinctrl_flexcan1`：CAN1 外设引脚
  - `pinctrl_wdog`：看门狗引脚
  - 自定义外设引脚：新建一个子节点，把该外设全部引脚写在 `fsl,pins` 属性里面

> 完整 iomuxc 节点 = dtsi 的基础 iomuxc 节点 + 板级 dts 通过 `&iomuxc` 追加的各个 pin 组。

**③ fsl,pins 单条引脚配置解析**
```bash
MX6UL_PAD_UART1_RTS_B__GPIO1_IO19 0x17059
```

一行由两部分组成：
- 宏 `MX6UL_PAD_UART1_RTS_B__GPIO1_IO19`
- 数值 `0x17059` → PAD 配置寄存器（conf_reg）的值，设置电气属性

**宏定义位置：** `arch/arm/boot/dts/imx6ul-pinfunc.h`，dts 可以直接引用头文件宏。

```c
#define MX6UL_PAD_UART1_RTS_B__GPIO1_IO19 0x0090 0x031C 0x0000 0x5 0x0
// 参数含义：<mux_reg conf_reg input_reg mux_mode input_val>
```

| 参数 | 值 | 含义 |
|------|-----|------|
| mux_reg | `0x0090` | 复用寄存器偏移；IOMUXC 基址 `0x020e0000` + `0x0090` = `0x020e0090`，即 IOMUXC_SW_MUX_CTL_PAD_UART1_RTS_B |
| conf_reg | `0x031C` | PAD 寄存器偏移；`0x020e0000` + `0x031C` = `0x020e031c` |
| input_reg | `0x0000` | 输入选择寄存器偏移，GPIO 功能时无效 |
| mux_mode | `0x5` | 复用寄存器写入值：把 PIN 复用成 GPIO1_IO19 |
| input_val | `0x0` | input_reg 写入值，GPIO 功能无效 |

> 宏里面没有 PAD 电气配置值，这个值写在宏后面，也就是 `0x17059`，写入 conf_reg 寄存器，配置上下拉、驱动能力、速度。

#### 2）PIN 驱动程序讲解（pinctrl-imx6ul.c）

这一小节涉及平台驱动、驱动分离分层，初学可以先了解流程，暂时不用深挖源码细节，不影响 gpioled 实验。

**① 设备树和驱动匹配 of_device_id**

文件：`drivers/pinctrl/freescale/pinctrl-imx6ul.c`

```c
static struct of_device_id imx6ul_pinctrl_of_match[] = {
    { .compatible = "fsl,imx6ul-iomuxc", .data = &imx6ul_pinctrl_info, },
    { .compatible = "fsl,imx6ull-iomuxc-snvs", .data = &imx6ull_snvs_pinctrl_info, },
    { /* sentinel */ }
};

static int imx6ul_pinctrl_probe(struct platform_device *pdev)
{
    const struct of_device_id *match;
    struct imx_pinctrl_soc_info *pinctrl_info;

    match = of_match_device(imx6ul_pinctrl_of_match, &pdev->dev);
    if (!match)
        return -ENODEV;

    pinctrl_info = (struct imx_pinctrl_soc_info *) match->data;
    return imx_pinctrl_probe(pdev, pinctrl_info);
}

static struct platform_driver imx6ul_pinctrl_driver = {
    .driver = {
        .name = "imx6ul-pinctrl",
        .owner = THIS_MODULE,
        .of_match_table = of_match_ptr(imx6ul_pinctrl_of_match),
    },
    .probe = imx6ul_pinctrl_probe,
    .remove = imx_pinctrl_remove,
};
```

1. `of_device_id` 数组：保存 compatible 匹配字符串，设备树 iomuxc 节点的 compatible 和这里匹配成功，驱动就会加载
2. `platform_driver` 平台驱动：设备树节点和驱动匹配成功，自动执行 probe 函数
3. `imx6ul_pinctrl_probe`：I.MX6ULL pinctrl 驱动入口函数

**② 函数调用链路**
```
imx6ul_pinctrl_probe()
    → imx_pinctrl_probe()
        → imx_pinctrl_probe_dt()  //解析设备树PIN配置
            → imx_pinctrl_parse_groups() //解析fsl,pins里面每一条pin信息
        → pinctrl_register() //向内核注册PIN控制器
```
`imx_pinctrl_parse_groups` 源码要点：
```c
static int imx_pinctrl_parse_groups(struct device_node *np,
struct imx_pin_group *grp,
struct imx_pinctrl_soc_info *info,
u32 index)
{
    //每一条pin：5个u32来自宏 + 1个u32 config，共6个u32（24字节）
    #define FSL_PIN_SIZE 24
    for (i = 0; i < grp->npins; i++) {
        u32 mux_reg = be32_to_cpu(*list++);
        u32 conf_reg;
        pin_reg->mux_reg = mux_reg;
        pin_reg->conf_reg = conf_reg;
        pin->input_reg = be32_to_cpu(*list++);
        pin->mux_mode = be32_to_cpu(*list++);
        pin->input_val = be32_to_cpu(*list++);
        config = be32_to_cpu(*list++);
        pin->config = config & ~IMX_PAD_SION;
    }
}
```

循环读取 `fsl,pins` 属性里的 6 个 u32：mux_reg、conf_reg、input_reg、mux_mode、input_val、config。保存寄存器偏移、复用值、PAD 配置值到内核数据结构，后续底层硬件操作函数使用。
**③ pinctrl_desc 结构体：PIN 控制器描述符**
```c
struct pinctrl_desc {
    const char *name;
    struct pinctrl_pin_desc const *pins;
    unsigned int npins;
    const struct pinctrl_ops *pctlops;    //pin组管理操作
    const struct pinmux_ops *pmxops;     //引脚复用操作
    const struct pinconf_ops *confops;    //引脚电气配置操作
    struct module *owner;
};
```

三个核心操作集 ops：

| ops | 负责功能 |
|-----|---------|
| `pctlops` | pin group 分组管理、debug 打印 |
| `pmxops` | 设置引脚复用功能、GPIO 方向 |
| `confops` | 读写引脚 PAD 电气配置（上下拉、驱动能力） |

> 这一套是 SOC 厂商（NXP）在内核中写好的，我们作为驱动开发者不需要自己实现这三个 ops，我们只需要在设备树写好 pin 配置即可。

在 `imx_pinctrl_probe` 内部：
1. 分配 `imx_pinctrl_desc` 内存
2. 填充 `pctlops / pmxops / confops`，赋值为 NXP 实现好的操作函数
3. 调用 `pinctrl_register`，向 Linux 内核注册 PIN 控制器，内核就拥有操作 I.MX6ULL 所有 PIN 的能力。

### 45.1.3 设备树中添加 pinctrl 节点

学会在 I.MX6ULL 板级设备树（imx6ull-alientek-emmc.dts）里，给自定义外设新增 pinctrl 引脚组。I.MX 系列 pinctrl 绑定文档参考：`Documentation/devicetree/bindings/pinctrl/fsl,imx-pinctrl.txt`

**操作位置：** `&iomuxc` → `imx6ul-evk` 内部，新增引脚组子节点

**步骤 1：创建 pinctrl 子节点**

同一个外设所有引脚放在同一个子节点内。节点命名规范：`pinctrl_xxx`，标签名 `xxxgrp`。

```c
pinctrl_test: testgrp {
    /* 具体 PIN 配置写在这里 */
};
```

- `pinctrl_test`：节点标签，后面外设节点要引用 `&pinctrl_test`
- `testgrp`：节点名字，随便取，一般 grp 代表 group 引脚组

**步骤 2：添加固定属性 fsl,pins**

属性名必须严格叫 `fsl,pins`。NXP 的 pinctrl 驱动就是读取这个属性，拿到引脚配置信息，不能改名字。

```c
pinctrl_test: testgrp {
    fsl,pins = <
        /* 填写一条或多条引脚配置 */
    >;
};
```

**步骤 3：填充 fsl,pins 里面的 PIN 配置**

格式：`宏定义 PAD配置值`

```c
pinctrl_test: testgrp {
    fsl,pins = <
        MX6UL_PAD_GPIO1_IO00__GPIO1_IO00  0x10b0
    >;
};
```
MX6UL_PAD_GPIO1_IO00__GPIO1_IO00：宏，来自imx6ul-pinfunc.h，指定引脚 + 复用功能（这里复用为 GPIO1_IO00）
0x10b0：conf 寄存器的值，设置引脚上下拉、驱动能力、速率（替换教材里的 config 占位符）
```c
&iomuxc {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_hog_1>;
    imx6ul-evk {
        pinctrl_hog_1: hoggrp-1 {
            fsl,pins = <
                MX6UL_PAD_UART1_RTS_B__GPIO1_IO19 0x17059
            >;
        };

        /*===== 我们新增的test外设引脚组 =====*/
        pinctrl_test: testgrp {
            fsl,pins = <
                MX6UL_PAD_GPIO1_IO00__GPIO1_IO00 0x10b0
            >;
        };

    };
};
```

---

## 45.2 GPIO 子系统

### 45.2.1 gpio 子系统简介

- **pinctrl**：负责 PIN/PAD 引脚复用、电气属性配置（把引脚配置成 GPIO 功能）
- **GPIO 子系统**：在 PIN 已经复用为 GPIO 之后，配置 GPIO 输入/输出、读写电平、GPIO 中断，对外提供统一 API。

### 45.2.2 I.MX6ULL 的 gpio 子系统驱动

#### 1）设备树中的 GPIO 信息（以 SD 卡检测引脚示例：GPIO1_IO19）

**① 第一步：pinctrl 配置（把引脚复用成 GPIO）**

在 `&iomuxc` 的 `imx6ul-evk` 节点内：

```c
pinctrl_hog_1: hoggrp-1 {
    fsl,pins = <
        MX6UL_PAD_UART1_RTS_B__GPIO1_IO19 0x17059 /* SD1 CD */
    >;
};
```

> `pinctrl_hog_1` 在 iomuxc 节点被引用，内核 iomux 驱动自动完成这一组引脚的初始化，不需要外设节点单独引用这个 pinctrl 组。

**② 第二步：外设节点内，添加 xxx-gpio 属性描述 GPIO**

`usdhc1`（SD 卡）设备节点：

```c
&usdhc1 {
    pinctrl-names = "default", "state_100mhz", "state_200mhz";
    pinctrl-0 = <&pinctrl_usdhc1>;
    pinctrl-1 = <&pinctrl_usdhc1_100mhz>;
    pinctrl-2 = <&pinctrl_usdhc1_200mhz>;
    cd-gpios = <&gpio1 19 GPIO_ACTIVE_LOW>;
    keep-power-in-suspend;
    enable-sdio-wakeup;
    vmmc-supply = <&reg_sd1_vmmc>;
    status = "okay";
};
```

`cd-gpios = <&gpio1 19 GPIO_ACTIVE_LOW>` 三参数含义：

| 参数 | 值 | 含义 |
|------|-----|------|
| 1 | `&gpio1` | 指向 GPIO 控制器 gpio1 节点 |
| 2 | `19` | GPIO1 组内的 IO 编号，GPIO1_IO19 |
| 3 | `GPIO_ACTIVE_LOW` | 低电平有效；`GPIO_ACTIVE_HIGH` 高电平有效 |

> 命名规范：检测/控制引脚属性一般命名 `xxx-gpios`，内核驱动可以用 `of_get_named_gpio` 读取这个属性。

**③ GPIO 控制器节点 gpio1（定义在 imx6ull.dtsi）**

```c
gpio1: gpio@0209c000 {
    compatible = "fsl,imx6ul-gpio", "fsl,imx35-gpio";
    reg = <0x0209c000 0x4000>;
    interrupts = <GIC_SPI 66 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 67 IRQ_TYPE_LEVEL_HIGH>;
    gpio-controller;         /* 标记：这是一个GPIO控制器 */
    #gpio-cells = <2>;        /* 每一条gpio属性有2个cell：IO编号 + 极性 */
    interrupt-controller;     /* GPIO可作为中断控制器 */
    #interrupt-cells = <2>;
};
```

- `reg = <0x0209c000 0x4000>`：GPIO1 寄存器物理基地址 `0x0209C000`，和参考手册寄存器表完全对应
- `#gpio-cells = <2>`：`<&gpio1 n 极性>`，两个参数
- `compatible`：`fsl,imx6ul-gpio` / `fsl,imx35-gpio`，用来匹配内核 `gpio-mxc.c` 驱动

**配套寄存器表：**

| 绝对物理地址 | 寄存器名称 | 作用 | 访问属性 |
|-------------|-----------|------|---------|
| 209_C000 | GPIO1_DR 数据寄存器 | 输出模式：写入输出电平；读回输出值 | R/W |
| 209_C004 | GPIO1_GDIR 方向寄存器 | 1 = 输出，0 = 输入 | R/W |
| 209_C008 | GPIO1_PSR pad 状态寄存器 | 只读，读取引脚外部真实电平（输入模式） | R |
| 209_C00C | GPIO1_ICR1 中断配置寄存器 1 | IO0~IO15 中断触发方式 | R/W |
| 209_C010 | GPIO1_ICR2 中断配置寄存器 2 | IO16~IO31 中断触发方式 | R/W |
| 209_C014 | GPIO1_IMR 中断掩码寄存器 | 1：使能该 IO 中断；0：屏蔽 | R/W |
| 209_C018 | GPIO1_ISR 中断状态寄存器 | w1c：写 1 清零中断标志 | w1c |
| 209_C01C | GPIO1_EDGE_SEL 边沿选择寄存器 | 1：双边沿触发，覆盖 ICR 设置 | R/W |

`imx35_gpio_hwdata` 结构体里面，保存的就是这些寄存器相对于基址的偏移：

```c
static struct mxc_gpio_hwdata imx35_gpio_hwdata = {
    .dr_reg = 0x00,
    .gdir_reg = 0x04,
    .psr_reg = 0x08,
    .icr1_reg = 0x0c,
    .icr2_reg = 0x10,
    .imr_reg = 0x14,
    .isr_reg = 0x18,
    .edge_sel_reg = 0x1c,
    ...
};
```

#### 2）GPIO 驱动程序简介（drivers/gpio/gpio-mxc.c）

平台驱动相关内容初学可先看懂流程，不用深挖源码细节，不影响 gpioled 实验。

**① of_device_id 匹配表**

`compatible = "fsl,imx35-gpio"` 匹配设备树 gpio1 节点，驱动文件 `gpio-mxc.c`。

```c
static const struct of_device_id mxc_gpio_dt_ids[] = {
    { .compatible = "fsl,imx1-gpio", .data = &mxc_gpio_devtype[IMX1_GPIO], },
    { .compatible = "fsl,imx21-gpio", .data = &mxc_gpio_devtype[IMX21_GPIO], },
    { .compatible = "fsl,imx31-gpio", .data = &mxc_gpio_devtype[IMX31_GPIO], },
    { .compatible = "fsl,imx35-gpio", .data = &mxc_gpio_devtype[IMX35_GPIO], },
    { /* sentinel */ }
};
···

**② 平台驱动结构体**

```c
static struct platform_driver mxc_gpio_driver = {
    .driver = {
        .name = "gpio-mxc",
        .of_match_table = mxc_gpio_dt_ids,
    },
    .probe = mxc_gpio_probe,
    .id_table = mxc_gpio_devtype,
};
```

设备树节点和驱动匹配成功，自动执行 `mxc_gpio_probe`（GPIO 驱动入口）。

**③ mxc_gpio_probe 执行流程**
```bash
mxc_gpio_probe()
├─ mxc_gpio_get_hw() → 选择硬件类型IMX35_GPIO，加载imx35_gpio_hwdata寄存器偏移表
├─ devm_kzalloc 分配mxc_gpio_port结构体（代表一组GPIO控制器，如GPIO1）
├─ platform_get_resource 获取设备树reg物理地址
├─ devm_ioremap_resource() 物理地址→内核虚拟地址映射
├─ platform_get_irq() 获取两组中断号（GPIO0~15，GPIO16~31）
├─ writel 初始化：关闭所有GPIO中断、清除中断标志
├─ bgpio_init()
│     → 初始化bgpio_chip(bgc)，填充gpio_chip的底层操作函数，绑定DR/GDIR/PSR寄存器
├─ port->bgc.gc.to_irq = mxc_gpio_to_irq;
├─ gpiochip_add(&port->bgc.gc); // 向内核注册gpio_chip，注册成功！
└─ irq_domain 初始化，用于GPIO中断
```

- `struct mxc_gpio_port`：抽象一组 GPIO（GPIO1/GPIO2...），包含虚拟基地址、中断号、bgpio_chip bgc
- `gpio_chip`：内核抽象的 GPIO 控制器，内部封装 `get/set/direction_input/direction_output` 等底层操作函数。`bgpio_init` 自动填充这些底层操作
- `gpiochip_add`：注册 gpio_chip，注册完成后，我们写驱动时就可以调用 gpiolib（gpio 子系统）对外 API，不用直接操作寄存器

**gpiolib 核心文件：** `drivers/gpio/` 目录

| 文件 | 作用 |
|------|------|
| `gpiolib.c / gpiolib.h` | GPIO 子系统核心，对外 API |
| `gpiolib-of.c` | 设备树解析 GPIO 相关接口（`of_get_named_gpio` 就在这里） |
| `gpiolib-sysfs.c` | sysfs 导出 GPIO，用户空间操作 |
| `gpio-mxc.c` | NXP IMX 系列 GPIO 硬件底层驱动 |

### 45.2.3 GPIO 子系统 API 笔记

**核心思想：** GPIO 子系统屏蔽底层寄存器操作，驱动只需要操作 GPIO 编号，不需要手动 ioremap、读写寄存器；配合设备树，使用 `of_get_named_gpio` 从 DTS 节点拿到 gpio 编号，再调用下面 API。

**头文件依赖：**

```c
#include <linux/gpio.h>
#include <linux/of_gpio.h>
```

#### 1）gpio_request — 申请 GPIO 引脚

```c
int gpio_request(unsigned gpio, const char *label)
```

| 参数 | 含义 |
|------|------|
| gpio | GPIO 编号，一般由 `of_get_named_gpio` 从设备树读取得到 |
| label | GPIO 名字，内核日志 /proc 里可以看到，自定义字符串 |

**返回：** 0 成功；负数错误码（被占用、gpio 号非法等）

> 必须先申请，才能使用该 GPIO，防止多个驱动抢占同一个引脚。

#### 2）gpio_free — 释放 GPIO 引脚

```c
void gpio_free(unsigned gpio)
```

| 参数 | 含义 |
|------|------|
| gpio | GPIO 编号，一般由 `of_get_named_gpio` 从设备树读取得到 |

**返回：** 无

> 释放 GPIO 引脚后，其他驱动可以申请使用该引脚。

#### 3）gpio_direction_input — 设置 GPIO 为输入方向

```c
void gpio_direction_input(unsigned gpio)
```
| 参数 | 含义 |
|------|------|
| gpio | GPIO 编号，一般由 `of_get_named_gpio` 从设备树读取得到 |

**返回：** 无

> 设置 GPIO 为输入方向后，该引脚可以作为输入引脚，驱动可以读取该引脚的电平状态。
#### 4）gpio_direction_output — 设置 GPIO 为输出方向
```c
void gpio_direction_output(unsigned gpio, int value)
```
| 参数 | 含义 |
|------|------|
| gpio | gpio 编号 |
| value | 上电配置后的默认电平，0 低电平，1 高电平 |

**返回：** 0 成功，负值失败

> LED 驱动最常用，点灯灭灯场景。

#### 5）gpio_get_value — 获取 GPIO 电平（读引脚）

```c
#define gpio_get_value __gpio_get_value
int __gpio_get_value(unsigned gpio)
```
**返回：** 0 / 1（引脚电平）；负值表示读取失败

> 输入引脚读取电平，也可以读取输出引脚当前输出值。

#### 6）gpio_set_value — 设置 GPIO 输出电平（写引脚）

```c
#define gpio_set_value __gpio_set_value
void __gpio_set_value(unsigned gpio, int value)
```

| 参数 | 含义 |
|------|------|
| gpio | gpio 编号 |
| value | 0 输出低，1 输出高 |

**无返回值**

> 输出引脚控制，LED 开关、继电器控制。

### 45.2.4 设备树添加 GPIO 节点模板

本节我们来学习一下如何创建 test 设备的 GPIO 节点。

**1）创建 test 设备节点**

```c
test {
    /* 节点内容 */
};
```
**2）添加 pinctrl 关联信息**

```c
test {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_test>;
    /* 其他节点内容 */
};
```

- `pinctrl-names`：状态名，`default` 代表默认状态（最常用）
- `pinctrl-0`：引用外部 pinctrl 子节点 `pinctrl_test`，把 PIN 配置绑定到 test 设备

**3）添加 GPIO 属性**

```c
test {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_test>;
    gpio = <&gpio1 0 GPIO_ACTIVE_LOW>;
};
```

`gpio = <&gpio1 0 GPIO_ACTIVE_LOW>` 解析：

| 参数 | 含义 |
|------|------|
| `&gpio1` | 引用 gpio1 控制器 |
| `0` | gpio1 下的 IO 编号，GPIO1_IO00 |
| `GPIO_ACTIVE_LOW` | 低电平有效；`GPIO_ACTIVE_HIGH` 高电平有效 |

> **注意：** 很多教材规范推荐属性名用 `gpios`（复数），这里示例写的 `gpio`，写驱动读取属性名要和 DTS 保持一致！

### 45.2.5 与 gpio 相关的 OF 函数

**头文件：** `#include <linux/of_gpio.h>`

#### 1）of_gpio_named_count

统计指定属性名里面 GPIO 的数量

```c
int of_gpio_named_count(struct device_node *np, const char *propname)
```
| 参数 | 含义 |
|------|------|
| np | 设备节点指针 |
| propname | 属性名字符串，如 "gpios" / "gpio" |

**返回：** 正数 = GPIO 个数；负数 = 失败

> **坑点：** 空占位的 GPIO（写 0）也会被计数！`gpios = <0 &gpio1 1 2 0 &gpio2 3 4>;` 统计结果是 4 个。

#### 2）of_gpio_count

专门统计 `gpios` 这个固定属性的 GPIO 数量，等价于 `of_gpio_named_count(np, "gpios")`

```c
int of_gpio_count(struct device_node *np)
```

**返回：** 正数 = GPIO 个数；负数 = 失败

#### 3）of_get_named_gpio

把 DTS 中 `<&gpio1 0 GPIO_ACTIVE_LOW>` 转换成内核需要的 GPIO 编号，供给 `gpio_request` 等 GPIO 子系统 API 使用

```c
int of_get_named_gpio(struct device_node *np,
                      const char *propname,
                      int index)
```

| 参数 | 含义 |
|------|------|
| np | 设备节点 |
| propname | DTS 里面 GPIO 属性名称，例 "gpio" / "gpios" |
| index | 索引，一个属性写多个 gpio 时选择第几个；只有 1 个 gpio 填 0 |

**返回：** 正数 → GPIO 编号；负数 → 获取失败（节点不存在、属性写错、gpio 非法）

**典型驱动调用片段：**

```c
int gpio_num;
//读取DTS test节点下gpio属性，第0个gpio
gpio_num = of_get_named_gpio(np, "gpio", 0);
if(gpio_num < 0){
    printk("of_get_named_gpio failed!\n");
    return -EINVAL;
}
//拿到gpio编号后，就可以调用gpio_request申请引脚
gpio_request(gpio_num, "test-gpio");
```

---

## 45.3 硬件原理图分析

本章实验硬件原理图参考 8.3 小节即可。

---

## 45.4 实验程序编写

对比上一章 dtsled：dtsled 是设备树传物理地址，驱动 ioremap 直接操作寄存器；本章完全不操作寄存器，交给 pinctrl+GPIO 子系统，驱动调用内核 GPIO API，是 Linux 标准写法。硬件：I.MX6ULL，LED 使用 GPIO1_IO03。

### 45.4.1 修改设备树文件

I.MX6U-ALPHA 开发板上的 LED 灯使用了 GPIO1_IO03 这个 PIN，打开 `imx6ull-alientek-emmc.dts`，在 iomuxc 节点的 `imx6ul-evk` 子节点下创建一个名为 `pinctrl_led` 的子节点：

```c
pinctrl_led: ledgrp {
    fsl,pins = <
        MX6UL_PAD_GPIO1_IO03__GPIO1_IO03 0x10B0 /* LED0 */
    >;
};
```

- `MX6UL_PAD_GPIO1_IO03__GPIO1_IO03`：把 PIN 复用为 GPIO 功能
- `0x10B0`：PAD 电气配置（上下拉、速率、驱动能力）

在根节点添加 `gpioled` 设备节点：

```c
gpioled {
    #address-cells = <1>;
    #size-cells = <1>;
    compatible = "atkalpha-gpioled";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_led>;
    led-gpio = <&gpio1 3 GPIO_ACTIVE_LOW>;
    status = "okay";
};
```
**重点解析：**

| 属性 | 含义 |
|------|------|
| `pinctrl-names = "default"` | 默认 PIN 配置状态 |
| `pinctrl-0 = <&pinctrl_led>` | 关联上面 pinctrl_led 节点 |
| `led-gpio = <&gpio1 3 GPIO_ACTIVE_LOW>` | 使用 gpio1 组，IO3，低电平点亮 LED |

> 属性名 `led-gpio`，驱动代码 `of_get_named_gpio` 第二个参数必须写 `"led-gpio"`，字符串严格匹配！

**PIN 冲突检查：**

GPIO1_IO03 在原厂 dts 里默认分配给 TSC 电阻触摸屏，两处冲突需要注释屏蔽：
- `pinctrl_tsc` 节点：`MX6UL_PAD_GPIO1_IO03__GPIO1_IO03 0xb0` 这一行注释掉
- `&tsc` 外设节点：`xnur-gpio = <&gpio1 3 GPIO_ACTIVE_LOW>;` 注释

> 原理：一个 PIN 同一时间只能分配给一个外设。不屏蔽的话，TSC 驱动占用该引脚，我们 `gpio_request` 申请 GPIO 会直接失败！

**编译与验证：**

```bash
make dtbs
```

### 45.4.2 gpioled.c 驱动代码解析

工程：`5_gpioled`，文件 `gpioled.c`

**设备结构体：**

```c
struct gpioled_dev{
    dev_t devid;                /* 设备号 */
    struct cdev cdev;           /* cdev */
    struct class *class;        /* 类 */
    struct device *device;      /* 设备 */
    int major;                  /* 主设备号 */
    int minor;                  /* 次设备号 */
    struct device_node *nd;     /* 设备树节点 */
    int led_gpio;               /* 保存GPIO编号，核心新增成员 */
};
struct gpioled_dev gpioled;
```

**文件操作集合：**

```c
static struct file_operations gpioled_fops = {
    .owner = THIS_MODULE,
    .open = led_open,
    .read = led_read,
    .write = led_write,
    .release = led_release,
};
```

- **led_open**：把全局设备结构体存入 `filp->private_data`，后续 write/release 可以拿到设备，驱动经典技巧

```c
static int led_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &gpioled;
    return 0;
}
```

- **led_read**：本实验不需要读 LED 状态，直接 return 0
- **led_write**：用户程序 write 写 1 开灯、写 0 关灯

```c
static ssize_t led_write(struct file *filp, const char __user *buf, size_t cnt, loff_t *offt)
{
    int retvalue;
    unsigned char databuf[1];
    unsigned char ledstat;
    struct gpioled_dev *dev = filp->private_data;

    retvalue = copy_from_user(databuf, buf, cnt); // 用户态→内核态拷贝
    if(retvalue < 0) {
        printk("kernel write failed!\r\n");
        return -EFAULT;
    }
    ledstat = databuf[0];

    if(ledstat == LEDON) {
        gpio_set_value(dev->led_gpio, 0);  // 低电平点亮，DTS是ACTIVE_LOW
    } else if(ledstat == LEDOFF) {
        gpio_set_value(dev->led_gpio, 1);  // 高电平熄灭
    }
    return 0;
}
```

> **重点：** `gpio_set_value` 直接操作 GPIO，没有 ioremap、没有寄存器读写，GPIO 子系统封装底层硬件。

**驱动入口 led_init 完整流程：**

```c
static int __init led_init(void)
{
    int ret = 0;
    /* 1、找到设备树gpioled节点 */
    gpioled.nd = of_find_node_by_path("/gpioled");
    if(gpioled.nd == NULL) {
        printk("gpioled node cant not found!\r\n");
        return -EINVAL;
    }

    /* 2、读取设备树led-gpio属性，得到GPIO编号 */
    gpioled.led_gpio = of_get_named_gpio(gpioled.nd, "led-gpio", 0);
    if(gpioled.led_gpio < 0) {
        printk("can't get led-gpio");
        return -EINVAL;
    }
    printk("led-gpio num = %d\r\n", gpioled.led_gpio);

    /* 3、设置GPIO为输出，默认输出高电平，LED默认熄灭 */
    ret = gpio_direction_output(gpioled.led_gpio, 1);
    if(ret < 0) {
        printk("can't set gpio!\r\n");
    }

    /* ========= 新字符设备注册流程 ========= */
    /* 申请设备号（自动分配主设备号） */
    alloc_chrdev_region(&gpioled.devid, 0, GPIOLED_CNT, GPIOLED_NAME);
    gpioled.major = MAJOR(gpioled.devid);
    gpioled.minor = MINOR(gpioled.devid);

    cdev_init(&gpioled.cdev, &gpioled_fops); //初始化cdev
    cdev_add(&gpioled.cdev, gpioled.devid, GPIOLED_CNT); //添加cdev

    gpioled.class = class_create(THIS_MODULE, GPIOLED_NAME); //创建类
    gpioled.device = device_create(gpioled.class, NULL, gpioled.devid, NULL, GPIOLED_NAME); //自动创建设备节点 /dev/gpioled
    return 0;
}
```

**驱动出口 led_exit：**

```c
static void __exit led_exit(void)
{
    // 实际项目一定要加这一行，释放GPIO资源
    gpio_free(gpioled.led_gpio);

    cdev_del(&gpioled.cdev);
    unregister_chrdev_region(gpioled.devid, GPIOLED_CNT);

    device_destroy(gpioled.class, gpioled.devid);
    class_destroy(gpioled.class);
}
```
调用
```bash
module_init(led_init);
module_exit(led_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("tantan");
```

### 45.4.3 测试 APP

直接复用前面章节的 `ledApp.c`

```bash
arm-linux-gnueabihf-gcc ledApp.c -o ledApp
```

---

## 45.5 运行测试

### 45.5.1 编译驱动程序和测试 APP

**1）驱动 Makefile**

```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)

obj-m := gpioled.o

build:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

- 核心改动：`obj-m := gpioled.o`，指定编译 gpioled.c 生成 gpioled.ko 内核模块
- `-C $(KERNELDIR)` 切换到内核源码

**编译命令（Ubuntu 主机）：**

```bash
make -j32
```

编译成功生成：`gpioled.ko`

**说明：** `-C $(KERNELDIR)` 切换到内核源码目录执行编译；`M=$(CURRENT_PATH)` 表示外部模块源码在当前目录。

**2）编译测试 APP**

输出可执行文件 `ledApp`，这个程序运行在开发板（ARM）上：

```bash
arm-linux-gnueabihf-gcc ledApp.c -o ledApp
```

编译成功生成：`ledApp`

### 45.5.2 开发板运行测试

#### 1）文件拷贝

将 `gpioled.ko`、`ledApp` 拷贝到开发板根文件系统目录：`rootfs/lib/modules/4.1.15`

拷贝方式可选：NFS 挂载、SD 卡、TFTP

#### 2）加载驱动模块

进入目录 `/lib/modules/4.1.15`：

```bash
depmod                 # 首次加载模块，生成模块依赖信息
modprobe gpioled.ko    # 加载 gpioled 驱动
```

加载成功内核打印信息：
```
gpioled node has been found!
led-gpio num = 3
gpioled major=xxx,minor=0
```

| 打印信息 | 含义 |
|---------|------|
| `node has been found` | 设备树 gpioled 节点读取成功 |
| `led-gpio num = 3` | GPIO1_IO03 对应的内核 gpio 编号是 3 |
| `major=xxx, minor=0` | 自动分配的主设备号和次设备号 |

> **自动创建设备节点：** 加载成功后，内核自动生成 `/dev/gpioled`，无需手动 `mknod`。

#### 3）APP 测试 LED

```bash
./ledApp /dev/gpioled 1   # 打开LED灯
./ledApp /dev/gpioled 0   # 关闭LED灯
```

**原理：** 应用程序向 `/dev/gpioled` 文件写入 1/0，触发驱动里 `led_write` 函数，调用 `gpio_set_value` 控制引脚电平。

#### 4）卸载驱动

```bash
modprobe -r gpioled.ko
```

卸载后，`/dev/gpioled` 节点会自动消失。



