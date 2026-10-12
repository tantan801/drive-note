# 第七十一章 Linux 4G 通信实验

前面我们学习了如何在 Linux 中使用有线网络或者 WIFI，但是使用有线网络或者 WIFI 有很多限制，因为要布线，即使是 WIFI 你也得先布线，然后再接个路由器。有很多场合是不方便布线的，这个时候就是 4G 大显身手的时候，产品可以直接通过 4G 连接到网络，实现无人值守。本章我们就来学一下如何在 I.MX6U-ALPHA 开发板中使用 4G 来实现联网功能。

## 71.1 4G 网络连接简介

提起 4G 网络连接，大家可能会觉得是个很难的东西，其实对于嵌入式 Linux 而言，4G 网络连接恰恰相反，不难！大家可以看一下其他的嵌入式 Linux 或者 Android 开发板，4G 模块都是 MiniPCIE 接口的，包括很多 4G 模块都是 MiniPCIE 接口的。但是大家稍微深入研究一下就会发现，这些 4G 模块虽然用了 MiniPCIE 接口，但是实际上的通信接口都是 USB，所以 4G 模块的驱动就转换为了 USB 驱动。而这些 4G 模块厂商都提供了详细的文档讲解如何在 Linux 下使用 4G 模块，以及如何修改 Linux 内核加入 4G 模块驱动。正点原子的 I.MX6U-ALPHA 开发板也有一个 MiniPCIE 形式的 4G 模块接口，虽然外形是 MiniPCIE 的，但是内心却是 USB 的。

I.MX6U-ALPHA 开发板的 4G 模块原理图如图1-1所示 U8 就是 MiniPCIE 接口，MiniPCIE 接口连接到了 GL850 这个 HUB 芯片的 DP2 和 DM2，也就是 GL850 的 USB2 接口上。U11 就是 Nano SIM 接口，I.MX6U-ALPHA 开发板使用 Nano SIM 卡，这样大家就可以直接拿自己的手机卡进行测试。在使用之前需要将 4G 模块插入到MiniPCIE 接口上，然后上紧两边的螺丝，Nano SIM 卡插入到图 71.1.2 中的 Nano SIM 座里面，注意 Nano SIM 卡的方向！ Nano SIM 卡的金属触点朝下！ Nano SIM 卡的金属触点朝下！ Nano SIM 卡的金属触点朝下！

![1-1 4g模块原理图](photo/Linux%204G通信实验/1-1%204g模块原理图.png)

理论上所有的 MiniPCIE 接口的 4G 模块都可以连接到正点原子的 I.MX6U-ALPHA 开发板上，因为这些 4G 模块都遵循同样的接口标准，但是大家在使用的时候还是要详细的看一下 4G 模块的接口引脚描述。不同的 4G 模块其驱动形式也不同，本章我们讲解两款 4G 模块在 I.MX6U-ALPHA 开发板上的使用，一个是上海移远公司的 EC20，另外一个是高新兴物联的 ME3630，这两款 4G 模块都有 MiniPCIE 接口。

4G 模块工作是需要天线的，因此在选购 4G 模块的时候一定要记得购买天线，否则无法进行测试。一般 MiniPCIE 接口的 4G 模块留出来的天线接口为 IPEX 座，因此购买天线的时候也要选择 IPEX 接口的，或者使用 IPEX 转 SMA 线来转接。推荐大家到正点原子官方店铺购买 4G 模块和相应的天线。

## 71.2 高新兴 ME3630 4G 模块实验

### 71.2.1 ME3630 4G 模块简介

ME3630是高新兴物联（原中兴物联）出品的LTE Cat.4七模全网通4G模块。LTE模式上行50Mbps，下行150Mbps，支持回退3G/2G网络，支持分集接收提升通信可靠性。 模块支持PAP、CHAP、PPP协议，具备GNSS、远程唤醒、短信、FoTA空中升级等功能，多用于智能抄表、安防采集、工业路由、车载、监控等M2M场景。

模组硬件特性：
①、一路USB2.0接口
②、一路UART接口
③、SIM卡接口支持1.8/3.0V
④、内置TCP、UDP、FTP、HTTP协议
⑤、支持RAS/ECM/NDIS
⑥、支持AT指令

版本选型：推荐全网通数据版；需要定位选择全网通数据+GNSS版本。

