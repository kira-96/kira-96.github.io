---
title: "weston桌面系统截屏方法"
date: 2023-02-03T11:34:14+08:00
draft: true
tags: ["linux"]
keywords: ["linux", "weston"]
categories: ["linux"]
---

使用`weston-screenshooter`

但必须启用weston桌面`--debug`选项，否则会出现以下错误：

```shell
[root@RK356X:/]# weston-screenshooter
[02:41:05.145] libwayland: error in client communication (pid 776)
weston_screenshooter@5: error 0: screenshooter failed: permission denied. Debug protocol must be enabled
```

以RK3568开发板，buildroot系统为例，修改`/etc/init.d/S50launcher`，找到weston所在行，添加`--debug`选项。

```shell
......
# Uncomment to disable mirror mode
# unset WESTON_DRM_MIRROR

export XDG_RUNTIME_DIR=${XDG_RUNTIME_DIR:-/var/run}
export QT_QPA_PLATFORM=${QT_QPA_PLATFORM:-wayland}

weston --tty=2 --debug --idle-time=0&
{
    # Wait for weston ready
    while [ ! -e ${XDG_RUNTIME_DIR}/wayland-0 ]; do
        sleep .1
    done
    /usr/bin/QLauncher &
}&
......
```

forlinx开发板使用的yocto系统也类似，修改`/lib/systemd/system/weston.service`，在weston后添加`--debug`选项。

```shell
$ vi /lib/systemd/system/weston.service
# 修改如下
# ExecStart=/usr/bin/weston --debug --log=${XDG_RUNTIME_DIR}/weston.log $OPTARGS
```

然后重启系统，之后就可以使用`weston-screenshooter`截取屏幕了。

**链接**

- [wayland-project/weston](https://github.com/wayland-project/weston)
- [weston.ini配置文件](https://zhuanlan.zhihu.com/p/396168706)
