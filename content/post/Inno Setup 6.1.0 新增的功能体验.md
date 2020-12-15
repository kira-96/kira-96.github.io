---
title: Inno Setup 6.1.0 新增的功能体验
date: 2020-08-20T10:00:10+08:00
draft: false
description: Inno Setup 6.1.0 功能体验
tags: [ "inno-setup" ]
keywords: [ "inno-setup" ]
categories: [ "分享" ]
isCJKLanguage: true
---

## 简介

[Inno Setup](https://jrsoftware.org/isinfo.php)是一个免费的安装包生成软件，完全开源免费，使用起来也非常方便，文档也十分全面。与其它同类软件相比十分的小巧便携，功能也十分全面。

近期Inno Setup的6.1.0版本也即将发布，也带来了更多的新功能。由于正式版本还没有发布，这里就使用的先行版本。

## 下载页面

Inno Setup 6.1版本新增了安装过程中的**下载页面**，在所有选项准备完毕，正式开始安装之前可以下载需要的文件。官方也给出了下载示例的代码 [CodeDownloadFiles.iss](https://jrsoftware.github.io/issrc/Examples/CodeDownloadFiles.iss)。

``` pascal
[Code]
var
  DownloadPage: TDownloadWizardPage;

function OnDownloadProgress(const Url, FileName: String; const Progress, ProgressMax: Int64): Boolean;
begin
  if Progress = ProgressMax then
    Log(Format('Successfully downloaded file to {tmp}: %s', [FileName]));
  Result := True;
end;

procedure InitializeWizard;
begin
  DownloadPage := CreateDownloadPage(SetupMessage(msgWizardPreparing), SetupMessage(msgPreparingDesc), @OnDownloadProgress);
end;

function NextButtonClick(CurPageID: Integer): Boolean;
begin
  if CurPageID = wpReady then begin
    DownloadPage.Clear;
    DownloadPage.Add('https://files.jrsoftware.org/is/6/innosetup-6.1.0-dev.exe', 'innosetup-6.1.0-dev.exe', '');
    DownloadPage.Add('https://jrsoftware.org/download.php/iscrypt.dll', 'ISCrypt.dll', '2f6294f9aa09f59a574b5dcd33be54e16b39377984f3d5658cda44950fa0f8fc');
    DownloadPage.Show;
    try
      try
        DownloadPage.Download;
        Result := True;
      except
        SuppressibleMsgBox(AddPeriod(GetExceptionMessage), mbCriticalError, MB_OK, IDOK);
        Result := False;
      end;
    finally
      DownloadPage.Hide;
    end;
  end else
    Result := True;
end;
```

上面代码在**初始化**时创建了一个下载页面，并在`wpReady`之后显示。

`CreateDownloadPage`的原型：

```pascal
function CreateDownloadPage(const ACaption, ADescription: String; const OnDownloadProgress: TOnDownloadProgress): TDownloadWizardPage;
```

创建一个下载页面用于下载文件并显示进度。

前两个参数指定页面的标题和页面描述，第3个参数是在下载进度更新后的回调函数`TOnDownloadProgress`，可以指定为`nil`（空）。

``` pascal
TOnDownloadProgress = function(const Url, FileName: string; const Progress, ProgressMax: Int64): Boolean;
```

`CreateDownloadPage`返回`TDownloadWizardPage`类型：

``` pascal
TDownloadWizardPage = class(TOutputProgressWizardPage)
  property AbortButton: TNewButton; read;
  procedure Add(const Url, BaseName, RequiredSHA256OfFile: String);
  procedure Clear;
  function Download: Int64;
end;
```

可以看到，`TDownloadWizardPage`有一个`Add`的方法，用于新增一个下载任务，它有3个参数：

`Url`：下载链接，`BaseName`：下载后的文件名称

`RequiredSHA256OfFile`：文件的哈希值，用于校验下载文件，值为空时，则忽略校验

`Clear`方法清空下载任务列表，`Download`方法开始下载任务。

具体的使用可以看上面的`NextButtonClick`函数里的写法。

另一个方法是使用`DownloadTemporaryFile`函数：

``` pascal
function DownloadTemporaryFile(const Url, FileName, RequiredSHA256OfFile: String; const OnDownloadProgress: TOnDownloadProgress): Int64;
```

``` pascal
[Code]
function OnDownloadProgress(const Url, Filename: string; const Progress, ProgressMax: Int64): Boolean;
begin
  if ProgressMax <> 0 then
    Log(Format('  %d of %d bytes done.', [Progress, ProgressMax]))
  else
    Log(Format('  %d bytes done.', [Progress]));
  Result := True;
end;

function InitializeSetup: Boolean;
begin
  try
    DownloadTemporaryFile('https://jrsoftware.org/download.php/is.exe', 'innosetup-latest.exe', '', @OnDownloadProgress);
    DownloadTemporaryFile('https://jrsoftware.org/download.php/iscrypt.dll', 'ISCrypt.dll', '2f6294f9aa09f59a574b5dcd33be54e16b39377984f3d5658cda44950fa0f8fc', @OnDownloadProgress);
    Result := True;
  except
    Log(GetExceptionMessage);
    Result := False;
  end;
end;
```

使用起来和前一种方法有所不同，但大致都是类似的，这里不再赘述。

## 消息框设计器

软件的工具菜单（Tools）中新增了消息框设计器（MessageBox Designer）工具。

![MessageBox Designer](https://cdn.jsdelivr.net/gh/kira-96/kira-96.github.io@gh-pages/images/Snipaste_2020-08-20_10-46-20.png)

工具提供了两种消息框，*Message Box*和*Task Dialog Message Box*，工具可以设置对话框的图标，按钮和默认选项等。

将鼠标指针放在需要插入对话框的代码位置，打开*MessageBox Designer*，完成选项后点击*OK*即可，然后就可以看到先前鼠标所在的位置插入了一段*MessageBox*代码。

``` ini
[CustomMessages]
DownloadComplete=下载完成
DownloadCompleteMessage=下载已完成。
```

``` pascal
// Display a message box
SuppressibleTaskDialogMsgBox(CustomMessage('DownloadComplete'), CustomMessage('DownloadCompleteMessage'), mbInformation, MB_OK, ['OK'], 0, IDOK);
```

## 链接

- [Inno Setup 6 Revision History](http://jrsoftware.github.io/issrc/whatsnew.htm)
- [Inno Setup 简体中文翻译](https://github.com/kira-96/Inno-Setup-Chinese-Simplified-Translation)
