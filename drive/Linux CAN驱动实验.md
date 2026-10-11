# 第六十六章 Linux CAN驱动实验

CAN是应用十分广泛的现场总线，大量用于汽车电子与工业领域。汽车内部众多传感器、控制模块普遍采用CAN总线通信。I.MX6ULL片上集成CAN外设，本章学习I.MX6U ALPHA开发板CAN接口驱动开发。

---

## 66.1 CAN协议简析

CAN协议详细资料参考开发板资料：4、参考资料->CAN入门教程.pdf，本小节内容源自该文档。

### 66.1.1 何为 CAN？

CAN全称 Controller Area Network，控制局域网络。由德国博世BOSCH开发，已经成为国际标准ISO 11898，是主流现场总线。 最初面向汽车电子，现已成为汽车网络标准；同时大量应用在工业自动化、医疗设备、船舶等场景。

以车载网络为例如图1-1所示：

![](photo/Linux%20CAN驱动实验/1-1%20车载网络示意图.jpeg)

空调、车门、发动机、各类传感器作为独立CAN节点挂接在总线上。 同一个CAN网络内所有节点通信波特率必须保持一致；不同子网波特率可以不同，子网之间通过网关完成数据转发。

CAN总线核心特点：

1. **多主控制** 总线空闲所有节点均可发送报文；多个节点同时发送时，依靠报文ID做总线仲裁。ID代表报文优先级，不是目标设备地址。ID逐位对比，优先级高的节点继续发送，失利节点立刻停止发送转为接收。
2. **系统的柔软性** 节点没有设备地址概念。总线上新增节点，现有其他节点软硬件不需要做任何修改。
3. **通信速度快，传输距离远** 最高速率1Mbps（总线距离＜40m）；速率降低到5Kbps时，最大通信距离可达10KM。
4. **错误检测、错误通知、错误恢复** 全部节点具备错误检测；检测错误立刻通知全网；发送节点出错立刻终止发送，并且自动反复重发直到发送成功。
5. **故障封闭功能** 区分瞬时噪声错误与硬件持续性故障。出现持续错误时，可以把故障节点隔离出总线，避免整个总线瘫痪。
6. **可连接大量节点** 理论上节点数量无上限；实际受总线电气负载、信号延时限制。波特率越低，允许接入节点数量越多；波特率越高，可接节点越少。

### 66.1.2 CAN电气属性

CAN使用两根差分信号线：CAN_H、CAN_L。控制器依靠两根线电位差判断总线电平，分为显性电平、隐性电平如图1-2所示。

**图 1-2 CAN电平**

![](photo/Linux%20CAN驱动实验/1-2%20CAN电平.jpeg)

- 显性电平（逻辑0）：CAN_H≈3.5V，CAN_L≈1.5V，电位差2V
- 隐性电平（逻辑1）：CAN_H≈2.5V，CAN_L≈2.5V，电位差0V

总线空闲时持续处于隐性电平。 所有节点并联到CAN_H、CAN_L，如图1-3所示；总线两端必须接120Ω终端电阻，做阻抗匹配，抑制信号反射，提升抗干扰能力。

**图 1-3 CAN网络示意图**

![](photo/Linux%20CAN驱动实验/1-3%20CAN网络示意图.jpeg)

普通CAN最高1Mbps；CAN FD最高可达5Mbps及以上，本章不涉及CAN FD。传输距离越短，支持波特率越高。

### 66.1.3 CAN协议

CAN定义5种帧类型：数据帧、遥控帧、错误帧、过载帧、帧间隔。 数据帧、遥控帧分为标准帧（11位ID）与扩展帧（29位ID）。

| 帧类型 | 作用 |
|--------|------|
| 数据帧 | 节点之间传输有效业务数据 |
| 遥控帧 | 接收方请求对应ID节点发送数据 |
| 错误帧 | 检测到错误，向全网通告错误 |
| 过载帧 | 接收节点通知自身尚未准备好接收 |
| 间隔帧 | 将数据帧/遥控帧和前面报文隔开 |

#### 1、数据帧

数据帧一共由7个段组成如图1-4所示：

**图 1-4 数据帧结构**

