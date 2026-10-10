# 第六十三章 Linux RS232/485/GPS 驱动实验

I.MX6U-ALPHA 开发板 RS232、RS485、GPS 全部接在 UART3；硬件电平由外部芯片转换，内核 UART 驱动 NXP 已经实现，使用者主要修改设备树。最终生成设备文件 `/dev/ttymxcX`。

---

## 63.1 Linux 下 UART 驱动框架

### 1、uart_driver 注册与注销

`uart_driver`：描述一个串口驱动整体，定义于 `include/linux/serial_core.h`

```c
struct uart_driver {
    struct module *owner;        /* 模块所属者 */
    const char *driver_name;     /* 驱动名字 */
    const char *dev_name;        /* 设备名字 */
    int major;                    /* 主设备号 */
    int minor;                    /* 次设备号 */
    int nr;                       /* 设备数量 */
    struct console *cons;         /* 控制台，该串口是否作为console */

    struct uart_state *state;     /* 内核私有，驱动不要操作 */
    struct tty_driver *tty_driver;/* 内核私有 */
};
```

注册接口：

```c
int uart_register_driver(struct uart_driver *drv);   // 成功返回0，失败负值
void uart_unregister_driver(struct uart_driver *drv); // 注销串口驱动
```

### 2、uart_port 的添加与移除

`uart_port` 代表一路实际的串口硬件端口，定义于 `include/linux/serial_core.h`

```c
struct uart_port {
    spinlock_t lock;               /* port自旋锁 */
    unsigned long iobase;          /* IO物理地址 */
    unsigned char __iomem *membase;/* 寄存器虚拟映射地址 */
    // ...省略部分成员
    const struct uart_ops *ops;    /* 底层硬件操作函数集合，核心 */
    unsigned int line;             /* port索引号 */
    resource_size_t mapbase;       /* 寄存器物理基地址 */
    struct device *dev;            /* 父设备device */
    // ...省略部分成员
};
```

把 port 挂载到 uart_driver：

```c
int uart_add_one_port(struct uart_driver *drv, struct uart_port *uport);  // 添加端口
int uart_remove_one_port(struct uart_driver *drv, struct uart_port *uport);// 移除端口
```

### 3、uart_ops 实现

`uart_ops` 是串口最底层硬件操作集合，直接操作寄存器，收发、配置波特率全部依赖这里，驱动开发者必须实现。

```c
struct uart_ops {
    unsigned int (*tx_empty)(struct uart_port *);
    void (*set_mctrl)(struct uart_port *, unsigned int mctrl);
    unsigned int (*get_mctrl)(struct uart_port *);
    void (*stop_tx)(struct uart_port *);
    void (*start_tx)(struct uart_port *);
    void (*throttle)(struct uart_port *);
    void (*unthrottle)(struct uart_port *);
    void (*send_xchar)(struct uart_port *, char ch);
    void (*stop_rx)(struct uart_port *);
    void (*enable_ms)(struct uart_port *);
    void (*break_ctl)(struct uart_port *, int ctl);
    int (*startup)(struct uart_port *);      /* 串口打开时初始化硬件 */
    void (*shutdown)(struct uart_port *);    /* 串口关闭 */
    void (*flush_buffer)(struct uart_port *);
    void (*set_termios)(struct uart_port *, struct ktermios *new,struct ktermios *old); /* 设置波特率、数据位校验位 */
    void (*set_ldisc)(struct uart_port *, struct ktermios *);
    void (*pm)(struct uart_port *, unsigned int state,unsigned int oldstate);
    const char *(*type)(struct uart_port *);
    void (*release_port)(struct uart_port *);
    int (*request_port)(struct uart_port *);
    void (*config_port)(struct uart_port *, int);
    int (*verify_port)(struct uart_port *, struct serial_struct *);
    int (*ioctl)(struct uart_port *, unsigned int, unsigned long);
#ifdef CONFIG_CONSOLE_POLL
    int (*poll_init)(struct uart_port *);
    void (*poll_put_char)(struct uart_port *, unsigned char);
    int (*poll_get_char)(struct uart_port *);
#endif
};
```

函数详细说明参考内核文档：`Documentation/serial/driver`

---

## 63.2 I.MX6U UART 驱动分析

驱动源码路径：`drivers/tty/serial/imx.c`

### 1、UART 的 platform 驱动框架

`imx6ull.dtsi` 中 UART3 设备节点：