使用前硬件准备：将模块插入MiniPCIE座并锁紧螺丝，装入Nano-SIM卡（金属触点朝下），接上4G天线。

### 71.2.2 ME3630 4G模块驱动修改

#### 1、添加USB设备PID、VID
编辑内核文件 drivers/usb/serial/option.c，找到option_ids数组，添加设备ID，完成后如图1-2。
```c
{ USB_DEVICE(0x19d2, 0x0117) }, /* ME3630*/
{ USB_DEVICE(0x19d2, 0x0199) },
{ USB_DEVICE(0x19d2, 0x1476) },
```
![1-2 添加PID和VID后的options_ids数组](photo/Linux%204G通信实验/1-2%20添加PID和VID后的options_ids数组.png)

#### 2、添加ECM支持，修改option_probe函数
同样在drivers/usb/serial/option.c的option_probe函数中添加如下代码，完成后如图1-3。
```c
/* EM3630 */
if (serial->dev->descriptor.idVendor == 0x19d2 &&
serial->dev->descriptor.idProduct == 0x1476 &&
serial->interface->cur_altsetting->desc.bInterfaceNumber == 3)
return -ENODEV;

if (serial->dev->descriptor.idVendor == 0x19d2 &&
serial->dev->descriptor.idProduct == 0x1476 &&
serial->interface->cur_altsetting->desc.bInterfaceNumber == 4)
return -ENODEV;

if (serial->dev->descriptor.idVendor == 0x19d2 &&
serial->dev->descriptor.idProduct == 0x1509 &&
serial->interface->cur_altsetting->desc.bInterfaceNumber == 4)
return -ENODEV;

if (serial->dev->descriptor.idVendor == 0x19d2 &&
serial->dev->descriptor.idProduct == 0x1509 &&
serial->interface->cur_altsetting->desc.bInterfaceNumber == 5)
return -ENODEV;
```
![1-3 向option_probe添加的内容](photo/Linux%204G通信实验/1-3%20向option_probe添加的内容.png)

#### 3、内核配置
-> Device Drivers
-> -*- Network device support
    -> USB Network Adapters
        -> -*- Multi-purpose USB Networking Framework      配置如图1-4
![1-4 使能USB网络](photo/Linux%204G通信实验/1-4%20使能USB网络.png)

-> Device Drivers
-> [*] USB support
    -> <*> USB Serial Converter support
        -> <*> USB driver for GSM and CDMA modems        配置如图1-5
![1-5 USB GSM和CDMA](photo/Linux%204G通信实验/1-5%20USB%20GSM和CDMA.png)

-> Device Drivers
-> [*] USB support
    -> <*> Support for Host-side USB
        -> <*> USB Modem (CDC ACM) support               配置如图1-6
![1-6 使能USB的CDC ACM功能](photo/Linux%204G通信实验/1-6%20使能USB的CDC%20ACM功能.png)

编译内核生成zImage，使用新内核启动开发板。模块正常识别会生成ttyUSB0 ~ ttyUSB2，如图1-7。
![1-7 ME3630虚拟USB信息](photo/Linux%204G通信实验/1-7%20ME3630虚拟USB信息.png)

执行ifconfig -a，出现usb0网卡，代表ECM驱动加载成功，如图1-8。
![1-8 ME3630对应的usb0网卡](photo/Linux%204G通信实验/1-8%20ME3630对应的usb0网卡.png)

### 71.2.3 ME3630 4G模块 ppp联网测试

#### 1、内核开启PPP功能
-> Device Drivers
-> [*] Network device support
    -> <*> PPP (point-to-point protocol) support
    -> <*> PPP BSD-Compress compression
    -> <*> PPP Deflate compression
    -> [*] PPP filtering
    -> <*> PPP MPPE compression (encryption)
    -> [*] PPP multilink support
    -> <*> PPP over Ethernet
    -> <*> PPP support for async serial ports
    -> <*> PPP support for sync tty ports

配置如图1-9，重新编译内核得到zImage并启动开发板。
![1-9 Linux内核PPP使能](photo/Linux%204G通信实验/1-9%20Linux内核PPP使能.png)

#### 2、移植pppd软件
注意：先删除根文件系统 /usr/sbin/chat，busybox自带chat会冲突。

