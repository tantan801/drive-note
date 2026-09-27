
# 41.1 Linux 下 LED 灯驱动原理

> **概括**：GPIO1_IO03 引脚。Linux 驱动本质同样操作硬件寄存器，但是必须遵循 Linux 驱动框架；开启 MMU 之后不能直接操作物理地址，要做地址映射。

---

## 41.1.1 地址映射 MMU

**MMU**（Memory Manager Unit，内存管理单元）

1. **地址映射**：虚拟地址 ↔ 物理地址映射
2. **内存保护**：设置存储器访问权限、缓冲属性

![内存映射](<photo/LED驱动开发实验/1-1%20内存映射.jpg>)

### 概念区分

| 概念                  | 说明                                                 |
| --------------------- | ---------------------------------------------------- |
| **PA 物理地址** | 硬件寄存器、DDR 真实的物理地址；裸机直接操作物理地址 |
| **VA 虚拟地址** | 开启 MMU 以后，CPU 程序访问的全部都是虚拟地址        |

### I.MX6ULL 例子

- 寄存器 `IOMUXC_SW_MUX_CTL_PAD_GPIO1_IO03` 物理地址：**`0X020E0068`**
- **裸机（关闭 MMU）**：直接读写 `0X020E0068` 物理地址
- **Linux 内核（MMU 开启）**：禁止直接访问物理地址，必须把物理地址映射成内核虚拟地址，使用 `ioremap`

---

### 1）ioremap — 建立地址映射

建立 **物理地址 → 内核虚拟地址** 映射

**头文件**：`arch/arm/include/asm/io.h`

```c
#define ioremap(cookie, size)  __arm_ioremap((cookie), (size), MT_DEVICE)
void __iomem *__arm_ioremap(phys_addr_t phys_addr, size_t size, unsigned int mtype);
```

| 参数                   | 含义                                                |
| ---------------------- | --------------------------------------------------- |
| `phys_addr` (cookie) | 物理起始地址                                        |
| `size`               | 需要映射的字节大小；寄存器 32 位，一般填**4** |
| `mtype`              | 映射类型，驱动寄存器使用**`MT_DEVICE`**     |
| **返回值**       | `void __iomem *`，映射完成后的内核虚拟地址指针    |

> `__iomem`：标记这个指针是 IO 映射后的虚拟地址，编译器做检查，不能当作普通内存指针。

**示例**：映射 GPIO1_IO03 复用寄存器

```c
#define SW_MUX_GPIO1_IO03_BASE  (0X020E0068)  // 物理地址

static void __iomem *SW_MUX_GPIO1_IO03;

// 物理地址 0X020E0068，映射 4 字节（一个 32 位寄存器）
SW_MUX_GPIO1_IO03 = ioremap(SW_MUX_GPIO1_IO03_BASE, 4);
```

---

### 2）iounmap — 取消地址映射

模块卸载的时候，释放 `ioremap` 创建的地址映射。

```c
void iounmap(volatile void __iomem *addr);
```

**示例**：

```c
iounmap(SW_MUX_GPIO1_IO03);
```

---

## 41.1.2 I/O 内存访问函数

### 概念区分

| 类型               | 说明                                 |
| ------------------ | ------------------------------------ |
| **I/O 端口** | 寄存器映射到独立 IO 空间（x86 架构） |
| **I/O 内存** | 寄存器映射到内存地址空间             |

> ARM 架构没有独立 IO 空间，只有 **I/O 内存**。

使用 `ioremap` 把寄存器物理地址映射为虚拟地址后，**不推荐直接指针访问**，内核推荐专用读写函数操作。

### 读操作函数

```c
u8  readb(const volatile void __iomem *addr)  // 8bit 读（1 字节）
u16 readw(const volatile void __iomem *addr)  // 16bit 读（2 字节）
u32 readl(const volatile void __iomem *addr)  // 32bit 读（4 字节）
```

