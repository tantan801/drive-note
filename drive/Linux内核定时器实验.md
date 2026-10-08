## 第五十章 Linux 内核定时器实验

定时器是我们最常用到的功能，一般用来完成定时功能，本章我们就来学习一下 Linux 内核提供的定时器 API 函数，通过这些定时器 API 函数我们可以完成很多要求定时的应用。Linux 内核也提供了短延时函数，比如微秒、纳秒、毫秒延时函数，本章我们就来学习一下这些和时间有关的功能。

---

### 50.1 Linux 时间管理和内核定时器简介

#### 50.1.1 内核时间管理简介

Linux 内核中有大量的函数需要时间管理，比如周期性的调度程序、延时程序、对于我们驱动编写者来说最常用的定时器。硬件定时器提供时钟源，时钟源的频率可以设置，设置好以后就周期性的产生定时中断，系统使用定时中断来计时。中断周期性产生的频率就是系统频率，也叫做节拍率 (tick rate)(有的资料也叫系统频率)，比如 1000Hz，100Hz 等等说的就是系统节拍率。系统节拍率是可以设置的，单位是 Hz，我们在编译 Linux 内核的时候可以通过图形化界面设置系统节拍率，按照如下路径打开配置界面：

Kernel Features -> Timer frequency ( [=y])

选中 "Timer frequency"。从配置界面可以看出，可选的系统节拍率为 100Hz、200Hz、250Hz、300Hz、500Hz 和 1000Hz，默认情况下选择 100Hz。

设置好以后打开 Linux 内核源码根目录下的 `.config` 文件，在此文件中有 `CONFIG_HZ` 定义，本教程默认 `CONFIG_HZ` 为 100。打开文件 `include/asm-generic/param.h`，有如下内容：

**示例代码 50.1.1.1 include/asm-generic/param.h 文件代码段**

```c
# undef HZ
# define HZ CONFIG_HZ
# define USER_HZ 100
# define CLOCKS_PER_SEC (USER_HZ)
```

第 7 行定义了一个宏 HZ，宏 HZ 就是 CONFIG_HZ，因此 HZ=100，我们后面编写 Linux 驱动的时候会常常用到 HZ，因为 HZ 表示一秒的节拍数，也就是频率。

大多数初学者看到系统节拍率默认为 100Hz 的时候都会有疑问，怎么这么小？100Hz 是可选的节拍率里面最小的。为什么不选择大一点的呢？这里就引出了一个问题：高节拍率和低节拍率的优缺点：
①、高节拍率会提高系统时间精度，如果采用 100Hz 的节拍率，时间精度就是 10ms，采用 1000Hz 的话时间精度就是 1ms，精度提高了 10 倍。高精度时钟的好处有很多，对于那些对时间要求严格的函数来说，能够以更高的精度运行，时间测量也更加准确。
②、高节拍率会导致中断的产生更加频繁，频繁的中断会加剧系统的负担，1000Hz 和 100Hz 的系统节拍率相比，系统要花费 10 倍的 "精力" 去处理中断。中断服务函数占用处理器的时间增加，但是现在的处理器性能都很强大，所以采用 1000Hz 的系统节拍率并不会增加太大的负载压力。根据自己的实际情况，选择合适的系统节拍率，本教程我们全部采用默认的 100Hz 系统节拍率。

Linux 内核使用全局变量 jiffies 来记录系统从启动以来的系统节拍数，系统启动的时候会将 jiffies 初始化为 0，jiffies 定义在文件 `include/linux/jiffies.h` 中，定义如下：

**示例代码 50.1.1.2 include/jiffies.h 文件代码段**

```c
extern u64 __jiffy_data jiffies_64;
extern unsigned long volatile __jiffy_data jiffies;
```