![](photo/Linux%20CAN驱动实验/1-4%20数据帧结构.jpeg)

1. **帧起始** 标志帧开始；标准、扩展格式都是1bit显性电平0。
2. **仲裁段** 决定报文优先级（图1-5）。

   **图 1-5 仲裁段结构**

   ![](photo/Linux%20CAN驱动实验/1-5仲裁段结构.jpeg)

   - 标准帧：11bit ID（ID10~ID0）；最高7位不能全部为隐性1。
   - 扩展帧：29bit ID，由基础ID+扩展ID构成，基础ID同样禁止高7位全1。

3. **控制段** 共6bit；包含保留位R0/R1，必须显性发送；DLC代表数据长度，取值0~8，如图1-6所示。

   **图 1-6 控制段结构**

   ![](photo/Linux%20CAN驱动实验/1-6%20控制段结构.jpeg)

4. **数据段** 有效负载，0～8字节，从最高位MSB开始发送，如图1-7所示。

   **图 1-7 数据段**

   ![](photo/Linux%20CAN驱动实验/1-7%20数据段.jpeg)

5. **CRC段** 15位CRC校验值 + 1位CRC界定符。 CRC计算范围包含帧起始、仲裁、控制、数据段。接收端同样运算CRC，比对校验，不一致判定报文出错，如图1-8所示，如图1-8。

   **图 1-8 CRC段结构**

   ![](photo/Linux%20CAN驱动实验/1-8%20CRC段结构.jpeg)

6. **ACK段** ACK槽 + ACK界定符。 发送方发送两个隐性位；接收正确报文的节点，在ACK槽输出显性电平，告知发送方接收成功。只有无CRC错误、格式错误，非总线关闭的节点才会回复ACK，如图1-9所示。

   **图 1-9 ACK段结构**

   ![](photo/Linux%20CAN驱动实验/1-9%20ACK段结构.jpeg)

7. **帧结束段** 标记数据帧结束，如图1-10所示。

   **图 1-10 帧结束段结构**

   ![](photo/Linux%20CAN驱动实验/1-10%20帧结束段结构.jpeg)

D代表显性电平逻辑0；R代表隐性电平逻辑1；D/R代表该位可为0或1。

#### 2、遥控帧

接收单元向发送单元请求数据的时候就用遥控帧，遥控帧由6个段组成：

①、帧起始，表示数据帧开始的段。 ②、仲裁段，表示该帧优先级的段。 ③、控制段，表示数据的字节数及保留位的段。 ④、CRC段，检查帧的传输错误的段。 ⑤、ACK段，表示确认正常接收的段。 ⑥、帧结束，表示数据帧结束的段。 遥控帧结构如图2 1所示

**图 2-1 遥控帧结构**

![](photo/Linux%20CAN驱动实验/2-1%20遥控帧结构.jpeg)

可以看出，遥控帧结构基本和数据帧一样，最主要的区别就是遥控帧没有数据段。遥控帧的RTR位为隐性的，数据帧的RTR位为显性，因此可以通过RTR位来区分遥控帧和没有数据的数据帧。遥控帧没有数据，因此DLC表示的是所请求的数据帧数据长度，遥控帧的其他段参考数据帧的描述即可。

#### 3、错误帧

当接收或发送消息出错的时候使用错误帧来通知，错误帧由错误标志和错误界定符两部分组成，错误帧结构如图2 2所示

**图 2-2 错误帧结构**

![](photo/Linux%20CAN驱动实验/2-2%20错误帧结构.jpeg)

错误标志有主动错误标志和被动错误标志两种，主动错误标志是6个显性位，被动错误标志是6个隐性位，错误界定符由8个隐性位组成。

#### 4、过载帧

接收单元尚未完成接收准备的话就会发送过载帧，过载帧由过载标志和过载界定符构成，过载帧结构如图2 3所示，过载标志由6个显性位组成，与主动错误标志相同，过载界定符由8个隐性位组成，与错误帧中的错误界定符构成相同。

**图 2-3 过载帧结构**

![](photo/Linux%20CAN驱动实验/2-3%20过载帧结构.jpeg)

#### 5、帧间隔

