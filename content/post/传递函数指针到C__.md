---
title: C#传递函数指针到C++
date: 2020-07-17T14:30:30+08:00
draft: false
description: C#传递函数指针到C++
tags: [ "C#", "C++" ]
keywords: [ "C#", "C++" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 简介

在编码的时候难免会遇到不同编程语言之间的接口调用，其中最通用的就是`C`的动态链接库，几乎所有语言都可以调用`C`的接口函数。那么这种关系能否反过来呢？用`C`调用其它的函数接口，当然也是可以的，只需要将函数指针作为参数传递进去就可以了。

C#中使用`Delegate`来表示函数指针。

## 准备工作

首先新建一个C++的动态库，随便定义一个函数指针类型，以及一个导出函数。大致内容如下：

``` cpp
#include "pch.h"
#include <iostream>

using namespace std;

#ifdef DYNLIB_EXPORTS
#define DLL_API extern "C" __declspec(dllexport)
#else
#define DLL_API extern "C" __declspec(dllimport)
#endif

typedef struct
{
    int Left;
    int Top;
    int Right;
    int Bottom;
} MyRect, * MyRectPtr;

// 函数指针
typedef VOID(CALLBACK* PRINTCALLBACK)(MyRectPtr);

BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}

// 导出函数，使用上面定义的函数指针类型作为参数
DLL_API VOID Print(MyRectPtr pRect, PRINTCALLBACK callback)
{
    if (callback == NULL)
    {
        cout << '\t' << pRect->Top << endl;
        cout << pRect->Left << "\t\t" << pRect->Right << endl;
        cout << '\t' << pRect->Bottom << endl;
    }
    else
    {
        callback(pRect);
    }
}
```

准备好了C++的部分，再来写C#部分的代码：

首先定义一个和C++部分相同的`MyRect`结构体和函数指针类型`delegate`

``` csharp
// 对应C++部分的MyRect
[StructLayout(LayoutKind.Sequential)]
struct MyRect
{
    public int Left;
    public int Top;
    public int Right;
    public int Bottom;
}

// 对应C++部分的 PRINTCALLBACK
[UnmanagedFunctionPointer(CallingConvention.StdCall)]
delegate void PrintRect(ref MyRect myRect);
```

这样准备工作就做完了，一定要保证C++部分和C#部分定义的数据类型和接口一致，不然调用时会出问题的。

## 调用

导入C++动态库的函数入口

``` csharp
[DllImport("DynLib.dll", EntryPoint = "#1", CallingConvention = CallingConvention.Cdecl)]
public static extern void PrintInCpp(ref MyRect pRect, PrintRect callback);
```

然后定义好C#这边的`delegate`实例，就完成了。

``` csharp
static void Print(ref MyRect myRect)
{
    Console.WriteLine("C# Print func called.");
    Console.WriteLine("[({0},{1}),({2},{3})]", myRect.Left, myRect.Top, myRect.Right, myRect.Bottom);
}

static void Main()
{
    var rect = new MyRect()
    {
        Left = 100,
        Top = 100,
        Right = 220,
        Bottom = 200
    };

    PrintInCpp(ref rect, null);  // callback为null
    PrintInCpp(ref rect, new PrintRect(Print));

    Console.WriteLine("Press any key exit...");
    Console.ReadKey(true);
}
```

这里调用了两次接口函数，为了方便看出区别，在`callback`参数为`NULL`时，会由c++打印结果，否则c++会调用外部的函数接口。

输出结果：

``` bash
./delegatefunc
        100
100             220
        200
C# Print func called.
[(100,100),(220,200)]
Press any key exit...
```
