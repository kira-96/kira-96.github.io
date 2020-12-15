---
title: 一些基本没什么用的MFC技巧
date: 2020-04-24T14:35:30+08:00
draft: false
description: MFC小技巧
tags: [ "MFC" ]
keywords: [ "MFC" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 前言

由于我很少用MFC，只有工作上需要的时候才会用到，所以我也是个新手，遇到问题需要到网上找很久资料。这里只是记录一些特殊情景下会用到的技巧，方便以后查找。

## 隐藏窗口任务栏图标

这个问题我在网上找了很久，大致有两种方案：

**修改窗口的扩展样式**

``` cpp
ModifyStyleEx(WS_EX_APPWINDOW,WS_EX_TOOLWINDOW);
```

但是这样做会导致整个窗口的样式会变得很难看。

**将一个隐藏窗口设置成主窗口的父窗口**

这样做比较麻烦，而且任务视图下也不能再看到窗口，显然不是我想要的效果。

最后终于找到了一个比较完美的解决方案，通过COM的方式移除任务栏列表中的图标。

``` cpp
// 显示/隐藏任务栏图标(COM方式)
bool ShowInTaskbar(HWND hWnd, bool isShow)
{
	CoInitialize(nullptr);

	ITaskbarList* pTaskbarList;
	HRESULT hr = CoCreateInstance(CLSID_TaskbarList, nullptr, CLSCTX_INPROC_SERVER, IID_ITaskbarList, (void**)&pTaskbarList);

	if (SUCCEEDED(hr))
	{
		pTaskbarList->HrInit();

		if (isShow)
		{
			pTaskbarList->AddTab(hWnd);
		}
		else
		{
			pTaskbarList->DeleteTab(hWnd);
		}

		CoUninitialize();

		return true;
	}

	CoUninitialize();
	return false;
}
```

## 程序启动时默认隐藏窗口

一种比较简单的方法是在程序启动时将窗口移动到屏幕外的不可见区域。
如果直接在`OnInitDialog`中设置`ShowWindow(SW_HIDE)`是无效的，因为此时窗口还没有显示出来，自然也无法隐藏。

这里的思路是在程序启动的时候先将窗口移动到屏幕外，然后通过另一个线程将窗口隐藏起来，这样做虽然程序启动后窗口还是会一闪即逝，但由于是在屏幕之外，实际上并不能看到，然后在窗口需要显示的时候调用`ShowWindow(SW_SHOW)`即可。

``` cpp
// CxxxDlg.h 头文件
#include <future>

class CxxxDlg : public CDialogEx
{
    ...
    ...
    ...

private:
    std::future<int> hideTask;  // 后台隐藏窗口线程
}
```

``` cpp
BOOL CxxxDlg::OnInitDialog()
{
    ...

    CRect rcClient;
	GetWindowRect(&rcClient);
    // 将窗口移动到屏幕外
	MoveWindow(-rcClient.Width(), rcClient.top, rcClient.Width(), rcClient.Height());
	// 新建线程，延时1s后隐藏窗口
	hideTask = std::async(
		std::launch::async, [&] {
		std::this_thread::sleep_for(std::chrono::seconds(1));
		// ShowWindow(SW_HIDE);
		ShowWindowAsync(m_hWnd, SW_HIDE);
		std::this_thread::sleep_for(std::chrono::seconds(1));
		// 延时1s，待窗口完全隐藏后再将窗口居中
		CenterWindow();
		return 0;
	});

    ...
	return TRUE;  // 除非将焦点设置到控件，否则返回 TRUE
}
```

## 点击关闭时隐藏窗口

有时候我们希望在点击主窗口的关闭按钮之后将窗口隐藏或者最小化，而不是退出程序。这时只需要拦截掉窗口的关闭消息即可。

``` cpp
// CxxxDlg.h 头文件
class CxxxDlg : public CDialogEx
{
    ...

protected:
	afx_msg void OnSysCommand(UINT nID, LPARAM lParam);

    ...
}

void CxxxDlg::OnSysCommand(UINT nID, LPARAM lParam)
{
	/////////////////////////////////////
	// 这里捕获窗口的关闭消息
	// 不直接关闭窗口，而是隐藏起来
	if (nID == SC_CLOSE)
	{
		ShowWindowAsync(m_hWnd, SW_HIDE);
	}
	/////////////////////////////////////
	else
	{
		CDialogEx::OnSysCommand(nID, lParam);
	}
}
```

这样就可以拦截掉窗口的关闭消息了，这样不仅仅是点击关闭按钮时会隐藏窗口，通过窗口菜单关闭窗口或是使用`Alt+F4`都不能真正关闭窗口。那么我怎样才能退出程序呢，总不能用任务管理器吧。

其实很简单，在需要退出程序的时候向窗口发送`WM_CLOSE`消息即可。

``` cpp
SendMessage(WM_CLOSE);
```

## `CFileDialog`导致`CDialogEx`“失去焦点”的解决方法

继承自`CDialogEx`的窗口在使用`CFileDialog`之后会导致窗口标题栏变成灰色，很像是窗口失去了焦点，此时窗口仍然能够正常操作，但即使鼠标点击在窗口上窗口的标题栏依旧是灰色，无法恢复到窗口激活状态的颜色，即使将窗口属性设置为`WS_EX_TOPMOST`（置顶）依旧是这样，必须点击窗口外的其它区域或者使用Tab切换一下才能恢复正常。

而继承自`CDialog`的窗口则没有这个问题，可以将窗口的基类改成`CDialog`来避免这个问题。对于我这样的强迫症来说是无法忍受的，所以用尽千方百计终于找到了一个可行的解决方案。

经过尝试，单纯让窗口获取焦点或者将窗口放到前台的方法都是无效的。

``` cpp
CFileDialog dlg(
	TRUE, _T("ini"), nullptr,
	OFN_HIDEREADONLY | OFN_OVERWRITEPROMPT,
	_T("ini(*.ini)|*.ini|TEXT(*.txt)|*.txt|"), this);

auto dlgResult = dlg.DoModal();

// 无效的方法
this->SetFocus();
this->SetForegroundWindow();

...
```

所以只能曲线救国，既然通过手动切换的方式可以恢复到正常状态，不妨先切换到其它窗口再切换回来。

``` cpp
CFileDialog dlg(
	TRUE, _T("ini"), nullptr,
	OFN_HIDEREADONLY | OFN_OVERWRITEPROMPT,
	_T("ini(*.ini)|*.ini|TEXT(*.txt)|*.txt|"), this);

auto dlgResult = dlg.DoModal();

// 先将焦点放到桌面，再切换回本窗口
::SetForegroundWindow(::GetDesktopWindow());
this->SetForegroundWindow();

...
```

完美解决了问题。

## 窗口启用视觉样式

启用视觉样式之后，可以让程序看起来更加现代化一些，只支持Window XP以后的系统。
可以通过为程序添加清单文件来实现，不过比较麻烦。在VC++ 2005之后，直接添加编译器指令到代码中就可以了。

在预编译头文件中添加下面代码即可。

``` cpp
#if defined _M_IX86
#pragma comment(linker,"/manifestdependency:\"type='win32' name='Microsoft.Windows.Common-Controls' version='6.0.0.0' processorArchitecture='x86' publicKeyToken='6595b64144ccf1df' language='*'\"")
#elif defined _M_IA64
#pragma comment(linker,"/manifestdependency:\"type='win32' name='Microsoft.Windows.Common-Controls' version='6.0.0.0' processorArchitecture='ia64' publicKeyToken='6595b64144ccf1df' language='*'\"")
#elif defined _M_X64
#pragma comment(linker,"/manifestdependency:\"type='win32' name='Microsoft.Windows.Common-Controls' version='6.0.0.0' processorArchitecture='amd64' publicKeyToken='6595b64144ccf1df' language='*'\"")
#else
#pragma comment(linker,"/manifestdependency:\"type='win32' name='Microsoft.Windows.Common-Controls' version='6.0.0.0' processorArchitecture='*' publicKeyToken='6595b64144ccf1df' language='*'\"")
#endif
```

## 任务栏显示进度

为窗口的任务栏图标添加进度显示，也可以为任务栏添加按钮。

``` cpp
ITaskbarList3* pTaskbar;

// 初始化COM组件
CoInitialize(NULL);
CoCreateInstance(CLSID_TaskbarList, NULL, CLSCTX_INPROC_SERVER, IID_PPV_ARGS(&pTaskbar));

// TBPF_NOPROGRESS	= 0,        // 正常状态，不显示进度
// TBPF_INDETERMINATE	= 0x1,  // 忙碌状态，不显示进度
// TBPF_NORMAL	= 0x2,          // 正常状态，显示进度（绿色）
// TBPF_ERROR	= 0x4,          // 错误状态，显示进度（红色）
// TBPF_PAUSED	= 0x8           // 停止状态，显示进度（黄色）
pTaskbar->SetProgressState(GetSafeHwnd(), TBPF_NORMAL);
pTaskbar->SetProgressValue(GetSafeHwnd(), 60, 100);

// 设置提示信息
pTaskbar->SetThumbnailTooltip(GetSafeHwnd(), TEXT("Tooltip"));

// 设置覆盖图标
HICON hIcon = AfxGetApp()->LoadIcon(IDI_ICON_ERR);
pTaskbar->SetOverlayIcon(GetSafeHwnd(), hIcon, _T("Error"));

// 添加任务栏按钮
THUMBBUTTONMASK dwMask = THB_ICON | THB_TOOLTIP;
THUMBBUTTON buttons[3];
buttons[0].iId = 0;
buttons[0].dwMask = dwMask;
buttons[0].hIcon = hIcon;
memcpy(buttons[0].szTip, TEXT("Tooltip"), sizeof(buttons[0].szTip));
// ...
pTaskbar->ThumbBarAddButtons(GetSafeHwnd(), 3, buttons);

// 最后释放COM组件
CoUninitialize();
```

## 未完待续，持续更新中...

**参考**

[MFC简单的启动时隐藏界面方式(仅启动时隐藏)](https://blog.csdn.net/wgxh05/article/details/83415463)

[Enabling Visual Styles](https://docs.microsoft.com/zh-cn/windows/win32/controls/cookbook-overview)