帧间隔用于分隔数据帧和遥控帧，数据帧和遥控帧可以通过插入帧间隔来将本帧与前面的任何帧隔开，过载帧和错误帧前不能插入帧间隔，帧间隔结构如图2 4所示，中间隔由3个隐性位构成，总线空闲为隐性电平，长度没有限制，本状态下表示总线空闲，发送单元可以访问总线。延迟发送由8个隐性位构成，处于被动错误状态的单元发送一个消息后的帧间隔中才会有延迟发送。

**图 2-4 帧间隔结构**

![](photo/Linux%20CAN驱动实验/2-4%20帧间隔结构.jpeg)

### 66.1.4 CAN 速率

CAN总线以帧的形式发送数据，但是最终到总线上的就是"0"和"1"这样的二进制数据，这里就涉及到了通信速率，也就是每秒钟发送多少位数据，前面说了CAN2.0最高速度为1Mbps/S。对于CAN总线，一个位分为4段： ①、同步段(SS) ②、传播时间段(PTS) ③、相位缓冲段1(PBS1) ④、相位缓冲段2(PBS2)

这些段由Tq(Time Quantum)组成，Tq是CAN总线的最小时间单位。帧由位构成，一个位由4个段构成，每个段又由若干个Tq组成，这个就是位时序。1位由多少个Tq构成、每个段又由多少个 Tq 构成等，可以任意设定位时序。通过设定位时序，多个单元可同时采样，也可任意设定采样点。采样点是指读取总线电平，并将读到的电平作为位值的点。位置在 PBS1 结束处。根据这个位时序，我们就可以计算CAN通信的波特率了。具体计算方法，我们等下再介绍，前面提到的CAN协议具有仲裁功能，下面我们来看看是如何实现的。

在总线空闲态，最先开始发送消息的单元获得发送权。 当多个单元同时开始发送时，各发送单元从仲裁段的第一位开始进行仲裁。连续输出显性电平最多的单元可继续发送。单元1 和单元2同时开始向总线发送数据，开始部分他们的数据格式是一样的，故无法区分优先级，直到T时刻，单元1输出隐性电平，而单元2输出显性电平，此时单元1仲裁失利，立刻转入接收状态工作，不再与单元2竞争，而单元2则顺利获得总线使用权，继续发送自己的数据。这就实现了仲裁，让连续发送显性电平多的单元获得总线使用权。

关于CAN协议就讲到这里，关于CAN协议更详细的内容请参考《CAN入门教程》。

### 66.1.5 I.MX6ULL FlexCAN 简介

I.MX6ULL 带有CAN控制器外设，叫做FlexCAN，FlexCAN符合CAN2.0B协议。FlexCAN 完全符合CAN协议，支持标准格式和扩展格式，支持64个消息缓冲。I.MX6ULL自带的 FlexCAN 模块特性如下： ①、支持CAN2.0B协议，数据帧和遥控帧支持标准和扩展两种格式，数据长度支持0~8字节，可编程速度，最高1Mbit/S。 ②、灵活的消息邮箱，最高支持8个字节。 ③、每个消息邮箱可以配置为接收或发送，都支持标准和扩展这两种格式的消息。 ④、每个消息邮箱都有独立的接收掩码寄存器。 ⑤、强大的接收FIFO ID过滤。 ⑥、未使用的空间可以用作通用RAM。 ⑦、可编程的回测模式，用于进行自测。 ⑧、可编程的优先级组合。

FlexCAN 支持四种模式：正常模式(Normal)、冻结模式(Freeze)、仅监听模式(Listen Only) 和回环模式(Loop Back)，另外还有两种低功耗模式：禁止模式(Disable)和停止模式(Stop)。 ①、正常模式(Normal) 在正常模式下，FlexCAN正常接收或发送消息帧，所有的CAN协议功能都使能。 ②、冻结模式(Freeze) 当MCR寄存器的FRZ位置1的时候使能此模式，在此模式下无法进行帧的发送或接收， CAN总线同步丢失。 ③、仅监听模式(Listen Only) 当CTRL寄存器的LOM位置1的时候使能此模式，在此模式下帧发送被禁止，所有错误计数器被冻结，CAN控制器工作在被动错误模式，此时只会接收其他CAN单元发出的ACK消息。 ④、回环模式(Loop Back) 当CTRL寄存器的LPB位置1的时候进入此模式，此模式下FlexCAN工作在内部回环模式，一般用来进行自测。此模式下发送出来的数据流直接反馈给内部接收单元。