第 76 行，定义了一个 64 位的 jiffies_64。第 77 行，定义了一个 unsigned long 类型的 32 位的 jiffies。jiffies_64 和 jiffies 其实是同一个东西，jiffies_64 用于 64 位系统，而 jiffies 用于 32 位系统。为了兼容不同的硬件，jiffies 其实就是 jiffies_64 的低 32 位。当我们访问 jiffies 的时候其实访问的是 jiffies_64 的低 32 位，使用 `get_jiffies_64` 这个函数可以获取 jiffies_64 的值。在 32 位的系统上读取 jiffies 的值，在 64 位的系统上 jiffes 和 jiffies_64 表示同一个变量，因此也可以直接读取 jiffies 的值。所以不管是 32 位的系统还是 64 位系统，都可以使用 jiffies。

前面说了 HZ 表示每秒的节拍数，jiffies 表示系统运行的 jiffies 节拍数，所以 `jiffies/HZ` 就是系统运行时间，单位为秒。不管是 32 位还是 64 位的 jiffies，都有溢出的风险，溢出以后会重新从 0 开始计数，相当于绕回来了，因此有些资料也将这个现象也叫做绕回。假如 HZ 为最大值 1000 的时候，32 位的 jiffies 只需要 49.7 天就发生了绕回，对于 64 位的 jiffies 来说大概需要 5.8 亿年才能绕回，因此 jiffies_64 的绕回忽略不计。处理 32 位 jiffies 的绕回显得尤为重要，Linux 内核提供了几个 API 函数来处理绕回。

| 函数 | 描述 |
|------|------|
| time_after(unkown, known) | unkown 通常为 jiffies，known 通常是需要对比的值。unkown 超过 known 返回真 |
| time_before(unkown, known) | unkown 没有超过 known 返回真 |
| time_after_eq(unkown, known) | unkown 大于等于 known 返回真 |
| time_before_eq(unkown, known) | unkown 小于等于 known 返回真 |

比如我们要判断某段代码执行时间有没有超时，此时就可以使用如下所示代码：

**示例代码 50.1.1.3 使用 jiffies 判断超时**

```c
unsigned long timeout;
timeout = jiffies + (2 * HZ); /* 超时的时间点 */

/*************************************
具体的代码
************************************/

/* 判断有没有超时 */
if(time_before(jiffies, timeout)) {
    /* 超时未发生 */
} else {
    /* 超时发生 */
}
```

timeout 就是超时时间点，比如我们要判断代码执行时间是不是超过了 2 秒，那么超时时间点就是 `jiffies+(2*HZ)`，如果 jiffies 大于 timeout 那就表示超时了，否则就是没有超时。通过函数 `time_before` 来判断 jiffies 是否小于 timeout，如果小于的话就表示没有超时。

为了方便开发，Linux 内核提供了几个 jiffies 和 ms、us、ns 之间的转换函数：

| 函数 | 描述 |
|------|------|
| int jiffies_to_usecs(const unsigned long j) | 将 jiffies 转为微秒 |
| int jiffies_to_msecs(const unsigned long j) | 将 jiffies 转为毫秒 |
| u64 jiffies_to_nsecs(const unsigned long j) | 将 jiffies 转为纳秒 |
| long msecs_to_jiffies(const unsigned int m) | 毫秒转 jiffies |
| long usecs_to_jiffies(const unsigned int u) | 微秒转 jiffies |
| unsigned long nsecs_to_jiffies(u64 n) | 纳秒转 jiffies |

---

#### 50.1.2 内核定时器简介

定时器是一个很常用的功能，需要周期性处理的工作都要用到定时器。Linux 内核定时器采用系统时钟来实现，并不是我们在裸机篇中讲解的 PIT 等硬件定时器。Linux 内核定时器使用很简单，只需要提供超时时间 (相当于定时值) 和定时处理函数即可，当超时时间到了以后设置的定时处理函数就会执行，和我们使用硬件定时器的套路一样，只是使用内核定时器不需要做一大堆的寄存器初始化工作。

重点：在使用内核定时器的时候要注意一点，内核定时器并不是周期性运行的，超时以后就会自动关闭，因此如果想要实现周期性定时，那么就需要在定时处理函数中重新开启定时器。

Linux 内核使用 `timer_list` 结构体表示内核定时器，`timer_list` 定义在文件 `include/linux/timer.h` 中：

**示例代码 50.1.2.1 timer_list 结构体**