- **参数 `addr`**：映射之后的虚拟地址（`__iomem` 标记 I/O 内存）
- **返回值**：读到的数据

### 写操作函数

```c
void writeb(u8 value,  volatile void __iomem *addr)  // 8bit 写
void writew(u16 value, volatile void __iomem *addr)  // 16bit 写
void writel(u32 value, volatile void __iomem *addr)  // 32bit 写
```

- **`value`**：待写入的值
- **`addr`**：目标虚拟地址

---

# 41.2 硬件原理图分析

![LED原理图](<photo/LED驱动开发实验/2-1%20LED原理图.png>)

### 原理图解读

- **LED0** 连接到 **GPIO_3**，也就是 **GPIO1_IO03**
- **电路**：3V3 → 510Ω 限流电阻 → LED 阳极，LED 阴极接到 GPIO1_IO03 引脚

|   GPIO1_IO03 输出   | LED 状态 | 原理                         |
| :------------------: | :------: | ---------------------------- |
| **低电平 (0)** | ✅ 点亮 | LED 两端产生压差，二极管导通 |
| **高电平 (1)** | ❌ 熄灭 | LED 两端电压基本相等，无电流 |

> **总结**：输出低电平点亮 LED0，高电平熄灭。

---

# 41.3 实验程序编写

> **例程路径**：开发板光盘 → 2、Linux 驱动例程 → `2_led`
>
> **硬件**：GPIO1_IO03；低电平点亮 LED，高电平熄灭。

## 驱动程序完整代码

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>

#define LED_MAJOR   200             /* 主设备号 */
#define LED_NAME    "led"           /* 设备名字 */

#define LEDOFF      0               /* 关灯 */
#define LEDON       1               /* 开灯 */

/* 寄存器物理地址 */
#define CCM_CCGR1_BASE              (0X020C406C)
#define SW_MUX_GPIO1_IO03_BASE      (0X020E0068)
#define SW_PAD_GPIO1_IO03_BASE      (0X020E02F4)
#define GPIO1_DR_BASE               (0X0209C000)
#define GPIO1_GDIR_BASE             (0X0209C004)

/* 映射后的寄存器虚拟地址指针 */
static void __iomem *IMX6U_CCM_CCGR1;
static void __iomem *SW_MUX_GPIO1_IO03;
static void __iomem *SW_PAD_GPIO1_IO03;
static void __iomem *GPIO1_DR;
static void __iomem *GPIO1_GDIR;

/*
 * @description : LED 打开/关闭
 * @param - sta  : LEDON(1) 打开 LED，LEDOFF(0) 关闭 LED
 * @return       : 无
 */
void led_switch(u8 sta)
{
    u32 val = 0;
    if (sta == LEDON) {
        val = readl(GPIO1_DR);
        val &= ~(1 << 3);           /* bit3 清零，输出低电平，点亮 LED */
        writel(val, GPIO1_DR);
    } else if (sta == LEDOFF) {
        val = readl(GPIO1_DR);
        val |= (1 << 3);            /* bit3 置 1，输出高电平，熄灭 LED */
        writel(val, GPIO1_DR);
    }
}

/*
 * @description : 打开设备
 * @param - inode: 传递给驱动的 inode
 * @param - filp : 设备文件，file 结构体有个叫做 private_data 的成员变量
 *                 一般在 open 的时候将 private_data 指向设备结构体。
 * @return       : 0 成功；其他 失败
 */
static int led_open(struct inode *inode, struct file *filp)
{
    return 0;
}

/*
 * @description : 从设备读取数据
 * @param - filp : 要打开的设备文件（文件描述符）
 * @param - buf  : 返回给用户空间的数据缓冲区
 * @param - cnt  : 要读取的数据长度
 * @param - offt : 相对于文件首地址的偏移
 * @return       : 读取的字节数，如果为负值，表示读取失败
 */
static ssize_t led_read(struct file *filp, char __user *buf, size_t cnt, loff_t *offt)
{
    return 0;
}

