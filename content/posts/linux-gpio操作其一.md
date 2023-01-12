---
title: "Linux GPIO 操作其一"
date: 2023-01-12T13:33:37+08:00
draft: false
description: Linux下GPIO的读写操作
tags: ["linux"]
keywords: ["linux", "GPIO"]
categories: ["技术"]
---

## 前言

由于换了新的工作，我的工作方向也有了很大的变化，之前基本上是单纯的写代码，现在则经常需要和硬件设备交互，开发平台也转到了Linux+Qt。硬件设备的控制，其中最基本的就是LED灯以及一些开关继电器的操作，其本质就是GPIO的操作。考虑到系统的精简和成本控制，最好是可以直接通过Linux系统去控制，当然也有其它替代方案，比如使用支持Modbus协议的IO模块。关于Modbus的使用，后面有空再讲，这里就记录一下最简单的Linux系统下的GPIO控制，**用户空间下的GPIO文件系统接口**。

在此之前，有必要再了解一下GPIO的概念。

“通用输入/输出”（GPIO）是一种灵活的软件控制数字信号。它们由多种芯片提供，对于使用嵌入式和定制硬件的Linux开发人员来说很熟悉。每个GPIO代表一个连接到特定引脚的位，即球栅阵列（BGA）封装上的“球”。电路板示意图显示了哪些外部硬件连接到哪些GPIO。驱动程序可以通用地编写，以便板设置代码将这样的引脚配置数据传递给驱动程序。

> A “General Purpose Input/Output” (GPIO) is a flexible software-controlled digital signal. They are provided from many kinds of chip, and are familiar to Linux developers working with embedded and custom hardware. Each GPIO represents a bit connected to a particular pin, or “ball” on Ball Grid Array (BGA) packages. Board schematics show which external hardware connects to which GPIOs. Drivers can be written generically, so that board setup code passes such pin configuration data to drivers.

在单片机上，我们可以很方便的控制GPIO，但在嵌入式Linux上则不一样，通常GPIO对于用户来说是不可见的，不过Linux系统也提供了相应的接口供用户控制GPIO，每个非专用的引脚都可以用作GPIO。

## 配置IO多路复用器（IOMUXC）

将需要复用的IO添加到`pinctrl_hog`节点，例：

``` c {13-18}
/* kernel/arch/arm64/boot/dts/freescale/OK8MP-C.dts */
&iomuxc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hog>;

	pinctrl_hog: hoggrp {
		fsl,pins = <
			MX8MP_IOMUXC_HDMI_DDC_SCL__HDMIMIX_HDMI_SCL	0x400001c3
			MX8MP_IOMUXC_HDMI_DDC_SDA__HDMIMIX_HDMI_SDA	0x400001c3
			MX8MP_IOMUXC_HDMI_HPD__HDMIMIX_HDMI_HPD		0x40000019
			MX8MP_IOMUXC_HDMI_CEC__HDMIMIX_HDMI_CEC		0x40000019
			/* GPIO */
			MX8MP_IOMUXC_GPIO1_IO07__GPIO1_IO07			0x159
			MX8MP_IOMUXC_GPIO1_IO09__GPIO1_IO09			0x159
			MX8MP_IOMUXC_GPIO1_IO12__GPIO1_IO12			0x159
			MX8MP_IOMUXC_ECSPI2_MOSI__GPIO5_IO11		0x159
			MX8MP_IOMUXC_ECSPI2_MISO__GPIO5_IO12		0x159
			MX8MP_IOMUXC_ECSPI2_SS0__GPIO5_IO13			0x159
		>;
	};
}
```

具体的GPIO名字要参照`xxxx-pinfunc.h`里面的定义，配置为GPIO时，一定要使用`IOMUXC_xxxx_xxxx__GPIOn_IOxx`的宏定义。

然后是后面的上下拉配置，具体的计算方法和参数意义如下：

``` c
PAD_CTL_HYS                     (1 << 16) /* Hysteresis 滞后使能*/
PAD_CTL_PUS_100K_DOWN           (0 << 14) /* 100KOhm Pull Down */
PAD_CTL_PUS_47K_UP              (1 << 14) /* 47KOhm Pull Up */
PAD_CTL_PUS_100K_UP             (2 << 14) /* 100KOhm Pull Up */
PAD_CTL_PUS_22K_UP              (3 << 14) /* 22KOhm Pull Up */
PAD_CTL_PUE                     (1 << 13) /* Pull / Keep Enable */
PAD_CTL_PKE                     (1 << 12) /* Pull / Keep Select 0: Keeper 1: Pull */
PAD_CTL_ODE                     (1 << 11) /* Open Drain Enable 漏极开路 */
PAD_CTL_SPEED_LOW               (1 << 6)  /* 带宽配置 */
PAD_CTL_SPEED_MED               (2 << 6)
PAD_CTL_SPEED_HIGH              (3 << 6)
PAD_CTL_DSE_DISABLE             (0 << 3)  /* Drive Strength Field 驱动能力 */
PAD_CTL_DSE_240ohm              (1 << 3)
PAD_CTL_DSE_120ohm              (2 << 3)
PAD_CTL_DSE_80ohm               (3 << 3)
PAD_CTL_DSE_60ohm               (4 << 3)
PAD_CTL_DSE_48ohm               (5 << 3)
PAD_CTL_DSE_40ohm               (6 << 3)
PAD_CTL_DSE_34ohm               (7 << 3)
PAD_CTL_SRE_FAST                (1 << 0)  /* Slew Rate Field 压摆率 */
PAD_CTL_SRE_SLOW                (0 << 0)
```

