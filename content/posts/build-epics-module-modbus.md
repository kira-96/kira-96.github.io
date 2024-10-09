---
title: "EPICS的MODBUS模块的编译和使用"
date: 2024-01-02T14:38:38+08:00
lastmod: 2024-01-03T08:52:53+08:00
draft: false
description: 交叉编译EPICS的MODBUS模块
tags: ["linux", "EPICS", "龙芯"]
keywords: ["linux", "EPICS", "龙芯"]
categories: ["EPICS"]
---

## 前言

MODBUS是一种应用层消息传递协议，通常用于 I/O 系统通信和可编程逻辑控制器（PLC）通信。

|链接类型|描述|
|:---:|:---:|
|MODBUS TCP|TCP/IP 使用502端口|
|MODBUS RTU|RTU通常通过串行通信链路运行，即RS-232、 RS-422 或 RS-485。RTU 使用额外的 CRC 进行数据包检查。协议直接将每个字节作为 8 个数据位传输，因此使用“二进制” 而不是 ASCII 编码。使用串行链路开始和结束时，消息帧是按时间而不是按特定字符检测的。|
|MODBUS ASCII|串行协议，通常在串行通信链路上运行，即 RS-232、RS-422 或 RS-485。串行 ASCII 使用额外的 LRC 数据包检查。该协议将每个字节编码为 2 个 ASCII 字符。消息帧的开始和结束由特定字符检测 （“:” 开始消息，CR/LF 结束消息）。该协议效率低于 Modbus RTU，但在某些环境中可能更可靠。|

Modbus 提供对以下 4 种类型的数据的访问：

|主表|对象类型|访问|说明|
|:---:|:---:|:---:|:---:|
|离散输入|1bit|只读|这种类型的数据可以由 I/O 系统提供。|
|线圈|1bit|读写|此类数据可由应用程序更改。|
|输入寄存器|16位字(2字节)|只读|这种类型的数据可以由 I/O 系统提供。|
|保持寄存器|16位字(2字节)|读写|此类数据可由应用程序更改。|

Modbus 通信由从 Modbus 客户端发送到 Modbus 服务器的请求消息组成。服务器使用响应消息进行回复。Modbus 请求消息包含：

- 描述数据传输类型的 Modbus 功能码（1字节）。
- Modbus 地址（2字节），用于描述从服务器中读取或写入数据的地址。
- 对于写入操作，则需要传输写入的数据。

Modbus模块 支持以下 9 个 Modbus 功能码：

|访问|功能说明|功能码|
|:---:|:---:|:---:|
|1bit|读取线圈|1|
|1bit|读取离散输入|2|
|1bit|写入单线圈|5|
|1bit|写入多个线圈|15|
|16位字访问(2字节)|读取输入寄存器|4|
|16位字访问(2字节)|读取保持寄存器|3|
|16位字访问(2字节)|写入单个寄存器|6|
|16位字访问(2字节)|写入多个寄存器|16|
|16位字访问(2字节)|读/写多个寄存器|23|

Modbus读取操作仅限于传输125个16位字或2000 bit。Modbus写入操作仅限于传输123个16位字或1968 bit。

## 编译MODBUS模块

### 使用到的模块下载地址

