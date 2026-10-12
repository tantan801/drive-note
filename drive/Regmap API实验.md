# 第七十四章 Regmap API 实验

我们在前面学习 I2C 和 SPI 驱动的时候，针对 I2C 和 SPI 设备寄存器的操作都是通过相关的 API 函数进行操作的。这样 Linux 内核中就会充斥着大量的重复、冗余代码，但是这些本质上都是对寄存器的操作，所以为了方便内核开发人员统一访问 I2C/SPI 设备的时候，为此引入了 Regmap 子系统，本章我们就来学习一下如何使用 Regmap API 函数来读写 I2C/SPI 设备寄存器。

## 74.1 Regmap API 简介

### 74.1.1 什么是 Regmap

Linux 下大部分设备的驱动开发都是操作其内部寄存器，比如 I2C/SPI 设备的本质都是一样的，通过 I2C/SPI 接口读写芯片内部寄存器。芯片内部寄存器也是同样的道理，比如 I.MX6ULL 的 PWM、定时器等外设初始化，最终都是要落到寄存器的设置上。

Linux 下使用 `i2c_transfer` 来读写 I2C 设备中的寄存器，SPI 接口的话使用 `spi_write/spi_read` 等。I2C/SPI 芯片又非常的多，因此 Linux 内核里面就会充斥了大量的 `i2c_transfer` 这类的冗余代码，再者，代码的复用性也会降低。比如 icm20608 这个芯片既支持 I2C 接口，也支持 SPI 接口。假设我们在产品设计阶段一开始将 icm20608 设计为 SPI 接口，但是后面发现 SPI 接口不够用，或者 SOC 的引脚不够用，我们需要将 icm20608 改为 I2C 接口。这个时候 icm20608 的驱动就要大改，我们需要将 SPI 接口函数换为 I2C 的，工作量比较大。

基于代码复用的原则，Linux 内核引入了 regmap 模型，regmap 将寄存器访问的共同逻辑抽象出来，驱动开发人员不需要再去纠结使用 SPI 或者 I2C 接口 API 函数，统一使用 regmap API 函数。这样的好处就是统一使用 regmap，降低了代码冗余，提高了驱动的可移植性。regmap 模型的重点在于：通过 regmap 模型提供的统一接口函数来访问器件的寄存器，SOC 内部的寄存器也可以使用 regmap 接口函数来访问。

regmap 是 Linux 内核为了减少慢速 I/O 在驱动上的冗余开销，提供了一种通用的接口来操作硬件寄存器。另外，regmap 在驱动和硬件之间添加了 cache，降低了低速 I/O 的操作次数，提高了访问效率，缺点是实时性会降低。

什么情况下会使用 regmap：
①、硬件寄存器操作，比如选用通过 I2C/SPI 接口来读写设备的内部寄存器，或者需要读写 SOC 内部的硬件寄存器。
②、提高代码复用性和驱动一致性，简化驱动开发过程。
③、减少底层 I/O 操作次数，提高访问效率。

本章教程我们就来重点学习一下如何将《第六十二章 Linux SPI 驱动实验》中编写的 SPI 接口的 icm20608 驱动改为使用 regmap API。

### 74.1.2 Regmap 驱动框架

#### 1、regmap 框架结构
regmap 框架分为三层：
①、底层物理总线：regmap 就是对不同的物理总线进行封装，目前 regmap 支持的物理总线有 i2c、i3c、spi、mmio、sccb、sdw、slimbus、irq、spmi 和 w1。
②、regmap 核心层，用于实现 regmap，我们不用关心具体实现。
③、regmap API 抽象层，regmap 向驱动编写人员提供的 API 接口，驱动编写人员使用这些 API 接口来操作具体的芯片设备，也是驱动编写人员重点要掌握的。

#### 2、regmap 结构体
Linux 内核将 regmap 框架抽象为 regmap 结构体，这个结构体定义在文件 `drivers/base/regmap/internal.h` 中，结构体内容如下(有缩减)：

**示例代码 74.1.2.1 regmap 结构体**

