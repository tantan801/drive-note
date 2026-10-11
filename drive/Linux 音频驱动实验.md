# 第六十五章 Linux 音频驱动实验

音频功能是嵌入式系统中最常用的应用之一，尤其在 Linux 和 Android 平台上，占据着重要的位置。I.MX6ULL 处理器集成了 SAI 接口，用于连接外部音频编解码器。在正点原子发布的 I.MX6ULL ALPHA 开发板（V2.4 及更早版本）中，采用了 WM8960 音频 DAC 芯片作为外接音频方案；而在后续的 V2.8 版本中，开发板改用 ES8388 音频 DAC 芯片，以适配新的应用需求。本章将详细介绍如何配置并启用 WM8960 或 ES8388 的音频驱动，确保系统能够通过对应的芯片实现音频播放和录音功能。

---

## 65.1 音频接口简介

### 65.1.1 为何需要音频编解码芯片？

处理器仅能识别二进制0、1数字信号，而自然界的声音属于模拟信号。因此设备想要实现收音功能，就需要通过ADC芯片将外界的模拟声音信号转换为处理器可识别的数字信号。反之，设备实现播音放音功能时，需要通过DAC芯片将处理器输出的数字信号转换为连续的模拟声音信号，对外播放声音。

理论上可以单独使用独立的ADC和DAC芯片实现基础的录音、播音功能，但实际音频应用不仅要求发声和收音，还对音质还原度、音效调节、声音处理、标准化开发接口等有更高要求。音频编解码芯片（Audio CODEC）整合了ADC、DAC核心功能，同时集成音效处理、增益调节等配套音频功能，是专门适配音频场景的专用芯片，日常设备标注的CODEC均指代音频编解码芯片。

采样率和采样位数是评判音频编解码芯片性能的核心指标，常见采样率包含8K、44.1K、48K、192K、384K、768K，常见采样位数有8位、16位、24位、32位。两项参数数值越高，芯片对声音的还原度越真实，能够实现HIFI级音质，也是高端音频设备性能更强、价格更高的核心原因。不过音频整体效果并非仅由这两项参数决定，还会受到设备其他硬件因素的影响。

### 65.1.2 音频编解码芯片简介

前面我们已经分析了为何需要音频编解码芯片，专用音频 CODEC 自带大量音频相关特性。下面以正点原子 ALPHA 开发板的 WM8960、ES8388 两颗芯片为例，介绍音频编解码芯片的主要特性。

#### （1）WM8960 音频芯片

WM8960 是欧胜 Wolfson 推出的低功耗高品质立体声音频 CODEC，内置 D 类喇叭功放，单通道可驱动 8Ω、1W 喇叭；具备 3 路立体声模拟输入，集成麦克风接口，ADC、DAC 均为 24bit。 核心指标： ① DAC 信噪比 98dB，3.3V/48KHz 条件下谐波失真 - 84dB； ② ADC 信噪比 94dB，3.3V/48KHz 条件下谐波失真 - 82dB； ③ 支持 3D 音效增强； ④ 内置立体声 D 类功放，直推 8Ω/1W 喇叭； ⑤ 自带耳机、麦克风接口； ⑥ 采样率覆盖 8K~48K。

WM8960框图如图1-1所示：

![](photo/Linux%20音频驱动实验/1-1%20WM8960框图.jpeg)

框图分为 4 个功能部分： ① 模拟输入接口：共 3 路立体声输入 LINPUT/RINPUT，接麦克风或者线路输入。PCB 阶段就要确定选用哪一路，硬件重点关注。 ② 模拟输出接口：喇叭输出 SPK_LP/LN、SPK_RP/RN，可外接 1W 喇叭；HP_L/R 为耳机输出。 ③ 数字音频 I2S 接口（驱动开发重点），共 5 根信号线，负责在主控和 CODEC 之间传输音频数据流： ADCDAT：ADC 数字音频输出，传给主控； ADCLRC：ADC 帧时钟 LRCK，标记左右声道，频率等于采样率，可复用为 GPIO； DACDAT：DAC 数据输入，主控下发音频数据给 CODEC； DACLRC：DAC 帧时钟 LRCK，功能同 ADCLRC； BCLK：位时钟，用于比特同步； MCLK：主时钟，由 I.MX6ULL 提供，频率为采样率的 256 倍或 384 倍。 ④ I2C 控制接口：用来读写寄存器，完成对 WM8960 的各项配置。

#### （2）ES8388 音频芯片

ES8388 是一款高性能、低功耗、低成本音频 CODEC，采用多位 Δ-Σ 调制技术，低抖动、低底噪，集成双通道 ADC/DAC、麦放、耳放、混音与音效处理。 核心特性： ① ADC：24bit，采样率 8k~96kHz，信噪比 95dB；支持单 / 立体声麦输入，内置 PGA，带 AGC 自动增益、噪声门、模拟混音。 ② DAC：24bit，采样率 8k~96kHz，信噪比 96dB；40mW 耳机驱动，支持免耦合电容输出，支持音效、混音与增益调节。 ③ 功耗：供电 1.8V~3.3V，播放 7mW，录放同时工作 16mW。 ④ 系统接口：I2C/SPI 做寄存器配置；I2S、DSP/PCM 音频数据接口；支持主从模式，兼容多种时钟、采样率。