前面在讲解CAN协议的时候说过CAN位时序，FlexCAN支持CAN协议的这些位时序，控制寄存器CTRL用于设置这些位时序，CTRL寄存器中的PRESDIV、PROPSEG、PSEG1、PSEG2 和RJW这5个位域用于设置CAN位时序。

PRESDIV 为CAN分频值，也即是设置CAN协议中的Tq值，fCANCLK 为 FlexCAN 模块时钟，这个根据时钟章节设置即可，设置好以后就是一个定值， 因此只需要修改PRESDIV即可修改FlexCAN的Tq频率值。

Tq 定了以后我们结合图66.1.4.1中的各个段来看一下如何设置FlexCAN的速率： SS：同步段(Synchronization Segment)，在 I.MX6ULL 参考手册中叫做 SYNC_SEG，此段固定为1个Tq长度，因此不需要我们去设置。 PTS：传播时间段(Propagation Segment)，FlexCAN 的 CTRL 寄存器中的 PROPSEG 位域设置此段，可以设置为07，对应18个Tq。 PBS1：相位缓冲段1(Phase Buffer Segment 1)，FlexCAN 的 CTRL 寄存器中的 PSEG1 位域设置此段，可以设置为07，对应18个Tq。 PBS2：相位缓冲段2(Phase Buffer Segment 2)，FlexCAN 的 CTRL 寄存器中的 PSEG2 位域设置此段，可以设置为17，对应28个Tq。 SJW：再同步补偿宽度(reSynchronization Jump Width)，FlexCAN 的 CTRL 寄存器中的RJW位域设置此段，可以设置03，对应14个Tq。

---

## 66.2 硬件原理图分析

正点原子I.MX6U-ALPHA开发板CAN接口原理图如图3-1所示

**图 3-1 CAN原理图**

![](photo/Linux%20CAN驱动实验/3-1%20CAN原理图.jpeg)

CAN1_TX 和CAN1_RX 是I.MX6ULL FlexCAN1 的发送和接收引脚，对应I. MX6ULL 的UART3_CTS和UART3_RTS这两个引脚。TJA1050是CAN收发器，通过TJA10 50 向外界提供CAN_H和CAN_L总线，R10是一个120欧的端接匹配电阻。

---

## 66.3 实验程序编写

### 66.3.1 修改设备树

NXP 原厂提供的设备树已经配置好了FlexCAN 的节点信息(FlexCAN1 和 FlexCAN2)，但是我们还是要来看一下如何配置I.MX6ULL的CAN1节点。首先看一下I.MX6ULL的FlexCAN设备树绑定文档，打开Documentation/devicetree/bindings/net/can/fsl flexcan.txt，此文档描述了FlexCAN节点下的相关属性信息，这里就不做介绍了，大家自行查阅。

#### 1、FlexCAN1引脚节点信息

首先肯定是CAN1引脚配置信息，打开imx6ull alientek emmc.dts，找到如下所示内容：

**示例代码66.3.1.1 CAN1引脚信息**

```dts
1 pinctrl_flexcan1: flexcan1grp{
2     fsl,pins = <
3         MX6UL_PAD_UART3_RTS_B__FLEXCAN1_RX 0x1b020
4         MX6UL_PAD_UART3_CTS_B__FLEXCAN1_TX 0x1b020
5     >;
6 };
```

第3和第4行将UART3_RTS和UART3_CTS这两个引脚分别复用为FlexCAN1的RX和TX，电气属性都设置为0x1b020。

#### 2、FlexCAN1控制器节点信息

打开imx6ull.dtsi文件，找到名为"flexcan1"的节点，内容如下：

**示例代码66.3.1.2 imx6ull.dtsi文件flexcan1节点信息**

