# 第六十七章 Linux USB 驱动实验

USB 是很常用的接口，目前大多数的设备都是 USB 接口的，比如鼠标、键盘、USB 摄像头等，我们在实际开发中也常常遇到 USB 接口的设备，本章我们就来学习一下如何使能 Linux 内核自带的 USB 驱动。注意！本章并不讲解具体的 USB 开发，因为 USB 接口很复杂，不同的设备其协议也不同，这不是简简单单一章内容就能说完的，USB 驱动开发本身就是一门复杂的课程。所以，如果想要学习如何编写代码开发一个全新的 USB 设备那就可以跳过本章。

## 67.1 USB 接口简介

关于 USB 详细的协议内容请参考《USB2.0 协议中文版.pdf》和《USB3.0 协议中文版.pdf》，这两份文档已经放到了开发板光盘中，存放在 "4、参考资料" 中。

### 67.1.1 什么是 USB？

USB 全称为 Universal Serial Bus，翻译过来就是通用串行总线。由英特尔与众多电脑公司提出来，用于规范电脑与外部设备的连接与通讯。目前 USB 接口已经得到了大范围的应用，已经是电脑、手机等终端设备的必配接口，甚至取代了大量的其他接口。比如最新的智能手机均采用 USB Typec 取代了传统的 3.5mm 耳机接口，苹果最新的 MacBook 只有 USB Typec 接口，至于其他的 HDMI、网口等均可以通过 USB Typec 扩展坞来扩展。

按照大版本划分，USB 目前可以划分为 USB1.0、USB2.0、USB3.0 以及即将到来的 USB4.0。

- **USB1.0**：USB 规范于 1995 年第一次发布，由 Intel、IBM、Microsoft 等公司组成的 USB_IF (USB Implement Forum) 组织提出。USB_IF 于 1996 年正式发布 USB1.0，理论速度为 1.5Mbps。1998 年 USB_IF 在 USB1.0 的基础上提出了 USB1.1 规范。
- **USB2.0**：USB2.0 依旧由 Intel、IBM、Microsoft 等公司提出并发布，USB2.0 分为两个版本：Full_Speed 和 High_Speed，也就是全速 (FS) 和高速 (HS)。USB2.0 FS 的速度为 12Mbps，USB2.0 HS 速度为 480Mbps。目前大多数单片机以及低端 Cortex_A 芯片配置的都是 USB2.0 接口，比如 STM32 和 ALPHA 开发板所使用的 I.MX6ULL。USB2.0 全面兼容 USB1.0 标准。
- **USB3.0**：USB3.0 同样由 Intel 等公司发起的，USB3.0 最大理论传输速度为 5.0Gbps，USB3.0 引入了全双工数据传输，USB2.0 的 480Mbps 为半双工。USB3.0 中两根线用于发送数据，另外两根用于接收数据。在 USB3.0 的基础上又提出了 USB3.1、USB3.2 等规范，USB3.1 理论传输速度提升到了 10Gbps，USB3.2 理论传输速度为 20Gbps。为了规范 USB3.0 标准的命名，USB_IF 公布了最新的 USB 命名规范，原来的 USB3.0 和 USB3.1 命名将不会采用，所有的 3.0 版本的 USB 都命名为 USB3.2，以前的 USB3.0、USB3.1 和 USB3.2 分别叫做 USB3.2 Gen1、USB3.2 Gen2、USB3.2 Gen 2X2。
- **USB4.0**：目前还在标准定制中，目前还没有设备搭载，据说是在 Intel 的雷电接口上改进而来。USB4.0 的速度将提升到了 40Gbps，最高支持 100W 的供电能力，只需要一根线就可以完成数据传输与供电，极大的简化了设备之间的链接线数，期待 USB4.0 设备上市。

如果按照接口类型划分的话 USB 就要分为很多种了，最常见的就是 USB A 插头和插座，使用过 JLINK 调试器的朋友应该还见过 USB B 插头和插座。

USB 插头在不断的缩小，由此产生了 Mini USB 接口，正点原子的 I.MX6ULL_ALPHA 开发板使用的就是 Mini USB。

比 Mini USB 更小的就是 Micro USB 接口了，以前的智能手机基本都是 Micro USB 接口的，Micro USB 插头和插座。

### 67.1.2 USB 电气特性

由于正点原子 I.MX6U_ALPHA 开发板使用的 Mini USB 接口，因此我们就以 Mini USB 为例讲解一下 USB 的基本电气属性。Mini USB 线一般都是一头为 USB A 插头，一头为 Mini USB 插头。一共有四个触点，也就是 4 根线，这四根线的顺序如图 1-1。

![图1-1 USB A插头线序](photo/Linux%20USB驱动实验/1-1%20USB%20A插头线序.png)

USB A 插头从左到右线序依次为 1,2,3,4，第 1 根线为 VBUS，电压为 5V，第 2 根线为 D-，第 3 根线为 D+，第 4 根线为 GND。USB 采用差分信号来传输数据，因此有 D- 和 D+ 两根差分信号线。大家仔细观察的话会发现 USB A 插头的 1 和 4 这两个触点比较长，2 和 3 这两个触点比较短。1 和 4 分别为 VBUS 和 GND，也就是供电引脚，当插入 USB 的时候会先供电，然后再接通数据线。拔出的时候先断开数据线，然后再断开电源线。

大家再观察一下 Mini USB 插头，会发现 Mini USB 插头有 5 个触点，也就是 5 根线，线序从左往右依次是 1~5。第 1 根线为 VCC(5V)，第 2 根线为 D-，第 3 根线为 D+，第 4 根线为 ID，第 5 根线为 GND。可以看出 Mini USB 插头相比 USB A 插头多了一个 ID 线，这个 ID 线用于实现 OTG 功能，通过 ID 线来判断当前连接的是主设备(HOST)还是从设备(SLAVE)。

USB 是一种支持热插拔的总线接口，使用差分线(D- 和 D+)来传输数据，USB 支持两种供电模式：总线供电和自供电，总线供电就是由 USB 接口为外部设备供电，在 USB2.0 下，总线供电最大可以提供 500mA 的电流。

### 67.1.3 USB 拓扑结构

USB 是主从结构的，也就是分为主机和从机两部分，一般主机叫做 Host，从机叫做 Device。主机就是提供 USB A 插座来连接外部的设备，比如电脑作为主机，对外提供 USB A 插座，我们可以通过 USB 线来连接一些 USB 设备，比如声卡、手机等。因此电脑带的 USB A 插座数量就决定了你能外接多少个 USB 设备，如果不够用的话我们可以购买 USB 集线器来扩展电脑的 USB 插口，USB 集线器也叫做 USB HUB。

主机一般会带几个原生的 USB 主控制器，比如 I.MX6ULL 就有两个原生的 USB 主控制器，因此 I.MX6ULL 对外提供两个 USB 接口，这两个接口肯定不够用，正点原子的 ALPHA 开发板上有 4 个 HOST 接口，其中一路是 USB1 的 OTG 接口，其他的三路就是 USB2 通过 USB HUB 芯片扩展出来的，稍后我们会讲解其原理图。