ES8388 整体框图如图 1-2 所示：

![](photo/Linux%20音频驱动实验/1-2%20ES8388框图.jpeg)

① 输入通路：LINPUT/RINPUT 两路模拟输入，可单端 / 差分输入，经过 PGA 可编程增益放大后送入 ADC；支持模拟混音，可启用 ALC 自动电平控制，增益支持手动或自动调节。 ② 输出通路：数字滤波器 + DAC + 模拟混音 + 输出驱动。支持播放、录放、旁路模式；DAC 输出可以和模拟输入混音，输出两组立体声通道 LOUT1/2；可驱动 16Ω/32Ω 耳机，也可作为线路输出，各通路可独立开关。

### 65.1.3 I2S 总线接口

I2S（Inter-IC Sound，也写作 IIS）是飞利浦提出的数字音频传输总线，用于主控与音频 CODEC 之间传输音频数据流。使用 I2S 要求主控和 CODEC 都支持该协议，I.MX6ULL 的 SAI 外设、WM8960 均支持 I2S，本章实验就采用 I2S 完成音频数据传输。

I2S 基础需要 3 根信号线；如果同时收发（录音 + 播放），则需要 4 根信号线，收发各自使用独立数据线：

1. **SCK（BCLK，位时钟）**：串行时钟，每 1bit 音频数据对应 1 个 SCK 脉冲。立体声双声道计算公式：SCK = 2 × 采样率 × 采样位数。 举例：44.1KHz、16 位立体声音频，SCK=2×44100×16=1411200Hz（1.4112MHz）。
2. **WS（LRCK / 帧时钟）**：声道选择信号，用来切换左右声道；WS=1 传输左声道，WS=0 传输右声道。WS 频率等于音频采样率，如 44.1KHz 音频，WS 就是 44.1KHz。
3. **SD（串行数据线）**：传输实际音频数据；录放同时工作时需要两根独立数据线，如 WM8960 的 ADCDAT（录音）、DACDAT（播放）。I2S 传输时高位优先发送，最高位在 LRCK 跳变后的第 2 个 SCK 脉冲处输出。

额外可选 MCLK（主时钟 / 系统时钟），用于主控和 CODEC 之间时钟同步，频率一般为采样率的 256 倍或 384 倍。

图1-3为一帧立体声音音频时序图

**图1-3 I2S时序图**

![](photo/Linux%20音频驱动实验/1-3%20I2S时序图.jpeg)

抓取的真实 I2S 时序见图 1-4：

**图 1-4 真实的 I2S 时序图**

![](photo/Linux%20音频驱动实验/1-4%20真实的I2S时序图.jpeg)

通道 0：LRCK 时钟，通道 1：BCLK，通道 2：DACDATA，通道 3：MCLK。

在 I2S 基础规范之上衍生出多种数据格式，依据 DATA 相对 LRCK、SCLK 的相对位置，分为左对齐 (Left Justified)、右对齐 (Right Justified)，两种格式时序见图 1-5：

**图1-5  I2S 左对齐和右对齐数据格式。**

![](photo/Linux%20音频驱动实验/1-5%20I2S左对齐和右对齐数据格式.jpeg)

### 65.1.4 I.MX6ULL SAI 简介

音频 CODEC 需要 I2S 协议，主控端同样要有对应的 I2S 接口外设。STM32 使用 SAI 外接音频 CODEC，I.MX6ULL 同样集成SAI（Synchronous Audio Interface，同步音频接口）。

I.MX6ULL 的 SAI 是全双工带帧同步的串行外设，支持 I2S、AC97、TDM、音频 DSP，主要特性： ①、单帧最大支持 32 个字； ②、字位宽可配置 8bit~32bit； ③、收发通道各自具备 32×32bit 的 FIFO 缓冲区； ④、FIFO 出错支持平滑重启恢复。

I.MX6ULL 的 SAI 框图如图 1 6 所示：

**图 1 6 SAI 框图**

![](photo/Linux%20音频驱动实验/1-6%20SAI框图.jpeg)

框图右侧以SAI_TX、SAI_RX开头即为 SAI 对外连接音频 CODEC 的信号引脚，硬件连接见 65.2 原理图小节。

---

## 65.2 硬件原理图分析

底板音频原理图如图 2 1 所示。

**图 2-1 底板音频原理图**

![](photo/Linux%20音频驱动实验/2-1%20底板音频原理图.jpeg)