/*
 * @description : 向设备写数据
 * @param - filp : 设备文件，表示打开的文件描述符
 * @param - buf  : 要写给设备写入的数据
 * @param - cnt  : 要写入的数据长度
 * @param - offt : 相对于文件首地址的偏移
 * @return       : 写入的字节数，如果为负值，表示写入失败
 */
static ssize_t led_write(struct file *filp, const char __user *buf, size_t cnt, loff_t *offt)
{
    int retvalue;
    unsigned char databuf[1];
    unsigned char ledstat;

    retvalue = copy_from_user(databuf, buf, cnt);
    if (retvalue < 0) {
        printk("kernel write failed!\r\n");
        return -EFAULT;
    }

    ledstat = databuf[0];           /* 获取状态值 */

    if (ledstat == LEDON) {
        led_switch(LEDON);          /* 打开 LED 灯 */
    } else if (ledstat == LEDOFF) {
        led_switch(LEDOFF);         /* 关闭 LED 灯 */
    }
    return 0;
}

/*
 * @description : 关闭/释放设备
 * @param - filp : 要关闭的设备文件（文件描述符）
 * @return       : 0 成功；其他 失败
 */
static int led_release(struct inode *inode, struct file *filp)
{
    return 0;
}

/* 设备操作函数 */
static struct file_operations led_fops = {
    .owner   = THIS_MODULE,
    .open    = led_open,
    .read    = led_read,
    .write   = led_write,
    .release = led_release,
};

/*
 * @description : 驱动入口函数
 * @param       : 无
 * @return      : 无
 */
static int __init led_init(void)
{
    int retvalue = 0;
    u32 val = 0;

    /* 1、寄存器地址映射 */
    IMX6U_CCM_CCGR1   = ioremap(CCM_CCGR1_BASE, 4);
    SW_MUX_GPIO1_IO03 = ioremap(SW_MUX_GPIO1_IO03_BASE, 4);
    SW_PAD_GPIO1_IO03 = ioremap(SW_PAD_GPIO1_IO03_BASE, 4);
    GPIO1_DR          = ioremap(GPIO1_DR_BASE, 4);
    GPIO1_GDIR        = ioremap(GPIO1_GDIR_BASE, 4);

    /* 2、使能 GPIO1 时钟 */
    val = readl(IMX6U_CCM_CCGR1);
    val &= ~(3 << 26);              /* 清除以前的设置 */
    val |= (3 << 26);               /* 设置新值 */
    writel(val, IMX6U_CCM_CCGR1);

    /* 3、设置 GPIO1_IO03 的复用功能，将其复用为 GPIO1_IO03，设置 IO 属性 */
    writel(5, SW_MUX_GPIO1_IO03);
    writel(0x10B0, SW_PAD_GPIO1_IO03);

    /* 4、设置 GPIO1_IO03 为输出功能 */
    val = readl(GPIO1_GDIR);
    val &= ~(1 << 3);               /* 清除 bit3 */
    val |= (1 << 3);                /* bit3 置 1，设置为输出 */
    writel(val, GPIO1_GDIR);

    /* 5、默认关闭 LED */
    val = readl(GPIO1_DR);
    val |= (1 << 3);
    writel(val, GPIO1_DR);

    /* 6、注册字符设备驱动 */
    retvalue = register_chrdev(LED_MAJOR, LED_NAME, &led_fops);
    if (retvalue < 0) {
        printk("register chrdev failed!\r\n");
        return -EIO;
    }
    return 0;
}

/*
 * @description : 驱动出口函数
 * @param       : 无
 * @return      : 无
 */
static void __exit led_exit(void)
{
    /* 取消映射 */
    iounmap(IMX6U_CCM_CCGR1);
    iounmap(SW_MUX_GPIO1_IO03);
    iounmap(SW_PAD_GPIO1_IO03);
    iounmap(GPIO1_DR);
    iounmap(GPIO1_GDIR);

    /* 注销字符设备驱动 */
    unregister_chrdev(LED_MAJOR, LED_NAME);
}