```c
struct regmap {
	union {
		struct mutex mutex;
		struct {
			spinlock_t spinlock;
			unsigned long spinlock_flags;
		};
	};
	regmap_lock lock;
	regmap_unlock unlock;
	void *lock_arg; /* This is passed to lock/unlock functions */

	struct device *dev; /* Device we do I/O on */
	void *work_buf; /* Scratch buffer used to format I/O */
	struct regmap_format format; /* Buffer format */
	const struct regmap_bus *bus;
	void *bus_context;
	const char *name;

	bool async;
	spinlock_t async_lock;
	wait_queue_head_t async_waitq;
	struct list_head async_list;
	struct list_head async_free;
	int async_ret;
......
	unsigned int max_register;
	bool (*writeable_reg)(struct device *dev, unsigned int reg);
	bool (*readable_reg)(struct device *dev, unsigned int reg);
	bool (*volatile_reg)(struct device *dev, unsigned int reg);
	bool (*precious_reg)(struct device *dev, unsigned int reg);
	const struct regmap_access_table *wr_table;
	const struct regmap_access_table *rd_table;
	const struct regmap_access_table *volatile_table;
	const struct regmap_access_table *precious_table;

	int (*reg_read)(void *context, unsigned int reg, unsigned int *val);
	int (*reg_write)(void *context, unsigned int reg, unsigned int val);
......
	struct rb_root range_tree;
	void *selector_work_buf; /* Scratch buffer used for selector */
};
```

要使用 regmap，肯定要先给驱动分配一个具体的 regmap 结构体实例，一会讲解如何分配 regmap 实例。大家可以看到示例代码 74.1.2.1 中第 90~100 行有很多的函数以及 table，这些需要驱动编写人员根据实际情况选择性的初始化，regmap 的初始化通过结构体 `regmap_config` 来完成。

#### 3、regmap_config 结构体
顾名思义，regmap_config 结构体就是用来初始化 regmap 的，这个结构体也定义在 `include/linux/regmap.h` 文件中，结构体内容如下：

**示例代码 74.1.2.2 regmap_config 结构体**

```c
struct regmap_config {
	const char *name;

	int reg_bits;
	int reg_stride;
	int pad_bits;
	int val_bits;

	bool (*writeable_reg)(struct device *dev, unsigned int reg);
	bool (*readable_reg)(struct device *dev, unsigned int reg);
	bool (*volatile_reg)(struct device *dev, unsigned int reg);
	bool (*precious_reg)(struct device *dev, unsigned int reg);
	regmap_lock lock;
	regmap_unlock unlock;
	void *lock_arg;

	int (*reg_read)(void *context, unsigned int reg, unsigned int *val);
	int (*reg_write)(void *context, unsigned int reg, unsigned int val);

	bool fast_io;

	unsigned int max_register;
	const struct regmap_access_table *wr_table;
	const struct regmap_access_table *rd_table;
	const struct regmap_access_table *volatile_table;
	const struct regmap_access_table *precious_table;
	const struct reg_default *reg_defaults;
	unsigned int num_reg_defaults;
	enum regcache_type cache_type;
	const void *reg_defaults_raw;
	unsigned int num_reg_defaults_raw;

	u8 read_flag_mask;
	u8 write_flag_mask;

	bool use_single_rw;
	bool can_multi_write;

	enum regmap_endian reg_format_endian;
	enum regmap_endian val_format_endian;

	const struct regmap_range_cfg *ranges;
	unsigned int num_ranges;
};
```