```dts
1 flexcan1: can@02090000 {
2     compatible = "fsl,imx6ul flexcan", "fsl,imx6q flexcan";
3     reg = <0x02090000 0x4000>;
4     interrupts = ;
5     clocks = <&clks IMX6UL_CLK_CAN1_IPG>,
6              <&clks IMX6UL_CLK_CAN1_SERIAL>;
7     clock names = "ipg", "per";
8     stop mode = <&gpr 0x10 1 0x10 17>;
9     status = "disabled";
10 };
```

注意示例代码66.3.1.2中的flexcan1节点不需要我们修改，这里只是告诉大家FlexCAN1完整节点信息。根据第2行的compatible属性就可以找到I.MX6ULL的FlexCAN驱动源文件，驱动文名为drivers/net/can/flexcan.c。第9行的status属性为disabled，所以FlexCAN1默认关闭的。 在imx6ull alientek emmc.dts中添加使能FlexCAN1的相关操作，找到如下所示代码：

**示例代码66.3.1.3 imx6ull alientek emmc.dtsi文件flexcan1节点信息**

```dts
1 &flexcan1 {
2     pinctrl names = "default";
3     pinctrl 0 = <&pinctrl_flexcan1>;
4     xceiver supply = <®_can_3v3>;
5     status = "okay";
6 };
```

第3行指定FlexCAN1所使用的pinctrl节点为pinctrl_flexcan1，也就是示例代码66.3.1.1中的pinctrl节点。 第4行xceiver supply属性指定CAN收发器的电压为3.3V。 第5行将flexcan1节点的status属性改为"okay"，也就是使能FlexCAN1。

#### 3、关闭FlexCAN2相关节点

I.MX6ULL 带有两个 CAN 控制器：FlexCAN1 和 FlexCAN2，NXP 官方的EVK 开发板这两个CAN接口都用到了，因此NXP官方的设备树将这两个CAN接口都使能了。但是，正点原子的I.MX6U ALPHA开发板将FlexCAN2的IO分配给了ECSPI3，所以正点原子的I.MX6U ALPHA 开发板就不能使用CAN2，否则的话ECSPI3外设就无法使用。关闭FlexCAN2节点很简单，在imx6ull alientek emmc.dts 文件中找到名为"flexcan2"的节点，然后将其屏蔽掉即可。

重新编译设备树，我们还需要配置Linux内核，使能内核里面的FlexCAN驱动。

### 66.3.2 使能 Linux 内核自带的FlexCAN驱动

NXP 官方提供的linux内核默认已经集成了I.MX6ULL的FlexCAN驱动，但是没有使能，因此我们需要配置Linux内核，打开FlexCAN驱动，步骤如下：

**1、使能CAN总线** 首先打开CAN总线子系统，在Linux下CAN总线是作为网络子系统的，配置路径如下：
-> Networking support -> <*> CAN bus subsystem support //打开 CAN 总线子系统

**2、使能Freescale 系CPU的FlexCAN外设驱动** 接着使能Freescale 系CPU的FlexCAN外设驱动，配置路径如下：
-> Networking support -> CAN bus subsystem support -> CAN Device Drivers -> Platform CAN drivers with Netlink support -> <*> Support for Freescale FLEXCAN based chips

配置好以后重新编译内核，然后使用新的内核和设备树启动开发板。

---

## 66.4 FlexCAN 测试

### 66.4.1 检查 CAN 网卡设备是否存在

使用新编译的内核和设备树启动开发板，然后输入如下命令：

```bash
ifconfig -a //查看所有网卡
```

可以看出，有一个名为"can0"的网卡，这个就是I.MX6U ALPHA开发板上的CAN1接口对应的can网卡设备。如果使能了I.MX6ULL上的FlexCAN2的话也会出现一个名为"can1"的can网卡设备。

### 66.4.2 移植 iproute2

在移植ip命令的时候必须先对根文件系统做个备份！防止操作失误导致系统启动失败！切记！笔者的血泪经验教训！

busybox 自带的ip命令并不支持对can的操作，因此我们需要重新移植ip命令，也就是iproute2，iproute2 源码下载地址为：`https://mirrors.edge.kernel.org/pub/linux/utils/net/iproute2/` 。这里我们下载4.4.0版本的，笔者试过最新的版本，但是编译一直有问题。4.4.0版本的iproute2已经下载好放到了开发板光盘中，路径为：1、例程源码 >7、第三方库源码 > iproute2 4.4.0.tar.gz。