虽然我们可以对原生的 USB 口数量进行扩展，但是我们不能对原生 USB 口的带宽进行扩展，比如 I.MX6ULL 的两个原生 USB 口都是 USB2.0 的，带宽最大为 480Mbps，因此接到下面的所有 USB 设备总带宽最大为 480Mbps。

USB 只能主机与设备之间进行数据通信，USB 主机与主机、设备与设备之间是不能通信的。因此两个正常通信的 USB 接口之间必定有一个主机，一个设备。为此使用了不同的插头和插座来区分主机与设备，比如主机提供 USB A 插座，从机提供 Mini USB、Micro USB 等插座。在一个 USB 系统中，仅有一个 USB 主机，但是可以有多个 USB 设备，包括 USB 功能设备和 USB HUB，最多支持 127 个设备。一个 USB 主控制器支持 128 个地址，地址 0 是默认地址，只有在设备枚举的时候才会使用，地址 0 不会分配给任何一个设备。所以一个 USB 主控制器最多可以分配 127 个地址。整个 USB 的拓扑结构就是一个分层的金字塔形，如图 1-2 所示，可以看出从 Root Hub 开始，一共有 7 层，金字塔顶部是 Root Hub，这个是 USB 控制器内部的。图中的 Hub 就是连接的 USB 集线器，Func 就是具体的 USB 设备。

![图1-2 USB金字塔拓扑](photo/Linux%20USB驱动实验/1-2%20USB金字塔拓扑.png)

USB 主机和从机之间的通信通过管道(Pipe)来完成，管道是一个逻辑概念，任何一个 USB 设备一旦上电就会存在一个管道，也就是默认管道，USB 主机通过管道来获取从机的描述符、配置等信息。在主机端管道其实就是一组缓冲区，用来存放主机数据，在设备端管道对应一个特定的端点。

### 67.1.4 什么是 USB OTG？

前面我们讲了，USB 分为 HOST(主机)和从机(或 DEVICE)，有些设备可能有时候需要做 HOST，有时候又需要做 DEVICE，配两个 USB 口当然可以实现，但是太浪费资源了。如果一个 USB 接口既可以做 HOST 又可以做 DEVICE 那就太好了，使用起来就方便很多。为此，USB OTG 应运而生，OTG 是 On_The_Go 的缩写，支持 USB OTG 功能的 USB 接口既可以做 HOST，也可以做 DEVICE。那么问题来了，一个 USB 接口如何知道应该工作在 HOST 还是 DEVICE 呢？这里就引入了 ID 线这个概念，前面讲解 USB 电气属性的时候已经说过了，Mini USB 插头有 5 根线，其中一条就是 ID 线。ID 线的高低电平表示 USB 口工作在 HOST 还是 DEVICE 模式：

- ID=1：OTG 设备工作在从机模式。
- ID=0：OTG 设备工作在主机模式。

支持 OTG 模式的 USB 接口一般都是那些带有 ID 线的接口，比如正点原子的 I.MX6ULL_ALPHA 开发板的 USB_OTG 接口就是支持 OTG 模式的，USB_OTG 连接到了 I.MX6ULL 的 USB1 接口上。如果只有一个 Mini USB 或者 Type_C USB 接口的话如果要使用 OTG 的主机模式，那么就需要一根 OTG 线。不管是 Mini USB OTG 还是 Type_C USB OTG 线，一头都是 USB A 插座，另外一头是 Mini USB 或者 Type_C USB 插头，将 Mini USB 或者 Type_C USB 插头插入机器的 Mini USB 或 Type_C 接口上，需要连接的 USB 设备插到另一端的 USB A 插座上，比如 U 盘啥的。USB OTG 线会将 ID 线拉低，这样机器就知道自己要做为一个主机，用来连接外部的从机设备(U 盘)。

### 67.1.5 I.MX6ULL USB 接口简介

I.MX6ULL 内部集成了两个独立的 USB 控制器，这两个 USB 控制器都支持 OTG 功能。I.MX6ULL 内部 USB 控制器特性如下：

①、有两个 USB2.0 控制器内核分别为 Core0 和 Core1，这两个 Core 分别连接到 OTG1 和 OTG2。
②、两个 USB2.0 控制器都支持 HS、FS 和 LS 模式，不管是主机还是从机模式都支持 HS/FS/LS，硬件支持 OTG 信号、会话请求协议和主机协商协议，支持 8 个双向端点。
③、支持低功耗模式，本地或远端可以唤醒。
④、每个控制器都有一个 DMA。

每个 USB 控制器都有两个模式：正常模式(normal mode)和低功耗模式(low power mode)。每个 USB OTG 控制器都可以运行在高速模式(HS 480Mbps)、全速模式(LS 12Mbps)和低速模式(1.5Mbps)。正常模式下每个 OTG 控制器都可以工作在主机(HOST)或从机(DEVICE)模式下，每个 USB 控制器都有其对应的接口。低功耗模式顾名思义就是为了节省功耗，USB2.0 协议中要求，设备在上行端口检测到空闲状态以后就可以进入挂起状态。在从机(DEVICE)模式下，端口停止活动 3ms 以后 OTG 控制器内核进入挂起状态。在主机(HOST)模式下，OTG 控制器内核不会自动进入挂起状态，但是可以通过软件设置。不管是本地还是远端的 USB 主从机都可以通过产生唤醒序列来重新开始 USB 通信。

两个 USB 控制器都兼容 EHCI，这里我们简单提一下 OHCI、UHCI、EHCI 和 xHCI，这三个是用来描述 USB 控制器规格的，区别如下：

- **OHCI**：全称为 Open Host Controller Interface，这是一种 USB 控制器标准，厂商在设计 USB 控制器的时候需要遵循此标准，用于 USB1.1 标准。OHCI 不仅仅用于 USB，也支持一些其他的接口，比如苹果的 Firewire 等，OHCI 由于硬件比较难，所以软件要求就降低了，软件相对来说比较简单。OHCI 主要用于非 X86 的 USB，比如扩展卡、嵌入式 USB 控制器。
- **UHCI**：全称是 Universal Host Controller Interface，UHCI 是 Intel 主导的一个用于 USB1.0/1.1 的标准，与 OHCI 不兼容。与 OHCI 相比 UHCI 硬件要求低，但是软件要求相应就高了，因此硬件成本上就比较低。
- **EHCI**：全称是 Enhanced Host Controller Interface，是 Intel 主导的一个用于 USB2.0 的 USB 控制器标准。I.MX6ULL 的两个 USB 控制器都是 2.0 的，因此兼容 EHCI 标准。EHCI 仅提供 USB2.0 的高速功能，至于全速和低速功能就由 OHCI 或 UHCI 来提供。
- **xHCI**：全称是 eXtensible Host Controller Interface，是目前最流行的 USB3.0 控制器标准，在速度、能效和虚拟化等方面比前三个都有较大的提高。xHCI 支持所有速度种类的 USB 设备，xHCI 出现的目的就是为了替换前面三个。

关于 I.MX6ULL 的 USB 控制器就简单的讲解到这里，至于更详细的内容请参考 I.MX6ULL 参考手册中的 "Chapter 56 Universal Serial Bus Controller(USB)" 章节。

## 67.2 硬件原理图分析

