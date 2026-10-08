## 第五十四章 platform设备驱动实验

---

### 54.1 Linux 驱动的分离与分层

#### 54.1.1 驱动的分隔与分离
对于 Linux 这样一个成熟、庞大、复杂的操作系统，代码的重用性非常重要，否则的话就会在 Linux 内核中存在大量无意义的重复代码。尤其是驱动程序，因为驱动程序占用了 Linux 内核代码量的大头，如果不对驱动程序加以管理，任由重复的代码肆意增加，那么用不了多久 Linux 内核的文件数量就庞大到无法接受的地步。

假如现在有三个平台 A、B 和 C，这三个平台 (这里的平台说的是 SOC) 上都有 MPU6050 这 个 I2C 接口的六轴传感器，按照我们写裸机 I2C 驱动的时候的思路，每个平台都有一个 MPU6050 的驱动，因此编写出来的最简单的驱动框架如图 54-1所示：

![图54-1 传统I2C设备驱动](./photo/platform设备驱动实验/54-1%20传统I2C设备驱动.jpeg)

从图 54-1 可以看出，每种平台下都有一个主机驱动和设备驱动，主机驱动肯定是必须要的，毕竟不同的平台其 I2C 控制器不同。但是右侧的设备驱动就没必要每个平台都写一个，因为不管对于那个 SOC 来说，MPU6050 都是一样，通过 I2C 接口读写数据就行了，只需要一个 MPU6050 的驱动程序即可。

如果再来几个 I2C 设备，比如 AT24C02、FT5206 (电容触摸屏) 等，如果按照图 54-1中的写法，那么设备端的驱动将会重复的编写好几次。显然在 Linux 驱动程序中这种写法是不推荐的，最好的做法就是每个平台的 I2C 控制器都提供一个统一的接口 (也叫做主机驱动)，每个设备的话也只提供一个驱动程序 (设备驱动)，每个设备通过统一的 I2C 接口驱动来访问，这样就可以大大简化驱动文件，比如 54-1中三种平台下的 MPU6050 驱动 框架就可以简化为图 54-2 所示：

![图54-2 改进后设备驱动](./photo/platform设备驱动实验/54-2%20改进后的设备驱动.jpeg)

实际的 I2C 驱动设备肯定有很多种，不止 MPU6050 这一个，那么实际的驱动架构如图54-3所示

![图54-3 分隔后的驱动框架](./photo/platform设备驱动实验/54-3%20分隔后的驱动框架.jpeg)

这个就是驱动的分隔，也就是将主机驱动和设备驱动分隔开来，比如 I2C、SPI 等等都会采用驱动分隔的方式来简化驱动的开发。在实际的驱动开发中，一般 I2C 主机控制器驱动已经由半导体厂家编写好了，而设备驱动一般也由设备器件的厂家编写好了，我们只需要提供设备信息即可，比如 I2C 设备的话提供设备连接到了哪个 I2C 接口上，I2C 的速度是多少等等。相当于将设备信息从设备驱动中剥离开来，驱动使用标准方法去获取到设备信息 (比如从设备树中获取到设备信息)，然后根据获取到的设备信息来初始化设备。

这样就相当于驱动只负责驱动，设备只负责设备，想办法将两者进行匹配即可。这个就是 Linux 中的总线 (bus)、驱动 (driver) 和设备 (device) 模型，也就是常说的驱动分离。总线就是驱动和设备信息的月老，负责给两者牵线搭桥，如图 54-4所示

![图54-4 Linux总线、驱动和设备模式](./photo/platform设备驱动实验/54-4%20Linux总线、驱动和设备模式.jpeg)

当我们向系统注册一个驱动的时候，总线就会在右侧的设备中查找，看看有没有与之匹配 的设备，如果有的话就将两者联系起来。同样的，当向系统中注册一个设备的时候，总线就会在左侧的驱动中查找看有没有与之匹配的设备，有的话也联系起来。Linux 内核中大量的驱动 程序都采用总线、驱动和设备模型，我们一会要重点讲解的 platform 驱动就是这一思想下的产物。

---

#### 54.1.2 驱动的分层
上一小节讲了驱动的分隔与分离，本节我们来简单看一下驱动的分层不同的层负责不同的内容。同样的，Linux 下的驱动往往也是分层的，分层的目的也是为了在不同的层处理不同的内容。以其他书籍或者资料常常使用到的 input (输入子系统，后面会有专门的章节详细的讲解) 为例，简单介绍一下驱动的分层。input 子系统负责管理所有 跟输入有关的驱动，包括键盘、鼠标、触摸等，最底层的就是设备原始驱动，负责获取输入设备的原始值，获取到的输入事件上报给 input 核心层。input 核心层会处理各种 IO 模型，并且提 供 file_operations 操作集合。我们在编写输入设备驱动的时候只需要处理好输入事件的上报即 可，至于如何处理这些上报的输入事件那是上层去考虑的，我们不用管。可以看出借助分层模 型可以极大的简化我们的驱动编写，对于驱动编写来说非常的友好。

---

### 54.2 platform 平台驱动模型简介
前面我们讲了设备驱动的分离，并且引出了总线 (bus)、驱动 (driver) 和设备 (device) 模型，比如 I2C、SPI、USB 等总线。但是在 SOC 中有些外设是没有总线这个概念的，但是又要使用总线、驱动和设备模型该怎么办呢？为了解决此问题，Linux 提出了 platform 这个虚拟总线，相应的就有 platform_driver 和 platform_device。

---

#### 54.2.1 platform 总线
Linux 系统内核使用 bus_type 结构体表示总线，此结构体定义在文件 include/linux/device.h，bus_type 结构体内容如下：

**示例代码 54.2.1.1 bus_type 结构体代码段**

```c
1 struct bus_type {
2 	const char *name; /* 总线名字 */
3 	const char *dev_name;
4 	struct device *dev_root;
5 	struct device_attribute *dev_attrs;
6 	const struct attribute_group **bus_groups; /* 总线属性 */
7 	const struct attribute_group **dev_groups; /* 设备属性 */
8 	const struct attribute_group **drv_groups; /* 驱动属性 */
9
10 	int (*match)(struct device *dev, struct device_driver *drv);
11 	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
12 	int (*probe)(struct device *dev);
13 	int (*remove)(struct device *dev);
14 	void (*shutdown)(struct device *dev);
15
16 	int (*online)(struct device *dev);
17 	int (*offline)(struct device *dev);
18 	int (*suspend)(struct device *dev, pm_message_t state);
19 	int (*resume)(struct device *dev);
20 	const struct dev_pm_ops *pm;
21 	const struct iommu_ops *iommu_ops;
22 	struct subsys_private *p;
23 	struct lock_class_key lock_key;
24 };
```

第 10 行，match 函数，此函数很重要，单词 match 的意思就是 “匹配、相配”，因此此函数就是完成设备和驱动之间匹配的，总线就是使用 match 函数来根据注册的设备来查找对应的驱动，或者根据注册的驱动来查找相应的设备，因此每一条总线都必须实现此函数。match 函数有两个参数：dev 和 drv，这两个参数分别为 device 和 device_driver 类型，也就是设备和驱动。

platform 总线是 bus_type 的一个具体实例，定义在文件 drivers/base/platform.c，platform 总线定义如下： 

**示例代码 54.2.1.2 platform 总线实例**

```c
1 struct bus_type platform_bus_type = {
2 	.name = "platform",
3 	.dev_groups = platform_dev_groups,
4 	.match = platform_match,
5 	.uevent = platform_uevent,
6 	.pm = &platform_dev_pm_ops,
7 };
```

