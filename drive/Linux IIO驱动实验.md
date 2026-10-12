# 第七十五章 Linux IIO 驱动实验

工业场合里面也有大量的模拟量和数字量之间的转换，也就是我们常说的 ADC 和 DAC。而且随着手机、物联网、工业物联网和可穿戴设备的爆发，传感器的需求只持续增强。比如手机或者手环里面的加速度计、光传感器、陀螺仪、气压计、磁力计等，这些传感器本质上都是 ADC，大家注意查看这些传感器的手册，会发现他们内部都会有个 ADC，传感器对外提供 IIC或者 SPI 接口，SOC 可以通过 IIC 或者 SPI 接口来获取到传感器内部的 ADC 数值，从而得到想要测量的结果。Linux 内核为了管理这些日益增多的 ADC 类传感器，特地推出了 IIO 子系统，本章我们就来学习如何使用 IIO 子系统来编写 ADC 类传感器驱动。

## 75.1 IIO 子系统简介

IIO 全称是 Industrial I/O，翻译过来就是工业 I/O，大家不要看到"工业"两个字就觉得 IIO 是只用于工业领域的。大家一般在搜索 IIO 子系统的时候，会发现大多数讲的都是 ADC，这是因为 IIO 就是为 ADC 类传感器准备的，当然了 DAC 也是可以的。大家常用的陀螺仪、加速度计、电压/电流测量芯片、光照传感器、压力传感器等内部都是有个 ADC，内部 ADC 将原始的模拟数据转换为数字量，然后通过其他的通信接口，比如 IIC、SPI 等传输给 SOC。

因此，当你使用的传感器本质是 ADC 或 DAC 器件的时候，可以优先考虑使用 IIO 驱动框架。

### 75.1.1 iio_dev

#### 1、iio_dev 结构体
IIO 子系统使用结构体 iio_dev 来描述一个具体 IIO 设备，此设备结构体定义在 `include/linux/iio/iio.h` 文件中，结构体内容如下(有省略)：

**示例代码 75.1.1.1 iio_dev 结构体**

```c
struct iio_dev {
	int					id;

	int					modes;
	int					currentmode;
	struct device		dev;

	struct iio_event_interface *event_interface;

	struct iio_buffer	*buffer;
	struct list_head	buffer_list;
	int					scan_bytes;
	struct mutex		mlock;

	const unsigned long	*available_scan_masks;
	unsigned			masklength;
	const unsigned long	*active_scan_mask;
	bool				scan_timestamp;
	unsigned			scan_index_timestamp;
	struct iio_trigger	*trig;
	struct iio_poll_func *pollfunc;

	struct iio_chan_spec const *channels;
	int					num_channels;

	struct list_head	channel_attr_list;
	struct attribute_group chan_attr_group;
	const char			*name;
	const struct iio_info *info;
	struct mutex		info_exist_lock;
	const struct iio_buffer_setup_ops *setup_ops;
	struct cdev			chrdev;
......
};
```

我们来看一下 iio_dev 结构体中几个比较重要的成员变量：

第 477 行，modes 为设备支持的模式，可选择的模如表 75.1.1.1 所示：

| 模式 | 描述 |
|------|------|
| INDIO_DIRECT_MODE | 提供 sysfs 接口。 |
| INDIO_BUFFER_TRIGGERED | 支持硬件缓冲触发。 |
| INDIO_BUFFER_SOFTWARE | 支持软件缓冲触发。 |
| INDIO_BUFFER_HARDWARE | 支持硬件缓冲区。 |

表 75.1.1.1 模式

第 478 行，currentmode 为当前模式。 第 483 行，buffer 为缓冲区。 第 484 行，buffer_list 为当前匹配的缓冲区列表。 第 485 行，scan_bytes 为捕获到，并且提供给缓冲区的字节数。 第 488 行，available_scan_masks 为可选的扫描位掩码，使用触发缓冲区的时候可以通过设置掩码来确定使能哪些通道，使能以后的通道会将捕获到的数据发送到 IIO 缓冲区。 第 490 行，active_scan_mask 为缓冲区已经开启的通道掩码。只有这些使能了的通道数据才能被发送到缓冲区。 第 491 行，scan_timestamp 为扫描时间戳，如果使能以后会将捕获时间戳放到缓冲区里面。 第 493 行，trig 为 IIO 设备当前触发器，当使用缓冲模式的时候。 第 494 行，pollfunc 为一个函数，在接收到的触发器上运行。 第 496 行，channels 为 IIO 设备通道，为 iio_chan_spec 结构体类型，稍后会详细讲解 IIO通道。 第 497 行，num_channels 为 IIO 设备的通道数。 第 501 行，name 为 IIO 设备名字。 第 502 行，info 为 iio_info 结构体类型，这个结构体里面有很多函数，需要驱动开发人员编写，非常重要！我们从用户空间读取 IIO 设备内部数据，最终调用的就是 iio_info 里面的函数。稍后会详细讲解 iio_info 结构体。 第 504 行，setup_ops 为 iio_buffer_setup_ops 结构体类型，内容如下：

**示例代码 75.1.1.2 iio_buffer_setup_ops 结构体**

```c
struct iio_buffer_setup_ops {
	int (*preenable)(struct iio_dev *);		/* 缓冲区使能之前调用 */
	int (*postenable)(struct iio_dev *); 	/* 缓冲区使能之后调用 */
	int (*predisable)(struct iio_dev *); 	/* 缓冲区禁用之前调用 */
	int (*postdisable)(struct iio_dev *); 	/* 缓冲区禁用之后调用 */
	bool (*validate_scan_mask)(struct iio_dev *indio_dev,
		const unsigned long *scan_mask); 	/* 检查扫描掩码是否有效 */
};
```

可以看出 iio_buffer_setup_ops 里面都是一些回调函数，在使能或禁用缓冲区的时候会调用这些函数。如果未指定的话就默认使用 iio_triggered_buffer_setup_ops。

继续回到示例代码 75.1.1.1 中第 505 行，chrdev 为字符设备，由 IIO 内核创建。

#### 2、iio_dev 申请与释放
在使用之前要先申请 iio_dev，申请函数为 iio_device_alloc，函数原型如下：

```c
struct iio_dev *iio_device_alloc(int sizeof_priv)
```

函数参数和返回值含义如下：
- sizeof_priv：私有数据内存空间大小，一般我们会将自己定义的设备结构体变量作为 iio_dev 的私有数据，这样可以直接通过 iio_device_alloc 函数同时完成 iio_dev 和设备结构体变量的内存申请。申请成功以后使用 iio_priv 函数来得到自定义的设备结构体变量首地址。
- 返回值：如果申请成功就返回 iio_dev 首地址，如果失败就返回 NULL。

一般 iio_device_alloc 和 iio_priv 之间的配合使用如下所示：

**示例代码 75.1.1.3 iio_device_alloc 和 iio_priv 函数的使用**

```c
struct icm20608_dev *dev;
struct iio_dev *indio_dev;

/* 1、申请 iio_dev 内存 */
indio_dev = iio_device_alloc(sizeof(*dev));
if (!indio_dev)
	return -ENOMEM;

/* 2、获取设备结构体变量地址 */
dev = iio_priv(indio_dev);
```

第 1 行，icm20608_dev 是自定义的设备结构体。 第 2 行，indio_dev 是 iio_dev 结构体变量指针。 第 5 行，使用 iio_device_alloc 函数来申请 iio_dev，并且一起申请了 icm20608_dev 的内存。 第 10 行，使用 iio_priv 函数从 iio_dev 中提取出私有数据，也就是 icm20608_dev 这个自定义结构体变量首地址。

如果要释放 iio_dev，需要使用 iio_device_free 函数，函数原型如下：

```c
void iio_device_free(struct iio_dev *indio_dev)
```

函数参数和返回值含义如下：
- indio_dev：需要释放的 iio_dev。
- 返回值：无。

也可以使用 devm_iio_device_alloc 来分配 iio_dev，这样就不需要我们手动调用 iio_device_free 函数完成 iio_dev 的释放工作。

#### 3、iio_dev 注册与注销
前面分配好 iio_dev 以后就要初始化各种成员变量，初始化完成以后就需要将 iio_dev 注册到内核中，需要用到 iio_device_register 函数，函数原型如下：

```c
int iio_device_register(struct iio_dev *indio_dev)
```

函数参数和返回值含义如下：
- indio_dev：需要注册的 iio_dev。
- 返回值：0，成功；其他值，失败。

如果要注销 iio_dev 使用 iio_device_unregister 函数，函数原型如下：

```c
void iio_device_unregister(struct iio_dev *indio_dev)
```

函数参数和返回值含义如下：
- indio_dev：需要注销的 iio_dev。
- 返回值：0，成功；其他值，失败。

### 75.1.2 iio_info

iio_dev 有个成员变量：info，为 iio_info 结构体指针变量，这个是我们在编写 IIO 驱动的时候需要着重去实现的，因为用户空间对设备的具体操作最终都会反映到 iio_info 里面。iio_info结构体定义在 `include/linux/iio/iio.h` 中，结构体定义如下(有省略)：

**示例代码 75.1.2.1 iio_info 结构体**

```c
struct iio_info {
	struct module *driver_module;
	struct attribute_group *event_attrs;
	const struct attribute_group *attrs;

	int (*read_raw)(struct iio_dev *indio_dev,
		struct iio_chan_spec const *chan,
		int *val,
		int *val2,
		long mask);

	int (*write_raw)(struct iio_dev *indio_dev,
		struct iio_chan_spec const *chan,
		int val,
		int val2,
		long mask);

	int (*write_raw_get_fmt)(struct iio_dev *indio_dev,
		struct iio_chan_spec const *chan,
		long mask);
......
};
```

第 355 行，attrs 是通用的设备属性。 第 357 和 370 行，分别为 read_raw 和 write_raw 函数，这两个函数就是最终读写设备内部数据的操作函数，需要程序编写人员去实现的。比如应用读取一个陀螺仪传感器的原始数据，那么最终完成工作的就是 read_raw 函数，我们需要在 read_raw 函数里面实现对陀螺仪芯片的读取操作。同理，write_raw 是应用程序向陀螺仪芯片写数据，一般用于配置芯片，比如量程、数据速率等。这两个函数的参数都是一样的，我们依次来看一下：

- indio_dev：需要读写的 IIO 设备。
- chan：需要读取的通道。
- val，val2：对于 read_raw 函数来说 val 和 val2 这两个就是应用程序从内核空间读取到数据，一般就是传感器指定通道值，或者传感器的量程、分辨率等。对于 write_raw 来说就是应用程序向设备写入的数据。val 和 val2 共同组成具体值，val 是整数部分，val2 是小数部分。但是 val2 也是对具体的小数部分扩大 N 倍后的整数值，因为不能直接从内核向应用程序返回一个小数值。比如现在有个值为 1.00236，那么 val 就是 1，val2 理论上来讲是 0.00236，但是我们需要对 0.00236 扩大 N 倍，使其变为整数，这里我们扩大 1000000 倍，那么 val2 就是 2360。因此 val=1，val2=2360。扩大的倍数我们不能随便设置，而是要使用 Linux 定义的倍数，Linux 内核里面定义的数据扩大倍数，或者说数据组合形式如表 75.1.2.1 所示：

