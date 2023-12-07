---
title: "EPICS Qt安装"
date: 2023-05-04T15:06:56+08:00
lastmod: 2023-12-07T15:57:05+08:00
draft: false
description: Linux上编译安装EPICS Qt
tags: ["linux", "EPICS", "Qt", "龙芯"]
keywords: ["linux", "EPICS", "Qt", "龙芯"]
categories: ["EPICS"]
---

## 前言

本文主要记录了EPICS Qt在Linux上的安装步骤。这里以loongnix操作系统为例，Ubuntu系统上编译安装步骤类似。

EPICS Qt是一个基于Qt的分层框架，使用Channel Access （CA） and PV Access（PVA）访问EPICS数据。它是为快速开发控制系统图形界面而设计的，最初是在澳大利亚同步加速器开发的。

## 安装EPICS

这里不再写具体步骤了，总之就是非常简单，下载、解压、编译即可。具体步骤可以参考[以前的文章](https://kira-96.github.io/posts/龙芯3a5000loongarch64上编译运行epics/)。

## 安装Qt

直接使用终端安装Qt
```sh
sudo apt update
sudo apt install qtbase5-dev qt5-qmake qtcreator
sudo apt install qtdeclarative5-dev qttools5-dev
# 安装Qt Svg库，编译QWT时需要用到
sudo apt install libqt5svg5-dev
```

## 安装QWT

Qt EPICS推荐使用Qwt 6.1.4，如果在Ubuntu 20.04上直接通过终端安装也是这个版本。我使用Qwt 6.2.0编译，也是没有问题的，这里以Qwt 6.2.0为例。

先下载Qwt的源码 [下载Qwt-6.2.0](https://sourceforge.net/projects/qwt/files/qwt/6.2.0/qwt-6.2.0.tar.bz2/download)。
下载完成后解压

```sh
# 解压tar.bz2
tar -jxvf qwt-6.2.0.tar.bz2
# 解压zip
unzip qwt-6.2.0.zip
```

解压完成后编译Qwt，使用QtCreator或者在终端使用`qmake`都可以。

然后手动将编译生成的文件复制到以下位置，例：

```sh
# 复制编译生成的qwt
sudo cp -r build-qwt-unknown-Release/lib/* /usr/lib/loongarch64-linux-gnu/
# 复制编译生成的designer插件
sudo cp build-qwt-unknown-Release/designer/plugins/designer/libqwt_designer_plugin.so /usr/lib/loongarch64-linux-gnu/qt5/plugins/designer/
# 复制qwt头文件
sudo mkdir /usr/include/qwt
sudo cp qwt-6.2.0/src/*.h /usr/include/qwt
```

## 安装ACAI

[ACAI Channel Access Interface](https://github.com/andrewstarritt/acai)

EPICS Qt依赖ACAI提供的Channel Access接口。

```sh
cd /usr/local/epics/modules/
git clone https://github.com/andrewstarritt/acai.git
cd acai
vi configure/RELEASE.local
# 修改EPICS_BASE路径，例：
# EPICS_BASE=/usr/local/epics/base-7.0.7
make -j8
# 等待编译完成
```

## 安装google protobuf

如果需要EPICS Qt支持[EPICS Archiver Appliance](https://slacmshankar.github.io/epicsarchiver_docs/index.html)，需要安装google protobuf。

```sh
sudo apt install protobuf-compiler libprotobuf-dev
```

## EPICS Qt

首先克隆EPICS Qt的两个代码仓库。

```sh
# framework and support libraries
git clone https://github.com/qtepics/qeframework.git
# QEGui display manager
git clone https://github.com/qtepics/qegui.git
```

这里我将代码都放在`~/QtEpics`目录。

在开始编译前，需要先配置一些环境变量（根据自己的实际情况设置）。具体可以参考 [EPICS Qt Environment Variables](https://qtepics.github.io/environment_variables.html)

```sh
export EPICS_HOST_ARCH=linux-loongarch64
export EPICS_BASE=/usr/local/epics/base-7.0.7
export ACAI=/usr/local/epics/modules/acai
export QWT_INCLUDE_PATH=/usr/include/qwt
export QWT_ROOT=/usr/lib/loongarch64-linux-gnu
export QE_FRAMEWORK="$HOME/QtEpics/qeframework"
# 支持PV Access
export QE_PVACCESS_SUPPORT=YES
# 支持Archiver Appliance
export QE_ARCHAPPL_SUPPORT=YES
export PROTOBUF_INCLUDE_PATH=/usr/include/google/protobuf
export PROTOBUF_LIB_DIR=/usr/lib/loongarch64-linux-gnu
```

如果环境变量设置了支持Archiver Appliance，需要先编译`archapplDataSup`。

```sh
cd ~/QtEpics/qeframework/archapplDataSup/
make
```

编译完成后，可以看到`~/QtEpics/qeframework/lib/linux-loongarch64`目录下有`libarchapplData.a`、`libarchapplData.so`两个文件。

然后依次编译 `qeframework` `qeplugin` `qegui`。EPICS Qt文档说明需要修改`configure/RELEASE`文件，但我这里修改后似乎没有生效，可能是使用了Qt Creator的原因，只能通过上面的环境变量设置。

> 注意：这里设置完环境变量，需要直接通过终端打开Qt Creator。

- 编译`qeframework`
`$HOME/QtEpics/qeframework/qeframeworkSup/project/framework.pro`
- 编译`qeplugin`
`$HOME/QtEpics/qeframework/qepluginApp/project/qeplugin.pro`
- 编译`qegui`
`$HOME/QtEpics/qegui/qeguiApp/project/QEGuiApp.pro`

最后将编译生成的文件复制到以下位置，例：

```sh
sudo cp ~/QtEpics/qeframework/lib/linux-loongarch64/libarchapplData.so /usr/lib/loongarch64-linux-gnu/
sudo cp ~/QtEpics/qeframework/lib/linux-loongarch64/libQEFramework.so /usr/lib/loongarch64-linux-gnu/
sudo cp ~/QtEpics/qeframework/lib/linux-loongarch64/designer/libQEPlugin.so /usr/lib/loongarch64-linux-gnu/qt5/plugins/designer/
```

运行`QEGuiApp`

```sh
cd ~/epics/qtepics/qegui/bin/linux-loongarch64
./qegui
```

## 运行测试

运行时环境变量设置，例：

```sh
export QE_ARCHIVE_TYPE=ARCHAPPL
export QE_ARCHIVE_LIST="http://192.168.1.2:17665/mgmt/bpl"
export EPICS_CA_ADDR_LIST="192.168.1.2:5732 192.168.1.3:6666"
```

![关于QEGui](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/2023-05-04_11-05-23.png)

![QE控件](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/2023-05-04_11-04-19.png)

## 问题汇总

*编译过程中可能会遇到一些问题*

- 找不到Qwt的头文件
  > 解决办法: 修改qeframework/qeframeworkSup/project/common/common.pri
  ```diff
   INCLUDEPATH += $$PWD
  +INCLUDEPATH += $$(QWT_INCLUDE_PATH)
  ```
- 找不到QEFramework的头文件
  > 解决办法: 修改对应项目的项目文件
  ```diff
  +INCLUDEPATH += $$(QE_FRAMEWORK)/include
  ```

*Windows上编译踩坑*

> 我这里使用的是MinGW编译器，仅供参考。Windows编译安装EPICS，可以参考我的[笔记](../../notes/windows上使用mingw编译安装epics/)。

- ACAI编译报错，这个是由于平台函数的差异导致的报错。
  > 我已经提交了修复补丁，参考 [fix build error on windows](https://github.com/andrewstarritt/acai/pull/2)。

- 编译google protobuf

  网上有很多关于windows编译google protobuf的文章。请自行搜索解决。

- Windows上运行程序
  
  将编译过程中生成的DLL全部复制到`qegui`程序目录下，大致汇总一下有：

  `qwt.dll`，`libprotobuf.dll`，`acai.dll`，`QEFramework.dll`，**`archapplData.dll`**（这个是在Windows上才会生成，在`qeframework\archapplDataSup\src\O.windows-x64-mingw\`目录下可以找到）

  EPICS相关的DLL:

  `ca.dll`，`Com.dll`，`nt.dll`，`pvAccess.dll`，`pvData.dll`

  以及Qt相关的DLL。

## 参考链接
- [EPICS Qt at GitHub](https://qtepics.github.io)
- [EPICS Qt Getting Started](https://qtepics.github.io/getting_started.html)
- [Archiver Appliance Support for EPICS Qt](https://qtepics.github.io/archiver_appliance.html)