platform_bus_type 就是 platform 平台总线，其中 platform_match 就是匹配函数。我们来看一下驱动和设备是如何匹配的，platform_match 函数定义在文件 drivers/base/platform.c 中，函数内容如下所示： 

**示例代码 54.2.1.3 platform 总线实例**

```c
1 static int platform_match(struct device *dev, struct device_driver *drv)
2 {
3 	struct platform_device *pdev = to_platform_device(dev);
4 	struct platform_driver *pdrv = to_platform_driver(drv);
5
6 	/*When driver_override is set,only bind to the matching driver*/
7 	if (pdev->driver_override)
8 		return !strcmp(pdev->driver_override, drv->name);
9
10 	/* Attempt an OF style match first */
11 	if (of_driver_match_device(dev, drv))
12 		return 1;
13
14 	/* Then try ACPI style match */
15 	if (acpi_driver_match_device(dev, drv))
16 		return 1;
17
18 	/* Then try to match against the id table */
19 	if (pdrv->id_table)
20 		return platform_match_id(pdrv->id_table, pdev) != NULL;
21
22 	/* fall-back to driver name match */
23 	return (strcmp(pdev->name, drv->name) == 0);
24 }
```

驱动和设备的匹配有四种方法，我们依次来看一下： 
第 11~12 行，第一种匹配方式， OF 类型的匹配，也就是设备树采用的匹配方式，of_driver_match_device 函数定义在文件 include/linux/of_device.h 中。device_driver 结构体 (表示设备驱动) 中有个名为 of_match_table 的成员变量，此成员变量保存着驱动的 compatible 匹配表，设备树中的每个设备节点的 compatible 属性会和 of_match_table 表中的所有成员比较，查看是否有相同的条目，如果有的话就表示设备和此驱动匹配，设备和驱动匹配成功以后 probe 函数就会执行。

第 15~16 行，第二种匹配方式，ACPI 匹配方式。

第 19~20 行，第三种匹配方式，id_table 匹配，每个 platform_driver 结构体有一个 id_table 成员变量，顾名思义，保存了很多 id 信息。这些 id 信息存放着这个 platform 驱动所支持的驱动类型。

第 23 行，第四种匹配方式，如果第三种匹配方式的 id_table 不存在的话就直接比较驱动和设备的 name 字段，看看是不是相等，如果相等的话就匹配成功。

对于支持设备树的 Linux 版本号，一般设备驱动为了兼容性都支持设备树和无设备树两种匹配方式。也就是第一种匹配方式一般都会存在，第三种和第四种只要存在一种就可以，一般用的最多的还是第四种，也就是直接比较驱动和设备的 name 字段，毕竟这种方式最简单了。

---

#### 54.2.2 platform 驱动
platform_driver 结构体表示 platform 驱动，此结构体定义在文件 include/linux/platform_device.h 中，内容如下： 

**示例代码 54.2.2.1 platform_driver 结构体**

```c
1 struct platform_driver {
2 	int (*probe)(struct platform_device *);
3 	int (*remove)(struct platform_device *);
4 	void (*shutdown)(struct platform_device *);
5 	int (*suspend)(struct platform_device *, pm_message_t state);
6 	int (*resume)(struct platform_device *);
7 	struct device_driver driver;
8 	const struct platform_device_id *id_table;
9 	bool prevent_deferred_probe;
10 };
```

第 2 行，probe 函数，当驱动与设备匹配成功以后 probe 函数就会执行，非常重要的函数！！一般驱动的提供者会编写，如果自己要编写一个全新的驱动，那么 probe 就需要自行实现。

第 7 行，driver 成员，为 device_driver 结构体变量，Linux 内核里面大量使用到了面向对象的思维，device_driver 相当于基类，提供了最基础的驱动框架。plaform_driver 继承了这个基类，然后在此基础上又添加了一些特有的成员变量。 
第 8 行，id_table 表，也就是我们上一小节讲解 platform 总线匹配驱动和设备的时候采用的第三种方法，id_table 是个表 (也就是数组)，每个元素的类型为 platform_device_id，platform_device_id 结构体内容如下： 

**示例代码 54.2.2.2 platform_device_id 结构体**

```c
1 struct platform_device_id {
2 	char name[PLATFORM_NAME_SIZE];
3 	kernel_ulong_t driver_data;
4 };
```

device_driver 结构体定义在 include/linux/device.h，device_driver 结构体内容如下： 

**示例代码 54.2.2.3 device_driver 结构体**

```c
1 struct device_driver {
2 	const char *name;
3 	struct bus_type *bus;
4
5 	struct module *owner;
6 	const char *mod_name; /* used for built-in modules */
7
8 	bool suppress_bind_attrs; /* disables bind/unbind via sysfs */
9
10 	const struct of_device_id *of_match_table;
11 	const struct acpi_device_id *acpi_match_table;
12
13 	int (*probe) (struct device *dev);
14 	int (*remove) (struct device *dev);
15 	void (*shutdown) (struct device *dev);
16 	int (*suspend) (struct device *dev, pm_message_t state);
17 	int (*resume) (struct device *dev);
18 	const struct attribute_group **groups;
19
20 	const struct dev_pm_ops *pm;
21
22 	struct driver_private *p;
23 };
```

第 10 行，of_match_table 就是采用设备树的时候驱动使用的匹配表，同样是数组，每个匹配项都为 of_device_id 结构体类型，此结构体定义在文件 include/linux/mod_devicetable.h 中，内容如下： 

**示例代码 54.2.2.4 of_device_id 结构体**

```c
1 struct of_device_id {
2 	char name[32];
3 	char type[32];
4 	char compatible[128];
5 	const void *data;
6 };
```

第 4 行的 compatible 非常重要，因为对于设备树而言，就是通过设备节点的 compatible 属性值和 of_match_table 中每个项目的 compatible 成员变量进行比较，如果有相等的就表示设备和此驱动匹配成功。

在编写 platform 驱动的时候，首先定义一个 platform_driver 结构体变量，然后实现结构体中的各个成员变量，重点是实现匹配方法以及 probe 函数。当驱动和设备匹配成功以后 probe 函数就会执行，具体的驱动程序在 probe 函数里面编写，比如字符设备驱动等等。

当我们定义并初始化好 platform_driver 结构体变量以后，需要在驱动入口函数里面调用 platform_driver_register 函数向 Linux 内核注册一个 platform 驱动，platform_driver_register 函数原型如下所示：

```c
int platform_driver_register (struct platform_driver *driver)
```

函数参数和返回值含义如下： 
- driver：要注册的 platform 驱动。 
返回值：负数，失败；0，成功。

还需要在驱动卸载函数中通过 platform_driver_unregister 函数卸载 platform 驱动，platform_driver_unregister 函数原型如下：

```c
void platform_driver_unregister(struct platform_driver *drv)
```

函数参数和返回值含义如下： 
- drv：要卸载的 platform 驱动。 
返回值：无。

platform 驱动框架如下所示： 

**示例代码 54.2.2.5 platform 驱动框架**