```c
struct timer_list {
    struct list_head entry;
    unsigned long expires; /* 定时器超时时间，单位是节拍数 */
    struct tvec_base *base;
    void (*function)(unsigned long); /* 定时处理函数 */
    unsigned long data; /* 要传递给function函数的参数 */
    int slack;
};
```

要使用内核定时器首先要先定义一个 `timer_list` 变量，表示定时器，`timer_list` 结构体的 `expires` 成员变量表示超时时间，单位为节拍数。比如我们现在需要定义一个周期为 2 秒的定时器，那么这个定时器的超时时间就是 `jiffies+(2HZ)`，因此 `expires=jiffies+(2HZ)`。`function` 就是定时器超时以后的定时处理函数，我们要做的工作就放到这个函数里面，需要我们编写这个定时处理函数。

定义好定时器以后还需要通过一系列的 API 函数来初始化此定时器，这些函数如下：
1、**init_timer 函数**
`init_timer` 函数负责初始化 `timer_list` 类型变量，当我们定义了一个 `timer_list` 变量以后一定要先用 `init_timer` 初始化一下。
函数原型：`void init_timer(struct timer_list *timer)`
参数：timer：要初始化定时器。
返回值：无

2、**add_timer 函数**
`add_timer` 函数用于向 Linux 内核注册定时器，使用 `add_timer` 函数向内核注册定时器以后，定时器就会开始运行。
函数原型：`void add_timer(struct timer_list *timer)`

3、**del_timer 函数**
`del_timer` 函数用于删除一个定时器，不管定时器有没有被激活，都可以使用此函数删除。在多处理器系统上，定时器可能会在其他的处理器上运行，因此在调用 `del_timer` 函数删除定时器之前要先等待其他处理器的定时处理器函数退出。
原型：`int del_timer(struct timer_list * timer)`
返回值：0，定时器还没被激活；1，定时器已经激活。

4、**del_timer_sync 函数**
`del_timer_sync` 函数是 `del_timer` 函数的同步版，会等待其他处理器使用完定时器再删除，`del_timer_sync` 不能使用在中断上下文中。
原型：`int del_timer_sync(struct timer_list *timer)`

5、**mod_timer 函数**
`mod_timer` 函数用于修改定时值，如果定时器还没有激活的话，`mod_timer` 函数会激活定时器！
原型：`int mod_timer(struct timer_list *timer, unsigned long expires)`
返回值：0，调用 `mod_timer` 函数前定时器未被激活；1，调用 `mod_timer` 函数前定时器已被激活。

内核定时器一般的使用流程演示：

**示例代码 50.1.2.2 内核定时器使用方法演示**

```c
struct timer_list timer; /* 定义定时器 */

/* 定时器回调函数 */
void function(unsigned long arg)
{
    /*
     * 定时器处理代码
     */

    /* 如果需要定时器周期性运行的话就使用mod_timer
     * 函数重新设置超时值并且启动定时器。
     */
    mod_timer(&dev->timertest, jiffies + msecs_to_jiffies(2000));
}

/* 初始化函数 */
void init(void)
{
    init_timer(&timer); /* 初始化定时器 */

    timer.function = function; /* 设置定时处理函数 */
    timer.expires=jiffies + msecs_to_jiffies(2000);/* 超时时间2秒 */
    timer.data = (unsigned long)&dev; /* 将设备结构体作为参数 */

    add_timer(&timer); /* 启动定时器 */
}

/* 退出函数 */
void exit(void)
{
    del_timer(&timer); /* 删除定时器 */
    /* 或者使用 */
    del_timer_sync(&timer);
}
```

---

#### 50.1.3 Linux 内核短延时函数

有时候我们需要在内核中实现短延时，尤其是在 Linux 驱动中。Linux 内核提供了毫秒、微秒和纳秒延时函数：

| 函数 | 描述 |
|------|------|
| void ndelay(unsigned long nsecs) | 纳秒延时 |
| void udelay(unsigned long usecs) | 微秒延时 |
| void mdelay(unsigned long mseces) | 毫秒延时 |

---

### 50.2 硬件原理图分析

