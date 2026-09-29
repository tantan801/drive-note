# 第 44 章 设备树下的 LED 驱动实验

## 44.1 设备树 LED 驱动原理

对比第 42 章新字符设备驱动：

| 对比项 | 旧写法（第 42 章） | 设备树写法（本章） |
|--------|-------------------|--------------------|
| 硬件信息存放 | 寄存器物理地址直接硬编码写在驱动 C 文件 | 硬件信息（寄存器物理地址）写在 DTS 设备树节点 |
| 地址映射 | 手动 `ioremap` 映射 | 驱动用 OF 函数读取 DTS 里的属性，推荐 `of_iomap` 做地址映射 |
| 耦合度 | 硬件和驱动代码紧耦合 | 硬件信息和驱动代码解耦，修改硬件只需改 dts |

**核心 3 步：**

1. 修改 `imx6ull-alientek-emmc.dts`，新增 `alphaled` 设备节点，填写 `reg` 寄存器地址、`compatible` 等属性
2. 修改驱动程序，使用 OF 函数读取节点属性，拿到寄存器物理地址，推荐 `of_iomap` 做地址映射
3. 复用之前的 `ledApp` 测试应用，读写 `/dev/dtsled` 控制 LED

**硬件：** GPIO1_IO03 控制 LED

---

## 44.2 实验程序编写

### 44.2.1 修改设备树文件

打开 `imx6ull-alientek-emmc.dts` 文件，在根节点 `/` 下添加 `alphaled` 子节点：

```c
alphaled {
    #address-cells = <1>;
    #size-cells = <1>;
    compatible = "atkalpha-led";
    status = "okay";
    reg = < 0X020C406C 0X04        /* CCM_CCGR1_BASE 时钟 */
            0X020E0068 0X04        /* SW_MUX_GPIO1_IO03_BASE 复用 */
            0X020E02F4 0X04        /* SW_PAD_GPIO1_IO03_BASE 电气属性 */
            0X0209C000 0X04        /* GPIO1_DR_BASE 数据寄存器 */
            0X0209C004 0X04 >;     /* GPIO1_GDIR_BASE 方向寄存器 */
};
```

**属性说明：**

| 属性 | 说明 |
|------|------|
| `#address-cells = <1>` | reg 里面起始地址占用 1 个 cell |
| `#size-cells = <1>` | reg 里面地址长度占用 1 个 cell |
| `compatible = "atkalpha-led"` | 兼容性字符串，用来后续 platform 驱动匹配（本章还没用到 platform，下一章才用） |
| `status = "okay"` | 设备使能；`disabled` 代表禁用 |
| `reg` | 一组地址 + 长度，存放 5 组寄存器物理基地址 + 寄存器长度 |

**编译设备树：**

```bash
#切到ubuntu的内核根目录下
cd ~/alpha/alientek-alpha/kernel-alientek
#编译
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- dtbs -j4
```

编译完成后，前往 `arch/arm/boot/dts/imx6ull-alientek-emmc.dtb`，把 dtb 移植到板子（用 scp 或者 adb 都行），放到 `/run/media/mmcblk1p1` —— 这个目录是运行 eMMC 启动读的目录，用来存放内核镜像 `zImage` 和设备树文件 `dtb`。

在 U-Boot 命令行中指定该 dtb 文件，boot 重启后进入 Linux 命令行终端即可查到。并且可以通过 `cd` 进入目录 `ls` 查看相关属性文件：

```bash
root@ATK-IMX6U:/proc/device-tree# ls
'#address-cells'  alphaled   chosen  compatible  gpio_keys@0                    leds    model  pxp_v4l2    reserved-memory  '#size-cells'  sound
aliases           backlight  clocks  cpus        interrupt-controller@00a01000  memory  name   regulators  sii902x-reset    soc            spi4
root@ATK-IMX6U:/proc/device-tree#
root@ATK-IMX6U:/proc/device-tree# cd alphaled/
root@ATK-IMX6U:/proc/device-tree/alphaled# ls
'#address-cells'  compatible  name  reg  '#size-cells'  status
```

### 44.2.2 LED 灯驱动程序编写

`dtsled` 把硬件信息放到设备树 dts，驱动用 OF 函数读取设备树信息，**硬件信息和驱动代码解耦，这就是设备树的核心意义**。

和之前的 `newchrled` 相比，改动点如下：

#### 1）设备结构体增加设备树节点指针

```c
struct dtsled_dev{
    dev_t devid;
    struct cdev cdev;
    struct class *class;
    struct device *device;
    int major;
    int minor;
    struct device_node *nd;  // 新增：保存设备树节点，OF函数都要用这个指针
};
```

#### 2）驱动入口 `led_init()` 开头，新增整套 OF 函数读取设备树

