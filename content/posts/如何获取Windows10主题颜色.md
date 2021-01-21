---
title: 如何获取Windows10主题颜色
date: 2020-04-24T12:12:00+08:00
draft: false
description: 获取Windows10主题颜色
tags: [ "Win10主题色" ]
keywords: [ "Windows 10" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 前言

最近公司的系统也开始陆续向 Windows 10 迁移了，我的办公电脑也终于换上了新系统。为了适应新的开发环境，有时候需要获取一些系统相关的信息，这里就稍微总结一下。

这篇文章并不只是写如何获取Win10系统主题色，也会包含一些其它的内容，主要是用`C++`和`C#`语言，可能会持续更新。

## 检测是否为 Windows10 系统

**C++ 方式**

在Win10上已经不能直接通过`GetVersion`或`GetVersionEx`的方式获取系统信息，单纯使用这两个函数编译时会报错。
这里提供另外一种方式：

``` cpp
bool IsWindows10()
{
    typedef void(__stdcall*NTPROC)(DWORD*, DWORD*, DWORD*);

	HMODULE inst = LoadLibrary("ntdll.dll");
	NTPROC ntProc = reinterpret_cast<NTPROC>(GetProcAddress(inst, "RtlGetNtVersionNumbers"));

	DWORD dwMajor, dwMinor, dwBuildNumber;
	ntProc(&dwMajor, &dwMinor, &dwBuildNumber);

	FreeLibrary(inst);

	return dwMajor == 10;
}
```

当然上面的方法也适用于`C#`，但这里使用C#的方法来检测系统版本。

``` csharp
/// <summary>
/// Gets if the Operating System is Windows 10
/// </summary>
/// <returns>True if Windows 10</returns>
public static bool IsWindows10
{
    get
    {
        // IMPORTANT: Windows 8.1. and Windows 10 will ONLY admit their real version if your program's manifest
        // claims to be compatible. Otherwise they claim to be Windows 8. See the first comment on:
        // https://msdn.microsoft.com/en-us/library/windows/desktop/ms724833%28v=vs.85%29.aspx

        // Get Operating system information
        OperatingSystem os = Environment.OSVersion;

        // Get the Operating system version information
        Version vi = os.Version;

        // Pre-NT versions of Windows are PlatformID.Win32Windows. We're not interested in those.

        if (os.Platform == PlatformID.Win32NT)
        {
            if (vi.Major == 10)
            {
                return true;
            }
        }

        return false;
    }
}
```
使用上面的方法时需要注意，首先要为你的程序添加**清单文件**(app.manifest)，并且取消对Windows 10系统兼容的注释。
如：

``` xml
  ...
  <compatibility xmlns="urn:schemas-microsoft-com:compatibility.v1">
    <application>
      <!-- 设计此应用程序与其一起工作且已针对此应用程序进行测试的
           Windows 版本的列表。取消评论适当的元素，
           Windows 将自动选择最兼容的环境。 -->

      <!-- Windows Vista -->
      <!--<supportedOS Id="{e2011457-1546-43c5-a5fe-008deee3d3f0}" />-->

      <!-- Windows 7 -->
      <supportedOS Id="{35138b9a-5d96-4fbd-8e2d-a2440225f93a}" />

      <!-- Windows 8 -->
      <!--<supportedOS Id="{4a2f28e3-53b9-4441-ba9c-d69d4a4a6e38}" />-->

      <!-- Windows 8.1 -->
      <!--<supportedOS Id="{1f676c76-80e1-4239-95bb-83d0f6d0da78}" />-->

      <!-- Windows 10 -->
      <supportedOS Id="{8e0f7a12-bfb3-4fe8-b9a5-48fd50a15a9a}" />

    </application>
  </compatibility>
  ...
```

**是否为 Win7 以下版本**

``` csharp
public static bool IsWindows7OrLower
{
    get
    {
        Version v = Environment.OSVersion.Version;

        int versionMajor = v.Major;
        int versionMinor = v.Minor;
        double version = versionMajor + (double)versionMinor / 10;
        return version <= 6.1;
    }
}
```

## 获取 Window 10 主题色(Accent Color)

在**UWP**中可以轻易的获取`SystemAccentColor`。但随着Win10移动端的失利，UWP基本已被微软宣告死亡。
这里就讲一下如何通过其它方式读取到系统的主题色。

在Win10上，当系统的主题色发生变化时，系统会给所有窗口都发送主题色变更的消息

``` cpp
#define WM_DWMCOLORIZATIONCOLORCHANGED  0x0320
```

``` cpp
LRESULT CxxxDlg::OnColorizationColorChanged(WPARAM wParam, LPARAM lParam)
{
	DWORD color = 0;
	BOOL opaque = FALSE;

	HRESULT hr = DwmGetColorizationColor(&color, &opaque);
	if (SUCCEEDED(hr))
	{
		// Update the application to use the new color.
	}

	return 0;
}
```

但我使用之后发现，并不能正确的获取到系统的主题颜色，而是DWM颜色。
所以这里使用了另一种方式，通过读取注册表的方式获取系统主题色。

``` csharp
// \HKEY_CURRENT_USER\Software\Microsoft\Windows\DWM\AccentColor
public static Color GetSystemAccentColor()
{
    using (RegistryKey dwm = Registry.CurrentUser.OpenSubKey(@"Software\Microsoft\Windows\DWM", false))
    {
        if (dwm.GetValueNames().Contains("AccentColor"))
        {
            // 这里不要尝试转换成uint，因为有可能符号位为 1（负数）,会导致强制转换错误
            // 直接进行下面的位操作即可
            int accentColor = (int)dwm.GetValue("AccentColor");
            // 注意：读取到的颜色为 AABBGGRR
            return Color.FromArgb(
                (byte)((accentColor >> 24) & 0xFF),
                (byte)(accentColor & 0xFF),
                (byte)((accentColor >> 8) & 0xFF),
                (byte)((accentColor >> 16) & 0xFF));
        }
    }

    return SystemParameters.WindowGlassColor;  // 近似的系统主题色
}
```

以上的方式是通过读取注册表的方式，所以理论上任何语言都是通用的，再结合`WM_DWMCOLORIZATIONCOLORCHANGED`消息，就可以完美做到程序跟随系统主题色。

同样的，也可以通过读取注册表的方式获取到DWM颜色。

``` csharp
// \HKEY_CURRENT_USER\Software\Microsoft\Windows\DWM\ColorizationColor
public static Color GetColorizationColor()
{
    using (RegistryKey dwm = Registry.CurrentUser.OpenSubKey(@"Software\Microsoft\Windows\DWM", false))
    {
        if (dwm.GetValueNames().Contains("ColorizationColor"))
        {
            int accentColor = (int)dwm.GetValue("ColorizationColor");
            // 注意：读取到的颜色为 AARRGGBB
            return Color.FromArgb(
                (byte)((accentColor >> 24) & 0xFF),
                (byte)((accentColor >> 16) & 0xFF),
                (byte)((accentColor >> 8) & 0xFF),
                (byte)(accentColor & 0xFF));
        }
    }

    return SystemParameters.WindowGlassColor;
}
```

## 亮色主题还是暗色主题

``` csharp
// \HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize\AppsUseLightTheme
// 应用是亮色还是暗色
public static bool AppsUseLightTheme()
{
    using (RegistryKey personalize = Registry.CurrentUser.OpenSubKey(@"Software\Microsoft\Windows\CurrentVersion\Themes\Personalize", false))
    {
        if (personalize.GetValueNames().Contains("AppsUseLightTheme"))
        {
            return (int)personalize.GetValue("AppsUseLightTheme") == 1;
        }
    }

    return true;
}
```

对于较高版本的Win10，系统也可以设置亮色/暗色模式，我使用的版本暂且不支持。

需要读取注册表的路径为`\HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize\SystemUsesLightTheme`。
读取方式应该和上面差别不大。

## 主题色是否应用到窗口标题栏和边框

如果没有在系统个性化设置中设置将主题色应用到窗口标题栏和边框时，Win10窗口的标题栏和边框会始终为**白色**（对应亮色模式）或**黑色**（对应暗色模式）。
设置了将主题色应用到窗口标题栏和边框后，窗口的标题栏会跟随系统主题色的变化而变化。
那么我们怎样知道用户是怎样设置的呢？依旧是通过读取注册表的方式。

``` csharp
// \HKEY_CURRENT_USER\Software\Microsoft\Windows\DWM\ColorPrevalence
public static bool IsWindowPrevalenceAccentColor()
{
    using (RegistryKey dwm = Registry.CurrentUser.OpenSubKey(@"Software\Microsoft\Windows\DWM", false))
    {
        if (dwm.GetValueNames().Contains("ColorPrevalence"))
        {
            int colorPrevalence = (int)dwm.GetValue("ColorPrevalence");

            return colorPrevalence == 1;
        }
    }

    return false;
}
```

``` cpp
// c++ 读取注册表
bool IsWindowPrevalenceAccentColor()
{
	LPCTSTR dwm = _T("Software\\Microsoft\\Windows\\DWM");

	HKEY hDwmKey;

	if (ERROR_SUCCESS != RegOpenKeyEx(HKEY_CURRENT_USER, dwm, 0, KEY_READ, &hDwmKey))
	{
		return false;
	}

	DWORD type = REG_DWORD;
	DWORD value = 0;
	DWORD cbData = 4;

	if (ERROR_SUCCESS != RegQueryValueEx(hDwmKey, _T("ColorPrevalence"), nullptr, &type, (LPBYTE)&value, &cbData))
	{
		RegCloseKey(hDwmKey);
		return false;
	}

	RegCloseKey(hDwmKey);

	return value == 1;
}
```

## 根据背景色计算前景色

~~还有一个遗留的问题就是，窗口的标题栏颜色跟随系统主题色变化时，标题文字的颜色也会动态变换成白色或者黑色，至于什么情况下是白色，什么时候是黑色，暂时还没研究出来。~~

最后在网上找到了一些算法，通过背景的颜色计算出前景色，效果还是很理想的。

方法1：

``` csharp
/// <summary>
/// 根据背景色计算前景色(白/黑)
/// https://github.com/loilo/windows-titlebar-color/blob/master/WindowsAccentColors.js#L53
/// </summary>
/// <param name="background">背景颜色</param>
/// <returns>前景颜色(白/黑)</returns>
public static Color GetForegroundColor(Color background)
{
    return (background.R * 2 + background.G * 5 + background.B) <= 1024 /* 8*128 */
        ? Colors.White : Colors.Black;
}
```

方法2：

``` csharp
/// <summary>
/// 计算能在任何背景色上清晰显示的前景色
/// https://www.cnblogs.com/walterlv/p/10236517.html
/// </summary>
/// <param name="background">背景颜色</param>
/// <returns>前景颜色(黑/白)</returns>
public static Color GetReverseForegroundColor(Color background)
{
    double grayLevel = (0.299 * background.R + 0.587 * background.G + 0.114 * background.B) / 255;

    return grayLevel > 0.5 ? Colors.Black : Colors.White;
}
```

持续更新中...

**参考**

[C++ 获取并判断操作系统版本，解决Win10、 Windows Server 2012 R2 读取失败的方案](https://www.cnblogs.com/zhengshuangliang/p/5258504.html)

[wpf/winform获取windows10系统颜色和主题色](https://www.cnblogs.com/blue-fire/p/11874519.html)

[WM_DWMCOLORIZATIONCOLORCHANGED message](https://docs.microsoft.com/en-us/windows/win32/dwm/wm-dwmcolorizationcolorchanged)

[DwmGetColorizationColor function](https://docs.microsoft.com/en-us/windows/win32/api/dwmapi/nf-dwmapi-dwmgetcolorizationcolor)

[分享一个算法，计算能在任何背景色上清晰显示的前景色](https://www.cnblogs.com/walterlv/p/10236517.html)
