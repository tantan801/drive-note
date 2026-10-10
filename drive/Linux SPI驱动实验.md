# 第六十二章 Linux SPI 驱动实验

实验目标：驱动 I.MX6U-ALPHA 板上 ICM-20608 SPI 六轴传感器，应用层读取原始传感器数据。

---

## 62.1 Linux 下 SPI 驱动框架简介

SPI 驱动分为 **SPI 主机控制器驱动**、**SPI 设备驱动**，结构和 I2C 框架高度相似。主机是 SOC 内部 SPI 控制器；一般由芯片原厂实现，使用者重点写 SPI 设备驱动。

---

### 62.1.1 SPI 主机驱动

内核使用 `spi_master` 结构体描述 SPI 主机控制器，定义在 `include/linux/spi/spi.h`。

核心成员：

| 成员 | 说明 |
|------|------|
| `bus_num` | SPI 总线编号 |
| `num_chipselect` | 片选信号数量 |
| `max_speed_hz` / `min_speed_hz` | 总线最大、最小时钟频率 |
| `transfer()` | SPI 控制器数据传输函数，和 I2C 的 `master_xfer` 作用一致 |
| `transfer_one_message()` | 发送单个 `spi_message` 消息，SPI 数据会封装成 message 队列发送 |

> 说明：I.MX6U 的 SPI 主机驱动 NXP 已经实现，用户不用编写主机驱动。

#### 1）spi_master 申请/释放

```c
// 申请spi_master，dev为platform设备device，size私有数据大小
struct spi_master *spi_alloc_master(struct device *dev, unsigned size);

// 释放spi_master
void spi_master_put(struct spi_master *master);
```

#### 2）spi_master 注册/注销

```c
// 注册master
int spi_register_master(struct spi_master *master);
// 注销master
void spi_unregister_master(struct spi_master *master);
```

I.MX6U 使用 bitbang 方式：`spi_bitbang_start()` 完成注册，注销对应 `spi_bitbang_stop()`，内部依然调用上面两个函数。

---

### 62.1.2 SPI设备驱动

spi 设备驱动和 i2c 设备驱动也很类似，`spi_driver` 结构体代表 SPI 设备驱动，`include/linux/spi/spi.h`：

```c
struct spi_driver {
    const struct spi_device_id *id_table;  // 传统id匹配表
    int (*probe)(struct spi_device *spi);  // 匹配成功执行
    int (*remove)(struct spi_device *spi);
    void (*shutdown)(struct spi_device *spi);
    struct device_driver driver;
};
```

注册注销 API：

`spi_driver` 初始化完成以后需要向 Linux 内核注册，`spi_driver` 注册函数为 `spi_register_driver`，函数原型如下：

```c
int spi_register_driver(struct spi_driver *sdrv);
```

函数参数和返回值含义如下：  
`sdrv`：要注册的 spi_driver。  
**返回值**：0，注册成功；赋值，注册失败。

注销 SPI 设备驱动以后也需要注销掉前面注册的 `spi_driver`，使用 `spi_unregister_driver` 函数完成 `spi_driver` 的注销，函数原型如下：

```c
void spi_unregister_driver(struct spi_driver *sdrv);
```

函数参数和返回值含义如下：  
`sdrv`：要注销的 spi_driver。  
**返回值**：无。

#### spi_driver 标准模板：

```c
/* probe函数 */
static int xxx_probe(struct spi_device *spi)
{
    return 0;
}

/* remove函数 */
static int xxx_remove(struct spi_device *spi)
{
    return 0;
}

/* 传统匹配ID列表 */
static const struct spi_device_id xxx_id[] = {
    {"xxx", 0},
    {}
};

/* 设备树匹配列表 */
static const struct of_device_id xxx_of_match[] = {
    { .compatible = "xxx" },
    { /* Sentinel */ }
};

/* SPI驱动结构体 */
static struct spi_driver xxx_driver = {
    .probe = xxx_probe,
    .remove = xxx_remove,
    .driver = {
        .owner = THIS_MODULE,
        .name = "xxx",
        .of_match_table = xxx_of_match,
    },
    .id_table = xxx_id,
};

static int __init xxx_init(void)
{
    return spi_register_driver(&xxx_driver);
}

static void __exit xxx_exit(void)
{
    spi_unregister_driver(&xxx_driver);
}

module_init(xxx_init);
module_exit(xxx_exit);
MODULE_LICENSE("GPL");
```

