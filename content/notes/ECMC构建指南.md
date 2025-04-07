---
title: "ecmc构建指南"
date: 2024-11-26T08:56:34+08:00
lastmod: 2024-11-26T08:56:34+08:00
draft: true
searchHidden: true
tags: ["EPICS", "EtherCAT"]
keywords: ["EPICS", "EtherCAT"]
categories: ["EPICS"]
---

## 软件包

* [epics-base/epics-base: The C/C++ core of the EPICS Base control system toolkit](https://github.com/epics-base/epics-base)
* [epics-modules/ecmc: EPICS Support for EtherCAT Motion Controller (ECMC) and Generic IO Controller](https://github.com/epics-modules/ecmc)
* [paulscherrerinstitute/exprtk-ecmc: ESS Customized exprtk : C++ Mathematical Expression Parsing And Evaluation Library](https://github.com/paulscherrerinstitute/exprtk-ecmc)
* [paulscherrerinstitute/ecmccfg: Module to handle configuration scripts for EtherCATMotion Controller (ECMC)](https://github.com/paulscherrerinstitute/ecmccfg)
* [epics-modules/asyn: EPICS module for driver and device support](https://github.com/epics-modules/asyn)
* [EuropeanSpallationSource/motor: APS BCDA synApps module: motor](https://github.com/EuropeanSpallationSource/motor)
* [pantor/ruckig: Motion Generation for Robots and Machines. ](https://github.com/pantor/ruckig)
* [EtherLab / EtherCAT Master · GitLab](https://gitlab.com/etherlab.org/ethercat)

## 构建顺序

1. epics-base
2. asyn
3. motor
4. etherlab
5. ruckig
6. ecmc

### epics-base

正常交叉编译。

### asyn

正常交叉编译。

### motor

配置如下：
``` shell { title="configure/CONFIG_SITE.local" }
# Uncomment the following line to build iocs in motor/modules/motorVendor/iocs
BUILD_IOCS = YES
```
``` shell { title="configure/RELEASE.local" }
EPICS_BASE=/path/to/epics/base-7.0.8.1
SUPPORT=$(EPICS_BASE)/../epics-modules
ASYN=$(SUPPORT)/asyn
```
### etherlab

需要内核源码头文件才能编译，和EPICS不相关。

### ruckig 编译

机器人运动控制库。

配置交叉编译工具链

``` cmake { title="toolchain.cmake" }
set(CMAKE_C_COMPILER "loongarch64-linux-gnu-gcc")
set(CMAKE_CXX_COMPILER "loongarch64-linux-gnu-g++")
```

编译步骤

``` shell
# 新建编译目录
mkdir build
cd build
# 交叉编译
cmake -DCMAKE_TOOLCHAIN_FILE=toolchain.cmake -DCMAKE_BUILD_TYPE=Release ..
make
```
### ecmc

把下载的`exprtk`解压到`exprtkSupport`目录下（也可以用`git` pull下来）。

配置如下：

``` shell { title="configure/RELEASE.local" }
EPICS_BASE=/path/to/epics/base-7.0.8.1
SUPPORT=$(EPICS_BASE)/../epics-modules
ASYN=$(SUPPORT)/asyn
MOTOR=$(SUPPORT)/motor
# 指定交叉编译架构
EPICS_HOST_ARCH=linux-loong64
```

修改`devEcmcSup/Makefile`

``` diff { title="devEcmcSup/Makefile" }
#************************************************************************
# Copyright (c) 2019 European Spallation Source ERIC
# ecmc is distributed subject to a Software License Agreement found
# in file LICENSE that is included with this distribution.
#
# Author: Jeong Han Lee <jeonghan.lee@gmail.com>
#
#*************************************************************************

TOP=..
include $(TOP)/configure/CONFIG
#----------------------------------------
#  ADD MACRO DEFINITIONS AFTER THIS LINE
#=============================


ECMC = $(TOP)/devEcmcSup

LIBRARY_IOC += ecmc

# Ubuntu needs the following ldflags
USR_LDFLAGS += -Wl,--no-as-needed
USR_LDFLAGS += -lstdc++

ifeq ($(T_A),linux-x86_64)
# Assume that the etherlab user library is done via
# https://github.com/icshwi/etherlabmaster
USR_INCLUDES += -I/opt/etherlab/include
USR_CFLAGS += -fPIC
USR_LDFLAGS += -L /opt/etherlab/lib
USR_LDFLAGS += -lethercat
USR_LDFLAGS += -Wl,-rpath=/opt/etherlab/lib
else
# Assume that the etherlab user library is done via
# Yocto ESS Linux bb recipe
+ # 这里注意etherlab的路径
USR_INCLUDES += -I$(SDKTARGETSYSROOT)/usr/include/etherlab
USR_CFLAGS   += -fPIC
USR_LDFLAGS  += -L $(SDKTARGETSYSROOT)/usr/lib/etherlab
USR_LDFLAGS  += -lethercat
USR_LDFLAGS  += -Wl,-rpath=$(SDKTARGETSYSROOT)/usr/lib/etherlab
endif

+ # ruckig路径
+ USR_INCLUDES += -I/path/to/ruckig/include
+ USR_LDFLAGS  += -L /path/to/ruckig/build
+ USR_LDFLAGS  += -lruckig

SRC_DIRS  += $(ECMC)/plc
ecmc_SRCS += ecmcPLC.cpp
ecmc_SRCS += ecmcPLCTask.cpp
ecmc_SRCS += ecmcPLCDataIF.cpp
ecmc_SRCS += ecmcPLCMain.cpp
+ ecmc_SRCS += ecmcPLCLib.cpp
+ ecmc_SRCS += ecmcPLCLibFunc.cpp


SRC_DIRS  += $(ECMC)/misc
ecmc_SRCS += ecmcMisc.cpp
ecmc_SRCS += ecmcEvent.cpp
ecmc_SRCS += ecmcEventConsumer.cpp
ecmc_SRCS += ecmcDataRecorder.cpp
ecmc_SRCS += ecmcDataStorage.cpp
ecmc_SRCS += ecmcCommandList.cpp

SRC_DIRS  += $(ECMC)/main
ecmc_SRCS += ecmcGeneral.cpp
ecmc_SRCS += ecmcError.cpp
ecmc_SRCS += ecmcMainThread.cpp
ecmc_SRCS += gitversion.c
  

SRC_DIRS  += $(ECMC)/ethercat
ecmc_SRCS += ecmcEthercat.cpp
ecmc_SRCS += ecmcEc.cpp
+ ecmc_SRCS += ecmcEcData.cpp
+ ecmc_SRCS += ecmcEcDomain.cpp
ecmc_SRCS += ecmcEcEntry.cpp
ecmc_SRCS += ecmcEcPdo.cpp
ecmc_SRCS += ecmcEcSDO.cpp
ecmc_SRCS += ecmcEcSlave.cpp
ecmc_SRCS += ecmcEcSyncManager.cpp
ecmc_SRCS += ecmcEcEntryLink.cpp  
+ ecmc_SRCS += ecmcEcAsyncSDO.cpp
ecmc_SRCS += ecmcAsynLink.cpp
ecmc_SRCS += ecmcEcMemMap.cpp


SRC_DIRS  += $(ECMC)/com
DBD       += ecmcController.dbd
+ ecmc_SRCS += ecmcDataItem.cpp
ecmc_SRCS += ecmcCom.cpp
ecmc_SRCS += ecmcOctetIF.c
ecmc_SRCS += ecmcCmdParser.c
ecmc_SRCS += ecmcAsynPortDriver.cpp
ecmc_SRCS += ecmcAsynPortDriverUtils.cpp
ecmc_SRCS += ecmcAsynDataItem.cpp

SRC_DIRS  += $(ECMC)/motion
ecmc_SRCS += ecmcMotion.cpp
ecmc_SRCS += ecmcAxisBase.cpp
+ ecmc_SRCS += ecmcAxisGroup.cpp
ecmc_SRCS += ecmcAxisReal.cpp
ecmc_SRCS += ecmcAxisVirt.cpp
ecmc_SRCS += ecmcDriveBase.cpp
ecmc_SRCS += ecmcDriveStepper.cpp
ecmc_SRCS += ecmcDriveDS402.cpp
ecmc_SRCS += ecmcEncoder.cpp
ecmc_SRCS += ecmcFilter.cpp
ecmc_SRCS += ecmcMonitor.cpp
+ ecmc_SRCS += ecmcMotionUtils.cpp
ecmc_SRCS += ecmcPIDController.cpp
+ ecmc_SRCS += ecmcPVTController.cpp
ecmc_SRCS += ecmcAxisSequencer.cpp
+ ecmc_SRCS += ecmcAxisPVTSequence.cpp
+ ecmc_SRCS += ecmcTrajectoryBase.cpp
+ ecmc_SRCS += ecmcTrajectoryS.cpp
ecmc_SRCS += ecmcTrajectoryTrapetz.cpp
ecmc_SRCS += ecmcAxisData.cpp

SRC_DIRS  += $(ECMC)/motor
DBD       += ecmcMotorRecordSupport.dbd
ecmc_SRCS += ecmcMotorRecordController.cpp
ecmc_SRCS += ecmcMotorRecordAxis.cpp

+ SRC_DIRS  += $(ECMC)/plugin
+ ecmc_SRCS += ecmcPluginLib.cpp
+ ecmc_SRCS += ecmcPlugin.cpp
+ ecmc_SRCS += ecmcPluginClient.cpp

ecmc_LIBS += exprtkSupport
ecmc_LIBS += $(EPICS_BASE_IOC_LIBS)



include $(TOP)/configure/RULES
#----------------------------------------
#  ADD RULES AFTER THIS LINE

gitversion.c:
    @$(RM) $@
    @sh $(TOP)/tools/gitversion.sh $@
```

修改`devEcmcSup/motion/ecmcTrajectoryS.h`

``` diff { title="devEcmcSup/motion/ecmcTrajectoryS.h" }
#include "ecmcecmcTrajectoryBase.h"
- #include <ruckig.hpp>
+ #include <ruckig/ruckig.hpp>
```

修改`ecmcExampleTop/configure/RELEASE.local`

``` shell { title="ecmcExampleTop/configure/RELEASE.local" }
EPICS_BASE=/home/ubuntu/epics/base-7.0.8.1
SUPPORT=$(EPICS_BASE)/../epics-modules
ASYN=$(SUPPORT)/asyn
MOTOR=$(SUPPORT)/motor
# 添加QSRV2支持
PVXS=$(SUPPORT)/pvxs
# 指定交叉编译架构
EPICS_HOST_ARCH=linux-loong64
```

修改`ecmcExampleTop/ecmcIocApp/src/Makefile`

``` diff
TOP=../..

include $(TOP)/configure/CONFIG
#----------------------------------------
#  ADD MACRO DEFINITIONS AFTER THIS LINE
#=============================

#=============================
# Build the IOC application

PROD_IOC = ecmcIoc

# Ubuntu needs the following ldflags
USR_LDFLAGS_Linux += -Wl,--no-as-needed

+ ifeq ($(T_A),linux-x86_64)
+ # Assume that the etherlab user library is done via
+ # https://github.com/icshwi/etherlabmaster
+ USR_INCLUDES += -I/opt/etherlab/include
+ USR_CFLAGS += -fPIC
+ USR_LDFLAGS += -L /opt/etherlab/lib
+ USR_LDFLAGS += -lethercat
+ USR_LDFLAGS += -Wl,-rpath=/opt/etherlab/lib
+ else
+ # Assume that the etherlab user library is done via
+ # Yocto ESS Linux bb recipe
+ # 这里注意etherlab的路径
+ USR_INCLUDES += -I$(SDKTARGETSYSROOT)/usr/include/etherlab
+ USR_CFLAGS   += -fPIC
+ USR_LDFLAGS  += -L $(SDKTARGETSYSROOT)/usr/lib/etherlab
+ USR_LDFLAGS  += -lethercat
+ USR_LDFLAGS  += -Wl,-rpath=$(SDKTARGETSYSROOT)/usr/lib/etherlab
+ endif

+ # ruckig路径
+ USR_INCLUDES += -I/path/to/ruckig/include
+ USR_LDFLAGS  += -L /path/to/ruckig/build
+ USR_LDFLAGS  += -lruckig

# ecmcioc.dbd will be created and installed
DBD += ecmcIoc.dbd

# opcuaIoc.dbd will be made up from these files:
ecmcIoc_DBD += base.dbd
ecmcIoc_DBD += ecmcController.dbd

# Add all the support libraries needed by this IOC
ecmcIoc_LIBS += asyn
ecmcIoc_LIBS += ecmc
+ ecmcIoc_LIBS += motor
ecmcIoc_LIBS += exprtkSupport

+ ecmcIoc_DBD += asyn.dbd
+ ecmcIoc_DBD += motorSupport.dbd

ecmcIoc_SRCS += ecmcIoc_registerRecordDeviceDriver.cpp

# Build the main IOC entry point on workstation OSs.
ecmcIoc_SRCS_DEFAULT += ecmcIocMain.cpp
ecmcIoc_SRCS_vxWorks += -nil-

+ # Link PVXS if available
+ ifdef PVXS_MAJOR_VERSION
+     ecmcIoc_LIBS += pvxsIoc pvxs
+     ecmcIoc_DBD += pvxsIoc.dbd
+ else
+ # Link QSRV (pvAccess Server) if available
+ ifdef EPICS_QSRV_MAJOR_VERSION
+     ecmcIoc_LIBS += qsrv
+     ecmcIoc_LIBS += $(EPICS_BASE_PVA_CORE_LIBS)
+     ecmcIoc_DBD += PVAServerRegister.dbd
+     ecmcIoc_DBD += qsrv.dbd
+ endif
+ endif

# Finally link to the EPICS Base libraries
ecmcIoc_LIBS += $(EPICS_BASE_IOC_LIBS)

#===========================
include $(TOP)/configure/RULES
#----------------------------------------
#  ADD RULES AFTER THIS LINE
```

编译的时候需要指定使用`c++17`的标准，不然有一些语法不支持，应该是`ruckig`库比较新。

添加`CPPFLAGS=-std=c++17`

``` shell
# 执行交叉编译
make CPPFLAGS=-std=c++17 \
LD=loongarch64-linux-gnu-ld \
CC=loongarch64-linux-gnu-gcc \
CCC=loongarch64-linux-gnu-g++ -j4
```

## 使用方法

主要使用`ecmccfg`软件包提供的预编写脚本进行`EtherCAT`主站、从站配置。

修改IOC环境变量配置`iocBoot/iocExample/envPaths`：
``` shell { title="envPaths" }
epicsEnvSet("IOC", "iocExample")
epicsEnvSet("TOP", "../..")
epicsEnvSet("SUPPORT", "/root/epics-modules")
epicsEnvSet("ASYN", "${SUPPORT}/asyn")
epicsEnvSet("ECMC", "${SUPPORT}/ecmc")
```

修改启动脚本`iocBoot/iocExample/st.cmd`：
``` shell { title="st.cmd" }
#!../../bin/linux-loong64/ecmcIoc

< envPaths

cd "${TOP}"

## Register all support components
dbLoadDatabase "dbd/ecmcIoc.dbd"
ecmcIoc_registerRecordDeviceDriver pdbbase

## Load record instances
#dbLoadRecords("db/xxx.db","user=${USER}")

# ecmccfg路径配置
epicsEnvSet "ecmccfg_DIR", "${ECMC}/scripts/"
epicsEnvSet "ecmccfg_DB", "${ECMC}/db/"
epicsEnvSet "ECMC_CONFIG_ROOT", "${ecmccfg_DIR}"
epicsEnvSet "ECMC_CONFIG_DB", "${ecmccfg_DB}"
epicsEnvSet "SCRIPTEXEC", "iocshLoad"

#- 初始化
${SCRIPTEXEC} "${ECMC_CONFIG_ROOT}initAll.cmd", "SM_MOTOR_PORT=MCU1,SM_ASYN_PORT=MC_CPU1,SM_PREFIX=${IOC}:"
# EtherCAT主站号
epicsEnvSet "MASTER_ID", 0
# 扫描频率，默认 1000
epicsEnvSet "EC_RATE", 100

#-
#-------------------------------------------------------------------------------
# Set EtherCAT frequency (defaults to 1000)
ecmcConfigOrDie "Cfg.SetSampleRate(${EC_RATE=1000})"
#-
#- Set current EtherCAT sample rate
#- Note: Not the same as ECMC_SAMPLE_RATE_MS which is for record update
epicsEnvSet "ECMC_EC_SAMPLE_RATE", ${EC_RATE=1000}
ecmcEpicsEnvSetCalc("ECMC_EC_SAMPLE_RATE_MS", 1000/${ECMC_EC_SAMPLE_RATE=1000})

# Update records in 10ms (100Hz) for FULL MODE and in EC_RATE for DAQ mode
ecmcEpicsEnvSetCalcTernary(ECMC_SAMPLE_RATE_MS, "'${ECMC_MODE=FULL}'=='DAQ'","${ECMC_EC_SAMPLE_RATE_MS}","10")
epicsEnvSet(ECMC_SAMPLE_RATE_MS_ORIGINAL, ${ECMC_SAMPLE_RATE_MS})

#- define naming convention script
#epicsEnvSet "ECMC_P_SCRIPT", "${NAMING=mXsXXX}"

#- Set master
${SCRIPTEXEC} "${ECMC_CONFIG_ROOT}addMaster.cmd", "MASTER_ID=${MASTER_ID=0}"
epicsEnvSet "ECMC_EC_MASTER_ID", ${MASTER_ID=0}

#- Add slaves
${SCRIPTEXEC} "${ECMC_CONFIG_ROOT}addSlave.cmd", "SLAVE_ID=0, HW_DESC=EK1100"
${SCRIPTEXEC} "${ECMC_CONFIG_ROOT}addSlave.cmd", "SLAVE_ID=1, HW_DESC=EL4024"
${SCRIPTEXEC} "${ECMC_CONFIG_ROOT}addSlave.cmd", "SLAVE_ID=2, HW_DESC=EL2624"
${SCRIPTEXEC} "${ECMC_CONFIG_ROOT}addSlave.cmd", "SLAVE_ID=3, HW_DESC=EL3742"
${SCRIPTEXEC} "${ECMC_CONFIG_ROOT}addSlave.cmd", "SLAVE_ID=4, HW_DESC=EK1110"

#- Apply hardware configuration
${SCRIPTEXEC} "${ECMC_CONFIG_ROOT}applyConfig.cmd"
#- Activate
${SCRIPTEXEC} "${ECMC_CONFIG_ROOT}setAppMode.cmd"

cd "${TOP}/iocBoot/${IOC}"
iocInit

## Start any sequence programs
#seq sncxxx,"user=${USER}"
```

* `ecmcXXxxxx.cmd`：从站配置脚本
* `ecmcXXxxxx.substitutions`：记录(records)

*参考链接*
* [ESS EPICS Environment (e3) — ESS EPICS Environment (e3) documentation](https://e3pages.readthedocs.io/en/latest/index.html)
* [ecmccfg 手册](https://paulscherrerinstitute.github.io/ecmccfg/)