正点原子的 I.MX6ULL_ALPHA 开发板 USB 部分原理图可以分为两部分：USB HUB 以及 USB OTG，我们依次来看一下这两部分的硬件原理图。

### 67.2.1 USB HUB 原理图分析

#### 1、V2.4 版本以前底板 USB HUB 原理图分析

首先来看一下 USB HUB 原理图，I.MX6ULL_ALPHA 使用 GL850G 这个 HUB 芯片将 I.MX6ULL 的 USB OTG2 扩展成了 4 路 HOST 接口，其中一路供 4G 模块使用，因此就剩下了三个通用的 USB A 插座，原理图如图 1-3。

![图1-3 V2.4版本以前的底板hub原理图](photo/Linux%20USB驱动实验/1-3%20V2.4版本以前的底板hub原理图.png)

#### 2、V2.4 版本及以后底板 USB HUB 原理图分析

V2.4 及以后版本的 I.MX6ULL_ALPHA 使用 SL2.1A 这个 HUB 芯片将 I.MX6ULL 的 USB OTG2 扩展成了 4 路 HOST 接口，其中一路供 4G 模块使用，因此就剩下了三个通用的 USB A 插座，原理图如图 1-4。

![图1-4 V2.4以后底板hub原理图](photo/Linux%20USB驱动实验/1-4%20V2.4以后底板hub原理图.png)

图 1-3 和图 1-4 中的 GL850G 和 SL2.1A，这两个都是符合 USB2.0 标准的 USB HUB 芯片，支持一拖四扩展，可以将一路 USB 扩展为 4 路 USB HOST 接口。这里我们将 I.MX6ULL 的 USB OTG2 扩展出了 4 路 USB HOST 接口，分别为 HUB_DP1/DM1、HUB_DP2/DM2、HUB_DP3/DM3 和 HUB_DP4/DM4。其中 HUB_DP4/DM4 用于 4G 模块，因此对外提供的只有三个 USB HOST 接口。

> 注意，使用 GL850G 扩展出来的 4 路 USB 接口只能用作 HOST！

### 67.2.2 V2.4 版本以前底板 USB OTG 原理图分析

I.MX6U_ALPHA 开发板上还有一路 USB OTG 接口，使用 I.MX6ULL 的 USB OTG1 接口。此路 USB OTG 既可以作为主机(HOST)，也可以作为从机(DEVICE)，从而实现完整的 OTG 功能，原理图如图 1-5。左侧的为 Mini USB 插座，当 OTG 作为从机(DEVICE)的时候 USB 线接入此接口。右侧为 USB A 插座，当 OTG 作为主机的时候将 USB 设备插入到此接口中。前面我们讲了，如果只有一个 Mini USB 插座的话如果要学习 OTG 那么就需要再购买一个 OTG 线，这样不方便我们使用。为此正点原子在开发板上集成了一个 USB HOST 接口，这样在做 OTG 实验的时候就不需要再另外单独购买一根 USB OTG 线了。这里就涉及到硬件对 USB ID 线的处理，图 67.2.2.1 中 R111 和 R31 就是完成此功能的，我们分两部分来分析，即 OTG 分别工作在 HOST 和 DEVICE 的时候硬件工作方式：

![图1-5 OTG原理图](photo/Linux%20USB驱动实验/1-5%20OTG原理图.png)

**从机(DEVICE)模式**：图 67.2.2.1 中 USB_OTG_VBUS 是 Mini USB 的电源线，只有插入 Mini USB 线以后 USB_OTG_VBUS 才有效(5V)。插入 Mini USB 线就表示开发板此时要做从机(此时不考虑接 OTG 线的情况)，USB_OTG_VBUS 就是电脑供的 5V 电压，由于分压电阻 R111 和 R31 的作用，此时 USB_OTG1_ID 的电压就是 4.5V 左右，很明显这一个高电平。前面我们讲了，当 ID 线为高的时候就表示 OTG 工作在从机模式。

**主机(HOST)模式**：主机模式下必须将 Mini USB 线拔出来，将 USB 设备连接到对应的 USB HOST 接口上。Mini USB 线拔出来以后 USB_OTG_VBUS 就没有电压了，此时 USB_OTG1_ID 线就被 R31 这个 100K 电阻下拉到地，因此 USB_OTG1_ID 线的电压就为 0，当 ID 线为 0 的时候就表示 OTG 工作在主机模式。

优点就是省去了购买一根 Mini USB OTG 线的麻烦，方便我们学习开发，但是在使用的时候要注意一下几点：

①、我们需要软件设置 USB_OTG1_ID 这个 IO 的电气属性，默认设置为下拉，也就是默认工作在主机(HOST)模式下。
②、由于我们修改了 OTG 硬件电路，因此就不能在 Mini USB 接口上接 OTG 线了，如果要使用 HOST 功能就将设备插到开发板板载的 USB HOST 接口上。

I.MX6U_ALPHA 开发板上的 USB OTG 接口如图 1-6，上面的就是主机(HOST)接口，下面的是从机(DEVICE)接口，两个不能同时使用！

![图1-6 OTG的HOST和DEVICE](photo/Linux%20USB驱动实验/1-6%20OTG的HOST和DEVICE.png)

### 67.2.3 V2.4 及以后版本底板 USB OTG 原理图分析

I.MX6U_ALPHA 开发板上有一路 USB OTG 接口，使用 I.MX6ULL 的 USB OTG1 接口。此路 USB OTG 既可以作为主机(HOST)，也可以作为从机(DEVICE)，从而实现完整的 OTG 功能，原理图如图 1-7 所示：只有一个 Type_C USB 插座，当 OTG 作为从机(DEVICE)的时候 USB 线接入此接口。当 OTG 作为主机的时候需要使用 Type_C OTG 线。

![图1-7 2.4后的OTG原理图](photo/Linux%20USB驱动实验/1-7%202.4后的OTG原理图.png)

USB OTG 通过 ID 线来完成主从机切换，这里就涉及到硬件对 USB ID 线的处理，Type_C 的 CC1 和 CC2 引脚就是完成传统的 USB ID 线功能的。我们简单分析一下主从机切换原理。

**从机(DEVICE)模式**：图 67.2.3.1 中 R111 这一个 49.9K 的电阻，默认将 USB_OTG1_ID 线拉高，前面我们讲了，当 ID 线为高的时候就表示 OTG 工作在从机模式。此时由于 USB_OTG1_ID 为高电平，因此 MOS1(SI2302)导通，因此 MT9700HT5 的 EN 脚就接地，此时 MT9700HT5 的 OUT 引脚就没有输出，所以 USB_OTG_VBUS 电压关闭。USB_OTG_VBUS 电压用于在 OTG 做 HOST 功能的时候，向外部设备提供 5V 电源。很明显，在 OTG 做从机的时候，OTG 就不需要向外界提供 USB_OTG_VBUS 电源了。这里使用 MT9700HT5 这个芯片来实现 VBUS 电源的开关控制。

