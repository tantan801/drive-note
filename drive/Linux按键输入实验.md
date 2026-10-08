## 第四十九章 Linux 按键输入实验

在 Linux 下编写 GPIO 输入驱动程序。I.MX6U-ALPHA 开发板上有一个按键，我们就使用此按键来完成 GPIO 输入驱动程序，同时利用第四十七章讲的原子操作来对按键值进行保护。

---

### 49.1 Linux 下按键驱动原理

按键驱动和 LED 驱动原理上来讲基本都是一样的，都是操作 GPIO，只不过一个是读取 GPIO 的高低电平，一个是从 GPIO 输出高低电平。在驱动程序中使用一个整形变量来表示按键值，应用程序通过 read 函数来读取按键值，判断按键有没有按下。在这里，这个保存按键值的变量就是个共享资源，驱动程序要向其写入按键值，应用程序要读取按键值。所以我们要对其进行保护，对于整形变量而言我们首选的就是原子操作，使用原子操作对变量进行赋值以及读取。

---

### 49.2 硬件原理图分析

本试验我们用到的硬件有：
1. LED 灯 LED0。
2. 蜂鸣器。
3. 1 个按键 KEY0。

按键 KEY0 的原理图：

![KEY0按键输入原理图](./photo/Linux按键输入实验/2-1%20key按键输入原理图.png)

从原理图可以看出，按键 KEY0 是连接到 I.MX6U 的 UART1_CTS 这个 IO 上的，KEY0 接了一个 10K 的上拉电阻，因此 KEY0 没有按下的时候 UART1_CTS 应该是高电平，当 KEY0 按下以后 UART1_CTS 就是低电平。
- R15（10K）上拉电阻接到 DCDC_3V3
- 按键松开：引脚被上拉到 3.3V → 高电平
- 按键按下：KEY0 引脚直接连通 GND → 低电平

这个引脚原本功能是 UART1_CTS（串口 1 的 CTS 硬件流控引脚），使用时需要在 pinctrl 中将引脚复用功能修改为 GPIO 输入，并且要确认 UART1 没有占用该引脚，若占用需要屏蔽。

---

### 49.3 实验程序编写

本实验对应的例程路径为：开发板光盘 → 2、Linux 驱动例程 → `11_key`。

---

#### 49.3.1 修改设备树文件

##### 1、添加 pinctrl 节点

I.MX6U-ALPHA 开发板上的 KEY 使用了 UART1_CTS_B 这个 PIN，打开 `imx6ull-alientek-emmc.dts`，在 iomuxc 节点的 imx6ul-evk 子节点下创建一个名为 "pinctrl_key" 的子节点，节点内容如下所示：

**示例代码 49.3.1.1 按键 pinctrl 节点**

```dts
pinctrl_key: keygrp {
    fsl,pins = <
        MX6UL_PAD_UART1_CTS_B__GPIO1_IO18 0xF080 /* KEY0 */
    >;
};
```

第 3 行，将 UART1_CTS_B 这个 PIN 复用为 GPIO1_IO18。

##### 2、添加 KEY 设备节点

在根节点 "/" 下创建 KEY 节点，节点名为 "key"，节点内容如下：

**示例代码 49.3.1.2 创建 KEY 节点**

```dts
key {
    #address-cells = <1>;
    #size-cells = <1>;
    compatible = "atkalpha-key";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_key>;
    key-gpio = <&gpio1 18 GPIO_ACTIVE_LOW>;
    status = "okay"; /* KEY0 */
};
```

第 6 行，pinctrl-0 属性设置 KEY 所使用的 PIN 对应的 pinctrl 节点。第 7 行，key-gpio 属性指定了 KEY 所使用的 GPIO。

##### 3、检查 PIN 是否被其他外设使用