Linux 内核里面已经对 regmap_config 各个成员变量进行了详细的讲解，这里我们只看一些比较重要的：
- `name`：名字。
- `reg_bits`：寄存器地址位数，必填字段。
- `reg_stride`：寄存器地址步长。
- `pad_bits`：寄存器和值之间的填充位数。
- `val_bits`：寄存器值位数，必填字段。
- `writeable_reg`：可选的可写回调函数，寄存器可写的话此回调函数就会被调用，并返回 true。
- `readable_reg`：可选的可读回调函数，寄存器可读的话此回调函数就会被调用，并返回 true。
- `volatile_reg`：可选的回调函数，当寄存器值不能缓存的时候此回调函数就会被调用，并返回 true。
- `precious_reg`：当寄存器值不能被读出来的时候此回调函数会被调用，比如很多中断状态寄存器读清零，读这些寄存器就可以清除中断标志位，但是并没有读出这些寄存器内部的值。
- `reg_read`：可选的读操作回调函数，所有读寄存器的操作此回调函数就会执行。
- `reg_write`：可选的写操作回调函数，所有写寄存器的操作此回调函数就会执行。
- `fast_io`：快速 I/O，使用 spinlock 替代 mutex 来提升锁性能。
- `max_register`：有效的最大寄存器地址，可选。
- `wr_table`：可写的地址范围，为 regmap_access_table 结构体类型。后面的 rd_table、volatile_table、precious_table 同理。
- `reg_defaults`：寄存器默认值，为 reg_default 结构体类型，此结构体有两个成员变量：reg 和 def，reg 是寄存器地址，def 是默认值。
- `num_reg_defaults`：默认寄存器表中的元素个数。
- `read_flag_mask`：读标志掩码。
- `write_flag_mask`：写标志掩码。

关于 regmap_config 结构体成员变量就介绍这些，其他没有介绍的自行查阅 Linux 内核中的相关描述。

### 74.1.3 Regmap 操作函数

#### 1、Regmap 申请与初始化
前面说了，regmap 支持多种物理总线，比如 I2C 和 SPI，我们需要根据所使用的接口来选择合适的 regmap 初始化函数。Linux 内核提供了针对不同接口的 regmap 初始化函数，SPI 接口初始化函数为 `regmap_init_spi`，函数原型如下：

```c
struct regmap * regmap_init_spi(struct spi_device *spi,
				const struct regmap_config *config)
```

函数参数和返回值含义如下：
- `spi`：需要使用 regmap 的 spi_device。
- `config`：regmap_config 结构体，需要程序编写人员初始化一个 regmap_config 实例，然后将其地址赋值给此参数。
- 返回值：申请到的并进过初始化的 regmap。

I2C 接口的 regmap 初始化函数为 `regmap_init_i2c`，函数原型如下：

```c
struct regmap * regmap_init_i2c(struct i2c_client *i2c,
				const struct regmap_config *config)
```

函数参数和返回值含义如下：
- `i2c`：需要使用 regmap 的 i2c_client。
- `config`：regmap_config 结构体，需要程序编写人员初始化一个 regmap_config 实例，然后将其地址赋值给此参数。
- 返回值：申请到的并进过初始化的 regmap。

还有很多其他物理接口对应的 regmap 初始化函数，这里就不介绍了，大家直接查阅 Linux 内核即可，基本和 SPI/I2C 的初始化函数相同。

在退出驱动的时候需要释放掉申请到的 regmap，不管是什么接口，全部使用 `regmap_exit` 这个函数来释放 regmap，函数原型如下：

```c
void regmap_exit(struct regmap *map)
```

函数参数和返回值含义如下：
- `map`：需要释放的 regmap
- 返回值：无。

我们一般会在 probe 函数中初始化 regmap_config，然后申请并初始化 regmap。

#### 2、regmap 设备访问 API 函数
不管是 I2C 还是 SPI 等接口，还是 SOC 内部的寄存器，对于寄存器的操作就两种：读和写。regmap 提供了最核心的两个读写操作：`regmap_read` 和 `regmap_write`。这两个函数分别用来读/写寄存器，`regmap_read` 函数原型如下：

```c
int regmap_read(struct regmap *map,
		unsigned int reg,
		unsigned int *val)
```

函数参数和返回值含义如下：
- `map`：要操作的 regmap。
- `reg`：要读的寄存器。
- `val`：读到的寄存器值。
- 返回值：0，读取成功；其他值，读取失败。

`regmap_write` 函数原型如下：

```c
int regmap_write(struct regmap *map,
		unsigned int reg,
		unsigned int val)
```

函数参数和返回值含义如下：
- `map`：要操作的 regmap。
- `reg`：要写的寄存器。
- `val`：要写的寄存器值。
- 返回值：0，写成功；其他值，写失败。

