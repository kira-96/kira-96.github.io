---
title: gRPC C# 初探
date: 2019-08-18T17:52:36+08:00
draft: false
description: gRpc简单使用
tags: [ "C#" , "进程通信" ]
keywords: [ "进程通信" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 简介

[gRPC](https://www.grpc.io/)是Google开源的一个现代化、高性能的[RPC](https://baike.baidu.com/item/远程过程调用协议)框架，基于`HTTP/2`标准设计，同时提供多个语言版本，并支持跨语言调用，可以在任何环境中运行。

## 创建项目

新建解决方案，包含3个项目

* gRpcSample

  类库，gRPC生成的接口，Server接口、Client接口等

* server

  控制台程序，服务端

* client

  控制台程序，客户端

分别给3个项目安装Nuget程序包`Grpc`并安装所需依赖，然后为`gRpcSample`项目安装`Grpc.Tools`和`Google.ProtoBuf`程序包。

同时，使项目`server`和`client`引用项目`gRpcSample`。

## 定义服务接口

在**gRpcSample**项目的文件夹下新建**Sample.proto**文件，以文本方式打开，修改其中接口定义

``` protobuf
syntax = "proto3";

option csharp_namespace = "kira.Interface";

package sampleservice;

service SampleService {
	rpc ServerVersion(VersionRequest) returns (VersionResponse) {}
	rpc SayHello(HelloRequest) returns (stream HelloResponse) {}
}

message VersionRequest {}

message VersionResponse {
    string name = 1;
    string version = 2;
}

message HelloRequest {
    string name = 1;
}

message HelloResponse {
    string message = 1;
}
```

这里指定了生成的类的命名空间，同时定义了两个服务函数，似乎每个函数都必须有参数（请求）和返回（响应），这一点不太清楚。具体的语法有条件的可以参考[proto3 language guide](https://developers.google.com/protocol-buffers/docs/proto3)。

## 生成服务接口

在进行下面操作前，建议先将`Grpc.Tools`拷贝到解决方案目录下，不然的话下面的命令会很长很长...

具体操作是将解决方案下`packages\Grpc.Tools.2.23.0-pre1\tools\windows_x64\`里面的`protoc.exe`和`grpc_csharp_plugin.exe`拷贝到解决方案目录下，完成后就可以进行下一步。

在解决方案目录下打开命令窗口，并执行下面命令

> Tip: 直接进入到相应文件夹下，按住`Shift`键，在空白出单击鼠标右键，就可以看到菜单中多出了一项**在此处打开命令窗口(W)**

``` bash
$ protoc -IgRpcSample --csharp_out gRpcSample gRpcSample\Sample.proto --grpc_out gRpcSample --plugin=protoc-gen-grpc=grpc_csharp_plugin.exe
```

执行完没有错误的话，就可以看到`gRpcSample`项目下多出了两个文件`Sample.cs`和`SampleGrpc.cs`，将这两个文件添加到项目`gRpcSample`。

编译`gRpcSample`通过。

进行到这里，基本的工作就都完成了，剩下的就是编写服务端和客户端程序了。

## 服务端程序

`service`项目新建类`MySampleService`，并继承`SampleService.SampleServiceBase`，重写刚刚定义的两个服务接口函数

``` csharp
namespace server
{
    using System.Threading.Tasks;
    using Grpc.Core;
    using kira.Interface;

    public class MySampleService : SampleService.SampleServiceBase
    {
        public override Task<VersionResponse> ServerVersion(VersionRequest request, ServerCallContext context)
        {
            return Task.FromResult<VersionResponse>(
                new VersionResponse()
                {
                    Name = "My Sample Service",
                    Version = "0.0.1.19"
                });
        }

        public override async Task SayHello(HelloRequest request, IServerStreamWriter<HelloResponse> responseStream, ServerCallContext context)
        {
            string[] hellos = { "你好", "Hello", "Hola", "Bonjour", "こんにちは", "hallo" };

            foreach (string item in hellos)
            {
                await responseStream.WriteAsync(new HelloResponse() { Message = $"{item} {request.Name}" });
            }
        }
    }
}
```

编写服务启动程序

``` csharp
namespace server
{
    using Grpc.Core;
    using kira.Interface;

    class Program
    {
        static void Main(string[] args)
        {
            Server myServer = new Server()
            {
                Services = { SampleService.BindService(new MySampleService()) },
                Ports = { new ServerPort("localhost", 10240, ServerCredentials.Insecure) }
            };
            myServer.Start();

            System.Console.WriteLine("Sample Server listening on localhost:10240 \nPress any key exit...");
            System.Console.ReadKey();

            myServer.ShutdownAsync().Wait();
        }
    }
}
```

## 客户端程序

``` csharp
namespace client
{
    using System.Threading.Tasks;
    using Grpc.Core;
    using kira.Interface;

    class Program
    {
        static void Main(string[] args)
        {
            Program program = new Program();

            program.TestService();

            System.Console.ReadKey();
        }

        async void TestService()
        {
            Channel channel = new Channel("localhost:10240", ChannelCredentials.Insecure);

            SampleService.SampleServiceClient cli = new SampleService.SampleServiceClient(channel);

            VersionResponse ver = cli.ServerVersion(new VersionRequest());
            System.Console.WriteLine("Remote Service Version: {0} - v{1}", ver.Name, ver.Version);

            AsyncServerStreamingCall<HelloResponse> greetings = cli.SayHello(new HelloRequest() { Name = "gRPC" });
            IAsyncStreamReader<HelloResponse> stream = greetings.ResponseStream;

            while (await stream.MoveNext())
            {
                System.Console.WriteLine(stream.Current.Message);
            }
        }
    }
}
```

## 运行测试

首先启动服务端程序，然后运行客户端程序，可以看到客户端输出

```
Remote Service Version: My Sample Service - v0.0.1.19
你好 gRPC
Hello gRPC
Hola gRPC
Bonjour gRPC
こんにちは gRPC
hallo gRPC
```

OK，大功告成！
