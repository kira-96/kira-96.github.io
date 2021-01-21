---
title: WPF和MFC进程间传递数据
date: 2019-05-27T21:54:45+08:00
draft: false
description: WPF和MFC进程间传递数据
tags: [ "WPF" , "MFC" ]
keywords: [ "进程通信" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 进程间传递数据的方法 ##

在进程间传递数据也就意味着两个不同的应用程序之间的通讯，大家可能会想到使用消息队列(Message Queue)来作为解决方案，当然这可能是最优解，然而这里我要讲的是另外一种方法，通过Windows的消息机制来传递数据，内容比较硬核。

依旧用到了两个Windows的API，`FindWindow`和`SendMessage`，以及WPF如何和MFC窗口通讯，可以参考[上一篇文章](https://kira-96.github.io/posts/wpf-ru-he-chu-liwindows-xiao-xi.html)。

## 传递数据的方式 ##

这里需要先知道一个Window消息`WM_COPYDATA`
它在`WinUser.h`中的定义如下

``` cpp
#define WM_COPYDATA 0x004A
```

这个消息就是这次要讲的内容，通过这个消息就可以在不同的窗口间传递数据了，它有两个参数，`WPARAM`是发送消息窗口的句柄，`LPARAM`是一个结构体的指针，这个结构体在`WinUser.h`里定义如下：

``` cpp
/*
 * lParam of WM_COPYDATA message points to...
 */
typedef struct tagCOPYDATASTRUCT {
    ULONG_PTR dwData;
    DWORD cbData;
    _Field_size_bytes_(cbData) PVOID lpData;
} COPYDATASTRUCT, *PCOPYDATASTRUCT;
```

嗯...看起来也不是很复杂，为了能够正确处理这个指针，需要在C#中也定义一个相同的结构体

``` csharp
/// <summary>
/// COPYDATASTRUCT
/// 对应 C++ 里的 COPYDATASTRUCT
/// 不能更改
/// </summary>
[StructLayout(LayoutKind.Sequential)]
public struct COPYDATASTRUCT
{
    public IntPtr dwData; //可以是任意值
    public int cbData;    //指定lpData内存区域的字节数
    public IntPtr lpData; //发送给目录窗口所在进程的数据
}
```

这样就可以了，其中最重要的就是`lpData`，它可以是任意对象的指针，只要C++和C#两边定义了一个相同的结构体，我们就可以用它来直接传递结构体对象:open_mouth:，这种方式比通过消息队列传递数据要快得多。

那么先来定义一个简单的结构体吧：

``` cpp
// C++
struct StructUser
{
	char UserName[32];
	char Password[32];

	StructUser()
	{
		ZeroMemory(UserName, 32);
		ZeroMemory(Password, 32);
	}
};
```

``` csharp
// C#
[StructLayout(LayoutKind.Sequential)]
public struct StructUser
{
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 32)]
    public string UserName;
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 32)]
    public string Password;
}
```

这里定义了一个User的结构体，`UserName`和`Password`都是长度为32的字符串，注意C#中结构体的定义方式，这里指定了字符串的长度，就是为了和C++的结构体保持一致。

先来看看C++中如何发送和处理`WM_COPYDATA`消息的：

``` cpp
// 发送
StructUser user;
char userName[32];
char password[32];
// GetDlgItem(IDC_EDIT_USERNAME)->GetWindowTextA(userName, 32);
// GetDlgItem(IDC_EDIT_PASSWORD)->GetWindowTextA(password, 32);
memcpy(user.UserName, userName, 32);
memcpy(user.Password, password, 32);

COPYDATASTRUCT copyData;
copyData.dwData = 0;
copyData.lpData = &user;
copyData.cbData = sizeof(StructUser);

HWND hWnd = nullptr;
if (m_pTargerWnd == nullptr)
{
	hWnd = ::FindWindow(nullptr, "WpfWindow");
}
else
{
	hWnd = m_pTargerWnd->GetSafeHwnd();
}
::SendMessage(hWnd, WM_COPYDATA, (WPARAM)GetSafeHwnd(), (LPARAM)& copyData);

// 接收
/* C++ 中相关代码
 * 处理 WM_COPYDATA 消息
 * Header File(.h)
---------------------------------------------------------------------
...
afx_msg BOOL OnCopyData(CWnd *pWnd, COPYDATASTRUCT *pCopyDataStruct);
...
DECLARE_MESSAGE_MAP()
---------------------------------------------------------------------
* Source File(.cpp)
BEGIN_MESSAGE_MAP(CxxxDlg, CDialogEx)
    ...
    ON_WM_COPYDATA()
    ...
    END_MESSAGE_MAP()
...
 */

BOOL CxxDlg::OnCopyData(CWnd* pWnd, COPYDATASTRUCT* pCopyDataStruct)
{
	if (pWnd != nullptr)
	{
		m_pTargerWnd = pWnd;
	}
	if (pCopyDataStruct != nullptr)
	{
		StructUser* pUser = (StructUser*)(pCopyDataStruct->lpData);
		// DWORD dwLen = pCopyDataStruct->cbData;

		// GetDlgItem(IDC_EDIT_USERNAME)->SetWindowTextA(pUser->UserName);
		// GetDlgItem(IDC_EDIT_PASSWORD)->SetWindowTextA(pUser->Password);
	}

	return CDialogEx::OnCopyData(pWnd, pCopyDataStruct);
}
```

C#会比较复杂一些

``` csharp
// 消息处理函数
private IntPtr WndProcFunc(IntPtr hwnd, int msg, IntPtr wParam, IntPtr lParam, ref bool handled)
{
    switch (msg)
    {
        case WM_COPYDATA:  // public const int WM_COPYDATA = 0x004A;
            IntPrt hWnd_Target = wParam;
            COPYDATASTRUCT param = Marshal.PtrToStructure<COPYDATASTRUCT>(lParam);
            StructUser user = Marshal.PtrToStructure<StructUser>(param.lpData);
            // UserName.Text = user.UserName;
            // Password.Text = user.Password;
            break;
        default:
            break;
    }

    return IntPtr.Zero;
}

// 发送消息
StructUser sctUser = new StructUser()
{
    UserName = UserName.Text,
    Password = Password.Text
};

IntPtr userPtr = Marshal.AllocHGlobal(Marshal.SizeOf<StructUser>());
Marshal.StructureToPtr<StructUser>(sctUser, userPtr, true);

COPYDATASTRUCT copyData = new COPYDATASTRUCT()
{
    dwData = IntPtr.Zero,
    cbData = Marshal.SizeOf<StructUser>(),
    lpData = userPtr,
};

IntPtr copyDataPtr = Marshal.AllocHGlobal(Marshal.SizeOf<COPYDATASTRUCT>());
Marshal.StructureToPtr<COPYDATASTRUCT>(copyData, copyDataPtr, true);

if (hWnd_Target == IntPtr.Zero)
    hWnd_Target = Win32Api.FindWindow(null, "MfcWindow");

hWnd_MainWnd = new WindowInteropHelper(this).Handle;

if (hWnd_Target != IntPtr.Zero)
    Win32Api.SendMessage(hWnd_Target, WM_COPYDATA, hWnd_MainWnd, copyDataPtr);

Marshal.FreeHGlobal(userPtr);     //
Marshal.FreeHGlobal(copyDataPtr); // 最后一定要释放掉非托管内存
```

处理消息的地方和上一篇一样，发送的地方比较复杂，总体来讲就是用`Marshal`开辟了两块非托管的内存来存放两个结构体的数据，等到消息被处理之后再将非托管内存释放掉，避免内存泄漏。

那么，这次的内容就到这里了，主要内容都是代码，但其实也不复杂，只是需要慢慢消化。
