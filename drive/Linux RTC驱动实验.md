## 第六十章 Linux RTC 驱动实验
RTC 即实时时钟，用来记录系统当前时间。Linux 系统中时间管理十分重要，本章学习 Linux RTC 驱动编写。

---

### 60.1 Linux 内核 RTC 驱动简介
RTC 属于标准字符设备驱动，应用层通过open/release/read/write/ioctl操作 RTC 硬件。RTC 硬件原理在裸机篇第 25 章已经讲解。
*回顾硬件：

![图1-1 外接晶振](./photo/Linux%20RTC驱动实验/1-1%20外接晶振.png)

内核使用rtc_device结构体抽象 RTC 设备，RTC 驱动核心工作：申请、初始化 rtc_device，然后注册到内核。RTC 底层硬件操作由rtc_class_ops函数集合实现。

**示例代码 60.1.1 rtc_device 结构体（include/linux/rtc.h）**
```c
struct rtc_device
{
    struct device dev;                 /* 基础设备 */
    struct module *owner;
    int id;                            /* RTC设备ID */
    char name[RTC_DEVICE_NAME_SIZE];   /* RTC设备名称 */
    const struct rtc_class_ops *ops;   /* RTC底层硬件操作函数集【重点】 */
    struct mutex ops_lock;

    struct cdev char_dev;             /* 内嵌字符设备 */
    unsigned long flags;

    unsigned long irq_data;
    spinlock_t irq_lock;
    wait_queue_head_t irq_queue;
    struct fasync_struct *async_queue;

    struct rtc_task *irq_task;
    spinlock_t irq_task_lock;
    int irq_freq;
    int max_user_freq;

    struct timerqueue_head timerqueue;
    struct rtc_timer aie_timer;
    struct rtc_timer uie_rtctimer;
    struct hrtimer pie_timer; /* sub second exp, so needs hrtimer */
    int pie_enabled;
    struct work_struct irqwork;
    /* Some hardware can't support UIE mode */
    int uie_unsupported;
    // 省略部分成员
};
```

重点：ops成员，指向rtc_class_ops，存放底层读写时间、闹钟的硬件操作函数，需要驱动开发者实现。

**示例代码 60.1.2 rtc_class_ops 结构体（include/linux/rtc.h）**
```c
struct rtc_class_ops {
    int (*open)(struct device *);
    void (*release)(struct device *);
    int (*ioctl)(struct device *, unsigned int, unsigned long);
    int (*read_time)(struct device *, struct rtc_time *);   /* 读取RTC时间 */
    int (*set_time)(struct device *, struct rtc_time *);    /* 设置RTC时间 */
    int (*read_alarm)(struct device *, struct rtc_wkalrm *);/* 读取闹钟 */
    int (*set_alarm)(struct device *, struct rtc_wkalrm *); /* 设置闹钟 */
    int (*proc)(struct device *, struct seq_file *);
    int (*set_mmss64)(struct device *, time64_t secs);
    int (*set_mmss)(struct device *, unsigned long secs);
    int (*read_callback)(struct device *, int data);
    int (*alarm_irq_enable)(struct device *, unsigned int enabled);
};
```

注意：rtc_class_ops是底层硬件操作接口，不是应用层的file_operations！ 内核通用 RTC 驱动rtc-dev.c已经帮我们实现好字符设备的 file_operations。

**示例代码 60.1.3 RTC 通用 file_operations 操作集 drivers/rtc/rtc-dev.c**
```c
static const struct file_operations rtc_dev_fops = {
    .owner = THIS_MODULE,
    .llseek = no_llseek,
    .read = rtc_dev_read,
    .poll = rtc_dev_poll,
    .unlocked_ioctl = rtc_dev_ioctl,
    .open = rtc_dev_open,
    .release = rtc_dev_release,
    .fasync = rtc_dev_fasync,
};
```

内核提供通用字符设备操作集，应用层调用 ioctl，最终会回调到我们驱动实现的rtc_class_ops里面的read_time/set_time等函数。

