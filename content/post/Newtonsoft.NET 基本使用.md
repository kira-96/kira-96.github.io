---
title: Newtonsoft.NET 基本使用
date: 2020-02-22T12:24:48+08:00
draft: false
description: Newtonsoft.NET 基本使用
tags: [ "C#" , "JSON" ]
keywords: [ "C#" , "JSON" ]
categories: [ "编程" ]
isCJKLanguage: true
---

## 简介

[JSON](https://baike.baidu.com/item/JSON)是一种常用的轻量级数据交换格式。与[XML](https://baike.baidu.com/item/%E5%8F%AF%E6%89%A9%E5%B1%95%E6%A0%87%E8%AE%B0%E8%AF%AD%E8%A8%80/2885849?fromtitle=xml&fromid=86251&fr=aladdin)相比，JSON无论是体积还是可读性都更好，所以在网络数据传输和应用程序中被广泛的应用。

那么，.NET平台使用最广泛的JSON库是什么呢？自然要数[Newtonsoft.NET](https://www.newtonsoft.com/json)了，打开[nuget](https://www.nuget.org/)包管理器第一个就是，在所有包下载量排行中排名第一。使用简单，性能可靠，[文档](https://www.newtonsoft.com/json/help/html/Introduction.htm)也很齐全。

## 使用

使用JSON最常用的就是对象的序列化和反序列化。

先来看最基本的使用

``` csharp
// 先定义一个类
public class TestJsonDeseClass
{
    public Guid MessageGuid { get; set; }

    public string Message { get; set; }
}
```

```csharp
TestJsonDeseClass test = new TestJsonDeseClass()
{
    MessageGuid = Guid.NewGuid(),
    Message = "Test Message"
};
string json = JsonConvert.SerializeObject(test);
TestJsonDeseClass des =
    JsonConvert.DeserializeObject<TestJsonDeseClass>(json);
```

只需要将类的成员属性设置为`get`和`set`就可以了，反序列化的时候，Json.NET会自动根据成员的名字为对象的成员赋值。

那么如果不想序列化/反序列化某个成员变量呢？

``` csharp {5}
using Newtonsoft.Json;

public class TestJsonDeseClass
{
    [JsonIgnore]
    public Guid MessageGuid { get; set; }

    public string Message { get; set; }
}
```

只需要在成员变量的定义前加上`[JsonIgnore]`的属性（Attribute）即可，序列化/反序列化的时候Json.NET会自动忽略该成员。

如果JSON字符串中的属性名字和定义的类中的成员名字不一样怎么办呢？怎样才能正确的给成员变量赋值呢？

```csharp {5}
using Newtonsoft.Json;

public class TestJsonDeseClass
{
    [JsonProperty("Guid")]
    public Guid MessageGuid { get; set; }

    public string Message { get; set; }
}
```

只需要在成员变量的定义前加上`[JsonProperty()]`的属性（Attribute）即可，序列化/反序列化的时候Json.NET会将Json字符串中的`"Guid"`属性赋值给`MessageGuid`。

那么，如果想让类的属性值只读的`get`，不想让外部能修改成员变量呢，如何设置呢？

当然这样也是可以的，不过需要我们给类添加构造方法，在构造方法中对成员赋值，不能再使用默认的构造方法，因为默认的构造方法不会对成员赋值，而外部也无法对成员赋值。在添加了构造方法后，Json.NET会自动调用类的构造方法。

``` csharp {9}
using Newtonsoft.Json;

public class TestJsonDeseClass
{
    public Guid MessageGuid { get; }

    public string Message { get; }

    public TestJsonDeseClass(Guid messageGuid, string message)
    {
        MessageGuid = messageGuid;
        Message = message;
    }
}
```

不过需要注意的是，构造方法的参数名称必须和成员变量（或者说是序列化/反序列化时的属性）名字一致，但可以不用区分大小写，才能正确对属性赋值。

如果把上面的构造方法改成下面这个样子

``` csharp {1}
public TestJsonDeseClass(Guid guid, string message)
{
    MessageGuid = guid;
    Message = message;
}
```

就会导致反序列化的对象`MessageGuid`属性不能正确赋值，因为Json.NET无法从json字符串中找到名为`guid`的属性，你也没告诉它要拿名为`MessageGuid`的属性，自然就会出错了。

那么，最后一个问题，如果我的类有多个构造方法，我怎样告诉Json.NET应该用哪一个呢？

``` csharp {13}
using Newtonsoft.Json;

public class TestJsonDeseClass
{
    public Guid MessageGuid { get; }

    public string Message { get; }

    public TestJsonDeseClass()
    {
    }

    [JsonConstructor]
    public TestJsonDeseClass(Guid messageGuid, string message)
    {
        MessageGuid = messageGuid;
        Message = message;
    }
}
```

只需要在对应的构造方法前面加上`[JsonConstructor]`的属性（Attribute）即可，反序列化的时候Json.NET会就会调用相应的构造方法来生成对象了。

以上就是一些基本的用法，基本上能满足正常的使用了。当然还有一些更加高级和灵活的用法，这里就不多记录了，需要的时候再去看文档就可以了。

**参考**

[文档](https://www.newtonsoft.com/json/help/html/Introduction.htm)

[Samples](https://www.newtonsoft.com/json/help/html/Samples.htm)
