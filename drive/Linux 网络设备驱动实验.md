# 第六十九章 Linux 网络驱动实验

网络驱动是 linux 里面驱动三巨头之一，linux 下的网络功能非常强大，嵌入式 linux 中也常常用到网络功能。前面我们已经讲过了字符设备驱动和块设备驱动，本章我们就来学习一下 linux 里面的网络设备驱动。

## 69.1 嵌入式网络简介

### 69.1.1 嵌入式下的网络硬件接口

本章节讨论的都是有线网络！

注意！正点原子 I.MX6U-ALPHA 开发板 V2.4 版本以前的底板使用的网络 PHY 为 LAN8720，V2.4 及其以后的版本使用的网络 PHY 为 SR8201F，而且网络 PHY 地址有改变，大家一定要看准自己所使用的底板版本号！原理基本一致，本章就以 LAN8720 为例进行讲解。

提起网络，我们一般想到的硬件就是"网卡"，"网卡"这个概念最早从电脑领域传出来，顾名思义就是能上网的卡。在电脑领域的"原始社会"，网卡是独立的硬件，如果电脑要上网就得买个网卡插上去，类似现在的显卡一样。但是大家现在观察自己的笔记本或者台式机主板会发现并没有类似显卡一样的网卡设备，原因是随着技术的不断发展，现在只需要一个芯片就可以实现有线网卡功能，因此网卡芯片都直接放到了主板上。所以大家在接触嵌入式的时候听到"网卡"这两个字，不要急着在开发板上找"卡"一样的东西。

既然现在网卡已经是通过一个芯片来完成了，那么是什么样的芯片呢？这个就要先了解一下嵌入式中的网络硬件方案。首先，嵌入式网络硬件分为两部分：MAC 和 PHY，大家都是通过看数据手册来判断一款 SOC 是否支持网络，如果一款芯片数据手册说自己支持网络，一般都是说的这款 SOC 内置 MAC，MAC 类似 I2C 控制器、SPI 控制器一样的外设。但是光有 MAC 还不能直接驱动网络，还需要另外一个芯片：PHY，因此对于内置 MAC 的 SOC，其外部必须搭配一个 PHY 芯片。

但是有些 SOC 内部没有 MAC，那也就没法搭配 PHY 芯片了，这些内部没有网络 MAC 的芯片如何上网呢？这里就要牵扯出常见的两个嵌入式网络硬件方案。

**1、SOC 内部没有网络 MAC 外设**

我们一般说某个 SOC 不支持网络，说的就是它没有网络 MAC。那么这个芯片就不能上网了吗？显然不是的，既然没有内部 MAC，那么可以找个外置的 MAC 芯片啊，不过一般这种外置的网络芯片都是 MAC+PHY 一体的。比如三星 linux 开发板里面用的最多的 DM9000，因为三星的芯片基本没有内部 MAC(比如 S3C2440、S5PV210，4412 等)，所以三星的开发板都是通过外置的 DM9000 来完成有线网络功能的，DM9000 对 SOC 提供了一个 SRAM 接口，SOC 会以 SRAM 的方式操作 DM9000。

有些外置的网络芯片更强大，内部甚至集成了硬件 TCP/IP 协议栈，对外提供一个 SPI 接口，比如 W5500。这个一般用于单片机领域，单片机通过 SPI 接口与 W5500 进行通信，由于 W5500 内置了硬件 TCP/IP 协议栈，因此单片机就不需要移植软件协议栈，直接通过 SPI 来操作 W5500，简化了单片机联网方案。

这种方案的优点就是让不支持网络的 SOC 能够另辟蹊径，实现网络功能，但是缺点就是网络效率不高，因为一般芯片内置的 MAC 会有网络加速引擎，比如网络专用 DMA，网络处理效率会很高。而且此类芯片网速都不快，基本就是 10/100M。另外，相比 PHY 芯片而言，此类芯片的成本也比较高，可选择比较少。

SOC 与外部 MAC+PHY 芯片的连接如图1-1所示。

![图1-1 主控SOC 与外置 MAC+PHY 芯片连接](photo/Linux%20网络设备驱动/1-1%20主控SOC%20与外置%20MAC+PHY%20芯片连接.png)

**2、SOC 内部集成网络 MAC 外设**

我们一般说某个 SOC 支持网络，说的就是他内部集成网络 MAC 外设，此时我们还需要外接一个网络 PHY 芯片。此时就有朋友会有疑问，PHY 芯片不能也集成进 SOC 吗？笔者目前还没见过将 PHY 也集成到芯片里面的 SOC。

一般常见的通用 SOC 都会集成网络 MAC 外设，比如 STM32F4/F7/H7 系列、NXP 的 I.MX 系列，内部集成网络 MAC 的优点如下：

①、内部 MAC 外设会有专用的加速模块，比如专用的 DMA，加速网速数据的处理。

②、网速快，可以支持 10/100/1000M 网速。

③、外接 PHY 可选择性多，成本低。

内部的 MAC 外设会通过 MII 或者 RMII 接口来连接外部的 PHY 芯片，MII/RMII 接口用来传输网络数据。另外主控需要配置或读取 PHY 芯片，也就是读写 PHY 的内部寄存器，所以还需要一个控制接口，叫做 MDIO，MDIO 很类似 IIC，也是两根线，一根数据线叫做 MDIO，一根时钟线叫做 MDC。

SOC 内部 MAC 外设与外部 PHY 芯片的连接如图1-2。

![图1-2 内部 MAC 与外部 PHY 之间的连接](photo/Linux%20网络设备驱动/1-2%20内部%20MAC%20与外部%20PHY%20之间的连接.png)

如果要用到网络功能，强烈建议大家选择内部带有网络 MAC 外设的主控 SOC！I.MX6ULL 就有两个 10M/100M 的网络 MAC 外设，正点原子 ALPHA 开发板板载了两颗 PHY 芯片，型号为 LAN8720 或 SR8201F。因此，本章节只讲解 SOC 内部 MAC+外置 PHY 芯片这种方案。

### 69.1.2 MII/RMII 接口

前面我们说了，内部 MAC 通过 MII/RMII 接口来与外部的 PHY 芯片连接，完成网络数据传输，本节我们就来学习一下什么是 MII 和 RMII 接口。

**1、MII 接口**

MII 全称是 Media Independent Interface，直译过来就是介质独立接口，它是 IEEE-802.3 定义的以太网标准接口，MII 接口用于以太网 MAC 连接 PHY 芯片，如图1-3所示。

![图1-3 MII 接口](photo/Linux%20网络设备驱动/1-3%20MII%20接口.png)

MII 接口一共有 16 根信号线，含义如下：

- **TX_CLK**：发送时钟，如果网速为 100M 的话时钟频率为 25MHz，10M 网速的话时钟频率为 2.5MHz，此时钟由 PHY 产生并发送给 MAC。
- **TX_EN**：发送使能信号。
- **TX_ER**：发送错误信号，高电平有效，表示 TX_ER 有效期内传输的数据无效。10Mbps 网速下 TX_ER 不起作用。
- **TXD[3:0]**：发送数据信号线，一共 4 根。
- **RXD[3:0]**：接收数据信号线，一共 4 根。
- **RX_CLK**：接收时钟信号，如果网速为 100M 的话时钟频率为 25MHz，10M 网速的话时钟频率为 2.5MHz，RX_CLK 也是由 PHY 产生的。
- **RX_ER**：接收错误信号，高电平有效，表示 RX_ER 有效期内传输的数据无效。10Mbps 网速下 RX_ER 不起作用。
- **RX_DV**：接收数据有效，作用类似 TX_EN。
- **CRS**：载波侦听信号。
- **COL**：冲突检测信号。

MII 接口的缺点就是所需信号线太多，这还没有算 MDIO 和 MDC 这两根管理接口的数据线，因此 MII 接口使用已经越来越少了。

**2、RMII 接口**

RMII 全称是 Reduced Media Independent Interface，翻译过来就是精简的介质独立接口，也就是 MII 接口的精简版本。RMII 接口只需要 7 根数据线，相比 MII 直接减少了 9 根，极大的方便了板子布线，如图1-4所示。

![图1-4 RMII 接口](photo/Linux%20网络设备驱动/1-4%20RMII%20接口.png)

RMII 接口信号：

- **TX_EN**：发送使能信号。
- **TXD[1:0]**：发送数据信号线，一共 2 根。
- **RXD[1:0]**：接收数据信号线，一共 2 根。
- **CRS_DV**：相当于 MII 接口中的 RX_DV 和 CRS 这两个信号的混合。
- **REF_CLK**：参考时钟，由外部时钟源提供，频率为 50MHz。这里与 MII 不同，MII 的接收和发送时钟是独立分开的，而且都是由 PHY 芯片提供的。

除了 MII 和 RMII 以外，还有其他接口，比如 GMII、RGMII、SMII 等，关于其他接口基本都是大同小异的，这里就不做讲解了。正点原子 ALPHA 开发板上的两个网口都是采用 RMII 接口来连接 MAC 与外部 PHY 芯片。

### 69.1.3 MDIO 接口

MDIO 全称是 Management Data Input/Output，直译过来就是管理数据输入输出接口，是一个简单的两线串行接口，一根 MDIO 数据线，一根 MDC 时钟线。驱动程序可以通过 MDIO 和 MDC 这两根线访问 PHY 芯片的任意一个寄存器。

MDIO 接口支持多达 32 个 PHY。同一时刻内只能对一个 PHY 进行操作，那么如何区分这 32 个 PHY 芯片呢？和 IIC 一样，使用器件地址即可。同一 MDIO 接口下的所有 PHY 芯片，其器件地址不能冲突，必须保证唯一，具体器件地址值要查阅相应的 PHY 数据手册。

因此，MAC 和外部 PHY 芯片进行连接的时候主要是 MII/RMII 和 MDIO 接口，另外可能还需要复位、中断等其他引脚。

### 69.1.4 RJ45 接口

网络设备是通过网线连接起来的，插入网线的叫做 RJ45 座，RJ45 座要与 PHY 芯片连接在一起，但是中间需要一个网络变压器，网络变压器用于隔离以及滤波等，网络变压器也是一个芯片，外形一般如图1-5所示，但是现在很多 RJ45 座子内部已经集成了网络变压器，比如正点原子 ALPHA 开发板所使用的 HR911105A 就是内置网络变压器的 RJ45 座。内置网络变压器的 RJ45 座和不内置的引脚一样，但是一般不内置的 RJ45 座会短一点。因此，大家在画板的时候一定要考虑你所使用的 RJ45 座是否内置网络变压器，如果不内置的话就要自行添加网络变压器部分电路！！！同理，如果你所设计的硬件是需要内置网络变压器的 RJ45 座，肯定不能随便焊接一个不内置变压器的 RJ45 座，否则网络工作不正常！

![图1-5 网络变压器](photo/Linux%20网络设备驱动/1-5%20网络变压器.png)

RJ45 座子上一般有两个灯，一个黄色(橙色)，一个绿色，绿色亮的话表示网络连接正常，黄色闪烁的话说明当前正在进行网络通信。这两个灯由 PHY 芯片控制，PHY 芯片会有两个引脚来连接 RJ45 座上的这两个灯。内部 MAC+外部 PHY+RJ45 座(内置网络变压器)就组成了一个完整的嵌入式网络接口硬件，如图1-6所示

![图1-6 嵌入式网络硬件接口示意图](photo/Linux%20网络设备驱动/1-6%20嵌入式网络硬件接口示意图.png)

### 69.1.5 I.MX6ULL ENET 接口简介

I.MX6ULL 有两个网络接口，也就是两个 MAC 外设，一个 MAC 连接一个 PHY 芯片形成一个完整网络接口，本节我们简单了解一下 I.MX6ULL 自带的 ENET 接口。I.MX6ULL 内部自带的 ENET 外设其实就是一个网络 MAC，支持 10/100M。实现了三层网络加速，用于加速那些通用的网络协议，比如 IP、TCP、UDP 和 ICMP 等，为客户端应用程序提供加速服务。

I.MX6ULL 内核集成了两个 10/100Mbit/S 的网络 MAC，符合 IEEE802.3-2002 标准，MAC 层支持双工、半双工局域网。MAC 可编程、可以作为 NIC 卡或其他一些交换器件。根据 IETF RFC 2819 协议，MAC 实现了 RMON(Remote Network Monitoring)计数功能。MAC 内核拥有硬件加速处理单元来提高网络性能，硬件加速单元用于处理 TCP/IP、UDP、ICMP 等协议。通过硬件来处理帧头等信息，效果要比用一大堆软件处理要好很多。ENET 外设有一个专用的 DMA，此 DMA 用于在 ENET 外设和 SOC 之间传输数据，并且支持可编程的增强型的缓冲描述符，用以支持 IEEE 1588。

I.MX6ULL 内部 ENET 外设主要特性如下：

1)、实现了全功能的 802.3 规范前导码/SFD 生成、帧填充、CRC 生成和检查。

2)、支持零长的前导码。

3)、支持 10/100M 动态配置。

4)、兼容 AMD 远端节点电源管理的魔术帧中断检测。

5)、可以通过如下接口无缝的连接 PHY 芯片：
- 4bit 的 MII 接口，频率为 2.5/25MHz。
- 4bit 的 MII-Lite 接口，也就是 MII 接口取消掉 CRS 和 COL 这两根线，频率也是 2.5/25MHz。
- 2bit 的 RMII 接口，频率为 50MHz。

6)、MAC 地址可编程。

7)、多播和单播地址过滤，降低更高层的处理负担。

8)、MDIO 主接口，用于管理和配置 PHY 设备。

……

I.MX6ULL 的 ENET 外设内容比较多，详细的介绍请查阅《I.MX6ULL 参考手册》的"Chapter 22 10/100-Mbps Ethernet MAC(ENET)"章节。我们在编写驱动的时候其实并不需要关注 ENET 外设的具体内容，因为这部分驱动是 SOC 厂商编写的，我们重点关注的是更换 PHY 芯片以后哪里需要调整。

## 69.2 PHY 芯片详解

### 69.2.1 PHY 基础知识简介