---

### 62.1.3 SPI 设备和驱动匹配过程

SPI 总线类型 `spi_bus_type`，定义在 `drivers/spi/spi.c`，匹配函数：`spi_match_device`。

**匹配优先级顺序**：

1. 设备树 OF 匹配：对比设备节点 `compatible` 和 `of_device_id`
2. ACPI 匹配
3. `spi_device_id` 传统 `id_table` 匹配
4. 对比 `spi_device->modalias` 与 `device_driver->name`

```c
struct bus_type spi_bus_type = {
    .name = "spi",
    .dev_groups = spi_dev_groups,
    .match = spi_match_device,
    .uevent = spi_uevent,
};
```

`spi_match_device` 逻辑：

```c
static int spi_match_device(struct device *dev, struct device_driver *drv)
{
    const struct spi_device *spi = to_spi_device(dev);
    const struct spi_driver *sdrv = to_spi_driver(drv);

    /* OF设备树匹配优先 */
    if (of_driver_match_device(dev, drv))
        return 1;

    /* ACPI匹配 */
    if (acpi_driver_match_device(dev, drv))
        return 1;

    /* id_table匹配 */
    if (sdrv->id_table)
        return !!spi_match_id(sdrv->id_table, spi);

    /* 对比modalias和driver name */
    return strcmp(spi->modalias, drv->name) == 0;
}
```

---

## 62.2 I.MX6U SPI主机驱动分析

ECSPI 主机驱动由 NXP 原厂已经实现，用户不需要编写主机控制器驱动。

### imx6ull.dtsi ecspi3 设备树节点

```dts
ecspi3: ecspi@02010000 {
    #address-cells = <1>;
    #size-cells = <0>;
    compatible = "fsl,imx6ul-ecspi", "fsl,imx51-ecspi"; // 用于匹配platform驱动
    reg = <0x02010000 0x4000>;        // ECSPI3寄存器基地址、寄存器空间大小
    interrupts = <GIC_SPI 33 IRQ_TYPE_LEVEL_HIGH>; // 中断信息
    clocks = <&clks IMX6UL_CLK_ECSPI3>,
             <&clks IMX6UL_CLK_ECSPI3>;
    clock-names = "ipg", "per";
    dmas = <&sdma 7 7 1>, <&sdma 8 7 2>; // DMA收发通道
    dma-names = "rx", "tx";
    status = "disabled";  // 默认状态关闭，需要在板级dts中修改为okay启用
};
```

通过 `compatible = "fsl,imx6ul-ecspi"` 在内核匹配对应的 platform 驱动文件：`drivers/spi/spi-imx.c`。

```c
/* platform设备id匹配表，无设备树时使用 */
static struct platform_device_id spi_imx_devtype[] = {
    {
        .name = "imx1-cspi",
        .driver_data = (kernel_ulong_t) &imx1_cspi_devtype_data,
    },
    ......
    {
        .name = "imx6ul-ecspi",
        .driver_data = (kernel_ulong_t) &imx6ul_ecspi_devtype_data,
    },
    {/* sentinel */}
};

/* 设备树OF匹配表 */
static const struct of_device_id spi_imx_dt_ids[] = {
    { .compatible = "fsl,imx1-cspi", .data = &imx1_cspi_devtype_data, },
    ......
    { .compatible = "fsl,imx6ul-ecspi", .data = &imx6ul_ecspi_devtype_data, },
    {/* sentinel */}
};
MODULE_DEVICE_TABLE(of, spi_imx_dt_ids);

/* SPI主机本身是platform_driver框架 */
static struct platform_driver spi_imx_driver = {
    .driver = {
        .name = DRIVER_NAME,
        .of_match_table = spi_imx_dt_ids,
        .pm = IMX_SPI_PM,
    },
    .id_table = spi_imx_devtype,
    .probe = spi_imx_probe,
    .remove = spi_imx_remove,
};
module_platform_driver(spi_imx_driver);
```