重点关注两组接口：SAI（音频数据流）、I2C（CODEC 寄存器配置）。 ① SAI 接口：I.MX6ULL 对接 WM8960 时一共 6 根信号线，完成主控与 CODEC 音频收发； ② I2C 配置接口：WM8960、ES8388 都依靠 I2C 配置内部寄存器，ES8388 使用 I2C1；ES8388 方案下 SAI 使用 5 根信号线传输音频数据。

---

## 65.3 音频驱动使能

适用范围：正点原子 I.MX6ULL 出厂内核源码，设备树imx6ull 14x14 evk.dts，适配 WM8960 / ES8388 不同底板版本，通过内核配置 + 设备树完成音频驱动适配。

注：新版出厂系统一般已经内置兼容两套 CODEC，本节为手动适配核对流程。

### 65.3.1 修改根设备树（sound 节点）

根节点下添加音频固定电源 regulator、sound es8388 machine 音频节点

```dts
/ {
    ......
    regulators {
        ......
        /* 音频固定电源，提供3.3V给ES8388模拟、数字供电 */
        reg_audio_pwr: regulator-audio-pwr {
            compatible = "regulator-fixed";
            regulator-name = "audio-pwr";
            regulator-min-microvolt = <3300000>;  /* 最小输出3.3V */
            regulator-max-microvolt = <3300000>;  /* 最大输出3.3V */
            enable-active-high;                   /* 高电平使能电源 */
        };
    };
    ......
    /* ASoC machine层节点，把SAI、CODEC、ASRC、音频通路路由绑定在一起 */
    sound-es8388 {
        compatible = "fsl,imx-audio-es8388";   /* 匹配machine驱动 */
        model = "es8388-audio";                /* 声卡名字，aplay -l可以看到 */
        audio-cpu = <&sai2>;                   /* cpu dai：使用SAI2外设 */
        audio-codec = <&codec>;                /* codec dai：指向i2c下es8388节点 */
        audio-asrc = <&asrc>;                  /* 引用ASRC异步采样率转换器 */
        gpr = <&gpr 4 0x100000 0x100000>;      /* I.MX6ULL内部GPR寄存器配置 */
        /* audio routing：硬件音频通路映射，内核ALSA控制通路 */
        audio-routing =
            "Headphone Jack", "LOUT1",
            "Headphone Jack", "ROUT1",
            "Ext Spk", "LOUT2",
            "Ext Spk", "ROUT2",
            "LINPUT1", "Line In Jack",
            "RINPUT1", "Line In Jack",
            "LINPUT2", "Mic Jack",
            "RINPUT2", "Mic Jack",
            "Mic Jack", "Mic Bias",
            "CPU Playback", "ASRC Playback",
            "Playback", "CPU Playback",
            "ASRC Capture", "CPU Capture",
            "CPU Capture", "Capture";
    };
    ......
};
```

### 65.3.2 修改 I2C 设备树（codec 节点）

CODEC 芯片挂载在 I2C1 总线下，同时放置 ES8388、WM8960 两个子节点，对应不同底板硬件。

```dts
&i2c1 {
    ......
    /* ES8388 codec节点，I2C从设备地址0x11 */
    codec: es8388@11 {
        compatible = "alientek,es8388";
        reg = <0x11>;                          /* I2C设备从地址 */
        clock-names = "mclk";
        clocks = <&clks IMX6UL_CLK_SAI2>;      /* MCLK主时钟源来自SAI2时钟 */
        hp-det-gpio = <&gpio5 4 GPIO_ACTIVE_HIGH>; /* 耳机检测GPIO5_IO4 */
        /* 引用前面定义的音频3.3V电源域 */
        AVDD-supply = <&reg_audio_pwr>;
        DVDD-supply = <&reg_audio_pwr>;
        HPVDD-supply = <&reg_audio_pwr>;
        PVDD-supply = <&reg_audio_pwr>;
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_es8388_hp_det_b>; /* 耳机检测引脚pinctrl配置 */
    };

    /* WM8960 codec节点，I2C从设备地址0x1a */
    codec1: wm8960@1a {
        compatible = "wlf,wm8960";
        reg = <0x1a>;                          /* WM8960 I2C从地址 */
        clocks = <&clks IMX6UL_CLK_SAI2>;      /* MCLK使用SAI2时钟 */
        clock-names = "mclk";
        wlf,shared-lrclk;                      /* 共享LRCLK帧时钟配置 */
    };
    ......
};
```

### 65.3.3 修改 SAI 设备树

正点原子所有底板，WM8960/ES8388 都使用 SAI2 作为 PCM 音频数据通道，配置 SAI2 引脚、父时钟、输出 MCLK。