```c
/* 设备结构体 */
1 struct xxx_dev{
2 	struct cdev cdev;
3 	/* 设备结构体其他具体内容 */
4 };
5
6 struct xxx_dev xxxdev; /* 定义个设备结构体变量 */
7
8 static int xxx_open(struct inode *inode, struct file *filp)
9 {
10 	/* 函数具体内容 */
11 	return 0;
12 }
13
14 static ssize_t xxx_write(struct file *filp, const char __user *buf, size_t cnt, loff_t *offt)
15 {
16 	/* 函数具体内容 */
17 	return 0;
18 }
19
20 /*
21  * 字符设备驱动操作集
22  */
23 static struct file_operations xxx_fops = {
24 	.owner = THIS_MODULE,
25 	.open = xxx_open,
26 	.write = xxx_write,
27 };
28
29 /*
30  * platform驱动的probe函数
31  * 驱动与设备匹配成功以后此函数就会执行
32  */
33 static int xxx_probe(struct platform_device *dev)
34 {
35 	......
36 	cdev_init(&xxxdev.cdev, &xxx_fops); /* 注册字符设备驱动 */
37 	/* 函数具体内容 */
38 	return 0;
39 }
40
41 static int xxx_remove(struct platform_device *dev)
42 {
43 	......
44 	cdev_del(&xxxdev.cdev);/* 删除cdev */
45 	/* 函数具体内容 */
46 	return 0;
47 }
48
49 /* 匹配列表 */
50 static const struct of_device_id xxx_of_match[] = {
51 	{ .compatible = "xxx-gpio" },
52 	{ /* Sentinel */ }
53 };
54
55 /*
56  * platform平台驱动结构体
57  */
58 static struct platform_driver xxx_driver = {
59 	.driver = {
60 		.name = "xxx",
61 		.of_match_table = xxx_of_match,
62 	},
63 	.probe = xxx_probe,
64 	.remove = xxx_remove,
65 };
66
67 /* 驱动模块加载 */
68 static int __init xxxdriver_init(void)
69 {
70 	return platform_driver_register(&xxx_driver);
71 }
72
73 /* 驱动模块卸载 */
74 static void __exit xxxdriver_exit(void)
75 {
76 	platform_driver_unregister(&xxx_driver);
77 }
78
79 module_init(xxxdriver_init);
80 module_exit(xxxdriver_exit);
81 MODULE_LICENSE("GPL");
82 MODULE_AUTHOR("zuozhongkai");
```

第 1~27 行，传统的字符设备驱动，所谓的 platform 驱动并不是独立于字符设备驱动、块设备驱动和网络设备驱动之外的其他种类的驱动。platform 只是为了驱动的分离与分层而提出来的一种框架，其驱动的具体实现还是需要字符设备驱动、块设备驱动或网络设备驱动。

第 33~39 行，xxx_probe 函数，当驱动和设备匹配成功以后此函数就会执行，以前在驱动入口 init 函数里面编写的字符设备驱动程序就全部放到此 probe 函数里面。比如注册字符设备驱动、添加 cdev、创建类等等。

第 41~47 行，xxx_remove 函数，platform_driver 结构体中的 remove 成员变量，当关闭 platform 设备驱动的时候此函数就会执行，以前在驱动卸载 exit 函数里面要做的事情就放到此函数中来。比如，使用 iounmap 释放内存、删除 cdev，注销设备号等等。

第 50~53 行，xxx_of_match 匹配表，如果使用设备树的话将通过此匹配表进行驱动和设备的匹配。第 51 行设置了一个匹配项，此匹配项的 compatible 值为 “xxx-gpio”，因此当设备树中设备节点的 compatible 属性值为 “xxx-gpio” 的时候此设备就会与此驱动匹配。第 52 行是一个标记，of_device_id 表最后一个匹配项必须是空的。

第 58~65 行，定义一个 platform_driver 结构体变量 xxx_driver，表示 platform 驱动，第 59~62 行设置 paltform_driver 中的 device_driver 成员变量的 name 和 of_match_table 这两个属性。其中 name 属性用于传统的驱动与设备匹配，也就是检查驱动和设备的 name 字段是不是相同。of_match_table 属性就是用于设备树下的驱动与设备检查。对于一个完整的驱动程序，必须提供有设备树和无设备树两种匹配方法。最后 63 和 64 这两行设置 probe 和 remove 这两成员变量。

第 68~71 行，驱动入口函数，调用 platform_driver_register 函数向 Linux 内核注册一个 platform 驱动，也就是上面定义的 xxx_driver 结构体变量。

第 74~77 行，驱动出口函数，调用 platform_driver_unregister 函数卸载前面注册的 platform 驱动。 
总体来说，platform 驱动还是传统的字符设备驱动、块设备驱动或网络设备驱动，只是套上了一张 “platform” 的皮，目的是为了使用总线、驱动和设备这个驱动模型来实现驱动的分离与分层。

---

#### 54.2.3 platform 设备
platform 驱动已经准备好了，我们还需要 platform 设备，否则的话单单一个驱动也做不了什么。platform_device 这个结构体表示 platform 设备，这里我们要注意，如果内核支持设备树的话就不要再使用 platform_device 来描述设备了，因为改用设备树去描述了。当然了，你如果一定要用 platform_device 来描述设备信息的话也是可以的。 

platform_device 结构体定义在文件 include/linux/platform_device.h 中，结构体内容如下： 

**示例代码 54.2.3.1 platform_device 结构体代码段**

```c
22 struct platform_device {
23 	const char *name;
24 	int id;
25 	bool id_auto;
26 	struct device dev;
27 	u32 num_resources;
28 	struct resource *resource;
29
30 	const struct platform_device_id *id_entry;
31 	char *driver_override; /* Driver name to force a match */
32
33 	/* MFD cell pointer */
34 	struct mfd_cell *mfd_cell;
35
36 	/* arch specific additions */
37 	struct pdev_archdata archdata;
38 };
```

第 23 行，name 表示设备名字，要和所使用的 platform 驱动的 name 字段相同，否则的话设备就无法匹配到对应的驱动。比如对应的 platform 驱动的 name 字段为 “xxx-gpio”，那么此 name 字段也要设置为 “xxx-gpio”。

第 27 行，num_resources 表示资源数量，一般为第 28 行 resource 资源的大小。

第 28 行，resource 表示资源，也就是设备信息，比如外设寄存器等。Linux 内核使用 resource 结构体表示资源，resource 结构体内容如下： 

**示例代码 54.2.3.2 resource 结构体代码段**

```c
18 struct resource {
19 	resource_size_t start;
20 	resource_size_t end;
21 	const char *name;
22 	unsigned long flags;
23 	struct resource *parent, *sibling, *child;
24 };
```

start 和 end 分别表示资源的起始和终止信息，对于内存类的资源，就表示内存起始和终止地址，name 表示资源名字，flags 表示资源类型，可选的资源类型都定义在了文件 include/linux/ioport.h 里面，如下所示： 

**示例代码 54.2.3.3 资源类型**

```c
29 #define IORESOURCE_BITS 0x000000ff /* Bus-specific bits */
30
31 #define IORESOURCE_TYPE_BITS 0x00001f00 /* Resource type */
32 #define IORESOURCE_IO 0x00000100 /* PCI/ISA I/O ports */
33 #define IORESOURCE_MEM 0x00000200
34 #define IORESOURCE_REG 0x00000300 /* Register offsets */
35 #define IORESOURCE_IRQ 0x00000400
36 #define IORESOURCE_DMA 0x00000800
37 #define IORESOURCE_BUS 0x00001000
......
104 /* PCI control bits. Shares IORESOURCE_BITS with above PCI ROM. */
105 #define IORESOURCE_PCI_FIXED (1<<4) /* Do not move resource */
```

在以前不支持设备树的 Linux 版本中，用户需要编写 platform_device 变量来描述设备信息，然后使用 platform_device_register 函数将设备信息注册到 Linux 内核中，此函数原型如下所示：

```c
int platform_device_register(struct platform_device *pdev)
```

函数参数和返回值含义如下： 
- pdev：要注册的 platform 设备。 
返回值：负数，失败；0，成功。

