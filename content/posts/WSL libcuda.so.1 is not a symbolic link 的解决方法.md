---
title: "WSL libcuda.so.1 is not a symbolic link 的解决方法"
date: 2022-02-09T08:48:30+08:00
draft: false
tags: [ "Linux", "WSL" ]
keywords: [ "分享" ]
isCJKLanguage: true
enableDisqus: true
---

## 问题描述

在使用 WSL 更新软件包的时候经常会遇到这样一个报错

``` shell
/sbin/ldconfig.real: /usr/lib/wsl/lib/libcuda.so.1 is not a symbolic link
```

意思是说 `/usr/lib/wsl/lib/libcuda.so.1` 不是一个符号链接。

## 问题分析

通过名字可以判断这应该是nVidia显卡驱动相关的库，进入 `/usr/lib/wsl/lib/` 目录，可以看到有 `libcuda.so`、`libcuda.so.1`、`libcuda.so.1.1` 三个文件，都是文件形式，而通过报错我们知道 `libcuda.so`、`libcuda.so.1` 应该是符号链接文件。

它们关系应该是：

libcuda.so -> libcuda.so.1 -> libcuda.so.1.1

知道原因就好解决了，把 `libcuda.so`、`libcuda.so.1` 删掉，再重新创建符号链接就可以了。

``` shell
ubuntu@dell:/usr/lib/wsl/lib$ sudo rm libcuda.so
rm: 无法删除 'libcuda.so': 只读文件系统
```

很遗憾，这样是不行的。最后经过多方查找，终于找到了解决方案。

## 解决方法

解决方法就是上面的方法，但不是在 WSL 中操作。

使用管理员权限执行 cmd 命令:

``` powershell
C:>cd C:\Windows\System32\lxss\lib
C:\Windows\System32\lxss\lib>del /s /q "libcuda.so"
C:\Windows\System32\lxss\lib>del /s /q "libcuda.so.1"
C:\Windows\System32\lxss\lib>mklink libcuda.so.1 libcuda.so.1.1
C:\Windows\System32\lxss\lib>mklink libcuda.so libcuda.so.1
```

或者在Powershell中执行：

``` powershell
cd C:\Windows\System32\lxss\lib
rm libcuda.so
rm libcuda.so.1
wsl -e /bin/bash
ln -s libcuda.so.1.1 libcuda.so.1
ln -s libcuda.so.1.1 libcuda.so
```

然后在 wsl 中执行:

``` shell
$ sudo ldconfig
```

**参考**

- [ldconfig: /usr/lib/wsl/lib/libcuda.so.1 is not a symbolic link](https://blog.kiyoko.io/2022/01/17/ldconfig-usr-lib-wsl-lib-libcuda-so-1-is-not-a-symbolic-link/)
- [libcuda.so.1 is not a symbolic link #5548](https://github.com/microsoft/WSL/issues/5548)
