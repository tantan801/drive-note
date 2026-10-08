## 第五十八章 Linux INPUT 子系统实验
按键、鼠标、键盘、触摸屏都属于输入设备，Linux 内核提供input 子系统统一管理各类输入事件。input 设备本质依然是字符设备，内核已经提前注册好 input 类与字符设备（主设备号固定为13），驱动开发者只需要注册input_dev、上报输入事件，不需要自己手动注册字符设备。

---

### 58.1 input 子系统
#### 58.1.1 input 子系统简介
input 子系统和 pinctrl、gpio 子系统一样，是内核专门针对一类设备设计的驱动框架。
驱动开发者不用关心应用层如何解析事件，只负责采集硬件信息、上报标准输入事件；

- 三层架构：驱动层、核心层 (input core)、事件处理层，最终在用户空间生成设备节点/dev/input/eventX。 
1. 驱动层：硬件底层驱动，采集硬件状态，向上上报事件（我们主要编写这一层）
2. 核心层 input core：内核自带，提供 input 设备注册、事件分发 API；
3. 事件层：内核自带各类 handler（Keyboard/Mouse/TS Handler），和用户空间交互。
![图58-1 input子系统结构图](./photo/Linux%20INPUT子系统实验/58-1%20input子系统结构图.png)

#### 58.1.2 input 驱动编写流程
内核`input.c`（核心层）在系统启动时自动执行`input_init`：
1. 注册input类，`/sys/class/input`目录；
2. 调用`register_chrdev_region`注册主设备号`INPUT_MAJOR=13`的字符设备。 ✅ 所以我们写 input 驱动，不用再手动注册字符设备，只需要注册input_dev。

##### ① input_dev 结构体
`input_dev`代表一个输入设备，定义在`include/linux/input.h`。 里面的位图成员用来描述设备支持哪些事件：
- `evbit`：事件类型位图（`EV_KEY`、`EV_ABS` 等）
- `keybit`：按键码位图，支持哪些按键
- `relbit`：相对坐标（鼠标）
- `absbit`：绝对坐标（触摸屏）

**常用事件类型**
```c
#define EV_SYN 0x00    // 同步事件，上报完事件必须发送
#define EV_KEY 0x01    // 按键事件（本章实验使用）
#define EV_REL 0x02    // 相对坐标，鼠标
#define EV_ABS 0x03    // 绝对坐标，触摸屏
#define EV_REP 0x14    // 按键连按重复事件
```

**input_dev 注册 / 注销 API**
```c
// 分配input_dev设备
struct input_dev *input_allocate_device(void);
// 释放input_dev
void input_free_device(struct input_dev *dev);

// 注册input_dev到内核input子系统
int input_register_device(struct input_dev *dev);
// 注销input_dev
void input_unregister_device(struct input_dev *dev);
```

**完整注册四步流程**
1. `input_allocate_device()` 申请input_dev
2. 初始化：设置设备名称，用`__set_bit`/`input_set_capability`配置支持的事件类型、按键码
3. `input_register_device()` 注册 input 设备
4. 模块卸载：`input_unregister_device()` → `input_free_device()`

**设置事件支持三种写法（效果相同）**
```c
// 方法1：__set_bit
__set_bit(EV_KEY, inputdev->evbit);
__set_bit(EV_REP, inputdev->evbit);
__set_bit(KEY_0, inputdev->keybit);

// 方法2：直接对位图赋值
inputdev->evbit[0] = BIT_MASK(EV_KEY) | BIT_MASK(EV_REP);
inputdev->keybit[BIT_WORD(KEY_0)] |= BIT_MASK(KEY_0);

// 方法3：input_set_capability（推荐，可读性好）
input_set_capability(inputdev, EV_KEY, KEY_0);
```

##### ② 上报输入事件 API
硬件状态发生变化（按键按下 / 松开），驱动需要上报事件给内核 input 子系统。基础总函数：
```c
void input_event(struct input_dev *dev, unsigned int type, unsigned int code, int value);
```
- `dev`：目标 input 设备
- `type`：事件类型 `EV_KEY`
- `code`：事件码，如`KEY_0`
- `value`：1 = 按下，0 = 松开

封装好的专用上报函数（推荐优先使用）
```c
// 上报按键事件，本质内部调用input_event
static inline void input_report_key(struct input_dev *dev, unsigned int code, int value);
void input_report_rel(struct input_dev *dev, unsigned int code, int value);  //相对坐标
void input_report_abs(struct input_dev *dev, unsigned int code, int value); //绝对坐标
```
**重点**：上报完事件，必须调用`input_sync`发送同步事件，通知内核本次事件上报完成
```c
void input_sync(struct input_dev *dev);
```