| 组合宏 | 描述 |
|--------|------|
| IIO_VAL_INT | 整数值，没有小数。比如 5000，那么就是 val=5000，不需要设置 val2 |
| IIO_VAL_INT_PLUS_MICRO | 小数部分扩大 1000000 倍，比如 1.00236，此时 val=1，val2=2360。 |
| IIO_VAL_INT_PLUS_NANO | 小数部分扩大 1000000000 倍，同样是 1.00236，此时 val=1，val2=2360000。 |
| IIO_VAL_INT_PLUS_MICRO_DB | dB 数据，和 IIO_VAL_INT_PLUS_MICRO 数据形式一样，只是在后面添加 db。 |
| IIO_VAL_INT_MULTIPLE | 多个整数值，比如一次要传回 6 个整数值，那么 val 和 val2 就不够用了。此宏主要用于 iio_info 的 read_raw_multi 函数。 |
| IIO_VAL_FRACTIONAL | 分数值，也就是 val/val2。比如 val=1，val2=4，那么实际值就是 1/4。 |
| IIO_VAL_FRACTIONAL_LOG2 | 值为 val>>val2，也就是 val 右移 val2 位。比如 val=25600，val2=4，那么真正的值就是 25600 >> 4 = 1600. |

表 75.1.2.1 数据组合表

- mask：掩码，用于指定我们读取的是什么数据，比如 ICM20608 这样的传感器，他既有原始的测量数据，比如 X,Y,Z 轴的陀螺仪、加速度计等，也有测量范围值，或者分辨率。比如加速度计测量范围设置为±16g，那么分辨率就是 32/65536≈0.000488，我们只有读出原始值以及对应的分辨率(量程)，才能计算出真实的重力加速度。此时就有两种数据值：传感器原始值、分辨率。Linux 内核使用 IIO_CHAN_INFO_RAW 和 IIO_CHAN_INFO_SCALE 这两个宏来表示原始值以及分辨率，这两个宏就是掩码。至于每个通道可以采用哪几种掩码，这个在我们初始化通道的时候需要驱动编写人员设置好。掩码有很多种，稍后讲解 IIO 通道的时候详细讲解！

第 376 行的 write_raw_get_fmt 用于设置用户空间向内核空间写入的数据格式，write_raw_get_fmt 函数决定了 write_raw 函数中 val 和 val2 的意义，也就是表 75.1.2.1 中的组合形式。比如我们需要在应用程序中设置 ICM20608 加速度计的量程为±8g，那么分辨率就是 16/65536≈0.000244，我们在 write_raw_get_fmt 函数里面设置加速度计的数据格式为 IIO_VAL_INT_PLUS_MICRO。那么我们在应用程序里面向指定的文件写入 0.000244 以后，最终传递给内核驱动的就是 0.000244*1000000=244。也就是 write_raw 函数的 val 参数为 0，val2 参数为 244。

### 75.1.3 iio_chan_spec

IIO 的核心就是通道，一个传感器可能有多路数据，比如一个 ADC 芯片支持 8 路采集，那么这个 ADC 就有 8 个通道。我们本章实验用到的 ICM20608，这是一个六轴传感器，可以输出三轴陀螺仪(X、Y、Z)、三轴加速度计(X、Y、Z)和一路温度，也就是一共有 7 路数据，因此就有 7 个通道。注意，三轴陀螺仪或加速度计的 X、Y、Z 这三个轴，每个轴都算一个通道。

Linux 内核使用 iio_chan_spec 结构体来描述通道，定义在 `include/linux/iio/iio.h` 文件中，内容如下：

**示例代码 75.1.3.1 iio_chan_spec 结构体**

```c
struct iio_chan_spec {
	enum iio_chan_type type;
	int			channel;
	int			channel2;
	unsigned long	address;
	int			scan_index;
	struct {
		char		sign;
		u8			realbits;
		u8			storagebits;
		u8			shift;
		u8			repeat;
		enum iio_endian endianness;
	} scan_type;
	long			info_mask_separate;
	long			info_mask_shared_by_type;
	long			info_mask_shared_by_dir;
	long			info_mask_shared_by_all;
	const struct iio_event_spec *event_spec;
	unsigned int	num_event_specs;
	const struct iio_chan_spec_ext_info *ext_info;
	const char		*extend_name;
	const char		*datasheet_name;
	unsigned		modified:1;
	unsigned		indexed:1;
	unsigned		output:1;
	unsigned		differential:1;
};
```

来看一下 iio_chan_spec 结构体中一些比较重要的成员变量：

第 224 行，type 为通道类型， iio_chan_type 是一个枚举类型，列举出了可以选择的通道类型，定义在 `include/uapi/linux/iio/types.h` 文件里面，内容如下：

**示例代码 75.1.3.2 iio_chan_type**

```c
enum iio_chan_type {
	IIO_VOLTAGE,		/* 电压类型 */
	IIO_CURRENT,		/* 电流类型 */
	IIO_POWER,			/* 功率类型 */
	IIO_ACCEL,			/* 加速度类型 */
	IIO_ANGL_VEL,		/* 角度类型(陀螺仪) */
	IIO_MAGN,			/* 电磁类型(磁力计) */
	IIO_LIGHT,			/* 灯光类型 */
	IIO_INTENSITY,		/* 强度类型(光强传感器) */
	IIO_PROXIMITY,		/* 接近类型(接近传感器) */
	IIO_TEMP,			/* 温度类型 */
	IIO_INCLI,			/* 倾角类型(倾角测量传感器) */
	IIO_ROT,			/* 旋转角度类型 */
	IIO_ANGL,			/* 转动角度类型(电机旋转角度测量传感器) */
	IIO_TIMESTAMP,		/* 时间戳类型 */
	IIO_CAPACITANCE,	/* 电容类型 */
	IIO_ALTVOLTAGE,		/* 频率类型 */
	IIO_CCT,			/* 笔者暂时未知的类型 */
	IIO_PRESSURE,		/* 压力类型 */
	IIO_HUMIDITYRELATIVE,/* 湿度类型 */
	IIO_ACTIVITY,		/* 活动类型(计步传感器) */
	IIO_STEPS,			/* 步数类型 */
	IIO_ENERGY,			/* 能量类型(卡路里) */
	IIO_DISTANCE,		/* 距离类型 */
	IIO_VELOCITY,		/* 速度类型 */
};
```

从示例代码 75.1.3.2 可以看出，目前 Linux 内核支持的传感器类型非常丰富，而且支持类型也会不断的增加。如果是 ADC，那就是 IIO_VOLTAGE 类型。如果是 ICM20608 这样的多轴传感器，那么就是复合类型了，陀螺仪部分是 IIO_ANGL_VEL 类型，加速度计部分是 IIO_ACCEL 类型，温度部分就是 IIO_TEMP。

继续来看示例代码 75.1.3.1 中的 iio_chan_spec 结构体，第 225 行，当成员变量 indexed 为 1 时候，channel 为通道索引。 第 226 行，当成员变量 modified 为 1 的时候，channel2 为通道修饰符。Linux 内核给出了可用的通道修饰符，定义在 `include/uapi/linux/iio/types.h` 文件里面，内容如下(有省略)：

**示例代码 75.1.3.3 iio_modifier**

```c
enum iio_modifier {
	IIO_NO_MOD,
	IIO_MOD_X,	/* X 轴 */
	IIO_MOD_Y,	/* Y 轴 */
	IIO_MOD_Z,	/* Z 轴 */
......
};
```

比如 ICM20608 的加速度计部分，类型设置为 IIO_ACCEL， X、Y、Z 这三个轴就用 channel2 的通道修饰符来区分。IIO_MOD_X、IIO_MOD_Y、IIO_MOD_Z 就分别对应 X、Y、Z 这三个轴。通道修饰符主要是影响 sysfs 下的通道文件名字，后面我们会讲解 sysfs 下通道文件名字组成形式。

继续回到示例代码 75.1.3.1，第 227 行的 address 成员变量用户可以自定义，但是一般会设置为此通道对应的芯片数据寄存器地址。比如 ICM20608 的加速度计 X 轴这个通道，它的数据首地址就是 0X3B。address 也可以用作其他功能，自行选择，也可以不使用 address，一切以实际情况为准。

第 228 行，当使用触发缓冲区的时候，scan_index 是扫描索引。 第 229~236，scan_type 是一个结构体，描述了扫描数据在缓冲区中的存储格式。我们依次来看一下 scan_type 各个成员变量的涵义：
- scan_type.sign：如果为‘u’表示数据为无符号类型，为‘s’的话为有符号类型。
- scan_type.realbits：数据真实的有效位数，比如很多传感器说的 10 位 ADC，其真实有效数据就是 10 位。
- scan_type.storagebits：存储位数，有效位数+填充位。比如有些传感器 ADC 是 12 位的，那么我们存储的话肯定要用到 2 个字节，也就是 16 位，这 16 位就是存储位数。
- scan_type.shift:右移位数，也就是存储位数和有效位数不一致的时候，需要右移的位数，这个参数不总是需要，一切以实际芯片的数据手册位数。
- scan_type.repeat：实际或存储位的重复数量。
- scan_type.endianness:数据的大小端模式，可设置为 IIO_CPU、IIO_BE(大端)或 IIO_LE(小端)。

第 237 行，info_mask_separate 标记某些属性专属于此通道，`include/linux/iio/types.h` 文件中的 iio_chan_info_enum 枚举类型描述了可选的属性值，如下所示：

**示例代码 75.1.3.4 iio_chan_info_enum**

```c
enum iio_chan_info_enum {
	IIO_CHAN_INFO_RAW = 0,
	IIO_CHAN_INFO_PROCESSED,
	IIO_CHAN_INFO_SCALE,
	IIO_CHAN_INFO_OFFSET,
......
	IIO_CHAN_INFO_DEBOUNCE_TIME,
};
```

比如 ICM20608 加速度计的 X、Y、Z 这三个轴，在 sysfs 下这三个轴肯定是对应三个不同的文件，我们通过读取这三个文件就能得到每个轴的原始数据。IIO_CHAN_INFO_RAW 这个属性表示原始数据，当我们在配置 X、Y、Z 这三个通道的时候，在 info_mask_separate 中使能 IIO_CHAN_INFO_RAW 这个属性，那么就表示在 sysfs 下生成三个不同的文件分别对应 X、Y、Z轴，这三个轴的 IIO_CHAN_INFO_RAW 属性是相互独立的。

第 238 行，info_mask_shared_by_type 标记导出的信息由相同类型的通道共享。也就是 iio_chan_spec.type 成员变量相同的通道。比如 ICM20608 加速度计的 X、Y、Z 轴他们的 type 都是 IIO_ACCEL，也就是类型相同。而这三个轴的分辨率(量程)是一样的，那么在配置这三个通道的时候就可以在 info_mask_shared_by_type 中使能 IIO_CHAN_INFO_SCALE 这个属性，表示这三个通道的分辨率是共用的，这样在 sysfs 下就会只生成一个描述分辨率的文件，这三个通道都可以使用这一个分辨率文件。

第 239 行，info_mask_shared_by_dir 标记某些导出的信息由相同方向的通道共享。 第 240 行，info_mask_shared_by_all 表示某些信息所有的通道共享，无论这些通道的类型、方向如何，全部共享。 第 246 行，modified 为 1 的时候，channel2 为通道修饰符。 第 247 行，indexed 为 1 的时候，channel 为通道索引。 第 248 行，output 表示为输出通道。 第 249 行，differential 表示为差分通道。
## 75.2 IIO 驱动框架创建

