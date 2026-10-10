## 第六十一章 Linux I2C驱动实验
I2C 是很常用的一个串行通信接口，用于连接各种外设、传感器等器件，本章学习如何在Linux下开发I2C接口器件驱动，重点是学习Linux下I2C驱动框架，按照指定框架编写I2C设备驱动，本章同样以开发板上的AP3216C这个三合一环境光传感器为例，通过AP3216C讲解如何编写Linux下的I2C设备驱动程序。

---

### 61.1 Linux I2C驱动框架简介
回想我们在裸机如何编写AP3216C驱动，通过编写文件：bsp_i2c.c、 
bsp_i2c.h、bsp_ap3216c.c 和 bsp_ap3216c.h。其中前2个是IIC接口驱动，后两个是AP3216C的I2C设备驱动文件即这四个文件构成了两部分驱动：
①I2C主机驱动、②I2C设备驱动。
对于I2C主机驱动，编写完成就不需要修改，而其他I2C设备直接调用主机驱动的API函数完成读写操作。正好符合Linux的驱动分离与分层思想，因此Linux内核也将I2C驱动分为两部分。
①I2C总线驱动，I2C总线驱动就是SOC的I2C控制器驱动，也叫做I2C适配器驱动；
②I2C设备驱动，I2C设备驱动就是针对具体I2C设备而编写的驱动。

---

#### 61.1.1 总线驱动
在讲 platform 的时候就说过，platform 是虚拟出来的一条总线，目的是为了实现总线、设备、驱动框架。对于 I2C 而言，不需要虚拟出一条总线，直接使用 I2C 总线即可。I2C总线驱动重点是I2C适配器（即SOC的I2C接口控制器）驱动，这里要用两个重要的数据结构：i2c_adapter和i2c_algorithm,Linux内核将适配器抽象成i2c_adapter，i2c_adapter 结构体定义在 include/linux/i2c.h 文件中，结构体内容
```c
// i2c_adapter：抽象SOC的I2C控制器
struct i2c_adapter {
    struct module *owner;
    unsigned int class;
    const struct i2c_algorithm *algo; /* 总线访问算法 */
    void *algo_data;
    struct rt_mutex bus_lock;
    int timeout;
    int retries;
    struct device dev;
    int nr;
    char name[48];
    struct completion dev_released;
    struct mutex userspace_clients_lock;
    struct list_head userspace_clients;
    struct i2c_bus_recovery_info *bus_recovery_info;
    const struct i2c_adapter_quirks *quirks;
};
```

i2c_algorithm 类型的指针变量 algo，对于一个 I2C 适配器，肯定要对外提供读 
写 API 函数，设备驱动程序可以使用这些 API 函数来完成读写操作。i2c_algorithm 就是 I2C 适配器与 IIC 设备进行通信的方法。
i2c_algorithm 结构体定义在 include/linux/i2c.h 文件中，内容如下(删除条件编译)：
```c
// i2c_algorithm：I2C适配器通信方法集
struct i2c_algorithm {
    // I2C主设备传输函数，完成I2C收发通信
    int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);
    // SMBus传输接口
    int (*smbus_xfer) (struct i2c_adapter *adap, u16 addr, unsigned short flags, char read_write,
                       u8 command, int size, union i2c_smbus_data *data);
    // 查询适配器支持哪些功能
    u32 (*functionality) (struct i2c_adapter *);
};
```

master_xfer 就是 I2C 适配器的传输函数，可以通过此函数来完成与 IIC 设备之间的通信。smbus_xfer 就是 SMBUS 总线的传输函数。
I2C 总线驱动，或者说 I2C 适配器驱动的主要工作就是初始化 i2c_adapter 结构体变量，然后设置 i2c_algorithm 中的 master_xfer 函数。完成以后通过 i2c_add_numbered_adapt 
er 或 i2c_add_adapter 这两个函数向系统注册设置好的 i2c_adapter，如果要删除 I2C 适配器的话使用 i2c_del_adapter 函数即可。
这三个函数的原型如下：
```c
// 动态总线号注册
int i2c_add_adapter(struct i2c_adapter *adapter);
// 静态指定总线号注册
int i2c_add_numbered_adapter(struct i2c_adapter *adap);
// 删除适配器
void i2c_del_adapter(struct i2c_adapter * adap);
```

前两个函数的区别在于 i2c_add_adapter 使用动态的总线号，而 i2c_add_numbered_adapter 
使用静态总线号。函数参数和返回值含义如下： 
adapter 或 adap：要添加到 Linux 内核中的 i2c_adapter，也就是 I2C 适配器。 
返回值：0，成功；负值，失败。
第三个函数参数和返回值含义如下： 
adap：要删除的 I2C 适配器。 
返回值：无。
*说明：I.MX6U 这类 SOC 的 I2C 适配器驱动 NXP 已经写好，我们不需要编写总线驱动，只需要写 I2C 设备驱动。

---

#### 61.1.2 I2C设备驱动
I2C 设备驱动重点关注两个数据结构：i2c_client 和 i2c_driver，根据总线、设备和驱动模型，i2c_client 就是描述设备信息的，i2c_driver 描述驱动内容，类似于 platform_driver。

