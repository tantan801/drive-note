## 46.1 蜂鸣器驱动原理

> **核心结论**：蜂鸣器驱动和 LED 驱动框架**完全一致**，都是 GPIO 输出控制，仅需修改引脚、GPIO 属性名、电平逻辑，直接复用 pinctrl + GPIO 子系统的标准流程。

### 开发流程 3 步

1. **设备树 PIN 配置**：在 `&iomuxc` 的 `imx6ul-evk` 子节点内，新增蜂鸣器的 `pinctrl_beep` 子节点，把 `SNVS_TAMPER1` 复用为 GPIO 功能，设置电气属性。
2. **设备树外设节点**：在根节点 `/` 下，创建 `beep` 设备节点，引用 pinctrl 配置，添加 `beep-gpio` 属性指定 `SNVS_TAMPER1`。
3. **驱动代码修改**：直接复制上一章 `gpioled.c`，仅修改 GPIO 编号、属性名、电平反转逻辑；测试 APP 复用 LED 的框架。

---

## 46.2 硬件原理图分析

蜂鸣器控制电路：
- **Q1**：S8550 PNP 三极管
- **R18**：1KΩ 基极限流电阻
- **控制引脚**：`SNVS_TAMPER1`（复用后为 `GPIO5_IO01`）

### 电平控制逻辑

| 引脚电平（GPIO5_IO01） | PNP 三极管状态 | 蜂鸣器状态 | 电流通路 |
|----------------------|--------------|----------|----------|
| **低电平（0）** | 导通 | 响 | DCDC_3V3 → Q1 发射极 → 集电极 → 蜂鸣器正极 → 蜂鸣器负极 → GND |
| **高电平（1）** | 截止 | 不响 | 回路断开 |

> **关键提示**：硬件用 PNP 三极管**低电平导通**，和 LED（NPN 高电平点亮）逻辑**相反**，驱动代码必须反转电平！

---

## 46.3 实验程序编写

> 核心：在 `gpioled`（LED 驱动）基础上修改，框架 100% 复用，仅改引脚、GPIO 名字、电平逻辑。
> 
> 例程路径：开发板光盘 → 2、Linux 驱动例程 → `6_beep`

---

### 46.3.1 修改设备树文件

打开板级设备树文件 `imx6ull-alientek-emmc.dts`，按以下步骤修改。

---

#### 1）添加 pinctrl 子节点

在 `&iomuxc` → `imx6ul-evk` 内部，新增 `pinctrl_beep` 子节点：

```dts
pinctrl_beep: beepgrp {
    fsl,pins = <
        MX6ULL_PAD_SNVS_TAMPER1__GPIO5_IO01 0x10B0 /* beep */
    >;
};
```

- **`MX6ULL_PAD_SNVS_TAMPER1__GPIO5_IO01`**：把 `SNVS_TAMPER1` 引脚复用为 `GPIO5_IO01` 功能
  - 宏定义位置：`arch/arm/boot/dts/imx6ull-pinfunc-snvs.h`（注意：SNVS 域的引脚宏在 `snvs` 专用头文件，普通引脚在 `imx6ul-pinfunc.h`）
- **`0x10B0`**：PAD 电气配置参数（上下拉、速度、驱动能力，和 LED 实验配置一致）

---

#### 2）添加 beep 设备节点（根节点 `/` 下）

```dts
beep {
    #address-cells = <1>;
    #size-cells = <1>;
    compatible = "atkalpha-beep";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_beep>;
    beep-gpio = <&gpio5 1 GPIO_ACTIVE_HIGH>;
    status = "okay";
};
```

- **`pinctrl-0 = <&pinctrl_beep>`**：绑定上面写好的 pinctrl 引脚配置
- **`beep-gpio`**：自定义 GPIO 属性名，驱动用 `of_get_named_gpio()` 读取这个属性，拿到 GPIO 编号
  - `<&gpio5 1>`：GPIO5 组下的 IO1（即 `GPIO5_IO01`）
  - **`GPIO_ACTIVE_HIGH`**：⚠️ **只是 DTS 属性标记**，实际硬件是 PNP 三极管**低电平导通**，所以驱动代码里电平要**反转写**！