**示例代码 60.1.4 rtc_dev_ioctl 函数代码段**
```c
static long rtc_dev_ioctl(struct file *file,
unsigned int cmd, unsigned long arg)
{
    int err = 0;
    struct rtc_device *rtc = file->private_data;
    const struct rtc_class_ops *ops = rtc->ops;
    struct rtc_time tm;
    struct rtc_wkalrm alarm;
    void __user *uarg = (void __user *) arg;

    err = mutex_lock_interruptible(&rtc->ops_lock);
    if (err)
        return err;

    switch (cmd) {
        case RTC_RD_TIME: /* 读取时间命令 */
            mutex_unlock(&rtc->ops_lock);
            err = rtc_read_time(rtc, &tm);
            if (err < 0)
                return err;
            if (copy_to_user(uarg, &tm, sizeof(tm)))
                err = -EFAULT;
            return err;

        case RTC_SET_TIME: /* 设置时间命令 */
            mutex_unlock(&rtc->ops_lock);
            if (copy_from_user(&tm, uarg, sizeof(tm)))
                return -EFAULT;
            return rtc_set_time(rtc, &tm);

        default:
            /* Finally try the driver's ioctl interface */
            if (ops->ioctl) {
                err = ops->ioctl(rtc->dev.parent, cmd, arg);
                if (err == -ENOIOCTLCMD)
                    err = -ENOTTY;
            } else
                err = -ENOTTY;
            break;
    }
done:
    mutex_unlock(&rtc->ops_lock);
    return err;
}
```
- RTC_RD_TIME：读取 RTC 时间，调用rtc_read_time
- RTC_SET_TIME：设置 RTC 时间，调用rtc_set_time

**示例代码 60.1.5 __rtc_read_time 函数代码段**
```c
static int __rtc_read_time(struct rtc_device *rtc, struct rtc_time *tm)
{
    int err;
    if (!rtc->ops)
        err = -ENODEV;
    else if (!rtc->ops->read_time)
        err = -EINVAL;
    else {
        memset(tm, 0, sizeof(struct rtc_time));
        /* 调用驱动自定义的read_time，读取硬件RTC时间【核心回调】 */
        err = rtc->ops->read_time(rtc->dev.parent, tm);
        if (err < 0) {
            dev_dbg(&rtc->dev, "read_time: fail to read: %d\n",err);
            return err;
        }
        err = rtc_valid_tm(tm); /* 校验读取到的时间合法性 */
        if (err < 0)
            dev_dbg(&rtc->dev, "read_time: rtc_time isn't valid\n");
    }
    return err;
}
```

核心逻辑：内核通用 RTC 框架，最终调用rtc->ops->read_time，也就是我们自己写的底层硬件读取函数。
因此 RTC驱动调用流程就很清晰了

![图1-2 RTC驱动调用流程](./photo/Linux%20RTC驱动实验/1-2%20RTC驱动调用流程.png)

---

**RTC 设备注册 / 注销 API**

**1）rtc_device_register：手动注册 rtc_device，内核分配 rtc_device**
```c
struct rtc_device *rtc_device_register(const char *name, struct device *dev,
                                      const struct rtc_class_ops *ops,
                                      struct module *owner);
```

*参数：
- name：RTC 设备名称
- dev：父设备指针（一般是 platform 设备的 device）
- ops：rtc_class_ops 底层操作函数集
- owner：模块所有者，填 THIS_MODULE 

返回值：成功返回 rtc_device 指针；失败返回错误指针。

**2）rtc_device_unregister：注销 rtc_device**
```c
void rtc_device_unregister(struct rtc_device *rtc);
```

**3）资源托管版本（推荐 devm_）**
devm_rtc_device_register / devm_rtc_device_unregister 
devm 版本：设备卸载时自动释放资源，不用手动释放，减少内存泄漏。

---