**1）i2c_client结构体**
i2c_client结构体定义在include/linux/i2c.h文件中内容如下：
```c
// i2c_client：代表一个I2C从设备
struct i2c_client {
    unsigned short flags;
    unsigned short addr;        // 7位从设备地址，低7位有效
    char name[I2C_NAME_SIZE];
    struct i2c_adapter *adapter;// 所属I2C适配器
    struct device dev;
    int irq;                    // 设备中断号
    struct list_head detected;
};
```
一个设备对应一个 i2c_client，每检测到一个 I2C 设备就会给这个 I2C 设备分配一个 i2c_client。

**2）i2c_driver结构体**
i2c_driver 类似 platform_driver，i2c_driver 结构体定义在 include/linux/i2c.h 文件中，内容如下：
```c
// i2c_driver：I2C设备驱动主体
struct i2c_driver {
    unsigned int class;
    // 匹配成功后执行probe
    int (*probe)(struct i2c_client *, const struct i2c_device_id *);
    int (*remove)(struct i2c_client *);
    void (*shutdown)(struct i2c_client *);
    void (*alert)(struct i2c_client *, unsigned int data);
    int (*command)(struct i2c_client *client, unsigned int cmd, void *arg);
    struct device_driver driver;
    const struct i2c_device_id *id_table;  // 传统id匹配表
    int (*detect)(struct i2c_client *, struct i2c_board_info *);
    const unsigned short *address_list;
    struct list_head clients;
};
```
*probe：设备和驱动匹配成功后自动执行，同platform一样驱动probe进行初始化
*of_match_table：放在 driver 成员内，用于设备树 compatible 匹配
*id_table：无设备树时，旧版 id 匹配表

对于我们 I2C 设备驱动编写人来说，重点工作就是构建 i2c_driver，构建完成以后需要向 Linux 内核注册这个 i2c_driver。i2c_driver 注册函数为 int i2c_register_driver
另外 i2c_add_driver 也常常用于注册 i2c_driver，i2c_add_driver 是一个宏，注销 I2C 设备驱动的时候需要将前面注册的 i2c_driver 从 Linux 内核中注销掉，需要用到 i2c_del_driver 函数，
```c
int i2c_register_driver(struct module *owner, struct i2c_driver *driver);
// 宏封装，等价 i2c_register_driver(THIS_MODULE, driver)
#define i2c_add_driver(driver) i2c_register_driver(THIS_MODULE, driver)
void i2c_del_driver(struct i2c_driver *driver);
```

前者函数参数和返回值含义如下： 
owner：一般为 THIS_MODULE。 
driver：要注册的 i2c_driver。 
返回值：0，成功；负值，失败。
中者i2c_add_driver 就是对 i2c_register_driver 做了一个简单的封装，只有一个参数，就是要注册的 i2c_driver。
后者函数参数和返回值含义如下： 
driver：要注销的 i2c_driver。 
返回值：无。

**I2C完整地驱动注册模板**
```c
/* i2c驱动probe函数 */
static int xxx_probe(struct i2c_client *client, const struct i2c_device_id *id)
{
    /* 硬件初始化、字符设备注册等业务代码 */
    return 0;
}

/* i2c驱动remove函数 */
static int xxx_remove(struct i2c_client *client)
{
    /* 资源释放 */
    return 0;
}

/* 传统匹配ID列表 */
static const struct i2c_device_id xxx_id[] = {
    {"xxx", 0},
    {}
};

/* 设备树匹配列表 */
static const struct of_device_id xxx_of_match[] = {
    { .compatible = "xxx" },
    { /* Sentinel */ }
};

/* i2c驱动结构体 */
static struct i2c_driver xxx_driver = {
    .probe = xxx_probe,
    .remove = xxx_remove,
    .driver = {
        .owner = THIS_MODULE,
        .name = "xxx",
        .of_match_table = xxx_of_match,
    },
    .id_table = xxx_id,
};

/* 驱动入口 */
static int __init xxx_init(void)
{
    int ret = 0;
    ret = i2c_add_driver(&xxx_driver);
    return ret;
}

/* 驱动出口 */
static void __exit xxx_exit(void)
{
    i2c_del_driver(&xxx_driver);
}

module_init(xxx_init);
module_exit(xxx_exit);
```
要点：probe 里面实现字符设备整套逻辑，和 platform 驱动写法很接近。

---

#### 61.1.3 I2C设备和驱动匹配过程
I2C设备和驱动匹配过程是由I2C核心来完成的，drivers/i2c/i2c-core.c 就是 I2C 的核心部分，I2C 核心提供了一些与具体硬件无关的 API 函数，比如前面的
1、i2c_adapter 注册/注销函数 
```c
int i2c_add_adapter(struct i2c_adapter *adapter) 
int i2c_add_numbered_adapter(struct i2c_adapter *adap) 
void i2c_del_adapter(struct i2c_adapter * adap) 
```
2、i2c_driver 注册/注销函数 
```c
int i2c_register_driver(struct module *owner, struct i2c_driver *driver) 
int i2c_add_driver (struct i2c_driver *driver) 
void i2c_del_driver(struct i2c_driver *driver) 
```

