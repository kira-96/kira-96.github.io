---
title: "EPICS IOC 使用 ProcServControl"
date: 2025-06-03T09:58:47+08:00
draft: true
description: EPICS IOC 使用 ProcServControl 过程记录
tags: ["linux", "EPICS"]
keywords: ["linux", "EPICS"]
categories: ["EPICS"]
---

## 软件包
* [epics-modules/asyn: EPICS module for driver and device support](https://github.com/epics-modules/asyn)
* [epics-modules/busy: APS BCDA synApps module](https://github.com/epics-modules/busy)
* [epics-modules/sequencer: The EPICS SNL Compiler and Sequencer](https://github.com/epics-modules/sequencer)
* [DiamondLightSource/procServControl: EPICS/CA control of running procServ instances](https://github.com/DiamondLightSource/procServControl)
* [ralphlange/procServ: Wrapper to start arbitrary interactive commands in the background, with telnet or Unix domain socket access to stdin/stdout](https://github.com/ralphlange/procServ)

## 编译软件包

### asyn

正常交叉编译。

### busy

> [!tip] 提示
> 编译**busy**模块时需要用到**autosave**，如果不需要编译**busy**测试程序，可以注释`configure/RELEASE`中的`AUTOSAVE`定义。

正常交叉编译。

### sequencer

> [!NOTE] Title
> 编译**sequencer**模块需要用到**re2c**，可通过包管理器安装。
> `apt install re2c`

正常交叉编译。

### procServControl

1. 修改`configure/RELEASE`：
``` diff { title="configure/RELEASE" }
index 244da04..98d30ca 100644
--- a/configure/RELEASE
+++ b/configure/RELEASE
@@ -20,15 +20,15 @@
 # CONFIG_SITE file.

 TEMPLATE_TOP=$(EPICS_BASE)/templates/makeBaseApp/top
-SUPPORT=/dls_sw/prod/R3.14.12.7/support
+SUPPORT=/home/kira/epics/epics-modules

 # If using the sequencer, point SNCSEQ at its top directory:
-SNCSEQ=$(SUPPORT)/seq/2-2-5dls1
-ASYN=$(SUPPORT)/asyn/4-41
-BUSY=$(SUPPORT)/busy/1-7-2dls5
+SNCSEQ=$(SUPPORT)/sequencer
+ASYN=$(SUPPORT)/asyn
+BUSY=$(SUPPORT)/busy

 # EPICS_BASE usually appears last so other apps can override stuff:
-EPICS_BASE=/dls_sw/epics/R3.14.12.7/base
+EPICS_BASE=/home/kira/epics/base-7.0.9

 # Set RULES here if you want to take build rules from somewhere
 # other than EPICS_BASE:
```

2. 如果需要交叉编译，修改`configure/CONFIG_SITE`：
``` diff { title="configure/CONFIG_SITE" }
index f30c124..53c5e23 100644
--- a/configure/CONFIG_SITE
+++ b/configure/CONFIG_SITE
@@ -19,7 +19,7 @@ CHECK_RELEASE = YES
 # Set this when you only want to compile this application
 #   for a subset of the cross-compiled target architectures
 #   that Base is built for.
-CROSS_COMPILER_TARGET_ARCHS =
+CROSS_COMPILER_TARGET_ARCHS = linux-loong64

 # To install files into a location other than $(TOP) define
 #   INSTALL_LOCATION here.
```

3. 编译
``` shell
make -j
```

### procServ

#### 构建

``` shell
autoreconf -fi
# 配置交叉编译器
./configure --host=loongarch64-linux-gnu CC=loongarch64-linux-gnu-gcc --disable-doc
# 编译
make -j
```

编译完成应该可以看到当前目录生成了`procServ`可执行程序。

#### 使用EPICS构建系统

``` shell
mkdir procServ && cd procServ
EPICS_HOST_ARCH=`${EPICS_BASE}/startup/EpicsHostArch`
${EPICS_BASE}/bin/${EPICS_HOST_ARCH}/makeBaseApp.pl -t example dummy
rm -rf dummyApp

git clone https://github.com/ralphlange/procServ.git procServApp

cat > configure/RELEASE.local << EOF
EPICS_BASE=${EPICS_BASE}
EOF

cd procServApp

# 修改源码
# 具体见下面 **修改procServ源码**

make
./configure --with-epics-top=.. --disable-doc
# 编译
make -j
```

编译完成可以看到`procServ/bin/${EPICS_HOST_ARCH}`目录生成了`procServ`可执行程序。

**修改procServ源码**

1. 修改`procServ.cc`：

``` diff { title="procServ.cc" }
index c8d04a2..385f788 100644
--- a/procServ.cc
+++ b/procServ.cc
@@ -65,7 +65,7 @@ int    connectionNo;             // Total number of connections
 char   *ignChars = NULL;         // Characters to ignore
 char   killChar = 0x18;          // Kill command character (default: ^X)
 char   toggleRestartChar = 0x14; // Toggle autorestart character (default: ^T)
-char   restartChar = 0x12;       // Restart character (default: ^R)
+char   restartChar = 0x18;       // Restart character (default: ^X)
 char   quitChar = 0x11;          // Quit character (default: ^Q)
 char   logoutChar = 0x00;        // Logout client connection character (default: none)
 int    killSig = SIGKILL;        // Kill signal (default: SIGKILL)
```

2. 修改`Makefile.Epics.in`：

``` diff { title="Makefile.Epics.in" }
index ef6073c..85336f9 100644
--- a/Makefile.Epics.in
+++ b/Makefile.Epics.in
@@ -17,6 +17,7 @@ procServ_OBJS = @LIBOBJS@

 USR_CXXFLAGS += @DEFS@
 procServ_SYS_LIBS += $(subst -l,,@LIBS@)
+procServ_SYS_LIBS += util

 include $(TOP)/configure/RULES
```

## 在 IOC 中使用 procServControl

1. 修改`configure/RELEASE`：

``` diff { title="configure/RELEASE" }
--- a/configure/RELEASE
+++ b/configure/RELEASE
@@ -29,9 +29,12 @@ SUPPORT=$(EPICS_BASE)/../epics-modules
 # If using the sequencer, point SNCSEQ at its top directory:
+SNCSEQ = $(SUPPORT)/seq
+ASYN=$(SUPPORT)/asyn
+BUSY=$(SUPPORT)/busy
+PROCSERVCTRL=$(SUPPORT)/procServControl

 # EPICS_BASE should appear last so earlier modules can override stuff:
 EPICS_BASE = /path/to/epics/base
```
2. 修改`exampleApp/src/Makefile`：

``` diff { title="exampleApp/src/Makefile" }
--- a/exampleApp/src/Makefile
+++ b/exampleApp/src/Makefile
@@ -109,12 +109,21 @@ DB_INSTALLS += $(IOCADMIN)/db/iocAdminScanMon.db
 IOCRELEASE_DB += iocRelease.db
 endif

+# procServControl
+ifneq ($(PROCSERVCTRL),)
+example_DBD += asyn.dbd
+example_DBD += drvAsynIPPort.dbd
+example_DBD += busySupport.dbd
+example_DBD += procServControl.dbd
+example_LIBS += asyn busy seq pv procServControl
+endif
+
 # Finally link IOC to the EPICS Base libraries
 sysStats_LIBS += $(EPICS_BASE_IOC_LIBS)
```
3. 修改`exampleApp/Db/Makefile`：

``` diff { title="exampleApp/Db/Makefile" }
--- a/exampleApp/Db/Makefile
+++ b/exampleApp/Db/Makefile
@@ -6,6 +6,7 @@ include $(TOP)/configure/CONFIG
 # Install databases, templates & substitutions like this
 DB += exampleVersion.db

+DB_INSTALLS += $(PROCSERVCTRL)/db/procServControl.template

# If <anyname>.db template is not named <anyname>*.template add
# <anyname>_TEMPLATE = <templatename>
```

4. 修改`iocBoot/iocexample/st.cmd`：

``` diff { title="iocBoot/iocexample/st.cmd" }
--- a/iocBoot/iocexample/st.cmd
+++ b/iocBoot/iocexample/st.cmd
@@ -12,11 +12,16 @@ dbLoadDatabase "dbd/example.dbd"
 sysStats_registerRecordDeviceDriver pdbbase

+## Connect to procServ
+epicsEnvSet("ASYN_PORT_NAME", "port1")
+drvAsynIPPortConfigure("${ASYN_PORT_NAME}", "localhost:7001", 100, 0, 0)
+
 ## Load record instances
 # dbLoadTemplate "db/user.substitutions"
 # dbLoadRecords "db/dbSubExample.db", "user=${USER}"
 dbLoadRecords "db/exampleVersion.db", "user=${NAME}"
 # dbLoadRecords "db/iocAdminSoft.db", "IOC=${NAME}"
+dbLoadRecords "db/procServControl.template", "P=${NAME},PORT=${ASYN_PORT_NAME}"

 #- Set this to see messages from mySub
 #-var mySubDebug 1
@@ -29,3 +34,4 @@ iocInit

 ## Start any sequence programs
 #seq sncExample, "user=${USER}"
+seq procServControl, "P=${NAME}"
```

5. 编译

``` shell
make -j
```

## 使用 procServ 运行 IOC

``` shell
cd /path/to/example/iocBoot/iocexample
procServ -n $NAME -P 7001 ./st.cmd
```

> [!important] 注意
> **procServ**指定的端口（例：7001）必需和**st.cmd**脚本中的端口对应。

> [!NOTE] The most important PVs provided are:
> * $(P):START - Start IOC
> * $(P):STOP - Stop IOC
> * $(P):RESTART - Restart IOC
> * $(P):IOCOUT - Last 20 lines of IOC output

![监控&操作页面](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/PixPin_2025-06-03_13-44-09.png)