本章使用通过设置一个定时器来实现周期性的闪烁 LED 灯，因此本章例程就使用到了一个 LED 灯，关于 LED 灯的硬件原理图参考参考之前的小节即可。

---

### 50.3 实验程序编写

本实验对应的例程路径为：开发板光盘 → 2、Linux 驱动例程 → `12_timer`。
本章实验我们使用内核定时器周期性的点亮和熄灭开发板上的 LED 灯，LED 灯的闪烁周期由内核定时器来设置，测试应用程序可以控制内核定时器周期。

---

#### 50.3.1 修改设备树文件

本章实验使用到了 LED 灯，LED 灯的设备树节点信息使用 45.4.1 小节创建的即可。

---

#### 50.3.2 定时器驱动程序编写

新建名为 "12_timer" 的文件夹，然后在 12_timer 文件夹里面创建 vscode 工程，工作区命名为 "timer"。工程创建好以后新建 `timer.c` 文件，在 `timer.c` 里面输入如下内容：

**示例代码 50.3.2.1 timer.c 文件**

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
#include <linux/timer.h>
#include <linux/spinlock.h>

/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : timer.c
作者 : 左忠凯
版本 : V1.0
描述 : Linux内核定时器实验
其他 : 无
论坛 : www.openedv.com
日志 : 初版V1.0 2019/7/24 左忠凯创建
***************************************************************/
#define TIMER_CNT 1 /* 设备号个数 */
#define TIMER_NAME "timer" /* 名字 */
#define CLOSE_CMD (_IO(0XEF, 0x1)) /* 关闭定时器 */
#define OPEN_CMD (_IO(0XEF, 0x2)) /* 打开定时器 */
#define SETPERIOD_CMD (_IO(0XEF, 0x3)) /* 设置定时器周期命令 */
#define LEDON 1 /* 开灯 */
#define LEDOFF 0 /* 关灯 */

/* timer设备结构体 */
struct timer_dev{
    dev_t devid; /* 设备号 */
    struct cdev cdev; /* cdev */
    struct class *class; /* 类 */
    struct device *device; /* 设备 */
    int major; /* 主设备号 */
    int minor; /* 次设备号 */
    struct device_node *nd; /* 设备节点 */
    int led_gpio; /* led所使用的GPIO编号 */
    int timeperiod; /* 定时周期,单位为ms */
    struct timer_list timer; /* 定义一个定时器 */
    spinlock_t lock; /* 定义自旋锁 */
};

struct timer_dev timerdev; /* timer设备 */

/*
 * @description : 初始化LED灯IO，open函数打开驱动的时候
 * 初始化LED灯所使用的GPIO引脚。
 * @param : 无
 * @return : 无
 */
static int led_init(void)
{
    int ret = 0;

    timerdev.nd = of_find_node_by_path("/gpioled");
    if (timerdev.nd== NULL) {
        return -EINVAL;
    }

    timerdev.led_gpio = of_get_named_gpio(timerdev.nd ,"led-gpio", 0);
    if (timerdev.led_gpio < 0) {
        printk("can't get led\r\n");
        return -EINVAL;
    }

    /* 初始化led所使用的IO */
    gpio_request(timerdev.led_gpio, "led"); /* 请求IO */
    ret = gpio_direction_output(timerdev.led_gpio, 1);
    if(ret < 0) {
        printk("can't set gpio!\r\n");
    }
    return 0;
}

/*
 * @description : 打开设备
 * @param – inode : 传递给驱动的inode
 * @param - filp : 设备文件，file结构体有个叫做private_data的成员变量
 * 一般在open的时候将private_data指向设备结构体。
 * @return : 0 成功;其他 失败
 */
static int timer_open(struct inode *inode, struct file *filp)
{
    int ret = 0;
    filp->private_data = &timerdev; /* 设置私有数据 */

    timerdev.timeperiod = 1000; /* 默认周期为1s */
    ret = led_init(); /* 初始化LED IO */
    if (ret < 0) {
        return ret;
    }
    return 0;
}

