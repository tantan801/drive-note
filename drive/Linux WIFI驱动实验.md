# 第七十章 Linux WIFI 驱动实验

正点原子 I.MX6U-ALPHA 开发板支持USB、SDIO两种接口 WIFI。
USB-WIFI：RTL8188EUS、RTL8188CUS
SDIO-WIFI：RTL8189FS（RTL8189FTV）
驱动由 Realtek 提供源码，不自己写驱动，把源码加入内核，配置编译为模块。

---

## 70.1 WIFI 驱动添加与编译

### 70.1.1 向 Linux 内核添加 WIFI 驱动

#### 1、rtl81xx 驱动文件浏览
光盘驱动源码路径：例程源码→5、模块驱动源码→1、RTL8XXX WIFI驱动源码→realtek
realtek 目录内部：
rtl8188EUS：RTL8188EUS USB-WIFI 驱动
rtl8189FS：RTL8189FS/FTV SDIO-WIFI 驱动
rtl8192CU：RTL8188CUS / RTL8192CU USB-WIFI 驱动
注意：RTL8188EUS 与 RTL8188CUS 驱动不通用。

realtek 目录下Kconfig，提供内核 menuconfig 配置入口：
```kconfig
menuconfig REALTEK_WIFI
    tristate "Realtek wifi"

if REALTEK_WIFI

choice
    prompt "select wifi type"
    default RTL8189FS

config RTL8189FS
    depends on REALTEK_WIFI
    tristate "rtl8189fs/ftv sdio wifi"

config RTL8188EUS
    depends on REALTEK_WIFI
    tristate "rtl8188eus usb wifi"

config RTL8192CU
    depends on REALTEK_WIFI
    tristate "Realtek 8192C USB WiFi"

endchoice
endif
```

realtek 目录下Makefile：
```makefile
obj-$(CONFIG_RTL8188EUS) += rtl8188EUS/
obj-$(CONFIG_RTL8189FS) += rtl8189FS/
obj-$(CONFIG_RTL8192CU) += rtl8192CU/
```

#### 2、屏蔽内核自带 RTL8192CU 驱动
内核自带 rtl8192cu 驱动稳定性差，使用光盘提供的源码，需要屏蔽内核自带驱动，避免编译冲突。
①修改 drivers/net/wireless/rtlwifi/Kconfig
删除以下全部内容：
```kconfig
config RTL8192CU
    tristate "Realtek RTL8192CU/RTL8188CU USB Wireless Network Adapter"
    depends on USB
    select RTLWIFI
    select RTLWIFI_USB
    select RTL8192C_COMMON
    ---help---
    This is the driver for Realtek RTL8192CU/RTL8188CU 802.11n USB
    wireless network adapters.

    If you choose to build it as a module, it will be called rtl8192cu
```
②修改 drivers/net/wireless/rtlwifi/Makefile
屏蔽这一行：
```makefile
# obj-$(CONFIG_RTL8192CU) += rtl8192cu/
```

#### 3、拷贝 realtek 驱动目录到内核
将完整realtek文件夹复制到内核源码路径： drivers/net/wireless/

#### 4、修改 drivers/net/wireless/Kconfig
在文件末尾 WLAN 菜单范围内添加一行，引入 realtek 的 Kconfig，menuconfig 才会显示 Realtek wifi 配置项：
```kconfig
source "drivers/net/wireless/realtek/Kconfig"
```
示例片段：
```kconfig
......
source "drivers/net/wireless/rsi/Kconfig"
source "drivers/net/wireless/realtek/Kconfig"

endif # WLAN
```

#### 5、修改 drivers/net/wireless/Makefile
添加，使编译系统扫描 realtek 子目录：
```makefile
obj-y += realtek/
```
示例片段：
```makefile
......
obj-$(CONFIG_CW1200) += cw1200/
obj-$(CONFIG_RSI_91X) += rsi/

obj-y += realtek/
```

到此内核源码层面修改完成，接下来就可以执行make menuconfig去选择对应的 WIFI 芯片，编译生成 ko 驱动模块。

---

### 70.1.2 配置 Linux 内核
编译 RTL8188、RTL8189 驱动前，完成内核基础配置。

#### 1、配置 USB 支持设备
```text
-> Device Drivers
-> <*> USB support
    -> <*> Support for Host-side USB
    -> <*> EHCI HCD (USB 2.0) support
    -> <*> OHCI HCD (USB 1.1) support
    -> <*> ChipIdea Highspeed Dual Role Controller
        -> [*] ChipIdea device controller
        -> [*] ChipIdea host controller
```

