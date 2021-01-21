---
title: Hprose C# 初探
date: 2019-08-19T19:51:52+08:00
draft: false
description: Hprose简单使用
tags: [ "C#" , "进程通信" ]
keywords: [ "进程通信" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 简介

[Hprose](https://hprose.com/)（High Performance Remote Object Service Engine）是一款先进的轻量级、跨语言、跨平台、无侵入式、高性能动态远程对象调用引擎库。它不仅简单易用，而且功能强大。

也是一个跨语言的RPC框架，但由于库的质量参差不齐，一些语言的库并不完善。这里以C#为例来实现一个简单的服务端和客户端程序。

## 创建项目

新建解决方案，包含两个项目

* server

  控制台程序，服务端

* client

  控制台程序，客户端

然后，通过Nuget分别为两个项目安装`Hprose.RPC`库。

这里并不需要再创建额外的服务接口项目，只需要手动定义一个接口即可。

## 创建服务接口

在`server`和`client`项目下都新建一个接口，作为服务接口

``` csharp
// IHello.cs
public class ServiceVersion
{
    public string Name { get; set; }
    public string Version { get; set; }

    public ServiceVersion() { }

    public ServiceVersion(string name, string ver)
    {
        Name = name;
        Version = ver;
    }
}

// 服务接口
public interface IHello
{
    ServiceVersion GetVersion();
    List<string> SayHello(string name);
}
```

`IHello`里面的两个接口函数就是服务接口了。

## 服务端程序

依旧是服务端实现接口，客户端来调用，在`server`项目下新建类`Hello.cs`

``` csharp
// Hello.cs
namespace server
{
    using System.Collections.Generic;

    public class Hello : IHello
    {
        public ServiceVersion GetVersion()
        {
            return new ServiceVersion("Hello Service", "0.0.1.21");
        }

        public List<string> SayHello(string name)
        {
            return new List<string>()
            {
                $"你好 {name}",
                $"Hello {name}",
                $"Hola {name}",
                $"Bonjour {name}",
                $"こんにちは {name}",
                $"hallo {name}"
            };
        }
    }
}
```

编写服务启动程序

``` csharp
namespace server
{
    using Hprose.RPC;
    using System.Net;

    class Program
    {
        static void Main()
        {
            HttpListener server = new HttpListener();
            server.Prefixes.Add("http://localhost:10240/");
            server.Start();

            Service service = new Service().Bind(server).AddInstanceMethods(new Hello());

            System.Console.WriteLine("Server listening at http://localhost:10240/ \n Press any key exit ...");
            System.Console.ReadKey();
            server.Stop();
        }
    }
}
```

## 客户端程序

由于`client`项目刚刚也定义了`IHello`接口，这里就可以直接调用接口函数了。

``` csharp
namespace client
{
    using Hprose.RPC;

    class Program
    {
        static void Main()
        {
            Client cli = new Client("http://localhost:10240/");
            IHello hello = cli.UseService<IHello>();

            ServiceVersion ver = hello.GetVersion();
            System.Console.WriteLine("Remote Service Version: {0} - v{1}", ver.Name, ver.Version);

            var hellos = hello.SayHello("Hprose");

            foreach (string item in hellos)
            {
                System.Console.WriteLine(item);
            }

            System.Console.ReadKey();
        }
    }
}
```

## 运行测试

先启动服务端程序，再启动客户端程序，可以看到客户端输出

```
Remote Service Version: Hello Service - v0.0.1.21
你好 Hprose
Hello Hprose
Hola Hprose
Bonjour Hprose
こんにちは Hprose
hallo Hprose
```

与`Thrift`和`gRPC`相比，`Hprose`实现起来要简单很多，暂时还没有尝试跨语言调用，不知道是不是同样简单。