前面我们已经对 IIO 设备、IIO 通道进行了详细的讲解，本节我们就来学习如何搭建 IIO 驱动框架。在上一小节分析 IIO 子系统的时候大家应该看出了，IIO 框架主要用于 ADC 类的传感器，比如陀螺仪、加速度计、磁力计、光强度计等，这些传感器基本都是 IIC 或者 SPI 接口的。因此 IIO 驱动的基础框架就是 IIC 或者 SPI，我们可以在 IIC 或 SPI 驱动里面在加上上一章讲解的 regmap。当然了，有些 SOC 内部的 ADC 也会使用 IIO 框架，那么这个时候驱动的基础框架就是 platfrom。

### 75.2.1 基础驱动框架建立

我们以 SPI 接口为例，首先是 SPI 驱动框架，如下所示：

**示例代码 75.2.1.1 SPI 驱动框架**

```c
/*
 * @description : spi 驱动的 probe 函数，当驱动与
 * 设备匹配以后此函数就会执行
 * @param - spi : spi 设备
 * @return : 0,成功；其他值，失败
 */
static int xxx_probe(struct spi_device *spi)
{
	return 0;
}

/*
 * @description	: spi 驱动的 remove 函数，移除 spi 驱动的时候此函数会执行
 * @param - spi : spi 设备
 * @return : 0，成功;其他负值,失败
 */
static int xxx_remove(struct spi_device *spi)
{
	return 0;
}

/* 传统匹配方式 ID 列表 */
static const struct spi_device_id xxx_id[] = {
	{"alientek,xxx", 0},
	{}
};

/* 设备树匹配列表 */
static const struct of_device_id xxx_of_match[] = {
	{ .compatible = "alientek,xxx" },
	{ /* Sentinel */ }
};

/* SPI 驱动结构体 */
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

/*
 * @description : 驱动入口函数
 * @param	: 无
 * @return	: 无
 */
static int __init xxx_init(void)
{
	return spi_register_driver(&xxx_driver);
}

/*
 * @description : 驱动出口函数
 * @param	: 无
 * @return : 无
 */
static void __exit xxx_exit(void)
{
	spi_unregister_driver(&xxx_driver);
}

module_init(xxx_init);
module_exit(xxx_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("ALIENTEK");
```

示例代码 75.2.1.1 就是标准的 SPI 驱动框架，如果所使用的传感器是 IIC 接口的，那么就是 IIC 驱动框架，没啥好说的。

### 75.2.2 IIO 设备申请与初始化

IIO 设备的申请、初始化以及注册在 probe 函数中完成，在注销驱动的时候还需要在 remove 函数中注销掉 IIO 设备、释放掉申请的一些内存。添加完 IIO 框架以后的 probe 和 remove 函数如下所示：

**示例代码 75.2.2.1 添加 IIO 框架**

```c
/* 自定义设备结构体 */
struct xxx_dev {
	struct spi_device *spi;		/* spi 设备 */
	struct regmap *regmap; 		/* regmap */
	struct regmap_config regmap_config;
	struct mutex lock;
};

/*
 * 通道数组
 */
static const struct iio_chan_spec xxx_channels[] = {

};

/*
 * @description : 读函数，当读取 sysfs 中的文件的时候最终此函数会执行，
 * ：此函数里面会从传感器里面读取各种数据，然后上传给应用。
 * @param - indio_dev : IIO 设备
 * @param - chan	: 通道
 * @param - val : 读取的值，如果是小数值的话，val 是整数部分。
 * @param - val2 : 读取的值，如果是小数值的话，val2 是小数部分。
 * @param - mask : 掩码。
 * @return : 0，成功；其他值，错误
 */
static int xxx_read_raw(struct iio_dev *indio_dev,
	struct iio_chan_spec const *chan,
	int *val, int *val2, long mask)
{
	return 0;
}

/*
 * @description	: 写函数，当向 sysfs 中的文件写数据的时候最终此函数
 * ：会执行，一般在此函数里面设置传感器，比如量程等。
 * @param - indio_dev : IIO 设备
 * @param - chan : 通道
 * @param - val : 应用程序写入值，如果是小数的话，val 是整数部分。
 * @param - val2 : 应用程序写入值，如果是小数的话，val2 是小数部分。
 * @return	: 0，成功；其他值，错误
 */
static int xxx_write_raw(struct iio_dev *indio_dev,
	struct iio_chan_spec const *chan,
	int val, int val2, long mask)
{
	return 0;
}

/*
 * @description	: 用户空间写数据格式，比如我们在用户空间操作 sysfs 来设
 * ：置传感器的分辨率，如果分辨率带小数，那么这个小数传递到
 * : 内核空间应该扩大多少倍，此函数就是用来设置这个的。
 * @param - indio_dev : iio_dev
 * @param - chan : 通道
 * @param - mask : 掩码
 * @return : 0，成功；其他值，错误
 */
static int xxx_write_raw_get_fmt(struct iio_dev *indio_dev,
	struct iio_chan_spec const *chan, long mask)
{
	return 0;
}

/*
 * iio_info 结构体变量
 */
static const struct iio_info xxx_info = {
	.read_raw = xxx_read_raw,
	.write_raw = xxx_write_raw,
	.write_raw_get_fmt = &xxx_write_raw_get_fmt,
};

/*
 * @description : spi 驱动的 probe 函数，当驱动与
 * 设备匹配以后此函数就会执行
 * @param - spi : spi 设备
 *
 */
static int xxx_probe(struct spi_device *spi)
{
	int ret;
	struct xxx_dev *data;
	struct iio_dev *indio_dev;

	/* 1、申请 iio_dev 内存 */
	indio_dev = devm_iio_device_alloc(&spi->dev, sizeof(*data));
	if (!indio_dev)
		return -ENOMEM;

	/* 2、获取 xxx_dev 结构体地址 */
	data = iio_priv(indio_dev);
	data->spi = spi;
	spi_set_drvdata(spi, indio_dev);
	mutex_init(&data->lock);

	/* 3、初始化 iio_dev 成员变量 */
	indio_dev->dev.parent = &spi->dev;
	indio_dev->info = &xxx_info;
	indio_dev->name = "xxx";
	indio_dev->modes = INDIO_DIRECT_MODE; /* 直接模式 */
	indio_dev->channels = xxx_channels;
	indio_dev->num_channels = ARRAY_SIZE(xxx_channels);

	iio_device_register(indio_dev);

	/* 4、regmap 相关设置 */

	/* 5、SPI 相关设置*/

	/* 6、芯片初始化 */

	return 0;

}

/*
 * @description : spi 驱动的 remove 函数，移除 spi 驱动的时候此函数会执行
 * @param - spi : spi 设备
 * @return : 0，成功;其他负值,失败
 */
static int xxx_remove(struct spi_device *spi)
{
	struct iio_dev *indio_dev = spi_get_drvdata(spi);
	struct xxx_dev *data;

	data = iio_priv(indio_dev); ;

	/* 1、其他资源的注销以及释放 */

	/* 2、注销 IIO */
	iio_device_unregister(indio_dev);

	return 0;
}
```

第 27 行，用户自定义的设备结构体。 第 12 行，IIO 通道数组。 第 1671 行， 这部分为 iio_info，当应用程序读取相应的驱动文件的时候，xxx_read_raw函数就会执行，我们在此函数中会读取传感器数据，然后返回给应用层。当应用层向相应的驱动写数据的时候，xxx_write_raw 函数就会执行。因此 xxx_read_raw 和 xxx_write_raw 这两个函数是非常重要的！需要我们根据具体的传感器来编写，这两个函数是编写 IIO 驱动的核心。

第 79114 行，xxx_probe 函数，此函数的核心就是分配并初始化 iio_dev，最后向内核注册iio_dev。第 86 行调用 devm_iio_device_alloc 函数分配 iio_dev 内存，这里连用户自定义的设备结构体变量内存一起申请了。第 91 行调用 iio_priv 函数从 iio_dev 中提取出私有数据，这个私有数据就是设备结构体变量。第 97102 行初始化 iio_dev，重点是第 98 行设置 iio_dev 的 info成员变量。第 101 行设置 iio_dev 的通道。初始化完成以后就要调用 iio_device_register 函数向内核注册 iio_dev。整个过程就是：申请 iio_dev、初始化、注册，和我们前面讲解的其他驱动框架步骤一样。

第 121~134 行，xxx_remove 函数里面需要做的就是释放 xxx_probe 函数申请到的 IIO 相关资源，比如第 131 行，使用 iio_device_unregister 注销掉前面注册的 iio_dev。由于前面我们使用devm_iio_device_alloc 函数申请的 iio_dev，因此不需要在 remove 函数中手动释放 iio_dev。

IIO 框架示例就讲解到这里，剩下的就是根据所使用的具体传感器，在 IIO 驱动框架里面添加相关的处理，接下来我们就以正点原子 I.MX6ULL 开发板上的 ICM20608 为例，进行 IIO 驱动实战。

## 75.3 实验程序编写

接下来我们就直接使用 IIO 驱动框架编写 ICM20608 驱动，ICM20608 驱动核心就是 SPI，上一章我们也讲解了如何在 SPI 总线上使用 regmap。因此本章 ICM20608 驱动底层没啥好讲解的，重点是如何套上 IIO 驱动框架，因此关于 ICM20608 芯片内部寄存器、SPI 驱动、regmap 等不会做讲解，有什么不懂的可以看前面相应的实验章节。

本实验对应的例程路径为：开发板光盘→01、程序源码→02、Linux 驱动例程→27_iio→spi

### 75.3.1 使能内核 IIO 相关配置

Linux 内核默认使能了 IIO 子系统，但是有一些 IIO 模块没有选择上，这样会导致我们编译驱动的时候会提示某些 API 函数不存在，需要使能的项目如下：
```
-> Device Drivers
	-> Industrial I/O support (IIO [=y])
		-> [*]Enable buffer support within IIO			//选中
		-> <*>Industrial I/O buffering based on kfifo	//选中
```
### 75.3.2 ICM20608 的 IIO 驱动框架搭建

#### 1、驱动框架搭建

设备树不需要做任何修改！

首先我们基于上一章实验，搭建出 ICM20608 的 IIO 驱动框架，IIO 框架部分我们已经在示例代码 75.2.2.1 中进行了详细的讲解。新建名为 icm20608.c 的驱动文件，搭建好的 ICM20608 IIO 驱动框架内容如下所示：

**示例代码 75.3.2.1 ICM20608 驱动框架**

