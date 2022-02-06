---
title: "交叉编译适用于 iMX6UL 的 x11vnc"
date: 2022-02-06T11:06:12+08:00
draft: false
tags: [ "Linux", "VNC" ]
keywords: [ "编程" ]
isCJKLanguage: true
enableDisqus: true
---

## 前言

由于这半年来一直在做嵌入式Linux系统软件开发工作，所以经常和嵌入式设备打交道，最早接触的嵌入式Linux应该就是树莓派了，而我的树莓派一般也不接屏幕，基本上都使用VNC远程连接，所以就想着能不能把VNC也移植到嵌入式设备上，最后找到了`x11vnc`。

VNC（虚拟网络计算）是一种非常有用的网络图形协议（应用程序在一台计算机上运行，但在另一台计算机上显示其窗口），但与X不同，查看端非常简单，不保持任何状态。它是一种远程帧缓冲区（RFB）协议。

x11vnc允许用户通过任何VNC viewer远程查看并与real X显示器（即与物理监视器、键盘和鼠标相对应的显示器）交互。

## 准备工作

需要先用git克隆下面两个仓库，`libvncserver`和`x11vnc`。
`x11vnc`是基于`libvncserver`的服务端程序。

- [GitHub - LibVNC/libvncserver: LibVNCServer/LibVNCClient are cross-platform C libraries that allow you to easily implement VNC server or client functionality in your program.](https://github.com/LibVNC/libvncserver.git)
- [GitHub - LibVNC/x11vnc: a VNC server for real X displays](https://github.com/LibVNC/x11vnc.git)

## 编译 libvncserver

编译x11vnc需要libvncserver，libvncserver按照 [README](https://github.com/LibVNC/libvncserver/blob/master/README.md) 编译即可

```shell
mkdir build
cd build
cmake ..
cmake --build .
```

编译完成后将build文件夹下生成的.so文件复制到 `sysroot`

例：

```shell
# 复制 .so 文件
$ sudo cp ./libvncclient.so* /opt/fsl-imx-x11/4.1.15-2.0.0/sysroots/cortexa7hf-neon-poky-linux-gnueabi/usr/lib/
# 复制 pkgconfig 文件
$ sudo cp ./libvnc*.pc /opt/fsl-imx-x11/4.1.15-2.0.0/sysroots/cortexa7hf-neon-poky-linux-gnueabi/usr/lib/pkgconfig/
# 复制头文件
$ sudo cp -r ../rfb /opt/fsl-imx-x11/4.1.15-2.0.0/sysroots/cortexa7hf-neon-poky-linux-gnueabi/usr/include/
$ sudo cp ./rfb/rfbconfig.h /opt/fsl-imx-x11/4.1.15-2.0.0/sysroots/cortexa7hf-neon-poky-linux-gnueabi/usr/include/rfb/
```

## 编译 x11vnc

x11vnc 需要使用 autoconf 和 automake 生成 configure 和 makefile

```shell
$ cd x11vnc-master
# 使用aclocal工具生成aclocal.m4
$ aclocal
# 使用autoconf工具生成configure文件
$ autoconf
# 使用autoheader工具生成config.h.in文件
$ autoheader
# 使用automake生成Makefile.in文件
$ automake --add-missing
# configure 配置交叉编译，如果libvncserver没有正确编译安装，这里会提示找不到libvncserver
$ ./configure CC=arm-poky-linux-gnueabi-gcc AR=arm-poky-linux-gnueabi-ar AS=arm-poky-linux-gnueabi-as LD=arm-poky-linux-gnueabi-ld --host=arm-poky-linux --prefix=/home/ubuntu/ CFLAGS="-march=armv7ve -mfpu=neon -mfloat-abi=hard -mcpu=cortex-a7 --sysroot=/opt/fsl-imx-x11/4.1.15-2.0.0/sysroots/cortexa7hf-neon-poky-linux-gnueabi"
# 编译安装
$ make install
```

安装完成后在 `/home/ubuntu/` （configure 步骤设置的 --prefix）目录下出现了`bin`和 `share`两个目录，`bin`目录下的就是 x11vnc 的可执行文件。

## 安装运行

将 libvncserver 编译得到的 *.so 文件和 x11vnc 可执行文件复制到开发板即可。

```shell
# 复制可执行文件
$ scp -r /home/ubuntu/bin root@192.168.10.7:~/
# 复制 libvncserver
$ scp /opt/fsl-imx-x11/4.1.15-2.0.0/sysroots/cortexa7hf-neon-poky-linux-gnueabi/usr/lib/libvnc*.so.0.9.13 root@192.168.10.7:/usr/lib/
```

ssh登录开发板

```shell
# 创建符号链接
root@imx6ulevk:~# ln -s /usr/lib/libvncclient.so.0.9.13 /usr/lib/libvncclient.so.1
root@imx6ulevk:~# ln -s /usr/lib/libvncclient.so.1 /usr/lib/libvncclient.so
root@imx6ulevk:~# ln -s /usr/lib/libvncserver.so.0.9.13 /usr/lib/libvncserver.so.1
root@imx6ulevk:~# ln -s /usr/lib/libvncserver.so.1 /usr/lib/libvncserver.so
# 重命名文件夹
root@imx6ulevk:~# mv ~/bin ~/x11vnc 
root@imx6ulevk:~# cd ~/x11vnc
root@imx6ulevk:~# chmod +x ./x11vnc
# 运行 x11vnc
# x11vnc -display :0
root@imx6ulevk:~# ./x11vnc
```

然后在电脑上启动 [VNC Viewer](https://www.realvnc.com/en/connect/download/viewer/), 输入开发板的ip地址就可以通过远程桌面访问设备了。

实际运行效果如图

![Snipaste_2022-01-28_12-31-30.png](https://s2.loli.net/2022/02/06/IiSnRUGJ4MA9FQx.png)

**参考**

- [autoconf / automake工具使用介绍](https://blog.csdn.net/gulansheng/article/details/42683809)
- [给imx6嵌入式平台移植x11vnc搭建远程控制环境](https://blog.csdn.net/wanvan/article/details/86506718)