**主机(HOST)模式**：如果要使用 OTG 的 HOST 功能，那么必须要使用到 Type_C OTG 线。Type_C OTG 线会将 CC1 和 CC2 拉低，因此 USB_OTG1_ID 线也会被拉低，当 ID 线为 0 的时候就表示 OTG 工作在主机模式。此时由于 USB_OTG1_ID 为低，因此 MOS1(SI2302)不导通，因此 MT9700HT5 的 EN 脚就会被 R31 这个 10K 电阻上拉到 5V，所以 MT9700HT5 的 OUT 引脚就会输出 5V 电压，也就是说 USB_OTG_VBUS 此时是 5V，可以向外部设备提供 5V 电源。

## 67.3 USB 协议简析

USB 协议中有很多的基础概念，本节就来看一下这些概念。

### 67.3.1 USB 描述符

顾名思义，USB 描述符就是用来描述 USB 信息的，描述符就是一串按照一定规则构建的字符串，USB 设备使用描述符来向主机报告自己的相关属性信息，常用的描述符如表 67.3.1.1 所示：

**表 67.3.1.1 USB 设备常用描述符**

| 描述符类型 | 名字 | 值 |
|---|---|---|
| Device Descriptor | 设备描述符 | 1 |
| Configuration Descriptor | 配置描述符 | 2 |
| String Descriptor | 字符串描述符 | 3 |
| Interface Descriptor | 接口描述符 | 4 |
| Endpoint Descriptor | 端点描述符 | 5 |

我们依次来看一下表 67.3.1.1 中这 5 个描述符的含义：

#### 1、设备描述符

设备描述符用于描述 USB 设备的一般信息，USB 设备只有一个设备描述符。设备描述符里面记录了设备的 USB 版本号、设备类型、VID(厂商 ID)、PID(产品 ID)、设备序列号等。设备描述符结构如表 67.3.1.2 所示：

**表 67.3.1.2 设备描述符结构**

| 偏移 | 域 | 大小(B) | 值类型 | 描述 |
|---|---|---|---|---|
| 0 | bLength | 1 | 数字 | 此设备描述符长度，18 个字节 |
| 1 | bDescriptorType | 1 | 常量 | 描述符类型，为 0X01 |
| 2 | bcdUSB | 2 | BCD 码 | USB 版本号 |
| 4 | bDeviceClass | 1 | 类 | 设备类 |
| 5 | bDeviceSubClass | 1 | 子类 | 设备子类 |
| 6 | bDevicePortocol | 1 | 协议 | 设备协议 |
| 7 | bMaxPacketSize0 | 1 | 数字 | 端点 0 的最大包长度 |
| 8 | idVendor | 2 | ID | 厂商 ID |
| 10 | idProduct | 2 | ID | 产品 ID |
| 12 | bcdDevice | 2 | BCD 码 | 设备版本号 |
| 14 | iManufacturer | 1 | 索引 | 厂商信息字符串描述符索引值 |
| 15 | iProduct | 1 | 索引 | 产品信息字符串描述符索引值 |
| 16 | iSerialNumber | 1 | 索引 | 产品序列号字符串描述符索引值 |
| 17 | bNumConfigurations | 1 | 索引 | 可能的配置描述符数目 |

#### 2、配置描述符

设备描述符的 bNumConfigurations 域定义了一个 USB 设备的配置描述符数量，一个 USB 设备至少有一个配置描述符。配置描述符描述了设备可提供的接口(Interface)数量、配置编号、供电信息等，配置描述符结构如表 67.3.1.3 所示：

**表 67.3.1.3 配置描述符结构**

| 偏移 | 域 | 大小(B) | 值类型 | 描述 |
|---|---|---|---|---|
| 0 | bLength | 1 | 数字 | 此配置描述符长度，9 个字节 |
| 1 | bDescriptorType | 1 | 常量 | 配置描述符类型，为 0X02 |
| 2 | wTotalLength | 2 | 数字 | 整个配置信息总长度(包括配置、接口、端点、设备类和厂家定义的描述符) |
| 4 | bNumInterfaces | 1 | 数字 | 此配置所支持的接口数 |
| 5 | bConfigurationValue | 1 | 数字 | 该配置的值，一个设备支持多种配置，通过配置值来区分不同的配置。 |
| 6 | bConfiguration | 1 | 数字 | 描述此配置的字符串描述索引 |
| 7 | bmAttributes | 1 | 数字 | 该设备的属性：D7：保留；D6：自给电源；D5：远程唤醒；D4:0：保留 |
| 8 | bMaxPower | 1 | 数字 | 此配置下所需的总线电流(单位 2mA) |

#### 3、字符串描述符

字符串描述符是可选的，字符串描述符用于描述一些方便人们阅读的信息，比如制造商、设备名称啥的。如果一个设备没有字符串描述符，那么其他描述符中和字符串有关的索引值都必须为 0，字符串描述符结构如表 67.3.1.4 所示：

**表 67.3.1.4 字符串描述符结构**

| 偏移 | 域 | 大小(B) | 值类型 | 描述 |
|---|---|---|---|---|
| 0 | bLength | 1 | N+2 | 此字符串描述符长度 |
| 1 | bDescriptorType | 1 | 常量 | 字符串描述符类型，为 0X03 |
| 2 | wLANGID[0] | 2 | 数字 | 语言标识码 0 |
| … | … | 2 | 数字 | … |
| N | wLANGID[x] | 2 | 数字 | 语言标识码 x |

此文档已经放到了开发板光盘中，路径为：4、参考资料-> USB_LANGIDs.pdf。

主机会再次根据自己所需的语言向设备请求字符串描述符，这次会主机会指明要得到的字符串索引值和语言。设备返回 Unicode 编码的字符串描述符，结构如表 67.3.1.5 所示：

**表 67.3.1.5 Unicode 编码的字符串描述符结构**

| 偏移 | 域 | 大小(B) | 值类型 | 描述 |
|---|---|---|---|---|
| 0 | bLength | 1 | N+2 | 此字符串描述符长度 |
| 1 | bDescriptorType | 1 | 常量 | 字符串描述符类型，为 0X03 |
| 2 | bString | N | 数字 | Unicode 编码的字符串 |

#### 4、接口描述符

配置描述符中指定了该配置下的接口数量，可以提供一个或多个接口，接口描述符用于描述接口属性。接口描述符中一般记录接口编号、接口对应的端点数量、接口所述的类等，接口描述符结构如表 67.3.1.6 所示：

**表 67.3.1.6 接口描述符结构**

| 偏移 | 域 | 大小(B) | 值类型 | 描述 |
|---|---|---|---|---|
| 0 | bLength | 1 | 数字 | 此接口描述符长度，9 个字节 |
| 1 | bDescriptorType | 1 | 常量 | 描述符类型，为 0X04 |
| 2 | bInterfaceNumber | 1 | 数字 | 当前接口编号，从 0 开始 |
| 3 | bAlternateSetting | 1 | 数字 | 当前接口备用编号 |
| 4 | bNumEndpoints | 1 | 数字 | 当前接口的端点数量 |
| 5 | bInterfaceClass | 1 | 类 | 当前接口所属的类 |
| 6 | bInterfaceSubClass | 1 | 子类 | 当前接口所属的子类 |
| 7 | bInterfaceProtocol | 1 | 协议 | 当前接口所使用的协议 |
| 8 | iInterface | 1 | 索引 | 当前接口字符串的索引值 |

