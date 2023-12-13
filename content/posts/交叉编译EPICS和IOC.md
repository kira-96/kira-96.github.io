---
title: "交叉编译EPICS和IOC"
date: 2023-12-12T16:26:35+08:00
draft: false
description: Linux交叉编译EPICS
tags: ["linux", "EPICS", "龙芯"]
keywords: ["linux", "EPICS", "龙芯"]
categories: ["EPICS"]
---

## 前言

之前已经讲过[在龙芯3A5000(loongarch64)上编译运行EPICS](../龙芯3a5000loongarch64上编译运行epics/)，不过这种情况只适用于有完整开发环境的情况下进行编译。一些时候，我们只有编译器，而缺少make，perl等工具，比如一些开发板厂商提供的开发套件。这种情况下，就需要通过交叉编译(cross-compiling)的方式来编译EPICS。

这里以**龙芯金龙2K500先锋开发板**为例，我们使用Ubuntu-20.04作为构建系统，详细讲解如何构建出可以在开发板上运行的EPICS工具包，并部署在开发板上。

由于开发板上没有开发环境，即使编译出目标平台的EPICS Base，我们依然不能直接在开发板上创建和编译IOC。所以，我们还是使用Ubuntu-20.04作为构建系统，创建并编译IOC，最后在开发板上运行。

## 配置交叉编译环境

