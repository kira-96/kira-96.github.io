---
title: "龙芯3A5000(loongarch64)上编译运行EPICS"
date: 2023-02-01T15:51:40+08:00
draft: false
description: 在龙芯3A5000(loongarch64)上编译运行EPICS
tags: ["linux", "EPICS", "龙芯"]
keywords: ["linux", "EPICS", "龙芯"]
categories: ["EPICS"]
---

## 前言

之前尝试过在龙芯3A4000上编译运行EPICS，由于3A4000还是`mips64`指令集，而3A5000则是龙芯的自主指令集`loongarch64`，适配起来步骤也会有所不同。

这次使用的是龙博特龙芯3A5000电脑主机。

![Loongson-3A5000-HV](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/2023-02-01_12-58-17.png)

虽然EPICS官方并没有适配`loongarch`和`mips64`，无法做到开箱即用，但只要有gcc、g++、make、perl这些工具，理论上就能编译运行EPICS，在开始编译前，确保你的设备上已经装好了这些工具。

## 下载 base

这里我们就以目前最新版本7.0.7为例，其它版本的Base也类似。

``` shell
$ cd ~/下载/
$ wget https://epics.anl.gov/download/base/base-7.0.7.tar.gz
$ tar -xzvf base-7.0.7.tar.gz
```

你可以在你觉得合适的位置编译安装Base，这里按我们的习惯，放在`/usr/local/epics`目录下。

``` shell
$ mkdir /usr/local/epics
$ mv base-7.0.7 /usr/local/epics/
```

# 编译

按照一般步骤，现在就可以开始编译了，我们可以先尝试一下，看看是什么结果。

``` shell
$ cd /usr/local/epics/base-7.0.7/
# 执行 `make` 命令
$ make
```

![3A5000编译错误](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/2023-02-01_13-04-33.png)

不出所料，果然失败了，输出的错误和在3A4000上编译时的错误也有一些不同。

下面是在3A4000上编译时输出的错误：

![3A4000编译错误](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/localhost-usr-local-epics-base-7.0.7.png)

下面一行报错是差不多的，在`loongarch64`上编译却多了上面一行报错，意思就是没有识别出`loongarch64`架构。

但是先不要慌，这里同时也给出了报错的位置，让我们看看`EpicsHostArch.pl`里写了些什么。

``` shell
$ vi ./src/tools/EpicsHostArch.pl
```

它其实就是一个`perl`脚本，用来判断当前的系统和cpu架构，而`loongarch64`显然没有做适配，所以就出现了上面错误。

> "Architecture 'loongarch64-linux-gnu-thread-multi' not recognized"

既然识别不了`loongarch64`，那我们就手动添加一行，让它可以识别就行了，即使看不太懂上面的脚本也没关系，看个半懂就行了。

![Architecture](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/2023-02-01_13-32-04.png)

我们在如图的光标位置添加一行内容，来让它可以识别`loongarch64`架构。

``` perl
return 'linux-loongarch64'  if m/^loongarch64-linux/;
```

此时我们再执行一下`make`命令。

![3A5000编译错误](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/2023-02-01_13-32-44.png)

可以看到，现在已经可以识别出`loongarch64-linux`了，报错和在3A4000上编译时也基本一样了。

> 以下步骤同样适用于在3A4000（mips64）上编译EPICS，只需要将`loongarch64`全部替换为`mips64`

剩下的报错就是，没有找到对应的编译配置项，我们同样可以仿照已经做了适配的架构来改写，直接按照下面步骤来就可以了。

1. 添加 CONFIG.Common.linux-loongarch64

``` shell
$ cd configure/os/
# 添加 CONFIG.Common.linux-loongarch64
$ cp CONFIG.Common.linux-aarch64 CONFIG.Common.linux-loongarch64
$ vi CONFIG.Common.linux-loongarch64
```

修改成如下内容：

``` shell
# CONFIG.Common.linux-loongarch64
#
# Definitions for linux-loongarch64 target builds
# Override these settings in CONFIG_SITE.Common.linux-loongarch64
#-------------------------------------------------------

# Include definitions common to all Linux targets
include $(CONFIG)/os/CONFIG.Common.linuxCommon

ARCH_CLASS = loongarch64
```

2. 添加 CONFIG.linux-loongarch64.Common

``` shell
# 添加 CONFIG.linux-loongarch64.Common
$ cp CONFIG.linux-aarch64.Common CONFIG.linux-loongarch64.Common
$ vi CONFIG.linux-loongarch64.Common
```

修改成如下内容(内容没有变化，可以不修改)：