#### 5、端点描述符

接口描述符定义了其端点数量，端点是设备与主机之间进行数据传输的逻辑接口，除了端点 0 是双向端口，其他的端口都是单向的。端点描述符描述了传输类型、方向、数据包大小、端点号等信息，端点描述符结构如表 67.3.1.7 所示：

**表 67.3.1.7 端点描述符结构**

| 偏移 | 域 | 大小(B) | 值类型 | 描述 |
|---|---|---|---|---|
| 0 | bLength | 1 | 数字 | 此端点描述符长度，7 个字节 |
| 1 | bDescriptorType | 1 | 常量 | 描述符类型，为 0X05 |
| 2 | bEndpointAddress | 1 | 数字 | 端点地址和方向：bit3:0：端点号；bit6:4：保留，为零；bit7：方向，0 输出端点(主机到设备)，1 输入端点(设备到主机) |
| 3 | bmAttributes | 1 | 数字 | 端点属性，bit1:0 表示传输类型：00：控制传输；01：同步传输；10：批量传输；11：中断传输；其他位保留 |
| 4 | wMaxPacketSize | 2 | 数字 | 端点能发送或接收的最大数据包长度 |
| 6 | bInterval | 1 | 子类 | 端点数据传输中周期时间间隙值，此域对于批量传输和控制传输无效，同步传输的话此域必须为 1ms，中断传输此域可以设置 1ms~255ms。 |

### 67.3.3 USB 数据包类型

USB 是串行通信，需要一位一位的去传输数据，USB 传输的时候先将原始数据进行打包，所以 USB 中传输的基本单元就是数据包。根据用途的不同，USB 协议定义了 4 种不同的包结构：令牌(Token)包、数据(Data)包、握手(Handshake)包和特殊(Special)包。这四种包通过包标识符 PID 来区分，PID 共有 8 位，USB 协议使用低 4 位 PID3~PID0，另外的高四位 PID7~PID4 是 PID3~PID0 的取反，传输顺序是 PID0、PID1、PID2、PID3…PID7。令牌包的 PID1~0 为 01，数据包的 PID1~0 为 11，握手包的 PID1~0 为 10，特殊包的 PID1~0 为 00。每种类型的包又有多种具体的包，如表 67.3.3.1 所示：

**表 67.3.3.1 数据包结构**

| PID 类型 | PID 名字 | PID\<3:0\> | 描述 |
|---|---|---|---|
| 令牌包 | OUT | 0001 | 主机向从机通知将要输出数据 |
| 令牌包 | IN | 1001 | 主机向从机通知将要输入数据 |
| 令牌包 | SOF | 0101 | 帧起始包 |
| 令牌包 | SETUP | 1101 | 开始一个控制传输 |
| 数据包 | DATA0 | 0011 | 偶数据包 |
| 数据包 | DATA1 | 1011 | 奇数据包 |
| 数据包 | DATA2 | 0111 | 适用于高速和等时传输的数据包 |
| 数据包 | MDATA | 1111 | |
| 握手包 | ACK | 0010 | 确认包，传输正确 |
| 握手包 | NAK | 1010 | 不确认包，传输错误 |
| 握手包 | STALL | 1110 | 设备不支持该请求，或者端点挂起 |
| 握手包 | NYET | 0110 | 接收未响应，未准备好 |
| 特殊包 | PRE | 1100 | 前导，令牌包 |
| 特殊包 | ERR | 1100 | 错误，握手包 |
| 特殊包 | SPLIT | 1000 | 分裂传输(Split)，令牌包 |
| 特殊包 | PING | 0100 | PING 测试，令牌包 |
| 特殊包 | Reserved | 0000 | 保留 |
一个完整的包分为多个域，所有的数据包都是以同步域(SYNC)开始，后面紧跟着包标识符(PID)，最终都以包结束(EOP)信号结束。不同的数据包中间位域不同，一般有包目标地址(ADDR)、包目标端点(ENDP)、数据、帧索引、CRC 等，这个要具体数据包具体分析。接下来简单看一下这些数据包的结构。

**1、令牌包**

令牌包结构如图 67.3.3.1 所示：

| 域 | 示例 |
|---|---|
| SYNC | 00000001 |
| SETUP(8b) | 0XB4 |
| ADDR(7b) | 0 |
| ENDP(4b) | 0 |
| CRC5(5b) | 0X08 |
| EOP | |

图 67.3.3.1 是一个 SETUP 令牌包结构，首先是 SYNC 同步域，包同步域为 00000001，也就是连续 7 个 0，后面跟一个 1，如果是高速设备的话就是 31 个 0 后面跟一个 1。紧跟着是 PID，这里是 SETUP 包，为 0XB4，大家可能会好奇为什么不是 0X2D(00101101)，0XB4 的原因如下：

①、SETUP 包的 PID3~PID0 为 1101，因此对应的 PID7~PID4 就是 0010。
②、PID 传输顺序为 PID0、PID1、PID2…PID7，因此按照传输顺序排列的话此处的 PID 就是 10110100=0XB4，并不是 0X2D。

PID 后面跟着地址域(ADDR)和端点域(ENDP)，为目标设备的地址和端点号。CRC5 域是 5 位 CRC 值，是 ADDR 和 ENDP 这两个域的校验值。最后就是包结束域(EOP)，标记本数据包结束。其他令牌包的结构和 SETUP 基本类似，只是 SOF 令牌包中间没有 ADDR 和 ENDP 这两个域，而是只有一个 11 位的帧号域。

**2、数据包**

数据包结构如图 67.3.3.2 所示：

| 域 | 示例 |
|---|---|
| SYNC | 00000001 |
| DATA0(8b) | 0XC3 |
| DATA | 字节0、字节1 … 字节N |
| CRC16 | 0 |
| EOP | |

图 67.3.3.2 数据包

数据包比较简单，同样的，数据包从 SYNC 同步域开始，然后紧跟着是 PID，这里就是 DATA0，PID 值为 0XC3。接下来就是具体的数据，数据完了以后就是 16 位的 CRC 校验值，最后是 EOP。

**3、握手包**

握手包结构如图 67.3.3.3 所示：

| 域 | 示例 |
|---|---|
| SYNC | 00000001 |
| ACK(8b) | 0X4B |
| EOP | |

图 67.3.3.3 ACK 握手包

图 67.3.3.3 是 ACK 握手包，很简单，首先是 SYNC 同步域，然后就是 ACK 包的 PID，为 0X4B，最后就是 EOP。其他的 NAK、STALL、NYET 和 ERR 握手包结构都是一样的，只是其中的 PID 不同而已。

### 67.3.4 USB 传输类型

在端点描述符中 bmAttributes 指定了端点的传输类型，一共有 4 种，本节我们来看一下这四种传输类型的区别。