设备和驱动的匹配过程也是由 I2C 总线完成的，I2C 总线的数据结构为 i2c_bus_type，定义在 drivers/i2c/i2c-core.c 文件，i2c_bus_type内容如下：
```c
struct bus_type i2c_bus_type = {
    .name = "i2c",
    .match = i2c_device_match,
    .probe = i2c_device_probe,
    .remove = i2c_device_remove,
    .shutdown = i2c_device_shutdown,
};
```

.match指向匹配函数I2C 总线的设备和驱动匹配函数，在这里就是 i2c_device_match 这个函数，此函数内容如下：
```c
static int i2c_device_match(struct device *dev, struct device_driver *drv)
{
    struct i2c_client *client = i2c_verify_client(dev);
    struct i2c_driver *driver;

    if (!client)
        return 0;

    /* 1、设备树OF匹配（优先）：对比compatible */
    if (of_driver_match_device(dev, drv))
        return 1;

    /* 2、ACPI匹配 */
    if (acpi_driver_match_device(dev, drv))
        return 1;

    driver = to_i2c_driver(drv);
    /* 3、传统id_table匹配，无设备树时使用 */
    if (driver->id_table)
        return i2c_match_id(driver->id_table, client) != NULL;

    return 0;
}
```
*of_driver_match_device 函数用于完成设备树设备和驱动匹配。比较 I2C 设备节点的 compatible 属性和 of_device_id 中的 compatible 属性是否相等，如果相当的话就表示 I2C设备和驱动匹配。
*acpi_driver_match_device 函数用于 ACPI 形式的匹配。
*i2c_match_id 函数用于传统的、无设备树的 I2C 设备和驱动匹配过程。比较 I2C 设备名字和 i2c_device_id 的 name 字段是否相等，相等的话就说明 I2C 设备和驱动匹配。

---

### 61.2 I.MX6U的I2C适配器驱动分析
I2C 设备驱动是需要用户根据不同的 I2C 设备去编写，而 I2C 适配器驱动一般都是 SOC 厂商去编写的，比如 NXP 就编写好了 I.MX6U 的I2C 适配器驱动。在 imx6ull.dtsi 文件中找到 I.MX6U 的 I2C1 控制器节点，节点内容如下
```dts
i2c1: i2c@021a0000 {
    #address-cells = <1>;
    #size-cells = <0>;
    compatible = "fsl,imx6ul-i2c", "fsl,imx21-i2c";
    reg = <0x021a0000 0x4000>;
    interrupts = <GIC_SPI 36 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_I2C1>;
    status = "disabled";
};
```

重点关注i2c1节点的compatible属性值通过compatible属性值可以在源码找到对应驱动文件，i2c1节点的compatible属性值有两个“fsl,imx6ul-i2c”和“fsl,imx21-i2c”，在 Linux 源码中搜索这两个字符串即可找到对应的驱动文件。
```c
static const struct of_device_id i2c_imx_dt_ids[] = {
    { .compatible = "fsl,imx1-i2c", .data = &imx1_i2c_hwdata, },
    { .compatible = "fsl,imx21-i2c", .data = &imx21_i2c_hwdata, },
    { .compatible = "fsl,vf610-i2c", .data = &vf610_i2c_hwdata, },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, i2c_imx_dt_ids);

static struct platform_driver i2c_imx_driver = {
    .probe = i2c_imx_probe,
    .remove = i2c_imx_remove,
    .driver = {
        .name = DRIVER_NAME,
        .owner = THIS_MODULE,
        .of_match_table = i2c_imx_dt_ids,
        .pm = IMX_I2C_PM,
    },
    .id_table = imx_i2c_devtype,
};

static int __init i2c_adap_imx_init(void)
{
    return platform_driver_register(&i2c_imx_driver);
}
subsys_initcall(i2c_adap_imx_init);

static void __exit i2c_adap_imx_exit(void)
{
    platform_driver_unregister(&i2c_imx_driver);
}
module_exit(i2c_adap_imx_exit);
```