源码光盘路径：例程源码->7、第三方库源码-> ppp-2.4.7.tar.gz
```bash
tar -zxvf ppp-2.4.7.tar.gz
cd ppp-2.4.7/
./configure
make CC=arm-linux-gnueabihf-gcc
```
编译缺少bison、flex，主机执行：
```bash
sudo apt-get install bison flex
```
拷贝编译产物到根文件系统：
```bash
sudo cp chat/chat /home/zuozhongkai/linux/nfs/rootfs/usr/bin/ -f
sudo cp pppd/pppd /home/zuozhongkai/linux/nfs/rootfs/usr/bin/ -f
sudo cp pppdump/pppdump /home/zuozhongkai/linux/nfs/rootfs/usr/bin/ -f
sudo cp pppstats/pppstats /home/zuozhongkai/linux/nfs/rootfs/usr/bin/ -f
```
执行 pppd -v，正常输出版本号即移植成功。

#### 3、PPP拨号上网测试
开发板创建目录：/etc/gosuncn，目录下新建4个文件。

##### ① ppp-on
```bash
#!/bin/sh
clear
OPTION_FILE="gosuncn_options"
DIALER_SCRIPT=$(pwd)/gosuncn_ppp_dialer
exec pppd file $OPTION_FILE connect "chat -v -f ${DIALER_SCRIPT}"
```

##### ② gosuncn_options
```text
/dev/ttyUSB2
115200
crtscts
modem
persist
lock
noauth
noipdefault
debug
nodetach
user Anyname
password Anypassword
ipcp-accept-local
ipcp-accept-remote
defaultroute
usepeerdns
noccp
nobsdcomp
novj
dump
```
联通/移动卡使用ttyUSB2；电信卡使用ttyUSB0。 电信卡将user、password改为 user card、password card。

##### ③ gosuncn_ppp_dialer
```text
ABORT "NO CARRIER"
ABORT "ERROR"
TIMEOUT 120
"" ATE
SAY "ATE"
ECHO ON
OK ATH
OK ATP
OK AT+CGDCONT=1,\"IP\",\"3GNET\"
OK ATD*99#
CONNECT
```
APN修改：联通3GNET；移动CMNET；电信修改指令为OK "AT+ZCAPN=card,card"，拨号改为ATD#777。

##### ④ disconnect
```bash
#!/bin/sh
killall pppd
```

添加可执行权限
```bash
chmod 777 ppp-on disconnect
```
创建运行所需目录
```bash
mkdir /var/run
mkdir /var/lock
touch /etc/ppp/resolv.conf
```
重要：NFS网络启动环境不能关闭eth0，测试需要把系统烧写到EMMC/NAND本地启动。 进入/etc/gosuncn执行拨号：
```bash
./ppp-on &
```
拨号成功生成ppp0网卡，执行ping百度验证网络。

### 71.2.4 ME3630 4G模块ECM联网测试

ECM模式下识别出usb0网卡。使用minicom打开ttyUSB1，波特率115200发送AT指令。

#### 1、切换到ECM模式
```text
AT+ZSWITCH=L
```
执行后重启开发板，已为ECM模式可跳过。

#### 2、设置APN（根据SIM卡修改）
```text
AT+CGDCONT=1,"IP","CMNET"
```

#### 3、启动ECM拨号
```text
AT+ZECMCALL=1
```

#### 4、网卡操作
```bash
ifconfig usb0 up
udhcpc -i usb0
```
获取IP后测试外网。如果域名解析报错，编辑/etc/resolv.conf添加DNS：
```text
nameserver 114.114.114.114
```
ping百度正常即ECM联网成功。

### 71.2.5 ME3630 4G 模块 GNSS 定位测试

注意！ME3630-C3C 的 GNSS 要用无源天线，不能使用有源天线，否则无法定位！

部分ME3630型号自带GNSS（GPS）定位功能，例如ME3630-C3C，为全网通4G+GNSS+分集版本。 71.2.2小节驱动加载完成后会生成ttyUSB0 ~ ttyUSB2三个串口设备，ttyUSB1为GNSS接口。ME3630的GNSS数据通过USB虚拟串口输出，可使用minicom读取定位原始数据，minicom移植参考63.5小节，配置教程参考63.6.2小节。

GNSS串口参数：ttyUSB1，波特率 115200。