/*
 * @description : ioctl函数，
 * @param – filp : 要打开的设备文件(文件描述符)
 * @param - cmd : 应用程序发送过来的命令
 * @param - arg : 参数
 * @return : 0 成功;其他 失败
 */
static long timer_unlocked_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    struct timer_dev *dev = (struct timer_dev *)filp->private_data;
    int timerperiod;
    unsigned long flags;

    switch (cmd) {
    case CLOSE_CMD: /* 关闭定时器 */
        del_timer_sync(&dev->timer);
        break;
    case OPEN_CMD: /* 打开定时器 */
        spin_lock_irqsave(&dev->lock, flags);
        timerperiod = dev->timeperiod;
        spin_unlock_irqrestore(&dev->lock, flags);
        mod_timer(&dev->timer, jiffies + msecs_to_jiffies(timerperiod));
        break;
    case SETPERIOD_CMD: /* 设置定时器周期 */
        spin_lock_irqsave(&dev->lock, flags);
        dev->timeperiod = arg;
        spin_unlock_irqrestore(&dev->lock, flags);
        mod_timer(&dev->timer, jiffies + msecs_to_jiffies(arg));
        break;
    default:
        break;
    }
    return 0;
}

/* 设备操作函数 */
static struct file_operations timer_fops = {
    .owner = THIS_MODULE,
    .open = timer_open,
    .unlocked_ioctl = timer_unlocked_ioctl,
};

/* 定时器回调函数 */
void timer_function(unsigned long arg)
{
    struct timer_dev *dev = (struct timer_dev *)arg;
    static int sta = 1;
    int timerperiod;
    unsigned long flags;

    sta = !sta; /* 每次都取反，实现LED灯反转 */
    gpio_set_value(dev->led_gpio, sta);

    /* 重启定时器 */
    spin_lock_irqsave(&dev->lock, flags);
    timerperiod = dev->timeperiod;
    spin_unlock_irqrestore(&dev->lock, flags);
    mod_timer(&dev->timer, jiffies + msecs_to_jiffies(dev->timeperiod));
}

/*
 * @description : 驱动入口函数
 * @param : 无
 * @return : 无
 */
static int __init timer_init(void)
{
    /* 初始化自旋锁 */
    spin_lock_init(&timerdev.lock);

    /* 注册字符设备驱动 */
    /* 1、创建设备号 */
    if (timerdev.major) { /* 定义了设备号 */
        timerdev.devid = MKDEV(timerdev.major, 0);
        register_chrdev_region(timerdev.devid, TIMER_CNT, TIMER_NAME);
    } else { /* 没有定义设备号 */
        alloc_chrdev_region(&timerdev.devid, 0, TIMER_CNT, TIMER_NAME);
        timerdev.major = MAJOR(timerdev.devid); /* 获取主设备号 */
        timerdev.minor = MINOR(timerdev.devid); /* 获取次设备号 */
    }

    /* 2、初始化cdev */
    timerdev.cdev.owner = THIS_MODULE;
    cdev_init(&timerdev.cdev, &timer_fops);

    /* 3、添加一个cdev */
    cdev_add(&timerdev.cdev, timerdev.devid, TIMER_CNT);

    /* 4、创建类 */
    timerdev.class = class_create(THIS_MODULE, TIMER_NAME);
    if (IS_ERR(timerdev.class)) {
        return PTR_ERR(timerdev.class);
    }

    /* 5、创建设备 */
    timerdev.device = device_create(timerdev.class, NULL, timerdev.devid, NULL, TIMER_NAME);
    if (IS_ERR(timerdev.device)) {
        return PTR_ERR(timerdev.device);
    }

    /* 6、初始化timer，设置定时器处理函数,还未设置周期，所有不会激活定时器 */
    init_timer(&timerdev.timer);
    timerdev.timer.function = timer_function;
    timerdev.timer.data = (unsigned long)&timerdev;
    return 0;
}

/*
 * @description : 驱动出口函数
 * @param : 无
 * @return : 无
 */