如果不再使用 platform 的话可以通过 platform_device_unregister 函数注销掉相应的 platform 设备，platform_device_unregister 函数原型如下：

```c
void platform_device_unregister(struct platform_device *pdev)
```

函数参数和返回值含义如下： 
- pdev：要注销的 platform 设备。 
返回值：无。

platform 设备信息框架如下所示： 

**示例代码 54.2.3.4 platform 设备框架**

```c
1 /* 寄存器地址定义*/
2 #define PERIPH1_REGISTER_BASE (0X20000000) /* 外设1寄存器首地址 */
3 #define PERIPH2_REGISTER_BASE (0X020E0068) /* 外设2寄存器首地址 */
4 #define REGISTER_LENGTH 4
5
6 /* 资源 */
7 static struct resource xxx_resources[] = {
8 	[0] = {
9 		.start = PERIPH1_REGISTER_BASE,
10 		.end = (PERIPH1_REGISTER_BASE + REGISTER_LENGTH - 1),
11 		.flags = IORESOURCE_MEM,
12 	},
13 	[1] = {
14 		.start = PERIPH2_REGISTER_BASE,
15 		.end = (PERIPH2_REGISTER_BASE + REGISTER_LENGTH - 1),
16 		.flags = IORESOURCE_MEM,
17 	},
18 };
19
20 /* platform设备结构体 */
21 static struct platform_device xxxdevice = {
22 	.name = "xxx-gpio",
23 	.id = -1,
24 	.num_resources = ARRAY_SIZE(xxx_resources),
25 	.resource = xxx_resources,
26 };
27
28 /* 设备模块加载 */
29 static int __init xxxdevice_init(void)
30 {
31 	return platform_device_register(&xxxdevice);
32 }
33
34 /* 设备模块注销 */
35 static void __exit xxx_resourcesdevice_exit(void)
36 {
37 	platform_device_unregister(&xxxdevice);
38 }
39
40 module_init(xxxdevice_init);
41 module_exit(xxxdevice_exit);
42 MODULE_LICENSE("GPL");
43 MODULE_AUTHOR("zuozhongkai");
```

第 7~18 行，数组 xxx_resources 表示设备资源，一共有两个资源，分别为设备外设 1 和外设 2 的寄存器信息。因此 flags 都为 IORESOURCE_MEM，表示资源为内存类型的。

第 21~26 行，platform 设备结构体变量，注意 name 字段要和所使用的驱动中的 name 字段一致，否则驱动和设备无法匹配成功。num_resources 表示资源大小，其实就是数组 xxx_resources 的元素个数，这里用 ARRAY_SIZE 来测量一个数组的元素个数。

第 29~32 行，设备模块加载函数，在此函数中调用 platform_device_register 向 Linux 内核注册 platform 设备。

第 35~38 行，设备模块卸载函数，在此函数中调用 platform_device_unregister 从 Linux 内核中卸载 platform 设备。

示例代码 54.2.3.4 主要是在不支持设备树的 Linux 版本中使用的，当 Linux 内核支持了设备树以后就不需要用户手动去注册 platform 设备了。因为设备信息都放到了设备树中去描述，Linux 内核启动的时候会从设备树中读取设备信息，然后将其组织成 platform_device 形式，至于设备树到 platform_device 的具体过程就不去详细的追究了，感兴趣的可以去看一下，网上也有很多博客详细的讲解了整个过程。

关于 platform 下的总线、驱动和设备就讲解到这里，我们接下来就使用 platform 驱动框架来编写一个 LED 灯驱动，本章我们不使用设备树来描述设备信息，我们采用自定义 platform_device 这种 “古老” 方式来编写 LED 的设备信息。下一章我们来编写设备树下的 platform 驱动，这样我们就掌握了无设备树和有设备树这两种 platform 驱动的开发方式。

---

### 54.3 硬件原理图分析
参考之前的 led 原理图即可

---

### 54.4 试验程序编写
本实验对应的例程路径为：开发板光盘 -> 2、Linux 驱动例程 -> 17_platform。 本章实验我们需要编写一个驱动模块和一个设备模块，其中驱动模块是 platform 驱动程序，设备模块是 platform 的设备信息。当这两个模块都加载成功以后就会匹配成功，然后 platform 驱动模块中的 probe 函数就会执行，probe 函数中就是传统的字符设备驱动那一套。

#### 54.4.1 platform 设备与驱动程序编写
新建名为 "17_platform" 的文件夹，然后在 17_platform 文件夹里面创建 vscode 工程，工作区命名为 "platform"。新建名为 leddevice.c 和 leddriver.c 这两个文件，这两个文件分别为 LED 灯的 platform 设备文件和 LED 灯的 platform 的驱动文件。在 leddevice.c 中输入如下所示内容：

**示例代码 54.4.1.1 leddevice.c 文件代码段**
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
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_gpio.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>
#include <linux/platform_device.h>
/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : leddevice.c
作者 : 左忠凯
版本 : V1.0
描述 : platform设备
其他 : 无
论坛 : www.openedv.com
日志 : 初版V1.0 2019/8/13 左忠凯创建
***************************************************************/

/*
 * 寄存器地址定义
 */
#define CCM_CCGR1_BASE (0X020C406C)
#define SW_MUX_GPIO1_IO03_BASE (0X020E0068)
#define SW_PAD_GPIO1_IO03_BASE (0X020E02F4)
#define GPIO1_DR_BASE (0X0209C000)
#define GPIO1_GDIR_BASE (0X0209C004)
#define REGISTER_LENGTH 4

/* @description : 释放flatform设备模块的时候此函数会执行
 * @param - dev : 要释放的设备
 * @return : 无
 */
static void led_release(struct device *dev)
{
	printk("led device released!\r\n");
}

/*
 * 设备资源信息，也就是LED0所使用的所有寄存器
 */
static struct resource led_resources[] = {
	[0] = {
		.start = CCM_CCGR1_BASE,
		.end = (CCM_CCGR1_BASE + REGISTER_LENGTH - 1),
		.flags = IORESOURCE_MEM,
	},
	[1] = {
		.start = SW_MUX_GPIO1_IO03_BASE,
		.end = (SW_MUX_GPIO1_IO03_BASE + REGISTER_LENGTH - 1),
		.flags = IORESOURCE_MEM,
	},
	[2] = {
		.start = SW_PAD_GPIO1_IO03_BASE,
		.end = (SW_PAD_GPIO1_IO03_BASE + REGISTER_LENGTH - 1),
		.flags = IORESOURCE_MEM,
	},
	[3] = {
		.start = GPIO1_DR_BASE,
		.end = (GPIO1_DR_BASE + REGISTER_LENGTH - 1),
		.flags = IORESOURCE_MEM,
	},
	[4] = {
		.start = GPIO1_GDIR_BASE,
		.end = (GPIO1_GDIR_BASE + REGISTER_LENGTH - 1),
		.flags = IORESOURCE_MEM,
	},
};


/*
 * platform设备结构体
 */
static struct platform_device leddevice = {
	.name = "imx6ul-led",
	.id = -1,
	.dev = {
		.release = &led_release,
	},
	.num_resources = ARRAY_SIZE(led_resources),
	.resource = led_resources,
};

/*
 * @description : 设备模块加载
 * @param : 无
 * @return : 无
 */
static int __init leddevice_init(void)
{
	return platform_device_register(&leddevice);
}

/*
 * @description : 设备模块注销
 * @param : 无
 * @return : 无
 */
static void __exit leddevice_exit(void)
{
	platform_device_unregister(&leddevice);
}