#### 2、配置支持 WIFI 设备
```text
-> Device Drivers
-> [*] Network device support
-> [*] Wireless LAN
    -> <*> IEEE 802.11 for Host AP (Prism2/2.5/3 and WEP/TKIP/CCMP)
    -> [*] Support downloading firmware images with Host AP driver
    -> [*] Support for non-volatile firmware download
```
配置完如图 1-1。
![图1-1 WIFI设备内核配置](photo/Linux%20WIFI驱动实验/1-1.png)

#### 3、配置支持 IEEE 802.11
```text
-> Networking support
-> -*- Wireless
    -> [*] cfg80211 wireless extensions compatibility
    -> <*> Generic IEEE 802.11 Networking Stack (mac80211)
```
配置完如图 1-2。
![图1-2 IEEE 802.11内核配置](photo/Linux%20WIFI驱动实验/1-2.png)

配置完成后编译内核，生成新 zImage。

---

### 70.1.3 编译 WIFI 驱动
执行make menuconfig，将 rtl81xx 系列驱动编译成模块：
```text
-> Device Drivers
-> Network device support (NETDEVICES [=y])
-> Wireless LAN (WLAN [=y])
    -> Realtek wifi (REALTEK_WIFI [=m])
        -> rtl8189ftv sdio wifi
        -> rtl8188eus usb wifi
        -> Realtek 8192C USB WiFi
```
配置结果如图 1-3。
![图1-3 WIFI驱动模块配置结果](photo/Linux%20WIFI驱动实验/1-3.png)

编译模块命令：
```bash
make modules -j12
```
编译完成，在对应源码目录生成驱动模块：
```text
rtl8188EUS/8188eu.ko
rtl8189FS/8189fs.ko
rtl8192CU/8192cu.ko
```
如图 1-4。
![图1-4 编译生成的WIFI驱动模块](photo/Linux%20WIFI驱动实验/1-4.png)

拷贝 ko 模块到开发板根文件系统模块目录：
```bash
sudo cp 8189fs.ko /home/zuozhongkai/linux/nfs/rootfs/lib/modules/4.1.15/ -rf
sudo cp 8188eu.ko /home/zuozhongkai/linux/nfs/rootfs/lib/modules/4.1.15/ -rf
sudo cp 8192cu.ko /home/zuozhongkai/linux/nfs/rootfs/lib/modules/4.1.15/ -rf
```
拷贝新 zImage 到 tftpboot：
```bash
cp arch/arm/boot/zImage /home/zuozhongkai/linux/tftpboot/ -f
```
重启开发板。

---

### 70.1.4 驱动加载测试

#### 1、RTL8188 USB WIFI 驱动测试
把 RTL8188 WIFI 模块接开发板 USB HOST 接口，进入lib/modules/4.1.15目录。
```bash
depmod                     # 首次加载驱动必须执行
modprobe 8188eu.ko        # RTL8188EUS加载
modprobe 8192cu.ko        # RTL8188CUS加载
```
驱动加载成功如图 1-5。
![图1-5 RTL8188驱动加载成功](photo/Linux%20WIFI驱动实验/1-5.png)

查看是否生成 WIFI 网卡：
```bash
ifconfig -a
```
正常出现wlan0网卡，如图 1-6。多个 WIFI 模块同时使用时网卡名会变成 wlan1、wlan2。
![图1-6 RTL8188生成wlan0网卡](photo/Linux%20WIFI驱动实验/1-6.png)

#### 2、RTL8189 SDIO WIFI 驱动测试
I.MX6U-ALPHA 开发板 SDIO-WIFI 与 SD 卡共用一组 SDIO 接口，SD 卡和 SDIO-WIFI 二者互斥，同一时间只能使用一个，测试 RTL8189 需要拔掉 SD 卡。SDIO WIFI 接口原理图如图 1-7。
![图1-7 SDIO WIFI接口原理图](photo/Linux%20WIFI驱动实验/1-7.png)

操作步骤：
1. 拔出 SD 卡，插上 RTL8189 SDIO-WIFI 模块。
2. 加载驱动模块
```bash
depmod
modprobe 8189fs.ko
```
驱动加载成功如图 1-8。 
![图1-8 RTL8189驱动加载成功](photo/Linux%20WIFI驱动实验/1-8.png)

