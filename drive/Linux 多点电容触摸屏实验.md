# 第六十四章 Linux 多点电容触摸屏实验

---

## 64.1 Linux下电容触摸屏驱动框架简介

电容触摸 IC（例 FT5426）为 I2C 接口，依靠 INT 中断引脚上报触摸信息，坐标、按下抬起属于 input 子系统。驱动由三套框架组合：

1. **I2C 设备驱动**：触摸 IC 通信基础
2. **中断驱动框架**：中断服务函数读取触摸数据
3. **input 子系统**：按照 MT 多点触摸协议上报触摸事件

内核参考文档：`Documentation/input/multi-touch-protocol.txt`

---

### MT 协议 Type A / Type B

| 类型 | 说明 |
|------|------|
| **Type A** | 无法区分追踪触摸点，上报原始数据，实际很少使用，调用 `input_mt_sync()` 产生 `SYN_MT_REPORT` |
| **Type B** | 硬件可追踪区分触摸点，使用 slot 机制，**FT5426 属于此类型**，工业主流 |

### 核心 ABS_MT 常用事件

| 事件 | 作用 |
|------|------|
| `ABS_MT_SLOT` | 指定当前操作的触摸点 slot 编号 |
| `ABS_MT_TRACKING_ID` | 触摸点 ID，`-1` 代表触摸点抬起失效 |
| `ABS_MT_POSITION_X` | 触摸点 X 坐标 |
| `ABS_MT_POSITION_Y` | 触摸点 Y 坐标 |

### Type A 上报时序（2 个触摸点）

```c
ABS_MT_POSITION_X x[0]
ABS_MT_POSITION_Y y[0]
input_mt_sync()            // SYN_MT_REPORT
ABS_MT_POSITION_X x[1]
ABS_MT_POSITION_Y y[1]
input_mt_sync()            // SYN_MT_REPORT
input_sync()               // SYN_REPORT 全部上报完成
```

### Type B 上报时序（2 个触摸点，新增触摸）

```c
input_mt_slot(dev, 0)                       // ABS_MT_SLOT 0
input_mt_report_slot_state(..., true)       // ABS_MT_TRACKING_ID
input_report_abs(..., ABS_MT_POSITION_X, x0)
input_report_abs(..., ABS_MT_POSITION_Y, y0)

input_mt_slot(dev, 1)
input_mt_report_slot_state(..., true)
input_report_abs(..., ABS_MT_POSITION_X, x1)
input_report_abs(..., ABS_MT_POSITION_Y, y1)

input_sync(dev); // SYN_REPORT
```

> 触摸点抬起：`input_mt_report_slot_state(..., false)`，内核自动把 `ABS_MT_TRACKING_ID` 置 `-1`。

### MT 协议核心 API

| API | 说明 |
|------|------|
| `input_mt_init_slots(dev, num_slots, flags)` | 初始化 slots，指定最大支持触摸点数，MT 驱动**必须调用** |
| `input_mt_slot(dev, slot)` | Type B，设置当前操作 slot |
| `input_mt_report_slot_state(dev, tool_type, active)` | `active=true`：手指按下；`active=false`：手指抬起 |
| `input_report_abs(dev, code, value)` | 上报 ABS 坐标数据 |
| `input_mt_report_pointer_emulation(dev, use_count)` | 单点模拟转换 |
| `input_sync(dev)` | 上报 `SYN_REPORT`，一帧触摸数据结束 |

---

### 多点触摸驱动整体编写步骤

#### 1. I2C 驱动框架

`i2c_driver`，`of_device_id` 匹配设备树，probe 做初始化。

#### 2. probe 函数内部

- 从设备树读取复位、中断 GPIO
- 硬件复位触摸 IC
- 使用 `devm_request_threaded_irq` 申请线程化中断（I2C 读取慢，耗时操作放中断线程）
- `devm_input_allocate_device()` 分配 `input_dev`
- 设置事件：`EV_KEY`、`EV_ABS`、`BTN_TOUCH`；配置 `ABS_X` / `ABS_Y` / `ABS_MT_POSITION_X` / `ABS_MT_POSITION_Y` 参数
- `input_mt_init_slots` 初始化多点 slot
- `input_register_device()` 注册 input 设备

#### 3. 中断处理函数