PHY 是 IEEE 802.3 规定的一个标准模块，前面说了，SOC 可以对 PHY 进行配置或者读取 PHY 相关状态，这个就需要 PHY 内部寄存器去实现了。PHY 芯片寄存器地址空间为 5 位，地址 0~31 共 32 个寄存器，IEEE 定义了 0~15 这 16 个寄存器的功能，16~31 这 16 个寄存器由厂商自行实现。也就是说不管你用的哪个厂家的 PHY 芯片，其中 0~15 这 16 个寄存器是一模一样的。仅靠这 16 个寄存器是完全可以驱动起 PHY 芯片的，至少能保证基本的网络数据通信，因此 Linux 内核有通用 PHY 驱动，按道理来讲，不管你使用的哪个厂家的 PHY 芯片，都可以使用 Linux 的这个通用 PHY 驱动来验证网络工作是否正常。事实上在实际开发中可能会遇到一些其他的问题导致 Linux 内核的通用 PHY 驱动工作不正常，这个时候就需要驱动开发人员去调试了。但是，随着现在的 PHY 芯片性能越来越强大，32 个寄存器可能满足不了厂商的需求，因此很多厂商采用分页技术来扩展寄存器地址空间，以求定义更多的寄存器。这些多出来的寄存器可以用于实现厂商特有的一些技术，因此 Linux 内核的通用 PHY 驱动就无法驱动这些特色功能了，这个时候就需要 PHY 厂商提供相应的驱动源码了，所以大家也会在 Linux 内核里面看到很多具体的 PHY 芯片驱动源码。不管你的 PHY 芯片有多少特色功能，按道理来讲，Linux 内核的通用 PHY 驱动是绝对可以让你这 PHY 芯片实现基本的网络通信，因此大家也不用担心更换 PHY 芯片以后网络驱动编写是不是会很复杂。

IEEE802.3 协议英文原版已经放到了开发板光盘中，路径为 4、参考资料->802.3 协议英文原版_2018 年.pdf，打开此文档，此文档有 5600 页，按照 SECTION 进行分类，一共 8 个 SECTION。选中"802.3-2018_SECTION2"，找到"22.2.4 Management functions"章节，此章节对 PHY 的前 16 个寄存器功能进行了规定，如图1-7所示

![图1-7 IEEE 规定的前 16 个寄存器](photo/Linux%20网络设备驱动/1-7%20IEEE%20规定的前%2016%20个寄存器.png)

关于这 16 个寄存器的内容协议里面也进行了详细的讲解，这里就不分析了。我们后面会以 ALPHA 开发板所使用的 LAN8720A/SR8201F 这两个 PHY 为例，详细的分析一下 PHY 芯片的寄存器。

### 69.2.2 LAN8720A 详解

V2.4 版本以前的底板 PHY 芯片采用 LAN8720A，因此请看本小节！虽然本教程讲解的是 LAN8720A 这颗 PHY，但是前面说了，IEEE 规定了 PHY 的前 16 个寄存器的功能，因此如果你所使用的板子用的其他厂家的 PHY 芯片，也是可以看本小节的。

**1、LAN8720A 简介**

LAN8720A 是低功耗的 10/100M 单以太网 PHY 层芯片，可应用于机顶盒、网络打印机、嵌入式通信设备、IP 电话等领域。I/O 引脚电压符合 IEEE802.3-2005 标准。LAN8720A 支持通过 RMII 接口与以太网 MAC 层通信，内置 10-BASE-T/100BASE-TX 全双工传输模块，支持 10Mbps 和 100Mbps。LAN8720A 可以通过自协商的方式选择与目的主机最佳的连接方式(速度和双工模式)。支持 HP Auto-MDIX 自动翻转功能，无需更换网线即可将连接更改为直连或交叉连接。

LAN8720A 的主要特点如下：

- 高性能的 10/100M 以太网传输模块
- 支持 RMII 接口以减少引脚数
- 支持全双工和半双工模式
- 两个状态 LED 输出
- 可以使用 25M 晶振以降低成本
- 支持自协商模式
- 支持 HP Auto-MDIX 自动翻转功能
- 支持 SMI 串行管理接口
- 支持 MAC 接口

LAN8720A 功能框图如图1-8

![图1-8 LAN8720A功能框图](photo/Linux%20网络设备驱动/1-8%20LAN8720A功能框图.png)

**2、LAN8720A 中断管理**

LAN8720A 的器件管理接口支持非 IEEE 802.3 规范的中断功能。当一个中断事件发生并且相应事件的中断位使能，LAN8720A 就会在 nINT(14 脚)产生一个低电平有效的中断信号。LAN8720A 的中断系统提供两种中断模式：主中断模式和复用中断模式。主中断模式是默认中断模式，LAN8720A 上电或复位后就工作在主中断模式，当模式控制/状态寄存器(十进制地址为 17)的 ALTINT 位为 0 时 LAN8720A 工作在主模式，当 ALTINT 位为 1 时工作在复用中断模式。正点原子的 ALPHA 开发板虽然讲 LAN8720A 的中断引脚连接到了 I.MX6ULL 上，但是并没有使用中断功能，关于中断的具体用法可以参考 LAN8720A 数据手册的 29~30 页。

**3、PHY 地址设置**

MAC 层通过 MDIO/MDC 总线对 PHY 进行读写操作，MDIO 最多可以控制 32 个 PHY 芯片，通过不同的 PHY 芯片地址来对不同的 PHY 操作。LAN8720A 通过设置 RXER/PHYAD0 引脚来设置其 PHY 地址，默认情况下为 0，其地址设置如表 69.2.2.1 所示。

表 69.2.2.1 LAN8720A 地址设置

| RXER/PHYAD0 引脚状态 | PHY 地址 |
| --- | --- |
| 上拉 | 0X01 |
| 下拉(默认) | 0X00 |

正点原子 ALPHA 开发板的 ENET1 网络的 LAN8720A 上的 RXER/PHYAD0 引脚为默认状态(原理图上有个 10K 下拉，但是没有焊接)，因此 ENET1 上的 LAN8720A 地址为 0。ENET2 网络上的 LAN8720A 上的 RXER/PHYAD0 引脚接了个 10K 上拉电阻，因此 ENET2 上的 LAN8720A 地址为 1。

**4、nINT/REFCLKO 配置**

nINTSEL 引脚(2 号引脚)用于设置 nINT/REFCLKO 引脚(14 号引脚)的功能。nINTSEL 配置如表 69.2.2.2 所示。

表 69.2.2.2 nINTSEL 配置

| nINTSEL 引脚值 | 模式 | nINT/REFCLKO 引脚功能 |
| --- | --- | --- |
| nINTSEL = 0 | REF_CLK Out 模式 | nINT/REFCLKO 作为 REF_CLK 时钟源 |
| nINTSEL = 1(默认) | REF_CLK In 模式 | nINT/REFCLKO 作为中断引脚 |

对于正点原子的 ALPHA 开发板的两个 LAN8720A 而言，全都工作在默认的 REF_CLK In 模式下。当 LAN8720A 工作在 REF_CLK In 模式时，50MHz 的外部时钟信号应接到 LAN8720 的 XTAL1/CKIN 引脚(5 号引脚)上，如图1-9所示

![图1-9 REF_CLK 连接外部 50MHz 时钟信号](photo/Linux%20网络设备驱动/1-9%20REF_CLK%20连接外部%2050MHz%20时钟信号.png)

为了降低成本，LAN8720A 可以从外部的 25MHz 的晶振中产生 REF_CLK 时钟。使用此功能时应工作在 REF_CLK Out 模式时。当工作在 REF_CLK Out 模式时 REF_CLK 的时钟源如图1-10所示

![图1-10 REF_CLK Out 模式时的 REF_CLK 时钟源](photo/Linux%20网络设备驱动/1-10%20REF_CLK%20Out%20模式时的%20REF_CLK%20时钟源.png)

正点原子的 ALPHA 开发板工作在 REF_CLK In 模式下，因此需要外部 50MHz 时钟信号，I.MX6ULL 有专用的网络时钟引脚，因此 ALPHA 开发板是通过 I.MX6ULL 的 ENET1_REF_CLK 和 ENET2_REF_CLK 这两个网络时钟引脚来为 LAN8720A 提供 50MHz 的时钟。

**5、LAN8720A 内部寄存器**

LAN8720A 的前 16 个寄存器满足 IEEE 的要求，在这里我们只介绍几个常用的寄存器，首先是 BCR(Basic Control Register)寄存器，地址为 0，BCR 寄存器各位如表 69.2.2.3 所示。

表 69.2.2.3 BCR 寄存器

| 位 | 描述 | 类型 |
| --- | --- | --- |
| 15 | 软件复位<br>1：软件复位，此位自动清零 | R/W |
| 14 | 回测<br>0：正常运行<br>1：使能回测模式 | R/W |
| 13 | 速度选择<br>0：10Mbps<br>1：100Mbps<br>注意：当使用自动协商功能时此位失能 | R/W |
| 12 | 自动协商功能<br>0：关闭自动协商功能<br>1：打开自动协商功能 | R/W |
| 11 | 掉电（power down）<br>0：正常运行<br>1：进入掉电模式<br>注意:进入掉电模式前自动协商必须失能 | R/W |
| 10 | 隔离<br>0：正常运行<br>1：PHY 的 RMII 接口电气隔离 | R/W |
| 9 | 重启自动协商功能<br>0：正常运行<br>1：重启自动协商功能<br>注意:此位会被自动清零 | R/W |
| 8 | 双工模式<br>0：半双工<br>1：全双工<br>注意:开启自动协商功能后此位失效 | R/W |
| 7:0 | 保留 | RO |

我们说的配置 PHY 芯片，重点就是配置 BCR 寄存器，由于 LAN8720A 是个 10/100M 的 PHY，因此表 69.2.2.3 中没有体现出 1000M 相关的配置。但是 10/100M 相关的配置是和 IEEE 的规定完全相符的，大家可以选择一个其他的 10/100M 的 PHY 芯片对比看一下，比如 NXP 官方 EVK 开发板使用的 KSZ8081。

接下来看一下 BSR(Basic Status Register)寄存器，地址为 1。此寄存器为 PHY 的状态寄存器，通过此寄存器可以获取到 PHY 芯片的工作状态，BSR 寄存器各位如表 69.2.2.4 所示：

表 69.2.2.4 BSR 寄存器

| 位 | 描述 | 类型 |
| --- | --- | --- |
| 15 | 100BAST-T4<br>0：不支持 T4<br>1：支持 T4 | RO |
| 14 | 100BAST-TX 全双工<br>0：不支持 TX 全双工<br>1：支持 TX 全双工 | RO |
| 13 | 100BAST-TX 半双工<br>0：不支持 TX 半双工<br>1：支持 TX 半双工 | RO |
| 12 | 10BAST-T 全双工<br>0：不支持 10Mbps 全双工<br>1：支持 10Mbps 全双工 | RO |
| 11 | 10BAST-T 半双工<br>0：不支持 10Mbps 半双工<br>1：支持 10Mbps 半双工 | RO |
| 10:6 | 保留 | RO |
| 5 | 自协商功能完成<br>0：自动协商未完成<br>1：自动协商完成 | RO |
| 4 | 远端错误<br>0：无远端错误<br>1：检测到远端错误 | RO/HL |
| 3 | 自协商功能<br>0：不能执行自协商功能<br>1：可以执行自协商功能 | RO |
| 2 | 连接状态<br>0：连接断开<br>1：连接建立 | RO/LL |
| 1 | Jabber 检测<br>0：未检测到 jabber<br>1：检测到 jabber | RO/LH |
| 0 | 扩展功能<br>0：不支持扩展寄存器<br>1：支持扩展寄存器 | RO |

从表 69.2.2.4 可以看出，和 IEEE 标准规定相比，LAN8720A 的 BSR 寄存器少了几个位，这个没关系的，不管什么 PHY 芯片，只要它实现了的位和 IEEE 规定相符就行。通过读取 BSR 寄存器的值我们可以得到当前的连接速度、双工状态和连接状态等。
<!-- 接上文 69.2.2 LAN8720A 内部寄存器（BCR/BSR之后） -->

接下来看一下 LAN8720A 的 PHY ID 寄存器 1 和 ID 寄存器 2，地址为 2 和 3，后面就称为寄存器 2 和寄存器 3。这两个寄存器都是 PHY 的 ID 寄存器。IEEE 规定寄存器 2 和寄存器 3 为 PHY 的 ID 寄存器，这两个寄存器组成一个 32 位的唯一 ID 值。IEEE 规定了一叫做 OUI 的 ID 组成方式，全称是 Organizationally Unique Identifier，OUI 一共 32 位，分为三部分：22 位的 ID+6 位厂商型号 ID+4 位厂商版本 ID，组成如图1-11

![图1-11 OUI组成方式](photo/Linux%20网络设备驱动/1-11%20OUI组成方式.png)

LAN8720A 的 ID 寄存器 2 如表69.2.2.5 所示：

表 69.2.2.5 PHY ID 寄存器 2

| 位 | 描述 | 类型 | 默认值 |
| --- | --- | --- | --- |
| 15:0 | PHY ID 号，对应于 OUI 的 3~18 位，bit15 对应 OUI 的 bit3，bit0 对应 OUI 的 bit18。和寄存器 3 的 bit15：10 共同组成 22 位 ID。 | R/W | 0007h |

ID 寄存器 3 如表 69.2.2.6 所示：

表 69.2.2.6 PHY ID 寄存器 3

| 位 | 描述 | 类型 | 默认值 |
| --- | --- | --- | --- |
| 15:10 | PHY ID 号，对应于 OUI 的 19~24 位，bit15 对应 OUI 的 19 位，bit10 对应 OUI 的 24 位。 | R/W | 110000h |
| 9:4 | 厂商型号 ID | R/W | 001111b |
| 3:0 | 厂商版本 ID | R/W | 视具体版本而定 |

最后来看一下 LAN8720A 的特殊控制/状态寄存器，此寄存器地址为 31，寄存器内容是 LAN8720A 厂商自定义的，此寄存器的各个位如表 69.2.2.7 所示:

表 69.2.2.7 LAN8720A 特殊控制/状态寄存器