3. 查看网卡设备
```bash
ifconfig -a
```
出现wlanX网卡，代表 RTL8189 驱动工作正常。

RTL8188 USB-WIFI、RTL8189 SDIO-WIFI 驱动 ok 之后，还需要第三方工具组件才能连接路由器上网。

---

## 70.2 wireless tools 工具移植与测试

### 70.2.1 wireless tools 移植
wireless tools 为 WIFI 操作工具集合：
iwconfig：设置无线网络参数
iwlist：扫描周边 WIFI 热点
iwspy：获取节点连接质量
iwpriv：操作驱动私有配置
ifrename：网卡重命名

源码包光盘路径：例程源码-》7、第三方库源码-》iwlist_for_visteon-master.tar.bz2
```bash
#拷贝到Ubuntu tool目录，解压
tar -xjvf iwlist_for_visteon-master.tar.bz2
cd iwlist_for_visteon-master
```
修改 Makefile，将CC、AR、RANLIB改为arm-linux-gnueabihf-交叉编译工具，修改后如图 2-1。
![图2-1 修改Wireless Tools Makefile](photo/Linux%20WIFI驱动实验/2-1.png)

编译：
```bash
make clean
make
```
编译产物：iwlist iwconfig iwspy iwpriv ifrename、libiw.so.29
拷贝到根文件系统：
```bash
sudo cp iwlist iwconfig iwspy iwpriv ifrename /home/zuozhongkai/linux/nfs/rootfs/usr/bin/ -f
sudo cp libiw.so.29 /home/zuozhongkai/linux/nfs/rootfs/usr/lib/ -f
```

### 70.2.2 wireless tools 工具测试
输入iwlist，输出对应信息代表工具可用，如图 2-2。
![图2-2 Wireless Tools工具验证](photo/Linux%20WIFI驱动实验/2-2.png)

以 RTL8188 USB-WIFI 举例测试扫描热点：
```bash
modprobe 8188eu.ko
ifconfig wlan0 up
iwlist wlan0 scan
```
命令输出周边全部 WIFI 热点，包含 MAC、ESSID、频率、速率、信号质量，如图 2-3。
![图2-3 扫描周边WIFI热点结果](photo/Linux%20WIFI驱动实验/2-3.png)

扫描到目标热点（例：ZZK）如图 2-4，连接 WIFI 需要wpa_supplicant工具。
![图2-4 扫描到目标热点ZZK](photo/Linux%20WIFI驱动实验/2-4.png)

---

## 70.3 wpa_supplicant 移植
wpa_supplicant 依赖openssl、libnl，顺序：先 openssl → libnl → wpa_supplicant

### 70.3.1 openssl 移植
源码光盘路径：例程源码-》7、第三方库源码-》openssl-1.1.1d.tar.gz
```bash
tar -xzvf openssl-1.1.1d.tar.gz
cd openssl-1.1.1d
mkdir ../openssl
./Configure linux-armv4 shared no-asm --prefix=/home/zuozhongkai/linux/IMX6ULL/tool/openssl CROSS_COMPILE=arm-linux-gnueabihf-
make
make install
```
编译输出目录如图 2-5，拷贝库文件到开发板根文件系统：
![图2-5 openssl编译输出目录](photo/Linux%20WIFI驱动实验/2-5.png)
```bash
sudo cp libcrypto.so* /home/zuozhongkai/linux/nfs/rootfs/lib/ -af
sudo cp shturl.cc* /home/zuozhongkai/linux/nfs/rootfs/lib/ -af
```

### 70.3.2 libnl 库移植
先安装主机依赖工具
```bash
sudo apt-get install bison flex
```
源码光盘路径：例程源码-》7、第三方库源码-》libnl-3.2.23.tar.gz
```bash
tar -xzvf libnl-3.2.23.tar.gz
cd libnl-3.2.23
mkdir ../libnl
./configure --host=arm-linux-gnueabihf --prefix=/home/zuozhongkai/linux/IMX6ULL/tool/libnl/
make -j12
make install
```
编译产物目录如图 2-6，拷贝库：
![图2-6 libnl编译产物目录](photo/Linux%20WIFI驱动实验/2-6.png)
```bash
sudo cp lib/* /home/zuozhongkai/linux/nfs/rootfs/usr/lib/ -rf
```

