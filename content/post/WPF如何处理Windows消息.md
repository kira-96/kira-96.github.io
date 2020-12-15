---
title: WPF如何处理Windows消息
date: 2019-05-26T15:30:50+08:00
draft: false
description: 在WPF中处理窗口事件
tags: [ "C#" , "WPF" ]
keywords: [ "Window消息" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 发送消息到指定窗口 ##

发送消息相对来说比较简单，这里先讲，这里需要用到两个Windows的API

``` csharp
// 查找指定窗口
[DllImport("User32.dll", EntryPoint = "FindWindow", CharSet = CharSet.Auto)]
public static extern IntPtr FindWindow(string lpClassName, string lpWindowName);

//消息发送API
[DllImport("User32.dll", EntryPoint = "SendMessage", CharSet = CharSet.Auto)]
public static extern int SendMessage(
    IntPtr hWnd,        // 信息发往的窗口的句柄
    int Msg,            // 消息ID
    IntPtr wParam,      // 参数1
    IntPtr lParam       // 参数2
);
```

`FindWindow`就是根据窗口的名字去找到相应的窗口句柄，`SendMessage`就是发送消息到指定窗口了，写过MFC的应该不陌生

``` csharp
IntPtr hWnd = FindWindow(null, windowName);
if (hWnd == IntPtr.Zero)
    return;
SendMessage(hWnd, WM_USER + 2, IntPtr.Zero, "msg");
```

发送消息就讲这么多，剩下的需要自行摸索，下面是用WPF窗口接收消息

## 获取WPF窗口句柄 ##

Windows消息是通过窗口句柄来传递给指定窗口的，所以想要处理WPF窗口的收到消息，首先就需要获取自身的窗口句柄。

``` csharp
HwndSource hWnd = PresentationSource.FromVisual(this) as HwndSource;
```

或者

``` csharp
IntPtr hwnd = new WindowInteropHelper(this).Handle;
HwndSource source = HwndSource.FromHwnd(hwnd);
```

这里的`this`就是WPF的窗口，当然也可以通过这种方式获取窗口任意控件的句柄

## 添加钩子(AddHook) ##

得到窗口的句柄之后就可以为该窗口添加钩子来处理窗口收到的消息了

``` csharp
protected override void OnSourceInitialized(EventArgs e)
{
    base.OnSourceInitialized(e);

    // Add Hook
    // HwndSource source = PresentationSource.FromVisual(this) as HwndSource;
    // source.AddHook(WndProcFunc);

    // 或者

    HwndSource.FromHwnd(new WindowInteropHelper(this).Handle).AddHook(new HwndSourceHook(WndProcFunc));
}
```

或者

``` csharp
public MainWindow()
{
    InitializeComponent();
    this.SourceInitialized += (s, e) => {
        HwndSource.FromHwnd(new WindowInteropHelper(this).Handle).AddHook(new HwndSourceHook(WndProcFunc));
    };
}
```

这个`WndProcFunc`就是我们的消息处理函数了，窗口在收到消息之后就会走到这个函数里面，它看起来像是这样

``` csharp
public IntPtr WndProc(IntPtr hwnd, int msg, IntPtr wParam, IntPtr lParam, ref bool handled)
{
    // Handle Msg Here
    switch (msg)
    {
        case WM_USER + 1:  // 你的消息值
        {
            // 处理 wParam, lParam
            break;
        }
        // ... 其它消息
        default: break;
    }

    return IntPtr.Zero;
}
```

这样就完了吗？是的，但是！目前还不能处理传递的参数，如果WPF程序和C++程序处于一个进程还好说，如果是两个进程，那么他们之间的内存是不共用的，所以即使WPF窗口拿到了指针也读不出指针里的内容。

那么要怎样才能在WPF程序和C++程序之间传递值呢，有空在讲...