| 位 | 描述 | 类型 |
| --- | --- | --- |
| 15:13 | 保留 | RO |
| 12 | 自协商完成<br>0：自协商未完成或者自协商关闭<br>1：自协商完成 | RO |
| 11:5 | 保留 | R/W |
| 4:2 | 速度指示<br>001：10BASE-T 半双工<br>101：10BAST-T 全双工<br>010：100BAST-TX 半双工<br>110：100BAST-TX 全双工 | RO |
| 1:0 | 保留 | RO |

特殊控制/状态寄存器中我们关心的是 bit2~bit4 这三位，因为通过这 3 位来确定连接的状态和速度，关于 LAN8720A 这个 PHY 就讲解到这里。

---

### 69.2.3 SR8201F 详解

V2.4 及以后版本的底板 PHY 芯片采用 SR8201F，因此请看本小节！

**1、SR8201F 简介**

SR8201F 是单芯片/单端口的 10/100Mbps 以太网 PHY 收发器。芯片支持 RMII (简化介质独立)接口，SR8201F 实现了所有 10/100M 以太网物理层，包括：物理编码子层 (PCS)，物理介质接入层(PMA)，双绞线物理介质相关子层 (TP-PMD)，同时芯片支持端口自动翻转（Auto-MDIX）。

SR8201F 的主要特点如下：
· 支持 IEEE 802.3az-2010（EEE）
· 兼容 100Base-TX IEEE 802.3u
· 兼容 10Base-T IEEE 802.3
· 支持 RMII 模式
· 全/半双工工作
· 支持自协商模式
· 支持 Auto-MDIX
· 支持中断功能
· SR8201F 提供两个网络状态的 LED
· 支持 25MHz 晶振或外部振荡输入
· RMII 模式支持 50MHz 外部时钟输入

SR8201F 功能框图如图2-1所示

![图2-1 SR8201F 功能框图](photo/Linux%20网络设备驱动/2-1%20SR8201F%20功能框图.png)

**2、SR8201F 中断管理**

SR8201F 检测到媒介端的状态改变，就会写入对应中断状态的寄存器（第 0 页寄存器 14），并且在中断管脚使能时，中断管脚会被置低表示发生一个中断事件。当 MAC 检测到中断事件时，可以通过 MDC/MDIO 端口访问到第 0 页寄存器 14 获取相应的中断状态。正点原子的 ALPHA 开发板虽然将 SR8201F 的中断引脚连接到了 I.MX6ULL 上，但是并没有使用中断功能，关于中断的具体用法可以参考 SR8201F 数据手册。

**3、PHY 地址设置**

MAC 层通过 MDIO/MDC 总线对 PHY 进行读写操作，MDIO 最多可以控制 32 个 PHY 芯片，通过不同的 PHY 芯片地址来对不同的 PHY 操作。SR8201F 通过设置 LED0/PHYAD[0]和 LED1/PHYAD[1]这两个引脚来设置其 PHY 地址，其地址设置如表 69.2.3.1 所示。

表 69.2.3.1 SR8201F 地址设置

| LED1/PHYAD1(IPD) | LED0/PHYAD0(IPD) | PHY 地址 |
| --- | --- | --- |
| 0 | 0 | 0X00 |
| 0 | 1 | 0X01 |
| 1 | 0 | 0X02 |
| 1 | 1 | 0X03 |

正点原子 ALPHA 开发板 ENET1 网络的 SR8201F 上的 LED1/PHYAD1 引脚上拉，LED0/PHYAD0 引脚下拉，因此 ENET1 上的 SR8201F 地址为 0X02。ENET2 网络上的 SR8201F 的 LED1/PHYAD1 引脚下拉，LED0/PHYAD0 引脚上拉，因此 ENET2 上的 SR8201F 地址为 1。

**4、SR8201F 内部寄存器**

SR8201F 的前 16 个寄存器满足 IEEE 的要求，在这里我们只介绍几个常用的寄存器，首先是 BCR(Basic Control Register)寄存器，地址为 0，BCR 寄存器各位如表 69.2.3.2 所示。

表 69.2.3.2 BCR 寄存器

| 位 | 描述 | 类型 | 默认值 |
| --- | --- | --- | --- |
| 15 | 软件复位<br>此位带自清除功能，复位完成后恢复为 0。<br>1:软件复位 0:正常工作<br>软件复位（设置 bit15 为 1）会将寄存器 0 和寄存器 1 恢复为默认值，同时此行为将改变内部 PHY 的状态和与 PHY 物理连接设备的相关状态 | R/W | 0 |
| 14 | loopback<br>设置发送到接收数据路径。<br>1：启用 loopback 0：正常工作 | R/W | 0 |
| 13 | 网络速度<br>1:100Mbps 0:10Mbps<br>在完成自协商完成后，此位表示当前速度状态 | R/W | 1 |
| 12 | 自协商功能<br>1：启用自协商；bits13 和 8 将被忽略；<br>0：禁用自协商；bits13 和 8 将确定链路 | R/W | 1 |
| 11 | 掉电（power down）<br>1：电源休眠 0：正常工作 | R/W | 0 |
| 10 | 隔离<br>1：从 MII/RMII 接口隔离 PHY<br>0：正常工作 | R/W | 0 |
| 9 | 重启自协商<br>1：重启自协商 0：正常工作 | SC | 0 |
| 8 | 双工模式：<br>1：全双工 0：半双工<br>自协商使能且在自协商完成之后，此位指示双工状态：<br>1：全双工 0：半双工 | R/W | 1 |
| 7 | 碰撞测试<br>1：碰撞测试启用<br>0：正常运行 | RO | 0 |
| 6~0 | 保留 | - | - |

我们说的配置 PHY 芯片，重点就是配置 BCR 寄存器，由于 SR8201F 是个 10/100M 的 PHY，因此表 69.2.3.2 中没有体现出 1000M 相关的配置。但是 10/100M 相关的配置是和 IEEE 的规定完全相符的，大家可以选择一个其他的 10/100M 的 PHY 芯片对比看一下，比如 NXP 官方 EVK 开发板使用的 KSZ8081。

接下来看一下 BSR(Basic Status Register)寄存器，地址为 1。此寄存器为 PHY 的状态寄存器，通过此寄存器可以获取到 PHY 芯片的工作状态，BSR 寄存器各位如表 69.2.3.4 所示：

表 69.2.3.4 BSR 寄存器

| 位 | 描述 | 类型 | 默认值 |
| --- | --- | --- | --- |
| 15 | 100BAST-T4<br>1：支持启用 100Base-T4<br>0：不支持 100Base-T4 | RO | 0 |
| 14 | 100BAST-TX 全双工<br>1：支持启用 100Base-TX 全双工<br>0：不支持 100Base-TX 全双工 | RO | 1 |
| 13 | 100BAST-TX 半双工<br>1：支持启用 100Base-TX 半双工<br>0：不支持 100Base-TX 半双工 | RO | 1 |
| 12 | 10BAST-T 全双工<br>1：支持启用 10Base-T 全双工<br>0：不支持 10Base-T 全双工 | RO | 1 |
| 11 | 10BAST-T 半双工<br>1：支持启用 10Base-T 半双工<br>0：不支持 10Base-T 半双工 | RO | 1 |
| 10~7 | 保留 | - | - |
| 6 | SR8201F/FL/FN 允许接收前导抑制的管理帧。 | RO | 1 |
| 5 | 自协商功能完成<br>1:自协商过程完成<br>0:自协商过程未完成 | RO | 0 |
| 4 | 远端错误<br>1:检测到远程错误情况（读取清除）<br>0:未检测到远程错误情况 | RO/HL | 0 |
| 3 | 自协商功能<br>1：PHY 能执行自协商<br>0：PHY 不能执行自协商 | RO | 1 |
| 2 | 连接状态<br>1：已建立有效链接<br>0：未建立有效链接 | RO | 0 |
| 1 | Jabber 检测<br>1：检测到 Jabber 条件<br>0：未检测到 jabber 条件 | RO | 0 |
| 0 | 扩展功能<br>1：延伸寄存器能力<br>0：无延伸寄存器能力 | RO | 1 |

从表 69.2.3.4 可以看出，和 IEEE 标准规定相比，SR8201F 的 BSR 寄存器少了几个位，这个没关系的，不管什么 PHY 芯片，只要它实现了的位和 IEEE 规定相符就行。通过读取 BSR 寄存器的值我们可以得到当前的连接速度、双工状态和连接状态等。

接下来看一下 SR8201F 的 PHY ID 寄存器 1 和 ID 寄存器 2，地址为 2 和 3，后面就称为寄存器 2 和寄存器 3。这两个寄存器都是 PHY 的 ID 寄存器。IEEE 规定寄存器 2 和寄存器 3 为 PHY 的 ID 寄存器，这两个寄存器组成一个 32 位的唯一 ID 值。

SR8201F 的 ID 寄存器 2 如表 69.2.3.5 所示：

表 69.2.3.5 PHY ID 寄存器 2

| 位 | 描述 | 类型 | 默认值 |
| --- | --- | --- | --- |
| 15~0 | 分配到 OUI 的第 6 到第 21 位组织唯一识别符。 | RO | 001Ch |

ID 寄存器 3 如表 69.2.3.6 所示：

表 69.2.3.6 PHY ID 寄存器 3

| 位 | 描述 | 类型 | 默认值 |
| --- | --- | --- | --- |
| 15~10 | 分配到 OUI 的第 0 到 5 位。 | RO | 110010 |
| 9~4 | 型号 | RO | 000001 |
| 3~0 | 版本号 | RO | 0110 |

关于 SR8201F 的 PHY 寄存器就简单讲这几个，其他的寄存器请自行查阅 SR8201F 数据手册。

---

## 69.3 Linux 内核网络驱动框架

### 69.3.1 net_device 结构体

Linux 内核使用 net_device 结构体表示一个具体的网络设备，net_device 是整个网络驱动的灵魂。网络驱动的核心就是初始化 net_device 结构体中的各个成员变量，然后将初始化完成以后的 net_device 注册到 Linux 内核中。net_device 结构体定义在 include/linux/netdevice.h 中，net_device 是一个庞大的结构体，内容如下(有缩减)：

示例代码 69.3.1.1 net_device 结构体
```c
1 struct net_device {
2 char	name[IFNAMSIZ];
3 struct hlist_node name_hlist;
4 char	*ifalias;
5 /*
6 * I/O specific fields
7 * FIXME: Merge these and struct ifmap into one
8 */
9 unsigned long	mem_end;
10 unsigned long	mem_start;
11 unsigned long	base_addr;
12 int	irq;
13
14 atomic_t	carrier_changes;
15
16 /*
17 * Some hardware also needs these fields (state,dev_list,
18 * napi_list,unreg_list,close_list) but they are not
19 * part of the usual set specified in Space.c.
20 */
21
22 unsigned long state;
23
24 struct list_head dev_list;
25 struct list_head napi_list;
26 struct list_head unreg_list;
27 struct list_head close_list;
......
60 const struct net_device_ops *netdev_ops;
61 const struct ethtool_ops	*ethtool_ops;
62 #ifdef CONFIG_NET_SWITCHDEV
63 const struct swdev_ops	*swdev_ops;
64 #endif
65
66 const struct header_ops	*header_ops;
67
68 unsigned int flags;
......
77 unsigned char if_port;
78 unsigned char dma;
79
80 unsigned int mtu;
81 unsigned short type;
82 unsigned short hard_header_len;
83
84 unsigned short needed_headroom;
85 unsigned short needed_tailroom;
86
87 /* Interface address info. */
88 unsigned char perm_addr[MAX_ADDR_LEN];
89 unsigned char addr_assign_type;
90 unsigned char addr_len;
......
130 /*
131 * Cache lines mostly used on receive path (including eth_type_trans())
132 */
133 unsigned long last_rx;
134
135 /* Interface address info used in eth_type_trans() */
136 unsigned char *dev_addr;
137
138
139 #ifdef CONFIG_SYSFS
140 struct netdev_rx_queue *_rx;
141
142 unsigned int num_rx_queues;
143 unsigned int real_num_rx_queues;
144
145 #endif
......
158 /*
159 * Cache lines mostly used on transmit path
160 */
161 struct netdev_queue *_tx ____cacheline_aligned_in_smp;
162 unsigned int num_tx_queues;
163 unsigned int real_num_tx_queues;
164 struct Qdisc *qdisc;
165 unsigned long tx_queue_len;
166 spinlock_t	tx_global_lock;
167 int	watchdog_timeo;
......
173 /* These may be needed for future network-power-down code. */
174
175 /*
176 * trans_start here is expensive for high speed devices on SMP,
177 * please use netdev_queue->trans_start instead.
178 */
179 unsigned long trans_start;
......
248 struct phy_device	*phydev;
249 struct lock_class_key *qdisc_tx_busylock;
250 };
```

下面介绍一些关键的成员变量，如下：
第 2 行：name 是网络设备的名字。
第 9 行：mem_end 是共享内存结束地址。
第 10 行：mem_start 是共享内存起始地址。
第 11 行：base_addr 是网络设备 I/O 地址。
第 12 行：irq 是网络设备的中断号。
第 24 行：dev_list 是全局网络设备列表。
第 25 行：napi_list 是 napi 网络设备的列表入口。
第 26 行：unreg_list 是注销(unregister)的网络设备列表入口。
第 27 行：close_list 是关闭的网络设备列表入口。
第 60 行：netdev_ops 是网络设备的操作集函数，包含了一系列的网络设备操作回调函数，类似字符设备中的 file_operations，稍后会讲解 netdev_ops 结构体。
第 61 行：ethtool_ops 是网络管理工具相关函数集，用户空间网络管理工具会调用此结构体中的相关函数获取网卡状态或者配置网卡。
第 66 行：header_ops 是头部的相关操作函数集，比如创建、解析、缓冲等。
第 68 行：flags 是网络接口标志，标志类型定义在 include/uapi/linux/if.h 文件中，为一个枚举类型，内容如下：