- I2C 读取触摸 IC 寄存器，获取每个触摸点 ID、坐标、按下/抬起状态
- Type B 时序循环调用：`input_mt_slot` → `input_mt_report_slot_state` → `input_report_abs` 上报 XY
- `input_mt_report_pointer_emulation`
- `input_sync()` 结束一帧上报

> `devm_` 系列接口：资源由内核自动释放，不需要手动 `free_irq` / `gpio_free`。

---

## 64.2 硬件原理图分析

参考裸机 28.2 章节。FT5426 硬件资源：

| 资源 | 说明 |
|------|------|
| I2C2 总线 | SDA / SCL |
| INT 中断 GPIO | GPIO1_IO09 |
| RST 复位 GPIO | SNVS_TAMPER9 (GPIO5_IO09) |
| I2C 器件地址 | **0x38** |

---

## 64.3 实验程序编写

> **例程路径**：开发板光盘 → 2、Linux 驱动例程 → `23_multitouch`

---

### 64.3.1 修改设备树

#### 1. pinctrl 配置

```dts
/* 中断IO pinctrl_tsc */
pinctrl_tsc: tscgrp {
    fsl,pins = <
        MX6UL_PAD_GPIO1_IO09__GPIO1_IO09 0xF080 /* TSC_INT */
    >;
};

/* 复位IO，属于iomuxc_snvs */
pinctrl_tsc_reset: tsc_reset {
    fsl,pins = <
        MX6ULL_PAD_SNVS_TAMPER9__GPIO5_IO09 0x10B0
    >;
};
```

> I2C2 的 `pinctrl_i2c2` NXP 官方已配置，无需修改；务必检查引脚没有被其他外设占用。

#### 2. i2c2 下添加 ft5426 子节点

```dts
&i2c2 {
    clock_frequency = <100000>;
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c2>;
    status = "okay";

    ft5426: ft5426@38 {
        compatible = "edt,edt-ft5426";
        reg = <0x38>;
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_tsc &pinctrl_tsc_reset>;
        interrupt-parent = <&gpio1>;
        interrupts = <9 0>;
        reset-gpios = <&gpio5 9 GPIO_ACTIVE_LOW>;
        interrupt-gpios = <&gpio1 9 GPIO_ACTIVE_LOW>;
    };
};
```

---

### 64.3.2 ft5x06.c 驱动关键要点

- 自定义 `ft5x06_dev` 私有结构体，保存 client、input_dev、gpio 引脚号
- `ft5x06_ts_reset()`：复位 FT5426，拉高复位引脚退出复位
- `ft5x06_read_regs` / `ft5x06_write_regs`：基于 `i2c_transfer` 实现多寄存器读写
- `ft5x06_handler` 线程中断服务函数：

| 步骤 | 操作 |
|:---:|------|
| 1 | 从 `0x02` 寄存器连续读取 29 字节触摸数据 |
| 2 | 循环 5 个触摸点，解析 event、x、y、slot id |
| 3 | Type B 时序上报坐标 |
| 4 | `input_sync()` 结束上报 |

- probe 函数：解析设备树引脚、复位、申请线程中断、初始化 input、`input_mt_init_slots(..., 5, 0)`（5 点触摸）、注册 `input_dev`
- `i2c_driver` 注册，`module_init` / `module_exit`

---

## 64.4 运行测试

### 64.4.1 编译驱动模块 Makefile

```makefile
KERNELDIR := 内核源码路径
CURRENT_PATH := $(shell pwd)

obj-m := ft5x06.o

build:
    $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules

clean:
    $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

编译：`make -j32`，生成 `ft5x06.ko`。

### 64.4.2 模块加载测试

1. 编译设备树，烧写 dtb 启动开发板
2. ko 拷贝到 `rootfs/lib/modules/4.1.15`

```bash
depmod
modprobe ft5x06.ko
```

3. 驱动成功生成 `/dev/input/eventX`（例 `event2`）

```bash
hexdump /dev/input/event2
```

触摸屏幕可看到 ABS_MT 系列事件原始数据。

#### event 数据分析示例（Type B 单点击按下抬起）

| 步骤 | 事件 | 说明 |
|:---:|------|------|
| 1 | `ABS_MT_SLOT` | 指定 slot=0 |
| 2 | `ABS_MT_TRACKING_ID` | 分配 ID，代表手指按下 |
| 3 | `ABS_MT_POSITION_X` / `ABS_MT_POSITION_Y` | 上报坐标 |
| 4 | `BTN_TOUCH 1` | 触摸按下 |
| 5 | `ABS_X` / `ABS_Y` | `input_mt_report_pointer_emulation` 生成单点兼容坐标 |
| 6 | `SYN_REPORT` | 一帧结束 |
| 7 | 抬起 | `ABS_MT_TRACKING_ID = -1`，`BTN_TOUCH 0`，`SYN_REPORT` |

### 64.4.3 将驱动编译进内核

1. 将 `ft5x06.c` 拷贝到 `drivers/input/touchscreen/`
2. 修改该目录下 Makefile：

```makefile
obj-y += ft5x06.o
```

3. 重新编译 zImage，烧写内核，开机自动加载驱动，生成 `/dev/input/eventX`

---

## 64.5 tslib 移植与使用

tslib 触摸屏工具库，提供多点触摸测试工具 `ts_test_mt`，电容屏一般不需要校准。

### 64.5.1 tslib 1.21 交叉编译

```bash
# 修改源码所有者
sudo chown zuozhongkai:zuozhongkai tslib-1.21 -R
# 安装依赖
sudo apt-get install autoconf automake libtool