在 `regmap_read` 和 `regmap_write` 的基础上还衍生出了其他一些 regmap 的 API 函数，首先是 `regmap_update_bits` 函数，此函数用来修改寄存器指定的 bit，函数原型如下：

```c
int regmap_update_bits (struct regmap *map,
			unsigned int reg,
			unsigned int mask,
			unsigned int val)
```

函数参数和返回值含义如下：
- `map`：要操作的 regmap。
- `reg`：要操作的寄存器。
- `mask`：掩码，需要更新的位必须在掩码中设置为 1。
- `val`：需要更新的位值。
- 返回值：0，写成功；其他值，写失败。

比如要将寄存器的 bit1 和 bit2 置 1，那么 mask 应该设置为 0X00000011，此时 val 的 bit1 和 bit2 应该设置为 1，也就是 0Xxxxxxx11。如果要清除寄存器的 bit4 和 bit7，那么 mask 应该设置为 0X10010000，val 的 bit4 和 bit7 设置为 0，也就是 0X0xx0xxxx。

接下来看一下 `regmap_bulk_read` 函数，此函数用于读取多个寄存器的值，函数原型如下：

```c
int regmap_bulk_read(struct regmap *map,
			unsigned int reg,
			void *val,
			size_t val_count)
```

函数参数和返回值含义如下：
- `map`：要操作的 regmap。
- `reg`：要读取的第一个寄存器。
- `val`：读取到的数据缓冲区。
- `val_count`：要读取的寄存器数量。
- 返回值：0，读成功；其他值，读失败。

另外也有多个寄存器写函数 `regmap_bulk_write`，函数原型如下：

```c
int regmap_bulk_write(struct regmap *map,
			unsigned int reg,
			const void *val,
			size_t val_count)
```

函数参数和返回值含义如下：
- `map`：要操作的 regmap。
- `reg`：要写的第一个寄存器。
- `val`：要写的寄存器数据缓冲区。
- `val_count`：要写的寄存器数量。
- 返回值：0，写成功；其他值，写失败。

关于 regmap 常用到 API 函数就讲解到这里，还有很多其他功能的 API 函数，大家自行查阅 Linux 内核即可，内核里面对每个 API 函数都有详细的讲解。

### 74.1.4 regmap_config 掩码设置

结构体 `regmap_config` 里面有两个关于掩码的成员变量：`read_flag_mask` 和 `write_flag_mask`，这二个掩码非常重要，本节我们来学习一下如何使用这两个掩码。我们在学习 icm20608 的时候讲过了，icm20608 支持 i2c 和 spi 接口，但是当使用 spi 接口的时候，读取 icm20608 寄存器的时候地址最高位必须置 1，写内部寄存器的是时候地址最高位要设置为 0。因此这里就涉及到对寄存器地址最高位的操作，在《第六十二章 SPI 驱动实验》中我们在使用 SPI 接口函数读取 icm20608 内部寄存器的时候手动将寄存器地址的最高位置 1，代码如下所示：

**示例代码 74.1.4.1 icm20608 驱动**

```c
static int icm20608_read_regs(struct icm20608_dev *dev, u8 reg,
void *buf, int len)
{
......
	txdata[0] = reg | 0x80; /* 写数据的时候首寄存器地址 bit7 要置 1 */
	t->tx_buf = txdata; /* 要发送的数据 */
	t->rx_buf = rxdata; /* 要读取的数据 */
	t->len = len+1; /* t->len=发送的长度+读取的长度 */
	spi_message_init(&m); /* 初始化 spi_message */
	spi_message_add_tail(t, &m);
	ret = spi_sync(spi, &m); /* 同步发送 */
......
	return ret;
}
```

示例代码 74.1.4.1 就是标准的 SPI 驱动，其中第 21 行将寄存器的地址 bit7 置 1，表示这是一个读操作。

当我们使用 regmap 的时候就不需要手动将寄存器地址的 bit7 置 1，在初始化 regmap_config 的时候直接将 `read_flag_mask` 设置为 0X80 即可，这样通过 regmap 读取 SPI 内部寄存器的时候就会将寄存器地址与 `read_flag_mask` 进行或运算，结果就是将 bit7 置 1，但是整个过程不需要我们来操作，全部由 regmap 框架来完成的。