```dts
&sai2 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_sai2>;            /* 引用SAI2引脚复用组 */
    /* 分配SAI2的时钟选择器与SAI2模块时钟 */
    assigned-clocks = <&clks IMX6UL_CLK_SAI2_SEL>, <&clks IMX6UL_CLK_SAI2>;
    assigned-clock-parents = <&clks IMX6UL_CLK_PLL4_AUDIO_DIV>; /* 时钟父源选择音频PLL4 */
    assigned-clock-rates = <0>, <11289600>; /* SAI2输出MCLK=11.2896MHz(256*44.1K) */
    status = "okay";                         /* 使能SAI2外设 */
};
```

### 65.3.4 修改 IOMUX 设备树

**1、SAI2 引脚复用 iomuxc**

```dts
&iomuxc {
    ……
    imx6ul-evk {
        ……
        /* SAI2外设引脚复用配置组 */
        pinctrl_sai2: sai2grp {
            fsl,pins = <
                /* JTAG_TDI复用为SAI2_TX_BCLK 位时钟BCLK */
                MX6UL_PAD_JTAG_TDI__SAI2_TX_BCLK 0x17088
                /* JTAG_TDO复用为SAI2_TX_SYNC 帧同步LRCK */
                MX6UL_PAD_JTAG_TDO__SAI2_TX_SYNC 0x17088
                /* JTAG_TRST_B复用SAI2_TX_DATA，发送数据DAC */
                MX6UL_PAD_JTAG_TRST_B__SAI2_TX_DATA 0x11088
                /* JTAG_TCK复用SAI2_RX_DATA，接收数据ADC */
                MX6UL_PAD_JTAG_TCK__SAI2_RX_DATA 0x11088
                /* JTAG_TMS复用SAI2_MCLK，输出主时钟MCLK给CODEC */
                MX6UL_PAD_JTAG_TMS__SAI2_MCLK 0x17088
            >;
        };
        ……
    };
    ……
};
```

**2、Audio节点：ES8388 耳机检测引脚 iomuxc_snvs**

WM8960 耳机检测使用内部寄存器，不需要额外 pinctrl；ES8388 (V2.8 底板) 使用 SNVS 域 GPIO5_IO4 做耳机插入检测。

```dts
&iomuxc {
    ……
    imx6ul-evk {
        ……
        /* SAI2外设引脚复用配置组 */
        pinctrl_sai2: sai2grp {
            fsl,pins = <
                /* JTAG_TDI复用为SAI2_TX_BCLK 位时钟BCLK */
                MX6UL_PAD_JTAG_TDI__SAI2_TX_BCLK 0x17088
                /* JTAG_TDO复用为SAI2_TX_SYNC 帧同步LRCK */
                MX6UL_PAD_JTAG_TDO__SAI2_TX_SYNC 0x17088
                /* JTAG_TRST_B复用SAI2_TX_DATA，发送数据DAC */
                MX6UL_PAD_JTAG_TRST_B__SAI2_TX_DATA 0x11088
                /* JTAG_TCK复用SAI2_RX_DATA，接收数据ADC */
                MX6UL_PAD_JTAG_TCK__SAI2_RX_DATA 0x11088
                /* JTAG_TMS复用SAI2_MCLK，输出主时钟MCLK给CODEC */
                MX6UL_PAD_JTAG_TMS__SAI2_MCLK 0x17088
            >;
        };
        ……
    };
    ……
};
```

### 65.3.5 使能内核 WM8960 驱动

设备树配置完成后，开启内核自带 WM8960 驱动，使用图形化内核配置，执行命令：

```bash
make menuconfig
```

**1、取消 ALSA 模拟 OSS API** 关闭 OSS 模拟层，配置路径：
-> Device Drivers -> Sound card support (SOUND [=y]) -> Advanced Linux Sound Architecture (SND [=y]) -> <> OSS Mixer API // 不选中 -> <> OSS PCM (digital audio) API // 不选中

结果如图 2 3 所示：

**图 2-3 取消 OSS API 接口**

![](photo/Linux%20音频驱动实验/2-2%20取消OSS%20API接口.jpeg)

**2、使能 I.MX6ULL 的 WM8960 驱动** 配置路径：
-> Device Drivers -> Sound card support (SOUND [=y]) -> Advanced Linux Sound Architecture (SND [=y]) -> ALSA for SoC audio support (SND_SOC [=y]) -> SoC Audio for Freescale CPUs -> <*> Asynchronous Sample Rate Converter (ASRC) module support // 异步采样率转换器 -> <*> SoC Audio support for i.MX boards with wm8960

如图 2 4 所示：

**图 2-4 使能 ASRC 和 wm8960 驱动**

![](photo/Linux%20音频驱动实验/2-3%20使能ASRC和wm8960驱动.jpeg)

配置完成后重新编译内核，生成新zImage与设备树 dtb。 使用新内核与设备树启动开发板，若设备树、内核驱动配置均正确，内核启动 log 会打印声卡识别相关日志。

### 65.3.6 使能内核 ES8388 驱动

同样执行内核图形配置命令

```bash
make menuconfig
```