```c
/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名	: icm20608.c
作者 	: 正点原子 Linux 团队
版本 	: V1.0
描述 	: ICM20608 SPI 驱动程序
其他 	: 无
论坛 	: www.openedv.com
日志 	: 初版 V1.0 2021/03/22 正点原子 Linux 团队创建
V1.1 2021/08/10	使用 regmap 读写 SPI 外设内部寄存器。
V1.2 2021/08/13	使用 IIO 框架，参考 bma220_spi.c
***************************************************************/
#include <linux/spi/spi.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/errno.h>
#include <linux/platform_device.h>
#include "icm20608reg.h"
#include <linux/gpio.h>
#include <linux/device.h>
#include <asm/uaccess.h>
#include <linux/cdev.h>
#include <linux/regmap.h>
#include <linux/iio/iio.h>
#include <linux/iio/sysfs.h>
#include <linux/iio/buffer.h>
#include <linux/iio/trigger.h>
#include <linux/iio/triggered_buffer.h>
#include <linux/iio/trigger_consumer.h>
#include <linux/unaligned/be_byteshift.h>

#define ICM20608_NAME "icm20608"

#define ICM20608_CHAN(_type, _channel2, _index) \
{ \
	.type = _type, \
	.modified = 1, \
	.channel2 = _channel2, \
	.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE), \
	.info_mask_separate = BIT(IIO_CHAN_INFO_RAW) | \
	BIT(IIO_CHAN_INFO_CALIBBIAS), \
	.scan_index = _index, \
	.scan_type = { \
		.sign = 's', \
		.realbits = 16, \
		.storagebits = 16, \
		.shift = 0, \
		.endianness = IIO_BE, \
	}, \
}

/*
 * ICM20608 的扫描元素，3 轴加速度计、
 * 3 轴陀螺仪、1 路温度传感器，1 路时间戳
 */
enum inv_icm20608_scan {
	INV_ICM20608_SCAN_ACCL_X,
	INV_ICM20608_SCAN_ACCL_Y,
	INV_ICM20608_SCAN_ACCL_Z,
	INV_ICM20608_SCAN_TEMP,
	INV_ICM20608_SCAN_GYRO_X,
	INV_ICM20608_SCAN_GYRO_Y,
	INV_ICM20608_SCAN_GYRO_Z,
	INV_ICM20608_SCAN_TIMESTAMP,
};

struct icm20608_dev {
	struct spi_device *spi;	/* spi 设备 */
	struct regmap *regmap; 		/* regmap */
	struct regmap_config regmap_config;
	struct mutex lock;
};

/*
 * icm20608 通道，1 路温度通道，3 路陀螺仪，3 路加速度计
 */
static const struct iio_chan_spec icm20608_channels[] = {
	/* 温度通道 */
	{
		.type = IIO_TEMP,
		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW)
		| BIT(IIO_CHAN_INFO_OFFSET)
		| BIT(IIO_CHAN_INFO_SCALE),
		.scan_index = INV_ICM20608_SCAN_TEMP,
		.scan_type = {
			.sign = 's',
			.realbits = 16,
			.storagebits = 16,
			.shift = 0,
			.endianness = IIO_BE,
		},
	},

	ICM20608_CHAN(IIO_ANGL_VEL, IIO_MOD_X,INV_ICM20608_SCAN_GYRO_X),
	ICM20608_CHAN(IIO_ANGL_VEL, IIO_MOD_Y,INV_ICM20608_SCAN_GYRO_Y),
	ICM20608_CHAN(IIO_ANGL_VEL, IIO_MOD_Z,INV_ICM20608_SCAN_GYRO_Z),

	ICM20608_CHAN(IIO_ACCEL, IIO_MOD_Y, INV_ICM20608_SCAN_ACCL_Y),
	ICM20608_CHAN(IIO_ACCEL, IIO_MOD_X, INV_ICM20608_SCAN_ACCL_X),
	ICM20608_CHAN(IIO_ACCEL, IIO_MOD_Z, INV_ICM20608_SCAN_ACCL_Z),
};

/*
 * @description : 读取 icm20608 指定寄存器值，读取一个寄存器
 * @param – dev	: icm20608 设备
 * @param – reg	: 要读取的寄存器
 * @return		: 读取到的寄存器值
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
 * @return		: 无
 */
static void icm20608_write_onereg(struct icm20608_dev *dev, u8 reg, u8 value)
{
	regmap_write(dev->regmap, reg, value);
}

/*
 * @description : ICM20608 内部寄存器初始化函数
 * @param – spi : 要操作的设备
 * @return : 无
 */
void icm20608_reginit(struct icm20608_dev *dev)
{
	u8 value = 0;

	icm20608_write_onereg(dev, ICM20_PWR_MGMT_1, 0x80);
	mdelay(50);
	icm20608_write_onereg(dev, ICM20_PWR_MGMT_1, 0x01);
	mdelay(50);

	value = icm20608_read_onereg(dev, ICM20_WHO_AM_I);
	printk("ICM20608 ID = %#X\r\n", value);

	icm20608_write_onereg(dev, ICM20_SMPLRT_DIV, 0x00);
	icm20608_write_onereg(dev, ICM20_GYRO_CONFIG, 0x18);
	icm20608_write_onereg(dev, ICM20_ACCEL_CONFIG, 0x18);
	icm20608_write_onereg(dev, ICM20_CONFIG, 0x04);
	icm20608_write_onereg(dev, ICM20_ACCEL_CONFIG2, 0x04);
	icm20608_write_onereg(dev, ICM20_PWR_MGMT_2, 0x00);
	icm20608_write_onereg(dev, ICM20_LP_MODE_CFG, 0x00);
	icm20608_write_onereg(dev, ICM20_INT_ENABLE, 0x01);
}

/*
 * @description : 读函数，当读取 sysfs 中的文件的时候最终此函数会执
 * ：行，此函数里面会从传感器里面读取各种数据，然后上传给应用。
 * @param - indio_dev	: iio_dev
 * @param - chan 		: 通道
 * @param - val 		: 读取的值，如果是小数值的话，val 是整数部分。
 * @param - val2 		: 读取的值，如果是小数值的话，val2 是小数部分。
 * @param - mask 		: 掩码。
 * @return : 0，成功；其他值，错误
 */
static int icm20608_read_raw(struct iio_dev *indio_dev,
		struct iio_chan_spec const *chan,
		int *val, int *val2, long mask)
{
	printk("icm20608_read_raw\r\n");
	return 0;
}

/*
 * @descriptio : 写函数，当向 sysfs 中的文件写数据的时候最终此函数会
 * ：执行，一般在此函数里面设置传感器，比如量程等。
 * @param - indio_dev	: iio_dev
 * @param – chan 		: 通道
 * @param – val 		: 应用程序写入的值，如果是小数值的话，val 是整数部分。
 * @param - val2 		: 应用程序写入的值，如果是小数值的话，val2 是小数部分。
 * @return : 0，成功；其他值，错误
 */
static int icm20608_write_raw(struct iio_dev *indio_dev,
		struct iio_chan_spec const *chan,
		int val, int val2, long mask)
{
	printk("icm20608_write_raw\r\n");
	return 0;

}

/*
 * @description : 用户空间写数据格式，比如我们在用户空间操作 sysfs 来设置传
 * ：感器的分辨率，如果分辨率带小数，那么这个小数传递到内核空间
 * : 应该扩大多少倍，此函数就是用来设置这个的。
 * @param - indio_dev	: iio_dev
 * @param - chan 		: 通道
 * @param - mask 		: 掩码
 * @return : 0，成功；其他值，错误
 */
static int icm20608_write_raw_get_fmt(struct iio_dev *indio_dev,
		struct iio_chan_spec const *chan, long mask)
{
	printk("icm20608_write_raw_get_fmt\r\n");
	return 0;

}

/*
 * iio_info 结构体变量
 */
static const struct iio_info icm20608_info = {
	.read_raw = icm20608_read_raw,
	.write_raw = icm20608_write_raw,
	.write_raw_get_fmt = &icm20608_write_raw_get_fmt,
};

/*
 * @description : spi 驱动的 probe 函数，当驱动与
 * 设备匹配以后此函数就会执行
 * @param – spi : spi 设备
 * @return : 0,成功；其他值，失败
 */
static int icm20608_probe(struct spi_device *spi)
{
	int ret;
	struct icm20608_dev *dev;
	struct iio_dev *indio_dev;

	/* 1、申请 iio_dev 内存 */
	indio_dev = devm_iio_device_alloc(&spi->dev, sizeof(*dev));
	if (!indio_dev)
		return -ENOMEM;

	/* 2、获取 icm20608_dev 结构体地址 */
	dev = iio_priv(indio_dev);
	dev->spi = spi;
	spi_set_drvdata(spi, indio_dev);
	mutex_init(&dev->lock);

	/* 3、iio_dev 的其他成员变量 */
	indio_dev->dev.parent = &spi->dev;
	indio_dev->info = &icm20608_info;
	indio_dev->name = ICM20608_NAME;
	indio_dev->modes = INDIO_DIRECT_MODE;/* 直接模式，提供 sysfs 接口 */
	indio_dev->channels = icm20608_channels;
	indio_dev->num_channels = ARRAY_SIZE(icm20608_channels);

	/* 4、注册 iio_dev */
	ret = iio_device_register(indio_dev);
	if (ret < 0) {
		dev_err(&spi->dev, "iio_device_register failed\n");
		goto err_iio_register;
	}

	/* 5、初始化 regmap_config 设置 */
	dev->regmap_config.reg_bits = 8; /* 寄存器长度 8bit */
	dev->regmap_config.val_bits = 8; /* 值长度 8bit */
	dev->regmap_config.read_flag_mask = 0x80; /* 读掩码设置为 0X80 */

	/* 6、初始化 SPI 接口的 regmap */
	dev->regmap = regmap_init_spi(spi, &dev->regmap_config);
	if (IS_ERR(dev->regmap)) {
		ret = PTR_ERR(dev->regmap);
		goto err_regmap_init;
	}

	/* 7、初始化 spi_device */
	spi->mode = SPI_MODE_0; /*MODE0，CPOL=0，CPHA=0*/
	spi_setup(spi);

	/* 初始化 ICM20608 内部寄存器 */
	icm20608_reginit(dev);
	return 0;

err_regmap_init:
	iio_device_unregister(indio_dev);
err_iio_register:
	return ret;
}

/*
 * @description : spi 驱动的 remove 函数，移除 spi 驱动的时候此函数会执行
 * @param - spi : spi 设备
 * @return : 0，成功;其他负值,失败
 */
static int icm20608_remove(struct spi_device *spi)
{
	struct iio_dev *indio_dev = spi_get_drvdata(spi);
	struct icm20608_dev *dev;

	dev = iio_priv(indio_dev);

	/* 1、删除 regmap */
	regmap_exit(dev->regmap);

	/* 2、注销 IIO */
	iio_device_unregister(indio_dev);
	return 0;
}

/* 传统匹配方式 ID 列表 */
static const struct spi_device_id icm20608_id[] = {
	{"alientek,icm20608", 0},
	{}
};

/* 设备树匹配列表 */
static const struct of_device_id icm20608_of_match[] = {
	{ .compatible = "alientek,icm20608" },
	{ /* Sentinel */ }
};

/* SPI 驱动结构体 */
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

/*
 * @description : 驱动入口函数
 * @param : 无
 * @return : 无
 */
static int __init icm20608_init(void)
{
	return spi_register_driver(&icm20608_driver);
}

/*
 * @description : 驱动出口函数
 * @param : 无
 * @return : 无
 */
static void __exit icm20608_exit(void)
{
	spi_unregister_driver(&icm20608_driver);
}

module_init(icm20608_init);
module_exit(icm20608_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("ALIENTEK");
MODULE_INFO(intree, "Y");
```

