---
title: "交叉编译 ACAI"
date: 2023-12-26T19:31:56+08:00
draft: true
description: 交叉编译 ACAI
tags: ["linux", "EPICS", "龙芯"]
keywords: ["linux", "EPICS", "龙芯"]
categories: ["EPICS"]
---

## 关于 ACAI

ACAI 是一个C++封装的`Channel Access`协议应用开发接口（API），提供异步通道访问接口。

[ACAI Channel Access Interface](https://github.com/andrewstarritt/acai)

EPICS Qt依赖ACAI提供的Channel Access接口。

## 前置步骤

这篇笔记是[交叉编译EPICS和IOC](../../posts/交叉编译epics和ioc/)内容的补充。

在进行下面步骤前，请完成**配置交叉编译环境**和**编译 EPICS Base**。

*这里依旧以龙芯架构为例。*

EPICS base 编译完成后，可以看到`bin`目录下有`linux-loong64`、`linux-x86_64`两个目录，`linux-x86_64`目录下比`linux-loong64`目录多出了许多`perl`脚本，我们需要把这些脚本复制到龙架构的目录下，下面编译需要用到。

``` shell
$ cp ./bin/linux-x86_64/*.pl ./bin/linux-loong64/
```

## 编译

在[EPICS-Qt安装](../../posts/epics-qt安装/)中已经介绍过编译ACAI。这次是使用交叉编译方式，步骤略有不同。

```sh
cd ~/loongson/
git clone https://github.com/andrewstarritt/acai.git
cd acai
vi configure/RELEASE.local

# 修改交叉编译的目标架构，和EPICS base中保持一致
# EPICS_HOST_ARCH=linux-loong64
# 修改EPICS_BASE路径，例：
EPICS_BASE=/home/ubuntu/loongson/base-7.0.8

# make LD=loongarch64-linux-gnu-ld CC=loongarch64-linux-gnu-gcc CCC=loongarch64-linux-gnu-g++
make
# 等待编译完成
```

编译完成后可以在`lib/linux-loong64/`目录下找到`libacai.so`。
