---
title: "Windows上使用MinGW编译安装EPICS"
date: 2023-11-29T18:57:02+08:00
lastmod: 2023-12-07T09:43:50+08:00
draft: true
tags: ["EPICS"]
keywords: ["EPICS"]
categories: ["EPICS"]
---

## 需要使用的软件

- [Strawberry Perl for Windows](https://strawberryperl.com/)
- [EPICS Base](https://epics.anl.gov/base/index.php)

编译Base需要有gcc、g++、make、perl这些工具，但其实我们只需要安装`Strawberry Perl`就可以了，安装完成后就有了`MinGW`的编译环境，足够编译安装EPICS了。

这里使用MinGW环境编译EPICS，不使用MSVC编译器。

## 安装 Strawberry Perl

这里选择 [Strawberry Perl 5.32.1.1](https://strawberryperl.com/release-notes/5.32.1.1-64bit.html)。经测试`base-7.0.7`可正常编译，后续版本的perl编译会报错。

直接安装即可，需要注意的是，安装路径不能有空格和中文，最好放在盘符的根目录下。  
例：`D:\Strawberry`

安装完成后检查**系统环境变量**，查看系统`Path`环境变量是否有`Strawberry Perl`的路径。没有则手动添加，以安装在D盘为例。

```shell
D:\Strawberry\c\bin
D:\Strawberry\perl\site\bin
D:\Strawberry\perl\bin
```

其中`D:\Strawberry\c\bin`就是MinGW环境的路径。

查看`Perl`版本，检查一下是不是装好了。

```shell
> perl -v

This is perl 5, version 32, subversion 1 (v5.32.1) built for MSWin32-x64-multi-thread

Copyright 1987-2021, Larry Wall

Perl may be copied only under the terms of either the Artistic License or the
GNU General Public License, which may be found in the Perl 5 source kit.

Complete documentation for Perl, including FAQ lists, should be found on
this system using "man perl" or "perldoc perl".  If you have access to the
Internet, point your browser at http://www.perl.org/, the Perl Home Page.
```

## 编译安装EPICS Base

修改base源码目录下的`startup/windows.bat`文件。

```bat
rem The location of Strawberry Perl (pathname).  If empty, Strawberry Perl
rem is assumed to already be in PATH and will not be added.  If nonempty,
rem Strawberry Perl will be added to PATH.
rem 设置Strawberry Perl安装路径
set _strawberry_perl_home=D:\Strawberry

rem The EPICS host architecture specification for EPICS_HOST_ARCH
rem (<os>-<arch>[-<toolset>] as defined in configure/CONFIG_SITE).
rem 设置编译主机架构，这里使用mingw
set _epics_host_arch=windows-x64-mingw

rem The install location of EPICS Base (pathname).  If nonempty and
rem _auto_path_append is yes, it will be used to add the host architecture
rem bin directory to PATH.
set _epics_base=C:\EPICS\base-7.0.7

rem Set the environment for Microsoft Visual Studio
rem 使用 rem 注释掉下面一行
rem call "%_visual_studio_home%\VC\Auxiliary\Build\vcvarsall.bat" x64
```

注意，编译需要使用命令行工具`cmd`，不能用`powershell`。

```bat
cd base-7.0.7
.\startup\windows.bat
gmake -j16
```

等待编译完成。  
编译完成后的工具在`bin\windows-x64-mingw`目录下。

测试使用：

```bat
cd bin\windows-x64-mingw
softIoc.exe
epics>
```