在本章实验中 KEY 使用的 PIN 为 UART1_CTS_B，因此先检查 PIN 为 UART1_CTS_B 这个 PIN 有没有被其他的 pinctrl 节点使用，如果有使用的话就要屏蔽掉，然后再检查 GPIO1_IO18 这个 GPIO 有没有被其他外设使用，如果有的话也要屏蔽掉。

设备树编写完成以后使用 `make dtbs` 命令重新编译设备树，然后使用新编译出来的 `imx6ull-alientek-emmc.dtb` 文件启动 Linux 系统。启动成功以后进入 `/proc/device-tree` 目录中查看 "key" 节点是否存在，如果存在的话就说明设备树基本修改成功 (具体还要驱动验证)。

---

#### 49.3.2 按键驱动程序编写

设备树准备好以后就可以编写驱动程序了，新建名为 "11_key" 的文件夹，然后在 11_key 文件夹里面创建 vscode 工程，工作区命名为 "key"。工程创建好以后新建 `key.c` 文件，在 `key.c` 里面输入如下内容：

**示例代码 49.3.2.1 key.c 文件代码**

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

/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : key.c
作者 : 左忠凯
版本 : V1.0
描述 : Linux按键输入驱动实验
其他 : 无
论坛 : www.openedv.com
日志 : 初版V1.0 2019/7/18 左忠凯创建
***************************************************************/
#define KEY_CNT 1 /* 设备号个数 */
#define KEY_NAME "key" /* 名字 */

/* 定义按键值 */
#define KEY0VALUE 0XF0 /* 按键值 */
#define INVAKEY 0X00 /* 无效的按键值 */

/* key设备结构体 */
struct key_dev{
    dev_t devid; /* 设备号 */
    struct cdev cdev; /* cdev */
    struct class *class; /* 类 */
    struct device *device; /* 设备 */
    int major; /* 主设备号 */
    int minor; /* 次设备号 */
    struct device_node *nd; /* 设备节点 */
    int key_gpio; /* key所使用的GPIO编号 */
    atomic_t keyvalue; /* 按键值 */
};

struct key_dev keydev; /* key设备 */

/*
 * @description : 初始化按键IO，open函数打开驱动的时候
 * 初始化按键所使用的GPIO引脚。
 * @param : 无
 * @return : 无
 */
static int keyio_init(void)
{
    keydev.nd = of_find_node_by_path("/key");
    if (keydev.nd== NULL) {
        return -EINVAL;
    }

    keydev.key_gpio = of_get_named_gpio(keydev.nd ,"key-gpio", 0);
    if (keydev.key_gpio < 0) {
        printk("can't get key0\r\n");
        return -EINVAL;
    }
    printk("key_gpio=%d\r\n", keydev.key_gpio);

    /* 初始化key所使用的IO */
    gpio_request(keydev.key_gpio, "key0"); /* 请求IO */
    gpio_direction_input(keydev.key_gpio); /* 设置为输入 */
    return 0;
}

/*
 * @description : 打开设备
 * @param – inode : 传递给驱动的inode
 * @param - filp : 设备文件，file结构体有个叫做private_data的成员变量
 * 一般在open的时候将private_data指向设备结构体。
 * @return : 0 成功;其他 失败
 */
static int key_open(struct inode *inode, struct file *filp)
{
    int ret = 0;
    filp->private_data = &keydev; /* 设置私有数据 */

    ret = keyio_init(); /* 初始化按键IO */
    if (ret < 0) {
        return ret;
    }

    return 0;
}

/*
 * @description : 从设备读取数据
 * @param - filp : 要打开的设备文件(文件描述符)
 * @param - buf : 返回给用户空间的数据缓冲区
 * @param - cnt : 要读取的数据长度
 * @param - offt : 相对于文件首地址的偏移
 * @return : 读取的字节数，如果为负值，表示读取失败
 */