module_init(led_init);
module_exit(led_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("zuozhongkai");
```

---

## 代码解析

### 1. 宏定义

| 宏                         |       值       | 说明                 |
| -------------------------- | :------------: | -------------------- |
| `LED_MAJOR`              |      200      | 静态指定主设备号     |
| `LED_NAME`               |   `"led"`   | 设备名               |
| `LEDON`                  |       1       | 点亮 LED             |
| `LEDOFF`                 |       0       | 熄灭 LED             |
| `CCM_CCGR1_BASE`         | `0X020C406C` | CCM 时钟寄存器       |
| `SW_MUX_GPIO1_IO03_BASE` | `0X020E0068` | IO 复用寄存器        |
| `SW_PAD_GPIO1_IO03_BASE` | `0X020E02F4` | IO 电气属性寄存器    |
| `GPIO1_DR_BASE`          | `0X0209C000` | GPIO 数据寄存器 DR   |
| `GPIO1_GDIR_BASE`        | `0X0209C004` | GPIO 方向寄存器 GDIR |

### 2. 虚拟地址指针

`void __iomem *` 类型指针，保存 `ioremap` 映射之后得到的内核虚拟 IO 地址。

### 3. `led_switch()` 函数

操作 `GPIO1_DR` 数据寄存器 bit3：

| 参数              | 操作      | 效果                      |
| ----------------- | --------- | ------------------------- |
| `sta == LEDON`  | bit3 清 0 | 输出低电平 → ✅ 点亮 LED |
| `sta == LEDOFF` | bit3 置 1 | 输出高电平 → ❌ 熄灭 LED |

> 使用 `readl()` 读寄存器，`writel()` 写寄存器。

### 4. `led_open` / `led_read` / `led_release`

- **`led_open`**：打开设备，空实现
- **`led_read`**：本实验不需要读 LED 状态，空实现
- **`led_release`**：关闭设备，空实现

### 5. `led_write()` — 核心函数

应用层调用 `write()` 系统调用，触发该驱动函数：

1. `copy_from_user(databuf, buf, cnt)` — 把用户空间传入的操作命令拷贝到内核缓冲区
2. 根据读到的值 `ledstat`，调用 `led_switch()` 做亮灯 / 灭灯

### 6. 驱动入口 `led_init()` — 模块加载（`insmod`）执行

执行顺序：

| 步骤 | 操作                                                              |
| :--: | ----------------------------------------------------------------- |
|  ①  | `ioremap()`：5 个寄存器物理地址映射为内核虚拟地址               |
|  ②  | 配置`CCM_CCGR1`，使能 GPIO1 外设时钟                            |
|  ③  | 设置 IO 复用寄存器`SW_MUX_GPIO1_IO03`，把引脚复用为 GPIO 功能   |
|  ④  | 设置 PAD 电气属性寄存器`SW_PAD_GPIO1_IO03`                      |
|  ⑤  | `GPIO1_GDIR` 寄存器 bit3 置 1，设置 GPIO1_IO03 为输出方向       |
|  ⑥  | 默认熄灭 LED 灯                                                   |
|  ⑦  | `register_chrdev(LED_MAJOR, LED_NAME, &led_fops)`：注册字符设备 |

### 7. 驱动出口 `led_exit()` — 模块卸载（`rmmod`）执行

| 步骤 | 操作                                                         |
| :--: | ------------------------------------------------------------ |
|  ①  | `iounmap()`：取消所有 `ioremap` 建立的地址映射，释放资源 |
|  ②  | `unregister_chrdev()`：注销字符设备                        |

---

## 41.3.2 编写测试 APP

向 `/dev/led` 文件写 **0** 表示关闭 LED 灯，写 **1** 表示打开 LED 灯。

**使用规则**：

```bash
./ledApp /dev/led 0    # 关闭 LED
./ledApp /dev/led 1    # 打开 LED
```

### ledApp.c

```c
#include "stdio.h"
#include "unistd.h"
#include "sys/types.h"
#include "sys/stat.h"
#include "fcntl.h"
#include "stdlib.h"
#include "string.h"

#define LEDOFF  0
#define LEDON   1

/*
 * @description : main 主程序
 * @param - argc: argv 数组元素个数
 * @param - argv: 具体参数
 * @return      : 0 成功；其他 失败
 */
int main(int argc, char *argv[])
{
    int fd, retvalue;
    char *filename;
    unsigned char databuf[1];

    if (argc != 3) {
        printf("Error Usage!\r\n");
        return -1;
    }

    filename = argv[1];

    /* 打开 led 驱动设备文件 */
    fd = open(filename, O_RDWR);
    if (fd < 0) {
        printf("file %s open failed!\r\n", argv[1]);
        return -1;
    }

    databuf[0] = atoi(argv[2]);     /* 要执行的操作：打开或关闭 */

    /* 向 /dev/led 文件写入数据 */
    retvalue = write(fd, databuf, sizeof(databuf));
    if (retvalue < 0) {
        printf("LED Control Failed!\r\n");
        close(fd);
        return -1;
    }

    retvalue = close(fd);           /* 关闭文件 */
    if (retvalue < 0) {
        printf("file %s close failed!\r\n", argv[1]);
        return -1;
    }
    return 0;
}
```

### 代码解析

|              步骤              | 说明                                                                           |
| :----------------------------: | ------------------------------------------------------------------------------ |
|       **参数校验**       | `argc != 3`：需要 3 个参数 `/dev/led` 和 `1`/`0`，不够则打印错误并退出 |
|   `open(filename, O_RDWR)`   | 以读写模式打开字符设备节点`/dev/led`，返回文件描述符 `fd`                  |
| `databuf[0] = atoi(argv[2])` | 把传入参数字符串转为数字，1 = 开 LED，0 = 关 LED                               |
|  `write(fd, databuf, ...)`  | 调用用户层`write` 系统调用，内核自动调用驱动的 `led_write` 函数            |
|         `close(fd)`         | 操作完成关闭设备文件描述符                                                     |

---

# 41.4 运行测试

## 41.4.1 编译驱动程序和测试 APP

### 1）编译驱动程序

编写 Makefile 文件：

```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
# ......
obj-m := led.o
# ......
clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

> 第 4 行，设置 `obj-m` 变量的值为 `led.o`。

编译命令：

```bash
make -j32
```

编译成功以后就会生成一个名为 **`led.ko`** 的驱动模块文件。

### 2）编译测试 APP

```bash
arm-linux-gnueabihf-gcc ledApp.c -o ledApp
```

编译成功以后就会生成 **`ledApp`** 这个应用程序。

---

## 41.4.2 运行测试

### 关闭出厂系统灯（可选）

```bash
echo none > /sys/class/leds/sys-led/trigger    # 改变 LED 的触发模式
```

### 加载驱动

将 `led.ko` 和 `ledApp` 拷贝到 `rootfs/lib/modules/4.1.15` 目录，重启开发板，进入该目录执行：

```bash
depmod                  # 第一次加载驱动的时候需要运行此命令
modprobe led.ko         # 加载驱动
```

### 创建设备节点

```bash
mknod /dev/led c 200 0
```

### 测试 LED 控制

```bash
./ledApp /dev/led 1     # 打开 LED 灯（开发板红色 LED 点亮）
./ledApp /dev/led 0     # 关闭 LED 灯（开发板红色 LED 熄灭）
```

> 如果 LED 按预期亮灭，说明驱动工作完全正常！至此，成功编写了第一个真正的 Linux 驱动设备程序。

### 卸载驱动

```bash
rmmod led.ko
```

---

> **例程路径**：开发板光盘 → 2、Linux 驱动例程 → `2_led`
> **作者**：zuozhongkai | 正点原子