1. 设备树 compatible 匹配 `spi_imx_dt_ids`，匹配成功执行 `spi_imx_probe`
2. `spi_imx_probe` 读取设备树资源，申请、初始化 `spi_master`
3. 调用 `spi_bitbang_start()` 注册 spi_master，内部封装调用 `spi_register_master()`

### 数据收发调用链路

```
spi_imx_transfer
    -> spi_imx_pio_transfer
        -> spi_imx_push
            -> spi_imx->tx
```

1. `spi_imx` 为 `spi_imx_data` 结构体指针，保存 tx/rx 收发函数
2. `spi_imx_setupxfer` 根据传输位宽选择对应的收发函数
3. 发送函数集合：

| 函数 | 位宽 |
|------|:---:|
| `spi_imx_buf_tx_u8` | 8bit 发送 |
| `spi_imx_buf_tx_u16` | 16bit 发送 |
| `spi_imx_buf_tx_u32` | 32bit 发送 |

4. 接收函数集合：

| 函数 | 位宽 |
|------|:---:|
| `spi_imx_buf_rx_u8` | 8bit 接收 |
| `spi_imx_buf_rx_u16` | 16bit 接收 |
| `spi_imx_buf_rx_u32` | 32bit 接收 |

#### 发送函数宏实现示例

```c
#define MXC_SPI_BUF_TX(type)                        \
static void spi_imx_buf_tx_##type(struct spi_imx_data *spi_imx)   \
{                                   \
    type val = 0;                           \
                                    \
    if (spi_imx->tx_buf) {                      \
        val = *(type *)spi_imx->tx_buf;             \
        spi_imx->tx_buf += sizeof(type);            \
    }                               \
                                    \
    spi_imx->count -= sizeof(type);                 \
                                    \
    writel(val, spi_imx->base + MXC_CSPITXDATA);            \
}

MXC_SPI_BUF_RX(u8)
MXC_SPI_BUF_TX(u8)
```

- 使用宏批量生成不同位宽收发函数
- `writel(val, spi_imx->base + MXC_CSPITXDATA)`：把数据写入 SPI 的 TXDATA 寄存器，和裸机操作寄存器逻辑一致

---

## 62.4 硬件原理图分析

实验使用硬件资源：  
① LED0 指示灯  
② RGB LCD 屏幕  
③ ICM-20608 六轴传感器  
④ 串口

ICM-20608 挂载底板 SPI 接口，硬件参考原理图如下。

![ICM-20608原理图](photo/Linux%20SPI驱动实验/1-1%20ICM-20608原理图.png)

---

## 62.5 实验程序编写

> **例程路径**：开发板光盘 → 2、Linux 驱动例程 → `22_spi`

---

### 62.5.1 修改设备树

#### 1、添加 ICM20608 IO pinctrl 节点（imx6ull-alientek-emmc.dts iomuxc 节点内）

```dts
pinctrl_ecspi3: icm20608 {
    fsl,pins = <
        MX6UL_PAD_UART2_TX_DATA__GPIO1_IO20        0x10b0  /* CS 复用为GPIO，软件控制片选 */
        MX6UL_PAD_UART2_RX_DATA__ECSPI3_SCLK       0x10b1  /* SCLK */
        MX6UL_PAD_UART2_RTS_B__ECSPI3_MISO         0x10b1  /* MISO */
        MX6UL_PAD_UART2_CTS_B__ECSPI3_MOSI         0x10b1  /* MOSI */
    >;
};
```

片选引脚不使用 ECSPI 硬件 SS，复用普通 GPIO，由 SPI 主机驱动控制片选。

