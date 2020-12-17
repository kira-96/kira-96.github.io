---
title: 在树莓派上安装 Power Shell 并用 oh-my-posh 美化
date: 2020-03-07T15:56:50+08:00
draft: false
description: 在树莓派上安装 Power Shell
tags: [ "分享" , "Power Shell" ]
keywords: [ "分享" ]
categories: [ "分享" ]
isCJKLanguage: true
---

## 前言

由于我平时是将树莓派(Respberry Pi)当成一个Linux电脑来使用，平时都是通过ssh连接到树莓派来进行操作的，所以一直都是通过终端进行操作的。而树莓派系统的终端又中规中距，不怎么好看。刚好这两天接触到了一个十分漂亮的Power Shell主题[oh-my-posh](https://github.com/JanDeDobbeleer/oh-my-posh)，所以就想着能不能弄到树莓派上。折腾了半天，终于成功了，过程还算顺利。

## 目标

我的目标是美化树莓派的终端，由于oh-my-posh是power shell的主题，所以首先需要安装power shell，然后再通过power shell安装oh-my-posh。

1. 安装 [Power Shell](https://github.com/PowerShell/PowerShell)
2. 安装 [oh-my-posh](https://github.com/JanDeDobbeleer/oh-my-posh)

## 安装 Power Shell Core

刚好前几天Power Shell Core 7发布了，所以我这里就安装了最新的版本。

Power Shell 官网是这样的说明的。

> 当前仅 Raspbian Stretch 支持 PowerShell。
> CoreCLR 和 PowerShell Core 仅适用于 Pi 2 和 Pi 3 设备，因为其他设备（如 Pi 0）有不受支持的处理器。

我是用的是树莓派 3B+，测试是可以的。

具体的操作按照Power Shell官网的[安装说明](https://docs.microsoft.com/zh-cn/powershell/scripting/install/installing-powershell-core-on-linux?view=powershell-7#raspbian)

首先安装Power Shell的依赖：

``` sh
# Prerequisites

# Update package lists
sudo apt-get update

# Install libunwind8 and libssl1.0
# Regex is used to ensure that we do not install libssl1.0-dev, as it is a variant that is not required
sudo apt-get install '^libssl1.0.[0-9]$' libunwind8 -y
```

然后到[这里](https://github.com/PowerShell/PowerShell/releases)下载最新的Power Shell二进制包。

这里需要下载arm32位的二进制包。

**powershell-7.0.0-linux-arm32.tar.gz**

下载完成后解压到任意目录即可运行。

或者使用官网的方式：

``` sh
# Download and extract PowerShell

# Grab the latest tar.gz
wget https://github.com/PowerShell/PowerShell/releases/download/v7.0.0/powershell-7.0.0-linux-arm32.tar.gz

# Make folder to put powershell
mkdir ~/powershell

# Unpack the tar.gz file
tar -xvf ./powershell-7.0.0-linux-arm32.tar.gz -C ~/powershell

# Start PowerShell
~/powershell/pwsh
```

最后，如果想要在任意位置都能启动Power Shell，需要创建启动Power Shell的软链接。

``` sh
sudo ln -s ~/path/to/powershell/pwsh /usr/bin/pwsh
```

或者参考官网的方式:

``` sh
# Start PowerShell from bash with sudo to create a symbolic link
sudo ~/powershell/pwsh -c New-Item -ItemType SymbolicLink -Path "/usr/bin/pwsh" -Target "\$PSHOME/pwsh" -Force

# alternatively you can run following to create a symbolic link
# sudo ln -s ~/powershell/pwsh /usr/bin/pwsh

# Now to start PowerShell you can just run "pwsh"
```

现在只要在终端输入`pwsh`就可以进入Power Shell了。

## 为 Power Shell 安装 oh-my-posh

安装之前需要先安装[powerline字体](https://github.com/powerline/fonts)，否则，oh-my-posh安装完成后会由于缺少字体而显示不正常。

``` sh
sudo apt-get install fonts-powerline
```

然后就可以为Power Shell安装oh-my-posh主题了。

首先要进入Power Shell，在终端输入`pwsh`即可。

在Power Shell下依次执行下面两个命令：

``` sh
Install-Module posh-git -Scope CurrentUser
Install-Module oh-my-posh -Scope CurrentUser
```

安装过程中全部选 **是(Y)** 就可以了。

安装完成后就可以使用oh-my-posh了：

``` sh
# Start the default settings
Set-Prompt
# Alternatively set the desired theme:
Set-Theme Agnoster
```

最后需要保存Power Shell的配置，这样每次进入Power Shell就是我们设定的主题了。

``` sh
# 在 Power Shell下执行下面命令，如果不存在配置文件就创建一个
if (!(Test-Path -Path $PROFILE )) { New-Item -Type File -Path $PROFILE -Force }

# 使用树莓派的编辑器修改配置文件
vi $PROFILE
```

在配置文件中输入下面内容：

``` sh
Import-Module posh-git
Import-Module oh-my-posh
Set-Theme Paradox
```

然后再重新进入Power Shell就可以看到主题已经成功应用了。

oh-my-posh提供了多种主题效果，可以看[这里](https://github.com/JanDeDobbeleer/oh-my-posh#themes)，如果需要更换主题，可以直接在Power Shell中执行：

``` sh
Set-Theme mytheme
```

## 效果

虽然 Power Shell 启动有一点慢，但显示效果还是很不错的。

最后放上实际运行的效果：

在树莓派中显示效果：

![respberry-pi-shell](https://i.loli.net/2020/12/17/Vji4rY71szntmeR.png)

在其它终端中的显示效果(Termius)：

![termius](https://i.loli.net/2020/12/17/kRtrNwd9lJnWxia.png)

**参考**

- [在树莓派上安装PowerShellCore](https://docs.microsoft.com/zh-cn/powershell/scripting/install/installing-powershell-core-on-linux?view=powershell-7#raspbian)
- [安装powerline-fonts](https://github.com/powerline/fonts)
- [安装oh-my-posh](https://github.com/JanDeDobbeleer/oh-my-posh)
