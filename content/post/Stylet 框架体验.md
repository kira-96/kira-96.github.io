---
title: Stylet 框架体验
date: 2019-12-12T19:20:36+08:00
draft: false
description: Stylet简单使用
tags: [ "WPF" , "MVVM" ]
keywords: [ "WPF" , "MVVM" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 简介

[Stylet](https://github.com/canton7/Stylet)是一个轻量且功能强大的**MVVM**框架。支持 .NET 4.5+ 和 .NET Core 3.0+。

Stylet的作者也是受到[Caliburn.Micro](https://github.com/Caliburn-Micro/Caliburn.Micro)的启发，并且在CM的基础上做了许多改进。所以Stylet使用起来感觉和Caliburn.Micro差别不是很大，但又有着一些不同。

## 项目结构

这里选择创建一个 .NET Core 的 WPF 项目。

这里项目结构风格和Caliburn.Micro类似，示例[源代码](https://github.com/kira-96/Stylet-Sample)

<pre><code>.
├── Views
│      └── ...
├── ViewModels
│      └── ...
├── App.xaml
├── Bootstrapper.cs
└── ...
</code></pre>

虽然Stylet官方给出的例子里面View和ViewModel是放在一起的，但经过实际使用后发现采用CM的风格也是可以的。依照习惯，将Views和ViewModels分别放在两个文件夹中。

## 使用

**`Bootstrapper.cs`**

``` csharp
public class Bootstrapper : Bootstrapper<ShellViewModel>
{}
```

这样就相当于执行了`DisplayRootViewFor<ShellViewModel>()`。

然后再修改`App.xaml`如下就可以让程序启动了。

**`App.xaml`**

``` xml
<Application
    x:Class="WpfSample.App"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:s="https://github.com/canton7/Stylet"
    xmlns:app="clr-namespace:WpfSample">
    <Application.Resources>
        <s:ApplicationLoader>
            <s:ApplicationLoader.Bootstrapper>
                <app:Bootstrapper />
            </s:ApplicationLoader.Bootstrapper>
        </s:ApplicationLoader>
    </Application.Resources>
</Application>
```

## 绑定

Stylet似乎不再支持Caliburn.Micro的通过`x:Name`来绑定的机制，必须通过`Binding`显式指定绑定属性的方式。

``` xml
<TextBox Text="{Binding YourName, UpdateSourceTrigger=PropertyChanged}" />
```

**Action**的写法则有了更明显的差异。

``` xml
<Button Content="Say Hello"
        xmlns:stylet="https://github.com/canton7/Stylet"
        Command="{stylet:Action SayHello}" />
```

``` csharp
// ShellViewModel.cs

private string _name;

public string YourName
{
    get => _name;
    set
    {
        SetAndNotify(ref _name, value);
        NotifyOfPropertyChange(() => CanSayHello);
    }
}

public bool CanSayHello => !string.IsNullOrEmpty(YourName);

public void SayHello()
{
    _logger.Info("Say Hello {0}", _name);
    _windowManager.ShowMessageBox($"Hello {_name}");
}
```

不同于Caliburn.Micro的写法`cal:Message.Attach="[Click]=[OnClick()]"`

Stylet可以直接指定相应的事件，可读性自然是提升了不少，并且依旧不需要在ViewModel中写`ICommand`，直接写函数即可。当然也可以通过`CommandParameter`传递参数。

上面是通过`Command`的方式绑定，你也可以绑定到`Click`或者其它的事件。

以下是官方的示例:

``` xml
<Button Click="{s:Action DoSomething}">Click me</Button>
```

``` csharp
public void HasNoArguments() { }

// This can accept EventArgs, or a subclass of EventArgs
public void HasOneSingleArgument(EventArgs e) { }

// Again, a subclass of EventArgs is OK
public void HasTwoArguments(object sender, EventArgs e) { }
```

可以根据需求在ViewModel对应的函数中定义参数或者不定义。

**ActionTarget**

``` csharp
class InnerViewModel
{
   public void DoSomething() { }
}
class ViewModel
{
   public InnerViewModel InnerViewModel { get; private set; }
   public ViewModel()
   {
      this.InnerViewModel = new InnerViewModel();
   }
}
```

``` xml
<Button s:View.ActionTarget="{Binding InnerViewModel}" Command="{s:Action DoSomething}">Click me</Button>
```

通过指定`Action.Target`来绑定到其它的ViewModel。

## 依赖注入

Stylet提供了两个类供ViewModel继承，`Screen`和`Conductor`，这一点和Caliburn.Micro类似。所有的ViewModel都会自动绑定到`IoC`，不需要在`Bootstrapper`中进行设置。

必要时也可以重写Bootstrapper中的`ConfigureIoC`来注册一些其它的服务。

这里注册了一个`Logging` service，Stylet中提供有`ILogger`的接口，但不建议使用，可以自己实现。

``` csharp
protected override void ConfigureIoC(IStyletIoCBuilder builder)
{
    base.ConfigureIoC(builder);

    builder.Bind<ILogger>().To<Logger>().InSingletonScope().AsWeakBinding();
}
```

Stylet提供了多种注入的方式。

**通过构造函数注入**

``` csharp
public NavViewModel(
    IEventAggregator eventAggregator,
    FirstTabViewModel tab1,
    SecondTabViewModel tab2)
{
    this._eventAggregator = eventAggregator;
    this.Items.Add(tab1);
    this.Items.Add(tab2);
}
```

**通过`[Inject]`自动注入**

使用`[Inject]`方式注入时也可以指定相应的Key

``` csharp
// Logger.cs
[Inject(Key = "filelogger")]
public class Logger : ILogger
{}
```

``` csharp
[Inject(Key = "filelogger")]
private ILogger _logger;
```

**抽象工厂**

Stylet提供了一种抽象工厂的模式来获取相应的服务。

这里我定义了一个`IViewModelFactory`的接口

``` csharp
public interface IViewModelFactory
{
    ShellViewModel GetShellViewModel();
    NavViewModel GetNavViewModel();
    FirstTabViewModel GetFirstTabViewModel();
    SecondTabViewModel GetSecondTabViewModel();
}
```

然后在Bootstrapper的`ConfigureIoC`中添加如下代码

``` csharp
// Bootstrapper.cs
protected override void ConfigureIoC(IStyletIoCBuilder builder)
{
    ...
    builder.Bind<IViewModelFactory>().ToAbstractFactory();
    ...
}
```

这样就可以通过注入的方式来获取到`IViewModelFactory`的实例了。

注意这个过程中我并没有手动去实现`IViewModelFactory`的接口。

``` csharp
// ShellViewModel.cs
[Inject]
private IViewModelFactory _viewModelFactory;

// 这时就可以通过Factory来获取相应的ViewModel
var vm = _viewModelFactory.GetNavViewModel();
```

## IoC

虽然可以通过注入的方式来获取服务，但有时也需要通过IoC Container来获取相应的服务。Stylet依然有多种方式来获取。

**注入IoC Container**

``` csharp
// 注入IoC Conatiner
[Inject]
private IContainer _container;

// 通过Container获取ViewModel
var vm = _container.Get<NavViewModel>();
```

**Static Service Locator**

用过Caliburn.Micro的可能都知道，CM提供了一种非常好用的获取服务的方式。

``` csharp
var vm = IoC.Get<MyDialogViewModel>();
```

而Stylet并没有提供这种方式。Stylet作者给出的原因是：

> I don't want to encourage people to write such horrible code.

但我就是喜欢简单粗暴的，通过`IoC.Get`的方式比较合我的胃口。Stylet的作者同样也给出了相应的方式[链接](https://github.com/canton7/Stylet/wiki/IoC%3A-Static-Service-Locator)

但作者给出的代码中`GetAllInstance`是不能正确使用的。可以参考我的修改版[SimpleIoC](https://github.com/kira-96/Stylet-Sample/blob/master/SimpleIoC.cs)

最后再Bootstrapper中添加下面代码即可。

``` cs
protected override void Configure()
{
    base.Configure();

    SimpleIoC.GetInstance = this.Container.Get;
    SimpleIoC.GetAllInstances = this.Container.GetAll;
    SimpleIoC.BuildUp = this.Container.BuildUp;
}
```

使用:

``` csharp
var vm = SimpleIoC.Get<NavViewModel>();
```

## WindowManager

``` csharp
public interface IWindowManager
{
    bool? ShowDialog(object viewModel);
    MessageBoxResult ShowMessageBox(string messageBoxText, string caption = "", MessageBoxButton buttons = MessageBoxButton.OK, MessageBoxImage icon = MessageBoxImage.None, MessageBoxResult defaultResult = MessageBoxResult.None, MessageBoxResult cancelResult = MessageBoxResult.None, IDictionary<MessageBoxResult, string> buttonLabels = null, FlowDirection? flowDirection = null, TextAlignment? textAlignment = null);
    void ShowWindow(object viewModel);
}
```

Stylet的`IWindowManager`提供了3个接口函数，一个`MessageBox`，其它两个用于显示窗口，和Caliburn.Micro用法相同，在使用时把ViewModel传如即可。`IWindowManager`可以直接通过注入的方式获得。

``` csharp
[Inject]
private IWindowManager _windowManager;

_windowManager.ShowWindow(_viewModelFactory.GetNavViewModel());
```

## EventAggregator

`EventAggregator`和Caliburn.Micro中的用法相同。结合`IHandle<T>`用于在ViewModel中传递消息。

**订阅消息：**

``` csharp
// ShellViewModel.cs
public class ShellViewModel : Screen, IHandle<TabChangedEvent>
{
    private readonly IEventAggregator _eventAggregator;
    ...
    public ShellViewModel(IEventAggregator eventAggregator)
    {
        DisplayName = "Hello Stylet!";
        _eventAggregator = eventAggregator;
        _eventAggregator.Subscribe(this);
    }
    protected override void OnClose()
    {
        _eventAggregator.Unsubscribe(this);
        base.OnClose();
    }
    public void Handle(TabChangedEvent message)
    {
        // TODO
    }
    ...
}
```

**发布消息：**

``` csharp
// AnotherViewModel.cs
private readonly IEventAggregator _eventAggregator;

public NavViewModel(IEventAggregator eventAggregator)
{
    this._eventAggregator = eventAggregator;
}

// 发布消息
private Publish()
{
    _eventAggregator.Publish(new TabChangedEvent());
}
```

Stylet中的`EventAggregator`同时提供了`channels`，可以在不同的管道之中**订阅/发布**消息。

## 总结

对于使用过Caliburn.Micro的朋友来说，Stylet非常容易上手，大部分的用法基本上都一样，同时Stylet又提供了一些新的内容。这里有些东西并没有讲到，如**ViewManager**等，但Stylet已经足够让我兴奋了，还有就是它的体积真的很小，只有100多KB，并且功能也足够强大，用起来也很方便，后面有机会可以在一些小项目中使用。