#### 2、ecspi3 追加 icm20608 子节点

```dts
&ecspi3 {
    fsl,spi-num-chipselects = <1>;          /* 片选设备数量1个 */
    cs-gpios = <&gpio1 20 GPIO_ACTIVE_LOW>;  /* 指定片选GPIO */
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi3>;
    status = "okay";                        /* 开启ECSPI3控制器 */

    spidev: icm20608@0 {                    /* @0：接在SPI通道0 */
        compatible = "alientek,icm20608";
        spi-max-frequency = <8000000>;      /* ICM20608最大SPI时钟8MHz */
        reg = <0>;
    };
};
```

修改完成执行 `make dtbs` 编译设备树，使用新 dtb 启动开发板。

---

### 62.5.2 编写 ICM20608 驱动

新建 `22_spi` 工程，创建 `icm20608.c`、`icm20608reg.h`。

#### icm20608reg.h（寄存器头文件，节选）

```c
#ifndef ICM20608_H
#define ICM20608_H

#define ICM20608G_ID    0XAF
#define ICM20608D_ID    0XAE

/* 寄存器地址节选 */
#define ICM20_SELF_TEST_X_GYRO  0x00
#define ICM20_SELF_TEST_Y_GYRO  0x01
#define ICM20_SELF_TEST_Z_GYRO  0x02
#define ICM20_SELF_TEST_X_ACCEL 0x0D
#define ICM20_SELF_TEST_Y_ACCEL 0x0E
#define ICM20_SELF_TEST_Z_ACCEL 0x0F

#define ICM20_PWR_MGMT_1        0x6B
#define ICM20_WHO_AM_I          0x75
#define ICM20_SMPLRT_DIV        0x19
#define ICM20_GYRO_CONFIG       0x1B
#define ICM20_ACCEL_CONFIG      0x1C
#define ICM20_CONFIG            0x1A
#define ICM20_ACCEL_CONFIG2     0x1D
#define ICM20_PWR_MGMT_2        0x6C
#define ICM20_LP_MODE_CFG       0x63
#define ICM20_FIFO_EN           0x23

#define ICM20_ACCEL_XOUT_H      0x3B

/* 加速度静态偏移 */
#define ICM20_XA_OFFSET_H       0x77
#define ICM20_XA_OFFSET_L       0x78
#define ICM20_YA_OFFSET_H       0x7A
#define ICM20_YA_OFFSET_L       0x7B
#define ICM20_ZA_OFFSET_H       0x7D
#define ICM20_ZA_OFFSET_L       0x7E

#endif
```

#### icm20608.c

##### 1、设备私有结构体

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/spi/spi.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <asm/io.h>
#include "icm20608reg.h"

#define ICM20608_CNT    1
#define ICM20608_NAME   "icm20608"

/* icm20608设备结构体 */
struct icm20608_dev {
    dev_t devid;
    struct cdev cdev;
    struct class *class;
    struct device *device;
    struct device_node *nd;
    int major;
    void *private_data;     /* 保存spi_device指针 */
    int cs_gpio;

    signed int gyro_x_adc;
    signed int gyro_y_adc;
    signed int gyro_z_adc;
    signed int accel_x_adc;
    signed int accel_y_adc;
    signed int accel_z_adc;
    signed int temp_adc;
};

static struct icm20608_dev icm20608dev;
```

##### 2、spi_driver 注册注销

```c
/* 传统id匹配表 */
static const struct spi_device_id icm20608_id[] = {
    {"alientek,icm20608", 0},
    {}
};

/* 设备树OF匹配表 */
static const struct of_device_id icm20608_of_match[] = {
    { .compatible = "alientek,icm20608" },
    { /* Sentinel */ }
};

/* SPI驱动结构体 */
static struct spi_driver icm20608_driver = {
    .probe = icm20608_probe,
    .remove = icm20608_remove,
    .driver = {
        .owner = THIS_MODULE,
        .name = "icm20608",
        .of_match_table = icm20608_of_match,
    },
    .id_table = icm20608_id,
};