示例代码 69.3.1.2 网络标志类型
```c
1 enum net_device_flags {
2	IFF_UP		= 1<<0, /* sysfs */
3	IFF_BROADCAST 	= 1<<1, /* volatile */
4	IFF_DEBUG 	= 1<<2, /* sysfs */
5	IFF_LOOPBACK 	= 1<<3, /* volatile */
6	IFF_POINTOPOINT = 1<<4, /* volatile */
7	IFF_NOTRAILERS = 1<<5, /* sysfs */
8	IFF_RUNNING 	= 1<<6, /* volatile */
9	IFF_NOARP 	= 1<<7, /* sysfs */
10	IFF_PROMISC 	= 1<<8, /* sysfs */
11	IFF_ALLMULTI 	= 1<<9, /* sysfs */
12	IFF_MASTER 	= 1<<10, /* volatile */
13	IFF_SLAVE 	= 1<<11, /* volatile */
14	IFF_MULTICAST 	= 1<<12, /* sysfs */
15	IFF_PORTSEL 	= 1<<13, /* sysfs */
16	IFF_AUTOMEDIA 	= 1<<14, /* sysfs */
17	IFF_DYNAMIC 	= 1<<15, /* sysfs */
18	IFF_LOWER_UP 	= 1<<16, /* volatile */
19	IFF_DORMANT 	= 1<<17, /* volatile */
20	IFF_ECHO 	= 1<<18, /* volatile */
21 };
```

继续回到示例代码 69.3.1.1 接着看 net_device 结构体。
第 77 行：if_port 指定接口的端口类型，如果设备支持多端口的话就通过 if_port 来指定所使用的端口类型。可选的端口类型定义在 include/uapi/linux/netdevice.h 中，为一个枚举类型，如下所示：

示例代码 69.3.1.3 端口类型
```c
1 enum {
2	IF_PORT_UNKNOWN = 0,
3	IF_PORT_10BASE2,
4	IF_PORT_10BASET,
5 	IF_PORT_AUI,
6 	IF_PORT_100BASET,
7 	IF_PORT_100BASETX,
8 	IF_PORT_100BASEFX
9 };
```

第 78 行：dma 是网络设备所使用的 DMA 通道，不是所有的设备都会用到 DMA。
第 80 行：mtu 是网络最大传输单元，为 1500。
第 81 行：type 用于指定 ARP 模块的类型，以太网的 ARP 接口为 ARPHRD_ETHER，Linux 内核所支持的 ARP 协议定义在 include/uapi/linux/if_arp.h 中，大家自行查阅。
第 88 行：perm_addr 是永久的硬件地址，如果某个网卡设备有永久的硬件地址，那么就会填充 perm_addr。
第 90 行：addr_len 是硬件地址长度。
第 133 行：last_rx 是最后接收的数据包时间戳，记录的是 jiffies。
第 136 行：dev_addr 也是硬件地址，是当前分配的 MAC 地址，可以通过软件修改。
第 140 行：_rx 是接收队列。
第 142 行：num_rx_queues 是接收队列数量，在调用 register_netdev 注册网络设备的时候会分配指定数量的接收队列。
第 143 行：real_num_rx_queues 是当前活动的队列数量。
第 161 行：_tx 是发送队列。
第 162 行：num_tx_queues 是发送队列数量，通过 alloc_netdev_mq 函数分配指定数量的发送队列。
第 163 行：real_num_tx_queues 是当前有效的发送队列数量。
第 179 行：trans_start 是最后的数据包发送的时间戳，记录的是 jiffies。
第 248 行：phydev 是对应的 PHY 设备。

#### 1、申请 net_device

编写网络驱动的时候首先要申请 net_device，使用 alloc_netdev 函数来申请 net_device，这是一个宏，宏定义如下：

示例代码 69.3.1.4 alloc_netdev
```c
1 #define alloc_netdev(sizeof_priv, name, name_assign_type, setup) \
2 alloc_netdev_mqs(sizeof_priv, name, name_assign_type, setup, 1, 1)
```

可以看出 alloc_netdev 的本质是 alloc_netdev_mqs 函数，此函数原型如下
```c
struct net_device * alloc_netdev_mqs ( int sizeof_priv,
const char *name,
void (*setup) (struct net_device *),
unsigned int txqs,
unsigned int rxqs);
```

函数参数和返回值含义如下：
sizeof_priv：私有数据块大小。
name：设备名字。
setup：回调函数，初始化设备的设备后调用此函数。
txqs：分配的发送队列数量。
rxqs：分配的接收队列数量。
返回值：如果申请成功的话就返回申请到的 net_device 指针，失败的话就返回 NULL。

事实上网络设备有多种，大家不要以为就只有以太网一种。Linux 内核内核支持的网络接口有很多，比如光纤分布式数据接口(FDDI)、以太网设备(Ethernet)、红外数据接口(InDA)、高性能并行接口(HPPI)、CAN 网络等。内核针对不同的网络设备在 alloc_netdev 的基础上提供了一层封装，比如我们本章讲解的以太网，针对以太网封装的 net_device 申请函数是 alloc_etherdev，这也是一个宏，内容如下：

示例代码 69.3.1.5 alloc_etherdev 函数
```c
1 #define alloc_etherdev(sizeof_priv) alloc_etherdev_mq(sizeof_priv, 1)
2 #define alloc_etherdev_mq(sizeof_priv, count) alloc_etherdev_mqs(sizeof_priv, count, count)
```

可以看出，alloc_etherdev 最终依靠的是 alloc_etherdev_mqs 函数，此函数就是对 alloc_netdev_mqs 的简单封装，函数内容如下：

示例代码 69.3.1.6 alloc_etherdev_mqs 函数
```c
1 struct net_device *alloc_etherdev_mqs(int sizeof_priv,
2 unsigned int txqs,
3 unsigned int rxqs)
4 {
5 return alloc_netdev_mqs(sizeof_priv, "eth%d", NET_NAME_UNKNOWN,
6 ether_setup, txqs, rxqs);
7 }
```

第 5 行调用 alloc_netdev_mqs 来申请 net_device，注意这里设置网卡的名字为“eth%d”，这是格式化字符串，大家进入开发板的 linux 系统以后看到的“eth0”、“eth1”这样的网卡名字就是从这里来的。同样的，这里设置了以太网的 setup 函数为 ether_setup，不同的网络设备其 setup 函数不同，比如 CAN 网络里面 setup 函数就是 can_setup。

ether_setup 函数会对 net_device 做初步的初始化，函数内容如下所示：

示例代码 69.3.1.7 ether_setup 函数
```c
1 void ether_setup(struct net_device *dev)
2 {
3	dev->header_ops = &eth_header_ops;
4	dev->type	= ARPHRD_ETHER;
5	dev->hard_header_len = ETH_HLEN;
6	dev->mtu	= ETH_DATA_LEN;
7	dev->addr_len = ETH_ALEN;
8	dev->tx_queue_len = 1000; /* Ethernet wants good queues */
9	dev->flags	= IFF_BROADCAST|IFF_MULTICAST;
10	dev->priv_flags |= IFF_TX_SKB_SHARING;
11
12	eth_broadcast_addr(dev->broadcast);
13 }
```

关于 net_device 的申请就讲解到这里，对于网络设备而言，使用 alloc_etherdev 或 alloc_etherdev_mqs 来申请 net_device。NXP 官方编写的网络驱动就是采用 alloc_etherdev_mqs 来申请 net_device。

#### 2、删除 net_device

当我们注销网络驱动的时候需要释放掉前面已经申请到的 net_device，释放函数为 free_netdev，函数原型如下：
```c
void free_netdev(struct net_device *dev)
```

函数参数和返回值含义如下：
dev：要释放掉的 net_device 指针。
返回值：无。

#### 3、注册 net_device

net_device 申请并初始化完成以后就需要向内核注册 net_device，要用到函数 register_netdev，函数原型如下：
```c
int register_netdev(struct net_device *dev)
```

函数参数和返回值含义如下：
dev：要注册的 net_device 指针。
返回值：0 注册成功，负值 注册失败。

#### 4、注销 net_device

既然有注册，那么必然有注销，注销 net_device 使用函数 unregister_netdev，函数原型如下：
```c
void unregister_netdev(struct net_device *dev)
```

函数参数和返回值含义如下：
dev：要注销的 net_device 指针。
返回值：无。

---

### 69.3.2 net_device_ops 结构体

net_device 有个非常重要的成员变量：netdev_ops，为 net_device_ops 结构体指针类型，这就是网络设备的操作集。net_device_ops 结构体定义在 include/linux/netdevice.h 文件中，net_device_ops 结构体里面都是一些以“ndo_”开头的函数，这些函数就需要网络驱动编写人员去实现，不需要全部都实现，根据实际驱动情况实现其中一部分即可。结构体内容如下所示(结构体比较大，这里有缩减)：

示例代码 69.3.2.1 net_device_ops 结构体
```c
1 struct net_device_ops {
2 int (*ndo_init)(struct net_device *dev);
3 void (*ndo_uninit)(struct net_device *dev);
4 int (*ndo_open)(struct net_device *dev);
5 int (*ndo_stop)(struct net_device *dev);
6 netdev_tx_t (*ndo_start_xmit) (struct sk_buff *skb,
7 struct net_device *dev);
8 u16 (*ndo_select_queue)(struct net_device *dev,
9 struct sk_buff *skb,
10 void *accel_priv,
11 select_queue_fallback_t fallback);
12 void (*ndo_change_rx_flags)(struct net_device *dev,
13 int flags);
14 void (*ndo_set_rx_mode)(struct net_device *dev);
15 int (*ndo_set_mac_address)(struct net_device *dev,
16 void *addr);
17 int (*ndo_validate_addr)(struct net_device *dev);
18 int (*ndo_do_ioctl)(struct net_device *dev,
19 struct ifreq *ifr, int cmd);
20 int (*ndo_set_config)(struct net_device *dev,
21 struct ifmap *map);
22 int (*ndo_change_mtu)(struct net_device *dev,
23 int new_mtu);
24 int (*ndo_neigh_setup)(struct net_device *dev,
25 struct neigh_parms *);
26 void (*ndo_tx_timeout) (struct net_device *dev);
......
36 #ifdef CONFIG_NET_POLL_CONTROLLER
37 void (*ndo_poll_controller)(struct net_device *dev);
38 int (*ndo_netpoll_setup)(struct net_device *dev,
39 struct netpoll_info *info);
40 void (*ndo_netpoll_cleanup)(struct net_device *dev);
41 #endif
......
104 int (*ndo_set_features)(struct net_device *dev,
105 netdev_features_t features);
......
166 };
```

第 2 行：ndo_init 函数，当第一次注册网络设备的时候此函数会执行，设备可以在此函数中做一些需要推后初始化的内容，不过一般驱动中不使用此函数，虚拟网络设备可能会使用。
第 3 行：ndo_uninit 函数，卸载网络设备的时候此函数会执行。
第 4 行：ndo_open 函数，打开网络设备的时候此函数会执行，网络驱动程序需要实现此函数，非常重要！以 NXP 的 I.MX 系列 SOC 网络驱动为例，会在此函数中做如下工作：
- 使能网络外设时钟。
- 申请网络所使用的环形缓冲区。
- 初始化 MAC 外设。
- 绑定接口对应的 PHY。
- 如果使用 NAPI 的话要使能 NAPI 模块，通过 napi_enable 函数来使能。
- 开启 PHY。
- 调用 netif_tx_start_all_queues 来使能传输队列，也可能调用 netif_start_queue 函数。
- ……

第 5 行：ndo_stop 函数，关闭网络设备的时候此函数会执行，网络驱动程序也需要实现此函数。以 NXP 的 I.MX 系列 SOC 网络驱动为例，会在此函数中做如下工作：
- 停止 PHY。
- 停止 NAPI 功能。
- 停止发送功能。
- 关闭 MAC。
- 断开 PHY 连接。
- 关闭网络时钟。
- 释放数据缓冲区。
- ……

第 6 行：ndo_start_xmit 函数，当需要发送数据的时候此函数就会执行，此函数有一个参数为 sk_buff 结构体指针，sk_buff 结构体在 Linux 的网络驱动中非常重要，sk_buff 保存了上层传递给网络驱动层的数据。也就是说，要发送出去的数据都存在了 sk_buff 中，关于 sk_buff 稍后会做详细的讲解。如果发送成功的话此函数返回 NETDEV_TX_OK，如果发送失败了就返回 NETDEV_TX_BUSY，如果发送失败了我们就需要停止队列。
第 8 行：ndo_select_queue 函数，当设备支持多传输队列的时候选择使用哪个队列。
第 14 行：ndo_set_rx_mode 函数，此函数用于改变地址过滤列表，根据 net_device 的 flags 成员变量来设置 SOC 的网络外设寄存器。比如 flags 可能为 IFF_PROMISC、IFF_ALLMULTI或 IFF_MULTICAST，分别表示混杂模式、单播模式或多播模式。
第 15 行：ndo_set_mac_address 函数，此函数用于修改网卡的 MAC 地址，设置 net_device 的 dev_addr 成员变量，并且将 MAC 地址写入到网络外设的硬件寄存器中。
第 17 行：ndo_validate_addr 函数，验证 MAC 地址是否合法，也就是验证 net_device 的 dev_addr 中的 MAC 地址是否合法，直接调用 is_valid_ether_addr 函数。
第 18 行：ndo_do_ioctl 函数，用户程序调用 ioctl 的时候此函数就会执行，比如 PHY 芯片相关的命令操作，一般会直接调用 phy_mii_ioctl 函数。
第 22 行：ndo_change_mtu 函数，更改 MTU 大小。
第 26 行：ndo_tx_timeout 函数，当发送超时的时候函数会执行，一般都是网络出问题了导致发送超时。一般可能会重启 MAC 和 PHY，重新开始数据发送等。
第 37 行：ndo_poll_controller 函数，使用查询方式来处理网卡数据的收发。
第 104 行：ndo_set_features 函数，修改 net_device 的 features 属性，设置相应的硬件属性。

---

### 69.3.3 sk_buff 结构体

网络是分层的，对于应用层而言不用关心具体的底层是如何工作的，只需要按照协议将要发送或接收的数据打包好即可。打包好以后都通过 dev_queue_xmit 函数将数据发送出去，接收数据的话使用 netif_rx 函数即可，我们依次来看一下这两个函数。

#### 1、dev_queue_xmit 函数

此函数用于将网络数据发送出去，函数定义在 include/linux/netdevice.h 中，函数原型如下：
```c
static inline int dev_queue_xmit(struct sk_buff *skb)
```