**1、控制传输** 控制传输一般用于特定的请求，比如枚举过程就是全部由控制传输来完成的，比如获取描述符、设置地址、设置配置等。控制传输分为三个阶段：建立阶段(SETUP)、数据阶段(DATA)和状态阶段(STATUS)，其中数据阶段是可选的。建立阶段使用 SETUP 令牌包，SETUP 使用 DATA0。数据阶段所有数据包必须是同一个方向的，比如都为 IN 或都为 OUT。数据阶段的第一个数据包必须是 DATA1，每次正确传输以后就在 DATA0 和 DATA1 之间进行切换。数据阶段完成以后就是状态阶段，状态阶段的传输方向要和数据阶段相反，比如数据阶段为 IN 的话状态阶段就要为 OUT，状态阶段使用 DATA1 包。比如一个读控制传输格式，如图 2-1 读控制传输阶段。

![图2-1 读控制传输阶段](photo/Linux%20USB驱动实验/2-1%20读控制传输阶段.png)

**2、同步传输** 同步传输用于周期性、低时延、数据量大的场合，比如音视频传输，这些场合对于时延要求很高，但是不要求数据 100%正确，允许有少量的错误。因此，同步传输没有握手阶段，即使数据传输出错了也不会重传。

**3、批量传输** 提起"批量"，我们第一反应就是"多"、"大"等，因此，批量传输就是用于大批量传输大块数据的，这些数据对实时性没有要求，比如 MSD 类设备(存储设备)，U 盘之类的。批量传输分为批量读(输入)和批量写(输出)，如果是批量读的话第一阶段发送 IN 令牌包，如果是批量写那么第一阶段就是 OUT 令牌包。

我们就以批量写为例简单介绍一下批量传输过程：

①、主机发出 OUT 令牌包，令牌包里面包含了设备地址、端点等信息。
②、如果 OUT 令牌包正确的话，也就是设备地址和端点号匹配，主机就会向设备发送一个数据(DATA)包，发送完成以后主机进入接收模式，等待设备返回握手包，一切都正确的话设备就会向主机返回一个 ACK 握手信号。

批量读的过程刚好相反：

①、主机发出 IN 令牌包，令牌包里面包含了设备地址、端点等信息。发送完成以后主机就进入到数据接收状态，等待设备返回数据。
②、如果 IN 令牌包正确的话，设备就会将一个 DATA 包放到总线上发送给主机。主机收到这个 DATA 包以后就会向设备发送一个 ACK 握手信号。

**4、中断传输** 这里的中断传输并不是我们传统意义上的硬件中断，而是一种保持一定频率的传输，中断传输适用于传输数据量小、具有周期性并且要求响应速度快的数据，比如键盘、鼠标等。中断的端点会在端点描述符中报告自己的查询时间间隔，对于时间要求严格的设备可以采用中断传输。

### 67.3.5 USB 枚举

当 USB 设备与 USB 主机连接以后主机就会对 USB 设备进行枚举，通过枚举来获取设备的描述符信息，主机得到这些信息以后就知道该加载什么样的驱动、如何进行通信等。USB 枚举过程如下：

①、第一回合，当 USB 主机检测到 USB 设备插入以后主机会发出总线复位信号来复位设备。USB 设备复位完成以后地址为 0，主机向地址 0 的端点 0 发送数据，请求设备的描述符。设备得到请求以后就会按照主机的要求将设备描述符发送给主机，主机得到设备发送过来的设备描述符以后，如果确认无误就会向设备返回一个确认数据包(ACK)。

②、第二回合，主机再次复位设备，进入地址设置阶段。主机向地址 0 的端点 0 发送设置地址请求数据包，新的设备地址就包含在这个数据包中，因此没有数据过程。设备进入状态过程，等待主机请求状态返回，收到以后设备就会向主机发送一个 0 字节状态数据包，表明设备已经设置好地址了，主机收到这个 0 字节状态数据包以后会返回一个确认包(ACK)。设备收到主机发送的 ACK 包以后就会使用这个新的设备地址，至此设备就得到了一个唯一的地址。

③、第三回合，主机向新的设备地址端点 0 发送请求设备描述符数据包，这一次主机要获取整个设备描述符，一共是 18 个字节。

④、和第③步类似，接下来依次获取配置描述符、配置集合、字符串描述符等等。

## 67.4 Linux 内核自带 HOST 实验

### 67.4.1 USB 鼠标键盘测试

首先做一下 USB HOST 试验，也就是 I.MX6U_ALPHA 开发板做 USB 主机，然后外接 USB 设备，比如 USB 鼠标键盘、USB 转 TTL 串口线、U 盘等设备。Linux 内核已经集成了大量的 USB 设备驱动，尤其是我们常见的 USB 鼠标键盘、U 盘等，本节我们就来学习一下如何使能 Linux 内核常见的 USB 设备驱动。

**1、USB 鼠标键盘驱动使能**

> 注意，NXP 官方的 Linux 内核默认已经使能了 USB 键盘鼠标驱动！

USB 鼠标键盘属于 HID 设备，内核已经集成了相应的驱动，NXP 官方提供的 linux 内核默认已经使能了 USB 鼠标键盘驱动，但是我们还要学习一下如何手动使能这些驱动。输入 `make menuconfig` 打开 linux 内核配置界面，首先打开 HID 驱动，按照如下路径到相应的配置项目：
```
-> Device Drivers
-> HID support
-> HID bus support (HID [=y])
-> <*> Generic HID driver
```

使能以后如图 3-2，大家可以将光标放到图 3-2 中"USB HID Transport layer"这一行，然后按下"?"键打开对应的帮助信息就可以看到对于这个配置项的描述，简单总结一下：

此选项对应配置项就是 CONFIG_USB_HID，也就是 USB 接口的 HID 设备。如果要使用 USB 接口的 keyboards(键盘)、mice(鼠标)、joysticks(摇杆)、graphic tablets(绘图板)等其他的 HID 设备，那么就需要选中"USB HID Transport layer"。但是要注意一点，此驱动和 HIDBP(Boot Protocol)键盘、鼠标的驱动不能一起使用！所以大家要是在网上查阅 linux 内核 USB 键盘鼠标驱动的时候，发现推荐使用"USB HIDBP Keyboard (simple Boot) support"和"USB HIDBP Mouse (simple Boot) support"这两个配置项的时候也不要觉得教程这里写错了。

![图3-2 使能USB鼠标键盘驱动](photo/Linux%20USB驱动实验/3-2%20使能USB鼠标键盘驱动.png)

**2、测试 USB 鼠标和键盘**

完成以后重新编译 linux 内核并且使用得到的 zImage 启动开发板。启动以后插入 USB 鼠标，会有如图 4-1 所示，可以看出，系统检测到了 Logitech(罗技)的鼠标，如果成功驱动的话就会在 `/dev/input` 目录下生成一个名为 eventX(X=0,1,2,3…)的文件，这个就是我们前面讲的输入子系统，鼠标和键盘都是作为输入子系统设备的。笔者这里对应的就是 `/dev/input/event3` 这个设备，使用如下命令查看鼠标的原始输入值，结果如图 4-2 就是鼠标作为输入子系统设备的原始输入值，这里就不去分析了，我们在移植 GUI 图形库以后就可以直接使用鼠标，比如 QT 等。

![图4-1 USB鼠标log信息](photo/Linux%20USB驱动实验/4-1%20USB鼠标log信息.png)

![图4-2 鼠标原始输入值](photo/Linux%20USB驱动实验/4-2%20鼠标原始输入值.png)