```dts
uart3: serial021ec000 {
    compatible = "fsl,imx6ul-uart","fsl,imx6q-uart", "fsl,imx21-uart";
    reg = <0x021ec000 0x4000>;    /* UART3寄存器基地址+长度 */
    interrupts = <GIC_SPI 26 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_UART3_IPG>,<&clks IMX6UL_CLK_UART3_SERIAL>;
    clock-names = "ipg", "per";
    dmas = <&sdma 29 4 0>, <&sdma 30 4 0>;
    dma-names = "rx", "tx";
    status = "disabled";  /* 默认关闭，板级dts覆盖status = "okay"启用 */
};
```

`imx.c` platform 驱动匹配表：

- `imx_uart_devtype`：传统 platform 匹配
- `imx_uart_dt_ids`：设备树 of 匹配，compatible 做匹配键

```c
static struct platform_driver serial_imx_driver = {
    .probe = serial_imx_probe,
    .remove = serial_imx_remove,
    .suspend = serial_imx_suspend,
    .resume = serial_imx_resume,
    .id_table = imx_uart_devtype,
    .driver = {
        .name = "imx-uart",
        .of_match_table = imx_uart_dt_ids,
    },
};

static int __init imx_serial_init(void)
{
    int ret = uart_register_driver(&imx_reg); // 注册uart_driver
    if (ret)
        return ret;
    ret = platform_driver_register(&serial_imx_driver);
    if (ret != 0)
        uart_unregister_driver(&imx_reg);
    return ret;
}

static void __exit imx_serial_exit(void)
{
    platform_driver_unregister(&serial_imx_driver);
    uart_unregister_driver(&imx_reg); // 注销uart_driver
}
module_init(imx_serial_init);
module_exit(imx_serial_exit);
```

### 2、uart_driver 初始化 imx_reg

```c
static struct uart_driver imx_reg = {
    .owner = THIS_MODULE,
    .driver_name = DRIVER_NAME,
    .dev_name = DEV_NAME,
    .major = SERIAL_IMX_MAJOR,
    .minor = MINOR_START,
    .nr = ARRAY_SIZE(imx_ports),
    .cons = IMX_CONSOLE,
};
```

### 3、uart_port 初始化与添加 serial_imx_probe

`struct imx_port`：NXP 自定义结构体，内部包含 `uart_port port` 成员。

```c
struct imx_port {
    struct uart_port port;    // 内嵌标准uart_port
    struct timer_list timer;
    unsigned int old_status;
    unsigned int have_rtscts:1;
    unsigned int dte_mode:1;
    unsigned int irda_inv_rx:1;
    unsigned int irda_inv_tx:1;
    unsigned short trcv_delay;
    // ...省略
};
```

`serial_imx_probe` 关键流程：

| 步骤 | 操作 |
|:---:|------|
| 1 | `devm_kzalloc` 分配 `imx_port` 内存 |
| 2 | `serial_imx_probe_dt` 读取设备树配置 |
| 3 | `platform_get_resource` 拿到寄存器物理地址，`devm_ioremap_resource` 做物理→虚拟地址映射 |
| 4 | `platform_get_irq` 获取串口中断号 |
| 5 | 填充 `sport->port`（uart_port）：dev、mapbase、membase、irq、`ops = &imx_pops`、rs485 配置、fifosize 等 |
| 6 | 获取 ipg、per 时钟，设置串口时钟频率 |
| 7 | `devm_request_irq` 申请收发中断 |
| 8 | `uart_add_one_port(&imx_reg, &sport->port)` 将初始化好的 port 挂载到 uart_driver |

### 4、imx_pops 结构体变量

`imx_pops` 就是 I.MX6ULL 平台 `uart_ops` 实例，所有函数直接操作 UART 硬件寄存器：

```c
static struct uart_ops imx_pops = {
    .tx_empty        = imx_tx_empty,
    .set_mctrl       = imx_set_mctrl,
    .get_mctrl       = imx_get_mctrl,
    .stop_tx         = imx_stop_tx,
    .start_tx        = imx_start_tx,
    .stop_rx         = imx_stop_rx,
    .enable_ms       = imx_enable_ms,
    .break_ctl       = imx_break_ctl,
    .startup         = imx_startup,
    .shutdown        = imx_shutdown,
    .flush_buffer    = imx_flush_buffer,
    .set_termios     = imx_set_termios,
    .type            = imx_type,
    .config_port     = imx_config_port,
    .verify_port     = imx_verify_port,
#if defined(CONFIG_CONSOLE_POLL)
    .poll_init      = imx_poll_init,
    .poll_get_char  = imx_poll_get_char,
    .poll_put_char  = imx_poll_put_char,
#endif
};
```

NXP 已经完整实现 UART 底层驱动，用户不需要写 `imx.c`；实际工作只需要在设备树中：

① 配置 pinctrl 串口 IO  
② `status = "okay"` 使能对应 uart 节点

内核加载后自动生成 `/dev/ttymxcX` 设备文件，RS485 额外配置 RS485 相关 DT 属性。