### 60.2 I.MX6U内部RTC驱动分析
1. I.MX6U 内部 RTC 驱动 NXP 原厂已完成，无需自行编写；学习目标是看懂原厂驱动实现思路
2. 设备树节点 imx6ull.dtsi 中 snvs_rtc：
```dts
snvs_rtc: snvs-rtc-lp {
    compatible = "fsl,sec-v4.0-mon-rtc-lp";
    regmap = <&snvs>;
    offset = <0x34>;
    interrupts = <GIC_SPI 19 IRQ_TYPE_LEVEL_HIGH>, <GIC_SPI 20 IRQ_TYPE_LEVEL_HIGH>;
};
```
3. 根据 compatible 字符串匹配驱动文件：drivers/rtc/rtc-snvs.c
4. platform 驱动框架：
```c
//设备匹配表
static const struct of_device_id snvs_dt_ids[] = {
    { .compatible = "fsl,sec-v4.0-mon-rtc-lp", },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, snvs_dt_ids);

static struct platform_driver snvs_rtc_driver = {
    .driver = {
        .name = "snvs_rtc",
        .pm = SNVS_RTC_PM_OPS,
        .of_match_table = snvs_dt_ids,
    },
    .probe = snvs_rtc_probe,
};
module_platform_driver(snvs_rtc_driver);
```

5. snvs_rtc_probe 函数流程：
- devm_kzalloc：分配驱动私有数据结构体
- syscon_regmap_lookup_by_phandle：获取 regmap，操作硬件寄存器
- platform_get_resource + devm_ioremap_resource：获取寄存器资源并做内存映射
- devm_regmap_init_mmio：初始化 regmap
- platform_get_irq：获取 RTC 中断号
- regmap_write：初始化 LPPGDR、清除 LPSR 中断状态寄存器
- snvs_rtc_enable：使能 RTC 外设
- devm_request_irq：申请闹钟中断，中断处理函数 snvs_rtc_irq_handler
- devm_rtc_device_register：注册 rtc 设备，绑定 rtc_class_ops 操作集

6. RTC 操作集 snvs_rtc_ops：
```c
static const struct rtc_class_ops snvs_rtc_ops = {
    .read_time = snvs_rtc_read_time,    //读取RTC时间
    .set_time = snvs_rtc_set_time,      //设置RTC时间
    .read_alarm = snvs_rtc_read_alarm,  //读取闹钟
    .set_alarm = snvs_rtc_set_alarm,    //设置闹钟
    .alarm_irq_enable = snvs_rtc_alarm_irq_enable, //闹钟中断使能
};
```

7. snvs_rtc_read_time：
```c
static int snvs_rtc_read_time(struct device *dev, struct rtc_time *tm)
{
    struct snvs_rtc_data *data = dev_get_drvdata(dev);
    unsigned long time = rtc_read_lp_counter(data); //读取RTC秒计数
    rtc_time_to_tm(time, tm); //秒数转为rtc_time结构体
    return 0;
}
```

rtc_time 结构体：存储时分秒、年月日、星期等时间信息
```c
struct rtc_time {
    int tm_sec;
    int tm_min;
    int tm_hour;
    int tm_mday;
    int tm_mon;
    int tm_year;
    int tm_wday;
    int tm_yday;
    int tm_isdst;
};
```

8. rtc_read_lp_counter:读取RTC 47位计数器
- 读取 SNVS_LPSRTCMR、SNVS_LPSRTCLR 两个寄存器拼接计数值
- 循环两次读取，对比校验，防止跨寄存器读取时时间跳变导致数据错误
- 将 47 位计数值转换为 32bit 秒数返回

---

### 60.3 RTC时间查看与设置
1. 内核启动自动识别 snvs_rtc，生成设备 /dev/rtc0
2. 查看系统时间
```shell
date
```
3. 设置系统时间（只修改系统软件时钟，不写入硬件RTC）
```shell
date -s "2019-08-31 18:13:00"
```
4. 将系统时间写入硬件RTC
```shell
hwclock -w
```

开发板底板接上纽扣电池，断电后 RTC 继续计时，重启时间不会丢失。