### 70.3.3 wpa_supplicant 移植
源码光盘路径：例程源码>7、第三方库源码>wpa_supplicant-2.7.tar.gz
```bash
tar -xzvf wpa_supplicant-2.7.tar.gz
cd wpa_supplicant-2.7/wpa_supplicant
cp defconfig .config
```
编辑.config，添加：
```makefile
CC = arm-linux-gnueabihf-gcc

#openssl 库和头文件路径
CFLAGS += -I/home/zuozhongkai/linux/IMX6ULL/tool/openssl/include
LIBS += -L/home/zuozhongkai/linux/IMX6ULL/tool/openssl/lib -lssl -lcrypto

#libnl 库和头文件路径
CFLAGS += -I/home/zuozhongkai/linux/IMX6ULL/tool/libnl/include/libnl3
LIBS += -L/home/zuozhongkai/linux/IMX6ULL/tool/libnl/lib
```
编译，必须设置PKG_CONFIG_PATH，否则报 libnl 找不到：
```bash
export PKG_CONFIG_PATH=/home/zuozhongkai/linux/IMX6ULL/tool/libnl/lib/pkgconfig:$PKG_CONFIG_PATH
make -j12
```
编译生成wpa_supplicant、wpa_cli，拷贝至根文件系统：
```bash
sudo cp wpa_cli wpa_supplicant /home/zuozhongkai/linux/nfs/rootfs/usr/bin/ -f
```
重启开发板，验证版本：
```bash
wpa_supplicant -v
```

---

## 70.4 WIFI 联网测试
USB-WIFI、SDIO-WIFI 通用联网操作步骤：
1、插上 WIFI 模块；SDIO-WIFI 务必拔掉 SD 卡，避免接口冲突。
2、加载对应 RTL8188 / RTL8189 驱动模块。
3、ifconfig打开无线网卡（wlan0/wlan1）。
4、iwlist扫描 WIFI 热点，确认目标热点可以被扫描到。

确认上面步骤正常后，使用wpa_supplicant连接 WIFI 热点。

### 70.4.1 RTL8188 USB WIFI 联网测试
注意：RTL8188EUS 用8188eu.ko；RTL8188CUS 用8192cu.ko。

确认 iwlist 可以扫描到目标 WIFI 热点，在开发板/etc新建wpa_supplicant.conf配置文件。
示例代码 70.4.1.1 wpa_supplicant.conf
```conf
ctrl_interface=/var/run/wpa_supplicant
ap_scan=1
network={
 ssid="ZZK"
 psk="xxxxxxxx"
}
```
ssid：WIFI 热点名称
psk：WIFI 密码
重要格式要求：=前后不能有空格，缩进只用空格，禁止 TAB；建议手动输入，不要直接复制粘贴，容易出现解析错误。

创建运行时目录：
```bash
mkdir /var/run/wpa_supplicant -p
```
启动 wpa_supplicant 连接热点：
```bash
wpa_supplicant -D wext -c /etc/wpa_supplicant.conf -i wlan0 &
```
连接成功打印：wlan0: CTRL-EVENT-CONNECTED。

DHCP 动态获取 IP：
```bash
udhcpc -i wlan0
```
获取 IP 成功输出如图 3-1。
![图3-1 WIFI获取IP成功](photo/Linux%20WIFI驱动实验/3-1.png)

查看网卡信息
```bash
ifconfig wlan0
```

连通性测试： 
1、电脑 ping 开发板 wlan0 的 IP； 
2、开发板用 WIFI 网卡 ping 外网：
```bash
ping -I 192.168.1.126 www.baidu.com
```
能 ping 通则 RTL8188 USB-WIFI 工作正常。

### 70.4.2 RTL8189 SDIO WIFI 联网测试
1、拔掉 SD 卡，插入 RTL8189 SDIO-WIFI 模块。 
2、加载驱动，打开wlan0网卡，iwlist确认可以扫描到目标热点。 
3、同样准备/etc/wpa_supplicant.conf配置文件，参考 70.4.1 小节。

RTL8189 驱动使用 nl80211 驱动接口，参数-D nl80211，不要填错。
```bash
wpa_supplicant -Dnl80211 -c /etc/wpa_supplicant.conf -i wlan0 &
```
DHCP 获取 IP：
```bash
udhcpc -i wlan0
```
查看网卡：
```bash
ifconfig wlan0
```
连通性测试：
```bash
ping -I 192.168.1.118 www.baidu.com
```