---

## 63.3 硬件原理图分析

本实验使用 I.MX6U 的 UART3 接口，开发板 RS232、RS485、GPS 全部挂载至 UART3。

### 1、RS232 原理图

![RS232原理图](photo/Linux%20RS232%E3%80%81485%E3%80%81GPS驱动实验/图1-1%20RS232原理图.jpeg)

- 电平转换芯片：SP3232
- 硬件连接：SP3232 接 I.MX6U 的 UART3，需要设置跳线帽 JP1，短接 1-3、2-4，SP3232 才和 UART3 连通

### 2、RS485 原理图

![RS485原理图](photo/Linux%20RS232%E3%80%81485%E3%80%81GPS驱动实验/图1-2%20RS485原理图.jpeg)

- 电平转换芯片：SP3485
- 引脚说明：RO 数据输出，RI 数据输入；RE 接收使能（低有效），DE 发送使能（高有效）
- 硬件处理：RE、DE 电路合并，由 RS485_RX 单路信号控制，节省一个控制 IO，上层可直接当作普通串口使用

### 3、GPS 原理图

![ATK MODULE原理图](photo/Linux%20RS232%E3%80%81485%E3%80%81GPS驱动实验/图1-3%20ATK%20MODULE原理图.jpeg)

- 适配模块：ATK1218-BD GPS + 北斗模块
- 模块接口同样接 UART3；UART3 驱动正常后，应用层可直接读取 GPS 输出数据

---

## 63.4 RS232 驱动编写

NXP 官方已经完成 I.MX6U UART 底层驱动，无需编写内核驱动，只修改设备树。

### 1、UART3 IO 节点创建

在 `iomuxc` 节点下添加 UART3 的 pinctrl 引脚配置，配置 TX、RX 引脚复用为 UART3 功能。

```dts
pinctrl_uart3: uart3grp {
    fsl,pins = <
        /* UART3_TX 复用为UART3_DCE_TX，配置电气属性 */
        MX6UL_PAD_UART3_TX_DATA__UART3_DCE_TX 0X1b0b1
        /* UART3_RX 复用为UART3_DCE_RX，配置电气属性 */
        MX6UL_PAD_UART3_RX_DATA__UART3_DCE_RX 0X1b0b1
    >;
};
```

重要检查：确认 UART3_TX、UART3_RX 引脚没有被其他外设占用，若被占用必须屏蔽冲突配置。

### 2、添加 uart3 设备节点

- `imx6ull-alientek-emmc.dts` 原有 uart2 节点占用 UART3 引脚，**删除 uart2 节点**
- 使用 `&uart3` 引用 dtsi 内部的 uart3 定义，增加 pinctrl、使能 status

```dts
&uart3 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_uart3>; /* 绑定上面定义的pinctrl_uart3引脚配置 */
    status = "okay";              /* 使能UART3外设 */
};
```

### 编译与设备文件

**1. 编译设备树**：

```bash
make dtbs
```

**2. 将新生成 `imx6ull-alientek-emmc.dtb` 下载到开发板启动**

**3. 驱动成功后生成设备文件**：`/dev/ttymxc2`，应用程序操作该文件完成串口收发。

---

## 63.5 移植 minicom

minicom 是 Linux 下串口调试工具，相当于 PC 端串口调试助手；依赖 ncurses 库，需要先交叉编译 ncurses。

### 1、移植 ncurses

#### 1）准备工作

- 源码：`ncurses-6.0.tar.gz`，存放路径：开发板光盘 → 例程源码 → 第三方库源码
- Ubuntu 创建工作目录：`/home/zuozhongkai/linux/IMX6ULL/tool`
- 解压源码，在 tool 下新建输出目录 `ncurses`

```bash
tar -vxzf ncurses-6.0.tar.gz
```

#### 2）配置（进入 ncurses-6.0 源码目录）

```bash
./configure --prefix=/home/zuozhongkai/linux/IMX6ULL/tool/ncurses \
--host=arm-linux-gnueabihf --target=arm-linux-gnueabihf \
--with-shared --without-profile --disable-stripping --without-progs \
--with-manpages --without-tests
```

| 参数 | 说明 |
|------|------|
| `--prefix` | 编译产物输出路径 |
| `--host` | 交叉编译器前缀 |

#### 3）编译安装

```bash
make
make install
```

#### 4）拷贝库、头文件到 NFS 根文件系统

```bash
sudo cp lib/* /home/zuozhongkai/linux/nfs/rootfs/usr/lib/ -rfa
sudo cp share/* /home/zuozhongkai/linux/nfs/rootfs/usr/share/ -rfa
sudo cp include/* /home/zuozhongkai/linux/nfs/rootfs/usr/include/ -rfa
```