module_init(leddevice_init);
module_exit(leddevice_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("zuozhongkai");
```

leddevice.c 文件内容就是按照示例代码 54.2.3.4 的 platform 设备模板编写的。 第 56~82 行，led_resources 数组，也就是设备资源，描述了 LED 所要使用到的寄存器信息，也就是 IORESOURCE_MEM 资源。 第 88~96，platform 设备结构体变量 leddevice，这里要注意 name 字段为 "imx6ul-led"，所以稍后编写 platform 驱动中的 name 字段也要为 "imx6ul-led"，否则设备和驱动匹配失败。 第 103~106 行，设备模块加载函数，在此函数里面通过 platform_device_register 向 Linux 内核注册 leddevice 这个 platform 设备。 第 113~116 行，设备模块卸载函数，在此函数里面通过 platform_device_unregister 从 Linux 内核中删除掉 leddevice 这个 platform 设备。

leddevice.c 文件编写完成以后就编写 leddriver.c 这个 platform 驱动文件，在 leddriver.c 里面输入如下内容：

**示例代码 54.4.1.2 leddriver.c 文件代码段**
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
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_gpio.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>
#include <linux/platform_device.h>
/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : leddriver.c
作者 : 左忠凯
版本 : V1.0
描述 : platform驱动
其他 : 无
论坛 : www.openedv.com
日志 : 初版V1.0 2019/8/13 左忠凯创建
***************************************************************/

#define LEDDEV_CNT 1 /* 设备号长度 */
#define LEDDEV_NAME "platled" /* 设备名字 */
#define LEDOFF 0
#define LEDON 1

/* 寄存器名 */
static void __iomem *IMX6U_CCM_CCGR1;
static void __iomem *SW_MUX_GPIO1_IO03;
static void __iomem *SW_PAD_GPIO1_IO03;
static void __iomem *GPIO1_DR;
static void __iomem *GPIO1_GDIR;

/* leddev设备结构体 */
struct leddev_dev{
	dev_t devid; /* 设备号 */
	struct cdev cdev; /* cdev */
	struct class *class; /* 类 */
	struct device *device; /* 设备 */
	int major; /* 主设备号 */
};

struct leddev_dev leddev; /* led设备 */

/*
 * @description : LED打开/关闭
 * @param - sta : LEDON(0) 打开LED，LEDOFF(1) 关闭LED
 * @return : 无
 */
void led0_switch(u8 sta)
{
	u32 val = 0;
	if(sta == LEDON){
		val = readl(GPIO1_DR);
		val &= ~(1 << 3);
		writel(val, GPIO1_DR);
	}else if(sta == LEDOFF){
		val = readl(GPIO1_DR);
		val|= (1 << 3);
		writel(val, GPIO1_DR);
	}
}

/*
 * @description : 打开设备
 * @param – inode : 传递给驱动的inode
 * @param - filp : 设备文件，file结构体有个叫做private_data的成员变量
 * 一般在open的时候将private_data指向设备结构体。
 * @return : 0 成功;其他 失败
 */
static int led_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &leddev; /* 设置私有数据 */
	return 0;
}

/*
 * @description : 向设备写数据
 * @param – filp : 设备文件，表示打开的文件描述符
 * @param - buf : 要写给设备写入的数据
 * @param - cnt : 要写入的数据长度
 * @param - offt : 相对于文件首地址的偏移
 * @return : 写入的字节数，如果为负值，表示写入失败
 */
static ssize_t led_write(struct file *filp, const char __user *buf, size_t cnt, loff_t *offt)
{
	int retvalue;
	unsigned char databuf[1];
	unsigned char ledstat;

	retvalue = copy_from_user(databuf, buf, cnt);
	if(retvalue < 0) {
		return -EFAULT;
	}

	ledstat = databuf[0]; /* 获取状态值 */
	if(ledstat == LEDON) {
		led0_switch(LEDON); /* 打开LED灯 */
	}else if(ledstat == LEDOFF) {
		led0_switch(LEDOFF); /* 关闭LED灯 */
	}
	return 0;
}

/* 设备操作函数 */
static struct file_operations led_fops = {
	.owner = THIS_MODULE,
	.open = led_open,
	.write = led_write,
};

/*
 * @description : flatform驱动的probe函数，当驱动与设备
 * 匹配以后此函数就会执行
 * @param - dev : platform设备
 * @return : 0，成功;其他负值,失败
 */
static int led_probe(struct platform_device *dev)
{
	int i = 0;
	int ressize[5];
	u32 val = 0;
	struct resource *ledsource[5];

	printk("led driver and device has matched!\r\n");
	/* 1、获取资源 */
	for (i = 0; i < 5; i++) {
		ledsource[i] = platform_get_resource(dev, IORESOURCE_MEM, i);
		if (!ledsource[i]) {
			dev_err(&dev->dev, "No MEM resource for always on\n");
			return -ENXIO;
		}
		ressize[i] = resource_size(ledsource[i]);
	}

	/* 2、初始化LED */
	/* 寄存器地址映射 */
	IMX6U_CCM_CCGR1 = ioremap(ledsource[0]->start, ressize[0]);
	SW_MUX_GPIO1_IO03 = ioremap(ledsource[1]->start, ressize[1]);
	SW_PAD_GPIO1_IO03 = ioremap(ledsource[2]->start, ressize[2]);
	GPIO1_DR = ioremap(ledsource[3]->start, ressize[3]);
	GPIO1_GDIR = ioremap(ledsource[4]->start, ressize[4]);

	val = readl(IMX6U_CCM_CCGR1);
	val &= ~(3 << 26); /* 清除以前的设置 */
	val |= (3 << 26); /* 设置新值 */
	writel(val, IMX6U_CCM_CCGR1);

	/* 设置GPIO1_IO03复用功能，将其复用为GPIO1_IO03 */
	writel(5, SW_MUX_GPIO1_IO03);
	writel(0x10B0, SW_PAD_GPIO1_IO03);

	/* 设置GPIO1_IO03为输出功能 */
	val = readl(GPIO1_GDIR);
	val &= ~(1 << 3); /* 清除以前的设置 */
	val |= (1 << 3); /* 设置为输出 */
	writel(val, GPIO1_GDIR);

	/* 默认关闭LED1 */
	val = readl(GPIO1_DR);
	val |= (1 << 3) ;
	writel(val, GPIO1_DR);

	/* 注册字符设备驱动 */
	/*1、创建设备号 */
	if (leddev.major) { /* 定义了设备号 */
		leddev.devid = MKDEV(leddev.major, 0);
		register_chrdev_region(leddev.devid, LEDDEV_CNT, LEDDEV_NAME);
	} else { /* 没有定义设备号 */
		alloc_chrdev_region(&leddev.devid, 0, LEDDEV_CNT, LEDDEV_NAME);
		leddev.major = MAJOR(leddev.devid);
	}

	/* 2、初始化cdev */
	leddev.cdev.owner = THIS_MODULE;
	cdev_init(&leddev.cdev, &led_fops);

	/* 3、添加一个cdev */
	cdev_add(&leddev.cdev, leddev.devid, LEDDEV_CNT);

	/* 4、创建类 */
	leddev.class = class_create(THIS_MODULE, LEDDEV_NAME);
	if (IS_ERR(leddev.class)) {
		return PTR_ERR(leddev.class);
	}

	/* 5、创建设备 */
	leddev.device = device_create(leddev.class, NULL, leddev.devid, NULL, LEDDEV_NAME);
	if (IS_ERR(leddev.device)) {
		return PTR_ERR(leddev.device);
	}

	return 0;
}

/*
 * @description :移除platform驱动的时候此函数会执行
 * @param - dev : platform设备
 * @return : 0，成功;其他负值,失败
 */
static int led_remove(struct platform_device *dev)
{
	iounmap(IMX6U_CCM_CCGR1);
	iounmap(SW_MUX_GPIO1_IO03);
	iounmap(SW_PAD_GPIO1_IO03);
	iounmap(GPIO1_DR);
	iounmap(GPIO1_GDIR);

	cdev_del(&leddev.cdev); /* 删除cdev */
	unregister_chrdev_region(leddev.devid, LEDDEV_CNT);
	device_destroy(leddev.class, leddev.devid);
	class_destroy(leddev.class);
	return 0;
}

/* platform驱动结构体 */
static struct platform_driver led_driver = {
	.driver = {
		.name = "imx6ul-led", /* 驱动名字，用于和设备匹配 */
	},
	.probe = led_probe,
	.remove = led_remove,
};

/*
 * @description : 驱动模块加载函数
 * @param : 无
 * @return : 无
 */
static int __init leddriver_init(void)
{
	return platform_driver_register(&led_driver);
}

/*
 * @description : 驱动模块卸载函数
 * @param : 无
 * @return : 无
 */
static void __exit leddriver_exit(void)
{
	platform_driver_unregister(&led_driver);
}

module_init(leddriver_init);
module_exit(leddriver_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("zuozhongkai");
```

leddriver.c 文件内容就是按照示例代码 54.2.2.5 的 platform 驱动模板编写的。 第 34~122 行，传统的字符设备驱动。 第 130~206 行，probe 函数，当设备和驱动匹配以后此函数就会执行，当匹配成功以后会在终端上输出 "led driver and device has matched!" 这样语句。在 probe 函数里面初始化 LED、注册字符设备驱动。也就是将原来在驱动加载函数里面做的工作全部放到 probe 函数里面完成。 第 213~226 行，remove 函数，当卸载 platform 驱动的时候此函数就会执行。在此函数里面释放内存、注销字符设备等。也就是将原来驱动卸载函数里面的工作全部都放到 remove 函数中完成。 第 229~235 行，platform_driver 驱动结构体，注意 name 字段为 "imx6ul-led"，和我们在 leddevice.c 文件里面设置的设备 name 字段一致。 第 242~245 行，驱动模块加载函数，在此函数里面通过 platform_driver_register 向 Linux 内核注册 led_driver 驱动。 第 252~255 行，驱动模块卸载函数，在此函数里面通过 platform_driver_unregister 从 Linux 内核卸载 led_driver 驱动。

#### 54.4.2 测试 APP 编写
测试 APP 的内容很简单，就是打开和关闭 LED 灯，新建 ledApp.c 这个文件，然后在里面输入如下内容：

**示例代码 54.4.2.1 ledApp.c 文件代码段**
```c
#include "stdio.h"
#include "unistd.h"
#include "sys/types.h"
#include "sys/stat.h"
#include "fcntl.h"
#include "stdlib.h"
#include "string.h"
/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : ledApp.c
作者 : 左忠凯
版本 : V1.0
描述 : platform驱动驱测试APP。
其他 : 无
使用方法 ：./ledApp /dev/platled 0 关闭LED
./ledApp /dev/platled 1 打开LED
论坛 : www.openedv.com
日志 : 初版V1.0 2019/8/16 左忠凯创建
***************************************************************/
#define LEDOFF 0
#define LEDON 1

/*
 * @description : main主程序
 * @param - argc : argv数组元素个数
 * @param - argv : 具体参数
 * @return : 0 成功;其他 失败
 */
int main(int argc, char *argv[])
{
	int fd, retvalue;
	char *filename;
	unsigned char databuf[2];

	if(argc != 3){
		printf("Error Usage!\r\n");
		return -1;
	}

	filename = argv[1];
	/* 打开led驱动 */
	fd = open(filename, O_RDWR);
	if(fd < 0){
		printf("file %s open failed!\r\n", argv[1]);
		return -1;
	}

	databuf[0] = atoi(argv[2]); /* 要执行的操作：打开或关闭 */
	retvalue = write(fd, databuf, sizeof(databuf));
	if(retvalue < 0){
		printf("LED Control Failed!\r\n");
		close(fd);
		return -1;
	}

	retvalue = close(fd); /* 关闭文件 */
	if(retvalue < 0){
		printf("file %s close failed!\r\n", argv[1]);
		return -1;
	}
	return 0;
}
```

ledApp.c 文件内容很简单，就是控制 LED 灯的亮灭，和第四十一章的测试 APP 基本一致，这里就不重复讲解了。

---

### 54.5 运行测试
#### 54.5.1 编译驱动程序和测试 APP
1、编译驱动程序 编写 Makefile 文件，本章实验的 Makefile 文件和第四十章实验基本一样，只是将 obj-m 变量的值改为 "leddevice.o leddriver.o"，Makefile 内容如下所示：

**示例代码 54.5.1.1 Makefile 文件**
```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)

obj-m := leddevice.o leddriver.o

build: kernel_modules

kernel_modules:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

第 4 行，设置 obj-m 变量的值为 "leddevice.o leddriver.o"。 
输入如下命令编译出驱动模块文件：
```shell
make -j32
```
编译成功以后就会生成 leddevice.ko、leddriver.ko 的驱动模块文件。

2、编译测试 APP 
输入如下命令编译测试 ledApp.c 这个测试程序：
```shell
arm-linux-gnueabihf-gcc ledApp.c -o ledApp
```
编译成功以后就会生成 ledApp 这个应用程序。

---

#### 54.5.2 运行测试
将上一小节编译出来 leddevice.ko、leddriver.ko 和 ledApp 拷贝到 rootfs/lib/modules/4.1.15 目录中，重启开发板，进入到目录 lib/modules/4.1.15 中，输入如下命令加载 leddevice.ko 设备模块和 leddriver.ko 这个驱动模块。
```shell
depmod //第一次加载驱动的时候需要运行此命令
modprobe leddevice.ko //加载设备模块
modprobe leddriver.ko //加载驱动模块
```

根文件系统中 /sys/bus/platform/ 目录下保存着当前板子 platform 总线下的设备和驱动，其中 devices 子目录为 platform 设备，drivers 子目录为 platform 驱动。查看 /sys/bus/platform/devices/ 目录，看看我们的设备是否存在，我们在 leddevice.c 中设置 leddevice (platform_device 类型) 的 name 字段为 "imx6ul-led"，也就是设备名字为 imx6ul-led，因此肯定在 /sys/bus/platform/devices/ 目录下存在一个名字 "imx6ul-led" 的文件，否则说明我们的设备模块加载失败。

同理，查看 /sys/bus/platform/drivers/ 目录，看一下驱动是否存在，我们在 leddriver.c 中设置 led_driver (platform_driver 类型) 的 name 字段为 "imx6ul-led"，因此会在 /sys/bus/platform/drivers/ 目录下存在名为 "imx6ul-led" 这个文件。

驱动模块和设备模块加载成功以后 platform 总线就会进行匹配，当驱动和设备匹配成功以后就会打印：
```shell
led driver and device has matched!
```

驱动和设备匹配成功以后就可以测试 LED 灯驱动了，输入如下命令打开 LED 灯：
```shell
./ledApp /dev/platled 1 //打开 LED 灯
```
再输入如下命令关闭 LED 灯：
```shell
./ledApp /dev/platled 0 //关闭 LED 灯
```

LED 灯能否打开和关闭，如果可以的话就说明驱动工作正常，如果要卸载驱动的话输入如下命令即可：
```shell
rmmod leddevice.ko
rmmod leddriver.ko
```
---
## 第五十五章 设备树下的 platform 驱动编写
上一章我们详细的讲解了 Linux 下的驱动分离与分层，以及总线、设备和驱动这样的驱动框架。基于总线、设备和驱动这样的驱动框架，Linux 内核提出来 platform 这个虚拟总线，相应的也有 platform 设备和 platform 驱动。上一章我们讲解了传统的、未采用设备树的 platform 设备和驱动编写方法。最新的 Linux 内核已经支持了设备树，因此在设备树下如何编写 platform 驱动就显得尤为重要，本章我们就来学习一下如何在设备树下编写 platform 驱动。

---

### 55.1 设备树下的 platform 驱动简介
platform 驱动框架分为总线、设备和驱动，其中总线不需要我们这些驱动程序员去管理，这个是 Linux 内核提供的，我们在编写驱动的时候只要关注于设备和驱动的具体实现即可。在没有设备树的 Linux 内核下，我们需要分别编写并注册 platform_device 和 platform_driver，分别代表设备和驱动。在使用设备树的时候，设备的描述被放到了设备树中，因此 platform_device 就不需要我们去编写了，我们只需要实现 platform_driver 即可。

在编写基于设备树的 platform 驱动的时候我们需要注意一下几点：
1.在设备树中创建设备节点 毫无疑问，肯定要先在设备树中创建设备节点来描述设备信息，重点是要设置好 compatible 属性的值，因为 platform 总线需要通过设备节点的 compatible 属性值来匹配驱动！这点要切记。比如，我们可以编写如下所示的设备节点来描述我们本章实验要用到的 LED 这个设备：

**示例代码 55.1.1 gpioled 设备节点**
```dts
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

示例 55.1.1 中的 gpioled 节点其实就是 45.4.1.2 小节中创建的 gpioled 设备节点，我们可以直接拿过来用。注意第 4 行的 compatible 属性值为 “atkalpha-gpioled”，因此一会在编写 platform 驱动的时候 of_match_table 属性表中要有 “atkalpha-gpioled”。

2.编写 platform 驱动的时候要注意兼容属性 上一章已经详细的讲解过了，在使用设备树的时候 platform 驱动会通过 of_match_table 来保存兼容性值，也就是表明此驱动兼容哪些设备。所以，of_match_table 将会尤为重要，比如本例程的 platform 驱动中 platform_driver 就可以按照如下所示设置：

**示例代码 55.1.2 of_match_table 匹配表的设置**
```c
static const struct of_device_id leds_of_match[] = {
    { .compatible = "atkalpha-gpioled" }, /* 兼容属性 */
    { /* Sentinel */ }
};

MODULE_DEVICE_TABLE(of, leds_of_match);

static struct platform_driver leds_platform_driver = {
    .driver = {
        .name = "imx6ul-led",
        .of_match_table = leds_of_match,
    },
    .probe = leds_probe,
    .remove = leds_remove,
};
```

第 1~4 行，of_device_id 表，也就是驱动的兼容表，是一个数组，每个数组元素为 of_device_id 类型。每个数组元素都是一个兼容属性，表示兼容的设备，一个驱动可以跟多个设备匹配。这里我们仅仅匹配了一个设备，那就是 55.1.1 中创建的 gpioled 这个设备。第 2 行的 compatible 值为 “atkalpha-gpioled”，驱动中的 compatible 属性和设备中的 compatible 属性相匹配，因此驱动中对应的 probe 函数就会执行。注意第 3 行是一个空元素，在编写 of_device_id 的时候最后一个元素一定要为空！

第 6 行，通过 MODULE_DEVICE_TABLE 声明一下 leds_of_match 这个设备匹配表。 
第 11 行，设置 platform_driver 中的 of_match_table 匹配表为上面创建的 leds_of_match，至此我们就设置好了 platform 驱动的匹配表了。

3.编写 platform 驱动 基于设备树的 platform 驱动和上一章无设备树的 platform 驱动基本一样，都是当驱动和设备匹配成功以后就会执行 probe 函数。我们需要在 probe 函数里面执行字符设备驱动那一套，当注销驱动模块的时候 remove 函数就会执行，都是大同小异的。

---

### 55.2 硬件原理图分析
本章实验我们只使用到 IMX6U-ALPHA 开发板上的 LED 灯，因此实验硬件原理图参考 LED小节即可。

---

### 55.3 实验程序编写
本实验对应的例程路径为：开发板光盘 -> 2、Linux 驱动例程 -> 18_dtsplatform。 本章实验我们编写基于设备树的 platform 驱动，所以需要在设备树中添加设备节点，然后我们只需要编写 platform 驱动即可。

#### 55.3.1 修改设备树文件
首先修改设备树文件，加上我们需要的设备信息，本章我们就使用到一个 LED 灯，因此可以直接使用 45.4.1 小节编写的 gpioled 子节点即可，不需要再重复添加。

#### 55.3.2 platform 驱动程序编写
设备已经准备好了，接下来就要编写相应的 platform 驱动了，新建名为 "18_dtsplatform" 的文件夹，然后在 18_dtsplatform 文件夹里面创建 vscode 工程，工作区命名为 "dtsplatform"。新建名为 leddriver.c 的驱动文件，在 leddriver.c 中输入如下所示内容：

**示例代码 55.3.2.1 leddriver.c 文件代码段**
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
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_gpio.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>
#include <linux/platform_device.h>
/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : leddriver.c
作者 : 左忠凯
版本 : V1.0
描述 : 设备树下的platform驱动
其他 : 无
论坛 : www.openedv.com
日志 : 初版V1.0 2019/8/13 左忠凯创建
***************************************************************/
#define LEDDEV_CNT 1 /* 设备号长度 */
#define LEDDEV_NAME "dtsplatled" /* 设备名字 */
#define LEDOFF 0
#define LEDON 1

/* leddev设备结构体 */
struct leddev_dev{
	dev_t devid; /* 设备号 */
	struct cdev cdev; /* cdev */
	struct class *class; /* 类 */
	struct device *device; /* 设备 */
	int major; /* 主设备号 */
	struct device_node *node; /* LED设备节点 */
	int led0; /* LED灯GPIO标号 */
};

struct leddev_dev leddev; /* led设备 */

/*
 * @description : LED打开/关闭
 * @param - sta : LEDON(0) 打开LED，LEDOFF(1) 关闭LED
 * @return : 无
 */
void led0_switch(u8 sta)
{
	if (sta == LEDON )
		gpio_set_value(leddev.led0, 0);
	else if (sta == LEDOFF)
		gpio_set_value(leddev.led0, 1);
}

/*
 * @description : 打开设备
 * @param – inode : 传递给驱动的inode
 * @param - filp : 设备文件，file结构体有个叫做private_data的成员变量
 * 一般在open的时候将private_data指向设备结构体。
 * @return : 0 成功;其他 失败
 */
static int led_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &leddev; /* 设置私有数据 */
	return 0;
}

