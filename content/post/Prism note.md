---
title: Prism note
date: 2020-09-04T09:10:00+08:00
draft: false
description: Prism学习笔记
tags: [ "WPF", "Prism" ]
keywords: [ "WPF", "Prism" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 简介

[Prism](https://github.com/PrismLibrary/Prism)是一个用于WPF、Xamarin Forms、WinUI等的MVVM框架，刚刚学习，这里只是个人总结的一些知识点笔记。

## IoC

**`IContainerProvider`**

``` csharp
protected override Window CreateShell()
{
    return Container.Resolve<MainWindow>();
}
```

``` csharp
public void OnInitialized(IContainerProvider containerProvider)
{
    var regionManager = containerProvider.Resolve<IRegionManager>();
    var viewA = containerProvider.Resolve<ViewA>();
    ...
}
```

**`IContainerRegistry`**

``` csharp
// App.xaml.cs
protected override void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.Register<IApplicationCommands, ApplicationCommands>();
    containerRegistry.RegisterDialog<NotificationDialog, NotificationDialogViewModel>();
    containerRegistry.RegisterForNavigation<Page1>();
    containerRegistry.RegisterForNavigation<Page2>();
    ...
}
```

## Module

**`IModule`**

``` csharp
public class SimpleModule : IModule
{
    public void OnInitialized(IContainerProvider containerProvider)
    {
    }

    public void RegisterTypes(IContainerRegistry containerRegistry)
    {
    }
}
```

**使用App.config加载模块**

``` xml
<!-- App.config -->
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="modules" type="Prism.Modularity.ModulesConfigurationSection, Prism.Wpf" />
  </configSections>
  <startup>
  </startup>
  <modules>
    <module assemblyFile="Simple.dll" moduleType="Simple.SimpleModule, Simple, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" moduleName="SimpleModule" startupLoaded="True" />
  </modules>
</configuration>
```

``` csharp
// App.xaml.cs
public partial class App : PrismApplication
{
    ...

    protected override IModuleCatalog CreateModuleCatalog()
    {
        return new ConfigurationModuleCatalog();
    }

    ...
}
```

**直接引用加载模块**

``` csharp
// App.xaml.cs
public partial class App : PrismApplication
{
    ...

    protected override void ConfigureModuleCatalog(IModuleCatalog moduleCatalog)
    {
        moduleCatalog.AddModule<SimpleModule>();
    }

    ...
}
```

**指定模块文件夹**

``` csharp
// App.xaml.cs
public partial class App : PrismApplication
{
    ...

    protected override IModuleCatalog CreateModuleCatalog()
    {
        return new DirectoryModuleCatalog() { ModulePath = @".\Modules" };
    }

    ...
}
```

**使用`ModuleCatalog`加载模块**

``` xml
<m:ModuleCatalog xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                 xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                 xmlns:m="clr-namespace:Prism.Modularity;assembly=Prism.Wpf">

    <m:ModuleInfo ModuleName="Simple"
                  ModuleType="Simple.SimpleModule, Simple, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />

</m:ModuleCatalog>
```

``` csharp
// App.xaml.cs
public partial class App : PrismApplication
{
    ...

    protected override IModuleCatalog CreateModuleCatalog()
    {
        return ModuleCatalog.CreateFromXaml(new Uri("/Modules;component/ModuleCatalog.xaml", UriKind.Relative));
    }

    ...
}
```

## Command

**`DelegateCommand`**

``` csharp
public DelegateCommand ExecuteDelegateCommand { get; }
public DelegateCommand<string> ExecuteGenericDelegateCommand { get; }
public DelegateCommand DelegateCommandObservesProperty { get; }
public DelegateCommand DelegateCommandObservesCanExecute { get; }

ExecuteDelegateCommand = new DelegateCommand(Execute, CanExecute);

DelegateCommandObservesProperty = new DelegateCommand(Execute, CanExecute).ObservesProperty(() => IsEnabled);

DelegateCommandObservesCanExecute = new DelegateCommand(Execute).ObservesCanExecute(() => IsEnabled);

ExecuteGenericDelegateCommand = new DelegateCommand<string>(ExecuteGeneric).ObservesCanExecute(() => IsEnabled);
```

**`CompositeCommand`**

``` csharp
public CompositeCommand SampleCommand { get; } = new CompositeCommand(true);

...

DelegateCommand UpdateCommand = new DelegateCommand(Update).ObservesCanExecute(() => CanUpdate);

SampleCommand.RegisterCommand(UpdateCommand);

...

private void OnIsActiveChanged()
{
    UpdateCommand.IsActive = IsActive;

    IsActiveChanged?.Invoke(this, new EventArgs());
}
```

**Event To Command**

``` xml
<i:Interaction.Triggers>
    <i:EventTrigger EventName="SelectionChanged">
        <prism:InvokeCommandAction Command="{Binding PersonSelectedCommand}"
            CommandParameter="{Binding ElementName=ListOfPerson, Path=SelectedItem}" />
    </i:EventTrigger>
</i:Interaction.Triggers>
```

## BindableBase

``` csharp
public class ViewAViewModel : BindableBase, IActiveAware
{
    ...
}
```

## ViewModelLocator

**AutoWireViewModel**

``` xml
<Window x:Class="Demo.Views.MainWindow"
    ...
    xmlns:prism="http://prismlibrary.com/"
    prism:ViewModelLocator.AutoWireViewModel="True">
```

**更改命名约定**

``` csharp
// App.xaml.cs
public partial class App : PrismApplication
{
    ...

    protected override void ConfigureViewModelLocator()
    {
        base.ConfigureViewModelLocator();
        ViewModelLocationProvider.SetDefaultViewTypeToViewModelTypeResolver((viewType) =>
        {
            var viewName = viewType.FullName.Replace(".ViewModels.", ".CustomNamespace.");
            var viewAssemblyName = viewType.GetTypeInfo().Assembly.FullName;
            var viewModelName = $"{viewName}ViewModel, {viewAssemblyName}";
            return Type.GetType(viewModelName);
        });
    }

    ...
}
```

**自定义ViewModel注册**

``` csharp
// App.xaml.cs
public partial class App : PrismApplication
{
    ...

    protected override void ConfigureViewModelLocator()
    {
        base.ConfigureViewModelLocator();

        // type / type
        ViewModelLocationProvider.Register(typeof(MainWindow).ToString(), typeof(CustomViewModel));

        // type / factory
        ViewModelLocationProvider.Register(typeof(MainWindow).ToString(), () => Container.Resolve<CustomViewModel>());

        // generic factory
        ViewModelLocationProvider.Register<MainWindow>(() => Container.Resolve<CustomViewModel>());

        // generic type
        ViewModelLocationProvider.Register<MainWindow, CustomViewModel>();
    }

    ...
}
```

## EventAggregator

**`IEventAggragator`**

``` csharp
public interface IEventAggregator
{
    TEventType GetEvent<TEventType>() where TEventType : EventBase;
}
```

**创建消息事件类**

``` csharp
public class SimpleMessageEvent : PubSubEvent<string>
{
}
```

**订阅事件**

``` csharp
private readonly IEventAggregator eventAggregator;

public MainPageViewModel(IEventAggregator ea)
{
    eventAggregator = ea;
    ea.GetEvent<SimpleMessageEvent>().Subscribe(ShowMessage);
    // Subscribing on the UI Thread
    // ea.GetEvent<SimpleMessageEvent>().Subscribe(ShowMessage, ThreadOption.UIThread);
}

public void ShowMessage(string payload)
{
    // TODO
}
```

**发布消息**

``` csharp
eventAggregator.GetEvent<SimpleMessageEvent>().Publish("Hello!");
```

**筛选订阅**

``` csharp
ea.GetEvent<SimpleMessageEvent>().Subscribe(ShowMessage, ThreadOption.UIThread, keepSubscriberReferenceAlive, x => x.Contains(" "));
```

**取消订阅**

``` csharp
eventAggregator.GetEvent<SimpleMessageEvent>().Unsubscribe(ShowMessage);
```

## RegionManager

``` xml
<Window x:Class="Regions.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:prism="http://prismlibrary.com/"
        Title="Shell">
    <Grid>
        <ContentControl prism:RegionManager.RegionName="ContentRegion" />
    </Grid>
</Window>
```

``` csharp
// IContainerProvider containerProvider
var regionManager = containerProvider.Resolve<IRegionManager>();
regionManager.RegisterViewWithRegion("ContentRegion", typeof(ViewA));
```

``` csharp
// IContainerProvider containerProvider
var regionManager = containerProvider.Resolve<IRegionManager>();
var region = regionManager.Regions["ContentRegion"];

region.Add(containerProvider.Resolve<ViewA>());
region.Add(containerProvider.Resolve<ViewB>());
region.Add(containerProvider.Resolve<ViewC>());
```

## RegionNavigation

``` csharp
// IRegionManager regionManager
regionManager.RequestNavigate(regionName: "NavigateRegion", source: "Page1");
```

**Navigation Callback**

``` csharp
// IRegionManager regionManager
regionManager.RequestNavigate(regionName: "NavigateRegion", source: "Page1", navigationCallback: NavigationComplete);

private void NavigationComplete(NavigationResult result)
{
    dialogService.ShowDialog("NotificationDialog", new DialogParameters($"message=Navigate to {result.Context.Uri} complete."), null);
}
```

**Navigation Parameters**

``` csharp
var parameters = new NavigationParameters
{
    { "content", "Hello!" }
};
// IRegionManager regionManager
regionManager.RequestNavigate(regionName: "NavigateRegion", source: "Page1", navigationParameters: parameters);
```

**INavigationAware**

``` csharp
public class Page1ViewModel : BindableBase, INavigationAware
{
    ...

    public bool IsNavigationTarget(NavigationContext navigationContext)
    {
        return true;
    }

    public void OnNavigatedFrom(NavigationContext navigationContext)
    {
    }

    public void OnNavigatedTo(NavigationContext navigationContext)
    {
        if (navigationContext.Parameters["content"] is string content)
        {
            // TODO
        }
    }

    ...
}
```

**IConfirmNavigationRequest**

``` csharp
public class Page1ViewModel : BindableBase, IConfirmNavigationRequest
{
    ...

    public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
    {
        bool result = true;
        ButtonResult buttonResult = ButtonResult.None;

        dialogService.ShowDialog("NotificationDialog",
            new DialogParameters($"message=Do you to navigate?"),
            res => { buttonResult = res.Result; });

        if (buttonResult != ButtonResult.OK)
            result = false;

        continuationCallback(result);
    }

    ...
}
```

**IRegionMemberLifetime**

``` csharp
public class Page1ViewModel : BindableBase, INavigationAware, IRegionMemberLifetime
{
    public bool KeepAlive
    {
        get
        {
            return false;
        }
    }

    public bool IsNavigationTarget(NavigationContext navigationContext)
    {
        return false;
    }

    public void OnNavigatedFrom(NavigationContext navigationContext)
    {
    }

    public void OnNavigatedTo(NavigationContext navigationContext)
    {
    }
}
```

**Navigation Journal**

``` csharp
public class Page1ViewModel : BindableBase, INavigationAware
{
    private IRegionNavigationJournal journal;

    public DelegateCommand GoForwardCommand { get; }
    public DelegateCommand GoBackCommand { get; }

    public Page1ViewModel()
    {
        GoForwardCommand = new DelegateCommand(GoForward, CanGoForward);
        GoBackCommand = new DelegateCommand(GoBack);
    }

    ...

    public bool IsNavigationTarget(NavigationContext navigationContext)
    {
        return true;
    }

    public void OnNavigatedFrom(NavigationContext navigationContext)
    {
    }

    public void OnNavigatedTo(NavigationContext navigationContext)
    {
        journal = navigationContext.NavigationService.Journal;
        GoForwardCommand.RaiseCanExecuteChanged();
    }

    ...

    private bool CanGoForward()
    {
        return journal != null && journal.CanGoForward;
    }

    private void GoForward()
    {
        journal?.GoForward();
    }

    private void GoBack()
    {
        journal?.GoBack();
    }
}
```

## DialogService

See DOC. [Dialog Service](https://prismlibrary.com/docs/wpf/dialog-service.html)

``` csharp
// DialogServiceModule.cs
public void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.RegisterDialog<NotificationDialog, NotificationDialogViewModel>();
    // containerRegistry.RegisterDialogWindow<MyRibbonWindow>();
}
```

``` csharp
// viewmodel
private readonly IDialogService dialogService;

public MainViewModel(IDialogService dialogService)
{
    this.dialogService = dialogService;
}

private void NavigationComplete(NavigationResult result)
{
    // Show Dialog with parameters.
    dialogService.ShowDialog("NotificationDialog", new DialogParameters($"message=Navigate to {result.Context.Uri} complete."), null);
}
```

## 参考

- [Documentation](https://prismlibrary.com/docs/index.html)
- [Prism-Samples-Wpf](https://github.com/PrismLibrary/Prism-Samples-Wpf)