示例代码 75.3.1.1 就是在上一章例程的基础上添加 IIO 驱动框架而来的，这里我们重点讲解一下 IIO 部分。
- 第 25~41 行，通道宏定义，用于陀螺仪和加速度计，第 28 行 modified 成员变量为 1，所以channel2就是通道修饰符，用来指定 X、Y、Z 轴。第 30 行设置相同类型的通道 IIO_CHAN_INFO_SCALE 属性相同，"scale"是比例的意思，在这里就是量程(分辨率)，因为 ICM20608 的陀螺仪和加速度计的量程是可以调整的，量程不同分辨率也就不同。陀螺仪或加速度计的三个轴也是一起设置的，因此对于陀螺仪或加速度计而言，X、Y、Z 这三个轴的量程是共享的。第 31 行，设置每个通道的 IIO_CHAN_INFO_RAW 和 IIO_CHAN_INFO_CALIBBIAS 这两个属性都是独立的，IIO_CHAN_INFO_RAW 表示 ICM20608 每个通道的原始值，这个肯定是每个通道独立的。IIO_CHAN_INFO_CALIBBIAS 是 ICM20608 每个通道的校准值，这个是 ICM20608 的特性，不是所有的传感器都有校准值，一切都要以实际所使用的传感器为准。第 34 行，设置扫描数据类型，也就是 ICM20608 原始数据类型，ICM20608 的陀螺仪和加速度计都是 16 位 ADC，因此这里是通用的：为有符号类型、实际位数 16bit，存储位数 16bit，大端模式(ICM20608数据寄存器为大端模式)。
- 第 47~56 行，自定义的扫描索引枚举类型 inv_icm20608_scan，包括陀螺仪、加速度计的 6 个通道，温度计的 1 个通道、以及 1 个 ICM20608 时间戳通道。
- 第 58~63 行，设备结构体，由于采用了 regmap 和 IIO 框架，因此 ICM20608 的设备结构体非常简单。
- 第 68~92 行，ICM20608 通道，这里定义了 7 个通道，分别是：1 个温度通道，3 个陀螺仪通道(X、Y、Z)，3 个加速度计通道(X、Y、Z)。温度通道有三个属性，IIO_CHAN_INFO_RAW为温度通道的原始值，IIO_CHAN_INFO_OFFSET是 ICM20608 温度 offset 值，这个要查阅数据手册。IIO_CHAN_INFO_SCALE是 ICM20608 的比例，也就是一个单位的原始值为多少℃，这个也要查阅 ICM20608 的数据手册。从这里可以看出，想要得到 ICM20608 的具体温度值，需要三个数据：原始值、offset 值、比例值，也就是应用程序需要能够从 IIO 驱动框架中的到这三个值，一般是应用程序读取相应的文件，所以这里就要有三个独立的文件分别表示原始值、offset 值、比例值，这就是三个属性的来源。剩下的陀螺仪和加速度计通道设置使用宏 ICM20608_CHAN 即可，IIO_MOD_X、IIO_MOD_Y 和 IIO_MOD_Z 分别是 X、Y、Z 这三个轴的修饰符。
- 第 204~208 行，这部分就是 iio_info，icm20608_read_raw 为读取函数，应用程序读取相应文件的时候此函数执行，icm20608_write_raw 为写函数，应用程序向相应的文件写数据的时候此函数执行。icm20608_write_raw_get_fmt 函数用来设置应用程序向驱动写入的数据格式，icm20608_info 就是具体的 iio_info 变量，初始化 iio_dev 的时候需要用到。
- 第 216 行，icm20608_probe 函数，一般在此函数里面申请 iio_dev、初始化并注册，初始化 regmap、初始化 ICM20608 等。第 216 行通过 devm_iio_device_alloc 函数申请 iio_dev 以及自定义设备结构体内存，本章节就是 icm20608_dev。第 234~239 行，初始化 iio_dev，第 242 行，调用 iio_device_register 函数想内核注册 iio_dev。

关于 ICM20608 的 IIO 驱动框架就讲解到这里，接下来就是测试此驱动框架。

#### 2、驱动框架测试

我们已经搭建好了 ICM20608 的 IIO 驱动框架，通道也已经设置好了，虽然还不能直接读取到 ICM20608 的原始数据，但是我们可以通过驱动框架来窥探 IIO 在用户空间的存在方式。

编译驱动，得到 icm20608.ko 驱动文件。执行如下命令加载 icm20608.ko 这个驱动模块。

```bash
depmod					#第一次加载驱动的时候需要运行此命令
modprobe icm20608.ko 	#加载驱动模块
```

我们在 icm20608_probe 函数里面设置了打印 ICM20608 ID 值，因此如果驱动加载成功，SPI 工作正常的话就会读取 ICM20608 的 ID 值并打印出来。

IIO 驱动框架提供了 sysfs 接口，因此加载成功以后我们可以在用户空间访问对应的 sysfs 目录项，进入目录 `/sys/bus/iio/devices/` 目录里面，此目录下都是 IIO 框架设备。

此时有两个 IIO 设备iio:device0，iio:device0是 I.MX6ULL 内部 ADC，iio:device1才是 ICM20608。大家进入到对应的设备目录就可以看出对应的 IIO 设备，我们进入iio:device1目录。

iio:device1对应 spi2.0 上的设备，也就是 ICM20608，此目录下有很多文件，比如 in_accel_scale、in_accel_x_calibias、in_accel_x_raw 等，这些就是我们设置的通道。in_accel_scale就是加速度计的比例，也就是分辨率(量程)，in_accel_x_calibias就是加速度计 X 轴的校准值，in_accel_x_raw就是加速度计的 X 轴原始值。我们在配置通道的时候，设置了类型相同的所有通道共用 SCALE，所以这里只有一个 in_accel_scale，而 X、Y、Z 轴的原始值和校准值每个轴都有一个文件，陀螺仪和温度计同理。

#### 3、通道文件命名方式

我们来看一下这些文件名字组成方式，以 in_accel_x_raw 为例，这是加速度计的 X 轴原始值，驱动代码中此通道的配置内容展开以后如下(演示代码)：

**示例代码 75.3.2.2 加速度计 X 轴通道配置演示代码**

```c
.type = IIO_ANGL_VEL,
.modified = 1,
.channel2 = IIO_MOD_X,
.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),
.info_mask_separate = BIT(IIO_CHAN_INFO_RAW) |
BIT(IIO_CHAN_INFO_CALIBBIAS),
.scan_index = INV_ICM20608_SCAN_GYRO_X,
.scan_type = {
	.sign = 's',
	.realbits = 16,
	.storagebits = 16,
	.shift = 0,
	.endianness = IIO_BE,
},
```

第 5 行设置了此通道有 IIO_CHAN_INFO_RAW 和 IIO_CHAN_INFO_CALIBBIAS 这两个专属属性，因此才会有 in_accel_x_raw 和 in_accel_x_calibias 这两个文件。

通道属性的命名模式为：`[direction]_[type]_[index]_[modifier]_[info_mask]`

**1. direction**：为属性对应的方向，iio_direction 结构体定义了方向。

**示例代码 75.3.2.3 iio_direction 结构体**

```c
static const char * const iio_direction[] = {
	[0] = "in",
	[1] = "out",
};
```

可以看出，就有两个方向：in 和 out。

**2. type**：也就是配置通道的时候 type 值，type 对应的字符可以参考 iio_chan_type_name_spec。

**示例代码 75.3.3.4 iio_chan_type_name_spec 结构体**

```c
static const char * const iio_chan_type_name_spec[] = {
	[IIO_VOLTAGE] = "voltage",
	[IIO_CURRENT] = "current",
	[IIO_POWER] = "power",
	[IIO_ACCEL] = "accel",
	[IIO_ANGL_VEL] = "anglvel",
	[IIO_MAGN] = "magn",
......
	[IIO_GRAVITY] = "gravity",
	[IIO_POSITIONRELATIVE] = "positionrelative",
	[IIO_PHASE] = "phase",
	[IIO_MASSCONCENTRATION] = "massconcentration",
};
```

所以，当通道的 type 设置为 IIO_ACCEL 的时候，对应的名字就是accel。

**3. index**：索引，如果配置通道的时候设置了 indexed=1，那么就会使用通道的 channel 成员变量来替代此部分命名。比如，有个 ADC 芯片支持 8 个通道，那么就可以使用 channel 来表示对应的通道，最终在用户空间呈现的每个通道文件名的 index 部分就是通道号。

**4. modifier**：当通道的 modified 成员变量为 1 的时候，channel2 就是修饰符，修饰符对应的字符串参考结构体 iio_modifier_names。

**示例代码 75.3.2.5 iio_modifier_names 结构体**

```c
static const char * const iio_modifier_names[] = {
	[IIO_MOD_X] = "x",
	[IIO_MOD_Y] = "y",
	[IIO_MOD_Z] = "z",
......
	[IIO_MOD_PM4] = "pm4",
	[IIO_MOD_PM10] = "pm10",
};
```

当通道的修饰符设置为 IIO_MOD_X 的时候，对应的名字就是x。

**5. info_mask**：属性掩码，也就是属性，不同属性对应的字符如下所示。

**示例代码 75.3.2.6 iio_chan_info_postfix 结构体**

```c
static const char * const iio_chan_info_postfix[] = {
	[IIO_CHAN_INFO_RAW] = "raw",
	[IIO_CHAN_INFO_PROCESSED] = "input",
	[IIO_CHAN_INFO_SCALE] = "scale",
	[IIO_CHAN_INFO_OFFSET] = "offset",
	[IIO_CHAN_INFO_CALIBSCALE] = "calibscale",
	[IIO_CHAN_INFO_CALIBBIAS] = "calibbias",
......
	[IIO_CHAN_INFO_DEBOUNCE_TIME] = "debounce_time",
	[IIO_CHAN_INFO_CALIBEMISSIVITY] = "calibemissivity",
	[IIO_CHAN_INFO_OVERSAMPLING_RATIO] = "oversampling_ratio",
};
```

可以看出，IIO_CHAN_INFO_RAW 属性对应的就是raw，IIO_CHAN_INFO_SCALE属性对应的是scale。

其他文件的命名组成方式类似，大家自行分析即可。

#### 4、文件读测试

我们可以测试一下对这些文件进行读写操作，看看有什么效果，比如 in_accel_x_raw 文件是 ICM20608 的加速度计 X 轴原始值通道，使用 cat 命令查看此文件内容，当用户空间读取相应文件的时候，iio_info 下的 read 函数会被调用，因此关于传感器数据读取的操作都是在此函数中完成的。由于驱动还不完善，大家就先不要测试向指定文件写操作，否则可能会报错，后面再测试写操作。大家也可以读取一下其他文件，比如陀螺仪的 X 轴文件 in_anglvel_x_raw，结果最终都是调用的 icm20608_read_raw 这个函数。

接下来的重点就是完善驱动中的 icm20608_read_raw 函数，实现传感器数据的读取操作。
### 75.3.3 完善 icm20608_read_raw 函数

应用程序所有的读取操作，最终都会汇总到 iio_info 的 read 函数，这里就是 icm20608_read_raw 函数。由于所有的读取操作都会触发 icm20608_read_raw 函数，比如加速度计、陀螺仪、温度计等，因此我们需要做区分。配置通道的时候设置了 type 值，就可以使用 type 值来区分是陀螺仪、加速度计还是温度计，修改后的 icm20608_read_raw 函数如下：

**示例代码 75.3.3.1 修改后的 icm20608_read_raw**