I2C 适配器本身是 platform 驱动，挂载在 platform 总线上，初始化完成后向内核注册 i2c_adapter，接入 I2C 总线子系统。
*“fsl,imx21-i2c”属性值，设备树中 i2c1 节点的 compatible 属性值就是与此匹配 
上的。因此 i2c-imx.c 文件就是 I.MX6U 的 I2C 适配器驱动文件。
*当设备和驱动匹配成功以后 i2c_imx_probe 函数就会执行，i2c_imx_probe 函数 
就会完成 I2C 适配器初始化工作。
*i2c_imx_probe核心流程
```c
static int i2c_imx_probe(struct platform_device *pdev)
{
    const struct of_device_id *of_id = of_match_device(i2c_imx_dt_ids, &pdev->dev);
    struct imx_i2c_struct *i2c_imx;
    struct resource *res;
    void __iomem *base;
    int irq, ret;

    // 获取中断号
    irq = platform_get_irq(pdev, 0);
    // 获取寄存器资源，内存映射得到虚拟地址
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(base))
        return PTR_ERR(base);

    // 分配imx_i2c_struct私有结构体
    i2c_imx = devm_kzalloc(&pdev->dev, sizeof(*i2c_imx), GFP_KERNEL);
    if (!i2c_imx)
        return -ENOMEM;

    // 初始化i2c_adapter，挂载i2c_algorithm通信方法
    strlcpy(i2c_imx->adapter.name, pdev->name, sizeof(i2c_imx->adapter.name));
    i2c_imx->adapter.owner = THIS_MODULE;
    i2c_imx->adapter.algo = &i2c_imx_algo;
    i2c_imx->adapter.dev.parent = &pdev->dev;
    i2c_imx->adapter.nr = pdev->id;
    i2c_imx->adapter.dev.of_node = pdev->dev.of_node;
    i2c_imx->base = base;

    // 获取并使能I2C时钟
    i2c_imx->clk = devm_clk_get(&pdev->dev, NULL);
    ret = clk_prepare_enable(i2c_imx->clk);

    // 注册I2C中断
    ret = devm_request_irq(&pdev->dev, irq, i2c_imx_isr, IRQF_NO_SUSPEND, pdev->name, i2c_imx);

    // 读取设备树clock-frequency属性，设置I2C通信速率，默认100K
    i2c_imx->bitrate = IMX_I2C_BIT_RATE;
    ret = of_property_read_u32(pdev->dev.of_node, "clock-frequency", &i2c_imx->bitrate);

    // 初始化I2C控制器寄存器
    imx_i2c_write_reg(i2c_imx->hwdata->i2cr_ien_opcode ^ I2CR_IEN,i2c_imx, IMX_I2C_I2CR);
    imx_i2c_write_reg(i2c_imx->hwdata->i2sr_clr_opcode, i2c_imx,IMX_I2C_I2SR);

    // 向内核注册I2C适配器
    ret = i2c_add_numbered_adapter(&i2c_imx->adapter);
    if (ret < 0) {
        dev_err(&pdev->dev, "registration failed\n");
        goto clk_disable;
    }

    // DMA资源申请
    i2c_imx_dma_request(i2c_imx, res->start);
    return 0;

clk_disable:
    clk_disable_unprepare(i2c_imx->clk);
    return ret;
}
```
probe 核心两件事：
1）初始化 i2c_adapter，绑定 i2c_algorithm，调用i2c_add_numbered_adapter注册适配器 
2）初始化 I2C 控制器寄存器、时钟、中断、DMA

*i2c_imx_algo通信算法结构体：
```c
static struct i2c_algorithm i2c_imx_algo = {
    .master_xfer = i2c_imx_xfer,
    .functionality = i2c_imx_func,
};
```
functionality：返回适配器支持的 I2C/SMBUS 能力即i2c_imx_func函数
```c
static u32 i2c_imx_func(struct i2c_adapter *adapter)
{
    return I2C_FUNC_I2C | I2C_FUNC_SMBUS_EMUL | I2C_FUNC_SMBUS_READ_BLOCK_DATA;
}
```

最后看i2c_imx_xfer函数，因为最终就是通过此函数来完成与I2C设备通信，此函数内容如下：
```c
static int i2c_imx_xfer(struct i2c_adapter *adapter, struct i2c_msg *msgs, int num)
{
    unsigned int i, temp;
    int result;
    bool is_lastmsg = false;
    struct imx_i2c_struct *i2c_imx = i2c_get_adapdata(adapter);

    /* 发起I2C起始信号 */
    result = i2c_imx_start(i2c_imx);
    if (result)
        goto fail0;

    /* 循环处理多条i2c_msg消息 */
    for (i = 0; i < num; i++) {
        if (i == num - 1)
            is_lastmsg = true;
        if (i) {
            // 重复起始信号
            temp = imx_i2c_read_reg(i2c_imx, IMX_I2C_I2CR);
            temp |= I2CR_RSTA;
            imx_i2c_write_reg(temp, i2c_imx, IMX_I2C_I2CR);
            result = i2c_imx_bus_busy(i2c_imx, 1);
            if (result)
                goto fail0;
        }
        // 判断读/写消息
        if (msgs[i].flags & I2C_M_RD)
            result = i2c_imx_read(i2c_imx, &msgs[i], is_lastmsg);
        else {
            // 写，满足阈值使用DMA写
            if (i2c_imx->dma && msgs[i].len >= DMA_THRESHOLD)
                result = i2c_imx_dma_write(i2c_imx, &msgs[i]);
            else
                result = i2c_imx_write(i2c_imx, &msgs[i]);
        }
        if (result)
            goto fail0;
    }

fail0:
    /* 停止I2C传输，发送停止位 */
    i2c_imx_stop(i2c_imx);
    return (result < 0) ? result : num;
}
```
i2c_imx_start、i2c_imx_read、i2c_imx_write 和 i2c_imx_stop 这些函数就是 I2C 寄存器的具体操作函数，自行研究源码即可。