函数参数和返回值含义如下：
skb：要发送的数据，这是一个 sk_buff 结构体指针，sk_buff 是 Linux 网络驱动中一个非常重要的结构体，网络数据就是以 sk_buff 保存的，各个协议层在 sk_buff 中添加自己的协议头，最终由底层驱动将 sk_buff 中的数据发送出去。网络数据的接收过程恰好相反，网络底层驱动将接收到的原始数据打包成 sk_buff，然后发送给上层协议，上层会取掉相应的头部，然后将最终的数据发送给用户。
返回值：0 发送成功，负值 发送失败。

dev_queue_xmit 函数太长，这里就不详细的分析了，dev_queue_xmit 函数最终是通过 net_device_ops 操作集里面的 ndo_start_xmit 函数来完成最终发送了，ndo_start_xmit 就是网络驱动编写人员去实现的，整个流程如图2-2 所示

![图2-2 dev_queue_xmit 执行流程](photo/Linux%20网络设备驱动/2-2%20dev_queue_xmit%20执行流程.png)

#### 2、netif_rx 函数

上层接收数据的话使用 netif_rx 函数，但是最原始的网络数据一般是通过轮询、中断或 NAPI 的方式来接收。netif_rx 函数定义在 net/core/dev.c 中，函数原型如下：
```c
int netif_rx(struct sk_buff *skb)
```

函数参数和返回值含义如下：
skb：保存接收数据的 sk_buff。
返回值：NET_RX_SUCCESS 成功，NET_RX_DROP 数据包丢弃。

我们重点来看一下 sk_buff 这个结构体，sk_buff 是 Linux 网络重要的数据结构，用于管理接收或发送数据包，sk_buff 结构体定义在 include/linux/skbuff.h 中，结构体内容如下(由于结构体比较大，为了缩小篇幅只列出部分重要的内容)：

示例代码 69.3.3.1 sk_buff 结构体
```c
1 struct sk_buff {
2 union {
3 struct {
4 /* These two members must be first. */
5 struct sk_buff *next;
6 struct sk_buff *prev;
7
8 union {
9 ktime_t	tstamp;
10 struct skb_mstamp skb_mstamp;
11 };
12 };
13 struct rb_node rbnode; /* used in netem & tcp stack */
14 };
15 struct sock	*sk;
16 struct net_device *dev;
17
18 /*
19 * This is the control buffer. It is free to use for every
20 * layer. Please put your private variables there. If you
21 * want to keep them across layers you have to do a skb_clone()
22 * first. This is owned by whoever has the skb queued ATM.
23 */
24 char	cb[48] __aligned(8);
25
26 unsigned long _skb_refdst;
27 void	(*destructor)(struct sk_buff *skb);
......
37 unsigned int len, data_len;
38 __u16	mac_len, hdr_len;
......
145 __be16	protocol;
146 __u16	transport_header;
147 __u16	network_header;
148 __u16	mac_header;
149
150 /* private: */
151 __u32	headers_end[0];
152 /* public: */
153
154 /* These elements must be at the end, see alloc_skb() for details. */
155 sk_buff_data_t	tail;
156 sk_buff_data_t end;
157 unsigned char *head, *data;
158 unsigned int truesize;
159 atomic_t	users;
160 };
```

第 5~6 行：next 和 prev 分别指向下一个和前一个 sk_buff，构成一个双向链表。
第 9 行：tstamp 表示数据包接收时或准备发送时的时间戳。
第 15 行：sk 表示当前 sk_buff 所属的 Socket。
第 16 行：dev 表示当前 sk_buff 从哪个设备接收到或者发出的。
第 24 行：cb 为控制缓冲区，不管哪个层都可以自由使用此缓冲区，用于放置私有数据。
第 27 行：destructor 函数，当释放缓冲区的时候可以在此函数里面完成某些动作。
第 37 行：len 为实际的数据长度，包括主缓冲区中数据长度和分片中的数据长度。data_len 为数据长度，只计算分片中数据的长度。
第 38 行：mac_len 为连接层头部长度，也就是 MAC 头的长度。
第 145 行：protocol 协议。
第 146 行：transport_header 为传输层头部。
第 147 行：network_header 为网络层头部
第 148 行：mac_header 为链接层头部。
第 155 行：tail 指向实际数据的尾部。
第 156 行：end 指向缓冲区的尾部。
第 157 行：head 指向缓冲区的头部，data 指向实际数据的头部。data 和 tail 指向实际数据的头部和尾部，head 和 end 指向缓冲区的头部和尾部。结构如图2-3

![图2-3 sk_buff 数据区结构示意图](photo/Linux%20网络设备驱动/2-3%20sk_buff%20数据区结构示意图.png)
<!-- 接上文 69.3.3 sk_buff 结构体（图2-3之后） -->

---

### 针对 sk_buff 的操作与管理函数

针对 sk_buff 内核提供了一系列的操作与管理函数，我们简单看一些常见的 API 函数：

#### 1、分配 sk_buff
要使用 sk_buff 必须先分配，首先来看一下 alloc_skb 这个函数，此函数定义在 include/linux/skbuff.h 中，函数原型如下：
```c
static inline struct sk_buff *alloc_skb(unsigned int size, gfp_t priority)
```
函数参数和返回值含义如下：
size：要分配的大小，也就是 skb 数据段大小。
priority：为 GFP MASK 宏，比如 GFP_KERNEL、GFP_ATOMIC 等。
返回值：分配成功的话就返回申请到的 sk_buff 首地址，失败的话就返回 NULL。

在网络设备驱动中常常使用 netdev_alloc_skb 来为某个设备申请一个用于接收的 skb_buff，此函数也定义在 include/linux/skbuff.h 中，函数原型如下：
```c
static inline struct sk_buff *netdev_alloc_skb(struct net_device *dev, unsigned int length)
```
函数参数和返回值含义如下：
dev：要给哪个设备分配 sk_buff。
length：要分配的大小。
返回值：分配成功的话就返回申请到的 sk_buff 首地址，失败的话就返回 NULL。

#### 2、释放 sk_buff
当使用完成以后就要释放掉 sk_buff，释放函数可以使用 kfree_skb，函数定义在 include/linux/skbuff.c 中，函数原型如下：
```c
void kfree_skb(struct sk_buff *skb)
```
函数参数和返回值含义如下：
skb：要释放的 sk_buff。
返回值：无。

对于网络设备而言最好使用如下所示释放函数：
```c
void dev_kfree_skb (struct sk_buff *skb)
```
函数只要一个参数 skb，就是要释放的 sk_buff。

#### 3、skb_put、skb_push、skb_pull 和 skb_reserve
这四个函数用于变更 sk_buff，先来看一下 skb_put 函数，此函数用于在尾部扩展 skb_buff 的数据区，也就将 skb_buff 的 tail 后移 n 个字节，从而导致 skb_buff 的 len 增加 n 个字节，原型如下：
```c
unsigned char *skb_put(struct sk_buff *skb, unsigned int len)
```
函数参数和返回值含义如下：
skb：要操作的 sk_buff。
len：要增加多少个字节。
返回值：扩展出来的那一段数据区首地址。

skb_put 操作之前和操作之后的数据区如图2-4
![图2-4 skb_put 函数操作前后对比](photo/Linux%20网络设备驱动/2-4%20skb_put%20函数操作前后对比.png)

skb_push 函数用于在头部扩展 skb_buff 的数据区，函数原型如下所示：
```c
unsigned char *skb_push(struct sk_buff *skb, unsigned int len)
```
函数参数和返回值含义如下：
skb：要操作的 sk_buff。
len：要增加多少个字节。
返回值：扩展完成以后新的数据区首地址。

skb_push 操作之前和操作之后的数据区如图 2-5所示
![图2-5 skb_push 函数操作前后对比](photo/Linux%20网络设备驱动/2-5%20skb_push%20函数操作前后对比.png)

skb_pull 函数用于从 sk_buff 的数据区起始位置删除数据，函数原型如下所示：
```c
unsigned char *skb_pull(struct sk_buff *skb, unsigned int len)
```
函数参数和返回值含义如下：
skb：要操作的 sk_buff。
len：要删除的字节数。
返回值：删除以后新的数据区首地址。

skb_pull 操作之前和操作之后的数据区如图2-6 
![图2-6 skb_pull 函数操作前后对比](photo/Linux%20网络设备驱动/2-6%20skb_pull%20函数操作前后对比.png)

skb_reserve 函数用于调整缓冲区的头部大小，方法很简单将 skb_buff 的 data 和 tail 同时后移 n 个字节即可，函数原型如下所示：
```c
static inline void skb_reserve(struct sk_buff *skb, int len)
```
函数参数和返回值含义如下：
skb：要操作的 sk_buff。
len：要增加的缓冲区头部大小。
返回值：无。

---

### 69.3.4 网络 NAPI 处理机制
如果玩过单片机的话应该都知道，像 IIC、SPI、网络等这些通信接口，接收数据有两种方法：轮询或中断。Linux 里面的网络数据接收也轮询和中断两种，中断的好处就是响应快，数据量小的时候处理及时，速度快，但是一旦当数据量大，而且都是短帧的时候会导致中断频繁发生，消耗大量的 CPU 处理时间在中断自身处理上。轮询恰好相反，响应没有中断及时，但是在处理大量数据的时候不需要消耗过多的 CPU 处理时间。Linux 在这两个处理方式的基础上提出了另外一种网络数据接收的处理方法：NAPI(New API)，NAPI 是一种高效的网络处理技术。NAPI 的核心思想就是不全部采用中断来读取网络数据，而是采用中断来唤醒数据接收服务程序，在接收服务程序中采用 POLL 的方法来轮询处理数据。这种方法的好处就是可以提高短数据包的接收效率，减少中断处理的时间。目前 NAPI 已经在 Linux 的网络驱动中得到了大量的应用，NXP 官方编写的网络驱动都是采用的 NAPI 机制。

关于 NAPI 详细的处理过程本章节不讨论，本章节就简单讲解一下如何在驱动中使用 NAPI，Linux 内核使用结构体 napi_struct 表示 NAPI，在使用 NAPI 之前要先初始化一个 napi_struct 实例。

#### 1、初始化 NAPI
首先要初始化一个 napi_struct 实例，使用 netif_napi_add 函数，此函数定义在 net/core/dev.c 中，函数原型如下：
```c
void netif_napi_add(struct net_device *dev,
struct napi_struct *napi,
int (*poll)(struct napi_struct *, int),
int weight)
```
函数参数和返回值含义如下：
dev：每个 NAPI 必须关联一个网络设备，此参数指定 NAPI 要关联的网络设备。
napi：要初始化的 NAPI 实例。
poll：NAPI 所使用的轮询函数，非常重要，一般在此轮询函数中完成网络数据接收的工作。
weight：NAPI 默认权重(weight)，一般为 NAPI_POLL_WEIGHT。
返回值：无。

#### 2、删除 NAPI
如果要删除 NAPI，使用 netif_napi_del 函数即可，函数原型如下：
```c
void netif_napi_del(struct napi_struct *napi)
```
函数参数和返回值含义如下：
napi：要删除的 NAPI。
返回值：无。

#### 3、使能 NAPI
初始化完 NAPI 以后，必须使能才能使用，使用函数 napi_enable，函数原型如下：
```c
inline void napi_enable(struct napi_struct *n)
```
函数参数和返回值含义如下：
n：要使能的 NAPI。
返回值：无。

#### 4、关闭 NAPI
关闭 NAPI 使用 napi_disable 函数即可，函数原型如下：
```c
void napi_disable(struct napi_struct *n)
```
函数参数和返回值含义如下：
n：要关闭的 NAPI。
返回值：无。

#### 5、检查 NAPI 是否可以进行调度
使用 napi_schedule_prep 函数检查 NAPI 是否可以进行调度，函数原型如下：
```c
inline bool napi_schedule_prep(struct napi_struct *n)
```
函数参数和返回值含义如下：
n：要检查的 NAPI。
返回值：如果可以调度就返回真，如果不可调度就返回假。

#### 6、NAPI 调度
如果可以调度的话就进行调度，使用__napi_schedule 函数完成 NAPI 调度，函数原型如下：
```c
void __napi_schedule(struct napi_struct *n)
```
函数参数和返回值含义如下：
n：要调度的 NAPI。
返回值：无。

我们也可以使用 napi_schedule 函数来一次完成 napi_schedule_prep 和__napi_schedule 这两个函数的工作，napi_schedule 函数内容如下所示：

示例代码 69.3.4.1 napi_schedule 函数
```c
1 static inline void napi_schedule(struct napi_struct *n)
2 {
3	if (napi_schedule_prep(n))
4	__napi_schedule(n);
5 }
```
从示例代码 69.3.4.1 可以看出，napi_schedule 函数就是对 napi_schedule_prep 和__napi_schedule 的简单封装，一次完成判断和调度。

#### 7、NAPI 处理完成
NAPI 处理完成以后需要调用 napi_complete 函数来标记 NAPI 处理完成，函数原型如下：
```c
inline void napi_complete(struct napi_struct *n)
```
函数参数和返回值含义如下：
n：处理完成的 NAPI。
返回值：无。

---

收到！懂了，原文没有图的地方，全部删掉“如图所示”；只有原文写了图编号（例如图1-1）才添加。精简提取核心，不全文搬运，不漏关键点。

---

## 69.4 I.MX6ULL 网络驱动简介

### 69.4.1 I.MX6ULL 网络外设设备树
核心：I.MX6ULL内置两路10/100M MAC（fec1/fec2），驱动逻辑一致；设备树规范参考内核文档Documentation/devicetree/bindings/net/fsl-fec.txt

#### 一、FEC以太网节点属性
✅ 必要属性
1.compatible：匹配字符串，"fsl,imx6ul-fec"、"fsl,imx6q-fec"，用于平台驱动匹配
2.reg：FEC寄存器基地址与地址长度
3.interrupts：以太网中断配置
4.phy-mode：PHY接口类型，RMII / MII，由硬件原理图决定
✅ 可选属性
phy-reset-gpios：PHY复位GPIO；配套phy-reset-duration设置复位延时，默认1ms，上限1000ms，超出强制1ms
phy-supply：PHY电源配置
phy-handle：指向mdio子节点内PHY设备引用
fsl,num-tx/rx-queues：收发队列数量，默认TX=1、RX=2
fsl,magic-packet：使能网卡魔术包硬件唤醒
fsl,wakeup_irq：唤醒中断索引
stop-mode：配置GPR寄存器，SOC停止模式相关