```c
static int icm20608_read_raw(struct iio_dev *indio_dev,
		struct iio_chan_spec const *chan,
		int *val, int *val2, long mask)

	struct icm20608_dev *dev = iio_priv(indio_dev);
	int ret = 0;
	unsigned char regdata = 0;
	printk("icm20608_read_raw\r\n");

	switch (mask) {
	case IIO_CHAN_INFO_RAW: /* 读取加速度计、陀螺仪、温度传感器原始值 */
		printk("read raw data\r\n");
		return ret;
	case IIO_CHAN_INFO_SCALE:
		switch (chan->type) {
		case IIO_ANGL_VEL:
			printk("read gyro sacle\r\n");
			return ret;
		case IIO_ACCEL:
			printk("read accel sacle\r\n");
			return 0;
		case IIO_TEMP:
			printk("read temp sacle\r\n");
			return ret;
		default:
			return -EINVAL;
		}
	case IIO_CHAN_INFO_OFFSET: /* ICM20608 温度传感器 offset 值 */
		switch (chan->type) {
		case IIO_TEMP:
			printk("read temp offset\r\n");
			return ret;
		default:
			return -EINVAL;
		}
		return ret;
	case IIO_CHAN_INFO_CALIBBIAS: /* ICM20608 加速度计和陀螺仪校准值 */
		switch (chan->type) {
		case IIO_ANGL_VEL: /* 陀螺仪的校准值 */
			printk("read gyro calibbias\r\n");
			return ret;
		case IIO_ACCEL: /* 加速度计的校准值 */
			printk("read accel calibbias\r\n");
			return ret;
		default:
			return -EINVAL;
		}
	default:
		return ret -EINVAL;
}
```

示例代码 75.3.3.1 给出了很多分支，这些分支对应不同的文件读操作。比如我们读取 `in_accel_scale` 这个文件，这个是加速度计的比例文件，对应的就是加速度计分支，输出了read accel scale这行字符串。我们就可以在这个分支里面读取 ICM20608 的加速度计比例值，也就是量程。其他文件也一样，最终都会对应到相应的分支里面，我们只需要在相应的分支里面做具体的操作就行了。剩下的操作就是读取 ICM20608 的内部寄存器数据，最终修改好的 icm20608_read_raw 函数和其他一些配套函数如下：

**示例代码 75.3.3.2 完善的 icm20608_read_raw 函数**

```c
#define ICM20608_TEMP_OFFSET 0
#define ICM20608_TEMP_SCALE 326800000

......

/* icm20608 陀螺仪分辨率，对应 250、500、1000、2000，计算方法：
 * 以正负 250 度量程为例，500/2^16=0.007629，扩大 1000000 倍，就是 7629
 */
static const int gyro_scale_icm20608[] = {7629, 15258,
30517, 61035};

/* icm20608 加速度计分辨率，对应 2、4、8、16 计算方法：
 * 以正负 2g 量程为例，4/2^16=0.000061035，扩大 1000000000 倍，就是 61035
 */
static const int accel_scale_icm20608[] = {61035, 122070,
244140, 488281};

......

/*
 * @description : 读取 ICM20608 传感器数据，可以用于陀螺仪、加速度计、温度
 * @param – dev : icm20608 设备
 * @param - reg : 要读取的通道寄存器首地址。
 * @param – anix : 需要读取的通道，比如 X，Y，Z。
 * @param - *val : 保存读取到的值。
 * @return : 0，成功；其他值，错误
 */
static int icm20608_sensor_show(struct icm20608_dev *dev, int reg,
int axis, int *val)
{
	int ind, result;
	__be16 d;

	ind = (axis - IIO_MOD_X) * 2;
	result = regmap_bulk_read(dev->regmap, reg + ind, (u8 *)&d, 2);
	if (result)
		return -EINVAL;
	*val = (short)be16_to_cpup(&d);

	return IIO_VAL_INT;
}

/*
 * @description	: 读取 ICM20608 陀螺仪、加速度计、温度通道值
 * @param - indio_dev : iio 设备
 * @param - chan : 通道。
 * @param - val : 保存读取到的通道值。
 * @return : 0，成功；其他值，错误
 */
static int icm20608_read_channel_data(struct iio_dev *indio_dev,
struct iio_chan_spec const *chan,
int *val)
{
	struct icm20608_dev *dev = iio_priv(indio_dev);
	int ret = 0;

	switch (chan->type) {
	case IIO_ANGL_VEL: /* 读取陀螺仪数据 */
		ret = icm20608_sensor_show(dev, ICM20_GYRO_XOUT_H,
		chan->channel2, val); /* channel2 为 X、Y、Z 轴 */
		break;
	case IIO_ACCEL: /* 读取加速度计数据 */
		ret = icm20608_sensor_show(dev, ICM20_ACCEL_XOUT_H,
		chan->channel2, val); /* channel2 为 X、Y、Z 轴 */
		break;
	case IIO_TEMP: /* 读取温度 */
		ret = icm20608_sensor_show(dev, ICM20_TEMP_OUT_H,
		IIO_MOD_X, val);
		break;
	default:
		ret = -EINVAL;
		break;
	}
	return ret;
}

/*
 * @description : 读函数，当读取 sysfs 中的文件的时候最终此函数会执行，此函
 * ：数里面会从传感器里面读取各种数据，然后上传给应用。
 * @param - indio_dev	: iio_dev
 * @param - chan 		: 通道
 * @param - val 		: 读取的值，如果是小数值的话，val 是整数部分。
 * @param - val2 		: 读取的值，如果是小数值的话，val2 是小数部分。
 * @param - mask 		: 掩码。
 * @return : 0，成功；其他值，错误
 */
static int icm20608_read_raw(struct iio_dev *indio_dev,
struct iio_chan_spec const *chan,
int *val, int *val2, long mask)
{
	struct icm20608_dev *dev = iio_priv(indio_dev);
	int ret = 0;
	unsigned char regdata = 0;

	switch (mask) {
	case IIO_CHAN_INFO_RAW: /* 读取 ICM20608 加速度计、陀螺仪、
	温度传感器原始值 */
		iio_device_claim_direct_mode(indio_dev);/* 保持 direct 模式 */
		mutex_lock(&dev->lock); /* 上锁 */
		ret = icm20608_read_channel_data(indio_dev, chan, val);
		mutex_unlock(&dev->lock); /* 释放锁 */
		iio_device_release_direct_mode(indio_dev);
		return ret;
	case IIO_CHAN_INFO_SCALE:
		switch (chan->type) {
		case IIO_ANGL_VEL:
			mutex_lock(&dev->lock);
			regdata = (icm20608_read_onereg(dev, ICM20_GYRO_CONFIG)
			& 0X18) >> 3;
			*val = 0;
			*val2 = gyro_scale_icm20608[regdata];
			mutex_unlock(&dev->lock);
			return IIO_VAL_INT_PLUS_MICRO; /* 值为 val+val2/1000000 */
		case IIO_ACCEL:
			mutex_lock(&dev->lock);
			regdata = (icm20608_read_onereg(dev, ICM20_ACCEL_CONFIG)
			& 0X18) >> 3;
			*val = 0;
			*val2 = accel_scale_icm20608[regdata];;
			mutex_unlock(&dev->lock);
			return IIO_VAL_INT_PLUS_NANO;/* 值为 val+val2/1000000000 */
		case IIO_TEMP:
			*val = ICM20608_TEMP_SCALE/ 1000000;
			*val2 = ICM20608_TEMP_SCALE % 1000000;
			return IIO_VAL_INT_PLUS_MICRO; /* 值为 val+val2/1000000 */
		default:
			return -EINVAL;
		}
		return ret;
	case IIO_CHAN_INFO_OFFSET: /* ICM20608 温度传感器 offset 值 */
		switch (chan->type) {
		case IIO_TEMP:
			*val = ICM20608_TEMP_OFFSET;
			return IIO_VAL_INT;
		default:
			return -EINVAL;
		}
		return ret;
	case IIO_CHAN_INFO_CALIBBIAS: /* ICM20608 加速度计和陀螺仪校准值 */
		switch (chan->type) {
		case IIO_ANGL_VEL:		/* 陀螺仪的校准值*/
			mutex_lock(&dev->lock);
			ret = icm20608_sensor_show(dev, ICM20_XG_OFFS_USRH,
			chan->channel2, val);
			mutex_unlock(&dev->lock);
			return ret;
		case IIO_ACCEL:		/* 加速度计的校准值*/
			mutex_lock(&dev->lock);
			ret = icm20608_sensor_show(dev, ICM20_XA_OFFSET_H,
			chan->channel2, val);
			mutex_unlock(&dev->lock);
			return ret;
		default:
			return -EINVAL;
		}

	default:
		return ret -EINVAL;
	}
}
```

- 第 68 行，gyro_scale_icm20608 是 ICM20608 的陀螺仪比例，也就是陀螺仪不同量程对应的分辨率。ICM20608 陀螺仪为 16bit，可选的量程有±250、±500、±1000 和±2000°/s。以±250 这个量程为例，每个数值对应的度数就是 500/2^10≈0.007629°/s。同理，±500 量程对应的度数为 0.015258，±1000 量程对应的度数为 0.030517，±2000 量程为 0.061035。假设现在设置量程为±2000，读取到的原始值为 12540，那么对应的度数就是 12540*0.061035≈765.37°/s。注意，这里扩大了 1000000 倍。
- 第 73 行，accel_scale_icm20608 是 ICM20608 的加速度计比例，也就是加速度计不同量程对应的分辨率，计算方法和陀螺仪一样。这里扩大了 1000000000 倍。
- 第 164~177 行，icm20608_sensor_show 函数用于读取加速度计、陀螺仪、温度的原始数据。
- 第 186~208 行，icm20608_read_channel_data 函数用于读取指定通道的数据，根据 type 类型的不同，给 icm20608_sensor_show 函数传递不同的参数，读取陀螺仪、加速度计或温度数据。
- 第 220~288 行，修改完善的 icm20608_read_raw 函数，我们就以读取陀螺仪、加速度计或温度原始数据为例，当应用程序读取这几个通道的原始数据的时候第 229 行的分支会执行。第 232 行直接调用 icm20608_read_channel_data 函数来根据通道的 type 值来读取 ICM20608 相应的寄存器。其他分支内容大同小异，这些就是 ICM20608 寄存器读取内容了，这里就不在多讲。

修改完成以后重新编译驱动文件，然后加载新的驱动，测试是否可以正常读取到相应的内容。我们读取一下 `in_accel_scale` 这个文件，这是加速度计的分辨率，我们默认设置了加速度计量程为±16g，因此分辨率为 0.000488281。

此时 `in_accel_scale` 为 0.000488281，是正确的。这时候有朋友可能会疑问，在示例代码 75.3.3.2 中，我们设置加速度计±16g 的分辨率为 488281，也就是扩大了 1000000000 倍，为啥这里读出来的是 0.000488281 这个原始值？大家注意看示例代码 75.3.3.2 中第 245~251 行所在的分支，这部分是读取加速度计的分辨率，读取完成以后在第 251 行返回 `IIO_VAL_INT_PLUS_NANO` 这个值，这里就是告诉用户空间，小数部分(val2)扩大了 1000000000 倍，因此用户空间得到分辨率以后会除以 1000000000，得到真实的分辨率，488281/1000000000=0.000488281，所以中得到的值就是 0.000488281。