---

### 61.3 I2C设备驱动编写流程
SOC 厂商已经完成 I2C 适配器驱动，我们只需要编写具体 I2C 外设的设备驱动。

---

#### 61.3.1 I2C设备信息描述
**1）未使用设备树**
是用i2c_board_info结构体在BSP中描述I2C设备信息
```c
struct i2c_board_info {
    char type[I2C_NAME_SIZE];     /* I2C设备名字 */
    unsigned short flags;         /* 标志 */
    unsigned short addr;          /* I2C器件地址 */
    void *platform_data;
    struct dev_archdata *archdata;
    struct device_node *of_node;
    struct fwnode_handle *fwnode;
    int irq;
};
```

type 和 addr 这两个成员变量是必须要设置的，一个是 I2C 设备的名字，一个是 I2C 设备的器件地址。打开 arch/arm/mach-imx/mach-mx27_3ds.c 文件，此文件中关于 OV2640 的 I2C 设备信息描述如下：
```c
#define I2C_BOARD_INFO(dev_type, dev_addr) \
.type = dev_type, .addr = (dev_addr)
```

示例代码 61.3.1.2 中使用 I2C_BOARD_INFO 来完成 mx27_3ds_i2c_camera 的初始化工作，I2C_BOARD_INFO 是一个宏，定义如下：
```c
static struct i2c_board_info mx27_3ds_i2c_camera = {
    I2C_BOARD_INFO("ov2640", 0x30),
};
```

在 Linux 源码里面全局搜索 i2c_board_info，会找到大量以 i2c_board_info 定义的 
I2C 设备信息，这些就是未使用设备树的时候 I2C 设备的描述方式，当采用了设备树以后就不会再使用 i2c_board_info 来描述 I2C 设备了。

**2）使用设备树**
在对应I2C控制器节点下创建子节点描述外设信息，示例 mag3110 磁力计NXP 官方的 EVK 开发板在 I2C1 上接了 mag3110 这个磁力计芯片，因此必须在 i2c1 节点下创建 mag3110 子节点，然后在这个子节点内描述 mag3110 这个芯片的相关信息。打开 imx6ull-14x14-evk.dts 这个设备树文件，然后找到如下内容：
```dts
&i2c1 {
    clock-frequency = <100000>;
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c1>;
    status = "okay";

    mag3110@0e {
        compatible = "fsl,mag3110";
        reg = <0x0e>;
        position = <2>;
    };
};
```
mag3110@0e：子节点名，@后面0e是 I2C 从机地址
compatible：匹配驱动
reg：I2C 从设备地址

---

#### 61.3.2 I2C设备数据首发处理流程
1.驱动入口：注册i2c_driver；设备驱动匹配成功后执行 probe 函数，probe 内部实现字符设备整套逻辑。
2.I2C 底层传输核心 API：i2c_transfer，最终调用适配器i2c_algorithm->master_xfer（IMX6U 为i2c_imx_xfer）
```c
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);
/*
adap：I2C适配器，i2c_client自带adapter指针
msgs：i2c_msg消息数组
num：消息数量
返回值：负数失败；非负=成功发送的msg个数
*/
```

来看一下 msgs 这个参数，这是一个 i2c_msg 类型的指针参数，I2C 进行数据收发 
说白了就是消息的传递，Linux 内核使用 i2c_msg 结构体来描述一个消息。i2c_msg 结构体定义在 include/uapi/linux/i2c.h 文件中，结构体内容如下：
```c
// i2c_msg 结构体，描述一条I2C消息
struct i2c_msg {
    __u16 addr;    /* 从机地址 */
    __u16 flags;   /* 标志 */
#define I2C_M_TEN	0x0010
#define I2C_M_RD	0x0001
#define I2C_M_STOP	0x8000
#define I2C_M_NOSTART	0x4000
#define I2C_M_REV_DIR_ADDR	0x2000
#define I2C_M_IGNORE_NAK	0x1000
#define I2C_M_NO_RD_ACK	0x0800
#define I2C_M_RECV_LEN	0x0400
    __u16 len;     /* 当前消息数据长度 */
    __u8 *buf;     /* 数据缓冲区 */
};
```

