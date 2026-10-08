## 第五十七章 Linux MISC 驱动实验
misc 意为混合、杂项，MISC 驱动也叫杂项驱动。MISC 本质是简化版字符设备驱动，经常搭配 platform 总线一起使用，适合简单外设。

---

### 57.1 MISC 设备驱动简介
1. 固定主设备号 10，只需要分配不同次设备号 (minor)，解决内核主设备号资源紧张问题。
2. MISC 会自动创建 cdev、class、device，不用手动调用 alloc_chrdev_region、cdev_init、class_create、device_create 这一系列函数，大幅简化字符设备代码。
3. 核心结构体 miscdevice（include/linux/miscdevice.h）
```c
struct miscdevice {
    int minor;                  /* 子设备号 */
    const char *name;           /* 设备名字，生成/dev/xxx */
    const struct file_operations *fops; /* 设备操作函数集 */
    struct list_head list;
    struct device *parent;
    struct device *this_device;
    const struct attribute_group **groups;
    const char *nodename;
    umode_t mode;
};
```

只需要重点填充：minor、name、fops 三个成员。 次设备号可以手动指定，也可以使用 **MISC_DYNAMIC_MINOR(255)** 让内核动态分配。

**核心 API**
- 注册 MISC 设备：`int misc_register(struct miscdevice * misc)` 
  返回 0 成功，负数失败。替代传统一整套字符设备注册流程。
- 注销 MISC 设备：`int misc_deregister(struct miscdevice *misc)` 
  卸载模块时调用，自动释放 cdev、设备、类等资源。
- 传统字符设备需要手动写一大堆注册 / 注销函数；MISC 只需要一对 `misc_register` / `misc_deregister`。

---

### 57.2 硬件原理图分析
本章实验使用 I.MX6U-ALPHA 开发板 BEEP 蜂鸣器，原理图参考 14.3 小节。

---

### 57.3 实验程序编写
例程路径：开发板光盘 -> 2、Linux 驱动例程 -> 19_miscbeep
框架：platform + misc，platform 负责设备树匹配、GPIO 资源获取；misc 负责字符设备文件创建。

#### 57.3.1 修改设备树
复用之前章节的 beep 设备节点，节点 compatible 属性为 `atkalpha-beep`，无需新建节点。

#### 57.3.2 miscbeep.c 驱动代码
```c
/* MISC设备结构体 */
static struct miscdevice beep_miscdev = {
    .minor = MISCBEEP_MINOR,
    .name = MISCBEEP_NAME,
    .fops = &miscbeep_fops,
};
```

1. platform_driver 注册入口，内核匹配设备树节点后执行probe函数。
2. probe 函数： 
- 通过`of_find_node_by_path`找到设备树 beep 节点
- `of_get_named_gpio`从设备树获取蜂鸣器 GPIO
- `gpio_request`申请 GPIO，设置为输出，默认蜂鸣器关闭
- `misc_register(&beep_miscdev)`：注册杂项设备，自动生成 `/dev/miscbeep`
3. remove 函数： 
- 关闭蜂鸣器、释放 GPIO
- `misc_deregister(&beep_miscdev)`注销 MISC 设备

字符设备操作集 fops 只实现 open、write：应用层 write 写入 1 开蜂鸣器，写入 0 关闭蜂鸣器。

#### 57.3.3 测试 APP miscbeepApp.c
- 入参格式：`./miscbeepApp /dev/miscbeep 1` 
  参数 1：设备文件路径
  参数 2：1 打开蜂鸣器，0 关闭蜂鸣器
- 逻辑：open 打开设备，write 写入状态值，close 关闭文件。

---

### 57.4 运行测试
#### 57.4.1 编译驱动程序和测试 APP
**1、编写 Makefile**：修改 `obj-m := miscbeep.o`
```makefile
KERNELDIR := 内核源码路径
CURRENT_PATH := $(shell pwd)
obj-m := miscbeep.o

build:
    $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules

clean:
    $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

**2、编译驱动模块**
```shell
make -j32
```

**3、交叉编译测试APP**
```shell
arm-linux-gnueabihf-gcc miscbeepApp.c -o miscbeepApp
```

#### 57.4.2 开发板加载测试
将 miscbeep.ko、miscbeepApp 拷贝至开发板根文件系统
```shell
depmod
modprobe miscbeep.ko
# 查看设备，主设备号10，次设备号144
ls -l /dev/miscbeep

# 控制蜂鸣器
./miscbeepApp /dev/miscbeep 1  # 打开
./miscbeepApp /dev/miscbeep 0  # 关闭

# 卸载驱动
rmmod miscbeep.ko
```

**misc 类设备统一放在** `/sys/class/misc/`