> 注意，有些鼠标可能会出现隔一段时间自动断开重连的现象，比如我测试的电脑城送的几块钱的罗技鼠标就会这样，我用 NXP 官方的 EVK 开发板测试也有这个问题！我自己用的雷蛇鼠标就不会有这种现象。但是，如果你一直使用这个鼠标，比如用 `hexdump` 命令一直查看鼠标上报值，或者在 QT 里面一直使用鼠标，那么这些鼠标就不会自动断开并重连。

最后再来测试一下 USB 键盘，屏幕已经驱动起来了，所以我们可以直接将屏幕作为终端，然后接上键盘直接输入命令来进行各种操作。首先将屏幕设置为控制台，打开开发板根文件系统中的 `/etc/inittab` 文件，然后在里面加入下面这一行：
```bash
tty1::askfirst:-/bin/sh
```

完成以后重启开发板，此时屏幕就会作为终端控制台，会有"Please press Enter to activate this console."这样提示，如图 4-3。接上键盘，然后根据图 4-3 提示，按下键盘上的 Enter(回车)键即可使能 LCD 屏幕控制台，然后我们就可以输入各种命令来执行相应的操作，如图 4-4。

![图4-3 终端控制台提示](photo/Linux%20USB驱动实验/4-3%20终端控制台提示.png)

![图4-4 键盘操作终端](photo/Linux%20USB驱动实验/4-4%20键盘操作终端.png)

### 67.4.2 U 盘实验

> 注意，NXP 官方的 Linux 内核默认已经使能了 U 盘！

NXP 提供的 Linux 内核默认也已经使能了 U 盘驱动，因此我们可以直接插上去使用。但是我们还是需要学习一下如何手动配置 Linux 内核，使能 U 盘驱动。

**1、使能 U 盘驱动**

U 盘使用 SCSI 协议，因此要先使能 Linux 内核中的 SCSI 协议，配置路径如下：
```
-> Device Drivers
-> SCSI device support
-> <*> SCSI disk support
```

![图4-5 使能SCSI](photo/Linux%20USB驱动实验/4-5%20使能SCSI.png)

我们还需要使能 USB Mass Storage，也就是 USB 接口的大容量存储设备，配置路径如下：
```
-> Device Drivers
-> USB support (USB_SUPPORT [=y])
-> Support for Host‑side USB (USB [=y])
-> <*> USB Mass Storage support
```

![图4-6 使能USB大容量存储设备](photo/Linux%20USB驱动实验/4-6%20使能USB大容量存储设备.png)

**2、U 盘测试**

准备好一个 U 盘，注意 U 盘要为 FAT32 格式的！NTFS 和 exFAT 由于版权问题所以在 Linux 下支持的不完善，操作的话可能会有问题，比如只能读，不能写或者无法识别等。准备好以后将 U 盘插入到开发板 USB HUB 扩展出来的 HOST 接口上，此时会输出如图 4-7。可以看出，系统检测到 U 盘插入，大小为 32GB，对应的设备文件为 `/dev/sda` 和 `/dev/sda1`，大家可以查看一下 `/dev` 目录下有没有 sda 和 sda1 这两个文件。`/dev/sda` 是整个 U 盘，`/dev/sda1` 是 U 盘的第一个分区，我们一般使用 U 盘的时候都是只有一个分区。要想访问 U 盘我们需要先对 U 盘进行挂载，理论上挂载到任意一个目录下都可以，这里我创建一个 `/mnt/usb_disk` 目录，然后将 U 盘挂载到 `/mnt/usb_disk` 目录下，命令如下：

```bash
mkdir /mnt/usb_disk -p                     #创建目录
mount /dev/sda1 /mnt/usb_disk/ -t vfat -o iocharset=utf8    #挂载
```

- `-t` 指定挂载所使用的文件系统类型，这里设置为 vfat，也就是 FAT 文件系统。
- `-o iocharset` 设置硬盘编码格式为 utf8，否则的话 U 盘里面的中文会显示乱码！

![图4-7 U盘log信息](photo/Linux%20USB驱动实验/4-7%20U盘log信息.png)

挂载成功以后进入到 `/mnt/usb_disk` 目录下，输入 `ls` 命令查看 U 盘文件，如图 4-8。至此 U 盘就能正常读写操作了，直接对 `/mnt/usb_disk` 目录进行操作就行了。如果要拔出 U 盘要执行一个 `sync` 命令进行同步，然后再使用 `umount` 进行 U 盘卸载，命令如下所示：

![图4-8 U盘文件](photo/Linux%20USB驱动实验/4-8%20U盘文件.png)

```bash
sync                     #同步
cd /                     #如果处于/mnt/usb_disk目录的话先退出来，否则卸载提示设备忙，卸载失败，切记！
umount /mnt/usb_disk     #卸载
```

## 67.5 Linux 内核自带 USB OTG 实验

### 67.5.1 修改设备树

> 注意，如果使用的是正点原子 I.MX6U_ALPHA 开发板，那么就需要修改 OTG ID 引脚的电气属性。如果使用的其他 6ULL 开发板，就要去咨询一下厂商，看看需不需要修改 ID 引脚的电气属性。

查阅原理图可以知道，USB OTG1 的 ID 引脚连接到了 I.MX6ULL 的 GPIO1_IO00 这个引脚上，在 67.2.2 小节分析 ALPHA 开发板 USB OTG 原理图的时候已经说过了，USB OTG 默认工作在主机(HOST)模式下，因此 ID 线应该是低电平。这里需要修改设备树中 GPIO1_IO00 这个引脚的电气属性，将其设置为默认下拉。打开设备树 `imx6ull_alientek_emmc.dts`，在 iomuxc 节点的 pinctrl_hog_1 子节点下添加 GPIO1_IO00 引脚信息，如下所示：

**示例代码 67.5.1.1 GPIO1_IO00 引脚描述信息**

```c
1 &iomuxc {
2     pinctrl_names = "default";
3     pinctrl_0 = <&pinctrl_hog_1>;
4     imx6ul_evk {
5         pinctrl_hog_1: hoggrp_1 {
6             fsl,pins = <
7                 ......
8                 MX6UL_PAD_GPIO1_IO00__ANATOP_OTG1_ID 0x13058 /*OTG1 ID */
9             >;
10        };
11        ......
12    };
```

第 8 行就是将 GPIO1_IO00 复用为 OTG1 ID，并且设置电气属性为 0X13058，默认下拉，设备树修改好以后重新编译并用新的设备树启动系统。

### 67.5.2 OTG 主机实验

系统重启成功以后就可以正常使用 USB OTG1 接口，OTG 既可以做主机，也可以做从机，做主机的话测试方法和 67.4 小节一模一样。如果是 V2.4 版本以前的底板，直接在 ALPHA 的 OTG HOST 接口上插入 USB 鼠标键盘、U 盘等设备。如果是 V2.4 及以后版本底板，要使用 Type-C OTG 线连接 USB 鼠标键盘，U 盘等设备。

> 注意！如果使用正点原子的 V2.4 版本以前的 ALPHA 开发板，切记不要使用 Mini OTG 线来外接 USB 设备，原因已经在 67.2.2 小节说明了，只需要将 USB 设备插入到开发板上的 OTG HOST 接口上即可！