static ssize_t key_read(struct file *filp, char __user *buf, size_t cnt, loff_t *offt)
{
    int ret = 0;
    unsigned char value;
    struct key_dev *dev = filp->private_data;

    if (gpio_get_value(dev->key_gpio) == 0) { /* key0按下 */
        while(!gpio_get_value(dev->key_gpio)); /* 等待按键释放 */
        atomic_set(&dev->keyvalue, KEY0VALUE);
    } else { /* 无效的按键值 */
        atomic_set(&dev->keyvalue, INVAKEY);
    }

    value = atomic_read(&dev->keyvalue); /* 保存按键值 */
    ret = copy_to_user(buf, &value, sizeof(value));
    return ret;
}

/* 设备操作函数 */
static struct file_operations key_fops = {
    .owner = THIS_MODULE,
    .open = key_open,
    .read = key_read,
};

/*
 * @description : 驱动入口函数
 * @param : 无
 * @return : 无
 */
static int __init mykey_init(void)
{
    /* 初始化原子变量 */
    atomic_set(&keydev.keyvalue, INVAKEY);

    /* 注册字符设备驱动 */
    /* 1、创建设备号 */
    if (keydev.major) { /* 定义了设备号 */
        keydev.devid = MKDEV(keydev.major, 0);
        register_chrdev_region(keydev.devid, KEY_CNT, KEY_NAME);
    } else { /* 没有定义设备号 */
        alloc_chrdev_region(&keydev.devid, 0, KEY_CNT, KEY_NAME);
        keydev.major = MAJOR(keydev.devid); /* 获取分配号的主设备号 */
        keydev.minor = MINOR(keydev.devid); /* 获取分配号的次设备号 */
    }

    /* 2、初始化cdev */
    keydev.cdev.owner = THIS_MODULE;
    cdev_init(&keydev.cdev, &key_fops);

    /* 3、添加一个cdev */
    cdev_add(&keydev.cdev, keydev.devid, KEY_CNT);

    /* 4、创建类 */
    keydev.class = class_create(THIS_MODULE, KEY_NAME);
    if (IS_ERR(keydev.class)) {
        return PTR_ERR(keydev.class);
    }

    /* 5、创建设备 */
    keydev.device = device_create(keydev.class, NULL, keydev.devid, NULL, KEY_NAME);
    if (IS_ERR(keydev.device)) {
        return PTR_ERR(keydev.device);
    }

    return 0;
}

/*
 * @description : 驱动出口函数
 * @param : 无
 * @return : 无
 */
static void __exit mykey_exit(void)
{
    /* 注销字符设备驱动 */
    gpio_free(keydev.key_gpio);
    cdev_del(&keydev.cdev); /* 删除cdev */
    unregister_chrdev_region(keydev.devid, KEY_CNT); /* 注销设备号 */

    device_destroy(keydev.class, keydev.devid);
    class_destroy(keydev.class);
}