static void __exit timer_exit(void)
{

    gpio_set_value(timerdev.led_gpio, 1); /* 卸载驱动的时候关闭LED */
    del_timer_sync(&timerdev.timer); /* 删除timer */
    #if 0
    del_timer(&timerdev.timer);
    #endif

    /* 注销字符设备驱动 */
    cdev_del(&timerdev.cdev); /* 删除cdev */
    unregister_chrdev_region(timerdev.devid, timerdev.devid, TIMER_CNT);

    device_destroy(timerdev.class, timerdev.devid);
    class_destroy(timerdev.class);
}

module_init(timer_init);
module_exit(timer_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("tantan");
```

第 38~50 行，定时器设备结构体，在 48 行定义了一个定时器成员变量 timer。
第 60~82 行，LED 灯初始化函数，从设备树中获取 LED 灯信息，然后初始化相应的 IO。
第 91~102 行，函数 `timer_open`，对应应用程序的 open 函数，应用程序调用 open 函数打开 `/dev/timer` 驱动文件的时候此函数就会执行。此函数设置文件私有数据为 `timerdev`，并且初始化定时周期默认为 1 秒，最后调用 `led_init` 函数初始化 LED 所使用的 IO。
第 111~137 行，函数 `timer_unlocked_ioctl`，对应应用程序的 ioctl 函数，应用程序调用 ioctl 函数向驱动发送控制信息，此函数响应并执行。此函数有三个参数：filp，cmd 和 arg，其中 filp 是对应的设备文件，cmd 是应用程序发送过来的命令信息，arg 是应用程序发送过来的参数，在本章例程中 arg 参数表示定时周期。
一共有三种命令 `CLOSE_CMD`，`OPEN_CMD` 和 `SETPERIOD_CMD`，这三个命令分别为关闭定时器、打开定时器、设置定时周期。这三个命令的作用如下：
- `CLOSE_CMD`：关闭定时器命令，调用 `del_timer_sync` 函数关闭定时器。
- `OPEN_CMD`：打开定时器命令，调用 `mod_timer` 函数打开定时器，定时周期为 `timerdev` 的 `timeperiod` 成员变量，定时周期默认是 1 秒。
- `SETPERIOD_CMD`：设置定时器周期命令，参数 arg 就是新的定时周期，设置 `timerdev` 的 `timeperiod` 成员变量为 arg 所表示定时周期。并且使用 `mod_timer` 重新打开定时器，使定时器以新的周期运行。

第 140~144 行，定时器驱动操作函数集 `timer_fops`。
第 147~162 行，函数 `timer_function`，定时器服务函数，此函数有一个参数 arg，在本例程中 arg 参数就是 `timerdev` 的地址，这样通过 arg 参数就可以访问到设备结构体。当定时周期到了以后此函数就会被调用。在此函数中将 LED 灯的状态取反，实现 LED 灯闪烁的效果。因为内核定时器不是循环的定时器，执行一次以后就结束了，因此在 161 行又调用了 `mod_timer` 函数重新开启定时器。
第 169~209 行，函数 `timer_init`，驱动入口函数。在第 205~207 行初始化定时器，设置定时器的定时处理函数为 `timer_function`，另外设置要传递给 `timer_function` 函数的参数为 `timerdev` 的地址。在此函数中并没有调用 `add_timer` 函数来开启定时器，因此定时器默认是关闭的，除非应用程序发送打开命令。
第 216~231 行，驱动出口函数，在 219 行关闭 LED，也就是卸载驱动以后 LED 处于熄灭状态。第 220 行调用 `del_timer_sync` 函数删除定时器，也可以使用 `del_timer` 函数。

---

#### 50.3.3 编写测试 APP

测试 APP 我们要实现的内容如下：
①、运行 APP 以后提示我们输入要测试的命令，输入 1 表示关闭定时器、输入 2 表示打开定时器，输入 3 设置定时器周期。
②、如果要设置定时器周期的话，需要让用户输入要设置的周期值，单位为毫秒。

新建名为 `timerApp.c` 的文件，然后输入如下所示内容：

**示例代码 50.3.2.2 timerApp.c 文件**

```c
#include "stdio.h"
#include "unistd.h"
#include "sys/types.h"
#include "sys/stat.h"
#include "fcntl.h"
#include "stdlib.h"
#include "string.h"
#include "linux/ioctl.h"
/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : timerApp.c
作者 : 左忠凯
版本 : V1.0
描述 : 定时器测试应用程序
其他 : 无
使用方法 ：./timerApp /dev/timer 打开测试App
论坛 : www.openedv.com
日志 : 初版V1.0 2019/7/24 左忠凯创建
***************************************************************/

/* 命令值 */
#define CLOSE_CMD (_IO(0XEF, 0x1)) /* 关闭定时器 */
#define OPEN_CMD (_IO(0XEF, 0x2)) /* 打开定时器 */
#define SETPERIOD_CMD (_IO(0XEF, 0x3)) /* 设置定时器周期命令 */

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
    unsigned int cmd;
    unsigned int arg;
    unsigned char str[100];

    if (argc != 2) {
        printf("Error Usage!\r\n");
        return -1;
    }

    filename = argv[1];

    fd = open(filename, O_RDWR);
    if (fd < 0) {
        printf("Can't open file %s\r\n", filename);
        return -1;
    }

    while (1) {
        printf("Input CMD:");
        ret = scanf("%d", &cmd);
        if (ret != 1) { /* 参数输入错误 */
            gets(str); /* 防止卡死 */
        }

        if(cmd == 1) /* 关闭定时器 */
            cmd = CLOSE_CMD;
        else if(cmd == 2) /* 打开定时器 */
            cmd = OPEN_CMD;
        else if(cmd == 3) {
            cmd = SETPERIOD_CMD; /* 设置周期值 */
            printf("Input Timer Period:");
            ret = scanf("%d", &arg);
            if (ret != 1) { /* 参数输入错误 */
                gets(str); /* 防止卡死 */
            }
        }
        ioctl(fd, cmd, arg); /* 控制定时器的打开和关闭 */
    }
    close(fd);
}
```

第 22~24 行，命令值。第 53~73 行，`while (1)` 循环，让用户输入要测试的命令，然后通过 ioctl 函数发送给驱动程序。如果是设置定时器周期命令 `SETPERIOD_CMD`，那么 ioctl 函数的 arg 参数就是用户输入的周期值。

---

### 50.4 运行测试

#### 50.4.1 编译驱动程序和测试 APP

##### 1、编译驱动程序

编写 Makefile 文件，本章实验的 Makefile 文件和第四十章实验基本一样，只是将 obj-m 变量的值改为 timer.o，Makefile 内容如下所示：

**示例代码 50.4.1.1 Makefile 文件**

```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)
obj-m := timer.o