#### 5）开发板 `/etc/profile`（无则新建）

```sh
#!/bin/sh
LD_LIBRARY_PATH=/lib:/usr/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH

export TERM=vt100
export TERMINFO=/usr/share/terminfo
```

### 2、移植 minicom

#### 1）准备源码：`minicom-2.7.1.tar.gz`，tool 目录新建输出目录 `minicom`

```bash
tar -vxzf minicom-2.7.1.tar.gz
cd minicom-2.7.1/
```

#### 2）configure 配置

```bash
./configure CC=arm-linux-gnueabihf-gcc \
--prefix=/home/zuozhongkai/linux/IMX6ULL/tool/minicom \
--host=arm-linux-gnueabihf \
CPPFLAGS=-I/home/zuozhongkai/linux/IMX6ULL/tool/ncurses/include \
LDFLAGS=-L/home/zuozhongkai/linux/IMX6ULL/tool/ncurses/lib \
--enable-cfg-dir=/etc/minicom
```

| 参数 | 说明 |
|------|------|
| `CPPFLAGS` | 指定 ncurses 头文件 |
| `LDFLAGS` | 指定 ncurses 库文件 |

#### 3）编译安装

```bash
make
make install
```

#### 4）拷贝可执行文件到根文件系统

```bash
sudo cp bin/* /home/zuozhongkai/linux/nfs/rootfs/usr/bin/
```

#### 5）开发板验证与排错

```bash
minicom -v        # 查看版本，输出2.7.1即编译正常
minicom -s        # 打开配置界面
```

报错提示 "Go away" 解决：新建 `/etc/passwd`

```
root:x:0:0:root:/root:/bin/sh
```

保存后重启开发板，再次执行 `minicom -s`。

---

## 63.6 RS232 驱动测试

### 63.6.1 RS232 连接设置

1. JP1 跳线帽：1-3、2-4 短接，UART3 接入 SP3232 (RS232)
2. PC 端：USB 转 DB9（CH340），电脑识别为 COM 口；SecureCRT 打开该 COM，波特率 115200

### 63.6.2 minicom 设置

```bash
minicom -s
```

选择 Serial port setup：

| 选项 | 设置 |
|------|------|
| A | 串口设备填写 `/dev/ttymxc2` |
| E | 设置波特率 115200，8N1 |
| F | 关闭硬件流控 |

回车确认，ESC 退出配置界面。

minicom 快捷键：

| 快捷键 | 功能 |
|------|------|
| Ctrl+A Z | 打开帮助菜单 |
| Ctrl+A E | 开关本地回显 |
| Ctrl+A X | 退出 minicom |

### 63.6.3 RS232 收发测试

**1. 发送测试（开发板 → PC）**

打开 minicom 本地回显，输入 `AAAA`；SecureCRT 收到 `AAAA`，发送正常。

**2. 接收测试（PC → 开发板）**

SecureCRT 会话开启 Local echo，PC 发送 `BBBB`；minicom 收到 `BBBB`，接收正常。

> 收发全部正常代表 UART3 驱动工作正常。

---

## 63.7 RS485 测试

硬件复用 UART3，不需要额外写驱动，直接使用 minicom。

### 63.7.1 RS485 连接设置

1. JP1 跳线帽：3-5、4-6 短接，切换为 RS485 通路
2. 使用 USB 转 RS485 转换器接 PC；接线：A 接 A，B 接 B，**严禁接反**
3. PC 识别 COM 口，波特率 115200，8N1，关闭流控

### 63.7.2 RS485 收发测试

开发板 minicom 打开 `/dev/ttymxc2`：

1. 开发板发送：minicom 输入 `AAAA`，PC 串口工具收到数据 → 发送正常
2. PC 发送 `BBBB`，minicom 收到 `BBBB` → 接收正常

---

## 63.8 GPS 测试（ATK1218-BD GPS + 北斗模块）

### 63.8.1 GPS 连接设置

1. JP1 跳线帽**全部拔掉**，断开 RS232/RS485，避免干扰 GPS
2. ATK1218-BD 模块靠左插到板上 ATK MODULE 接口
3. GPS 天线接好，天线必须放置**室外**，室内搜不到卫星

### 63.8.2 GPS 数据接收测试

minicom 配置 `/dev/ttymxc2`：

| 参数 | 设置 |
|------|------|
| 波特率 | **38400**（模块默认） |
| 数据位 | 8 |
| 停止位 | 1 |
| 软硬件流控 | 关闭 |

模块冷启动搜星需要数分钟，搜星成功后输出 NMEA 定位字符串（`$GPRMC`、`$GPGGA` 等）。

---

> **作者**：zuozhongkai | 正点原子