开启以下核心配置项：
- `SND_SOC_FSL_ASOC_CARD = y` //i.MX 平台 ASoC card 机器层驱动支持
- `SND_SOC_ES8388 = y` // ES8388 codec 编解码芯片驱动

重新编译内核、设备树并烧录启动。 系统启动成功后，ALSA 声卡列表会出现声卡名称es8388 audio，该名称和设备树中sound es8388节点内model属性保持一致。

常用验证命令，板子启动后执行：

```bash
aplay -l      # 查看识别到的声卡
amixer        # 查看音频通路、音量控制项
```

---

## 65.4 alsa-lib和alsa-utils移植

首先下载alsa-lib和alsa-utils源码，官网地址：`http://www.alsa-project.org/main/index.php/Main_Page` ，本章使用版本1.2.2。源码也放在开发板光盘：例程源码 → 7、第三方库源码 → alsa-lib-1.2.2.tar.bz2、alsa-utils-1.2.2.tar.bz2。

### 65.4.1 alsa-lib 移植

说明：alsa-lib编译会生成带绝对路径的配置文件，为保证Ubuntu主机和开发板根文件系统路径一致，两边都创建完全相同目录 `/usr/share/arm-alsa`。

Ubuntu主机端创建目录：

```bash
cd /usr/share
sudo mkdir arm-alsa
```

开发板根文件系统创建目录：

```bash
mkdir -p /usr/share/arm-alsa
```

alsa-utils依赖alsa-lib，必须先交叉编译alsa-lib。alsa-lib是ALSA基础库，应用程序调用该库操作声卡。 新建存放编译产物的目录，将源码包拷贝到Ubuntu并解压：

```bash
tar -vxjf alsa-lib-1.2.2.tar.bz2
```

解压得到alsa-lib-1.2.2源码文件夹，进入源码目录执行配置、编译：

```bash
cd alsa-lib-1.2.2/
# --host：指定交叉编译器；--prefix：编译产物安装路径；--with-configdir：alsa配置文件目录
./configure --host=arm-linux-gnueabihf \
--prefix=/home/zuozhongkai/linux/IMX6ULL/tool/alsa-lib \
--with-configdir=/usr/share/arm-alsa

make
# 直接sudo make install会报交叉编译器找不到的问题
sudo -s               # 切换root用户
source /etc/profile   # 加载交叉编译环境变量
make install          # root下执行安装，不加sudo
su zuozhongkai        # 切回普通用户
```

报错说明：libatopology.la编译失败，根源是sudo切换root后丢失交叉编译器环境变量，按上面root下加载profile方式解决。

编译产物输出到前面`--prefix`指定目录，Ubuntu主机`/usr/share/arm-alsa`存放alsa配置文件。 拷贝库与配置文件到NFS根文件系统：

```bash
# 拷贝alsa库文件到开发板/usr/lib
cd /home/zuozhongkai/linux/IMX6ULL/tool/alsa-lib
sudo cp lib/* /home/zuozhongkai/linux/nfs/rootfs/lib/ -af

# 拷贝alsa配置文件
cd /usr/share/arm-alsa
sudo cp * /home/zuozhongkai/linux/nfs/rootfs/usr/share/arm-alsa/ -raf
```

### 65.4.2 alsa-utils 移植

alsa-utils是ALSA配套工具集，用来测试声卡（aplay、arecord、amixer等）。 解压源码：

```bash
tar -vxjf alsa-utils-1.2.2.tar.bz2
cd alsa-utils-1.2.2/
```

配置、编译，指定alsa-lib头文件与库路径：

```bash
./configure --host=arm-linux-gnueabihf \
--prefix=/home/zuozhongkai/linux/IMX6ULL/tool/alsa-utils \
--with-alsa-inc-prefix=/home/zuozhongkai/linux/IMX6ULL/tool/alsa-lib/include/ \
--with-alsa-prefix=/home/zuozhongkai/linux/IMX6ULL/tool/alsa-lib/lib/ \
--disable-alsamixer \
--disable-xmlto

make
sudo make install
```

重点说明： 配置项`--disable-alsamixer`：alsamixer是ncurses库支持的图形化声卡调节工具，多次尝试交叉编译很难成功，社区普遍选择关闭。 替代方案：使用amixer（命令行版本，功能等价），本章后面会介绍其他方式移植alsamixer。 遇到问题可在正点原子论坛www.openedv.com发帖交流。

编译完成后，产物在alsa-utils目录，包含bin、sbin、share。拷贝到根文件系统：

```bash
cd /home/zuozhongkai/linux/IMX6ULL/tool/alsa-utils
sudo cp bin/* /home/zuozhongkai/linux/nfs/rootfs/bin/ -rfa
sudo cp sbin/* /home/zuozhongkai/linux/nfs/rootfs/sbin/ -rfa
sudo cp share/* /home/zuozhongkai/linux/nfs/rootfs/usr/share/ -rfa
```

