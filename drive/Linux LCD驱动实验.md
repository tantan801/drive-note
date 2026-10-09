# 第五十九章 Linux LCD 驱动实验

LCD 是很常用的一个外设，在裸机篇中我们讲解了如何编写 LCD 裸机驱动，在 Linux 下 LCD 的使用更加广泛，在搭配 QT 这样的 GUI 库下可以制作出精美的 UI 界面。本章我们就来学习一下如何在 Linux 下驱动 LCD 屏幕。

---

## 59.1 Linux下LCD驱动简析

先来回顾一下裸机的时候 LCD 驱动是怎么编写的，裸机 LCD 驱动编写流程如下：  
①、初始化 I.MX6U 的 eLCDIF 控制器，重点是 LCD 屏幕宽 (width)、高 (height)、hspw、hbp、hfp、vspw、vbp 和 vfp 等时序信息。  
②、初始化 LCD 像素时钟。  
③、设置 RGB LCD 显存。  
④、应用程序直接通过操作显存来操作 LCD，实现在 LCD 上显示字符、图片等信息。

在 Linux 中应用程序最终也是通过操作 RGB LCD 的显存来实现在 LCD 上显示字符、图片等信息。

裸机：可以随意分配显存  
Linux：内存管理严格，显存必须专门申请；加上虚拟内存机制，驱动侧、应用侧看到的地址不一样，但必须指向同一片物理内存。

---

### 59.1.1 Framebuffer设备

为了解决上述问题，Framebuffer（帧缓冲，简称 fb）应运而生。 fb 是一种内核显示机制，把显示相关软硬件封装，虚拟出 fb 设备。驱动加载成功后会生成 `/dev/fbX`（X=0,1,2...）。应用程序直接读写 `/dev/fbX` 即可操作 LCD 屏幕。

I.MX6U 官方内核默认已经开启 LCD 驱动，系统启动后就能看到 `/dev/fb0`。

`/dev/fb0` 就是 LCD 对应的设备文件，`/dev/fb0` 属于字符设备，拥有对应的 file_operations 操作集。 fb 的 file_operations 在 `drivers/video/fbdev/core/fbmem.c` 中定义，

1. fb_fops 成员：var（可变参数）、fix（固定参数）、fbops（操作集）、screen_base（显存虚拟地址）、screen_size（显存大小）、pseudo_palette（伪调色板）。
2. mxsfb_probe 核心流程：  
    - 获取寄存器、中断资源；  
    - 申请 mxsfb_info、fb_info；  
    - 寄存器地址映射；  
    - 初始化 fb_info；  
    - 从设备树读取屏幕参数；  
    - 初始化 eLCDIF 控制器；  
    - 注册 framebuffer 设备。

---

### 59.1.2 LCD原理图分析

![LCD原理图](photo/Linux%20LCD驱动实验/59-1%20LCD原理图.png)

本实验用到硬件资源：  
① 指示灯 LED0  
② RGB LCD 接口  
③ DDR3  
④ eLCDIF

底板 RGB LCD 原理图有 3 颗 SGM3157，隔离 LCD_DATA7、LCD_DATA15、LCD_DATA23。 屏幕 R7/G7/B7 带上下拉电阻，用于屏幕 ID 识别；这三个引脚复用为 I.MX6U BOOT 配置引脚，直连会干扰启动。

不使用 LCD：开关断开，隔离线路；启用 LCD：LCD_DE 控制开关导通

接线：40P FPC 排线连接 ATK7016 屏幕与 I.MX6U-ALPHA 开发板。

---

## 59.3 LCD驱动程序编写

I.MX6ULL eLCDIF 驱动内核已经自带，无需修改驱动 C 代码，只需要修改设备树，修改重点 3 项：LCD IO 配置、LCD 屏幕时序参数节点、LCD 背光节点。

### 1）LCD 屏幕 IO 配置

文件：`imx6ull-alientek-emmc.dts`，iomuxc 节点内

- **pinctrl_lcdif_dat**：24 根 RGB 数据线配置
- **pinctrl_lcdif_ctrl**：LCD_CLK、ENABLE、HSYNC、VSYNC 控制线
- **pinctrl_pwm1**：背光 PWM 引脚 GPIO1_IO08，复用为 PWM1_OUT

修改要点：IO 电气属性 `0x79` 改为 `0x49`，降低驱动能力，适配板上 SGM3157 模拟开关；无模拟开关不用修改。

