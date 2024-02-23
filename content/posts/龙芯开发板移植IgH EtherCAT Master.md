---
title: "龙芯开发板移植 IgH EtherCAT Master"
date: 2024-02-23T12:54:31+08:00
draft: true
description: 龙芯2K0500开发板移植EtherCAT主站程序和EPICS EtherCAT模块
tags: ["linux", "EPICS", "龙芯"]
keywords: ["linux", "EPICS", "EtherCAT", "龙芯"]
categories: ["EPICS"]
---

## 前言

IgH EtherCAT Master是一个开源的EtherCAT主站驱动程序，用于管理EtherCAT从站设备，支持Linux操作系统，工控上使用的比较多。

dls ethercat是由英国钻石光源开发的用于 EPICS 控制系统 EtherCAT 设备的支持程序，基于 IgH Master 主站程序开发，实现对 EtherCAT 总线设备的读写。

交叉编译环境：Ubuntu

运行开发板：龙芯2K0500金龙开发板

内核版本：5.10.0-lsgd

## 相关软件包下载地址

- [epics-base - (launchpad.net)](https://git.launchpad.net/epics-base) / [epics-base/epics-base](https://github.com/epics-base/epics-base) / [EPICS Base (anl.gov)](https://epics.anl.gov/base/index.php)

- [epics-modules/asyn: EPICS module for driver and device support](https://github.com/epics-modules/asyn)

- [epics-modules/busy: APS BCDA synApps module: busy](https://github.com/epics-modules/busy)

- [epics-modules/autosave: APS BCDA synApps module: autosave](https://github.com/epics-modules/autosave)

- [dls-controls/ethercat: EPICS support to read/write to ethercat based hardware](https://github.com/dls-controls/ethercat)

- [IgH EtherCAT Master for Linux](https://gitlab.com/etherlab.org/ethercat/-/tree/stable-1.5)

## 配置交叉编译环境

关于这一节，之前的文章已经详细讲过，参考[配置交叉编译环境](../龙芯2k500开发板上实现的呼吸灯效果/#配置交叉编译环境)。

如果你使用的是其他开发套件，请按照开发手册安装配置好环境。

## 编译 IgH EtherCAT Master

### 打补丁

``` shell
# 先将补丁复制到ethercat驱动源码下
cp ethercat-master/etc/makeDocumentation/configurable-error-suppression.patch ethercat-stable-1.5/

cd ethercat-stable-1.5/
# 打补丁
patch -p1 < configurable-error-suppression.patch
# 这里需要注意一下patch的输出结果
```

由于补丁的时间比较早，跟现有的源码不太匹配，有些地方补丁会失败（FAILED）。比如我这里的报错，可能需要再手动修改一下。

``` shell
patching file lib/Makefile.am
Hunk #1 succeeded at 32 (offset -2 lines).
patching file lib/common.c
patching file lib/domain.c
patching file lib/liberror-documentation.txt
patching file lib/liberror.c
patching file lib/liberror.h
patching file lib/master.c
Hunk #1 succeeded at 37 (offset -2 lines).
...
Hunk #12 FAILED at 419.
Hunk #13 FAILED at 448.
...
2 out of 31 hunks FAILED -- saving rejects to file lib/master.c.rej
patching file lib/reg_request.c
Hunk #1 succeeded at 39 (offset -2 lines).
...
patching file lib/sdo_request.c
Hunk #1 succeeded at 39 (offset -2 lines).
...
patching file lib/slave_config.c
Hunk #1 succeeded at 39 (offset -1 lines).
...
patching file lib/voe_handler.c
Hunk #1 succeeded at 40 (offset -2 lines).
...
```

我们参考`lib/master.c.rej`，手动修改一下。

``` diff
# lib/master.c.rej
--- lib/master.c	Tue Feb 12 17:31:08 2013 +0100
+++ lib/master.c	Mon Mar 23 15:52:53 2015 +0000
@@ -419,7 +431,8 @@
         if (EC_IOCTL_ERRNO(ret) == EIO && abort_code) {
             *abort_code = download.abort_code;
         }
-        fprintf(stderr, "Failed to execute SDO download: %s\n",
+        ecrt_errcode = ECRT_ERRMASTERSDODOWNLOAD;
+        ERRPRINTF("Failed to execute SDO download: %s\n",
             strerror(EC_IOCTL_ERRNO(ret)));
         return -EC_IOCTL_ERRNO(ret);
     }
@@ -448,7 +461,8 @@
         if (EC_IOCTL_ERRNO(ret) == EIO && abort_code) {
             *abort_code = download.abort_code;
         }
-        fprintf(stderr, "Failed to execute SDO download: %s\n",
+        ecrt_errcode = ECRT_ERRMASTERSDODOWNLOADCOMPLETE;
+        ERRPRINTF("Failed to execute SDO download: %s\n",
             strerror(EC_IOCTL_ERRNO(ret)));
         return -EC_IOCTL_ERRNO(ret);
     }
```

然后这个补丁还有点问题，我们需要手动修改一下`lib/liberror.c`。这里源文件和头文件变量定义不一致，编译会报错，以头文件为准。

``` c
#include "liberror.h"

int ecrt_err_to_stderr = 1;
// char *ecrt_errstring[ERRSTRING_LEN];
char ecrt_errstring[ERRSTRING_LEN];

int ecrt_errcode;
```

### 准备内核源码

由于需要将源码编译成相应的系统驱动，所以这里需要使用内核的源码。

如果是标准系统，可以直接安装`linux-headers`。

``` shell
# Ubuntu/Debian
sudo apt install linux-headers-$(uname -r)
```

树莓派系统

``` shell
sudo apt install raspberrypi-kernel-headers
```

**而对于开发板系统，我们可以使用随开发板提供的内核源码。**

*内核的编译步骤请根据开发板的用户手册完成。*

``` shell
# 解压内核源码
tar -xvf linux-5.10-2k500-src-f45937d-build.20230721100738.tar.gz
# linux-5.10-2k500-cbd-src
cd linux-5.10-2k500-cbd-src/
# 配置交叉编译器
./set_env.sh

# 编译内核
make loongson_2k500_defconfig
# make menuconfig
make uImage

# 编译模块
make modules
```

### 编译 EtherCAT Master 驱动

**源码一定要下载stable-1.5分支的，其他版本我也没有测试！**

``` shell
cd ethercat-stable-1.5/
# to create the configure script, if downloaded from the repo
./bootstrap
# 这里需要注意是否出现报错，需要安装 autoconf、pkg-config 等工具

# 执行configure
# --host 指定程序运行的主机架构
# --with-linux-dir 指定源码目录
# 如果是安装的linux-headers，通常在 /usr/src/linux-headers-xxx
# 如果直接使用内核源码，则必须通过上述编译步骤！
# --prefix 指定安装目录
./configure --host=loongarch64-linux-gnu CC=loongarch64-linux-gnu-gcc --enable-generic=yes --enable-8139too=no --with-linux-dir=/path/to/linux-5.10-2k500-cbd-src --prefix=/path/to/__install_dir

# 编译
make all modules
# 或者
# make
# make ARCH=loongarch CORSS_COMPILE=loongarch64-linux-gnu- modules

# 安装生成的文件
# make install
```

如果完全按照上述步骤，应该可以编译成功。

下面整理一下生成的文件。

``` shell
# 复制生成的主程序
cp tool/ethercat /path/to/__install_dir/bin/
# 复制生成的驱动程序
cp device/ec_generic.ko /path/to/__install_dir/modules/
cp master/ec_master.ko /path/to/__install_dir/modules/
```

这是我整理的文件，后面需要将这些文件下载到开发板。

```
__install_dir
├─ bin
│   └─ ethercat (主程序)
├─ etc (配置)
├─ include (头文件)
├─ lib (libethercat.so)
├─ modules (驱动目录)
│   ├─ ec_master.ko
│   └─ ec_generic.ko
├─ sbin
│   └─ ethercatctl
└─ share
    └─ bash-completion/ (bash自动补全)
```

## 编译 EPICS ethercat 模块

**以下步骤需要先安装好EPICS Base!**

### 编译 asyn

``` shell
cd asyn
touch configure/RELEASE.local
vi configure/RELEASE.local

# 修改成和EPICS Base一样的架构
EPICS_HOST_ARCH=linux-la64
# EPICS Base路径（示例）
EPICS_BASE=/home/ubuntu/loongson/base-7.0.8
# 放置EPICS模块的路径（示例）
SUPPORT=/home/ubuntu/loongson/modules
# SSCAN模块路径
# SSCAN=$(SUPPORT)/sscan
# CALC模块路径
# CALC=$(SUPPORT)/calc

# 直接编译
# make
# 交叉编译（示例）
make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
```

### 编译 autosave

``` shell
cd autosave
touch configure/RELEASE.local
vi configure/RELEASE.local

# 修改成和EPICS Base一样的架构
EPICS_HOST_ARCH=linux-la64
# EPICS Base路径（示例）
EPICS_BASE=/home/ubuntu/loongson/base-7.0.8
# 放置EPICS模块的路径（示例）
SUPPORT=/home/ubuntu/loongson/modules

# 直接编译
# make
# 交叉编译（示例）
make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
```

### 编译 busy

``` shell
cd busy
touch configure/RELEASE.local
vi configure/RELEASE.local

# 修改成和EPICS Base一样的架构
EPICS_HOST_ARCH=linux-la64
# EPICS Base路径（示例）
EPICS_BASE=/home/ubuntu/loongson/base-7.0.8
# 放置EPICS模块的路径（示例）
SUPPORT=/home/ubuntu/loongson/modules
# ASYN模块路径
ASYN=$(SUPPORT)/asyn
# AUTOSAVE模块路径
AUTOSAVE=$(SUPPORT)/autosave
# BUSY模块路径
BUSY=$(SUPPORT)/busy

# 直接编译
# make
# 交叉编译（示例）
make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
```

### 编译 ethercat

这一步可以说是最麻烦，问题最多的。编译出什么问题都需要去找到相应的`Makefile`修改。

首先需要安装所需的包。但我们实际上并不需要用这个软件包，我们只需要它的头文件。

相应的动态库(.so)文件，我们则需要从开发板系统中拷贝出来，无法直接用编译电脑的动态库。

``` shell
cd ethercat-master/

# 创建 3rd 目录，用于放置所需的头文件和动态库
mkdir 3rd
mkdir 3rd/include
mkdir 3rd/lib

# 安装 libxml2-dev
sudo apt install libxml2-dev
# 从系统目录中复制出libxml2的头文件
# 因为还需要做一些修改，不能直接使用
cp -r /usr/include/libxml2/ ./3rd/include/

# 复制刚刚编译生成的 libethercat
cp /path/to/__install_dir/lib/libethercat.so* ./3rd/lib/
# 从开发板系统复制所需要的动态库
# libxml2 依赖 libz 和 liblzma
scp root@192.168.1.10:/usr/lib/libxml2.so.2.9.12 ./3rd/lib/
scp root@192.168.1.10:/usr/lib/libz.so.1.2.11 ./3rd/lib/
scp root@192.168.1.10:/usr/lib/liblzma.so.5.2.5 ./3rd/lib/
# 手动创建一下链接
cd 3rd/lib/
ln -s libxml2.so.2.9.12 libxml2.so.2
ln -s libxml2.so.2 libxml2.so
ln -s liblzma.so.5.2.5 liblzma.so.5
ln -s liblzma.so.5 liblzma.so
ln -s libz.so.1.2.11 libz.so.1
ln -s libz.so.1 libz.so
```

然后需要修改一下`libxml2`的头文件，不然编译的时候会报错。

报错信息：

``` shell
In file included from ../../../libxml2/libxml/parser.h:812,
                 from ../../../libxml2/libxml/globals.h:18,
                 from ../../../libxml2/libxml/threads.h:35,
                 from ../../../libxml2/libxml/xmlmemory.h:218,
                 from ../../../libxml2/libxml/tree.h:1307,
                 from ../parser.c:10:
../../../libxml2/libxml/encoding.h:31:10: fatal error: unicode/ucnv.h: No such file or directory
 #include <unicode/ucnv.h>
          ^~~~~~~~~~~~~~~~
compilation terminated.
```

解决方法：

``` shell
# 修改 xmlversion.h
vi 3rd/include/libxml2/libxml/xmlversion.h

# 找到下面行，禁用 LIBXML_ICU_ENABLED
#define LIBXML_ICU_ENABLED
# 将 #if 1 改为 #if 0
```

``` c
/**
 * LIBXML_ICU_ENABLED:
 *
 * Whether icu support is available
 */
#if 0
#define LIBXML_ICU_ENABLED
#endif
```

做好上面的准备工作，还需要修改源码中的路径配置，然后才能正常编译。  
下面都是我踩坑留下的记录。

1. 修改 configure/RELEASE

``` shell
cd ethercat-master/

# 首先修改 configure/RELEASE
vi configure/RELEASE

# 需要修改的有4项
# 放置EPICS模块的路径（示例）
SUPPORT=/home/ubuntu/loongson/modules
# ASYN模块路径
ASYN=$(SUPPORT)/asyn
# BUSY模块路径
BUSY=$(SUPPORT)/busy

# 修改成和EPICS Base一样的架构
EPICS_HOST_ARCH=linux-la64
# EPICS Base路径（示例）
EPICS_BASE=/home/ubuntu/loongson/base-7.0.8
```

2. 修改 ethercatApp/scannerSrc/Makefile

``` shell
cd ethercat-master/
# 修改 ethercatApp/scannerSrc/Makefile
vi ethercatApp/scannerSrc/Makefile

# 需要修改 EtherCAT Master 源码相关路径、
# libxml2头文件路径、动态库(.so)路径

# 修改 ETHERLAB 源码路径
ETHERLAB=/path/to/ethercat-stable-1.5
ETHERLABPREFIX=$(ETHERLAB)

USR_INCLUDES += -I$(ETHERLABPREFIX)/include

# 修改动态库路径
USR_LDFLAGS += -L$(TOP)/3rd/lib -Wl,-rpath=$(TOP)/3rd/lib
# 修改 libxml2 头文件路径
USR_INCLUDES += -I$(TOP)/3rd/include/libxml2
USR_SYS_LIBS += ethercat xml2

# 下面类似
scanner_INCLUDES += -I$(ETHERLAB)/lib

serialtool_INCLUDES += -I$(ETHERLAB)/master

get-slave-revisions_INCLUDES += -I$(ETHERLAB)/master
```

3. 修改 ethercatApp/src/Makefile，与上面类似。

``` shell
cd ethercat-master/
# 修改 ethercatApp/src/Makefile
vi ethercatApp/src/Makefile

# 需要修改 EtherCAT Master 源码相关路径、
# libxml2头文件路径、动态库(.so)路径

# 修改 ETHERLAB 源码路径
ETHERLAB=/path/to/ethercat-stable-1.5
# ecAsyn_INCLUDES += -I$(ETHERLAB)/src/ethercat-$(subst -,.,$(VERSION))/include
# gadc_INCLUDES += -I$(ETHERLAB)/src/ethercat-$(subst -,.,$(VERSION))/include
ecAsyn_INCLUDES += -I$(ETHERLAB)/include
gadc_INCLUDES += -I$(ETHERLAB)/include

# 修改 libxml2 头文件路径
USR_INCLUDES += -I$(TOP)/3rd/include/libxml2

# 添加动态库路径
USR_LDFLAGS += -L$(TOP)/3rd/lib -Wl,-rpath=$(TOP)/3rd/lib
USR_SYS_LIBS += xml2
```

4. 编译

最后终于可以开始编译了。

``` shell
cd ethercat-master/
# 执行交叉编译
make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
```

到这里，编译过程还可能会出错，不过已经可以编译出我们所需要的东西了。  
最终编译得到`scanner`程序就可以了。

以下是我编译时的输出：

``` shell
...
Installing library ../../../lib/linux-la64/libscannerlib.a
...
Installing created executable ../../../bin/linux-la64/scanner
Installing created executable ../../../bin/linux-la64/parsertest
...
Installing shared library ../../../lib/linux-la64/libecAsyn.so
Installing library ../../../lib/linux-la64/libecAsyn.a
...
Installing template file ../../../db/EK1100.template
...
make -C ./protocol install
make[2]: 进入目录“/home/deepin/ethercat-master/ethercatApp/protocol”
make[2]: *** 没有规则可制作目标“install”。 停止。
make[2]: 离开目录“/home/deepin/ethercat-master/ethercatApp/protocol”
make[1]: *** [/usr/local/epics/base-7.0.8/configure/RULES_DIRS:85：protocol.install] 错误 2
make[1]: 离开目录“/home/deepin/ethercat-master/ethercatApp”
make: *** [/usr/local/epics/base-7.0.8/configure/RULES_DIRS:85：ethercatApp.install] 错误 2
```

最后的错误，我直接忽略了，因为已经得到了需要的可执行程序。

整理一下编译生成的文件：

```
ethercat-master
├─ bin
│   └─ linux-la64
├─ db
├─ dbd
├─ lib
    └─ linux-la64
```

## 测试运行

将整理好的文件下载到开发板后，我们测试运行一下。

测试安装 EtherCAT 主站驱动程序。

``` shell
[root@LS-GD modules]# modinfo ec_generic.ko
filename:       /root/__install/modules/ec_generic.ko
version:        1.5.2 unknown
license:        GPL
description:    EtherCAT master generic Ethernet device module
author:         Florian Pose <fp@igh-essen.com>
srcversion:     848BB80F1C588A2FDA42EDB
depends:        ec_master
name:           ec_generic
vermagic:       5.10.0.lsgd preempt mod_unload modversions LOONGARCH 64BIT
[root@LS-GD modules]# insmod ec_master.ko
[root@LS-GD modules]# insmod ec_generic.ko
[root@LS-GD modules]# lsmod
Module                  Size  Used by
ec_generic              6427  0
ec_master             464125  1 ec_generic
```

测试`ethercat`主程序。

``` shell
[root@LS-GD tool]# ./ethercat
Please specify a command!

Usage: ethercat <COMMAND> [OPTIONS] [ARGUMENTS]

Commands (can be abbreviated):
  alias      Write alias addresses.
  config     Show slave configurations.
  crc        CRC error register diagnosis.
  cstruct    Generate slave PDO information in C language.
  data       Output binary domain process data.
  debug      Set the master's debug level.
  domains    Show configured domains.
  download   Write an SDO entry to a slave.
  eoe        Display Ethernet over EtherCAT statictics.
  foe_read   Read a file from a slave via FoE.
  foe_write  Store a file on a slave via FoE.
  graph      Output the bus topology as a graph.
  master     Show master and Ethernet device information.
  pdos       List Sync managers, PDO assignment and mapping.
  reg_read   Output a slave's register contents.
  reg_write  Write data to a slave's registers.
  rescan     Rescan the bus.
  sdos       List SDO dictionaries.
  sii_read   Output a slave's SII contents.
  sii_write  Write SII contents to a slave.
  slaves     Display slaves on the bus.
  soe_read   Read an SoE IDN from a slave.
  soe_write  Write an SoE IDN to a slave.
  states     Request application-layer states.
  upload     Read an SDO entry from a slave.
  version    Show version information.
  xml        Generate slave information XML.

Global options:
  --master  -m <master>  Comma separated list of masters
                         to select, ranges are allowed.
                         Examples: '1,3', '5-7,9', '-3'.
                         Default: '-' (all).
  --force   -f           Force a command.
  --quiet   -q           Output less information.
  --verbose -v           Output more information.
  --help    -h           Show this help.

Numerical values can be specified either with decimal (no
prefix), octal (prefix '0') or hexadecimal (prefix '0x') base.

Call 'ethercat <COMMAND> --help' for command-specific help.

Send bug reports to fp@igh.de.
```

测试`scanner`主程序。

``` shell
[root@LS-GD linux-la64]# chmod +x scanner
[root@LS-GD linux-la64]# ./scanner
usage: scanner [-m master_index] [-s] [-q] scanner.xml socket_path
```

可以看到，驱动程序和主程序都能在开发板上运行，说明已经编译完成了。至于具体如何使用，下次再讲吧。

## 参考

- [IgH EtherCAT Master](https://docs.etherlab.org/ethercat/1.5/doxygen/index.html)
- [Building and installing IgH EtherCAT Master](https://gitlab.com/etherlab.org/ethercat/-/blob/master/INSTALL.md)
- [IgH EtherCAT Master for Linux](https://github.com/landaurobotics/igh-ethercat-master/blob/main/README.md)
- [EtherCAT DRIVER AND TOOLS FOR EPICS AND LINUX AT PSI](https://accelconf.web.cern.ch/pcapac2018/papers/wep01.pdf)
- [INTEGRATION OF EtherCAT HARDWARE INTO THE EPICS BASED DISTRIBUTED CONTROL SYSTEM AT iThemba LABS](https://accelconf.web.cern.ch/cyclotrons2019/papers/tup004.pdf)
- [在“福珑2.0”主机上编译EPICS Ehtercat驱动软件的体验](https://blog.csdn.net/honeymelon3/article/details/113175863)
