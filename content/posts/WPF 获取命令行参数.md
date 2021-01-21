---
title: WPF 获取命令行参数
date: 2020-03-25T10:42:32+08:00
draft: false
description: WPF 获取命令行参数
tags: [ "WPF" ]
keywords: [ "WPF" ]
categories: [ "编程" ]
isCJKLanguage: true
---

虽然平时很少会用到命令行参数，但有时候可以使用命令行参数来使程序执行不同的行为。

在写控制台程序的时候，我们可以直接得到程序命令行参数。

``` csharp
static void Main(string[] args)
{
    // args 就是命令行参数
}
```

那么如果不是控制台程序如何获取命令行参数呢？

**在 WPF 中有两种方法获取命令行参数**

第一种方法是**重写应用的`OnStartup`方法**，通过`StartupEventArgs`来获取命令行参数。

``` csharp
// App.xaml.cs
/// <summary>
/// App.xaml 的交互逻辑
/// </summary>
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // 获取命令行参数
        string[] args = e.Args;

        // do something
    }
}
```

如果没有命令行参数，那么`args`就为`null`。

第二种方法则比较灵活，可以在任意地方获取到命令行参数。

``` csharp
string[] args = System.Environment.GetCommandLineArgs();
```

直接使用`Environment`的静态方法来获取命令行参数，需要注意的是，第二种方法获取到的参数和前面一种方法结果不同。

第二种方法获取的结果不会为`null`，通过`Environment`获取到的命令行参数第一个是**当前程序的路径**，从第2项开始才是命令行参数（如果有）。