> 注意：不要直接设置为`0`，没有任何作用。可以使用`0x80000000`，它表示“我不知道，保持默认值”。

## 用户空间下的GPIO读写操作

用户空间下的GPIO文件系统接口在`/sys/class/gpio/`目录下。

> 注意：以下操作都需要root权限！

### 使能GPIO

在此之前，需要先知道GPIO对应的编号数值，计算方法如下：

GPIOn_IOx = (n - 1) × 32 + x

例：GPIO5_IO13 = (5 - 1) × 32 + 13 = 141

执行命令

``` sh
# echo N > /sys/class/gpio/export
# N为GPIO对应的编号，例：
echo 141 > /sys/class/gpio/export
```

如果需要取消使能GPIO，则执行命令

``` sh
# echo N > /sys/class/gpio/unexport
# N为GPIO对应的编号，例：
echo 141 > /sys/class/gpio/unexport
```

### GPIO配置

使能GPIO之后，`/sys/class/gpio/`目录下就多出来了相应的GPIO节点目录。例：`gpio141`

GPIO节点有以下属性可以配置：

`/sys/class/gpio/gpioN/`

* **direction**

  读取为`in`或者`out`。通常可以写入此值。写入`out`默认输出为低。为了确保操作无误，可以写入值“low”和“high”将GPIO配置为具有该初始值的输出。

  请注意，如果内核不支持更改GPIO的方向，或者该属性是由内核代码导出的，而内核代码没有明确允许用户空间重新配置该GPIO方向，则该属性将不存在。

* **value**

  读取为`0`（low）或`1`（high）。如果GPIO被配置为输出，则可以写入该值；**任何非零值都被视为高**。
  
  如果引脚可以配置为中断生成，并且它已经配置为生成中断（请参阅“edge”的描述），那么您可以对该文件进行轮询（poll），每当触发中断时，轮询（poll）将返回。如果使用poll，请设置事件为`POLLPRI`和`POLLERR`。如果使用select，则将文件描述符设置为`exceptfds`。轮询返回后，要么lseek到sysfs文件的开头并读取新值，要么关闭文件并重新打开以读取值。

* **edge**

  读取为`none`、`rising`、`falling`或者`both`。编写这些字符串以选择将对“value”文件返回进行轮询的信号边缘。
  
  仅当引脚可以配置为中断生成输入引脚时，此属性才存在。

* **active_low**

  读取为0（假）或1（真）。写入任何非零值以反转读取和写入的值属性。现有和后续轮询支持通过边缘属性配置“rising”和“falling”边缘将遵循此设置。

例：

将GPIO配置为输出

``` sh
echo out > /sys/class/gpio/gpio{N}/direction
```

将GPIO配置为输入，上升沿触发中断

``` sh
echo in > /sys/class/gpio/gpio{N}/direction
echo rising > /sys/class/gpio/gpio{N}/edge
```

### GPIO读写

GPIO配置为输入（in）时，只能读取输入值，不能写入

GPIO配置为输出（out）时，可以读取当前值和写入新值

``` sh
# 读取时
cat /sys/class/gpio/gpio{N}/value
# 写入时
echo 0 > /sys/class/gpio/gpio{N}/value
echo 1 > /sys/class/gpio/gpio{N}/value
```

### 程序控制GPIO

示例读取：

``` cpp
int readGpio(unsigned short io)
{
    QFile file(QString("/sys/class/gpio/gpio%1/value").arg(io));
    if (!file.open(QIODevice::ReadOnly))
        return 0;

    QByteArray ba = file.readAll();
    file.close();

    return QString(ba).toInt();
}
```

示例写入：

``` cpp
bool writeGpio(unsigned short io, unsigned char value)
{
    char buf[128];
    sprintf(buf, "echo %d > /sys/class/gpio/gpio%d/value", value, io);
    return ::system(buf) == 0;
}
```

**参考**

* [Legacy GPIO Interfaces](https://www.kernel.org/doc/html/latest/driver-api/gpio/legacy.html)
* [GPIO Sysfs Interface for Userspace](https://www.kernel.org/doc/html/latest/admin-guide/gpio/sysfs.html)
* [Definitive GPIO guide](https://kosagi.com/w/index.php?title=Definitive_GPIO_guide)