/*
 * @description : 向设备写数据
 * @param - filp : 设备文件，表示打开的文件描述符
 * @param - buf : 要写给设备写入的数据
 * @param - cnt : 要写入的数据长度
 * @param – offt : 相对于文件首地址的偏移
 * @return : 写入的字节数，如果为负值，表示写入失败
 */
static ssize_t led_write(struct file *filp, const char __user *buf, size_t cnt, loff_t *offt)
{
	int retvalue;
	unsigned char databuf[2];
	unsigned char ledstat;

	retvalue = copy_from_user(databuf, buf, cnt);
	if(retvalue < 0) {
		printk("kernel write failed!\r\n");
		return -EFAULT;
	}

	ledstat = databuf[0];
	if (ledstat == LEDON) {
		led0_switch(LEDON);
	} else if (ledstat == LEDOFF) {
		led0_switch(LEDOFF);
	}
	return 0;
}

/* 设备操作函数 */
static struct file_operations led_fops = {
	.owner = THIS_MODULE,
	.open = led_open,
	.write = led_write,
};

/*
 * @description : flatform驱动的probe函数，当驱动与
 * 设备匹配以后此函数就会执行
 * @param - dev : platform设备
 * @return : 0，成功;其他负值,失败
 */
static int led_probe(struct platform_device *dev)
{
	printk("led driver and device was matched!\r\n");
	/* 1、设置设备号 */
	if (leddev.major) {
		leddev.devid = MKDEV(leddev.major, 0);
		register_chrdev_region(leddev.devid, LEDDEV_CNT, LEDDEV_NAME);
	} else {
		alloc_chrdev_region(&leddev.devid, 0, LEDDEV_CNT, LEDDEV_NAME);
		leddev.major = MAJOR(leddev.devid);
	}

	/* 2、注册设备 */
	cdev_init(&leddev.cdev, &led_fops);
	cdev_add(&leddev.cdev, leddev.devid, LEDDEV_CNT);

	/* 3、创建类 */
	leddev.class = class_create(THIS_MODULE, LEDDEV_NAME);
	if (IS_ERR(leddev.class)) {
		return PTR_ERR(leddev.class);
	}

	/* 4、创建设备 */
	leddev.device = device_create(leddev.class, NULL, leddev.devid, NULL, LEDDEV_NAME);
	if (IS_ERR(leddev.device)) {
		return PTR_ERR(leddev.device);
	}

	/* 5、初始化IO */
	leddev.node = of_find_node_by_path("/gpioled");
	if (leddev.node == NULL){
		printk("gpioled node nost find!\r\n");
		return -EINVAL;
	}

	leddev.led0 = of_get_named_gpio(leddev.node, "led-gpio", 0);
	if (leddev.led0 < 0) {
		printk("can't get led-gpio\r\n");
		return -EINVAL;
	}

	gpio_request(leddev.led0, "led0");
	gpio_direction_output(leddev.led0, 1); /*设置为输出，默认高电平 */
	return 0;
}