- [epics-base - (launchpad.net)](https://git.launchpad.net/epics-base) / [epics-base/epics-base](https://github.com/epics-base/epics-base) / [EPICS Base (anl.gov)](https://epics.anl.gov/base/index.php)

- [epics-modules/asyn: EPICS module for driver and device support](https://github.com/epics-modules/asyn)

- [epics-modules/modbus: EPICS support for communication with Programmable Logic Controllers (PLCs) and other devices via the Modbus protocol over TCP, serial RTU, and serial ASCII links ](https://github.com/epics-modules/modbus)

- [epics-modules/sscan: APS BCDA synApps module: sscan](https://github.com/epics-modules/sscan)

- [epics-modules/calc: APS BCDA synApps module: calc](https://github.com/epics-modules/calc)

- [epics-modules/ipac: IPAC Carrier and Communication Module Drivers](https://github.com/epics-modules/ipac)

- [sequencer](https://www-csr.bessy.de/control/SoftDist/sequencer/repo/branch-2-2.git/) / [Download and Installation — EPICS Sequencer Version 2.2 (bessy.de)](https://www-csr.bessy.de/control/SoftDist/sequencer/Installation.html)

**以下步骤需要先安装好EPICS Base.**

### 编译 SSCAN（可选）

``` shell
cd sscan
touch configure/RELEASE.local
vi configure/RELEASE.local

# 修改成和EPICS Base一样的架构
# EPICS_HOST_ARCH=linux-loong64
# EPICS Base路径（示例）
EPICS_BASE=/home/ubuntu/loongson/base-7.0.8
# 放置EPICS模块的路径（示例）
SUPPORT=/home/ubuntu/loongson/modules

# 直接编译
# make
# 交叉编译（示例）
# make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
make
```

### 编译 CALC（可选）

``` shell
cd calc
touch configure/RELEASE.local
vi configure/RELEASE.local

# 修改成和EPICS Base一样的架构
# EPICS_HOST_ARCH=linux-loong64
# EPICS Base路径（示例）
EPICS_BASE=/home/ubuntu/loongson/base-7.0.8
# 放置EPICS模块的路径（示例）
SUPPORT=/home/ubuntu/loongson/modules
# SSCAN模块路径
SSCAN=$(SUPPORT)/sscan

# 直接编译
# make
# 交叉编译（示例）
# make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
make
```

### 编译 asyn（必需）

``` shell
cd asyn
touch configure/RELEASE.local
vi configure/RELEASE.local

# 修改成和EPICS Base一样的架构
# EPICS_HOST_ARCH=linux-loong64
# EPICS Base路径（示例）
EPICS_BASE=/home/ubuntu/loongson/base-7.0.8
# 放置EPICS模块的路径（示例）
SUPPORT=/home/ubuntu/loongson/modules
# SSCAN模块路径
SSCAN=$(SUPPORT)/sscan
# CALC模块路径
CALC=$(SUPPORT)/calc

# 直接编译
# make
# 交叉编译（示例）
# make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
make
```

### 编译 modbus

``` shell
cd modbus
touch configure/RELEASE.local
vi configure/RELEASE.local

# 修改成和EPICS Base一样的架构
# EPICS_HOST_ARCH=linux-loong64
# EPICS Base路径（示例）
EPICS_BASE=/home/ubuntu/loongson/base-7.0.8
# 放置EPICS模块的路径（示例）
SUPPORT=/home/ubuntu/loongson/modules
# ASYN模块路径
ASYN=$(SUPPORT)/asyn

# 直接编译
# make
# 交叉编译（示例）
# make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
make
```

编译完成后，可以看到`bin\<EPICS_HOST_ARCH>`路径下生成了可执行程序`modbusApp`，它就是与Modbus设备通信的主程序了。

## 使用 MODBUS 程序

在Modbus模块的`iocBoot\iocTest`目录下，可以看到很多示例程序。这里总结一下，我们使用时主要需要编写两部分内容。

- 用于配置设备连接和通信的`.cmd`文件
- 用于使用模板解析数据的`.substitutions`文件

这里给出示例并做简要说明。

`envPaths`文件：用于配置程序运行时的环境变量路径。  
这里需要配置好`base`、`asyn`、`modbus`模块的路径。

``` shell
# envPaths

epicsEnvSet("IOC","app")
epicsEnvSet("TOP","..")
epicsEnvSet("SUPPORT","/root/modules")
epicsEnvSet("ASYN","/root/modules/asyn")
epicsEnvSet("MODBUS","/root/modules/modbus")
epicsEnvSet("EPICS_BASE","/root/base")
# epicsEnvSet("EPICS_CAS_SERVER_PORT", 9001)
```

``` shell
# AMSAMOTION.cmd

< envPaths

dbLoadDatabase("$(MODBUS)/dbd/modbusApp.dbd")
modbusApp_registerRecordDeviceDriver(pdbbase)

# MODBUS TCP 配置
# Use the following commands for TCP/IP
#drvAsynIPPortConfigure(const char *portName,
#                       const char *hostInfo,
#                       unsigned int priority,
#                       int noAutoConnect,
#                       int noProcessEos);
drvAsynIPPortConfigure("AMSAMOTION","192.168.xxx.xxx:502",0,0,1)
#asynSetOption("AMSAMOTION",0, "disconnectOnReadTimeout", "Y")

# MODBUS RTU配置
#drvAsynSerialPortConfigure(const char *portName, 
#                           const char *ttyName, 
#                           unsigned int priority,
#                           int noAutoConnect,
#                           int noProcessEos);

# drvAsynSerialPortConfigure("Koyo1", "/dev/ttyS1", 0, 0, 0)
# asynSetOption("Koyo1",0,"baud","38400")
# asynSetOption("Koyo1",0,"parity","none")
# asynSetOption("Koyo1",0,"bits","8")
# asynSetOption("Koyo1",0,"stop","1")

# Modbus ASCII 还需配置其他项
# asynOctetSetOutputEos("Koyo1",0,"\r\n")
# asynOctetSetInputEos("Koyo1",0,"\r\n")

# 超时设置
#modbusInterposeConfig(const char *portName,
#                      modbusLinkType linkType,
#                      int timeoutMsec,
#                      int writeDelayMsec)
# Modbus Link Type: 0 = TCP/IP，1 = RTU，2 = ASCII
modbusInterposeConfig("AMSAMOTION",0,5000,0)

# 读取/写入配置
#drvModbusAsynConfigure(portName,
#                       tcpPortName,
#                       slaveAddress,
#                       modbusFunction,
#                       modbusStartAddress,
#                       modbusLength,
#                       dataType,
#                       pollMsec,
#                       plcType);
drvModbusAsynConfigure("AMSA:AI", "AMSAMOTION",  1, 4, 0, 6, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:AO1","AMSAMOTION",  1, 6, 0, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:AO2","AMSAMOTION",  1, 6, 1, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:AOSta","AMSAMOTION",1, 3, 0, 2, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DI", "AMSAMOTION",  1, 2, 0, 8, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DO1","AMSAMOTION",  1, 5, 0, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DO2","AMSAMOTION",  1, 5, 1, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DO3","AMSAMOTION",  1, 5, 2, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DO4","AMSAMOTION",  1, 5, 3, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DO5","AMSAMOTION",  1, 5, 4, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DO6","AMSAMOTION",  1, 5, 5, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DO7","AMSAMOTION",  1, 5, 6, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DO8","AMSAMOTION",  1, 5, 7, 1, 0, 100, "AMSA")
drvModbusAsynConfigure("AMSA:DOSta","AMSAMOTION",1, 1, 0, 8, 0, 100, "AMSA")

# Enable ASYN_TRACEIO_HEX on modbus server
asynSetTraceIOMask("AMSAMOTION",0,4)
# Dump up to 512 bytes in asynTrace
asynSetTraceIOTruncateSize("AMSAMOTION",0,512)

dbLoadTemplate("AMSAMOTION.substitutions")

iocInit
```

``` shell
# AMSAMOTION.substitutions

# asyn record for the underlying asyn octet port
file "$(ASYN)/db/asynRecord.db" { pattern
{P,             R,            PORT,         ADDR,   IMAX,    OMAX}
{AMSAMOTION:    OctetAsyn,    AMSAMOTION,      0,      80,      80}
}

file "$(TOP)/db/ai.template" { pattern
{P,              R,        PORT,    OFFSET,     BITS,  EGUL,     EGUF,   PREC,        SCAN}
{AMSAMOTION:,    AI1,   AMSA:AI,         0,   0xFFFF,     0,    65535,      0,  "I/O Intr"}
{AMSAMOTION:,    AI2,   AMSA:AI,         1,   0xFFFF,     0,    65535,      0,  "I/O Intr"}
{AMSAMOTION:,    AI3,   AMSA:AI,         2,   0xFFFF,     0,    65535,      0,  "I/O Intr"}
{AMSAMOTION:,    AI4,   AMSA:AI,         3,   0xFFFF,     0,    65535,      0,  "I/O Intr"}
{AMSAMOTION:,    AI5,   AMSA:AI,         4,   0xFFFF,     0,    65535,      0,  "I/O Intr"}
{AMSAMOTION:,    AI6,   AMSA:AI,         5,   0xFFFF,     0,    65535,      0,  "I/O Intr"}
}

file "$(TOP)/db/ao.template" { pattern
{P,              R,         PORT,   OFFSET,     BITS,  EGUL,    EGUF,   PREC}
{AMSAMOTION:     AO1,   AMSA:AO1,        0,   0xFFFF,     0,    65535,     0}
{AMSAMOTION:     AO2,   AMSA:AO2,        0,   0xFFFF,     0,    65535,     0}
}

file "$(TOP)/db/ai.template" { pattern
{P,              R,           PORT,    OFFSET,     BITS,  EGUL,    EGUF,   PREC,        SCAN}
{AMSAMOTION:,    AO1:STATE,   AMSA:AOSta,   0,   0xFFFF,     0,    65535,      0,  "1 second"}
{AMSAMOTION:,    AO2:STATE,   AMSA:AOSta,   1,   0xFFFF,     0,    65535,      0,  "1 second"}
}

file "$(TOP)/db/bi_bit.template" { pattern
{P,              R,         PORT,  OFFSET,  ZNAM,   ONAM,       ZSV,    OSV,         SCAN}
{AMSAMOTION:     DI1,    AMSA:DI,       0,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DI2,    AMSA:DI,       1,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DI3,    AMSA:DI,       2,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DI4,    AMSA:DI,       3,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DI5,    AMSA:DI,       4,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DI6,    AMSA:DI,       5,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DI7,    AMSA:DI,       6,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DI8,    AMSA:DI,       7,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
}

file "$(TOP)/db/bi_bit.template" { pattern
{P,              R,               PORT,     OFFSET,  ZNAM,   ONAM,       ZSV,    OSV,         SCAN}
{AMSAMOTION:     DO1:STATE,    AMSA:DOSta,       0,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DO2:STATE,    AMSA:DOSta,       1,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DO3:STATE,    AMSA:DOSta,       2,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DO4:STATE,    AMSA:DOSta,       3,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DO5:STATE,    AMSA:DOSta,       4,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DO6:STATE,    AMSA:DOSta,       5,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DO7:STATE,    AMSA:DOSta,       6,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
{AMSAMOTION:     DO8:STATE,    AMSA:DOSta,       7,   OFF,     ON,  NO_ALARM,  MAJOR,   "I/O Intr"}
}


file "$(TOP)/db/bo_bit.template" { pattern
{P,                R,         PORT,  OFFSET,  ZNAM, ONAM}
{AMSAMOTION:     DO1,     AMSA:DO1,       0,   OFF,   ON}
{AMSAMOTION:     DO2,     AMSA:DO2,       0,   OFF,   ON}
{AMSAMOTION:     DO3,     AMSA:DO3,       0,   OFF,   ON}
{AMSAMOTION:     DO4,     AMSA:DO4,       0,   OFF,   ON}
{AMSAMOTION:     DO5,     AMSA:DO5,       0,   OFF,   ON}
{AMSAMOTION:     DO6,     AMSA:DO6,       0,   OFF,   ON}
{AMSAMOTION:     DO7,     AMSA:DO7,       0,   OFF,   ON}
{AMSAMOTION:     DO8,     AMSA:DO8,       0,   OFF,   ON}
}
```

最后运行程序，在终端执行：

``` shell
/path/to/modbus/bin/<EPICS_HOST_ARCH>/modbusApp AMSAMOTION.cmd
```

或者在`.cmd`文件第一行添加下面一行：

``` shell
#!../bin/<EPICS_HOST_ARCH>/modbusApp
```

然后直接执行`.cmd`脚本。

``` shell
chmod +x AMSAMOTION.cmd
./AMSAMOTION.cmd
```

**参考**

- [Overview of Modbus](https://epics-modbus.readthedocs.io/en/latest/overview.html)
- [Creating a modbus port driver](https://epics-modbus.readthedocs.io/en/latest/creating_driver.html)
- [EPICS Process Database Concepts](https://docs.epics-controls.org/en/latest/process-database/EPICS_Process_Database_Concepts.html)