修改开发板根文件系统 `/etc/profile`，添加环境变量，指定alsa配置文件路径：

```bash
export ALSA_CONFIG_PATH=/usr/share/arm-alsa/alsa.conf
```

---

## 65.5 声卡设置与测试

### 65.5.1 amixer 使用方法

声卡各项功能开关默认处于关闭状态，耳机、喇叭声道输出都需要手动配置，amixer是alsa utils提供的声卡调节工具。

**1、查看帮助信息**

```bash
amixer --help    #查看amixer帮助信息
```

amixer分为两套命令集：
- 简单组（带s前缀）：scontrols、scontents、sset、sget，命令简短，本教程使用该组；
- 完整组：controls、contents、cset、cget，功能和简单组一致。

**2、查看设置项**

```bash
amixer scontrols   #查看简化版声卡控制项
amixer controls    #查看全部声卡控制项
```

控制项数量较多，无需全部了解，重点关注耳机、喇叭音量、输入增益等常用项。

**3、查看设置值**

```bash
amixer scontents   #查看各项当前配置值
```

例如Headphone代表耳机音量，范围0 127；部分选项为布尔类型，只有on/off。

**4、设置声卡**

```bash
amixer sset 设置项目 设置值
#或者
amixer cset 设置项目 设置值
```

**5、获取声卡设置值**

```bash
amixer sget 设置项目
#或者
amixer cget 设置项目
```

### 65.5.2 音乐播放测试

**1、使用amixer设置声卡** 播放前打开通路并配置耳机、喇叭音量：

```bash
amixer sset Headphone 100,100
amixer sset Speaker 120,120
amixer sset 'Right Output Mixer PCM' on
amixer sset 'Left Output Mixer PCM' on
```

**2、使用aplay播放WAV格式音乐** 在开发板建立music目录存放音频文件，执行播放：

```bash
aplay test.wav
```

I.MX6U ALPHA支持耳机喇叭自动切换：未插耳机从喇叭播放；插入耳机，喇叭静音，切换耳机输出。

### 65.5.3 WM8960 录音测试

#### 1、MIC录音测试（V2.4及更早底板，板载麦克风）

在`/music`目录新建脚本`mic_in_config.sh`，配置录音通路：

```bash
#!/bin/sh
#正点原子@ALIENTEK
#设置捕获的音量
amixer cset name='Capture Volume' 90,90

#PCM
amixer sset 'PCM Playback' on
amixer sset 'Playback' 256
amixer sset 'Right Output Mixer PCM' on
amixer sset 'Left Output Mixer PCM' on

#ADC PCM
amixer sset 'ADC PCM' 200

#耳机/喇叭（扬声器）设置播放音量，直流/交流
#Turn on Headphone
amixer sset 'Headphone Playback ZC' on
#Set the volume of your headphones(98% volume，127 is the MaxVolume)
amixer sset Headphone 125,125
#Turn on the speaker
amixer sset 'Speaker Playback ZC' on
#Set the volume of your Speaker(98% volume，127 is the MaxVolume)
amixer sset Speaker 125,125
#Set the volume of your Speaker AC(80% volume，100 is the MaxVolume)
amixer sset 'Speaker AC' 4
#Set the volume of your Speaker DC(80% volume，5 is the MaxVolume)
amixer sset 'Speaker DC' 4

#音频输入，左声道管理
#Turn on Left Input Mixer Boost
amixer sset 'Left Input Mixer Boost' off
amixer sset 'Left Boost Mixer LINPUT1' off
amixer sset 'Left Input Boost Mixer LINPUT1' 0
amixer sset 'Left Boost Mixer LINPUT2' off
amixer sset 'Left Input Boost Mixer LINPUT2' 0
#Turn off Left Boost Mixer LINPUT3
amixer sset 'Left Boost Mixer LINPUT3' off
amixer sset 'Left Input Boost Mixer LINPUT3' 0

#音频输入，右声道管理
#Turn on Right Input Mixer Boost
amixer sset 'Right Input Mixer Boost' on
amixer sset 'Right Boost Mixer RINPUT1' off
amixer sset 'Right Input Boost Mixer RINPUT1' 0
amixer sset 'Right Boost Mixer RINPUT2' off
amixer sset 'Right Input Boost Mixer RINPUT2' 0
amixer sset 'Right Boost Mixer RINPUT2' on
amixer sset 'Right Input Boost Mixer RINPUT2' 127
amixer sset 'Right Boost Mixer RINPUT3' off
amixer sset 'Right Input Boost Mixer RINPUT3' 0
```

赋予权限并运行脚本：

```bash
chmod 777 mic_in_config.sh
./mic_in_config.sh
```

使用arecord录制10秒CD质量音频：

```bash
arecord -f cd -d 10 record.wav
```

- `-f cd`：CD录音质量
- `-d 10`：录制时长10秒