/*
 * @description : remove函数，移除platform驱动的时候此函数会执行
 * @param - dev : platform设备
 * @return : 0，成功;其他负值,失败
 */
static int led_remove(struct platform_device *dev)
{
	gpio_set_value(leddev.led0, 1); /* 卸载驱动的时候关闭LED */

	cdev_del(&leddev.cdev); /* 删除cdev */
	unregister_chrdev_region(leddev.devid, LEDDEV_CNT);
	device_destroy(leddev.class, leddev.devid);
	class_destroy(leddev.class);
	return 0;
}

/* 匹配列表 */
static const struct of_device_id led_of_match[] = {
	{ .compatible = "atkalpha-gpioled" },
	{ /* Sentinel */ }
};
MODULE_DEVICE_TABLE(of, led_of_match);

/* platform驱动结构体 */
static struct platform_driver led_driver = {
	.driver = {
		.name = "imx6ul-led", /* 驱动名字，用于和设备匹配 */
		.of_match_table = led_of_match, /* 设备树匹配表 */
	},
	.probe = led_probe,
	.remove = led_remove,
};

/*
 * @description : 驱动模块加载函数
 * @param : 无
 * @return : 无
 */
static int __init leddriver_init(void)
{
	return platform_driver_register(&led_driver);
}

/*
 * @description : 驱动模块卸载函数
 * @param : 无
 * @return : 无
 */