```dts
pinctrl_lcdif_dat: lcdifdatgrp {
    fsl,pins = <
        MX6UL_PAD_LCD_DATA00__LCDIF_DATA00  0x49
        MX6UL_PAD_LCD_DATA01__LCDIF_DATA01  0x49
        MX6UL_PAD_LCD_DATA02__LCDIF_DATA02  0x49
        MX6UL_PAD_LCD_DATA03__LCDIF_DATA03  0x49
        MX6UL_PAD_LCD_DATA04__LCDIF_DATA04  0x49
        MX6UL_PAD_LCD_DATA05__LCDIF_DATA05  0x49
        MX6UL_PAD_LCD_DATA06__LCDIF_DATA06  0x49
        MX6UL_PAD_LCD_DATA07__LCDIF_DATA07  0x49
        MX6UL_PAD_LCD_DATA08__LCDIF_DATA08  0x49
        MX6UL_PAD_LCD_DATA09__LCDIF_DATA09  0x49
        MX6UL_PAD_LCD_DATA10__LCDIF_DATA10  0x49
        MX6UL_PAD_LCD_DATA11__LCDIF_DATA11  0x49
        MX6UL_PAD_LCD_DATA12__LCDIF_DATA12  0x49
        MX6UL_PAD_LCD_DATA13__LCDIF_DATA13  0x49
        MX6UL_PAD_LCD_DATA14__LCDIF_DATA14  0x49
        MX6UL_PAD_LCD_DATA15__LCDIF_DATA15  0x49
        MX6UL_PAD_LCD_DATA16__LCDIF_DATA16  0x49
        MX6UL_PAD_LCD_DATA17__LCDIF_DATA17  0x49
        MX6UL_PAD_LCD_DATA18__LCDIF_DATA18  0x49
        MX6UL_PAD_LCD_DATA19__LCDIF_DATA19  0x49
        MX6UL_PAD_LCD_DATA20__LCDIF_DATA20  0x49
        MX6UL_PAD_LCD_DATA21__LCDIF_DATA21  0x49
        MX6UL_PAD_LCD_DATA22__LCDIF_DATA22  0x49
        MX6UL_PAD_LCD_DATA23__LCDIF_DATA23  0x49
    >;
};

pinctrl_lcdif_ctrl: lcdifctrlgrp {
    fsl,pins = <
        MX6UL_PAD_LCD_CLK__LCDIF_CLK         0x49
        MX6UL_PAD_LCD_ENABLE__LCDIF_ENABLE   0x49
        MX6UL_PAD_LCD_HSYNC__LCDIF_HSYNC     0x49
        MX6UL_PAD_LCD_VSYNC__LCDIF_VSYNC     0x49
    >;
};

pinctrl_pwm1: pwm1grp {
    fsl,pins = <
        MX6UL_PAD_GPIO1_IO08__PWM1_OUT   0x110b0
    >;
};
```

### 2）LCD 屏幕参数节点修改，ATK7016（1024\*600）示例

文件 `imx6ull-alientek-emmc.dts`，`&lcdif` 追加节点，删除未使用的 `pinctrl_lcdif_reset`；设置 `bits-per-pixel=24`，`bus-width=24` 开启 RGB888，修改 `display-timings` 屏幕时序。

```dts
&lcdif {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_lcdif_dat
                 &pinctrl_lcdif_ctrl>;
    display = <&display0>;
    status = "okay";

    display0: display {
        bits-per-pixel = <24>;
        bus-width = <24>;

        display-timings {
            native-mode = <&timing0>;
            timing0: timing0 {
                clock-frequency = <51200000>;
                hactive = <1024>;
                vactive = <600>;
                hfront-porch = <160>;
                hback-porch = <140>;
                hsync-len = <20>;
                vback-porch = <20>;
                vfront-porch = <12>;
                vsync-len = <3>;

                hsync-active = <0>;
                vsync-active = <0>;
                de-active = <1>;
                pixelclk-active = <0>;
            };
        };
    };
};
```

### 3）LCD背光节点信息

背光 IO：GPIO1_IO08，复用 PWM1_OUT，驱动文件 `drivers/video/backlight/pwm_bl.c`，`compatible="pwm-backlight"`。

pwm1 节点追加配置：

```dts
&pwm1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_pwm1>;
    status = "okay";
};
```

backlight背光节点：