录制完成执行`aplay record.wav`回放；板载MIC只接左声道，录制文件只有左声道有声音。

#### 单声道MIC录制立体声音频

硬件MIC只接左声道，可以修改WM8960寄存器，让右声道ADC复用左声道数据，输出立体声。 WM8960 R23寄存器（地址0x17）bit3 2控制ADC数据源：

- 00：左ADC给左声道，右ADC给右声道（默认）
- 01：左右声道都使用左ADC
- 10：左右声道都使用右ADC
- 11：左右声道交换

内核文件wm8960.c，修改wm8960_reg_defaults数组：

```c
static const struct reg_default wm8960_reg_defaults[] = {
    { 0x0, 0x00a7 },
    ......
    { 0x16, 0x00c3 },
    /*{ 0x17, 0x01c0 },*/
    { 0x17, 0x01c4 },
    { 0x18, 0x0000 },
    ......
    { 0x37, 0x00e9 },
};
```

修改完成重新编译内核，烧录启动，MIC录制即可输出立体声。

#### 2、LINE IN 录音测试

如果前面修改R23寄存器为0x01c4，做Line in建议改回0x01c0，Line in硬件是完整双声道。 Line in不能接麦克风；使用3.5mm公对公音频线，一端接手机/电脑，一端接开发板Line in接口。

新建脚本`line_in_config.sh`

```bash
#!/bin/sh
#正点原子@ALIENTEK
#设置捕获的音量
amixer cset name='Capture Volume' 100,100

#PCM
amixer sset 'PCM Playback' on
amixer sset 'Playback' 256
amixer sset 'Right Output Mixer PCM' on
amixer sset 'Left Output Mixer PCM' on

#ADC PCM
amixer sset 'ADC PCM' 200

#耳机/喇叭（扬声器）设置播放音量，直流/交流
#Turn on Headphone
amixer sset 'Headphone Playback ZC' on
#Set the volume of your headphones(98% volume，127 is the MaxVolume)
amixer sset Headphone 125,125
#Turn on the speaker
amixer sset 'Speaker Playback ZC' on
#Set the volume of your Speaker(98% volume，127 is the MaxVolume)
amixer sset Speaker 125,125
#Set the volume of your Speaker AC
amixer sset 'Speaker AC' 4
#Set the volume of your Speaker DC
amixer sset 'Speaker DC' 4

#音频输入，左声道管理
#Turn off Left Input Mixer Boost
amixer sset 'Left Input Mixer Boost' on
amixer sset 'Left Boost Mixer LINPUT1' off
amixer sset 'Left Input Boost Mixer LINPUT1' 0
amixer sset 'Left Boost Mixer LINPUT2' on
amixer sset 'Left Input Boost Mixer LINPUT2' 127
amixer sset 'Left Boost Mixer LINPUT3' off
amixer sset 'Left Input Boost Mixer LINPUT3' 0

#音频输入，右声道管理
#Turn on Right Input Mixer Boost
amixer sset 'Right Input Mixer Boost' on
amixer sset 'Right Boost Mixer RINPUT1' off
amixer sset 'Right Input Boost Mixer RINPUT1' 0
amixer sset 'Right Boost Mixer RINPUT2' off
amixer sset 'Right Input Boost Mixer RINPUT2' 0

#Line in 使用RINPUT3
amixer sset 'Right Boost Mixer RINPUT3' on
amixer sset 'Right Input Boost Mixer RINPUT3' 127
```

赋予权限运行：

```bash
chmod 777 line_in_config.sh
./line_in_config.sh
```

录制音频：

```bash
arecord -f cd -d 10 record.wav
```

回放`aplay record.wav`，Line in硬件为双声道，录制得到立体声。

### 65.5.4 ES8388 录音测试

V2.8及以后底板使用ES8388，出厂文件系统预置测试脚本：`/home/root/shell/audio/atk-audio.sh`。 1、板载MIC录音：直接执行脚本`./atk-audio.sh`，按提示完成MIC录音测试。 2、Line IN录音：3.5mm公对公音频线接信号源与Line in接口，执行`./atk-audio.sh`完成线路输入录音测试。

---

## 65.6 开机自动配置声卡

开发板重启后amixer配置全部丢失，使用alsactl保存声卡状态，开机自动恢复。

**1、创建配置存放目录**

```bash
mkdir /var/lib/alsa -p
```

**2、先使用amixer把声卡通路、音量配置完成，执行保存：**

```bash
alsactl -f /var/lib/alsa/asound.state store
```

- `-f`：指定配置文件路径；store：保存当前声卡状态。 查看帮助：`alsactl -h`。

恢复声卡配置命令：

```bash
alsactl -f /var/lib/alsa/asound.state restore
```

**3、修改开机启动脚本`/etc/init.d/rcS`，文件末尾追加：**

```bash
if [ -f "/var/lib/alsa/asound.state" ]; then
    echo "ALSA: Restoring mixer setting......"
    /sbin/alsactl -f /var/lib/alsa/asound.state restore &
fi
```