打开minicom连接ttyUSB1，ME3630-C3C默认关闭GNSS输出，需要通过AT指令开启。完整AT指令参考文档：《高新兴物联 ME3630&ME3630-W 模组 AT 指令手册_V3.1.pdf》，光盘路径：开发板光盘->1、例程源码->5、模块驱动源码->3、4G模块->高新兴物联->ME3630。

ME3630-C3C GPS配置AT指令（指令下发，OK为模块返回）：
```text
ATI                     //查看固件信息
Manufacturer: GOSUNCNWELINK
Model: ME3630-W
Revision: ME3630C3CV1.0B03
IMEI: 864863045876287
OK

AT+ZGINIT               //初始化 GPS
OK

AT+ZGMODE=3             //设置定位模式
OK

AT+ZGPORT=0             //定位信息从 AT、MODEM、UART 三个口同时上报
OK

AT+ZGNMEA=31            //设置 GPS 数据为 NMEA 格式
OK

AT+ZGPSR=1              //使能 ZGPSR 数据
OK

AT+ZGRUN=2              //连续定位模式
OK
```
AT指令配置完毕，模块开始搜星。

GNSS天线必须放置室外，室内很难搜星定位。

搜星成功之后模块输出 NMEA格式定位报文，应用开发时直接解析NMEA报文，即可获取经纬度、速度、高度、UTC时间等定位信息。

## 71.3 EC20 4G 模块实验

### 71.3.1 EC20 4G 模块简介

EC20详细资料向模块卖家索取。 EC20存在多种版本：全网通纯数据版、语音版、带GNSS版等，推荐选用全网通数据版，主要用于数据通信。 移远EC20采用LTE 3GPP Rel.11技术，下行最大150Mbps，上行最大50Mbps。

模块特性：
①、一路USB2.0高速接口，最高480Mbps
②、一组模拟语音接口(可选)
③、1.8V/3.0V SIM卡接口
④、1个UART接口
⑤、W_DISABLE#(飞行模式控制)
⑥、LED_WWAN#(网络状态指示)

EC20支持AT指令，本章不讲解AT指令，查阅模块配套文档。

硬件准备：模块插入MiniPCIE座锁紧螺丝，安装Nano-SIM卡，接上天线，完成后开始驱动调试。

### 71.3.2 EC20 4G模块驱动修改

#### 1、添加USB设备信息
模块走USB通信，修改内核文件 drivers/usb/serial/option.c。

首先定义ID宏：
```c
/* EC20 4G */
#define QUECTEL_VENDOR_ID 0X2C7C
#define QUECTEL_PRODUCT_EC20 0X0125
```

option_ids数组添加设备条目：
```c
{ USB_DEVICE(QUECTEL_VENDOR_ID, QUECTEL_PRODUCT_EC20) }, /* EC20 4G */
```

option_probe函数增加如下代码：
```c
/* EC20 */
if (dev_desc->idVendor == cpu_to_le16(0x05c6) &&
dev_desc->idProduct == cpu_to_le16(0x9003) &&
iface_desc->bInterfaceNumber >= 4)
return -ENODEV;

if (dev_desc->idVendor == cpu_to_le16(0x05c6) &&
dev_desc->idProduct == cpu_to_le16(0x9215) &&
iface_desc->bInterfaceNumber >= 4)
return -ENODEV;

if (dev_desc->idVendor == cpu_to_le16(0x2c7c) &&
iface_desc->bInterfaceNumber >= 4)
return -ENODEV;
```

修改option_1port_device结构体，添加休眠唤醒接口。