使用 i2c_transfer 函数发送数据之前要先构建好 i2c_msg，使用 i2c_transfer 进行 I2C 数据收发的示例代码
```c
/* 设备私有结构体 */
struct xxx_dev {
    ......
    void *private_data; /* 保存i2c_client */
};

/*
 * @description: 读取I2C设备多个寄存器数据
 * @param – dev : I2C设备
 * @param – reg : 要读取的寄存器首地址
 * @param – val : 读取到的数据存放缓冲区
 * @param – len : 要读取的数据长度
 * @return : 操作结果，0成功，负数失败
 */
static int xxx_read_regs(struct xxx_dev *dev, u8 reg, void *val, int len)
{
    int ret;
    struct i2c_msg msg[2];
    struct i2c_client *client = (struct i2c_client *)dev->private_data;

    /* msg[0]：写消息，发送寄存器地址 */
    msg[0].addr = client->addr;
    msg[0].flags = 0;
    msg[0].buf = &reg;
    msg[0].len = 1;

    /* msg[1]：读消息，读取寄存器数据 */
    msg[1].addr = client->addr;
    msg[1].flags = I2C_M_RD;
    msg[1].buf = val;
    msg[1].len = len;

    ret = i2c_transfer(client->adapter, msg, 2);
    if(ret == 2) {
        ret = 0;
    } else {
        ret = -EREMOTEIO;
    }
    return ret;
}

/*
 * @description: 向I2C设备多个寄存器写入数据
 * @param – dev : 要写入的设备结构体
 * @param – reg : 要写入的寄存器首地址
 * @param – buf : 待写入数据缓冲区
 * @param – len : 要写入的数据长度
 * @return : i2c_transfer返回值
 */
static s32 xxx_write_regs(struct xxx_dev *dev, u8 reg, u8 *buf, u8 len)
{
    u8 b[256];
    struct i2c_msg msg;
    struct i2c_client *client = (struct i2c_client *)dev->private_data;

    b[0] = reg;                  // 寄存器首地址
    memcpy(&b[1],buf,len);       // 拷贝待写入数据

    msg.addr = client->addr;
    msg.flags = 0;               // 写标记
    msg.buf = b;
    msg.len = len + 1;           // +1字节寄存器地址

    return i2c_transfer(client->adapter, &msg, 1);
}
```
*读寄存器：需要 2 条 i2c_msg，先发寄存器地址，再发起读（重复起始）
*写寄存器：1 条 i2c_msg，数据包首字节为寄存器地址，后面跟随写入数据

另外还有两个 API 函数分别用于 I2C 数据的收发操作，这两个函数最终都会调用 i2c_transfer。首先来看一下 I2C 数据发送函数 i2c_master_send，函数原型如下：
```c
// I2C发送数据 
int i2c_master_send(const struct i2c_client *client, const char *buf, int count);
```
函数参数和返回值含义如下： 
client：I2C 设备对应的 i2c_client。 
buf：要发送的数据。 
count：要发送的数据字节数，要小于 64KB，因为 i2c_msg 的 len 成员变量是一个 u16(无 
符号 16 位)类型的数据。 
返回值：负值，失败，其他非负值，发送的字节数。

I2C 数据接收函数为 i2c_master_recv，函数原型如下：
```c
// I2C接收数据 
int i2c_master_recv(const struct i2c_client *client, char *buf, int count);
```
函数参数和返回值含义如下： 
client：I2C 设备对应的 i2c_client。 
buf：要接收的数据。 
count：要接收的数据字节数，要小于 64KB，因为 i2c_msg 的 len 成员变量是一个 u16(无 
符号 16 位)类型的数据。 
返回值：负值，失败，其他非负值，发送的字节数。

---

### 61.4 硬件原理图
本实验用到资源： 
1、指示灯 LED0 
2、RGB LCD 屏幕 	
3、AP3216C 
4、串口
AP3216C 位于 I.MX6U-ALPHA 开发板底板，原理图如图 1-1。

![图1-1 AP3216C原理图](./photo/Linux%20I2C驱动实验/1-1%20AP3216C原理图.png)

AP3216C 挂载在 I2C1 总线：
*I2C1_SCL 复用 UART4_TXD
*I2C1_SDA 复用 UART4_RXD

---

### 61.5 实验程序编写
例程路径：开发板光盘 -> 2、Linux 驱动例程 -> 21_iic

---

#### 61.5.1 修该设备树
**1）修改IO配置**
首先肯定是要修改 IO，AP3216C 用到了 I2C1 接口，I.MX6U-ALPHA 开发板上的 I2C1 接口使用到了 UART4_TXD 和 UART4_RXD，因此肯定要在设备树里面设置这两个 IO。如果要用到 AP3216C 的中断功能的话还需要初始化 AP_INT 对应的 GIO1_IO01 这个 IO，本章实验我们不使用中断功能。此只需要设置 UART4_TXD 和 UART4_RXD 这两个 IO
```dts
pinctrl_i2c1: i2c1grp {
    fsl,pins = <
        MX6UL_PAD_UART4_TX_DATA__I2C1_SCL 0x4001b8b0
        MX6UL_PAD_UART4_RX_DATA__I2C1_SDA 0x4001b8b0
    >;
};
```
pinctrl_i2c1 就是 I2C1 的 IO 节点，这里将 UART4_TXD 和 UART4_RXD 这两个 IO 分别复用为 I2C1_SCL 和 I2C1_SDA，电气属性都设置为 0x4001b8b0。

**2）在i2c1节点追加ap3216c子节点**
删除原有 mag3110、fxls8471 节点，新增 ap3216c 子节点
```dts
&i2c1 {
    clock-frequency = <100000>;
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c1>;
    status = "okay";

    ap3216c@1e {
        compatible = "alientek,ap3216c";
        reg = <0x1e>;
    };
};
```