---

#### 3）引脚冲突检查

1. **搜索 SNVS_TAMPER1**：在 dts 和 dtsi 文件里全局搜索，若被其他 pinctrl 节点占用（如 RTC、加密模块），注释屏蔽。
2. **检查 GPIO5_IO01**：确认没有其他外设使用该 GPIO，冲突同样屏蔽。
3. **编译验证**：
   ```bash
   make dtbs    # 编译设备树，生成 .dtb
   ```
4. **启动验证**：用新 dtb 启动开发板，进入 `/proc/device-tree/`，若存在 `beep/` 文件夹 → 设备树节点编译成功。

---

### 46.3.2 蜂鸣器驱动程序

直接复制 `gpioled.c` 为 `beep.c`，仅修改以下内容：

---

#### 1）宏定义与设备结构体

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/of_gpio.h>
#include <linux/gpio.h>

/* ========= 宏定义（和 LED 驱动区分） ========= */
#define BEEP_CNT    1    /* 设备数量 */
#define BEEP_NAME   "beep"
#define BEEPOFF     0    /* 应用层传 0 → 关蜂鸣器 */
#define BEEPON      1    /* 应用层传 1 → 开蜂鸣器 */

/* ========= 设备结构体（和 LED 结构体几乎完全一致） ========= */
struct beep_dev {
    dev_t devid;              /* 设备号 */
    struct cdev cdev;         /* cdev */
    struct class *class;      /* 类 */
    struct device *device;    /* 设备 */
    int major;                /* 主设备号 */
    int minor;                /* 次设备号 */
    struct device_node *nd;   /* 设备树节点 */
    int beep_gpio;            /* 保存从设备树读到的 GPIO 编号（核心新增） */
};