打开drivers/usb/serial/usb_wwan.c，修改usb_wwan_setup_urb函数，增加零包处理：
```c
static struct urb *usb_wwan_setup_urb(struct usb_serial_port *port,
int endpoint,
int dir, void *ctx, char *buf, int len,
void (*callback) (struct urb *))
{
    struct usb_serial *serial = port->serial;
    struct urb *urb;

    urb = usb_alloc_urb(0, GFP_KERNEL); /* No ISO */
    if (!urb)
        return NULL;

    usb_fill_bulk_urb(urb, serial->dev,
    usb_sndbulkpipe(serial->dev, endpoint) | dir,
    buf, len, callback, ctx);

    /* EC20 */
    if (dir == USB_DIR_OUT) {
        struct usb_device_descriptor *desc = &serial->dev->descriptor;
        if (desc->idVendor == cpu_to_le16(0x05c6) &&
        desc->iProduct == cpu_to_le16(0x9090))
        urb->transfer_flags |= URB_ZERO_PACKET;

        if (desc->idVendor == cpu_to_le16(0x05c6) &&
        desc->iProduct == cpu_to_le16(0x9003))
        urb->transfer_flags |= URB_ZERO_PACKET;

        if (desc->idVendor == cpu_to_le16(0x05c6) &&
        desc->iProduct == cpu_to_le16(0x9215))
        urb->transfer_flags |= URB_ZERO_PACKET;

        if (desc->idVendor == cpu_to_le16(0x2c7c))
        urb->transfer_flags |= URB_ZERO_PACKET;
    }

    return urb;
}
```
18~34行为新增零包处理代码，内核代码修改完成，编译内核检查编译报错。

#### 2、配置Linux内核
内核配置项和71.2.2小节ME3630配置完全一致，参考该小节开启USB NET、GSM/CDMA modem、CDC-ACM。 编译生成zImage，使用新内核启动开发板。模块识别成功生成 /dev/ttyUSB0 ~ ttyUSB3。

**表71.3.2.1 ttyUSB接口定义**

| ttyUSB   | 描述                     |
|----------|--------------------------|
| ttyUSB0  | DM                       |
| ttyUSB1  | GPS NMEA信息输出接口     |
| ttyUSB2  | AT指令接口               |
| ttyUSB3  | PPP连接或AT指令接口      |

常用：ttyUSB1读取GPS数据，ttyUSB2下发AT指令。

#### 3、添加移远GobiNet驱动
光盘源码路径：1、例程源码->5、模块驱动源码->3、4G模块->移远 EC20\EC20_R2.1_Mini_PCIe-C->05 Driver->Linux->GobiNet-> Quectel_WCDMA<E_Linux&Android_GobiNet_Driver_V1.3.0.zip

把压缩包内src全部.c、.h复制到内核目录 drivers/net/usb。

修改drivers/net/usb/Makefile末尾添加：
```makefile
obj-$(CONFIG_USB_GOBI_NET) += GobiNet.o
GobiNet-objs := GobiUSBNet.o QMIDevice.o QMI.o
```

修改drivers/net/usb/Kconfig添加配置项：
```kconfig
config USB_GOBI_NET
    tristate"Gobi USB Net driver for Quectel module"
    help
      Support Quectel module.

      A modemmanager with support for GobiNet is recommended.
      To compile this driver as a module, choose M here: the module will be called GobiNet.
```

内核配置开启：
-> Device Drivers
-> [*] Network device support
-> -*- USB Network Adapters
-> <*> Gobi USB Net driver for Quectel module

重新编译内核，启动开发板。存在/dev/qcqmi2代表GobiNet驱动加载成功。EC20内核驱动全部完成。

### 71.3.3 quectel-CM移植

quectel-CM为移远官方网络管理工具。 源码光盘路径：1、例程源码->5、模块驱动源码->3、4G模块->移远 EC20->EC20_R2.1_Mini_PCIe-C->05 Driver->Linux->GobiNet->WCDMA&LTE_QConnectManager_Linux&Android_V1.1.34.zip

解压得到quectel-CM文件夹，拷贝至Ubuntu，交叉编译：
```bash
make CROSS_COMPILE=arm-linux-gnueabihf-
```

将编译产物quectel-CM复制到开发板根文件系统/usr/bin目录。
```bash
sudo cp quectel-CM /home/zuozhongkai/linux/nfs/rootfs/usr/bin/ -f
```

### 71.3.4 EC20上网测试

开发板执行拨号命令：
```bash
quectel-CM -s cenet &
```
quectel-CM依赖udhcpc获取IP，根文件系统必须存在udhcpc工具。

联网成功，网卡名称为eth2，查看网卡信息：
```bash
ifconfig eth2
```

注意：NFS网络启动环境无法关闭eth0，网络测试需要将u-boot、zImage、dtb、根文件系统全部烧写至EMMC/NAND本地启动。从本地启动后执行拨号，保证系统仅保留eth2网卡，ping百度外网，ping通代表EC20工作正常。