cd tslib-1.21
./autogen.sh
./configure --host=arm-linux-gnueabihf --prefix=/home/zuozhongkai/linux/IMX6ULL/tool/tslib
make
make install
```

将 prefix 输出目录下全部文件拷贝到开发板根文件系统。

**开发板配置**：

**1.** `/etc/ts.conf`，打开 `module_raw input`（去掉 `#` 注释）

**2.** `/etc/profile` 添加环境变量

```sh
export TSLIB_TSDEVICE=/dev/input/event1
export TSLIB_CALIBFILE=/etc/pointercal
export TSLIB_CONFFILE=/etc/ts.conf
export TSLIB_PLUGINDIR=/lib/ts
export TSLIB_CONSOLEDEVICE=none
export TSLIB_FBDEVICE=/dev/fb0
```

> `TSDEVICE` 填写实际的触摸 event 设备节点。

### 64.5.2 tslib 测试命令

| 命令 | 说明 |
|------|------|
| `ts_calibrate` | 校准，电容屏不需要；误校准删除 `/etc/pointercal` |
| `ts_test_mt` | 多点触摸测试工具，支持 Drag 拖拽、Draw 绘图、Quit 退出 |

> Draw 模式下多指同时触摸，屏幕画出多条轨迹，代表多点触摸工作正常。

---

## 64.6 使用内核自带 edt-ft5x06.c 驱动

1. 移除自己编译进内核的 ft5x06，删除 Makefile `obj-y += ft5x06.o`
2. 使用光盘提供修改好的 `edt-ft5x06.c` 替换内核源码 `drivers/input/touchscreen/edt-ft5x06.c`
3. `make menuconfig` 开启内核配置

```
Device Drivers  --->
    Input device support  --->
        Touchscreens  --->
            <*> EDT FocalTech FT5x06 I2C Touchscreen support
```

4. 修改设备树 ft5426 节点 compatible，增加内核驱动支持的匹配字符串

```dts
compatible = "edt,edt-ft5426", "edt,edt-ft5406";
```

5. 编译 zImage、dtb 启动，使用 `ts_test_mt` 验证

---

## 64.7 4.3 寸屏 GT9147 触摸驱动

- 触摸 IC：GT9147，I2C 地址 **0x14**

1. 修改 `pinctrl_tsc` 复位+中断引脚配置
2. i2c2 节点添加 gt9147 子节点

```dts
gt9147: gt9147@14 {
    compatible = "goodix,gt9147", "goodix,gt9xx";
    reg = <0x14>;
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_tsc>;
    interrupt-parent = <&gpio1>;
    interrupts = <9 0>;
    reset-gpios = <&gpio5 9 GPIO_ACTIVE_LOW>;
    interrupt-gpios = <&gpio1 9 GPIO_ACTIVE_LOW>;
    status = "okay";
};
```

3. 修改 `&lcdif` 设备树节点，填入 4.3 寸屏幕对应的时序参数（480×272 / 800×480 两套时序）
4. 编译加载 gt9147.ko，tslib 测试；注意：光盘 gt9147 示例驱动**仅实现单点触摸**

---

## 64.8 7 寸 GT911 触摸芯片

GT911 硬件兼容 GT9147，直接复用 `gt9147.c` 驱动；设备树 pinctrl、i2c 子节点配置参考 GT9147，I2C 地址 **0x14**。