#### 二、mdio子节点（可选）
mdio是FEC的子节点，作为MDIO总线容器，用来挂载PHY设备；PHY属性参考Documentation/devicetree/bindings/net/phy.txt
PHY节点属性
reg：PHY在MDIO总线地址，必填
compatible：ethernet-phy-ieee802.3-c22（默认）/ c45；未知PHY ID可写ethernet-phy-idAAAA.BBBB
AAAA：PHY ID1寄存器；BBBB：PHY ID2寄存器
max-speed：PHY最高支持速率（10/100/1000）
interrupts、interrupt-parent：PHY中断，可选

#### 三、两层设备树节点说明
1.imx6ull.dtsi：NXP官方定义fec1/fec2基础节点，默认status = "disabled"，不能直接使用，需要板级dts覆盖修改
```dts
fec1: ethernet@02188000 {
    compatible = "fsl,imx6ul-fec", "fsl,imx6q-fec";
    reg = <0x02188000 0x4000>;
    interrupts = <GIC_SPI 118 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 119 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_ENET>,
             <&clks IMX6UL_CLK_ENET_AHB>,
             <&clks IMX6UL_CLK_ENET_PTP>,
             <&clks IMX6UL_CLK_ENET_REF>,
             <&clks IMX6UL_CLK_ENET_REF>;
    clock-names = "ipg", "ahb", "ptp", "enet_clk_ref", "enet_out";
    stop-mode = <&gpr 0x10 3>;
    fsl,num-tx-queues=<1>;
    fsl,num-rx-queues=<1>;
    fsl,magic-packet;
    fsl,wakeup_irq = <0>;
    status = "disabled";
};
fec2: ethernet@020b4000 {
    // 结构同fec1，地址、中断、stop-mode参数不同
    status = "disabled";
};
```

fec1 和 fec2 分别对应 I.MX6ULL 的 ENET1 和 ENET2，至于节点的具体属性就不分析了， 
上面在讲解绑定文档的时候就已经详细的讲过了。示例代码 69.4.1.1 是 NXP 官方编写的，我们 
不需要去修改，但是示例代码 69.4.1.1 是不能工作的，还需要根据实际情况添加或修改一些属 
性。打开imx6ull-alientek-emmc.dts：板级设备树，追加修改，开启网卡、配置引脚与PHY：
```dts
&fec1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_enet1 &pinctrl_enet1_reset>;
    phy-mode = "rmii";
    phy-handle = <&ethphy0>;
    phy-reset-gpios = <&gpio5 7 GPIO_ACTIVE_LOW>;
    phy-reset-duration = <200>;
    status = "okay";
};
&fec2 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_enet2 &pinctrl_enet2_reset>;
    phy-mode = "rmii";
    phy-handle = <&ethphy1>;
    phy-reset-gpios = <&gpio5 8 GPIO_ACTIVE_LOW>;
    phy-reset-duration = <200>;
    status = "okay";

    mdio {
        #address-cells = <1>;
        #size-cells = <0>;
        ethphy0: ethernet-phy@0 {
            compatible = "ethernet-phy-ieee802.3-c22";
            reg = <0>;
        };
        ethphy1: ethernet-phy@1 {
            compatible = "ethernet-phy-ieee802.3-c22";
            reg = <1>;
        };
    };
};
```

要点：
pinctrl-0：绑定网口信号引脚+PHY复位引脚
status="okay"：启用设备
mdio子节点放在&fec2下，一条MDIO总线同时管理两个PHY（MDIO总线最多管理32个PHY）

#### 四、Pinctrl引脚配置
pinctrl_enet1：ENET1数据/时钟信号引脚复用配置
pinctrl_enet2：ENET2数据、MDC/MDIO引脚复用
pinctrl_enet1_reset / pinctrl_enet2_reset：PHY复位GPIO配置
注意：双网口同时启用时，MDC/MDIO挂载在ENET2，若挂载ENET1会导致网络异常；复位引脚属于SNVS域，单独分组定义

示例代码 69.4.1.2 网络引脚 pinctrl 信息 
```dts
1 pinctrl_enet1: enet1grp { 
2 
fsl,pins = < 
3 
MX6UL_PAD_ENET1_RX_EN__ENET1_RX_EN 
0x1b0b0 
4 
MX6UL_PAD_ENET1_RX_ER__ENET1_RX_ER 
0x1b0b0 
5 
MX6UL_PAD_ENET1_RX_DATA0__ENET1_RDATA00 0x1b0b0 
6 
MX6UL_PAD_ENET1_RX_DATA1__ENET1_RDATA01 0x1b0b0 
7 
MX6UL_PAD_ENET1_TX_EN__ENET1_TX_EN 
0x1b0b0 
8 
MX6UL_PAD_ENET1_TX_DATA0__ENET1_TDATA00 
0x1b0b0 
9 
MX6UL_PAD_ENET1_TX_DATA1__ENET1_TDATA01 0x1b0b0 
10 
MX6UL_PAD_ENET1_TX_CLK__ENET1_REF_CLK1 0x4001b009 
11 
>; 
12 }; 
13 
14 pinctrl_enet2: enet2grp { 
15 
fsl,pins = <
16 
MX6UL_PAD_GPIO1_IO07__ENET2_MDC 
0x1b0b0 
17 
MX6UL_PAD_GPIO1_IO06__ENET2_MDIO 
0x1b0b0 
18 
MX6UL_PAD_ENET2_RX_EN__ENET2_RX_EN 
0x1b0b0 
19 
MX6UL_PAD_ENET2_RX_ER__ENET2_RX_ER 
0x1b0b0 
20 
MX6UL_PAD_ENET2_RX_DATA0__ENET2_RDATA00 0x1b0b0 
21 
MX6UL_PAD_ENET2_RX_DATA1__ENET2_RDATA01 0x1b0b0 
22 
MX6UL_PAD_ENET2_TX_EN__ENET2_TX_EN 
0x1b0b0 
23 
MX6UL_PAD_ENET2_TX_DATA0__ENET2_TDATA00 0x1b0b0 
24 
MX6UL_PAD_ENET2_TX_DATA1__ENET2_TDATA01 0x1b0b0 
25 
MX6UL_PAD_ENET2_TX_CLK__ENET2_REF_CLK2 0x4001b009 
26 
>; 
27 }; 
28 
29 /*enet1 reset zuozhongkai*/ 
30 pinctrl_enet1_reset: enet1resetgrp { 
31 fsl,pins = < 
32 
/* used for enet1 reset */ 
33 
MX6ULL_PAD_SNVS_TAMPER7__GPIO5_IO07 0x10B0 
34 >; 
35 }; 
36 
37 /*enet2 reset zuozhongkai*/ 
38 pinctrl_enet2_reset: enet2resetgrp { 
39 fsl,pins = < 
40 
/* used for enet2 reset */ 
41 MX6ULL_PAD_SNVS_TAMPER8__GPIO5_IO08 0x10B0 
42 >; 
43 };
```

---

### 69.4.2 I.MX6ULL 网络驱动源码简析
IMX6ULL网络驱动分为两部分：SOC内置MAC控制器驱动(fec_main.c) + PHY通用驱动；一般不需要开发者从零编写。 驱动文件路径：drivers/net/ethernet/freescale/fec_main.c

#### 1、fec_probe 函数简析
platform_driver匹配表
```c
static const struct of_device_id fec_dt_ids[] = {
    { .compatible = "fsl,imx25-fec", .data = &fec_devtype[IMX25_FEC], },
    { .compatible = "fsl,imx27-fec", .data = &fec_devtype[IMX27_FEC], },
    { .compatible = "fsl,imx28-fec", .data = &fec_devtype[IMX28_FEC], },
    { .compatible = "fsl,imx6q-fec", .data = &fec_devtype[IMX6Q_FEC], },
    { .compatible = "fsl,mvf600-fec", .data = &fec_devtype[MVF600_FEC], },
    { .compatible = "fsl,imx6sx-fec", .data = &fec_devtype[IMX6SX_FEC], },
    { .compatible = "fsl,imx6ul-fec", .data = &fec_devtype[IMX6UL_FEC], }, /* IMX6ULL匹配项 */
    { /* sentinel */ }
};

static struct platform_driver fec_driver = {
    .driver = {
        .name = DRIVER_NAME,
        .pm = &fec_pm_ops,
        .of_match_table = fec_dt_ids,   /*设备树匹配表*/
    },
    .id_table = fec_devtype,
    .probe = fec_probe,                 /*匹配成功执行probe*/
    .remove = fec_drv_remove,
};
```
设备树compatible = "fsl,imx6ul-fec"匹配成功，执行fec_probe。
fec_probe关键流程（核心片段注释）
```c
static int fec_probe(struct platform_device *pdev)
{
    struct fec_enet_private *fep;
    struct fec_platform_data *pdata;
    struct net_device *ndev;
    int i, irq, ret = 0;
    struct resource *r;
    const struct of_device_id *of_id;
    static int dev_id;
    struct device_node *np = pdev->dev.of_node, *phy_node;
    int num_tx_qs;
    int num_rx_qs;

    fec_enet_get_queue_num(pdev, &num_tx_qs, &num_rx_qs); /*读取设备树收发队列数量*/

    /* 1.分配网络设备 net_device，同时分配私有数据fec_enet_private */
    ndev = alloc_etherdev_mqs(sizeof(struct fec_enet_private),num_tx_qs, num_rx_qs);
    if (!ndev)
        return -ENOMEM;

    SET_NETDEV_DEV(ndev, &pdev->dev);
    fep = netdev_priv(ndev); /*获取net_device私有数据指针 fec_enet_private*/

    of_id = of_match_device(fec_dt_ids, &pdev->dev);
    if (of_id)
        pdev->id_entry = of_id->data;
    fep->quirks = pdev->id_entry->driver_data; /*获取SOC差异标志位*/

    fep->netdev = ndev;
    fep->num_rx_queues = num_rx_qs;
    fep->num_tx_queues = num_tx_qs;

    pinctrl_pm_select_default_state(&pdev->dev); /*设置引脚默认状态*/

    /*2.获取寄存器物理地址，映射为内核虚拟地址*/
    r = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    fep->hwp = devm_ioremap_resource(&pdev->dev, r);
    if (IS_ERR(fep->hwp)) {
        ret = PTR_ERR(fep->hwp);
        goto failed_ioremap;
    }

    fep->pdev = pdev;
    fep->dev_id = dev_id++;
    platform_set_drvdata(pdev, ndev); /*把net_device挂到platform_device私有数据*/

    fec_enet_of_parse_stop_mode(pdev); /*解析stop-mode属性*/

    if (of_get_property(np, "fsl,magic-packet", NULL)) /*判断是否支持魔术包唤醒*/
        fep->wol_flag |= FEC_WOL_HAS_MAGIC_PACKET;

    phy_node = of_parse_phandle(np, "phy-handle", 0); /*获取PHY节点句柄*/
    if (!phy_node && of_phy_is_fixed_link(np)) {
        ret = of_phy_register_fixed_link(np);
        if (ret < 0) {
            dev_err(&pdev->dev,"broken fixed-link specification\n");
            goto failed_phy;
        }
        phy_node = of_node_get(np);
    }
    fep->phy_node = phy_node;

    ret = of_get_phy_mode(pdev->dev.of_node); /*读取phy-mode RMII/MII*/
    if (ret < 0) {
        pdata = dev_get_platdata(&pdev->dev);
        if (pdata)
            fep->phy_interface = pdata->phy;
        else
            fep->phy_interface = PHY_INTERFACE_MODE_MII;
    } else {
        fep->phy_interface = ret;
    }

    /*获取各类时钟 ipg ahb ptp enet_clk_ref enet_out*/
    fep->clk_ipg = devm_clk_get(&pdev->dev, "ipg");
    if (IS_ERR(fep->clk_ipg)) {
        ret = PTR_ERR(fep->clk_ipg);
        goto failed_clk;
    }
    fep->clk_ahb = devm_clk_get(&pdev->dev, "ahb");
    if (IS_ERR(fep->clk_ahb)) {
        ret = PTR_ERR(fep->clk_ahb);
        goto failed_clk;
    }
    fep->itr_clk_rate = clk_get_rate(fep->clk_ahb);
    fep->clk_enet_out = devm_clk_get(&pdev->dev, "enet_out");
    if (IS_ERR(fep->clk_enet_out))
        fep->clk_enet_out = NULL;

    fep->ptp_clk_on = false;
    mutex_init(&fep->ptp_clk_mutex);

    fep->clk_ref = devm_clk_get(&pdev->dev, "enet_clk_ref");
    if (IS_ERR(fep->clk_ref))
        fep->clk_ref = NULL;

    fep->bufdesc_ex = fep->quirks & FEC_QUIRK_HAS_BUFDESC_EX;
    fep->clk_ptp = devm_clk_get(&pdev->dev, "ptp");
    if (IS_ERR(fep->clk_ptp)) {
        fep->clk_ptp = NULL;
        fep->bufdesc_ex = false;
    }

    pm_runtime_enable(&pdev->dev);
    ret = fec_enet_clk_enable(ndev, true); /*使能网卡时钟*/
    if (ret)
        goto failed_clk;

    fep->reg_phy = devm_regulator_get(&pdev->dev, "phy"); /*获取PHY电源*/
    if (!IS_ERR(fep->reg_phy)) {
        ret = regulator_enable(fep->reg_phy);
        if (ret) {
            dev_err(&pdev->dev,"Failed to enable phy regulator: %d\n", ret);
            goto failed_regulator;
        }
    } else {
        fep->reg_phy = NULL;
    }

    fec_reset_phy(pdev); /*执行PHY硬件复位*/

    if (fep->bufdesc_ex)
        fec_ptp_init(pdev); /*初始化PTP时间戳*/

    ret = fec_enet_init(ndev);
    /*
     * fec_enet_init:
     * 1.分配DMA环形缓冲区
     * 2.设置netdev_ops、ethtool_ops
     * 3.注册NAPI，poll函数：fec_enet_rx_napi
     * 4.初始化MAC硬件寄存器
     */
    if (ret)
        goto failed_init;

    /*获取、注册网络中断*/
    for (i = 0; i < FEC_IRQ_NUM; i++) {
        irq = platform_get_irq(pdev, i);
        if (irq < 0) {
            if (i)
                break;
            ret = irq;
            goto failed_irq;
        }
        ret = devm_request_irq(&pdev->dev, irq, fec_enet_interrupt,0, pdev->name, ndev);
        if (ret)
            goto failed_irq;
        fep->irq[i] = irq;
    }

    ret = of_property_read_u32(np, "fsl,wakeup_irq", &irq); /*读取唤醒中断*/
    if (!ret && irq < FEC_IRQ_NUM)
        fep->wake_irq = fep->irq[irq];
    else
        fep->wake_irq = fep->irq[0];

    init_completion(&fep->mdio_done);
    ret = fec_enet_mii_init(pdev);
    /*
     * fec_enet_mii_init：
     * 1.初始化mii_bus结构体
     * 2.赋值mdio读写回调：fec_enet_mdio_read / fec_enet_mdio_write
     * 3.调用of_mdiobus_register注册MDIO总线
     */
    if (ret)
        goto failed_mii_init;

    netif_carrier_off(ndev); /*初始链路状态down，交由phylib处理*/
    fec_enet_clk_enable(ndev, false);
    pinctrl_pm_select_sleep_state(&pdev->dev);

    ret = register_netdev(ndev); /*向内核注册net_device设备！*/
    if (ret)
        goto failed_register;

    device_init_wakeup(&ndev->dev, fep->wol_flag & FEC_WOL_HAS_MAGIC_PACKET);
    if (fep->bufdesc_ex && fep->ptp_clock)
        netdev_info(ndev, "registered PHC device %d\n", fep->dev_id);

    fep->rx_copybreak = COPYBREAK_DEFAULT;
    INIT_WORK(&fep->tx_timeout_work, fec_enet_timeout_work);
    return 0;
......
    return ret;
}
```