将iproute2 4.4.0.tar.gz 发送到 ubuntu 中并解压，命令如下：

```bash
tar -vxzf iproute2-4.4.0.tar.gz
```

解压完成以后会得到一个名为"iproute2 4.4.0"的目录，进入此目录中，打开 Makefile 并修改。在Makefile中找到下面这行：

```makefile
CC := gcc
```

改为

```makefile
CC:=arm-linux-gnueabihf-gcc
```

Makefile 修改完成以后直接使用"make"命令编译，编译成功以后就会在iproute2源码的 ip 目录下得到一个名为"ip"的命令。

以下操作请严格按照教程步骤来！否则可能会导致系统无法启动！

**1、将交叉编译得到的ip拷贝到开发板中** 首先将交叉编译到的ip命令拷贝到开发板中，先不要替换开发板根文件系统中原有的ip命令！切记！切记！先拷贝到开发板根文件系统的其他目录里面，比如我这里就拷贝到/lib/modules/4.1.15 这个目录里面，命令如下：

```bash
sudo cp ip /home/zuozhongkai/linux/nfs/test_rootfs/lib/modules/4.1.15/ -f
```

拷贝完成以后在开发板上先执行一下新的ip命令，查看一下版本号，命令如下：

```bash
cd lib/modules/4.1.15/
./ip -V //执行新的ip命令，查看版本号
```

如果新编译的ip命令运行正确的话就会打印出其版本号。

**2、在开发板根文件系统中用新的ip命令替换原来的** 注意，此步骤在开发板中执行！开发板根文件系统中原来的ip命令是busybox自带的，存放在/sbin 目录下。接下来，我们使用新的ip命令替换原来的，在开发板中执行如下命令：

```bash
cd lib/modules/4.1.15/
cp ip /sbin/ip -f
```

拷贝完成以后将/lib/modules/4.1.15/目录下的 ip 命令删除掉，重启开发板，查看根文件系统是否可以正常启动！如果正常启动的话输入如下命令查看ip命令版本号：

```bash
ip -V //查看ip 命令版本号
```

至此，iproute2 中的ip 命令就已经移植好了，稍后的can测试中我们会使用ip命令来设置 can0 网卡的相关信息。

**3、替换ip命令以后系统启动失败怎么办？** 如果在替换ip命令的时候操作失误可能会导致开发板系统启动失败，系统启动失败，提示Object "it" is unknown, try "ip help".，这是因为ip 命令替换错误导致的！所以说一定要严格按照本节教程讲解的步骤替换ip命令。遇到错误把以前的ip命令替换回来就行了，这就是前面强烈建议大家对根文件系统做个备份的原因！！！最简单的方法就是用备份的根文件系统重新做一遍！

### 66.4.3 移植 can utils 工具

can0 网卡已经出现了，但是工作正不正常还不知道，必须要进行数据收发测试。这里我们使用can utils 这个工具来对can0网卡进行测试，因此要先移植can utils到我们的开发板根文件系统中。can utils 源码我们已经下载下来放到了开发板光盘中，路径为：1、例程源码 >7、第三方库源码 > can utils 2020.02.04.tar.gz。

在ubuntu 中新建一个名为"can utils"的目录来存放can utils的编译结果。然后将can utils 源码拷贝到ubuntu中并解压，命令如下：

```bash
tar -vxzf can-utils-2020.02.04.tar.gz //解压
```

解压完成以后得到一个名为"can utils 2020.02.04"的目录，这个就是can utils源码，进入到此目录中，然后配置并编译，命令如下：

```bash
cd can-utils-2020.02.04 //进入can-utils 源码目录
./autogen.sh //先执行autogen.sh，生成配置文件configure
./configure --target=arm-linux-gnueabihf --host=arm-linux-gnueabihf --prefix=/home/zuozhongkai/linux/IMX6ULL/tool/can-utils --disable-static --enable-shared //配置
make //编译
make install
```