编译命令：
```shell
make dtbs
```
设备树生效后，/sys/bus/i2c/devices下出现0-001e目录，即为 AP3216C 设备。

#### 61.5.2 AP3216C 驱动编写
新建 21_iic 文件夹，创建ap3216creg.h、ap3216c.c 

**1）ap3216creg.h**
```c
#ifndef AP3216C_H
#define AP3216C_H
/* AP3216C寄存器 */
#define AP3216C_SYSTEMCONG 0x00   /* 配置寄存器 */
#define AP3216C_INTSTATUS  0X01   /* 中断状态寄存器 */
#define AP3216C_INTCLEAR   0X02   /* 中断清除寄存器 */
#define AP3216C_IRDATALOW  0x0A   /* IR数据低字节 */
#define AP3216C_IRDATAHIGH 0x0B   /* IR数据高字节 */
#define AP3216C_ALSDATALOW 0x0C   /* ALS数据低字节 */
#define AP3216C_ALSDATAHIGH 0X0D  /* ALS数据高字节 */
#define AP3216C_PSDATALOW  0X0E   /* PS数据低字节 */
#define AP3216C_PSDATAHIGH 0X0F   /* PS数据高字节 */
#endif
```

**2）ap3216c.c**
```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of_gpio.h>
#include <linux/semaphore.h>
#include <linux/timer.h>
#include <linux/i2c.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>
#include "ap3216creg.h"

#define AP3216C_CNT 1
#define AP3216C_NAME "ap3216c"

struct ap3216c_dev {
    dev_t devid;                /* 设备号 */
    struct cdev cdev;           /* cdev */
    struct class *class;        /* 类 */
    struct device *device;      /* 设备 */
    struct device_node *nd;     /* 设备节点 */
    int major;                  /* 主设备号 */
    void *private_data;         /* 私有数据，保存i2c_client */
    unsigned short ir, als, ps; /* 三个传感器数据 */
};

static struct ap3216c_dev ap3216cdev;

/*
 * @description : 从ap3216c读取多个寄存器数据
 * @param – dev  : ap3216c设备
 * @param – reg  : 要读取的寄存器首地址
 * @param – val  : 读取到的数据
 * @param – len  : 要读取的数据长度
 * @return : 操作结果
 */
static int ap3216c_read_regs(struct ap3216c_dev *dev, u8 reg, void *val, int len)
{
    int ret;
    struct i2c_msg msg[2];
    struct i2c_client *client = (struct i2c_client *)dev->private_data;

    /* msg[0]发送寄存器地址 */
    msg[0].addr = client->addr;
    msg[0].flags = 0;
    msg[0].buf = &reg;
    msg[0].len = 1;

    /* msg[1]读取寄存器数据 */
    msg[1].addr = client->addr;
    msg[1].flags = I2C_M_RD;
    msg[1].buf = val;
    msg[1].len = len;

    ret = i2c_transfer(client->adapter, msg, 2);
    if(ret == 2) {
        ret = 0;
    } else {
        printk("i2c rd failed=%d reg=%06x len=%d\n",ret, reg, len);
        ret = -EREMOTEIO;
    }
    return ret;
}

/*
 * @description : 向ap3216c多个寄存器写入数据
 * @param – dev  : ap3216c设备
 * @param – reg  : 要写入的寄存器首地址
 * @param – val  : 要写入的数据缓冲区
 * @param – len  : 要写入的数据长度
 * @return : 操作结果
 */
static s32 ap3216c_write_regs(struct ap3216c_dev *dev, u8 reg, u8 *buf, u8 len)
{
    u8 b[256];
    struct i2c_msg msg;
    struct i2c_client *client = (struct i2c_client *)dev->private_data;

    b[0] = reg;
    memcpy(&b[1],buf,len);

    msg.addr = client->addr;
    msg.flags = 0;
    msg.buf = b;
    msg.len = len + 1;

    return i2c_transfer(client->adapter, &msg, 1);
}

/* 读取单个寄存器 */
static unsigned char ap3216c_read_reg(struct ap3216c_dev *dev, u8 reg)
{
    u8 data = 0;
    ap3216c_read_regs(dev, reg, &data, 1);
    return data;
}

/* 写单个寄存器 */
static void ap3216c_write_reg(struct ap3216c_dev *dev, u8 reg, u8 data)
{
    u8 buf = 0;
    buf = data;
    ap3216c_write_regs(dev, reg, &buf, 1);
}

/* 读取AP3216C IR、ALS、PS原始数据 */
void ap3216c_readdata(struct ap3216c_dev *dev)
{
    unsigned char i =0;
    unsigned char buf[6];

    for(i = 0; i < 6; i++)
    {
        buf[i] = ap3216c_read_reg(dev, AP3216C_IRDATALOW + i);
    }

    if(buf[0] & 0X80)
        dev->ir = 0;
    else
        dev->ir = ((unsigned short)buf[1] << 2) | (buf[0] & 0X03);

    dev->als = ((unsigned short)buf[3] << 8) | buf[2];

    if(buf[4] & 0x40)
        dev->ps = 0;
    else
        dev->ps = ((unsigned short)(buf[5] & 0X3F) << 4) | (buf[4] & 0X0F);
}

static int ap3216c_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &ap3216cdev;
    ap3216c_write_reg(&ap3216cdev, AP3216C_SYSTEMCONG, 0x04);
    mdelay(50);
    ap3216c_write_reg(&ap3216cdev, AP3216C_SYSTEMCONG, 0X03);
    return 0;
}

static ssize_t ap3216c_read(struct file *filp, char __user *buf, size_t cnt, loff_t *off)
{
    short data[3];
    long err = 0;
    struct ap3216c_dev *dev = (struct ap3216c_dev *)filp->private_data;

    ap3216c_readdata(dev);
    data[0] = dev->ir;
    data[1] = dev->als;
    data[2] = dev->ps;
    err = copy_to_user(buf, data, sizeof(data));
    return 0;
}

static int ap3216c_release(struct inode *inode, struct file *filp)
{
    return 0;
}

static const struct file_operations ap3216c_ops = {
    .owner = THIS_MODULE,
    .open = ap3216c_open,
    .read = ap3216c_read,
    .release = ap3216c_release,
};

/* i2c匹配成功后执行probe */
static int ap3216c_probe(struct i2c_client *client, const struct i2c_device_id *id)
{
    if (ap3216cdev.major) {
        ap3216cdev.devid = MKDEV(ap3216cdev.major, 0);
        register_chrdev_region(ap3216cdev.devid, AP3216C_CNT, AP3216C_NAME);
    } else {
        alloc_chrdev_region(&ap3216cdev.devid, 0, AP3216C_CNT, AP3216C_NAME);
        ap3216cdev.major = MAJOR(ap3216cdev.devid);
    }

    cdev_init(&ap3216cdev.cdev, &ap3216c_ops);
    cdev_add(&ap3216cdev.cdev, ap3216cdev.devid, AP3216C_CNT);

    ap3216cdev.class = class_create(THIS_MODULE, AP3216C_NAME);
    if (IS_ERR(ap3216cdev.class)) {
        return PTR_ERR(ap3216cdev.class);
    }

    ap3216cdev.device = device_create(ap3216cdev.class, NULL,ap3216cdev.devid, NULL, AP3216C_NAME);
    if (IS_ERR(ap3216cdev.device)) {
        return PTR_ERR(ap3216cdev.device);
    }

    ap3216cdev.private_data = client;
    return 0;
}

static int ap3216c_remove(struct i2c_client *client)
{
    cdev_del(&ap3216cdev.cdev);
    unregister_chrdev_region(ap3216cdev.devid, AP3216C_CNT);
    device_destroy(ap3216cdev.class, ap3216cdev.devid);
    class_destroy(ap3216cdev.class);
    return 0;
}

/* 传统匹配ID表 */
static const struct i2c_device_id ap3216c_id[] = {
    {"alientek,ap3216c", 0},
    {}
};

/* 设备树匹配表 */
static const struct of_device_id ap3216c_of_match[] = {
    { .compatible = "alientek,ap3216c" },
    { /* Sentinel */ }
};

static struct i2c_driver ap3216c_driver = {
    .probe = ap3216c_probe,
    .remove = ap3216c_remove,
    .driver = {
        .owner = THIS_MODULE,
        .name = "ap3216c",
        .of_match_table = ap3216c_of_match,
    },
    .id_table = ap3216c_id,
};

static int __init ap3216c_init(void)
{
    int ret = 0;
    ret = i2c_add_driver(&ap3216c_driver);
    return ret;
}

static void __exit ap3216c_exit(void)
{
    i2c_del_driver(&ap3216c_driver);
}

module_init(ap3216c_init);
module_exit(ap3216c_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("zuozhongkai");
```


#### 61.5.3 编写测试APP
ap3216cApp.c
```c
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

int main(int argc, char *argv[])
{
    int fd;
    char *filename;
    unsigned short databuf[3];
    unsigned short ir, als, ps;
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
            ir = databuf[0];
            als = databuf[1];
            ps = databuf[2];
            printf("ir = %d, als = %d, ps = %d\r\n", ir, als, ps);
        }
        usleep(200000);
    }
    close(fd);
    return 0;
}
```

---

### 61.6 运行测试

---

#### 61.6.1 编译驱动程序和测试 APP
**1）Makefile**
```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)
obj-m := ap3216c.o

build:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules
clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```
编译驱动：
```shell
make -j32
```
生成ap3216c.ko

**2）交叉编译APP**
```shell
arm-linux-gnueabihf-gcc ap3216cApp.c -o ap3216cApp
```

---

#### 61.6.2 开发板测试
1. 将 ap3216c.ko、ap3216cApp 拷贝到开发板 /lib/modules/4.1.15
2. 加载模块
```shell
depmod
modprobe ap3216c.ko
```
3. 运行测试程序
```shell
./ap3216cApp /dev/ap3216c
```
测试：手电筒照射、手指靠近传感器，观察 ir/als/ps 数值变化。