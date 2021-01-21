---
title: Thrift C# 初探
date: 2019-08-19T19:51:15+08:00
draft: false
description: Thrift简单使用
tags: [ "C#" , "进程通信" ]
keywords: [ "进程通信" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 简介

[Thrift](http://thrift.apache.org/)是由Facebook为“大规模跨语言服务开发”而开发的一种接口描述语言和二进制通讯协议，它被用来定义和创建跨语言的服务。目前被作为一个**RPC**框架使用。

## 下载

使用之前需要先[下载](http://thrift.apache.org/download)Thrift的源代码和Thrift编译器。

下载源代码后解压，进入到`thrift-0.12.0\lib\csharp\src`目录下，打开`Thrift.sln`，根据需要编译相应的库，这里选择`Thrift.45`，即.NET 4.5可以使用的库，编译生成`Thrift45.dll`。

## 创建项目

新建解决方案，包含3个项目

* ThriftSample

  类库，Thrift生成的服务接口

* server

  控制台程序，服务端

* client

  控制台程序，客户端

这3个项目都需要引用刚刚编译的`Thrift45.dll`，为什么不用Nuget来安装Thrift呢？因为我注意到Nuget上的Thrift已经好几年没更新了，还是手动编译最新的要好。

然后，`server`和`client`同时引用项目`ThriftSample`。

## 定义服务接口

在解决方案目录下新建一个文件`Sample.thrift`来定义服务接口，Thrift语法可以在网上找到

``` protobuf
namespace csharp kira.Interface

service SampleService {
    ServiceVersion GetVersion()
    list<string> SayHello(1: string name)
}

struct ServiceVersion {
    1: required string name;
    2: required string version;
}
```

这里指定了生成类的命名空间，以及定义了一个结构体作为返回值

## 生成服务接口

这里就需要用到之前下载的Thrift编译器，可以直接在Thrift官网找到。将下载的`thrift-0.12.0.exe`也拷贝到解决方案目录下（和Sample.thrift相同目录），打开命令窗口，执行以下命令

``` bash
$ thrift-0.12.0.exe -gen csharp Sample.thrift
```

不得不说，Thrift的命令行语法真的比gRPC简洁多了。

执行完没有错误的话，就可以看到目录下又多出了一个`gen-csharp`的文件夹，里面有对应命名空间的文件夹，最后找到生成的`.cs`文件。将文件全部拷贝到`ThriftSample`项目目录下，并将它们添加到项目，编译生成库。

## 服务端程序

在`server`项目下新建类`MySampleService`并实现接口`SampleService.Iface`，重写其中的两个服务接口函数

``` csharp
namespace server
{
    using System.Collections.Generic;
    using kira.Interface;

    public class MySampleService : SampleService.Iface
    {
        public ServiceVersion GetVersion()
        {
            return new ServiceVersion()
            {
                Name = "My Sample Service",
                Version = "0.0.1.20"
            };
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
    using Thrift.Server;
    using Thrift.Transport;
    using kira.Interface;

    class Program
    {
        static void Main(string[] args)
        {
            MySampleService service = new MySampleService();

            SampleService.Processor processor = new SampleService.Processor(service);

            TServerTransport serverTransport = new TServerSocket(10240);
            TServer server = new TThreadPoolServer(processor, serverTransport);
            System.Console.WriteLine("server listening at tcp://localhost:10240/");
            server.Serve();
        }
    }
}
```

## 客户端程序

``` csharp
namespace client
{
    using Thrift.Protocol;
    using Thrift.Transport;
    using kira.Interface;

    class Program
    {
        static void Main(string[] args)
        {
            TTransport transport = new TSocket("localhost", 10240);
            transport.Open();

            TProtocol protocol = new TBinaryProtocol(transport);
            SampleService.Client cli = new SampleService.Client(protocol);

            ServiceVersion ver = cli.GetVersion();
            System.Console.WriteLine("Remote Service Version: {0} - v{1}", ver.Name, ver.Version);

            var hellos = cli.SayHello("Thrift");

            foreach (string item in hellos)
            {
                System.Console.WriteLine(item);
            }

            System.Console.ReadKey();
            transport.Close();
        }
    }
}
```

## 运行测试

老样子，先启动服务端程序，然后运行客户端程序，可以看到客户端输出

```
Remote Service Version: My Sample Service - v0.0.1.20
你好 Thrift
Hello Thrift
Hola Thrift
Bonjour Thrift
こんにちは Thrift
hallo Thrift
```

## 总结

对比Thrift和gRPC两个主流的RPC框架，个人感觉Thrift使用起来要更加灵活一些，当然这只是初步接触，较深层次的内容还没有去研究，实际项目的应用感觉两个都可以，有对比说Thrift框架的性能要优于gRPC，但对于小的项目来说已经完全够用了。
