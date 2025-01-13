---
title: "EPICS IOC 使用 SNCSEQ"
date: 2025-01-13T10:45:01+08:00
lastmod: 2025-01-13T10:46:12+08:00
draft: true
description: EPICS IOC 使用 SNCSEQ 过程记录
tags: ["linux", "EPICS"]
keywords: ["linux", "EPICS"]
categories: ["EPICS"]
---

IOC 配置`SNCSEQ`：
``` toml
# example/configure/RELEASE.local
EPICS_BASE = /path/to/your/build/of/epics-base
SUPPORT = ${EPICS_BASE}/../epics-modules
PVXS = ${SUPPORT}/pvxs
# 添加 SNCSEQ
SNCSEQ = ${SUPPORT}/sequencer
```

example工程中相关的代码：
``` cpp
# example/iocExampleApp/src/Makefile
...

# Link in the code from our support library
iocExample_LIBS += iocExampleSupport

# To build SNL programs, SNCSEQ must be defined
# in the <top>/configure/RELEASE file
ifneq ($(SNCSEQ),)
    # Build sncExample into iocExampleSupport
    sncExample_SNCFLAGS += +r
    iocExample_DBD += sncExample.dbd
    # A .stt sequence program is *not* pre-processed:
    iocExampleSupport_SRCS += sncExample.stt
    iocExampleSupport_LIBS += seq pv
    iocExample_LIBS += seq pv

    # Build sncProgram as a standalone program
    PROD_HOST += sncProgram
    sncProgram_SNCFLAGS += +m
    # A .st sequence program *is* pre-processed:
    sncProgram_SRCS += sncProgram.st
    sncProgram_LIBS += seq pv
    sncProgram_LIBS += $(EPICS_BASE_HOST_LIBS)
endif

...
```

``` cpp
# sncExample.dbd

# The name below is derived from the name of the SNL program
# inside the source file, not from its filename. Here the
# program is called sncExample, but is compiled in both the
# sncExample.stt and sncProgram.st source files.
registrar(sncExampleRegistrar)
```

``` cpp
# sncProgram.st

#include "../sncExample.stt"
```

``` python
# sncExample.stt

program sncExample
double v;
assign v to "{user}:aiExample";
monitor v;

ss ss1 {
    state init {
        when (delay(10)) {
            printf("sncExample: Startup delay over\n");
        } state low
    }
    state low {
        when (v > 5.0) {
            printf("sncExample: Changing to high\n");
        } state high
    }
    state high {
        when (v <= 5.0) {
            printf("sncExample: Changing to low\n");
        } state low
    }
}
```

修改启动脚本，在`IOC`启动时运行SNL程序：
``` sh
...

## Start any sequence programs
seq sncExample, "user=$(USER)"
```

编译：
``` sh
make -j 8
```

此时查看`example/dbd/iocExample.dbd`，应该可以看到：
``` cpp
...
registrar(sncExampleRegistrar)
...
```
