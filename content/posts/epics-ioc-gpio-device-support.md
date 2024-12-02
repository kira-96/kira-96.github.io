---
title: "EPICS IOC GPIO设备支持的使用"
date: 2024-12-02T09:45:59+08:00
draft: false
description: EPICS IOC GPIO设备支持的使用
tags: ["linux", "EPICS", "GPIO"]
keywords: ["linux", "EPICS", "GPIO"]
categories: ["EPICS"]
---

## 前言

之前讲过[Linux的GPIO操作](../linux-gpio操作其一)，主要是使用程序访问Linux系统提供的用户空间接口。这次是介绍如何使用EPICS IOC控制GPIO的输入或输出。EPICS有许多设备支持程序，其中就包括Linux GPIO驱动，对于某些需要使用EPICS控制设备GPIO的情况十分有用。

## 编译GPIO设备支持模块

### 使用到的软件模块

- [epics-base - (launchpad.net)](https://git.launchpad.net/epics-base) / [epics-base/epics-base](https://github.com/epics-base/epics-base) / [EPICS Base (anl.gov)](https://epics.anl.gov/base/index.php)

- [ffeldbauer/epics-devgpio: EPICS device support to control GPIOs on the BeagleBone Black / Raspberry Pi via the /sys/class/gpio interface](https://github.com/ffeldbauer/epics-devgpio/)

**需要先安装好EPICS Base.**

### 编译 devgpio

> 注意：gpio设备支持用到了Linux 5.x 内核提供的gpio应用程序接口。

> Version 2 of this device support uses the new V2 ABI for GPIO character device (c.f. /usr/include/linux/gpio.h) which was introduced in Kernel 5.x.

如果你使用的是旧版的linxu内核，可能需要使用[R1-0-6](https://github.com/ffeldbauer/epics-devgpio/releases/tag/R1-0-6)。

交叉编译时`gpio.h`的路径：`${SDKTARGETSYSROOT}/usr/include/linux/gpio.h`

编译步骤：

``` shell
cd epics-devgpio
touch configure/RELEASE.local
vi configure/RELEASE.local

# 修改成和EPICS Base一样的架构
# EPICS_HOST_ARCH=linux-loong64
# EPICS Base路径（示例）
EPICS_BASE=/home/ubuntu/epics/base-7.0.8.1

# 直接编译
# make
# 交叉编译（示例）
# make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
make
```

## 使用GPIO的设备支持库

为`IOC`程序添加GPIO设备支持，和其他设备支持程序使用方法一样。

示例：

1. 修改`configure/RELEASE`

  添加`devgpio`的模块位置

  ``` Makefile
  SUPPORT=/home/ubuntu/epics/epics-modules
  DEVGPIO=$(SUPPORT)/epics-devgpio
  ```

2. 修改程序的`Makefile`

  例：修改`exampleApp/src/Makefile`

  ``` Makefile
  # 添加以下内容
  ifneq ($(DEVGPIO),)
  example_DBD += devgpio.dbd
  example_LIBS += devgpio
  endif
  ```

3. 编写`db`

  例：添加`exampleApp/Db/gpio.db`

  ``` js
  record(bo, "${IOC}:GPIO:IO39:OUT"） {
    field(DESC, "GPIO 39 output")
    field(DTYP, "devgpio")
    field(OUT, "@39")
    field(ZNAM, "OFF")
    field(ONAM, "ON")
  }

  record(bi, "${IOC}:GPIO:IO38:IN"） {
    field(DESC, "GPIO 38 input")
    field(DTYP, "devgpio")
    field(INP, "@38 both")
    field(SCAN, "I/O Intr")
    field(ZNAM, "OFF")
    field(ONAM, "ON")
  }
  ```

  修改`exampleApp/Db/Makefile`

  ``` Makefile
  # 添加编写的db
  DB += gpio.db
  ```

4. 修改启动脚本`st.cmd`

  例：修改`iocBoot/iocexample/st.cmd`

  ``` shell
  # 设置gpio设备，示例：
  GpioChip "/dev/gpiochip0"
  # 加载db
  dbLoadRecords "db/gpio.db", "IOC=${IOC}"
  ```

5. 编译IOC

  ``` shell
  cd example
  make
  ```

至此，所有步骤都完成了，现在可以运行测试。

6. 测试运行

``` shell
cd iocBoot/iocexample
./st.cmd
```

输入模式：

``` shell
> camonitor iocsysStats:GPIO:IO38:IN
iocsysStats:GPIO:IO38:IN       <undefined> OFF UDF INVALID
iocsysStats:GPIO:IO38:IN       2024-12-02 11:21:05.738442 ON
iocsysStats:GPIO:IO38:IN       2024-12-02 11:21:13.115063 OFF
iocsysStats:GPIO:IO38:IN       2024-12-02 11:21:48.090581 ON
iocsysStats:GPIO:IO38:IN       2024-12-02 11:21:51.567303 OFF
```

输出模式：

``` shell
> caput iocsysStats:GPIO:IO39:OUT 1
Old : iocsysStats:GPIO:IO39:OUT      OFF
New : iocsysStats:GPIO:IO39:OUT      ON
> caput iocsysStats:GPIO:IO39:OUT 0
Old : iocsysStats:GPIO:IO39:OUT      ON
New : iocsysStats:GPIO:IO39:OUT      OFF
```

## 关于`record`的编写

*以下说明中 <> 为必填内容，[] 为可选内容。*

* `devgpio`支持`bi`、`bo`、`mbbi/o`、`mbbi/oDirect`类型的记录。
* `record`的`DTYP`字段必须为`devgpio`。
* `record`的`INP`字段语法为：`@<GPIO1> [GPIO2] [LOW] [FALLING/RISING/BOTH]`
  `GPIOx`为GPIO的编号，具体可以查阅相应开发板的手册。对于`bi`类型的记录，只支持一个GPIO。  
  `LOW`标志可以将GPIO切换到低电平模式，对应的选项为`low`或者`l`。
  `FALLING/RISING/BOTH`表示GPIO输入的中断方式。`falling`/`f`表示*下降沿*触发中断，`rising`/`r`表示*上升沿*触发中断，`both`/`b`表示*两侧*均可以触发中断。**只有启用了此选项，才可以将`SCAN`字段设置为`I/O Intr`**。
* `record`的`OUT`字段语法为：`@<GPIO1> [GPIO2] [LOW]`
  `GPIOx`为GPIO的编号，具体可以查阅相应开发板的手册。对于`bo`类型的记录，只支持一个GPIO。  
  `LOW`标志可以将GPIO切换到低电平模式，对应的选项为`low`或者`l`。