static void __exit leddriver_exit(void)
{
	platform_driver_unregister(&led_driver);
}

module_init(leddriver_init);
module_exit(leddriver_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("zuozhongkai");
```

第 33~112 行，传统的字符设备驱动，没什么要说的。 
第 120~164 行，platform 驱动的 probe 函数，当设备树中的设备节点与驱动之间匹配成功以后此函数就会执行，原来在驱动加载函数里面做的工作现在全部放到 probe 函数里面完成。 
第 171~180 行，remove 函数，当卸载 platform 驱动的时候此函数就会执行。在此函数里面释放内存、注销字符设备等，也就是将原来驱动卸载函数里面的工作全部都放到 remove 函数中完成。 
第 183~186 行，匹配表，描述了此驱动都和什么样的设备匹配，第 184 行添加了一条值为 "atkalpha-gpioled" 的 compatible 属性值，当设备树中某个设备节点的 compatible 属性值也为 “atkalpha-gpioled” 的时候就会与此驱动匹配。 
第 189~196 行，platform_driver 驱动结构体，191 行设置这个 platform 驱动的名字为 “imx6ul-led”，因此，当驱动加载成功以后就会在 /sys/bus/platform/drivers/ 目录下存在一个名为 “imx6ul-led” 的文件。 
第 192 行设置 of_match_table 为上面的 led_of_match。 
第 203~206 行，驱动模块加载函数，在此函数里面通过 platform_driver_register 向 Linux 内核注册 led_driver 驱动。 
第 213~216 行，驱动模块卸载函数，在此函数里面通过 platform_driver_unregister 从 Linux 内核卸载 led_driver 驱动。

#### 55.3.3 编写测试 APP
测试 APP 就直接使用上一章 54.4.2 小节编写的 ledApp.c 即可。

---

### 55.4 运行测试
#### 55.4.1 编译驱动程序和测试 APP
1、编译驱动程序 编写 Makefile 文件，本章实验的 Makefile 文件和第四十章实验基本一样，只是将 obj-m 变量的值改为 "leddriver.o"， Makefile 内容如下所示：

**示例代码 55.4.1.1 Makefile 文件**
```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)

obj-m := leddriver.o

build: kernel_modules

kernel_modules:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

第 4 行，设置 obj-m 变量的值为 "leddriver.o"。 
输入如下命令编译出驱动模块文件：
```shell
make -j32
```
编译成功以后就会生成一个名为 "leddriver.ko" 的驱动模块文件。

2、编译测试 APP 测试 APP 直接使用上一章的 ledApp 这个测试软件即可。

#### 55.4.2 运行测试
将上一小节编译出来 leddriver.ko 拷贝到 rootfs/lib/modules/4.1.15 目录中，重启开发板，进入到目录 lib/modules/4.1.15 中，输入如下命令加载 leddriver.ko 这个驱动模块。
```shell
depmod //第一次加载驱动的时候需要运行此命令
modprobe leddriver.ko //加载驱动模块
```

驱动模块加载完成以后到 /sys/bus/platform/drivers/ 目录下查看驱动是否存在，我们在 leddriver.c 中设置 led_driver (platform_driver 类型) 的 name 字段为 "imx6ul-led"，因此会在 /sys/bus/platform/drivers/ 目录下存在名为 "imx6ul-led" 这个文件。

同理，在 /sys/bus/platform/devices/ 目录下也存在 led 的设备文件，也就是设备树中 gpioled 这个节点。

驱动和模块都存在，当驱动和设备匹配成功以后就会打印：
led driver and device was matched!

驱动和设备匹配成功以后就可以测试 LED 灯驱动了，输入如下命令打开 LED 灯：
```shell
./ledApp /dev/dtsplatled 1 //打开 LED 灯
```
再输入如下命令关闭 LED 灯
```shell
./ledApp /dev/dtsplatled 0 //关闭 LED 灯
```

LED 灯能否打开和关闭，如果可以的话就说明驱动工作正常，如果要卸载驱动的话输入如下命令即可：
```shell
rmmod leddriver.ko
```