struct beep_dev beep;    /* 全局设备实例 */
```

---

#### 2）beep_write 函数（**核心修改：电平反转**）

```c
static ssize_t beep_write(struct file *filp, const char __user *buf,
                          size_t cnt, loff_t *offt)
{
    int retvalue;
    unsigned char databuf[1];
    unsigned char beepstat;
    struct beep_dev *dev = filp->private_data;

    /* 用户态→内核态拷贝 */
    retvalue = copy_from_user(databuf, buf, cnt);
    if (retvalue < 0) {
        printk("kernel write failed!\r\n");
        return -EFAULT;
    }

    beepstat = databuf[0];
    if (beepstat == BEEPON) {
        /* 应用层传 1 → 驱动输出 0（低电平）→ PNP 导通 → 蜂鸣器响 */
        gpio_set_value(dev->beep_gpio, 0);
    } else if (beepstat == BEEPOFF) {
        /* 应用层传 0 → 驱动输出 1（高电平）→ PNP 截止 → 蜂鸣器关 */
        gpio_set_value(dev->beep_gpio, 1);
    }
    return 0;
}
```

> **关键差异**：应用层和硬件电平**完全反转**，这是 PNP 三极管硬件决定的！
> - LED（NPN）：应用传 1 → 驱动输出 1 → 亮
> - 蜂鸣器（PNP）：应用传 1 → 驱动输出 0 → 响

---

#### 3）驱动入口 beep_init

```c
static int __init beep_init(void)
{
    int ret = 0;

    /* 1. 找到设备树里 /beep 节点 */
    beep.nd = of_find_node_by_path("/beep");
    if (beep.nd == NULL) {
        printk("beep node can not found!\r\n");
        return -EINVAL;
    }
    printk("beep node has been found!\r\n");

    /* 2. 读取设备树中 "beep-gpio" 属性，获取 GPIO 编号 */
    beep.beep_gpio = of_get_named_gpio(beep.nd, "beep-gpio", 0);
    if (beep.beep_gpio < 0) {
        printk("can't get beep-gpio\r\n");
        return -EINVAL;
    }
    printk("beep-gpio num = %d\r\n", beep.beep_gpio);

    /* 3. 申请 GPIO 引脚（和 LED 驱动不同，必须显式申请） */
    ret = gpio_request(beep.beep_gpio, "beep-gpio");
    if (ret < 0) {
        printk("beep gpio request failed!\r\n");
        return -EINVAL;
    }

    /* 4. 设置 GPIO 为输出，**默认输出高电平**（防止开机蜂鸣器乱叫！） */
    ret = gpio_direction_output(beep.beep_gpio, 1);
    if (ret < 0) {
        printk("can't set gpio direction!\r\n");
        goto err_gpio_free;
    }

    /* ========= 字符设备注册流程（和 LED 驱动完全一致） ========= */
    /* 自动分配设备号 */
    ret = alloc_chrdev_region(&beep.devid, 0, BEEP_CNT, BEEP_NAME);
    if (ret < 0) {
        printk("alloc chrdev region failed!\r\n");
        goto err_gpio_free;
    }
    beep.major = MAJOR(beep.devid);
    beep.minor = MINOR(beep.devid);
    printk("beep major=%d, minor=%d\r\n", beep.major, beep.minor);

    /* 初始化并添加 cdev */
    cdev_init(&beep.cdev, &beep_fops);
    beep.cdev.owner = THIS_MODULE;
    ret = cdev_add(&beep.cdev, beep.devid, BEEP_CNT);
    if (ret < 0) {
        printk("cdev add failed!\r\n");
        goto err_unregister_chrdev;
    }

    /* 创建类 */
    beep.class = class_create(THIS_MODULE, BEEP_NAME);
    if (IS_ERR(beep.class)) {
        printk("class create failed!\r\n");
        goto err_cdev_del;
    }

    /* 自动创建设备节点 /dev/beep */
    beep.device = device_create(beep.class, NULL, beep.devid, NULL, BEEP_NAME);
    if (IS_ERR(beep.device)) {
        printk("device create failed!\r\n");
        goto err_class_destroy;
    }

    return 0;

/* ========= 错误处理跳转标签 ========= */
err_class_destroy:
    class_destroy(beep.class);
err_cdev_del:
    cdev_del(&beep.cdev);
err_unregister_chrdev:
    unregister_chrdev_region(beep.devid, BEEP_CNT);
err_gpio_free:
    gpio_free(beep.beep_gpio);
    return ret;
}
```

> **关键细节**：
> 1. `gpio_request()`：显式申请 GPIO（部分内核版本的 pinctrl 驱动会自动申请，但**主动申请更规范**，防止引脚被占用）
> 2. `gpio_direction_output(gpio, 1)`：默认输出**高电平**（PNP 截止，蜂鸣器开机不响，符合用户体验）
> 3. 新增**错误处理跳转链**（`goto err_xxx`）：驱动开发规范，防止中间步骤失败导致资源泄漏

---

#### 4）驱动出口 beep_exit

```c
static void __exit beep_exit(void)
{
    /* 释放 GPIO 资源（必须调用，否则其他驱动无法使用该引脚） */
    gpio_free(beep.beep_gpio);

    /* 销毁字符设备相关资源（和 LED 驱动顺序一致：先销毁设备，再类，再 cdev） */
    device_destroy(beep.class, beep.devid);
    class_destroy(beep.class);
    cdev_del(&beep.cdev);
    unregister_chrdev_region(beep.devid, BEEP_CNT);

    printk("beep driver exited!\r\n");
}