build: kernel_modules

kernel_modules:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules
clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

第 4 行，设置 obj-m 变量的值为 timer.o。

输入如下命令编译出驱动模块文件：

```bash
make -j32
```

编译成功以后就会生成一个名为 `timer.ko` 的驱动模块文件。

##### 2、编译测试 APP

输入如下命令编译测试 `timerApp.c` 这个测试程序：

```bash
arm-linux-gnueabihf-gcc timerApp.c -o timerApp
```

编译成功以后就会生成 `timerApp` 这个应用程序。

---

#### 50.4.2 运行测试

将上一小节编译出来的 `timer.ko` 和 `timerApp` 这两个文件拷贝到 `rootfs/lib/modules/4.1.15` 目录中，重启开发板，进入到目录 `lib/modules/4.1.15` 中，输入如下命令加载 `timer.ko` 驱动模块：

```bash
depmod         //第一次加载驱动的时候需要运行此命令
modprobe timer.ko //加载驱动
```

驱动加载成功以后如下命令来测试：

```bash
./timerApp /dev/timer
```

输入上述命令以后终端提示输入命令。输入 "2"，打开定时器，此时 LED 灯就会以默认的 1 秒周期开始闪烁。再输入 "3" 来设置定时周期，根据提示输入要设置的周期值，输入 "500"，表示设置定时器周期值为 500ms，设置好以后 LED 灯就会以 500ms 为间隔，开始闪烁。最后可以通过输入 "1" 来关闭定时器，如果要卸载驱动的话输入如下命令即可：

```bash
rmmod timer.ko
```