static int __init icm20608_init(void)
{
    return spi_register_driver(&icm20608_driver);
}

static void __exit icm20608_exit(void)
{
    spi_unregister_driver(&icm20608_driver);
}

module_init(icm20608_init);
module_exit(icm20608_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("zuozhongkai");
```

##### 3、probe、remove 函数

```c
static int icm20608_probe(struct spi_device *spi)
{
    /* 1.分配设备号 */
    if (icm20608dev.major) {
        icm20608dev.devid = MKDEV(icm20608dev.major, 0);
        register_chrdev_region(icm20608dev.devid, ICM20608_CNT, ICM20608_NAME);
    } else {
        alloc_chrdev_region(&icm20608dev.devid, 0, ICM20608_CNT, ICM20608_NAME);
        icm20608dev.major = MAJOR(icm20608dev.devid);
    }

    /* 2.初始化注册cdev */
    cdev_init(&icm20608dev.cdev, &icm20608_ops);
    cdev_add(&icm20608dev.cdev, icm20608dev.devid, ICM20608_CNT);

    /* 3.创建类 */
    icm20608dev.class = class_create(THIS_MODULE, ICM20608_NAME);
    if (IS_ERR(icm20608dev.class))
        return PTR_ERR(icm20608dev.class);

    /* 4.创建设备节点 /dev/icm20608 */
    icm20608dev.device = device_create(icm20608dev.class, NULL, icm20608dev.devid, NULL, ICM20608_NAME);
    if (IS_ERR(icm20608dev.device))
        return PTR_ERR(icm20608dev.device);

    /* 设置SPI模式0 CPOL=0 CPHA=0 */
    spi->mode = SPI_MODE_0;
    spi_setup(spi);
    icm20608dev.private_data = spi;

    icm20608_reginit();
    return 0;
}

static int icm20608_remove(struct spi_device *spi)
{
    cdev_del(&icm20608dev.cdev);
    unregister_chrdev_region(icm20608dev.devid, ICM20608_CNT);
    device_destroy(icm20608dev.class, icm20608dev.devid);
    class_destroy(icm20608dev.class);
    return 0;
}
```

##### 4、寄存器读写、初始化函数

```c
/* 连续读多个寄存器 */
static int icm20608_read_regs(struct icm20608_dev *dev, u8 reg, void *buf, int len)
{
    int ret = -1;
    unsigned char txdata[1];
    unsigned char *rxdata;
    struct spi_message m;
    struct spi_transfer *t;
    struct spi_device *spi = (struct spi_device *)dev->private_data;

    t = kzalloc(sizeof(struct spi_transfer), GFP_KERNEL);
    if(!t) return -ENOMEM;

    rxdata = kzalloc(sizeof(char) * len, GFP_KERNEL);
    if(!rxdata) goto out1;

    txdata[0] = reg | 0x80;  /* SPI读：地址最高位置1 */
    t->tx_buf = txdata;
    t->rx_buf = rxdata;
    t->len = len + 1;        /* 1字节地址 + len字节数据 */

    spi_message_init(&m);
    spi_message_add_tail(t, &m);
    ret = spi_sync(spi, &m);
    if(ret) goto out2;

    memcpy(buf , rxdata+1, len);

out2:
    kfree(rxdata);
out1:
    kfree(t);
    return ret;
}

/* 连续写多个寄存器 */
static s32 icm20608_write_regs(struct icm20608_dev *dev, u8 reg, u8 *buf, u8 len)
{
    int ret = -1;
    unsigned char *txdata;
    struct spi_message m;
    struct spi_transfer *t;
    struct spi_device *spi = (struct spi_device *)dev->private_data;

    t = kzalloc(sizeof(struct spi_transfer), GFP_KERNEL);
    if(!t) return -ENOMEM;

    txdata = kzalloc(sizeof(char)+len, GFP_KERNEL);
    if(!txdata) goto out1;

    *txdata = reg & ~0x80; /* SPI写：地址最高位清零 */
    memcpy(txdata+1, buf, len);
    t->tx_buf = txdata;
    t->len = len+1;

    spi_message_init(&m);
    spi_message_add_tail(t, &m);
    ret = spi_sync(spi, &m);

out2:
    kfree(txdata);
out1:
    kfree(t);
    return ret;
}

/* 读单个寄存器 */
static unsigned char icm20608_read_onereg(struct icm20608_dev *dev, u8 reg)
{
    u8 data = 0;
    icm20608_read_regs(dev, reg, &data, 1);
    return data;
}

/* 写单个寄存器 */
static void icm20608_write_onereg(struct icm20608_dev *dev, u8 reg, u8 value)
{
    u8 buf = value;
    icm20608_write_regs(dev, reg, &buf, 1);
}

/* 读取六轴原始ADC数据 */
void icm20608_readdata(struct icm20608_dev *dev)
{
    unsigned char data[14] = { 0 };
    icm20608_read_regs(dev, ICM20_ACCEL_XOUT_H, data, 14);

    dev->accel_x_adc = (signed short)((data[0] << 8) | data[1]);
    dev->accel_y_adc = (signed short)((data[2] << 8) | data[3]);
    dev->accel_z_adc = (signed short)((data[4] << 8) | data[5]);
    dev->temp_adc    = (signed short)((data[6] << 8) | data[7]);
    dev->gyro_x_adc  = (signed short)((data[8] << 8) | data[9]);
    dev->gyro_y_adc  = (signed short)((data[10] << 8) | data[11]);
    dev->gyro_z_adc  = (signed short)((data[12] << 8) | data[13]);
}

/* ICM20608寄存器初始化 */
void icm20608_reginit(void)
{
    u8 value = 0;
    icm20608_write_onereg(&icm20608dev, ICM20_PWR_MGMT_1, 0x80);
    mdelay(50);
    icm20608_write_onereg(&icm20608dev, ICM20_PWR_MGMT_1, 0x01);
    mdelay(50);

    value = icm20608_read_onereg(&icm20608dev, ICM20_WHO_AM_I);
    printk("ICM20608 ID = %#X\r\n", value);

    icm20608_write_onereg(&icm20608dev, ICM20_SMPLRT_DIV, 0x00);
    icm20608_write_onereg(&icm20608dev, ICM20_GYRO_CONFIG, 0x18);
    icm20608_write_onereg(&icm20608dev, ICM20_ACCEL_CONFIG, 0x18);
    icm20608_write_onereg(&icm20608dev, ICM20_CONFIG, 0x04);
    icm20608_write_onereg(&icm20608dev, ICM20_ACCEL_CONFIG2, 0x04);
    icm20608_write_onereg(&icm20608dev, ICM20_PWR_MGMT_2, 0x00);
    icm20608_write_onereg(&icm20608dev, ICM20_LP_MODE_CFG, 0x00);
    icm20608_write_onereg(&icm20608dev, ICM20_FIFO_EN, 0x00);
}
```

##### 5、字符设备 file_operations

```c
static int icm20608_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &icm20608dev;
    return 0;
}