逻辑：判断配置文件存在，开机自动加载声卡参数。

保存重启开发板，使用aplay播放音频，验证开机自动配置生效。

---

## 65.7 mplayer 播放器移植与使用

LCD显示和音频驱动调试完成后，可移植mplayer多媒体播放器用来播放音视频。mplayer是开源播放器，自带多种解码器，无需额外单独安装解码器。

### 65.7.1 mplayer 移植

依赖关系：mplayer依赖zlib库、alsa-lib库，必须优先交叉编译zlib与alsa-lib。

**1、移植zlib库**

zlib源码地址：`http://zlib.net/` ，本章使用版本1.2.11 光盘路径：例程源码 → 7、第三方库源码 → zlib-1.2.11.tar.gz

拷贝源码到Ubuntu主机并解压：

```bash
tar -vxzf zlib-1.2.11.tar.gz
```

新建zlib目录存放编译产物，进入源码目录配置编译：

```bash
cd zlib-1.2.11/
CC=arm-linux-gnueabihf-gcc LD=arm-linux-gnueabihf-ld AD=arm-linux-gnueabihf-as \
./configure --prefix=/home/zuozhongkai/linux/IMX6ULL/tool/zlib

make
make install
```

编译完成后拷贝库文件到开发板NFS根文件系统：

```bash
sudo cp lib/* /home/zuozhongkai/linux/nfs/rootfs/lib/ -rfa
```

**2、移植mplayer**

mplayer源码地址：`http://www.mplayerhq.hu/MPlayer/releases/` ，版本1.4 光盘路径：例程源码 → 7、第三方库源码 → MPlayer-1.4.tar.gz

解压源码：

```bash
tar -vxzf MPlayer-1.4.tar.gz
```

新建mplayer目录存放编译产物，进入源码目录配置编译：

```bash
cd MPlayer-1.4/
./configure --cc=arm-linux-gnueabihf-gcc \
--host-cc=gcc \
--target=arm-linux-gnueabihf \
--disable-ossaudio \
--enable-alsa \
--prefix=/home/zuozhongkai/linux/IMX6ULL/tool/mplayer \
--extra-cflags="-I /home/zuozhongkai/linux/IMX6ULL/tool/zlib/include -I /home/zuozhongkai/linux/IMX6ULL/tool/alsa-lib/include" \
--extra-ldflags="-L/home/zuozhongkai/linux/IMX6ULL/tool/zlib/lib -L/home/zuozhongkai/linux/IMX6ULL/tool/alsa-lib/lib -lasound" \
--enable-fbdev \
--disable-mencoder

make
```

重要修改：打开`config.mak`，找到`INSTALLSTRIP = -s`，删除后面的 `-s`，否则`make install`会失败。

```bash
make install
```

编译产物`bin/mplayer`就是播放器可执行文件，拷贝到开发板根文件系统：

```bash
sudo cp bin/mplayer /home/zuozhongkai/linux/nfs/rootfs/bin/ -f
```

mplayer移植完成。

### 65.7.2 mplayer 使用

查看帮助：

```bash
mplayer -h
```

**1、播放音频**

```bash
mplayer test.flac
```

播放时键盘快捷键：
- `9`：音量增加
- `0`：音量减小
- `左方向键`：倒退10秒
- `右方向键`：快进10秒
- `上方向键`：倒退1分钟
- `下方向键`：快进1分钟
- `空格`：暂停/播放

注意：mplayer音量拉满但耳机音量仍然偏小，需要使用amixer调整声卡硬件音量。

**2、视频播放测试**

基础播放命令：`mplayer 视频文件名`，画面默认不居中。 添加`-fs`参数实现全屏居中播放：

```bash
mplayer test.avi -fs
```

⚠️ I.MX6ULL无硬件视频解码，CPU性能弱，不支持高分辨率、高码率、高帧率视频；推荐分辨率 640×480左右。

---

## 65.8 alsamixer 简介

前面移植alsa-utils时，alsamixer是图形化声卡配置工具，依赖ncurses库，直接编译alsa-utils很难成功编译出alsamixer，教程中采用替代方案：使用Buildroot编译生成alsamixer，再单独拷贝可执行文件到开发板根文件系统。

不熟悉Buildroot的同学可以跳过本节，直接使用amixer命令行配置声卡。 产品开发推荐使用Buildroot/Yocto构建根文件系统，自动化处理各类库依赖。

开发板直接输入命令打开图形界面：

```bash
alsamixer
```

按键说明：
- `F1`：帮助
- `F2`：系统信息
- `F3`：播放控制页
- `F4`：录音控制页
- `F6`：选择声卡（多声卡场景）
- `左右方向键`：切换控制项（Headphone、Speaker等）
- `上下方向键`：调节音量数值
- `M`：静音切换；MM=静音，OO=开启