#### 2、MDIO总线注册
mii_bus结构体，代表MDIO总线，定义在include/linux/phy.h
```c
struct mii_bus {
    const char *name;
    char id[MII_BUS_ID_SIZE];
    void *priv;
    /* MDIO读写PHY寄存器回调，SOC驱动必须实现 */
    int (*read)(struct mii_bus *bus, int phy_id, int regnum);
    int (*write)(struct mii_bus *bus, int phy_id, int regnum, u16 val);
    int (*reset)(struct mii_bus *bus);

    struct mutex mdio_lock; /*MDIO总线互斥锁，防止并发读写*/

    struct device *parent;
    enum {
        MDIOBUS_ALLOCATED = 1,
        MDIOBUS_REGISTERED,
        MDIOBUS_UNREGISTERED,
        MDIOBUS_RELEASED,
    } state;
    struct device dev;

    struct phy_device *phy_map[PHY_MAX_ADDR]; /*总线上所有PHY设备数组*/
    u32 phy_mask; /*需要忽略探测的PHY地址掩码*/

    int *irq; /*每个PHY对应的中断号数组*/
};
```

of_mdiobus_register函数
```c
int of_mdiobus_register(struct mii_bus *mdio, struct device_node *np)
{
    struct device_node *child;
    const __be32 *paddr;
    bool scanphys = false;
    int addr, rc, i;

    mdio->phy_mask = ~0; /*关闭自动扫描PHY，完全使用设备树子节点*/

    if (mdio->irq)
        for (i=0; i<PHY_MAX_ADDR; i++)
            mdio->irq[i] = PHY_POLL; /*默认使用轮询模式*/

    mdio->dev.of_node = np;

    rc = mdiobus_register(mdio); /*真正注册MDIO总线到内核*/
    if (rc)
        return rc;

    /*遍历mdio节点下所有PHY子节点*/
    for_each_available_child_of_node(np, child) {
        addr = of_mdio_parse_addr(&mdio->dev, child); /*解析PHY地址(@后面数字)*/
        if (addr < 0) {
            scanphys = true;
            continue;
        }
        rc = of_mdiobus_register_phy(mdio, child, addr); /*注册单个PHY设备*/
        if (rc)
            continue;
    }

    if (!scanphys)
        return 0;
......
    return 0;
}
```

of_mdiobus_register_phy函数，创建并注册phy_device
```c
static int of_mdiobus_register_phy(struct mii_bus *mdio,struct device_node *child,u32 addr)
{
    struct phy_device *phy;
    bool is_c45;
    int rc;
    u32 phy_id;

    /*判断PHY是C22还是C45标准*/
    is_c45 = of_device_is_compatible(child,"ethernet-phy-ieee802.3-c45");

    if (!is_c45 && !of_get_phy_id(child, &phy_id))
        phy = phy_device_create(mdio, addr, phy_id, 0, NULL); /*创建设备*/
    else
        phy = get_phy_device(mdio, addr, is_c45);

    if (!phy || IS_ERR(phy))
        return 1;

    rc = irq_of_parse_and_map(child, 0); /*解析PHY中断*/
    if (rc > 0) {
        phy->irq = rc;
        if (mdio->irq)
            mdio->irq[addr] = rc;
    } else {
        if (mdio->irq)
            phy->irq = mdio->irq[addr];
    }

    of_node_get(child);
    phy->dev.of_node = child; /*绑定设备树节点*/

    rc = phy_device_register(phy); /*向内核注册phy_device*/
    if (rc) {
        phy_device_free(phy);
        of_node_put(child);
        return 1;
    }

    dev_dbg(&mdio->dev, "registered phy %s at address %i\n",child->name, addr);
    return 0;
}
```

整体流程：注册MDIO总线 → 遍历mdio子节点 → 创建phy_device → 注册PHY设备。
![图3-1 MDIO 总线注册流程](photo/Linux%20网络设备驱动/3-1%20MDIO%20总线注册流程.png)
<!-- 接上文 69.4.2 MDIO总线注册流程 -->

#### 3、fec_drv_remove 卸载函数
```c
static int fec_drv_remove(struct platform_device *pdev)
{
    struct net_device *ndev = platform_get_drvdata(pdev);
    struct fec_enet_private *fep = netdev_priv(ndev);

    cancel_delayed_work_sync(&fep->time_keep);
    cancel_work_sync(&fep->tx_timeout_work);
    unregister_netdev(ndev);        /*注销net_device*/
    fec_enet_mii_remove(fep);       /*注销MDIO总线，mdiobus_unregister/mdiobus_free*/
    if (fep->reg_phy)
        regulator_disable(fep->reg_phy); /*关闭PHY电源*/
    if (fep->ptp_clock)
        ptp_clock_unregister(fep->ptp_clock); /*注销PTP时钟*/
    of_node_put(fep->phy_node);
    free_netdev(ndev);              /*释放net_device内存*/
    return 0;
}
```

---

### 69.4.3 fec_netdev_ops 操作集
```c
static const struct net_device_ops fec_netdev_ops = {
    .ndo_open         = fec_enet_open,        /*ifconfig up打开网卡执行*/
    .ndo_stop         = fec_enet_close,       /*ifconfig down关闭网卡执行*/
    .ndo_start_xmit   = fec_enet_start_xmit, /*数据包发送入口*/
    .ndo_select_queue = fec_enet_select_queue,
    .ndo_set_rx_mode  = set_multicast_list,
    .ndo_change_mtu   = eth_change_mtu,
    .ndo_validate_addr = eth_validate_addr,
    .ndo_tx_timeout   = fec_timeout,          /*发送超时回调*/
    .ndo_set_mac_address = fec_set_mac_address, /*设置MAC地址*/
    .ndo_do_ioctl     = fec_enet_ioctl,
#ifdef CONFIG_NET_POLL_CONTROLLER
    .ndo_poll_controller = fec_poll_controller,
#endif
    .ndo_set_features = fec_set_features,
};
```

#### 1、fec_enet_open 网卡打开
```c
static int fec_enet_open(struct net_device *ndev)
{
    struct fec_enet_private *fep = netdev_priv(ndev);
    const struct platform_device_id *id_entry = platform_get_device_id(fep->pdev);
    int ret;

    pinctrl_pm_select_default_state(&fep->pdev->dev);
    ret = fec_enet_clk_enable(ndev, true); /*打开网卡时钟*/
    if (ret)
        return ret;

    ret = fec_enet_alloc_buffers(ndev); /*申请收发DMA环形buffer*/
    if (ret)
        goto err_enet_alloc;

    fec_restart(ndev); /*重启MAC硬件*/

    ret = fec_enet_mii_probe(ndev); /*探测、绑定PHY设备*/
    if (ret)
        goto err_enet_mii_probe;

    napi_enable(&fep->napi); /*使能NAPI*/
    phy_start(fep->phy_dev); /*启动PHY*/
    netif_tx_start_all_queues(ndev); /*开启发送队列*/
......
    return 0;

err_enet_mii_probe:
    fec_enet_free_buffers(ndev);
err_enet_alloc:
    fep->miibus_up_failed = true;
    if (!fep->mii_bus_share)
        pinctrl_pm_select_sleep_state(&fep->pdev->dev);
    return ret;
}
```

#### 2、fec_enet_close 网卡关闭
```c
static int fec_enet_close(struct net_device *ndev)
{
    struct fec_enet_private *fep = netdev_priv(ndev);

    phy_stop(fep->phy_dev); /*停止PHY*/

    if (netif_device_present(ndev)) {
        napi_disable(&fep->napi); /*关闭NAPI*/
        netif_tx_disable(ndev);   /*停止发送队列*/
        fec_stop(ndev);          /*关闭MAC硬件*/
    }

    phy_disconnect(fep->phy_dev); /*断开PHY连接*/
    fep->phy_dev = NULL;

    fec_enet_clk_enable(ndev, false); /*关闭网卡时钟*/
    pm_qos_remove_request(&fep->pm_qos_req);
    pinctrl_pm_select_sleep_state(&fep->pdev->dev);
    pm_runtime_put_sync_suspend(ndev->dev.parent);
    fec_enet_free_buffers(ndev); /*释放DMA缓冲区*/
    return 0;
}
```

#### 3、fec_enet_start_xmit 数据包发送入口
```c
static netdev_tx_t fec_enet_start_xmit(struct sk_buff *skb,struct net_device *ndev)
{
    struct fec_enet_private *fep = netdev_priv(ndev);
    int entries_free;
    unsigned short queue;
    struct fec_enet_priv_tx_q *txq;
    struct netdev_queue *nq;
    int ret;

    queue = skb_get_queue_mapping(skb);
    txq = fep->tx_queue[queue];
    nq = netdev_get_tx_queue(ndev, queue);

    /*判断是否为GSO大数据包，TSO网卡分段卸载*/
    if (skb_is_gso(skb))
        ret = fec_enet_txq_submit_tso(txq, skb, ndev);
    else
        ret = fec_enet_txq_submit_skb(txq, skb, ndev);
    if (ret)
        return ret;

    entries_free = fec_enet_get_free_txdesc_num(fep, txq); /*获取空闲发送描述符*/
    if (entries_free <= txq->tx_stop_threshold)
        netif_tx_stop_queue(nq); /*描述符不足，暂停发送队列*/

    return NETDEV_TX_OK;
}
```

#### 4、fec_enet_interrupt 中断服务函数（NAPI上半部）
中断只做标记、调度NAPI，不做实际报文接收处理，报文处理交给NAPI poll下半部
```c
static irqreturn_t fec_enet_interrupt(int irq, void *dev_id)
{
    struct net_device *ndev = dev_id;
    struct fec_enet_private *fep = netdev_priv(ndev);
    uint int_events;
    irqreturn_t ret = IRQ_NONE;

    int_events = readl(fep->hwp + FEC_IEVENT); /*读取中断事件寄存器*/
    writel(int_events, fep->hwp + FEC_IEVENT); /*写回清除中断标志*/
    fec_enet_collect_events(fep, int_events);   /*统计收发中断事件*/

    if ((fep->work_tx || fep->work_rx) && fep->link) {
        ret = IRQ_HANDLED;
        if (napi_schedule_prep(&fep->napi)) {
            writel(FEC_ENET_MII, fep->hwp + FEC_IMASK); /*临时关闭中断*/
            __napi_schedule(&fep->napi); /*调度NAPI poll函数*/
        }
    }

    if (int_events & FEC_ENET_MII) { /*MDIO完成中断，唤醒等待completion*/
        ret = IRQ_HANDLED;
        complete(&fep->mdio_done);
    }

    if (fep->ptp_clock)
        fec_ptp_check_pps_event(fep);

    return ret;
}
```

#### 5、fec_enet_rx_napi NAPI poll下半部函数
```c
static int fec_enet_rx_napi(struct napi_struct *napi, int budget)
{
    struct net_device *ndev = napi->dev;
    struct fec_enet_private *fep = netdev_priv(ndev);
    int pkts;

    pkts = fec_enet_rx(ndev, budget); /*真正接收数据包，budget是允许处理包上限*/
    fec_enet_tx(ndev);               /*处理发送完成后的清理*/

    /*收到数据包数量小于budget代表没有更多报文，结束NAPI，重新打开硬件中断*/
    if (pkts < budget) {
        napi_complete(napi);
        writel(FEC_DEFAULT_IMASK, fep->hwp + FEC_IMASK);
    }
    return pkts; /*返回本次处理报文数量*/
}
```

关键逻辑总结：
1.中断上半部：读中断标志、清中断、调用__napi_schedule；关闭硬件中断，避免重复触发。
2.NAPI poll下半部：fec_enet_rx收包，fec_enet_tx处理发送完成。
3.当包数小于budget：调用napi_complete退出NAPI，重新使能硬件中断。

---

### 69.4.3 Linux内核PHY子系统与MDIO总线简析
核心思想：PHY子系统完全遵循Linux设备-总线-驱动模型；phy_device代表PHY设备，phy_driver代表PHY驱动，MDIO总线完成设备与驱动匹配。网卡MAC控制器通过MDIO总线读写PHY寄存器，完成速率、双工、链路状态管理。