static ssize_t icm20608_read(struct file *filp, char __user *buf, size_t cnt, loff_t *off)
{
    signed int data[7];
    long err = 0;
    struct icm20608_dev *dev = (struct icm20608_dev *)filp->private_data;

    icm20608_readdata(dev);
    data[0] = dev->gyro_x_adc;
    data[1] = dev->gyro_y_adc;
    data[2] = dev->gyro_z_adc;
    data[3] = dev->accel_x_adc;
    data[4] = dev->accel_y_adc;
    data[5] = dev->accel_z_adc;
    data[6] = dev->temp_adc;

    err = copy_to_user(buf, data, sizeof(data));
    return 0;
}

static int icm20608_release(struct inode *inode, struct file *filp)
{
    return 0;
}

static const struct file_operations icm20608_ops = {
    .owner = THIS_MODULE,
    .open = icm20608_open,
    .read = icm20608_read,
    .release = icm20608_release,
};
```

---

### 62.5.3 编写测试 APP

```c
#include "stdio.h"
#include "unistd.h"
#include "sys/types.h"
#include "sys/stat.h"
#include "fcntl.h"
#include "stdlib.h"
#include "string.h"

int main(int argc, char *argv[])
{
    int fd;
    char *filename;
    signed int databuf[7];
    signed int gyro_x_adc, gyro_y_adc, gyro_z_adc;
    signed int accel_x_adc, accel_y_adc, accel_z_adc;
    signed int temp_adc;

    float gyro_x_act, gyro_y_act, gyro_z_act;
    float accel_x_act, accel_y_act, accel_z_act;
    float temp_act;
    int ret = 0;

    if (argc != 2) {
        printf("Error Usage!\r\n");
        return -1;
    }
    filename = argv[1];
    fd = open(filename, O_RDWR);
    if(fd < 0) {
        printf("can't open file %s\r\n", filename);
        return -1;
    }

    while (1) {
        ret = read(fd, databuf, sizeof(databuf));
        if(ret == 0) {
            gyro_x_adc = databuf[0];
            gyro_y_adc = databuf[1];
            gyro_z_adc = databuf[2];
            accel_x_adc = databuf[3];
            accel_y_adc = databuf[4];
            accel_z_adc = databuf[5];
            temp_adc   = databuf[6];

            /* 转换成实际物理值，驱动不做浮点运算，放在应用层 */
            gyro_x_act = (float)(gyro_x_adc) / 16.4;
            gyro_y_act = (float)(gyro_y_adc) / 16.4;
            gyro_z_act = (float)(gyro_z_adc) / 16.4;

            accel_x_act = (float)(accel_x_adc) / 2048;
            accel_y_act = (float)(accel_y_adc) / 2048;
            accel_z_act = (float)(accel_z_adc) / 2048;

            temp_act = ((float)(temp_adc) - 25 ) / 326.8 + 25;

            printf("\r\n原始值:\r\n");
            printf("gx = %d, gy = %d, gz = %d\r\n", gyro_x_adc, gyro_y_adc, gyro_z_adc);
            printf("ax = %d, ay = %d, az = %d\r\n", accel_x_adc, accel_y_adc, accel_z_adc);
            printf("temp = %d\r\n", temp_adc);
            printf("实际值: gx=%.2f°/S gy=%.2f°/S gz=%.2f°/S\r\n",gyro_x_act,gyro_y_act,gyro_z_act);
            printf("ax=%.2fg ay=%.2fg az=%.2fg\r\n",accel_x_act,accel_y_act,accel_z_act);
            printf("temp = %.2f°C\r\n", temp_act);
        }
        usleep(100000);
    }
    close(fd);
    return 0;
}
```

---

## 62.6 运行测试

### 62.6.1 编译驱动与 APP

#### 1、Makefile

```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)

obj-m := icm20608.o

build:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

编译驱动：`make -j32`，输出 `icm20608.ko`

#### 2、交叉编译 APP，开启 NEON 硬件浮点

```bash
arm-linux-gnueabihf-gcc -march=armv7-a -mfpu=neon -mfloat-abi=hard icm20608App.c -o icm20608App
```

校验硬件浮点：`arm-linux-gnueabihf-readelf -A icm20608App`

---

### 62.6.2 开发板测试

#### 1、将 icm20608.ko、icm20608App 拷贝至开发板 `/lib/modules/4.1.15`

```bash
depmod
modprobe icm20608.ko
```

#### 2、运行测试程序

```bash
./icm20608App /dev/icm20608
```

#### 现象说明

- **开发板静止**：Z 轴加速度 ≈ 1g 重力；陀螺仪三轴角速度接近 0°/S；温度三十多度
- **晃动开发板**：陀螺仪、加速度计数值会跟随姿态变化

> ⚠️ **注意**：内核驱动内部禁止浮点运算；ADC 原始值转换成物理量的浮点计算全部放在用户空间 APP。