/* 模块加载/卸载入口 */
module_init(beep_init);
module_exit(beep_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("zuozhongkai");
MODULE_DESCRIPTION("I.MX6ULL BEEP Driver based on pinctrl+GPIO subsystem");
```

---

### 46.3.3 编写测试 app

基于上一章的 `ledApp.c` 改写为 `beepApp.c`，内容和框架完全一致：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdlib.h>
#include <string.h>

#define BEEPOFF  0   /* 关闭蜂鸣器 */
#define BEEPON   1   /* 打开蜂鸣器 */

int main(int argc, char *argv[])
{
    int fd, retvalue;
    char *filename;
    unsigned char databuf[1];

    /* 参数校验：程序名 + 设备节点 + 控制值 = 3 个参数 */
    if (argc != 3) {
        printf("Error Usage!\r\n");
        return -1;
    }

    filename = argv[1];

    /* 打开驱动生成的设备文件 /dev/beep，可读可写 */
    fd = open(filename, O_RDWR);
    if (fd < 0) {
        printf("file %s open failed!\r\n", argv[1]);
        return -1;
    }

    /* 把传入的字符串参数转成数字：0 关 / 1 开 */
    databuf[0] = atoi(argv[2]);

    /* write 系统调用：把 1/0 传给内核驱动的 beep_write 函数 */
    retvalue = write(fd, databuf, sizeof(databuf));
    if (retvalue < 0) {
        printf("BEEP Control Failed!\r\n");
        close(fd);
        return -1;
    }

    close(fd);
    return 0;
}
```

---

## 46.4 运行测试

### 46.4.1 编译驱动与测试 app

---

#### 1）驱动 Makefile

```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)

obj-m := beep.o    # 重点：改成 beep.o，和 LED 实验的 led.o 区分

build:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

- 编译命令：`make -j32`
- 产物：**`beep.ko`**（内核模块，ARM 平台可动态加载）
- 原理：`obj-m` 表示把 `beep.c` 编译成可动态加载的内核模块 `.ko`

---

#### 2）编译测试 app

交叉编译命令（Ubuntu 主机）：

```bash
arm-linux-gnueabihf-gcc beepApp.c -o beepApp
```

- 产物：**`beepApp`**（ARM 平台可执行程序，**不能在 Ubuntu 主机直接运行**）

---

### 46.4.2 开发板运行测试

---

#### 1）文件拷贝

把 `beep.ko`、`beepApp` 拷贝到开发板根文件系统目录：

```bash
# 开发板端（假设用 NFS 挂载到 /mnt/nfs）
cp /mnt/nfs/beep.ko /lib/modules/4.1.15/
cp /mnt/nfs/beepApp /bin/
chmod +x /bin/beepApp
```

---

#### 2）加载驱动

进入 `/lib/modules/4.1.15` 目录：

```bash
cd /lib/modules/4.1.15

depmod          # 首次加载模块才需要：生成模块依赖关系表（仅首次执行）
modprobe beep.ko   # 加载蜂鸣器驱动（推荐，自动处理依赖）
```

**加载成功内核打印信息**：

```text
beep node has been found!
beep-gpio num = 129
beep major=241, minor=0
```

> **GPIO 编号计算规则**：内核对所有 GPIO 控制器的引脚做**全局编号**，公式为 `GPIO(n)_IO(m) → (n-1)*32 + m`
> - GPIO5_IO01：`(5-1)*32 + 1 = 129`（和内核打印一致）
> - 对比上一章 GPIO1_IO03：`(1-1)*32 + 3 = 3`（完全匹配）

**两种模块加载方式对比**：
| 命令 | 特点 | 推荐场景 |
|------|------|----------|
| `modprobe beep.ko` | 自动查找依赖、支持别名 | ✅ 常规驱动加载 |
| `insmod beep.ko` | 简单直接、**不自动处理依赖** | 无依赖的简单模块、调试阶段 |

---

#### 3）测试 APP 控制蜂鸣器

```bash
# 打开蜂鸣器
./beepApp /dev/beep 1

# 关闭蜂鸣器
./beepApp /dev/beep 0
```

> **控制逻辑**：
> 应用层传 `1` → 驱动反转输出 `0` → PNP 导通 → 蜂鸣器响
> 应用层传 `0` → 驱动反转输出 `1` → PNP 截止 → 蜂鸣器关

---

#### 4）卸载驱动

测试完成后，卸载驱动释放资源：

```bash
rmmod beep.ko
```

> 卸载后，内核会自动删除 `/dev/beep` 设备节点，同时 `gpio_free()` 释放 `GPIO5_IO01` 引脚。