编译完成以后就会前面创建的"can utils"目录下就会多出一个"bin"目录，此目录下保存着can utils 的各种小工具，所有can utils 小工具全部拷贝到开发板根文件系统下的/usr/bin目录下，命令如下：

```bash
sudo cp bin/* /home/zuozhongkai/linux/nfs/rootfs/usr/bin/ -f
```

拷贝完成以后我们就可以使用这些小工具来测试CAN了。

### 66.4.4 CAN 通信测试

正点原子的I.MX6U ALPHA 开发板上只有一个CAN接口，因此我们还需要另外一个CAN 设备，可以使用另一块I.MX6U ALPHA开发板，或者USB转CAN设备，这里我两个都测试一下。

#### 1、两块ALPHA开发板连接测试

准备两块ALPHA开发板，然后将CAN接口连接起来将两个开发板的CAN接口连接起来，注意，CAN_H接CAN_H，CAN_L接CAN_L！

##### ①、收发测试

首先使用ip 命令设置两个开发板的CAN接口，首先设置CAN接口的速度，输入如下所示命令：

```bash
ip link set can0 type can bitrate 500000
```

上述命令设置can0速度为500Kbit/S，两个CAN设备的速度要设置为一样的！速度设置好以后打开can0网卡，命令如下：

```bash
ifconfig can0 up //打开can0
```

can0 打开以后就可以使用can utils里面的小工具进行数据收发测试了。一个开发板用来接收数据，一个用来发送数据，接收数据的开发板使用candump命令，输入如下命令：

```bash
candump can0 //接收数据
```

发送数据的开发板使用cansend命令向接收单元发送8个字节的数据：0X11、0X22、0X33、0X44、0X55、0X66、0X77、0X88。输入如下命令：

```bash
cansend can0 5A1#11.22.33.44.55.66.77.88
```

cansend 命令用于发送can数据，"5A1"是帧ID，" #"号后面的"11.22.33.44.55.66.77.88"就是要发送的数据，十六进制。CAN2.0一次最多发送8个字节的数据，8个字节的数据之间用"."隔开。

如果CAN工作正常的话接收端就会接收到上面发送过来的这8个字节的数据，接收端的can0接口接收到了8个字节的数据，帧ID为5A1，说明CAN驱动工作正常。

如果要关闭can0的话输入如下命令：

```bash
ifconfig can0 down
```

##### ②、回环测试

如果要在一个板子上进行CAN回环测试，按照如下命令设置CAN：

```bash
ifconfig can0 down  //如果can0 已经打开了，先关闭
ip link set can0 type can bitrate 500000 loopback on //开启回环测试
ifconfig can0 up //重新打开can0
candump can0 & //candump 后台接收数据
cansend can0 5A1#11.22.33.44.55.66.77.88 //cansend 发送数据
```

如果回环测试成功的话那么开发板就会收到发送给自己的数据。

#### 2、USB转CAN卡测试

接下来我们使用一个USB转CAN卡与正点原子的I.MX6U ALPHA开发板进行数据收发测试，这里我使用的是周立功出品的USBCAN设备，型号为"USBCAN I+"。

首先设置开发板的can0接口，速度为500Kbit/S，命令如下：

```bash
ip link set can0 type can bitrate 500000 //设置can0，速率500Kbit
ifconfig can0 up //打开can0
candump can0 & //candump 后台接收数据
```

按照USBCAN 说明手册设置好USB CAN卡，速度设置为500Kbit，首先通过USBCAN 向ALPHA发送数据，可以看出，USBCAN发送了5帧数据帧，帧类型为标准帧，这5帧数据都是一样的，帧ID分别为0~4。开发板CAN接口工作正常的话就会接收到这5帧数据。

开发板can0接口接收到了5帧数据，和USBCAN发送的一致。 也可以通过开发板向USBCAN发送数据，输入如下命令发送数据：

```bash
cansend can0 5A1#11.22.33.44.55.66.77.88
```

USBCAN接收到数据。

关于CAN驱动就讲解到这里，如果要编写CAN总线应用的话就直接使用Linux提供的SocketCAN 接口，使用方法类似网络通信，本教程不讲解应用编程。