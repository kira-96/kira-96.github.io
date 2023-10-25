---
title: 使用 Named Pipe 进行进程间通讯
date: 2019-11-15T19:12:30+08:00
draft: false
description: Named Pipe简单使用
tags: [ "C#" , "进程通信" ]
keywords: [ "进程通信" ]
categories: [ "编程" ]
isCJKLanguage: true
---

这段时间公司的一个项目打算使用`Named Pipe`进行进程间的通讯，刚好花了点时间了解了一下，这里做一下笔记。

Named Pipe（命名管道），顾名思义，是通过在两个进程间搭建一个管道来进行通讯，这种方式的好处在于两者可以进行**全双工**的通讯，服务端也可以通过管道向客户端发送消息，对于两个进程之间的通讯来说再合适不过了，使用起来也相对比较灵活。

## 服务端（Server）

**Named Pipe 命名空间**

``` csharp
using System.IO.Pipes;
```

**创建管道**

``` csharp
PipeSecurity security = new PipeSecurity();  // 管道权限
// 设置规则，只有用户admin可以对管道进行读写，其它用户无权访问
security.AddAccessRule(new PipeAccessRule("admin", PipeAccessRights.ReadWrite, AccessControlType.Allow));

NamedPipeServerStream server = new NamedPipeServerStream(
    "SimpleServer",  // pipe name
    PipeDirection.InOut,  // 数据传输方向，这里使用双工通讯
    1,               // MaxNumberOfServerInstance
    PipeTransmissionMode.Byte,  // 字节流传输
    PipeOptions.Asynchronous | PipeOptions.WriteThrough,
    4096,  // 输入缓冲大小
    4096,  // 输出缓冲大小
    security);  // 管道访问权限，这里只做笔记，通常不需要设置
```

管道创建好之后还不能立刻发送数据，因为管道的另一端（客户端）还没有连接，所以服务端需要等待连接。

``` csharp
server.WaitForConnection();  // 阻塞方式
```

这里使用非阻塞的方式等待连接，当然也可以用阻塞的方式等待连接，不过需要放到一个新的线程中，避免将主线程阻塞。

``` csharp
server.BeginWaitForConnection(new AsyncCallback(WaitConnectionCallback), server);  // 非阻塞方式
```

``` csharp
private void WaitConnectionCallback(IAsyncResult asyncResult)
{
    NamedPipeServerStream server = asyncResult.AsyncState as NamedPipeServerStream;
    server.EndWaitForConnection(asyncResult);

    StartListen(server);
}


private void StartListen(NamedPipeServerStream server)
{
    Task.Run(async () =>
    {
        int bytesToRead;
        byte[] buffer;

        // server.WaitForConnection();
        while (true)
        {
            try
            {
                bytesToRead = 256 * server.ReadByte();
                bytesToRead += server.ReadByte();

                // 演示
                // 将收到的数据立马发送出去
                server.WriteByte((byte)(bytesToRead / 256));
                server.WriteByte((byte)(bytesToRead % 256));

                buffer = new byte[bytesToRead];
                server.Read(buffer, 0, bytesToRead);  // 读取消息

                // 演示
                // 将收到的数据立马发送出去
                await server.WriteAsync(buffer, 0, bytesToRead);
            }
            catch (System.IO.IOException)
            {
                // break if another pipe end closed
                break;
            }
            catch (Exception)
            {
                break;
            }

            // 处理数据
            string content = Encoding.UTF8.GetString(buffer);
            Console.WriteLine(content);
        }

        server.Disconnect();  // 断开连接
        // 重新等待连接
        server.BeginWaitForConnection(new AsyncCallback(WaitConnectionCallback), server);
    });
}
```

在客户端连接之后，立马启动一个新的线程循环读取来自客户端的消息，这里的消息前两个字节指定了消息的长度。同时将收到的消息马上返回到管道的另一端（这里用于测试是否真的是全双工工作）。

最后将消息的读取放到`try{ ... } catch(...){ ... }`中，因为并没有消息或者事件通知服务端客户端已经断开连接。但当客户端断开之后，服务端在读取时会抛出`IOException`，可以通过抓取这个错误来判断管道是否已经断开。

当客户端断开之后，中断读取循环，服务端也断开连接，并再次等待客户端连接。

## 客户端（Client）

**创建客户端**

``` csharp
NamedPipeClientStream client = new NamedPipeClientStream(
    ".", // The name of the remote computer, "." 指本机
    "SimpleServer",  // pipe name
    PipeDirection.InOut,  // 数据传输方向
    PipeOptions.Asynchronous | PipeOptions.WriteThrough);
```

**连接到服务端**

``` csharp
client.Connect();  // 阻塞方式
```

``` csharp
Task.Run(() =>
{
    int bytesToRead;
    byte[] buffer;

    client.Connect();  // 连接管道
    while (true)
    {
        try
        {
            // 读取消息长度
            bytesToRead = 256 * client.ReadByte();
            bytesToRead += client.ReadByte();

            buffer = new byte[bytesToRead];
            client.Read(buffer, 0, bytesToRead);  // 读取消息
        }
        catch (System.IO.IOException)
        {
            break;
        }
        catch (Exception)
        {
            break;
        }

        // 处理消息
        string content = Encoding.UTF8.GetString(buffer);
        Console.WriteLine(content);
    }

    client.Close();
});
```

这里使用一个新的线程去连接管道服务端，连接成功后循环读取来自服务端的消息。

**客户端发送**

``` csharp
string text = "Hello";
byte[] bytes = System.Text.Encoding.UTF8.GetBytes(text);

client.WriteByte((byte)(bytes.Length / 256));  // 发送消息头
client.WriteByte((byte)(bytes.Length % 256));
client.Write(bytes, 0, bytes.Length);          // 发送消息

text = "World";
bytes = System.Text.Encoding.UTF8.GetBytes(text);

client.WriteByte((byte)(bytes.Length / 256));  // 发送消息头
client.WriteByte((byte)(bytes.Length % 256));
client.Write(bytes, 0, bytes.Length);          // 发送消息
```

## 总结

总的来说，`Named Pipe`使用还是比较简单的，结合序列化就可以直接在两个进程中传递消息对象了。需要注意的是一个服务端只能有一个客户端连接，而且在客户端断开连接之后，服务端也需要断开连接，并重新等待客户端连接，不然再有客户端尝试连接管道也无法建立。

**参考**

[How to: Use Named Pipes for Network Interprocess Communication](https://docs.microsoft.com/en-us/dotnet/standard/io/how-to-use-named-pipes-for-network-interprocess-communication)