同理 `write_flag_mask` 用法也一样的，只是 `write_flag_mask` 用于写寄存器操作。

打开 `regmap_spi.c` 文件，这个文件就是 regmap 的 spi 总线文件，找到如下所示内容：

**示例代码 74.1.4.2 regmap_spi.c 代码段**

```c
static struct regmap_bus regmap_spi = {
	.write = regmap_spi_write,
	.gather_write = regmap_spi_gather_write,
	.async_write = regmap_spi_async_write,
	.async_alloc = regmap_spi_async_alloc,
	.read = regmap_spi_read,
	.read_flag_mask = 0x80,
	.reg_format_endian_default = REGMAP_ENDIAN_BIG,
	.val_format_endian_default = REGMAP_ENDIAN_BIG,
};

struct regmap *regmap_init_spi(struct spi_device *spi,
const struct regmap_config *config)
{
	return regmap_init(&spi->dev, &regmap_spi, &spi->dev, config);
}
```

第 105~114 行初始化了一个 regmap_bus 实例：regmap_spi，我们重点看一下第 111 行中 `read_flag_mask` 默认为 0X80。注意，这里是将 regmap_bus 的 read_flag_mask 成员变量设置为 0X80。regmap_bus 结构体大家自行查看一下，这里就不讲了。

第 125~129 行为 `regmap_init_spi` 函数，前面说了要想在 spi 总线中使用 regmap 框架，首先要使用 `regmap_init_spi` 函数用于并申请一个 SPI 总线的 regmap。从第 128 行可以看出 `regmap_init_spi` 函数只是对 `regmap_init` 的简单封装，因此最终完成 regmap 申请并初始化的是 `regmap_init` 函数。在 `regmap_init` 函数中找到如下所示内容：

**示例代码 74.1.4.3 regmap_init 函数代码段**

```c
if (config->read_flag_mask || config->write_flag_mask) {
	map->read_flag_mask = config->read_flag_mask;
	map->write_flag_mask = config->write_flag_mask;
} else if (bus) {
	map->read_flag_mask = bus->read_flag_mask;
}
```

第 598~601 行就是用 regmap_config 中的读写掩码来初始化 regmap_bus 中的掩码。由于 `regmap_spi` 默认将 `read_flag_mask` 设置为 0X80，当你所使用的 SPI 设备不需要读掩码，在初始化 regmap_config 的时候一定要将 `read_flag_mask` 设置为 0X00。

regmap 框架就讲解到这里，接下来学习如何将《第六十二章 Linux SPI 驱动实验》中编写的 icm20608 驱动改为 regmap 框架。

## 74.2 实验程序编写

本实验不需要修改设备树，直接使用《第六十二章 Linux SPI 驱动实验》中的 ICM20608 设备树。注意！第六十二章以前版本教程里面没有使用 Linux 内核自带的片选信号，新版本教程(V1.6 及以后版本以后)改为了使用内部片选信号。因此如果你的 ICM20608 设备树节点按照以前教程编写的，在这里请将其参考新版教程修改。

本实验对应的例程路径为：开发板光盘-> 01、程序源码->02、Linux 驱动例程->26_regmap->spi。

#### 1、修改设备结构体，添加 regmap 和 regmap_config
regmap 框架的核心就是 regmap 和 regmap_config 结构体，我们一般都是在自定义的设备结构体里面添加这两个类型的成员变量，所以我们首先在 icm20608_dev 结构体里面添加 regmap 和 regmap_config，修改完成以后的 icm20608_dev 结构体内容如下：

**示例代码 74.2.1.1 修改后的 icm20608_dev 结构体**