module_init(mykey_init);
module_exit(mykey_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("tantan");
```

第 36~46 行，结构体 `key_dev` 为按键的设备结构体，第 45 行的原子变量 `keyvalue` 用于记录按键值。
第 56~74 行，函数 `keyio_init` 用于初始化按键，从设备树中获取按键的 gpio 信息，然后设置为输入。将按键的初始化代码提取出来，将其作为独立的一个函数有利于提高程序的模块化设计。
第 83~94 行，`key_open` 函数通过调用 `keyio_init` 函数来初始化按键所使用的 IO，应用程序每次打开按键驱动文件的时候都会初始化一次按键 IO。
第 104~120 行，`key_read` 函数，应用程序通过 read 函数读取按键值的时候此函数就会执行。第 110 行读取按键 IO 的电平，如果为 0 的话就表示按键按下了，如果按键按下的话第 111 行就等待按键释放。按键释放以后标记按键值为 `KEY0VALUE`。
第 135~171 行，驱动入口函数，第 138 行调用 `atomic_set` 函数初始化原子变量默认为无效值。
第 178~187 行，驱动出口函数。`key.c` 文件代码很简单，重点就是 `key_read` 函数读取按键值，要对 `keyvalue` 进行保护。

---

#### 49.3.3 编写测试 APP

新建名为 `keyApp.c` 的文件，然后输入如下所示内容：

**示例代码 49.3.2.2 keyApp.c 文件代码**

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
文件名 : keyApp.c
作者 : 左忠凯
版本 : V1.0
描述 : 按键输入测试应用程序
其他 : 无
使用方法 ：./keyApp /dev/key
论坛 : www.openedv.com
日志 : 初版V1.0 2019/1/30 左忠凯创建
***************************************************************/

/* 定义按键值 */
#define KEY0VALUE 0XF0
#define INVAKEY 0X00

/*
 * @description : main主程序
 * @param - argc : argv数组元素个数
 * @param - argv : 具体参数
 * @return : 0 成功;其他 失败
 */
int main(int argc, char *argv[])
{
    int fd, ret;
    char *filename;
    unsigned char keyvalue;

    if(argc != 2){
        printf("Error Usage!\r\n");
        return -1;
    }

    filename = argv[1];

    /* 打开key驱动 */
    fd = open(filename, O_RDWR);
    if(fd < 0){
        printf("file %s open failed!\r\n", argv[1]);
        return -1;
    }

    /* 循环读取按键值数据！ */
    while(1) {
        read(fd, &keyvalue, sizeof(keyvalue));
        if (keyvalue == KEY0VALUE) { /* KEY0 */
            printf("KEY0 Press, value = %#X\r\n", keyvalue);/* 按下 */
        }
    }

    ret= close(fd); /* 关闭文件 */
    if(ret < 0){
        printf("file %s close failed!\r\n", argv[1]);
        return -1;
    }
    return 0;
}
```

第 51~56 行，循环读取 `/dev/key` 文件，也就是循环读取按键值，并且将按键值打印出来。

---

### 49.4 运行测试

#### 49.4.1 编译驱动程序和测试 APP

##### 1、编译驱动程序

编写 Makefile 文件，本章实验的 Makefile 文件和第四十章实验基本一样，只是将 obj-m 变量的值改为 key.o，Makefile 内容如下所示：

**示例代码 49.4.1.1 Makefile 文件**

```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)
obj-m := key.o

build: kernel_modules

kernel_modules:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules
clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

第 4 行，设置 obj-m 变量的值为 key.o。

输入如下命令编译出驱动模块文件：

```bash
make -j32
```

编译成功以后就会生成一个名为 `key.ko` 的驱动模块文件。

##### 2、编译测试 APP

输入如下命令编译测试 `keyApp.c` 这个测试程序：

```bash
arm-linux-gnueabihf-gcc keyApp.c -o keyApp
```

编译成功以后就会生成 `keyApp` 这个应用程序。

---

#### 49.4.2 运行测试

将上一小节编译出来的 `key.ko` 和 `keyApp` 这两个文件拷贝到 `rootfs/lib/modules/4.1.15` 目录中，重启开发板，进入到目录 `lib/modules/4.1.15` 中，输入如下命令加载 `key.ko` 驱动模块：

```bash
depmod         //第一次加载驱动的时候需要运行此命令
modprobe key.ko //加载驱动
```

驱动加载成功以后如下命令来测试：

```bash
./keyApp /dev/key
```

输入上述命令以后终端等待按键，按下开发板上的 KEY0 按键，`keyApp` 就会获取并且输出按键信息。从运行结果可以看出，当我们按下 KEY0 以后就会打印出 `KEY0 Press, value = 0XF0`，表示按键按下。但是大家可能会发现，有时候按下一次 KEY0 但是会输出好几行 `KEY0 Press, value = 0XF0`，这是因为我们的代码没有做按键消抖处理。

如果要卸载驱动的话输入如下命令即可：

```bash
rmmod key.ko
```