关于这一节，之前的文章已经详细讲过，参考[配置交叉编译环境](../龙芯2k500开发板上实现的呼吸灯效果/#配置交叉编译环境)。

如果你使用的是其他开发套件，请按照开发手册安装配置好环境。

## 编译 EPICS Base

首先，下载、解压Base，参考[以前的文章](../龙芯3a5000loongarch64上编译运行epics/)。

在[龙芯3A5000(loongarch64)上编译运行EPICS](../龙芯3a5000loongarch64上编译运行epics/)中我已经详细讲解了如何在龙架构上编译EPICS，这次，需要在原来对源码修改的基础上，再增加对交叉编译的支持。

添加`configure/os/CONFIG.linux-x86_64.linux-loongarch64`

``` shell
# CONFIG.linux-x86_64.linux-loongarch64
#
# Definitions for linux-x86_64 host - linux-loongarch64 target builds
# Sites may override these in CONFIG_SITE.linux-x86_64.linux-loongarch64
#-------------------------------------------------------

VALID_BUILDS = Ioc Command
GNU_TARGET = loongarch64-linux-gnu

# prefix of compiler tools
CMPLR_SUFFIX =
CMPLR_PREFIX = $(addsuffix -,$(GNU_TARGET))

# Provide a link-time path for readline if needed
OP_SYS_INCLUDES += $(READLINE_DIR:%=-I%/include)
READLINE_LDFLAGS = $(READLINE_DIR:%=-L%/lib)
RUNTIME_LDFLAGS_READLINE_YES_NO = $(READLINE_DIR:%=-Wl,-rpath,%/lib)
RUNTIME_LDFLAGS += \
    $(RUNTIME_LDFLAGS_READLINE_$(LINKER_USE_RPATH)_$(STATIC_BUILD))
SHRLIBDIR_LDFLAGS += $(READLINE_LDFLAGS)
PRODDIR_LDFLAGS += $(READLINE_LDFLAGS)

# Library flags
STATIC_LDFLAGS_YES= -Wl,-Bstatic
STATIC_LDFLAGS_NO=
STATIC_LDLIBS_YES= -Wl,-Bdynamic
STATIC_LDLIBS_NO=
```

添加`configure/os/CONFIG_SITE.linux-x86_64.linux-loongarch64`

``` shell
# CONFIG_SITE.linux-x86_64.linux-loongarch64
#
# Site specific definitions for linux-x86_64 host - linux-loongarch64 target builds
#-------------------------------------------------------

# Set GNU crosscompiler target name
GNU_TARGET = loongarch64-linux-gnu

# Set GNU tools install path
# Examples is the installation at the APS:
GNU_DIR = /opt/toolchain-loongarch64-linux-gnu-gcc8-host-x86_64-2022-07-18

# If cross-building shared libraries and the paths on the target machine are
# different than on the build host, you should uncomment the lines below to
# disable embedding compile-time library paths into the generated files.
# You will need to provide another way for programs to find their shared
# libraries at runtime, such as by setting LD_LIBRARY_PATH or (better) using
# mechanisms related to /etc/ld.so.conf
#SHRLIBDIR_RPATH_LDFLAGS_YES_NO =
#PRODDIR_RPATH_LDFLAGS_YES_NO =
# However it is usually simpler to set STATIC_BUILD=YES here and not
# try to use shared libraries at all when cross-building, like this:
STATIC_BUILD=YES
SHARED_LIBRARIES=NO

# To use libreadline, point this to its install prefix
#READLINE_DIR = $(GNU_DIR)
#READLINE_DIR = /tools/cross/linux-x86.linux-arm/readline
# See CONFIG_SITE.Common.linux-arm for other COMMANDLINE_LIBRARY values
#COMMANDLINE_LIBRARY = READLINE
```

`GNU_DIR`需要改为安装交叉编译工具链的路径。  
`STATIC_BUILD`和`SHARED_LIBRARIES`可以设置是否为非*静态编译*，即是否生成动态库`.so`，**两个设置项必须一起修改**，这里的设置会覆盖掉`configure/CONFIG_SITE`中的设置。由于动态库在编译一些其他工具时还会用到，所以这里我选择生成动态库。

``` shell
# STATIC_BUILD=YES
# SHARED_LIBRARIES=NO
```

然后，需要设置交叉编译的目标架构。

新增`configure/CONFIG_SITE.local`，或者直接修改`configure/CONFIG_SITE`（不推荐）。

``` shell
CROSS_COMPILER_TARGET_ARCHS=linux-loongarch64
```

最后，进行编译即可。（确保构建系统上有make和perl，应该都有吧。）

``` shell
# 到源码目录下
$ cd ~/loongson/base-7.0.7
$ make -j8
```

等待编译完成即可。

编译完成后，可以看到`bin`和`lib`目录下，都有`linux-loongarch64`、`linux-x86_64`两个目录，其中`linux-loongarch64`目录下就是我们要在开发板上运行的EPICS工具包了。`linux-x86_64`目录下的则是编译生成的本机的EPICS工具包，待会儿我们还会用到。

由于开发板的存储空间很小，只有几百兆，所以，我们只能单独将龙架构的内容下载到板子上。

目录如下：

```
base
├─ bin
│   └─ linux-loongarch64
├─ db
├─ dbd
└─ lib
    └─ linux-loongarch64
```

将目录中的内容全部打包下载到开发板即可。

``` shell
# 在开发板上运行
$ cd base/bin/linux-loongarch64
# 运行软IOC测试
$ ./softIoc
epics> 
```

> ※ 如果使用非静态编译（生成动态库）的方式编译，运行时可能会提示找不到动态库。需要将动态库添加到系统的动态库路径。

如下（根据实际情况修改路径）：

``` shell
$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/root/base/lib/linux-loongarch64
```

## 编译 IOC

前面已经讲了，我么需要在构建主机上交叉编译IOC，具体步骤和直接创建编译IOC基本一样。

``` shell
$ cd ~
# 创建目录
$ mkdir test
$ cd test/
# 替换成自己编译EPICS的目录，注意是使用linux-x86_64目录下的脚本
$ ~/loongson/base-7.0.7/bin/linux-x86_64/makeBaseApp.pl -t example test
$ ~/loongson/base-7.0.7/bin/linux-x86_64/makeBaseApp.pl -i -t example test

The following target architectures are available in base:
    linux-loongarch64
    linux-x86_64
What architecture do you want to use? linux-loongarch64
The following applications are available:
    test
What application should the IOC(s) boot?
The default uses the IOC's name, even if not listed above.
Application name? test
```

**这里唯一多的步骤就是选择目标架构，输入`linux-loongarch64`即可。**

然后修改编译设置，这里就不用生成动态库了，直接使用静态编译。

添加`configure/CONFIG_SITE.local`，启用静态编译。

``` shell
# Build shared libraries (DLLs on Windows).
#  Must be either YES or NO.  Definitions in the target-specific
#  os/CONFIG.Common.<target> and os/CONFIG_SITE.Common.<target> files may
#  override this setting.  On Windows only these combinations are valid:
#    SHARED_LIBRARIES = YES and STATIC_BUILD = NO
#    SHARED_LIBRARIES = NO  and STATIC_BUILD = YES
SHARED_LIBRARIES=NO

# Build client objects statically.
#  Must be either YES or NO.
STATIC_BUILD=YES
```

最后编译。

``` shell
$ cd ~/test
$ make
```

等待编译完成即可。然后将生成的可执行文件下载到开发板，这里依旧是只下载运行IOC所必需的内容。

目录如下：

```
test
├─ bin
│   └─ linux-loongarch64
├─ db
├─ dbd
└─ iocBoot
    └─ ioctest
```

在把IOC复制到开发板上之后，还需要根据实际情况修改IOC运行的环境变量。

修改`iocBoot/ioctest/envPaths`

``` shell
epicsEnvSet("IOC","ioctest")
epicsEnvSet("TOP","/root/test")
epicsEnvSet("EPICS_BASE","/root/base")
epicsEnvSet("EPICS_HOST_ARCH","linux-loongarch64")
```

最后在开发板上运行IOC。

``` shell
# 在开发板上运行
$ cd ~/test/iocBoot/ioctest
# 添加可执行权限
$ chmod +x st.cmd
$ ./st.cmd

#!../../bin/linux-loongarch64/test
< envPaths
epicsEnvSet("IOC","ioctest")
epicsEnvSet("TOP","/root/test")
epicsEnvSet("EPICS_BASE","/root/base")
epicsEnvSet("EPICS_HOST_ARCH","linux-loongarch64")
cd "/root/test"
## Register all support components
dbLoadDatabase "dbd/test.dbd"
test_registerRecordDeviceDriver pdbbase
Warning: IOC is booting with TOP = "/root/test"
          but was built with TOP = "/home/ubuntu/test"
## Load record instances
dbLoadTemplate "db/user.substitutions"
dbLoadRecords "db/testVersion.db", "user=lsgd"
dbLoadRecords "db/dbSubExample.db", "user=lsgd"
cd "/root/test/iocBoot/ioctest"
iocInit
Starting iocInit
############################################################################
## EPICS R7.0.7
## Rev. 2023-12-12T14:06+0800
## Rev. Date build date/time:
############################################################################
iocRun: All initialization complete
## Start any sequence programs
#seq sncExample, "user=lsgd"
epics>
```

*这里可以看到启动时会有一个`Warning`，警告IOC运行时和编译时的`TOP`路径不一致，但实际上并不影响IOC运行，忽略即可。*

此时可以使用`dbl`、`dbpr`等命令查看变量。或者在其他终端运行`camonitor`程序。
