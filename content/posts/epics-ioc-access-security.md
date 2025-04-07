---
title: "EPICS IOC 访问安全"
date: 2024-03-18T09:52:05+08:00
draft: false
description: EPICS IOC 数据访问安全配置
tags: ["linux", "EPICS"]
keywords: ["linux", "EPICS"]
categories: ["EPICS"]
---

原文：[IOC Access Security](https://docs.epics-controls.org/en/latest/appdevguide/AccessSecurity.html)

### 功能

访问安全功能用于保护IOC数据库，限制来自未经授权的CA或pvAccess客户端访问。访问安全性基于以下几点：

**Who** 客户端的用户ID（Channel Access/pvAccess）。  
**Where** 用户登录的主机 ID。客户端运行的主机，但不会分辨用户是本地用户或远程登录到主机的用户。  
**What** 记录的各个字段都受到保护。每条记录都有一个字段包含记录的访问安全组（ASG）。每个字段都有一个访问安全级别（ASL0或ASL1）。安全级别在记录定义文件（.dbd）中定义。  
**When** 访问规则可以包含类似于CALC Record的输入计算。

### 定义

**ASL** 访问安全级别  
**ASG** 访问安全组  
**UAG** 用户访问组  
**HAG** 主机访问组

### 快速上手

为了启用特定 IOC 的访问安全性，需要完成以下操作：

- 创建访问安全文件（.acf）
- 可能需要修改IOC数据库

记录实例可能需要设置访问安全组`ASG`字段。如果`ASG`为空，记录将会使用“`DEFAULT`”访问安全组。

访问安全文件可以在`iocInit`之后通过`asSubInit`和`asSubProcess`作为关联的子程序重新加载。将值*1*写入此记录将导致重新加载。

必须启动脚本在的`iocInit`之前包含以下命令：

``` c { title="st.cmd" }
asSetFilename("/full/path/to/accessSecurityFile")
/* 下面是一个可选命令 */
/* 使用宏替换 */
asSetSubstitutions("var1=sub1,var2=sub2,...")
```

如果在`iocInit`之前未执行`asSetFilename`，就不会启用访问安全限制。

如果给定`asSetFilename`，但在首次初始化访问安全性时发生错误，则对该IOC的所有访问都会被拒绝。

成功启动访问安全性后，尝试重新启动时出现错误，将会保持上次的访问安全配置。

启动IOC并启用访问安全后，可以通过`asSetFilename`、`asSetSubstitutions`和`asInit`来更改访问安全规则。也可以使用函数`asInitialize`、`asInitFile`和`asInitFP`。  
**在启动IOC之后重新初始化访问安全配置操作是“非常昂贵”的操作，尽量不要这样做。**

### 访问安全配置文件

本节介绍包含用户访问组（UAG）、主机访问组（HAG）和访问安全组（ASG）。IOC会读取访问配置文件（建议使用扩展名.acf）然后创建访问配置数据库。首先给出一个简单的例子，然后是完整的语法描述。

**简单示例**

``` c { title="accessSecurityFile.acf" }
UAG(uag) {user1,user2}
HAG(hag) {host1,host2}
ASG(DEFAULT) {
        RULE(1,READ)
        RULE(1,WRITE) {
                UAG(uag)
                HAG(hag)
       }
}
```

上面的规则提供了无限制的读权限（READ），而位于主机`host1`和`host2`上的用户`user1`和`user2`则拥有写权限（WRITE）。

**语法定义**

在以下描述中：

`[]` 可选项  
`|` 备选项  
`...` 任意数量的定义

元素`<name>`、`<user>`、`<host>`、`<pvname>`和`<calculation>`可以是带引号或不带引号的字符串。

``` c { title="accessSecurityFile.acf" }
UAG(<name>) [{ <user> [, <user> ...] }]
...
HAG(<name>) [{ <host> [, <host> ...] }]
...
ASG(<name>) [{
    [INP<index>(<pvname>)
    ...]
    RULE(<level>,NONE | READ | WRITE [, NOTRAPWRITE | TRAPWRITE]) {
        [UAG(<name> [,<name> ...])]
        [HAG(<name> [,<name> ...])]
        CALC(<calculation>)
    }
    ...
}]
...
```

**UAG**：用户访问组。这是用户名列表，列表可以空。一个用户名可以出现在多个UAG中。用户名必须和运行CA客户端的主机上的用户名相同。对于vxWorks客户端，用户名通常取自引导参数的用户字段。

**HAG**：主机访问组。这是主机名列表，列表可以空。同一主机名可以出现在多个HAG中。主机名必须和运行CA客户端的主机主机名相同。对于vxWorks客户端，主机名通常取自引导参数的目标名称。

**ASG**：访问安全组。`DEFAULT`是默认的访问安全组。

**INP`<index>`**：`index`必须是A到L中的一个值。类似于CALC record的INP字段。如果在ASG的规则中定义了CALC字段，则需要INP字段。

**RULE**：定义访问权限`<level>`必须为`0`或`1`。级别`1`字段的权限继承了级别`0`字段的权限。权限为`NONE`、`READ`和`WRITE`，`WRITE`也继承了`READ`权限。标准EPICS记录类型的所有字段除`VAL`、`CMD`（命令）和`RES`（重置）外都设置为1级。可选参数指定是否应捕获写入，如果未给定，则默认为`NOTRAPWRITE`。

`UAG`指定可以访问的用户访问组列表。如果未定义UAG，则允许所有用户访问。

`HAG`指定具有访问权限的主机访问组列表。如果未定义HAG，则允许所有主机访问。

`CALC`与计算记录的`CALC`字段类似，但结果必须计算为`TRUE`或`FALSE`。只有当计算结果为`TRUE`才适用该规则（RULE），其中实际测试对于（0.99 < result < 1.01）为`TRUE`。任何其他结果都被认为`FALSE`，并将导致该规则被忽略。

可以为ASG定义多条RULE，相同的RULE级别和访问权限也可以有多个。用于客户端的TRAPWRITE设置由通过规则检查的第一个WRITE规则确定。

每个记录类型的字段都有一个关联的访问安全级别`ASL0`或`ASL1`（默认值）。操作员通常更改的字段被分配为`ASL0`，其他字段被分配给`ASL1`。例如，模拟输出记录的`VAL`字段被分配为`ASL0`，其他字段分配为`ASL1`。这是因为在正常操作过程中只应修改`VAL`字段。

创建或修改访问配置文件后，可以使用`ascheck`命令查找语法错误：

``` shell
ascheck -S "xxx=yyy,..." < "filename"
```

`-S`表示使用宏替换。此命令会显示语法错误的位置，正确则不会有任何输出。

### 实验

首先新建一个示例IOC。

``` shell
$ mkdir example
$ cd example/
$ makeBaseApp.pl -t example test
$ makeBaseApp.pl -i -t example test

The following target architectures are available in base:
    linux-loong64
    linux-x86_64
What architecture do you want to use? linux-x86_64
The following applications are available:
    test
What application should the IOC(s) boot?
The default uses the IOC's name, even if not listed above.
Application name? test

$ make
```

然后创建访问安全配置文件*accessSecurity.acf*。

``` shell
cd iocBoot/ioctest/
touch accessSecurity.acf
```

修改配置文件内容，示例：

``` c { title="accessSecurityFile.acf" }
UAG(read) {deepin}
UAG(write) {deepin}
HAG(hosts) {LAPTOP-CTDCXXXX, 172.19.176.1}

ASG(DEFAULT) {
	RULE(1,READ)
	RULE(1,WRITE) {
		HAG(hosts)
	}
}
ASG(deepin) {
	RULE(1,READ) {
		UAG(read,write)
		HAG(hosts)
	}
	RULE(1,WRITE,TRAPWRITE) {
		UAG(write)
		HAG(hosts)
	}
}
```

稍微解释一下：  
创建了两个**用户访问组**（UAG），名称为read和write，两个用户访问组都只包含用户*deepin*。  
创建了一个**主机访问组**（HAG），名称为hosts，包含主机名*LAPTOP-CTDCXXXX*和一个IP地址。  
创建了默认（DEFAULT）**访问安全组**（ASG），不限制读取（READ）权限，只有hosts主机访问组的用户拥有写入（WRITE）权限。  
创建了**访问安全组**（ASG），名称为deepin，hosts主机访问组所包含主机上的deepin用户才拥有读取（READ）和写入（WRITE）权限。

可以使用`ascheck`工具检查一下语法是否正确。

``` shell
ascheck accessSecurity.acf
```

然后还可以修改一下db文件，例：

``` shell
cd example/db/
vi dbExample2.db
```

``` diff
record(ai, "$(user):aiExample$(no)")
{
	field(DESC, "Analog input No. $(no)")
	field(INP, "$(user):calcExample$(no).VAL NPP NMS")
	field(EGUF, "10")
	field(EGU, "Counts")
	field(HOPR, "10")
	field(LOPR, "0")
	field(HIHI, "8")
	field(HIGH, "6")
	field(LOW, "4")
	field(LOLO, "2")
	field(HHSV, "MAJOR")
	field(HSV, "MINOR")
	field(LSV, "MINOR")
	field(LLSV, "MAJOR")
+	field(ASG, "deepin")
}
alias("$(user):aiExample$(no)","$(user):ai$(no)")
```

这里指定`$(user):aiExample$(no)` record使用 *deepin* **访问安全组**。

最后，修改*st.cmd*来启用访问安全配置功能。

``` shell
cd example/iocBoot/ioctest/
vi st.cmd
```

``` diff
  #- Run this to trace the stages of iocInit
  #-traceIocInit

+ #- Set asCheckClientIP=1 to translate hostnames into IPs
+ var asCheckClientIP 1
+ asSetFilename("${TOP}/iocBoot/${IOC}/accessSecurity.acf")

  cd "${TOP}/iocBoot/${IOC}"
  iocInit
```

启动IOC。

``` shell
cd example/iocBoot/ioctest/
./st.cmd

#!../../bin/linux-x86_64/test
< envPaths
epicsEnvSet("IOC","ioctest")
epicsEnvSet("TOP","/home/deepin/example")
epicsEnvSet("EPICS_BASE","/usr/local/epics/base-7.0.8")
epicsEnvSet("EPICS_HOST_ARCH", "linux-x86_64")
cd "/home/deepin/example"
## Register all support components
dbLoadDatabase "dbd/test.dbd"
test_registerRecordDeviceDriver pdbbase
## Load record instances
dbLoadTemplate "db/user.substitutions"
dbLoadRecords "db/testVersion.db", "user=deepin"
dbLoadRecords "db/dbSubExample.db", "user=deepin"
asSetFilename("/home/deepin/example/iocBoot/ioctest/accessSecurity.acf")
cd "/home/deepin/example/iocBoot/ioctest"
iocInit
Starting iocInit
############################################################################
## EPICS R7.0.8
## Rev. 2024-03-01T16:27+0800
## Rev. Date build date/time:
############################################################################
iocRun: All initialization complete
## Start any sequence programs
#seq sncExample, "user=deepin"
epics>
```

然后打开一个新的终端窗口进行测试。  
注意，我这里的主机名是*LAPTOP-CTDCXXXX*，主机有两个用户*deepin*和*root*。  
分别使用两个用户访问使用默认（DEFAULT）和名为deepin**访问安全组**的变量。

``` shell
deepin@LAPTOP-CTDCXXXX:~$ caget deepin:circle:angle
deepin:circle:angle            186
deepin@LAPTOP-CTDCXXXX:~$ caget deepin:aiExample1
deepin:aiExample1              6
```

用户*deepin*对使用不同**访问安全组**的变量都可以访问。

``` shell
# 使用root用户
deepin@LAPTOP-CTDCXXXX:~$ sudo su

root@LAPTOP-CTDCXXXX:/home/deepin# caget deepin:circle:angle
deepin:circle:angle            55
root@LAPTOP-CTDCXXXX:/home/deepin# caget deepin:aiExample1
Read operation timed out: some PV data was not read.
deepin:aiExample1              *** no read access
```

用户*root*可以访问使用默认（DEFAULT）**访问安全组**的变量，而不可访问使用名为deepin**访问安全组**的变量。

测试结果与编写的访问安全配置规则相符合，说明访问安全配置成功。

不过，此次实验只测试了本机上的不同用户，对于更为复杂的控制系统的数据安全访问权限，则需要做更完善的安全配置。