我们再读取一下 `in_accel_z_raw` 这个文件，这个文件是加速度计的 Z 轴原始值，静态情况下 Z 轴应该是 1g 的重力加速度计，我们可以读取 `in_accel_z_raw` 这个文件的值，然后在结合上面读取到的加速度计分辨率，计算一下对应的 Z 轴重力值，看看是不是 1g 左右。

### 75.3.4 完善 icm20608_write_raw 函数

最后我们完善 icm20608_write_raw 函数，用户空间向驱动写数据的时候 icm20608_write_raw 函数会执行。我们可以在用户空间设置陀螺仪、加速度计的量程、校准值等，这时候就需要向驱动写入数据。本章我们简单一点，就只实现设置陀螺仪和加速度计的量程，至于其他的设置项大家可以自行实现，修改后的 icm20608_write_raw 函数和其他相关内容如下：

**示例代码 75.3.4.1 完善的 icm20608_write_raw 函数**

```c
/*
 * @description : 设置 ICM20608 传感器，可以用于陀螺仪、加速度计设置
 * @param - dev : icm20608 设备
 * @param - reg : 要设置的通道寄存器首地址。
 * @param – anix : 要设置的通道，比如 X，Y，Z。
 * @param - val : 要设置的值。
 * @return : 0，成功；其他值，错误
 */
static int icm20608_sensor_set(struct icm20608_dev *dev, int reg,
int axis, int val)
{
	int ind, result;
	__be16 d = cpu_to_be16(val);

	ind = (axis - IIO_MOD_X) * 2;
	result = regmap_bulk_write(dev->regmap, reg + ind, (u8 *)&d, 2);
	if (result)
		return -EINVAL;

	return 0;
}

......

/*
 * @description : 设置 ICM20608 的陀螺仪计量程(分辨率)
 * @param - dev : icm20608 设备
 * @param - val : 量程(分辨率值)。
 * @return : 0，成功；其他值，错误
 */
static int icm20608_write_gyro_scale(struct icm20608_dev *dev,
int val)
{
	int result, i;
	u8 d;

	for (i = 0; i < ARRAY_SIZE(gyro_scale_icm20608); ++i) {
		if (gyro_scale_icm20608[i] == val) {
			d = (i << 3);
			result = regmap_write(dev->regmap, ICM20_GYRO_CONFIG, d);
			if (result)
				return result;
			return 0;
		}
	}
	return -EINVAL;
}

/*
 * @description : 设置 ICM20608 的加速度计量程(分辨率)
 * @param - dev : icm20608 设备
 * @param - val : 量程(分辨率值)。
 * @return : 0，成功；其他值，错误
 */
static int icm20608_write_accel_scale(struct icm20608_dev *dev,
int val)
{
	int result, i;
	u8 d;

	for (i = 0; i < ARRAY_SIZE(accel_scale_icm20608); ++i) {
		if (accel_scale_icm20608[i] == val) {
			d = (i << 3);
			result = regmap_write(dev->regmap,
			ICM20_ACCEL_CONFIG, d);
			if (result)
				return result;
			return 0;
		}
	}
	return -EINVAL;
}

......

/*
 * @description : 写函数，当向 sysfs 中的文件写数据的时候最终此函数会执行，
 * ：一般在此函数里面设置传感器，比如量程等。
 * @param - indio_dev	: iio_dev
 * @param - chan : 通道
 * @param - val	: 应用程序写入的值，如果是小数值的话，val 是整数部分。
 * @param - val2 : 应用程序写入的值，如果是小数值的话，val2 是小数部分。
 * @return : 0，成功；其他值，错误
 */
static int icm20608_write_raw(struct iio_dev *indio_dev,
struct iio_chan_spec const *chan,
int val, int val2, long mask)
{
	struct icm20608_dev *dev = iio_priv(indio_dev);
	int ret = 0;

	iio_device_claim_direct_mode(indio_dev);
	switch (mask) {
	case IIO_CHAN_INFO_SCALE: /* 设置陀螺仪和加速度计的分辨率 */
		switch (chan->type) {
		case IIO_ANGL_VEL: /* 设置陀螺仪 */
			mutex_lock(&dev->lock);
			ret = icm20608_write_gyro_scale(dev, val2);
			mutex_unlock(&dev->lock);
			break;
		case IIO_ACCEL: /* 设置加速度计 */
			mutex_lock(&dev->lock);
			ret = icm20608_write_accel_scale(dev, val2);
			mutex_unlock(&dev->lock);
			break;
		default:
			ret = -EINVAL;
			break;
		}
		break;
	case IIO_CHAN_INFO_CALIBBIAS: /* 设置陀螺仪和加速度计的校准值 */
		switch (chan->type) {
		case IIO_ANGL_VEL:	/* 设置陀螺仪校准值 */
			mutex_lock(&dev->lock);
			ret = icm20608_sensor_set(dev, ICM20_XG_OFFS_USRH,
			chan->channel2, val);
			mutex_unlock(&dev->lock);
			break;
		case IIO_ACCEL:	/* 加速度计校准值 */
			mutex_lock(&dev->lock);
			ret = icm20608_sensor_set(dev, ICM20_XA_OFFSET_H,
			chan->channel2, val);
			mutex_unlock(&dev->lock);
			break;
		default:
			ret = -EINVAL;
			break;
		}
		break;
	default:
		ret = -EINVAL;
		break;
	}

	iio_device_release_direct_mode(indio_dev);
	return ret;
}

/*
 * @description : 用户空间写数据格式，比如我们在用户空间操作 sysfs 来设置传
 * ：感器的分辨率，如果分辨率带小数，那么这个小数传递到内核空间
 * : 应该扩大多少倍，此函数就是用来设置这个的。
 * @param - indio_dev : iio_dev
 * @param - chan : 通道
 * @param - mask : 掩码
 * @return : 0，成功；其他值，错误
 */
static int icm20608_write_raw_get_fmt(struct iio_dev *indio_dev,
struct iio_chan_spec const *chan, long mask)
{
	switch (mask) {
	case IIO_CHAN_INFO_SCALE:
		switch (chan->type) {
		case IIO_ANGL_VEL:
			return IIO_VAL_INT_PLUS_MICRO;
		default:
			return IIO_VAL_INT_PLUS_NANO;
		}
	default:
		return IIO_VAL_INT_PLUS_MICRO;
	}
	return -EINVAL;
}

/*
 * iio_info 结构体变量
 */
static const struct iio_info icm20608_info = {
	.read_raw = icm20608_read_raw,
	.write_raw = icm20608_write_raw,
	.write_raw_get_fmt = &icm20608_write_raw_get_fmt,
};
```

- 第 188~200 行，icm20608_sensor_set 函数用于设置指定通道，也就是向 ICM20608 的指定寄存器写入数据，用于设置陀螺仪和加速度计的校准值。
- 第 262~277 行，icm20608_write_gyro_scale 函数用于设置 ICM20608 陀螺仪量程。
- 第 285~300 行，icm20608_write_accel_scale 函数用于设置 ICM20608 加速度计量程。
- 第 391~443 行，icm20608_write_raw 函数，也就是 iio_info 的 write 函数，用户空间向驱动程序写入数据以后此函数就会执行，这里我们用来配置 ICM20608 的陀螺仪和加速度量程和校准值。第 400 行的 `IIO_CHAN_INFO_SCALE` 表示此分支是设置比例(量程的)，第 404 行设置陀螺仪的量程，第 409 行设置加速度计的量程。第 417 行的 `IIO_CHAN_INFO_CALIBBIAS` 表示此分支是设置校准值的，第 421 行设置陀螺仪各轴校准值，第 427 行设置加速度计各轴校准值。
- 第 454~469 行，icm20608_write_raw_get_fmt 函数，也就是 iio_info 的 write_raw_get_fmt 函数。此函数用来指定用户空间写入的数据格式，第 458 行的 `IIO_CHAN_INFO_SCALE` 分支表示要操作的是比例，也就是用户空间要设置陀螺仪和加速度计的量程。第 460 行表示如果写入的是陀螺仪比例，那么小数部分就要扩大 1000000 倍。第 463 行表示如果是其他的就扩 1000000000 倍，比如加速度计。比如我们在用户空间要设置加速度计量程为±4g，只需要向 `in_accel_scale` 写入 0.000122070，那么最终传入到驱动里面的就是 0.000122070*1000000000=122070。

ICM20608 的 IIO 驱动就已经编写完成了，接下来就是编写测试 APP。
## 75.4 测试应用程序编写

### 75.4.1 linux 文件流读取
前面我们都是直接使用 cat 命令读取对应文件的内容，如果要连续不断的读取传感器数据就不能用 cat 命令了，我们需要编写对应的 APP 软件，在编写 APP 之前我们先了解一下所要用到的 API 函数。

首先我们要知道，前面使用 cat 命令读取到的文件内容字符串，虽然看起来像是数字。比如我们使用 cat 命令读取到的 in_accel_scale 如图 75.4.1 所示： 图 75.4.1 in_accel_scale 文件内容
图 75.4.1 中 in_accel_scale 文件内容为 0.000488281，但是这里的 0.000488281 是字符串，并不是具体的数字，所以我们需要将其转换为对应的数字。另外 in_accel_scale 是流文件，也叫做标准文件 I/O 流，因此打开、读写操作要使用文件流操作函数。

#### 1、打开文件流
打开文件流使用 `fopen` 函数，函数原型如下：
```c
FILE *fopen(const char *pathname, const char *mode)
```
函数参数和返回值含义如下：
- pathname：需要打开的文件流路径。
- mode：打开方式，可选的打开方式如表 75.4.1 所示：

| mode | 描述 |
|------|------|
| r | 打开只读文件。 |
| r+ | 打开读写文件。 |
| w | 打开只写文件，如文件存在则文件长度清零，如文件不存在就自动创建，文件流指针调整到文件头部。 |
| w+ | 打开可读写文件，如文件存在则文件长度清零，如文件不存在就自动创建，文件流指针调整到文件头部。 |
| a | 以追加的方式打开只写文件，如果文件不存在就新建文件，如果存在就将数据追加到文件末尾。 |
| a+ | 以追加的方式打开读写文件，如果文件不存在就新建文件，如果存在就将数据追加到文件末尾。 |

表 75.4.1 打开方式
返回值：NULL，打开错误；其他值，打开成功的文件流指针，为 FILE 类型。

#### 2、关闭文件流
关闭文件流使用函数 `fclose`，函数原型如下：
```c
int fclose(FILE *stream)
```
函数参数和返回值含义如下：
- stream：要关闭的文件流指针。
- 返回值：0，关闭成功；EOF，关闭错误。

#### 3、读取文件流
要读取文件流使用 `fread` 函数，函数原型如下：
```c
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream)
```
`fread` 函数用于从给定的输入流中读取最多 `nmemb` 个对象到数组 `ptr` 中，函数参数和返回值含义如下：
- ptr：要读取的数组中首个对象的指针。
- size：每个对象的大小。
- nmemb：要读取的对象个数。
- stream：要读取的文件流。
- 返回值：返回读取成功的对象个数，如果出现错误或到文件末尾，那么返回一个短计数值(或者 0)。

#### 4、写文件流
要向文件流写入数据，使用 `fwrite` 函数，函数原型如下：
```c
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```
`fwrite` 函数用于向给定的文件流中写入最多 `nmemb` 个对象，函数参数和返回值含义如下：
- ptr：要写入的数组中首个对象的指针。
- size：每个对象的大小。
- nmemb：要写入的对象个数。
- stream：要写入的文件流。
- 返回值：返回成功写入的对象个数，如果出现错误或到文件末尾，那么返回一个短计数值(或者 0)。