**按键上报示例（放在按键消抖定时器里面）**
```c
void timer_function(unsigned long arg)
{
    unsigned char value;
    value = gpio_get_value(keydesc->gpio);
    if(value == 0){
        input_report_key(inputdev, KEY_0, 1); // 按下
        input_sync(inputdev);
    } else {
        input_report_key(inputdev, KEY_0, 0); //松开
        input_sync(inputdev);
    }
}
```

---

#### 58.1.3 input_event 结构体（用户空间读取）
应用程序读取`/dev/input/eventX`，读到的数据就是`input_event`结构体，所有输入设备统一这个数据格式。
```c
struct input_event {
    struct timeval time;  //事件发生时间
    __u16 type;           //事件类型 EV_KEY
    __u16 code;           //事件码 KEY_0
    __s32 value;          //事件值：1按下，0松开
};

struct timeval {
    __kernel_time_t tv_sec;     //秒
    __kernel_suseconds_t tv_usec; //微秒
};
```
- `timeval`：事件发生的时间戳
- 不管是按键、鼠标、触摸屏，应用读取到的数据都是这个结构体，实现驱动和应用层接口标准化。

---

### 58.2 硬件原理图分析
本章实验使用 I.MX6U-ALPHA 开发板 KEY0 按键，原理图参考按键驱动。

---

### 58.3 实验程序编写
例程路径：开发板光盘 -> 2、Linux 驱动例程 -> 20_input 本章实验目标：编写基于 input 子系统的 KEY0 按键驱动；同时了解内核自带gpio-keys平台驱动，只需要设备树描述硬件即可，不用写驱动 C 代码。

#### 58.3.1 修改设备树文件
直接复用 49.3.1 小节已经建好的`key`节点，无需新增节点。

#### 58.3.2 按键 input 驱动程序编写
新建`20_input`工程文件夹，创建`keyinput.c`。
代码来源：在之前 13_irq 中断实验`imx6uirq.c`基础上，删除字符设备相关代码，增加 input 子系统代码。

**1. 设备结构体**
```c
struct keyinput_dev{
    dev_t devid;
    struct cdev cdev;
    struct class *class;
    struct device *device;
    struct device_node *nd;
    struct timer_list timer;        //按键消抖定时器
    struct irq_keydesc irqkeydesc[KEY_NUM];
    unsigned char curkeynum;
    struct input_dev *inputdev;     // input设备核心指针
};
```
在设备结构体增加`struct input_dev *inputdev`，代表输入设备。

**2. 中断服务函数 `key0_handler`**
中断触发后不直接上报事件，只启动 10ms 定时器，用于软件消抖，避免机械按键抖动造成多次误触发。
```c
static irqreturn_t key0_handler(int irq, void *dev_id)
{
    struct keyinput_dev *dev = (struct keyinput_dev *)dev_id;
    dev->curkeynum = 0;
    dev->timer.data = (volatile long)dev_id;
    mod_timer(&dev->timer, jiffies + msecs_to_jiffies(10));
    return IRQ_RETVAL(IRQ_HANDLED);
}
```

**3. 定时器回调函数 `timer_function`**
核心：上报 input 事件，定时器超时（消抖完成），读取 GPIO 真实电平，上报按键事件。
```c
void timer_function(unsigned long arg)
{
    unsigned char value;
    unsigned char num;
    struct irq_keydesc *keydesc;
    struct keyinput_dev *dev = (struct keyinput_dev *)arg;

    num = dev->curkeynum;
    keydesc = &dev->irqkeydesc[num];
    value = gpio_get_value(keydesc->gpio);

    if(value == 0){
        //按键按下，上报KEY_0，value=1
        input_report_key(dev->inputdev, keydesc->value, 1);
        input_sync(dev->inputdev);  //同步事件，必须调用
    } else {
        //按键松开，value=0
        input_report_key(dev->inputdev, keydesc->value, 0);
        input_sync(dev->inputdev);
    }
}
```
**关键点**：`input_report_key()`上报按键事件，`input_sync()`发送同步事件 EV_SYN，必不可少，通知内核本次一组事件上报完毕。

**4. `keyio_init`硬件与 input 设备初始化**
- `of`接口解析设备树，获取 GPIO 号、解析中断号；
- `request_irq`申请双边沿中断（上升沿 + 下降沿，按下 / 松开都触发中断）；
- `input_allocate_device()`分配input_dev；
- 设置支持事件`EV_KEY`、`EV_REP`，指定按键码`KEY_0`；
- `input_register_device()`注册 input 设备到内核。

```c
// 三种设置事件能力方式，本章选用第三种 input_set_capability
keyinputdev.inputdev->evbit[0] = BIT_MASK(EV_KEY) | BIT_MASK(EV_REP);
input_set_capability(keyinputdev.inputdev, EV_KEY, KEY_0);

ret = input_register_device(keyinputdev.inputdev);
```