```dts
backlight {
    compatible = "pwm-backlight";
    pwms = <&pwm1 0 5000000>;
    brightness-levels = <0 4 8 16 32 64 128 255>;
    default-brightness-level = <6>;
    status = "okay";
};
```

backlight 属性说明：

| 属性 | 说明 |
|------|------|
| `pwms` | 绑定 pwm1，周期 5000000ns，频率 200Hz |
| `brightness-levels` | 8 级背光亮度 |
| `default-brightness-level` | 默认亮度等级 6，对应 50.19% 占空比 |

修改完成后重新编译 dtb 设备树，替换开发板设备树启动。

---

## 59.4 运行测试

### 59.4.1 LCD 屏幕基本测试

#### 1）编译新设备树

设备树修改完成，执行命令编译 dtb：

```bash
make dtbs
```

编译输出 `imx6ull-alientek-emmc.dtb`，使用新 dtb 启动内核。

#### 2）使能 Linux 开机企鹅 logo

内核图形化配置路径：

```
-> Device Drivers
    -> Graphics support
        -> Bootup logo
```

勾选三项：

- Standard black and white Linux logo
- Standard 16-color Linux logo
- Standard 224-color Linux logo

![logo配置项](photo/Linux%20LCD驱动实验/59-2%20logo配置项.png)

保存配置，重新编译内核生成 zImage。 使用新 zImage + 新 dtb 启动，LCD 左上角出现彩色小企鹅，代表 LCD 驱动基本正常。

![启动logo显示](photo/Linux%20LCD驱动实验/59-3%20启动logo显示.png)

---

### 59.4.2 设置LCD作为终端控制台

#### 1）U-Boot设置bootargs，串口+tty1控制

```bash
setenv bootargs 'console=tty1 console=ttymxc0,115200 root=/dev/nfs rw nfsroot=192.168.1.250:/home/zuozhongkai/linux/nfs/rootfs ip=192.168.1.251:192.168.1.250:192.168.1.1:255.255.255.0::eth0:off'
saveenv
```

| 参数 | 说明 |
|------|------|
| `console=tty1` | LCD 屏幕作为控制台 |
| `console=ttymxc0,115200` | 串口保留控制台 |

启动后串口和 LCD 同时打印内核 log。

#### 2）根文件系统 `/etc/inittab` 添加 tty1 终端

```
tty1::askfirst:-/bin/sh
```

重启开发板，LCD 打印：`Please press Enter to activate this console.`

- 板载 KEY 按键已经注册为回车按键，按 KEY 激活 LCD 终端
- 也可以外接 USB 键盘操作 LCD 终端

测试输出文字到 LCD 屏幕：

```bash
echo hello linux > /dev/tty1
```

---

### 59.4.3 LCD 背光调节

背光 sysfs 路径：`/sys/devices/platform/backlight/backlight/backlight`

文件说明：

| 文件 | 说明 |
|------|------|
| `brightness` | 当前背光等级，范围 0~7 |
| `max_brightness` | 最大背光等级，固定 7 |

调节命令示例：

```bash
echo 7 > brightness   # 最大亮度
echo 0 > brightness   # 关闭背光，屏幕熄灭
```

---

### 59.4.4 LCD自动熄屏解决方法

默认闲置 10 分钟自动熄屏，三种方案：

#### 1、按键唤醒

按回车（板载 KEY 或 USB 键盘）直接唤醒屏幕

#### 2、内核源码永久关闭熄屏

文件：`drivers/tty/vt/vt.c`

```c
// static int blankinterval = 10*60;  // 默认10分钟
static int blankinterval = 0;        // 修改为0，关闭自动熄屏
```

修改后重新编译内核 zImage。

#### 3、应用程序方式关闭熄屏，`lcd_always_on.c`

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/ioctl.h>

int main(int argc, char *argv[])
{
    int fd;
    fd = open("/dev/tty1", O_RDWR);
    write(fd, "\033[9;0]", 8);  // 关闭终端黑屏
    close(fd);
    return 0;
}
```

交叉编译命令：

```bash
arm-linux-gnueabihf-gcc lcd_always_on.c -o lcd_always_on
```

拷贝程序到开发板 `/usr/bin`，增加可执行权限。

`/etc/init.d/rcs` 末尾添加开机自启：

```bash
cd /usr/bin
./lcd_always_on
cd ..
```

重启开发板生效。