#### 5、格式化输入文件流
`fscanf` 函数用于从一个文件流中格式化读取数据，`fscanf` 函数在遇到空格和换行符的时候就会结束。前面我们说了 IIO 框架下的 sysfs 文件内容都是字符串，比如 in_accel_scale 文件内容为"0.000488281"，这是一串字符串，并不是具体的数字，因此我们在读取的时候就需要使用字符串读取格式。在这里就可以使用 `fscanf` 函数来格式化读取文件内容，函数原型如下：
```c
int fscanf(FILE *stream, const char *format, [argument...])
```
`fscanf` 用法和 `scanf` 类似，函数参数和返回值含义如下：
- stream：要操作的文件流。
- format：格式。
- argument：保存读取到的数据。
- 返回值：成功读取到的数据个数，如果读到文件末尾或者读取错误就返回 EOF。

### 75.4.2 编写测试 APP
新建名为 icm20608App.c 的文件，输入如下内容：

**示例代码 75.4.2.1 icm20608App.c 文件代码段**
```c
/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名	: icm20608.c
作者 	: 左忠凯
版本 	: V1.0
描述 	: icm20608 设备 iio 框架测试程序。
其他 	: 无
使用方法 ：./icm20608App
论坛 	: www.openedv.com
日志 	: 初版 V1.0 2021/8/17 左忠凯创建
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

/* icm20608 iio 框架对应的文件路径 */
static char *file_path[] = {
	"/sys/bus/iio/devices/iio:device1/in_accel_scale",
	"/sys/bus/iio/devices/iio:device1/in_accel_x_calibbias",
	"/sys/bus/iio/devices/iio:device1/in_accel_x_raw",
	"/sys/bus/iio/devices/iio:device1/in_accel_y_calibbias",
	"/sys/bus/iio/devices/iio:device1/in_accel_y_raw",
	"/sys/bus/iio/devices/iio:device1/in_accel_z_calibbias",
	"/sys/bus/iio/devices/iio:device1/in_accel_z_raw",
	"/sys/bus/iio/devices/iio:device1/in_anglvel_scale",
	"/sys/bus/iio/devices/iio:device1/in_anglvel_x_calibbias",
	"/sys/bus/iio/devices/iio:device1/in_anglvel_x_raw",
	"/sys/bus/iio/devices/iio:device1/in_anglvel_y_calibbias",
	"/sys/bus/iio/devices/iio:device1/in_anglvel_y_raw",
	"/sys/bus/iio/devices/iio:device1/in_anglvel_z_calibbias",
	"/sys/bus/iio/devices/iio:device1/in_anglvel_z_raw",
	"/sys/bus/iio/devices/iio:device1/in_temp_offset",
	"/sys/bus/iio/devices/iio:device1/in_temp_raw",
	"/sys/bus/iio/devices/iio:device1/in_temp_scale",
};

/* 文件路径索引，要和 file_path 里面的文件顺序对应 */
enum path_index {
	IN_ACCEL_SCALE = 0,
	IN_ACCEL_X_CALIBBIAS,
	IN_ACCEL_X_RAW,
	IN_ACCEL_Y_CALIBBIAS,
	IN_ACCEL_Y_RAW,
	IN_ACCEL_Z_CALIBBIAS,
	IN_ACCEL_Z_RAW,
	IN_ANGLVEL_SCALE,
	IN_ANGLVEL_X_CALIBBIAS,
	IN_ANGLVEL_X_RAW,
	IN_ANGLVEL_Y_CALIBBIAS,
	IN_ANGLVEL_Y_RAW,
	IN_ANGLVEL_Z_CALIBBIAS,
	IN_ANGLVEL_Z_RAW,
	IN_TEMP_OFFSET,
	IN_TEMP_RAW,
	IN_TEMP_SCALE,
};

/*
 * icm20608 数据设备结构体
 */
struct icm20608_dev{
	int accel_x_calibbias, accel_y_calibbias, accel_z_calibbias;
	int accel_x_raw, accel_y_raw, accel_z_raw;

	int gyro_x_calibbias, gyro_y_calibbias, gyro_z_calibbias;
	int gyro_x_raw, gyro_y_raw, gyro_z_raw;

	int temp_offset, temp_raw;

	float accel_scale, gyro_scale, temp_scale;

	float gyro_x_act, gyro_y_act, gyro_z_act;
	float accel_x_act, accel_y_act, accel_z_act;
	float temp_act;
};

struct icm20608_dev icm20608;

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
 * @description : 获取 ICM20608 数据
 * @param - dev : 设备结构体
 * @return : 0 成功;其他 失败
 */
static int sensor_read(struct icm20608_dev *dev)
{
	int ret = 0;
	char str[50];

	/* 1、获取陀螺仪原始数据 */
	SENSOR_FLOAT_DATA_GET(ret, IN_ANGLVEL_SCALE, str, gyro_scale);
	SENSOR_INT_DATA_GET(ret, IN_ANGLVEL_X_RAW, str, gyro_x_raw);
	SENSOR_INT_DATA_GET(ret, IN_ANGLVEL_Y_RAW, str, gyro_y_raw);
	SENSOR_INT_DATA_GET(ret, IN_ANGLVEL_Z_RAW, str, gyro_z_raw);

	/* 2、获取加速度计原始数据 */
	SENSOR_FLOAT_DATA_GET(ret, IN_ACCEL_SCALE, str, accel_scale);
	SENSOR_INT_DATA_GET(ret, IN_ACCEL_X_RAW, str, accel_x_raw);
	SENSOR_INT_DATA_GET(ret, IN_ACCEL_Y_RAW, str, accel_y_raw);
	SENSOR_INT_DATA_GET(ret, IN_ACCEL_Z_RAW, str, accel_z_raw);

	/* 3、获取温度值 */
	SENSOR_FLOAT_DATA_GET(ret, IN_TEMP_SCALE, str, temp_scale);
	SENSOR_INT_DATA_GET(ret, IN_TEMP_OFFSET, str, temp_offset);
	SENSOR_INT_DATA_GET(ret, IN_TEMP_RAW, str, temp_raw);

	/* 3、转换为实际数值 */
	dev->accel_x_act = dev->accel_x_raw * dev->accel_scale;
	dev->accel_y_act = dev->accel_y_raw * dev->accel_scale;
	dev->accel_z_act = dev->accel_z_raw * dev->accel_scale;

	dev->gyro_x_act = dev->gyro_x_raw * dev->gyro_scale;
	dev->gyro_y_act = dev->gyro_y_raw * dev->gyro_scale;
	dev->gyro_z_act = dev->gyro_z_raw * dev->gyro_scale;

	dev->temp_act = ((dev->temp_raw - dev->temp_offset) / dev->temp_scale) + 25;
	return ret;
}

/*
 * @description : main 主程序
 * @param – argc : argv 数组元素个数
 * @param – argv : 具体参数
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
		ret = sensor_read(&icm20608);
		if(ret == 0) { /* 数据读取成功 */
			printf("\r\n 原始值:\r\n");
			printf("gx = %d, gy = %d, gz = %d\r\n",icm20608.gyro_x_raw,icm20608.gyro_y_raw,icm20608.gyro_z_raw);
			printf("ax = %d, ay = %d, az = %d\r\n",icm20608.accel_x_raw,icm20608.accel_y_raw,icm20608.accel_z_raw);
			printf("temp = %d\r\n", icm20608.temp_raw);
			printf("实际值:");
			printf("act gx = %.2f°/S, act gy = %.2f°/S,act gz = %.2f°/S\r\n", icm20608.gyro_x_act,icm20608.gyro_y_act, icm20608.gyro_z_act);
			printf("act ax = %.2fg, act ay = %.2fg,act az = %.2fg\r\n", icm20608.accel_x_act,icm20608.accel_y_act,icm20608.accel_z_act);
			printf("act temp = %.2f°C\r\n", icm20608.temp_act);
		}
		usleep(100000); /*100ms */
	}

	return 0;
}
```

- 第 17~19 行，SENSOR_FLOAT_DATA_GET 宏用于读取指定路径的文件内容，然后将读到的浮点型字符串数据转换为具体的浮点数据。第 19 行使用 `atof` 函数将浮点字符串转换为具体的浮点数值。
- 第 22~24 行，SENSOR_INT_DATA_GET 宏用于读取指定路径的文件内容，将读取到的整数型字符串数据转换为具体的整数值，第 24 行使用 `atoi` 函数将整数字符串转换为具体的整数值。
`atof` 和 `atoi` 这两个函数是标准的 C 库函数，并不需要我们自己编写，添加 "stdlib.h" 头文件就可以直接调用。这两个函数都只有一个参数，就是要转换的字符串。返回值就是转换成功以后字符串对应的数值。
- 第 27~45 行，需要操作的文件路径。 第 48~66 行，文件路径索引，顺序要和 `file_path` 里面的文件路径对应。 第 71~85 行，icm20608_dev 结构体为自定义设备结构体，用于保存 ICM20608 传感器数据。
- 第 95~115 行，file_data_read 函数用于读取指定文件，第一个参数 filename 是要读取的文件路径。第二个参数 str 为读取到的文件内容，为字符串类型，因为本章例程所读取的文件内容都是字符串。第 100 行调用 `fopen` 函数打开指定的文件流，第 106 行调用 `fscanf` 函数进行格式化读取，也就是按照字符串方式读取文件，文件内容保存到 str 参数里面。当读取到文件末尾的时候，第 111 行调用 `fseek` 函数将读取指针调整到文件头，以备下次重新读取。第 113 行调用 `fclose` 函数关闭对应的文件流。
- 第 122~155 行，sensor_read 函数用于读取 ICM20608 传感器数据，包括陀螺仪、加速度和温度计的原始值，还有加速度计和陀螺仪的分辨率等，最后将获取到的原始值转换为具体的数值。
- 第 163~188 行，main 函数，在 while 循环中调用 sensor_read 函数读取 ICM20608 数据，并将读到的数据打印出来。

### 75.4.3 运行测试
输入如下命令编译测试 icm20608App.c 这个测试程序：
```bash
arm-linux-gnueabihf-gcc icm20608App.c -o icm20608App
```
编译成功以后就会生成 icm20608App 这个应用程序。
将 icm20608.ko 和 icm20608App 这两个文件拷贝到 rootfs/lib/modules/4.1.15 目录中，重启开发板，进入到目录 `lib/modules/4.1.15` 中，输入如下命令加载 icm20608.ko 驱动模块：
```bash
depmod				#第一次加载驱动的时候需要运行此命令
modprobe icm20608.ko 	#加载驱动
```
驱动加载成功以后使用如下命令来测试：
```bash
./icm20608App
```
如果驱动和 APP 工作正常的话就会不断的打印出 ICM20608 数据，包括陀螺仪和加速度计的原始数据和转换后的实际数值、温度等。
另外，我们也编写 AP3216C 这个光传感器的对应的 IIO 驱动，但是由于 AP3216C 没有数据准备就绪中断，因此 AP3216C 的 IIO 驱动没有实现触发缓冲功能。AP3216C 的 IIO 驱动程序已经放到了开发板光盘中，路径为：开发板光盘->01、程序源码->02、Linux 驱动例程->27_iio->iic。
