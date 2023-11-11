---
title: "Linux LED子系统"
date: 2023-11-11T15:45:11+08:00
draft: false
description: Linux下的LED控制
tags: ["linux"]
keywords: ["linux", "GPIO", "LED"]
categories: ["技术"]
---

## 前言

Linux 内核提供了丰富的设备驱动接口，其中GPIO和LED属于是最基本的一类了。之前就已经讲过[用户空间下的GPIO读写操作](../linux-gpio操作其一/#用户空间下的gpio读写操作)，LED设备的操作也基本相同。其实完全可以使用GPIO驱动去控制LED，但LED的驱动针对LED提供了更多的功能，一起来看一下吧。

## 配置设备树

设备树中的LED节点配置，例：

``` c
/* kernel/arch/arm/boot/dts/imx6ul-14x14-evk-c-emmc.dts */
leds {
        compatible = "gpio-leds";
        pinctrl-names = "default";
        status = "okay";

        led1{
                label = "led1";
                gpios = <&gpio5 9 GPIO_ACTIVE_LOW>;
                default-state = "off";
        };
        led2{
                label = "led2";
                gpios = <&gpio1 9 GPIO_ACTIVE_LOW>;
                default-state = "off";
        };
        led3{
                label = "heartbeat";
                gpios = <&gpio5 5 GPIO_ACTIVE_LOW>;
                linux,default-trigger = "heartbeat";
        };
};
```

节点属性说明：

`label`：LED设备的名字，名字必须是唯一的。如果没有设置，则会使用节点的名字。

`gpios`：GPIO的编号，以及高低电平设置，`GPIO_ACTIVE_LOW`低电平点亮，`GPIO_ACTIVE_HIGH`高电平点亮。

`default-state`：默认状态，`on/off`。

`linux,default-trigger`：设置LED的触发器。`backlight`-背光灯，`heartbeat`-心跳灯，`timer`-定时，`default-on`-默认开状态，`disk-activity`-硬盘状态，`gpio`，`none`。

## 用户空间下的LED操作

> 注意：以下操作都需要root权限！

用户空间下的GPIO文件系统接口在`/sys/class/leds/{label}`目录下。

LED节点有以下属性可以配置：

- **trigger**

  设置LED的触发器。

``` sh
echo heartbeat > /sys/class/leds/led1/trigger
```

- **brightness**

  设置LED的开关或者亮度。

``` sh
# 关闭LED
echo 0 > /sys/class/leds/led1/brightness
# 打开LED
echo 1 > /sys/class/leds/led1/brightness
```

  对于写入的任何非0值，都会是打开LED的操作，可写入值范围为`0~255`。

  而对于使用PWM控制的LED灯，`brightness`才可以控制灯的亮度。参考[呼吸灯](../龙芯2k500开发板上实现的呼吸灯效果/)的实现。

## 程序控制LED灯

示例读取：

``` cpp
quint8 readLed(const QString &led)
{
    QFile file(QString("/sys/class/leds/%1/brightness").arg(led));
    if (!file.open(QIODevice::ReadOnly))
        return 0;

    QByteArray ba = file.readAll();
    file.close();

    return (quint8)QString(ba).toInt();
}
```

示例写入：

``` cpp
bool writeLed(const QString &led, quint8 value)
{
    QFile file(QString("/sys/class/leds/%1/brightness").arg(led));
    if (!file.open(QIODevice::WriteOnly))
        return false;

    QByteArray ba = QString::number(value).toLatin1();
    return file.write(ba) == ba.size();
}
```

**参考**

* [Documentation/devicetree/bindings/leds/common.txt (v4.13)](https://lwn.net/Articles/730227/)
* [LED子系统详解](https://zhuanlan.zhihu.com/p/633680990)