### 67.5.3 OTG 从机实验

OTG 从机就是将开发板作为一个 USB 设备连接到其他的主机上，这里我们来做两个 USB 从机实验：模拟 U 盘以及 USB 声卡。

**1、模拟 U 盘实验**

模拟 U 盘实验就是将开发板当做一个 U 盘，可以将开发板上的 U 盘或者 TF 卡挂载到 PC 上去，首先需要配置 Linux，配置路径如下：
```
-> Device Drivers
-> USB support (USB_SUPPORT [=y])
-> USB Gadget Support (USB_GADGET [=y])
-> [M]USB Gadget Drivers (<choice> [=m])
	//选中 USB Gadget 驱动
-> [M]Mass Storage Gadget
	//大容量存储
```

![图5-1 USB Gadget 中大容量存储设备驱动](photo/Linux%20USB驱动实验/5-1%20USB%20Gadget%20中大容量存储设备驱动.png)

这里我们需要将驱动编译为模块！使用的时候直接输入命令加载驱动模块即可。配置好以后重新编译 Linux 内核，会得到三个 `.ko` 驱动模块(带路径)：
```
drivers/usb/gadget/libcomposite.ko
drivers/usb/gadget/function/usb_f_mass_storage.ko
drivers/usb/gadget/legacy/g_mass_storage.ko
```

将上述三个 `.ko` 模块拷贝到开发板根文件系统中，命令如下：

```bash
cd drivers/usb/gadget/
#进入 gadget 目录下
sudo cp libcomposite.ko /home/zuozhongkai/linux/nfs/rootfs/lib/modules/4.1.15/
sudo cp function/usb_f_mass_storage.ko /home/zuozhongkai/linux/nfs/rootfs/lib/modules/4.1.15/
sudo cp legacy/g_mass_storage.ko /home/zuozhongkai/linux/nfs/rootfs/lib/modules/4.1.15/
```

拷贝完成以后使用新编译出来的 zImage 启动开发板，在开发板上插入一个 U 盘，记住这个 U 盘对应的设备文件，比如我们这里是 `/dev/sda` 和 `/dev/sda1`，以后要将 `/dev/sda1` 挂载到 PC 上，也就是把 `/dev/sda1` 作为模拟 U 盘的存储区域。

V2.4 版本以前的底板使用 Mini USB 线将开发板的 USB OTG 接口与电脑连接起来，如图 5-2。连接好以后依次加载 `libcomposite.ko`、`usb_f_mass_storage.ko` 和 `g_mass_storage.ko` 这三个驱动文件，顺序不能错了！命令如下：

```bash
depmod
modprobe libcomposite.ko
modprobe usb_f_mass_storage.ko
modprobe g_mass_storage.ko file=/dev/sda1 removable=1
```

![图5-2 usb type-c接口](photo/Linux%20USB驱动实验/5-2%20usb%20type-c接口.png)

加载 `g_mass_storage.ko` 的时候使用 file 参数指定使用的大容量存储设备，我这里使用 U 盘对应的 `/dev/sda1`。如果加载成功的话电脑就会出现一个 U 盘，这个 U 盘就是我们开发板模拟的，如图 5-3。我们可以直接在电脑上对这个 U 盘进行读写操作，实际上操作的就是插在开发板上的 U 盘。操作完成以后要退出的话执行如下命令：

```bash
rmmod g_mass_storage.ko
```

![图5-3 电脑识别u盘](photo/Linux%20USB驱动实验/5-3%20电脑识别u盘.png)

> 注意！不要将开发板上的 EMMC 或者 NAND 作为模拟 U 盘的存储区域，因为 linux 下 EMMC 和 NAND 使用的文件系统一般都是 EXT3/EXT4 和 UBIFS，这些文件系统类型和 windows 下的不兼容，如果挂载的话就会在 windows 下提示要你格式化 U 盘！

**2、USB 声卡实验**

USB 声卡就是 USB 接口的外置声卡，一般电脑内部都自带了声卡，但是内部自带的声卡效果相对来说比较差，不能满足很多 HIFI 玩家的需求。USB 声卡通过 USB 接口来传递音频数据，具体的 ADC 和 DAC 过程由声卡完成，摆脱了电脑主板体积的限制，外置 USB 声卡就可以做的很好。ALPHA 开发板板载了音频解码芯片，因此可以将 ALPHA 开发板作为一个外置 USB 声卡，配置 Linux 内核，配置路径如下：
```
-> Device Drivers
-> USB support (USB_SUPPORT [=y])
-> USB Gadget Support (USB_GADGET [=y])
-> [M]USB Gadget Drivers
	//选中 USB Gadget 驱动
-> [M] Audio Gadget
	//选中音频
->UAC 1.0 (Legacy)
	//选中 UAC
```

配置如图 5-4。注意，这里也是编译为驱动模块，配置完成以后重新编译内核，得到新的 zImage 和三个 `.ko` 驱动模块文件：

![图5-4 使能USB Gadget下音频驱动](photo/Linux%20USB驱动实验/5-4%20使能USB%20Gadget下音频驱动.png)
```
drivers/usb/gadget/libcomposite.ko
drivers/usb/gadget/function/usb_f_uac1.ko
drivers/usb/gadget/legacy/g_audio.ko
```

将上述三个 `.ko` 模块拷贝到开发板根文件系统中，命令如下：

```bash
cd drivers/usb/gadget/
sudo cp libcomposite.ko /home/zuozhongkai/linux/nfs/rootfs/lib/modules/4.1.15/
sudo cp function/usb_f_uac1.ko /home/zuozhongkai/linux/nfs/rootfs/lib/modules/4.1.15/
sudo cp legacy/g_audio.ko /home/zuozhongkai/linux/nfs/rootfs/lib/modules/4.1.15/
```

拷贝完成以后使用新编译出来的 zImage 启动开发板，首先按照 65.5.2 小节讲解的方法配置 ALPHA 的声卡，保证声卡播放正常！使用 Mini USB 线将开发板与电脑连接起来，最后依次加载 `libcomposite.ko`、`usb_f_uac1.ko` 和 `g_audio.ko` 这三个驱动模块，命令如下：

```bash
depmod
modprobe libcomposite.ko
modprobe usb_f_uac1.ko
modprobe g_audio.ko
```

加载完成以后稍等一会虚拟出一个 USB 声卡，打开电脑的设备管理器，选择"声音、视频和游戏控制器"，会发现有一个名为"AC Interface"设备就是开发板模拟出来的 USB 声卡，设置 windows，选择音频输出使用"AC Interface"，设置好以后就可以从开发板上听到电脑输出的声音，此时开发板就完全是一个 USB 声卡设备了。

---

关于 USB 驱动就讲解到这里，本章并没有深入到 USB 驱动具体编写方式，只是对 USB 的协议做了简单的介绍，后面讲解了一下 Linux 内核自带的 USB HOST 和 DEVICE 驱动的使用，Linux 内核已经集成了大量的 USB 设备驱动，至于其他特殊的就需要具体情况具体分析了，比如本教程后面讲解的 USB WIFI 和 4G 模块驱动。

