---
title: "EPICS PVAccess 使用方法研究"
date: 2025-01-13T10:40:45+08:00
lastmod: 2025-01-13T10:43:39+08:00
draft: false
description: EPICS PVAccess 使用方法研究笔记
tags: ["linux", "EPICS"]
keywords: ["linux", "EPICS"]
categories: ["EPICS"]
---

## 前言

EPICS V7 中引入了 pvAccess、pvData等相关模块，增加了对结构化数据的支持。
pvData (Process Variable Data, 过程变量数据) 是EPICS核心软件的一部分，它是一个运行时类型系统，具有用于处理结构化数据的序列化和内省（introspection）功能。

**pvData**有四种类型的数据字段：`scalar`、`scalarArray`、`structure`和`structureArray`。`scalar`（标量）可以是以下标量类型之一：Boolean、Byte、Short、Int、Long、U(nsigned)Byte、Unsigned Short、Unsigned Int、Unsigned Long、Float、Double和String。`scalarArray`是一维数组，元素类型为任何标量类型。`structure`（结构体）是一组有序的字段，其中每个字段都有一个名称和类型。`structureArray`是结构体数组，由于字段可以是结构，因此可以创建复杂的结构。

[**QSRV**](https://epics-base.github.io/pva2pva/qsrv_page.html)是一个使用PVAccess协议的网络服务器，在EPICS IOC进程中运行，允许客户端请求访问其中的过程变量（PV）。
[**PVXS**](https://epics-base.github.io/pvxs/)是一个PVAccess协议客户端/服务器的程序模块。功能等同于 [pvDataCPP](https://github.com/epics-base/pvDataCPP)，并希望最终能取代[CPP](https://github.com/epics-base/pvAccessCPP)模块。

PVAccess 默认端口：5076

环境变量表：

| Variable                         | Client | Server |
| -------------------------------- | ------ | ------ |
| EPICS_PVA_ADDR_LIST              | ×      | ×      |
| EPICS_PVAS_BEACON_ADDR_LIST      |        | ×      |
| EPICS_PVA_AUTO_ADDR_LIST         | ×      | ×      |
| EPICS_PVAS_AUTO_BEACON_ADDR_LIST |        | ×      |
| EPICS_PVAS_INTF_ADDR_LIST        |        | ×      |
| EPICS_PVA_SERVER_PORT            | ×      | ×      |
| EPICS_PVAS_SERVER_PORT           |        | ×      |
| EPICS_PVA_BROADCAST_PORT         | ×      | ×      |
| EPICS_PVAS_BROADCAST_PORT        |        | ×      |
| EPICS_PVAS_IGNORE_ADDR_LIST      |        | ×      |
| EPICS_PVA_CONN_TMO               | ×      | ×      |
| EPICS_PVA_NAME_SERVERS           | ×      |        |

## 快速使用

使用`softIocPVA`软件。

``` cpp
cat <<EOF > p2pexample.db
record(calc, "p2p:example:counter") {
    field(INPA, "p2p:example:counter")
    field(CALC, "A+1")
    field(SCAN, "1 second")
}
EOF

./bin/linux-x86_64/softIocPVA -d p2pexample.db
```
## 添加 QSRV 到 IOC
如果使用EPICS V7创建IOC，那么可以看到程序已经默认添加了**QSRV**到IOC中。如：

``` cpp
# example/iocExampleApp/src/Makefile

# Link QSRV (pvAccess Server) if available
ifdef EPICS_QSRV_MAJOR_VERSION
    iocExampleApp_LIBS += qsrv
    iocExampleApp_LIBS += $(EPICS_BASE_PVA_CORE_LIBS)
    iocExampleApp_DBD += PVAServerRegister.dbd
    iocExampleApp_DBD += qsrv.dbd
endif

# Finally link IOC to the EPICS Base libraries
iocExampleApp_LIBS += $(EPICS_BASE_IOC_LIBS)
```
编译运行IOC后，可直接使用`pvget`、`pvput`、`pvmonitor`、`pvinfo`等命令行工具访问过程变量。

## 添加 PVXS 到 IOC

需要先编译完成EPICS V7。

**源码构建 PVXS**
获取`pvxs`源码：
``` shell
git clone --recursive https://github.com/epics-base/pvxs.git
```
配置`EPICS_BASE`环境变量：
``` shell
cat <<EOF > pvxs/configure/RELEASE.local
EPICS_BASE=\$(TOP)/../epics-base
EOF
```
※ 编译`libevent`（可选）：
``` shell
make -C pvxs/bundle libevent # implies .$(EPICS_HOST_ARCH)
```
※ 交叉编译`libevent`（可选）：
修改 *bundle/Makefile*：
``` cpp
ifneq (,$(filter linux-%,$(EPICS_HOST_ARCH)))
# cross mingw hosted on linux
CMAKE_TOOLCHAIN_windows-x64-mingw ?= x86_64-w64-mingw32
# 添加下面一行
CMAKE_TOOLCHAIN_$(EPICS_HOST_ARCH) ?= $(EPICS_HOST_ARCH).cmake
endif
```

*linux-loong64.cmake*
``` python
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR AMD64)
set(CMAKE_C_COMPILER loongarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER loongarch64-linux-gnu-g++)
set(CMAKE_FIND_ROOT_PATH  /usr/linux-loong64 )
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

交叉编译：
``` shell
make -C pvxs/bundle libevent.linux-loong64 EPICS_HOST_ARCH=linux-loong64
```

命令行工具：
- `pvxcall` - 与 `pvcall` 相似
- `pvxget` -  与 `pvget` 相似
- `pvxinfo` -  与 `pvinfo` 相似
- `pvxmonitor` -  与 `pvmonitor` 或 `pvget -m` 相似
- `pvxput` -  与 `pvput` 相似
- `pvxvct` - UDP search/beacon Troubleshooting tool.

**添加 PVXS 到 IOC**

配置`EPICS_BSAE`和`PVXS`：
``` shell
cat <<EOF >> configure/RELEASE.local
EPICS_BASE=/path/to/your/build/of/epics-base
PVXS=/path/to/your/build/of/pvxs
EOF
```

将`pvxs`和`pvxsIoc`作为依赖库添加到IOC：
``` cpp
# example/iocExampleApp/src/Makefile

# Link PVXS if available
ifdef PVXS_MAJOR_VERSION
    iocExampleApp_LIBS += pvxsIoc pvxs
    iocExampleApp_DBD += pvxsIoc.dbd
else
# Link QSRV (pvAccess Server) if available
ifdef EPICS_QSRV_MAJOR_VERSION
    iocExampleApp_LIBS += qsrv
    iocExampleApp_LIBS += $(EPICS_BASE_PVA_CORE_LIBS)
    iocExampleApp_DBD += PVAServerRegister.dbd
    iocExampleApp_DBD += qsrv.dbd
endif
endif

# Finally link IOC to the EPICS Base libraries
iocExampleApp_LIBS += $(EPICS_BASE_IOC_LIBS)
```

`pvxsIoc`只应包含在IOC中，在编写应用程序时只应该依赖`pvxs`。

**编译运行**
``` shell
make -j 8
```

## QSRV

**单个 PV**
“单个”PV是由CA服务器（RSRV）提供的记录名字，所有记录字段都可以被访问。因此，所有可通过CA访问的数据也可通过PVAccess访问。
QSRV将所有“单个”PV呈现为符合规范类型`NTScalar`、`NTScalarArray`或`NTEnum`的结构，具体取决于原生DBF字段类型。

**定义PV组**
“组”是使用JSON语法定义的，组名称也是PV名称。与“记录”不同：
* Records have fields （记录拥有字段）
* Channels/PVs have properties （通道/PV 拥有属性）
组定义可以在多个记录中拆分，参考example/iocExampleApp/Db/circle.db，或者包含在单独的JSON文件中，参考[JSON reference](https://epics-base.github.io/pvxs/qgroup.html#json-reference)。

拆分写法：
``` json
record(ai, "rec:X") {
    info(Q:group, {
        "grp:name": {
            "X": {+channel:"VAL"}
        }
    })
}
record(ai, "rec:Y") {
    info(Q:group, {
        "grp:name": {
            "Y": {+channel:"VAL"} # .VAL in enclosing record()
        }
    })
}
```

JSON文件写法：
``` json
# Store in some .db
record(ai, "rec:X") {}
record(ai, "rec:Y") {}

# Store in some .json
{
    "grp:name": {
        "X": {"+channel":"rec:X.VAL"}, # full PV name
        "Y": {"+channel":"rec:Y.VAL"}
    }
}
```

加载JSON文件：
``` shell
# void dbLoadGroup(const char *file, const char *macros)
# Load Group definitions from a separate JSON file. eg.
dbLoadGroup "db/some.json", "user=root"
```

**PVAccess 链接**

PVA 链接 [JSON schema](https://epics-base.github.io/pvxs/pvalink-schema-0.json)。

例：
``` json
record(longin, "tgt") {}
record(longin, "src") {
    field(INP, {pva:{pv:"tgt"}})
}
```
或：
``` json
record(longout, "src") {
    field(INP, {pva:{
        pv:"target:pv",
        proc:"CP"
    }})
}
```

**参考链接**
* [EPICS V4 规范类型](https://docs.epics-controls.org/en/latest/pv-access/Normative-Types-Specification.html)
* [EPICS 7, pvAccess and pvData — EPICS Documentation documentation](https://docs.epics-controls.org/en/latest/pv-access/OverviewOfpvData.html)
* [pva2pva: pva2pva Home of QSRV and pvAccess 2 pvAccess gateway](https://epics-base.github.io/pva2pva/)
* [PVXS client/server for PVA Protocol](https://epics-base.github.io/pvxs/) 
* [EPICS 101 - Beam Line Controls](https://wiki-ext.aps.anl.gov/blc/index.php?title=EPICS_101)
* [IOC 101 - Beam Line Controls](https://wiki-ext.aps.anl.gov/blc/index.php?title=IOC_101)