```c
//1. 根据节点绝对路径查找 /alphaled
dtsled.nd = of_find_node_by_path("/alphaled");
if(dtsled.nd == NULL) {
    printk("alphaled node can not found!\r\n");
    return -EINVAL;
}
```

> 这里就是前面修改 U-Boot `fdt_file` 的目的。如果内核加载的 dtb 不是带 `alphaled` 节点的 `imx6ull-alientek-emmc.dtb`，这里直接打印找不到节点，`insmod` 直接失败。

```c
//2. 读取compatible属性
proper = of_find_property(dtsled.nd, "compatible", NULL);

//3. 读取status字符串属性
ret = of_property_read_string(dtsled.nd, "status", &str);

//4. 读取reg数组属性（存放寄存器物理地址+长度）
ret = of_property_read_u32_array(dtsled.nd, "reg", regdata, 10);
```

#### 3）寄存器物理地址映射：新增 `of_iomap`（设备树推荐）

```c
#if 0
//老办法：手动取出reg里的物理地址，再调用ioremap
IMX6U_CCM_CCGR1 = ioremap(regdata[0], regdata[1]);
...
#else
//设备树标准API：of_iomap(node, reg索引)，自动读reg+做ioremap，代码简洁
IMX6U_CCM_CCGR1 = of_iomap(dtsled.nd, 0);
SW_MUX_GPIO1_IO03 = of_iomap(dtsled.nd, 1);
SW_PAD_GPIO1_IO03 = of_iomap(dtsled.nd, 2);
GPIO1_DR = of_iomap(dtsled.nd, 3);
GPIO1_GDIR = of_iomap(dtsled.nd, 4);
#endif
```

> `of_iomap` 内部自动解析 `reg` 属性，不用我们自己从数组拿物理地址，设备树驱动优先用这个。

#### 4）剩下字符设备框架代码完全复用 42 章 `newchrled`

- `file_operations` 操作集：`open` / `write` / `release`
- 动态分配设备号 `alloc_chrdev_region`
- `cdev_init` + `cdev_add` 注册字符设备
- `class_create` + `device_create`：自动在 `/dev` 生成 `dtsled` 设备节点，不用手动 `mknod`
- 出口函数 `led_exit`：`iounmap` 释放虚拟地址，注销 cdev、销毁 class 和 device

### 44.2.3 编写测试程序

同之前一样，复用 `ledApp.c`。

---

## 44.3 运行测试

### 44.3.1 编译驱动程序和测试 APP

#### 1）编译驱动程序

编写 Makefile 文件，本章实验的 Makefile 和第四十章实验基本一样，只是将 `obj-m` 变量的值改为 `dtsled.o`：

```makefile
1 KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel 
_imx_4.1.15_2.1.0_ga_alientek 
...... 
4 obj-m := dtsled.o 
...... 
11 clean: 
12 $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

第 4 行，设置 `obj-m` 变量的值为 `dtsled.o`。

输入如下命令编译出驱动模块文件：

```bash
make -j32
```

编译成功以后就会生成一个名为 `dtsled.ko` 的驱动模块文件。

#### 2）编译测试 APP

输入如下命令编译测试 `ledApp.c`：

```bash
arm-linux-gnueabihf-gcc ledApp.c -o ledApp
```

编译成功以后就会生成 `ledApp` 这个应用程序。

### 44.3.2 运行测试

将上一小节编译出来的 `dtsled.ko` 和 `ledApp` 这两个文件拷贝到 `rootfs/lib/modules/4.1.15` 目录中，重启开发板，进入到目录 `lib/modules/4.1.15` 中，输入如下命令加载 `dtsled.ko` 驱动模块：

```bash
depmod  //第一次加载驱动的时候需要运行此命令 
modprobe dtsled.ko  //加载驱动
```

驱动加载成功以后会在终端中输出一些信息：
```
alphaled node find
compatible =  atkalpha-led
status = okay
reg data： xxxx
dtsled major = 249，minor = 0
```
驱动加载成功以后就可以使用 ledApp 软件来测试驱动是否工作正常，输入如下命令打开 LED 灯：
``` bash 
./ledApp /dev/dtsled 1  //打开 LED 灯
```
输入上述命令以后观察 I.MX6U-ALPHA 开发板上的红色 LED 灯是否点亮，如果点亮的话说明驱动工作正常。在输入如下命令关闭 LED 灯：
``` bash
./ledApp /dev/dtsled 0 //关闭 LED 灯
```

输入上述命令以后观察 I.MX6U-ALPHA 开发板上的红色 LED 灯是否熄灭。如果要卸载驱动的话输入如下命令即可：
```bash
rmmod dtsled.ko  //卸载驱动
```