```c
struct icm20608_dev {
	struct spi_device *spi;		/* spi 设备 */
	dev_t devid;				/* 设备号 */
	struct cdev cdev;			/* cdev */
	struct class *class;		/* 类 */
	struct device *device;		/* 设备 */
	struct device_node *nd;		/* 设备节点 */
	signed int gyro_x_adc;		/* 陀螺仪 X 轴原始值 */
	signed int gyro_y_adc;		/* 陀螺仪 Y 轴原始值 */
	signed int gyro_z_adc;		/* 陀螺仪 Z 轴原始值 */
	signed int accel_x_adc;		/* 加速度计 X 轴原始值 */
	signed int accel_y_adc;	/* 加速度计 Y 轴原始值 */
	signed int accel_z_adc;		/* 加速度计 Z 轴原始值 */
	signed int temp_adc;		/* 温度原始值 */
	struct regmap *regmap;
	struct regmap_config regmap_config;
};
```

第 15 行，regmap 指针变量，regmap 我们需要使用 `regmap_init_spi` 函数来申请和初始化，所以这里是指针类型。 第 16 行，regmap_config 结构体成员变量，用来配置 regmap。

#### 2、初始化 regmap
一般在 probe 函数中初始化 regmap，本章节就是 icm20608_probe 函数，初始化内容如下：

**示例代码 74.2.1.2 icm20608_probe 函数**

```c
static int icm20608_probe(struct spi_device *spi)
{
	int ret;
	struct icm20608_dev *icm20608dev;

	/* 分配 icm20608dev 对象的空间 */
	icm20608dev = devm_kzalloc(&spi->dev, sizeof(*icm20608dev), GFP_KERNEL);
	if(!icm20608dev)
		return -ENOMEM;

	/* 初始化 regmap_config 设置 */
	icm20608dev->regmap_config.reg_bits = 8;	/* 寄存器长度 8bit */
	icm20608dev->regmap_config.val_bits = 8;	/* 值长度 8bit */
	icm20608dev->regmap_config.read_flag_mask = 0x80; /* 读掩码 */

	/* 初始化 SPI 接口的 regmap */
	icm20608dev->regmap = regmap_init_spi(spi, &icm20608dev->regmap_config);
	if (IS_ERR(icm20608dev->regmap)) {
		return PTR_ERR(icm20608dev->regmap);
	}

	/* 注册字符设备驱动 */
	/* 1、创建设备号 */
	ret = alloc_chrdev_region(&icm20608dev->devid, 0, ICM20608_CNT, ICM20608_NAME);
	if(ret < 0) {
		pr_err("%s Couldn't alloc_chrdev_region, ret=%d\r\n", ICM20608_NAME, ret);
		goto del_regmap;
	}

......

	return 0;
destroy_class:
	device_destroy(icm20608dev->class, icm20608dev->devid);
del_cdev:
	cdev_del(&icm20608dev->cdev);
del_unregister:
	unregister_chrdev_region(icm20608dev->devid, ICM20608_CNT);
del_regmap:
	regmap_exit(icm20608dev->regmap);
	return -EIO;
}
```

第 11~14 行，regmap_config 的初始化，icm20608 的寄存器地址长度为 8bit，寄存器值也是8bit，因此 `reg_bits` 和 `val_bits` 都设置为 8。由于 icm20608 通过 SPI 接口读取的时候地址寄存器最高位要设置为 1，因此 `read_flag_mask` 设置为 0X80。 第 17 行，通过 `regmap_init_spi` 函数来申请并初始化 SPI 总线的 regmap。 第 70 行，如果要删除 regmap 就使用 `regmap_exit` 函数。

同理，在 remove 函数中要删除 probe 里面申请的 regmap，icm20608_remove 函数内容如下：

**示例代码 74.2.1.3 icm20608_remove 函数**

```c
static int icm20608_remove(struct spi_device *spi)
{
	struct icm20608_dev *icm20608dev = spi_get_drvdata(spi);

......

	/* 4、注销类 */
	class_destroy(icm20608dev->class);
	/* 5、删除 regmap */
	regmap_exit(icm20608dev->regmap);
	return 0;
}
```

卸载驱动的时候使用 `regmap_exit` 删除掉 probe 函数中申请的 regmap。

#### 3、读写设备内部寄存器
regmap 已经设置好了，接下来就是使用 regmap API 函数来读写 icm20608 内部寄存器了。以前我们使用 spi 驱动框架编写读写函数，现在直接使用 `regmap_read`、`regmap_write` 等函数即可，修改后的 icm20608 内部寄存器读写函数如下：