**5. 模块出口 `keyinput_exit`**
卸载顺序：删除定时器 → 释放中断 → 释放 GPIO → 先 unregister，再 free input_dev
```c
static void __exit keyinput_exit(void)
{
    unsigned int i = 0;
    del_timer_sync(&keyinputdev.timer);
    for (i = 0; i < KEY_NUM; i++) {
        free_irq(keyinputdev.irqkeydesc[i].irqnum, &keyinputdev);
    }
    for (i = 0; i < KEY_NUM; i++) {
        gpio_free(keyinputdev.irqkeydesc[i].gpio);
    }
    input_unregister_device(keyinputdev.inputdev);
    input_free_device(keyinputdev.inputdev);
}
```

#### 58.3.3 编写测试 APP `keyinputApp.c`
应用程序通过读取`/dev/input/eventX`，拿到`struct input_event`标准事件结构体。
```c
static struct input_event inputevent;

int main(int argc, char *argv[])
{
    int fd, err;
    char *filename;
    filename = argv[1];
    if(argc != 2) {
        printf("Error Usage!\r\n");
        return -1;
    }
    fd = open(filename, O_RDWR);
    if (fd < 0) {
        printf("Can't open file %s\r\n", filename);
        return -1;
    }
    while (1) {
        err = read(fd, &inputevent, sizeof(inputevent));
        if (err > 0) {
            switch (inputevent.type) {
            case EV_KEY:
                if (inputevent.code < BTN_MISC) {
                    printf("key %d %s\r\n", inputevent.code, inputevent.value ? "press" : "release");
                } else {
                    printf("button %d %s\r\n", inputevent.code, inputevent.value ? "press" : "release");
                }
                break;
            case EV_REL:
            case EV_ABS:
            case EV_MSC:
            case EV_SW:
                break;
            }
        }
    }
    return 0;
}
```
使用方法：`./keyinputApp /dev/input/event1`
- `read`阻塞读取 event 节点，读到`input_event`；
- 判断`type == EV_KEY`，打印按键 code 和 press/release 状态；
- `KEY_0` 的 code=11。

---

### 58.4 运行测试
#### 58.4.1 编译驱动和 APP
Makefile：`obj-m := keyinput.o`，`make`编译生成`keyinput.ko`
交叉编译 APP：`arm-linux-gnueabihf-gcc keyinputApp.c -o keyinputApp`

#### 58.4.2 开发板测试
1. 拷贝 ko 与 APP 到开发板根文件系统
2. 查看`ls /dev/input`，加载驱动前只有 event0、mice
3. 加载模块
```shell
depmod
modprobe keyinput.ko
```
加载成功后，`/dev/input`新增`event1`，就是我们注册的 input 设备。
4. 运行测试程序
```shell
./keyinputApp /dev/input/event1
```
按下 / 松开 KEY0，终端打印 `key 11 press` / `key 11 release`

**备选测试命令**：直接`hexdump /dev/input/event1`，查看原始十六进制 input_event 数据包
- 数据包结构：`tv_sec | tv_usec | type | code | value`
- `type=0x01`：EV_KEY 按键事件
- `type=0x00`：EV_SYN 同步事件
- `code=0xb(11)`：KEY_0
- `value=1` 按下，`value=0` 松开

---

### 58.5 Linux 自带按键驱动 `gpio_keys` 使用（不用自己写驱动 C 代码）
#### 58.5.1 `gpio_keys.c` 源码简析
文件路径：`drivers/input/keyboard/gpio_keys.c`，标准 platform 驱动
- `compatible = "gpio-keys"`，设备树节点匹配；
- `probe`函数：`devm_input_allocate_device`分配 input_dev，解析设备树获取 gpio、按键码；
- 中断处理函数`gpio_keys_irq_isr`内部调用`input_event + input_sync`上报按键事件；
原理和我们手写的`keyinput.c`完全一致，内核已经帮我们写好整套 input + 中断 + 消抖逻辑。

**内核配置路径**
```shell
-> Device Drivers
    -> Input device support
        -> Keyboards
            -> GPIO Buttons (CONFIG_KEYBOARD_GPIO=y)
```

#### 58.5.2 使用自带 `gpio-keys`，只需要写设备树节点
设备树节点示例：
```dts
gpio-keys {
    compatible = "gpio-keys";
    #address-cells = <1>;
    #size-cells = <0>;
    autorepeat;  /*开启按键连按*/
    key0 {
        label = "GPIO Key Enter";
        linux,code = <KEY_ENTER>;
        gpios = <&gpio1 18 GPIO_ACTIVE_LOW>;
    };
};
```

**属性说明**：
1. `compatible = "gpio-keys"`：和内核`gpio_keys`驱动匹配；
2. `autorepeat`：支持按键长按重复触发；
3. `linux,code`：指定这个 GPIO 按键模拟成哪个标准按键（本例 KEY_ENTER 回车键）；
4. `gpios`：指定按键对应的 GPIO，低电平有效；

应用场景：LCD 熄屏后，按下这个按键，相当于按下回车，可以唤醒屏幕。