``` shell
# CONFIG.linux-loongarch64.Common
#
# Definitions for linux-loongarch64 host builds
# Sites may override these definitions in CONFIG_SITE.linux-loongarch64.Common
#-------------------------------------------------------

#Include definitions common to unix hosts
include $(CONFIG)/os/CONFIG.UnixCommon.Common
```

3. 添加 CONFIG.linux-loongarch64.linux-loongarch64

``` shell
# 添加 CONFIG.linux-loongarch64.linux-loongarch64
$ cp CONFIG.linux-aarch64.linux-aarch64 CONFIG.linux-loongarch64.linux-loongarch64
$ vi CONFIG.linux-loongarch64.linux-loongarch64
```

修改成如下内容(内容没有变化，可以不修改)：

``` shell
# CONFIG.linux-loongarch64.linux-loongarch64
#
# Definitions for native linux-loongarch64 builds
# Override these definitions in CONFIG_SITE.linux-loongarch64.linux-loongarch64
#-------------------------------------------------------

# Include common gnu compiler definitions
include $(CONFIG)/CONFIG.gnuCommon
```

4. 添加 CONFIG_SITE.Common.linux-loongarch64

``` shell
# 添加 CONFIG_SITE.Common.linux-loongarch64
$ cp CONFIG_SITE.Common.linux-aarch64 CONFIG_SITE.Common.linux-loongarch64
$ vi CONFIG_SITE.Common.linux-loongarch64
```

内容没有变化，可以不修改。

5. 添加 CONFIG_SITE.linux-loongarch64.linux-loongarch64

``` shell
# 添加 CONFIG_SITE.linux-loongarch64.linux-loongarch64
$ cp CONFIG_SITE.linux-aarch64.linux-aarch64 CONFIG_SITE.linux-loongarch64.linux-loongarch64
$ vi CONFIG_SITE.linux-loongarch64.linux-loongarch64
```

修改成如下内容：

``` shell
# CONFIG_SITE.linux-loongarch64.linux-loongarch64
#
# Site specific definitions for native linux-loongarch64 builds
#-------------------------------------------------------

# It makes sense to include debugging symbols even in optimized builds
# in case you want to attach gdb to the process or examine a core-dump.
# This does cost disk space, but not memory as debug symbols are not
# loaded into RAM when the binary is loaded.
#OPT_CFLAGS_YES += -g
#OPT_CXXFLAGS_YES += -g
```

这里是对编译器的优化选项，暂时不知道怎么改，我就直接注释掉了。

## 重新编译

到这里就全部改好了，其实最主要修改的就是第一个文件。下面就可以尝试编译了，这次应该没问题了。

``` shell
$ cd /usr/local/epics/base-7.0.7/
# 执行 `make` 命令
$ make -j8
```

接下来就静静等待编译完成。

编译完后查看编译输出目录`bin/linux-loongarch64/`。

![编译输出目录](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/2023-02-01_14-09-04.png)

## 添加到PATH

为了方便以后使用，我们将编译输出的可执行文件目录添加到`PATH`。

```shell
$ cd ~
$ mkdir .epics
$ cd .epics/
$ touch env
$ vi env
```

编辑 env 如下

``` shell
#!/bin/sh
# EPICS base shell setup
# affix colons on either side of $PATH to simplify matching
case ":${PATH}:" in
    *:"/usr/local/epics/base-7.0.7/bin/linux-loongarch64":*)
        ;;
    *)
        # Prepending path in case a system-installed epics needs to be overridden
        export PATH="/usr/local/epics/base-7.0.7/bin/linux-loongarch64:$PATH"
        ;;
esac
```

然后修改 `.bashrc`

```shell
$ cd ~
$ vi .bashrc
```

在文件最后添加一行

```shell
. "$HOME/.epics/env"
```

执行下面命令，使添加PATH生效

```shell
$ . .bashrc
```

## 运行EPICS IOC

做完上面的步骤，EPICS base的配置就完成了，我们来尝试运行一下。

在终端执行`softIoc`。

![softIoc](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/2023-02-01_14-13-20.png)

运行正常，大功告成！

只要base可以成功运行，其它的一些模块应该也没有问题，后续我会继续尝试在龙芯上安装EPICS其它的模块。

## 链接

- [EPICS - Experimental Physics and Industrial Control System (anl.gov)](https://epics.anl.gov/index.php)
- [epics-base - (launchpad.net)](https://git.launchpad.net/epics-base) / [epics-base/epics-base](https://github.com/epics-base/epics-base) / [EPICS Base (anl.gov)](https://epics.anl.gov/base/index.php)