**示例代码 74.2.1.4 icm20608 寄存器读写函数**

```c
/*
 * @description : 读取 icm20608 指定寄存器值，读取一个寄存器
 * @param – dev : icm20608 设备
 * @param – reg : 要读取的寄存器
 * @return      : 读取到的寄存器值
 */
static unsigned char icm20608_read_onereg(struct icm20608_dev *dev, u8 reg)
{
	u8 ret;
	unsigned int data;

	ret = regmap_read(dev->regmap, reg, &data);
	return (u8)data;
}

/*
 * @description : 向 icm20608 指定寄存器写入指定的值，写一个寄存器
 * @param – dev : icm20608 设备
 * @param – reg : 要写的寄存器
 * @param – data : 要写入的值
 * @return      : 无
 */
static void icm20608_write_onereg(struct icm20608_dev *dev, u8 reg, u8 value)
{
	regmap_write(dev->regmap, reg, value);
}

/*
 * @description : 读取 ICM20608 的数据，读取原始数据，包括三轴陀螺仪、
 *              三轴加速度计和内部温度。
 * @param - dev : ICM20608 设备
 * @return : 无。
 */
void icm20608_readdata(struct icm20608_dev *dev)
{
	u8 ret;
	unsigned char data[14];

	ret = regmap_bulk_read(dev->regmap, ICM20_ACCEL_XOUT_H, data, 14);

	dev->accel_x_adc = (signed short)((data[0] << 8) | data[1]);
	dev->accel_y_adc = (signed short)((data[2] << 8) | data[3]);
	dev->accel_z_adc = (signed short)((data[4] << 8) | data[5]);
	dev->temp_adc    = (signed short)((data[6] << 8) | data[7]);
	dev->gyro_x_adc  = (signed short)((data[8] << 8) | data[9]);
	dev->gyro_y_adc  = (signed short)((data[10] << 8) | data[11]);
	dev->gyro_z_adc  = (signed short)((data[12] << 8) | data[13]);
}
```

第 7~14 行，`icm20608_read_onereg` 函数用于读取 icm20608 内部单个寄存器，这里直接使用 `regmap_read` 函数来完成寄存器读取操作。 第 24~27 行，`icm20608_write_onereg` 函数用于向 icm20608 指定寄存器写入数据，这里也直接使用 `regmap_write` 函数来完成写操作。 第 35~49 行，`icm20608_readdata` 函数用于读取 icm20608 内部陀螺仪、加速度计和温度计的数据，从 `ICM20_ACCEL_XOUT_H` 寄存器开始，连续读取 14 个寄存器。这里直接使用 `regmap_bulk_read` 函数实现多个寄存器的读取。

对比《第六十二章 Linux SPI 驱动实验》中的 icm20608 驱动，采用 regmap API 以后驱动程序精简了很多。具体涉及到 SPI 总线的部分全部由 regmap 来处理了，驱动编写人员不用管，极大的方便了我们的驱动编写。而且驱动的可移植性提高了很多，即使将来更换为 IIC 接口，也只需要更改很少的一部分即可。

## 74.3 运行测试

测试 APP 直接使用第六十二章编写的 `icm20608App.c` 即可。测试方法也和六十二章一样，输入如下命令：
```bash
depmod					//第一次加载驱动的时候需要运行此命令
modprobe icm20608.ko	//加载驱动模块
./icm20608App /dev/icm20608	//app 读取内部数据
```

如果 regmap API 工作正常，那么就会正确的初始化 icm20608，并且读出传感器数据，结果和第四十五章一样。

IIC 总线的 regmap 框架基本和 SPI 一样，只是需要使用 `regmap_init_i2c` 来申请并初始化对应的 regmap，同样都是使用 `regmap_read` 和 `regmap_write` 来读写 I2C 设备内部寄存器。这里我们也已经将《第六十一章 Linux I2C 驱动实验》中的 ap3216c 驱动改为了 regmap API 接口的，相应的驱动程序已经放到了开发板光盘中，路径为：开发板光盘->01、程序源码->02、Linux 驱动例程->26_regmap->iic。大家自行查阅，这里就不详细详解了。