#### 1、PHY设备 phy_device
头文件：include/linux/phy.h
phy_device 结构体实例对应真实的一颗PHY芯片，关键成员释义：
```c
struct phy_device {
    struct phy_driver *drv;         // 绑定成功后的PHY驱动
    struct mii_bus *bus;            // 所属MII/MDIO总线
    struct device dev;              // 内核标准device设备
    u32 phy_id;                     // 从PHY寄存器读出来的芯片ID
    enum phy_state state;           // PHY运行状态：就绪、挂起、链路断开等
    phy_interface_t interface;      // 接口类型：MII/RMII/RGMII
    int addr;                       // PHY芯片MDIO地址 0~31
    int speed;                      // 协商得到速率：10/100/1000
    int duplex;                     // 双工模式：全双工/半双工
    int link;                       // 链路状态，1链路通，0链路断开
    int irq;                        // PHY中断号，-1代表无中断，使用轮询
    void *priv;                     // PHY驱动私有数据
    struct net_device *attached_dev;// 绑定的MAC网络设备(eth0)
    void (*adjust_link)(struct net_device *dev); // MAC侧回调，PHY链路变化通知MAC
};
```

PHY设备创建&注册流程
1.get_phy_device(bus, addr, is_c45)
内部调用get_phy_id()：通过MDIO总线读取PHY芯片ID寄存器，拿到phy_id；
判断读到ID是否全为0xFFFFFFFF，代表该MDIO地址上没有PHY硬件，返回NULL；
调用phy_device_create()：申请内存，初始化phy_device各个成员，返回phy_device实例。

2.phy_device_register(struct phy_device *phy) 将创建好的phy_device向MDIO总线注册，生成sysfs设备节点，等待和phy_driver匹配。

简单流程：读PHY硬件ID → 创建phy_device对象 → 注册到MDIO总线。

#### 2、PHY驱动 phy_driver
同样定义在include/linux/phy.h，是对PHY芯片操作方法集合。
```c
struct phy_driver {
    u32 phy_id;                 // 该驱动支持的PHY芯片ID
    unsigned int phy_id_mask;   // ID掩码，规定哪些bit参与匹配
    char *name;                 // PHY驱动名称，dmesg可打印看到
    u32 features;               // PHY硬件能力：千兆、pause帧等

    // PHY操作回调函数，芯片厂商驱动按需实现，可复用通用phy函数
    int (*soft_reset)(struct phy_device *phydev);    // PHY软件复位
    int (*config_init)(struct phy_device *phydev);   // PHY初始化配置
    int (*probe)(struct phy_device *phydev);         // 匹配成功后probe
    int (*suspend)(struct phy_device *phydev);       // 休眠
    int (*resume)(struct phy_device *phydev);        // 唤醒
    int (*config_aneg)(struct phy_device *phydev);   // 启动自动协商
    int (*aneg_done)(struct phy_device *phydev);     // 判断自协商是否完成
    int (*read_status)(struct phy_device *phydev);    // 读取PHY链路、速率、双工状态
    int (*ack_interrupt)(struct phy_device *phydev);  // 清除PHY中断标志
    int (*config_intr)(struct phy_device *phydev);    // 配置PHY中断使能

    struct device_driver driver; // 内嵌标准device_driver
};
```

PHY驱动注册API
1.单个驱动注册
```c
int phy_driver_register(struct phy_driver *new_driver);
```
注册后该驱动挂载到mdio_bus_type MDIO总线上。

2.批量注册多个PHY驱动 同一厂商多款PHY芯片，放在数组中一次性注册：
```c
int phy_drivers_register(struct phy_driver *new_driver, int n);
```
new_driver：phy_driver数组首地址
n：数组元素个数

3.卸载驱动
```c
void phy_driver_unregister(struct phy_driver *drv);
```
便捷宏：module_phy_driver(drv_array)，封装了模块init/exit，自动完成注册卸载，smsc.c中就使用该宏。

#### 3、MDIO总线 mdio_bus_type
文件路径：drivers/net/phy/mdio_bus.c
MDIO是PHY子系统的内核总线，负责匹配phy_device(设备)与phy_driver(驱动)。
```c
struct bus_type mdio_bus_type = {
    .name = "mdio_bus",
    .match = mdio_bus_match,  // 核心匹配函数
    .pm = MDIO_BUS_PM_OPS,
    .dev_groups = mdio_dev_groups,
};
```

匹配函数 mdio_bus_match 匹配优先级（从上到下）
```c
static int mdio_bus_match(struct device *dev, struct device_driver *drv)
{
    struct phy_device *phydev = to_phy_device(dev);
    struct phy_driver *phydrv = to_phy_driver(drv);

    // ① 设备树compatible匹配（of匹配）
    if (of_driver_match_device(dev, drv))
        return 1;

    // ② PHY驱动自定义match回调函数
    if (phydrv->match_phy_device)
        return phydrv->match_phy_device(phydev);

    // ③ PHY_ID + ID掩码匹配（IMX6ULL LAN8720使用该方式）
    return (phydrv->phy_id & phydrv->phy_id_mask) ==
           (phydev->phy_id & phydrv->phy_id_mask);
}
```
1.of_driver_match_device：优先设备树compatible匹配；大部分传统PHY不使用。
2.match_phy_device：驱动自定义匹配函数；
3.PHY_ID掩码对比（IMX6ULL-ALPHA开发板LAN8720使用此方式）
phy_device->phy_id：运行时从PHY硬件寄存器读出来的真实ID
phy_driver->phy_id / phy_id_mask：驱动代码写死的支持ID与掩码
两边同时和mask做与运算，结果相等，代表设备和驱动匹配成功。

匹配失败：会降级使用内核内置Generic PHY通用驱动来接管该PHY芯片。

#### 4、通用PHY驱动 Generic PHY
文件：drivers/net/phy/phy_device.c
phy_init()是PHY子系统入口函数：
1.初始化MDIO总线mdio_bus_init()
2.调用phy_drivers_register()注册genphy_driver数组，包含两套通用驱动
Generic PHY：用于10/100/1000M PHY
Generic 10G PHY：用于万兆PHY
```c
static struct phy_driver genphy_driver[] = {
{
    .phy_id = 0xffffffff,
    .phy_id_mask = 0xffffffff,
    .name = "Generic PHY",
    .soft_reset = genphy_soft_reset,
    .config_init = genphy_config_init,
    .config_aneg = genphy_config_aneg,
    .aneg_done = genphy_aneg_done,
    .read_status = genphy_read_status,
    .suspend = genphy_suspend,
    .resume = genphy_resume,
},
// Generic 10G PHY省略...
};
```
很多厂商PHY驱动不会全部重写回调，大量复用genphy_xxx通用函数，只重写芯片特殊部分，例如LAN8720A。

#### 5、LAN8720A PHY驱动简析（smsc.c）
路径：drivers/net/phy/smsc.c
内核配置开启：
```
Device Drivers
    → Network device support
        → PHY Device support and infrastructure
            → Drivers for SMSC PHYs
```
smsc_phy_driver数组，支持多款SMSC PHY，LAN8710/LAN8720条目：
```c
{
    .phy_id = 0x0007c0f0,
    .phy_id_mask = 0xfffffff0,  // 只比较高28bit，低4bit忽略
    .name = "SMSC LAN8710/LAN8720",
    .features = (PHY_BASIC_FEATURES | SUPPORTED_Pause | SUPPORTED_Asym_Pause),
    .flags = PHY_HAS_INTERRUPT | PHY_HAS_MAGICANEG,

    .config_aneg = genphy_config_aneg,   // 直接复用通用PHY自协商函数
    .read_status = lan87xx_read_status,  // SMSC自己实现状态读取
    .config_init = smsc_phy_config_init, // 芯片私有初始化
    .soft_reset = smsc_phy_reset,        // 芯片私有复位

    .ack_interrupt = smsc_phy_ack_interrupt,
    .config_intr = smsc_phy_config_intr,

    .suspend = genphy_suspend,  // 复用通用休眠
    .resume = genphy_resume,
    .driver = { .owner = THIS_MODULE, }
}
```
末尾使用module_phy_driver(smsc_phy_driver)宏，完成数组中全部PHY驱动注册。

关键点：
1.phy_id_mask = 0xfffffff0：低4bit忽略匹配，兼容同系列不同小版本PHY；
2.部分回调复用内核genphy_*通用函数，只实现芯片特有逻辑，减少重复代码；
3.支持PHY硬件中断，也可回退到内核轮询检测链路变化。

---

## 69.5 网络驱动实验测试 & 69.6 单网卡使用 笔记整理

### 69.5.1 LAN8720 PHY驱动测试
1.驱动修改：参考37.4.3小节完成内核配置，开启Drivers for SMSC PHYs。
2.启动内核，dmesg日志会打印PHY驱动名称：SMSC LAN8710/LAN8720；执行ifconfig up启用网卡同样会输出该驱动名。
3.网络验证：开发板ping PC/Ubuntu主机IP，ping通代表网络链路收发正常。

调试命令
```bash
dmesg | grep -i phy
dmesg | grep -i eth
ethtool eth0
```

### 69.5.2 通用PHY驱动测试（Generic PHY）
注意：IMX6ULL平台LAN8720直接使用通用PHY驱动存在坑点，需要MAC层TX-CLK引脚配置（fec_main.c中fec_probe修改，参考37.4.3），不保证100%可用。

操作步骤：
1.内核配置关闭Drivers for SMSC PHYs，不编译LAN8720厂商驱动。
2.重新编译zImage，烧录启动。
3.dmesg看到驱动名Generic PHY，代表使用内核通用PHY驱动。

原理：MDIO总线匹配不到厂商phy_driver，降级匹配内核内置通用PHY驱动。部分PHY芯片硬件特殊行为，通用驱动无法完全适配，会出现丢包、链路识别异常。

### 69.5.3 DHCP功能配置（udhcpc动态获取IP）
busybox内置udhcpc客户端，用于向路由器DHCP服务器申请动态IP，必须配套default.script脚本，否则申请IP失败。

准备脚本文件
脚本源路径：busybox-1.29.0/examples/udhcp/simple.script

1.在根文件系统创建目录
```bash
mkdir -p rootfs/usr/share/udhcpc
```
2.复制并重命名脚本
```bash
cp busybox-1.29.0/examples/udhcp/simple.script rootfs/usr/share/udhcpc/default.script
chmod +x rootfs/usr/share/udhcpc/default.script
```

开发板端执行命令
```bash
ifconfig eth1 up               # 启用网卡
udhcpc -i eth1                 # 指定eth1网卡执行DHCP申请
```
脚本会自动配置IP、子网掩码、网关，同时更新/etc/resolv.confDNS配置。
使用ifconfig查看分配得到的IP地址。

常见故障：没有default.script → udhcpc收到DHCP报文但是不会配置网卡IP。

---

### 69.6 单网卡使用
IMX6ULL有两路MAC：ENET1(fec1)、ENET2(fec2)。产品引脚资源紧张时，关闭一路网口，释放IO给其他外设。

#### 69.6.1 只使用ENET2（fec2），关闭ENET1（fec1）
修改设备树：imx6ull-alientek-emmc.dts 将&fec1节点status设置为disabled
```dts
&fec1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_enet1 &pinctrl_enet1_reset>;
    phy-mode = "rmii";
    phy-handle = <&ethphy0>;
    phy-reset-gpios = <&gpio5 7 GPIO_ACTIVE_LOW>;
    phy-reset-duration = <200>;
    status = "disabled";      /* 关闭ENET1 */
};
```

编译dtb启动，执行
```bash
ifconfig -a
```
系统只有eth0（对应ENET2），eth1消失。pinctrl_enet1对应的IO就可以复用为别的外设。

#### 69.6.2 只使用ENET1（fec1），关闭ENET2（fec2）
⚠️ 不能仅仅简单设置fec2的status="disabled"，因为MDIO/MDC引脚默认归属ENET2，需要完整迁移MDIO子节点与pinmux配置。

步骤：
1.删除/屏蔽设备树中&fec2完整节点。
2.修改&fec1节点，内部增加mdio {}子节点，把PHY定义放到fec1内部mdio总线下。
```dts
&fec1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_enet1 &pinctrl_enet1_reset>;
    phy-mode = "rmii";
    phy-handle = <&ethphy0>;
    phy-reset-gpios = <&gpio5 7 GPIO_ACTIVE_LOW>;
    phy-reset-duration = <200>;
    status = "okay";

    mdio {
        #address-cells = <1>;
        #size-cells = <0>;
        ethphy0: ethernet-phy@0 {
            compatible = "ethernet-phy-ieee802.3-c22";
            reg = <0>; /* PHY地址0 */
        };
    };
};
```
3.删除/屏蔽pinctrl_enet2节点。
4.修改pinctrl_enet1，将GPIO1_IO07 / GPIO1_IO06复用为ENET1_MDC、ENET1_MDIO（原本这两个IO给ENET2使用）。
```dts
pinctrl_enet1: enet1grp {
    fsl,pins = <
        MX6UL_PAD_GPIO1_IO07__ENET1_MDC        0x1b0b0
        MX6UL_PAD_GPIO1_IO06__ENET1_MDIO       0x1b0b0
        MX6UL_PAD_ENET1_RX_EN__ENET1_RX_EN     0x1b0b0
        MX6UL_PAD_ENET1_RX_ER__ENET1_RX_ER     0x1b0b0
        MX6UL_PAD_ENET1_RX_DATA0__ENET1_RDATA00 0x1b0b0
        MX6UL_PAD_ENET1_RX_DATA1__ENET1_RDATA01 0x1b0b0
        MX6UL_PAD_ENET1_TX_EN__ENET1_TX_EN     0x1b0b0
        MX6UL_PAD_ENET1_TX_DATA0__ENET1_TDATA00 0x1b0b0
        MX6UL_PAD_ENET1_TX_DATA1__ENET1_TDATA01 0x1b0b0
        MX6UL_PAD_ENET1_TX_CLK__ENET1_REF_CLK1 0x4001b009
    >;
};
```
5.重新编译设备树，烧录启动。
执行ifconfig -a，系统仅存在eth0，此时eth0对应硬件ENET1，不要混淆编号。