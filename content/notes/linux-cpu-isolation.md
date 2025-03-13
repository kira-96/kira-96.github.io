---
title: "Linux CPU核心隔离"
date: 2025-03-13T14:59:45+08:00
draft: true
description: Linux CPU核心隔离技术
tags: ["linux"]
keywords: ["linux"]
categories: ["linux"]
---

CPU隔离主要是为了确保关键进程获得更高的性能，减少干扰。常见的隔离方法有cgroups、taskset、isolcpus内核参数，还有实时内核（PREEMPT RT）配置。还有CPU亲和性、IRQ屏蔽技术。

- `isolcpus`：在启动内核时隔离某些CPU核心，防止普通进程使用。但需要结合cgroups或taskset来分配进程到隔离核心。
- `nohz_full`：在隔离核心禁用时钟中断（Tickless 模式）。
- `rcu_nocbs`：将 RCU 回调任务移出隔离核心。

**修改内核启动参数**

主要参数：`ioslcpus=? nohz_full=? rcu_nocbs=?`

例：
`earlycon console=tty console=ttyS0,115200 isolcpus=1,2 nohz_full=1,2 rcu_nocbs=1,2 acpi=off rdinit=/sbin/init rootdelay=5 root=/dev/sda1`

**通过GRUB配置**

``` sh
# /etc/default/grub

# 添加isolcpus参数
GRUB_CMDLINE_LINUX="isolcpus=1,2 nohz_full=1,2 rcu_nocbs=1,2"
```

修改完成后更新GRUB配置：`sudo update-grub`

**指定进程CPU亲和性**

``` sh
# 启动时绑定到核心 1,2
taskset -a -c 1,2 ./st.cmd
# 修改运行中进程的 CPU 亲和性
taskset -cp <core_list> <pid>
```

**IgH EtherCAT驱动指定CPU亲和性**

在脚本中添加启动参数`run_on_cpu=?`，例：

``` shell
## /sbin/ethercatctl

...
# load master module
if ! ${MODPROBE} ${MODPROBE_FLAGS} ec_master \
    main_devices="${DEVICES}" backup_devices="${BACKUPS}" run_on_cpu=1;  
then
    exit 1
fi

...
```
