---
title: "Qt5编写Qt Quick2插件"
date: 2025-03-13T10:23:47+08:00
lastmod: 2025-03-23T10:55:52+08:00
draft: false
description: 使用Qt5编写Qt Quick2插件遇到的问题记录
tags: ["Qt", "QML"]
keywords: ["Qt", "QML"]
categories: ["Qt"]
---

## 前言

最近在试着把自定义的QML控件封装成插件供其他程序调用，由于我使用的还是Qt5，在构建的过程中遇到不少问题，这里做一下简单汇总。

## 新建项目

新建*库→Qt Quick2 Extension Plugin*，`Qt 5`版本仅支持使用`QMake`作为构建套件。`Qt 6`才支持使用`CMake`作为QML扩展插件的构建套件。

## 编写插件

例如，我新建了一个叫做`Test`的插件库，项目会自动创建一个`TestPlugin`的类，这个类继承自`QQmlExtensionPlugin`，我们需要重写它的两个虚函数。

``` cpp
/* testplugin.h */
#include <QQmlExtensionPlugin>

class TestPlugin : public QQmlExtensionPlugin
{
    Q_OBJECT
    Q_PLUGIN_METADATA(IID QQmlExtensionInterface_iid)

public:
    void registerTypes(const char *uri) Q_DECL_OVERRIDE;
    void initializeEngine(QQmlEngine *engine, const char *uri) Q_DECL_OVERRIDE;
};
```

``` cpp
/* testplugin.cpp */
#include "testplugin.h"
#include "myitem.h"
#include <qqml.h>

void TestPlugin::registerTypes(const char *uri)
{
    // @uri com.mycompany.qmlcomponents
    qmlRegisterType<MyItem>(uri, 1, 0, "MyItem");
}

void TestPlugin::initializeEngine(QQmlEngine *engine, const char *uri)
{
    QQmlExtensionPlugin::initializeEngine(engine, uri);
}
```

其中，`MyItem`就是我们自定义的控件。简单看一下示例代码：

``` cpp
/*myitem.h*/
#include <QtQuick/QQuickPaintedItem>

class MyItem : public QQuickPaintedItem
{
    Q_OBJECT
    QML_ELEMENT
    Q_DISABLE_COPY(MyItem)
public:
    explicit MyItem(QQuickItem *parent = Q_NULLPTR);
    void paint(QPainter *painter) Q_DECL_OVERRIDE;
    ~MyItem() Q_DECL_OVERRIDE;
};
```

`MyItem`继承自`QQuickItem`，并使用`QML_ELEMENT`声明了它是一个QML元素，最后再使用`qmlRegisterType`注册。这样`MyItem`就可以在QML中使用了。

使用这种方式也可以注册继承自`QObject`的类。例如：

``` cpp
/* myobject.h */
class MyObject : public QObject
{
    Q_OBJECT
    Q_DISABLE_COPY(MyObject)
public:
    explicit MyObject(QObject *parent = Q_NULLPTR);
};
```

然后在`Plugin`类中进行注册：

``` cpp
/* testplugin.cpp */
void TestPlugin::registerTypes(const char *uri)
{
    // @uri com.mycompany.qmlcomponents
    qmlRegisterType<MyItem>(uri, 1, 0, "MyItem");
    qmlRegisterType<MyObject>(uri, 1, 0, "MyObject");
}
```

## 自定义QML控件

QML插件库当然也可以封装`.qml`的自定义插件。例如：

``` qml
/* MyLabel.qml */
import QtQuick 2.15
import QtQuick.Controls 2.15

Label {
    id: control

    property string backgroundColor

    background: Rectangle {
        color: backgroundColor
    }
}
```

把自定义的qml控件添加到资源文件，然后可以在`Plugin`类中进行注册。

``` cpp
/* testplugin.cpp */
void TestPlugin::registerTypes(const char *uri)
{
    // @uri com.mycompany.qmlcomponents
    qmlRegisterType<MyItem>(uri, 1, 0, "MyItem");
    qmlRegisterType<MyObject>(uri, 1, 0, "MyObject");

    // register qml types
    qmlRegisterType(QUrl("qrc:/qml/MyLabel.qml"),
                    uri, MAJOR, MINOR, "MyLabel");
}
```

## qmldir文件

`qmldir`文件用于QML插件模块管理，描述了qml插件的基本信息。示例：

``` shell
module com.mycompany.qmlcomponents
plugin testplugin
classname TestPlugin
typeinfo plugins.qmltypes
designersupported
depends QtQuick.Controls 2.12
```

- `module` 和插件的uri配置相同
- `plugin` 插件文件名字，通常是小写
- `classname` 插件类的名字
- `typeinfo` 插件的元数据文件，稍后会写怎么生成这个文件

## 项目文件配置

下面是我的一个项目配置，包含插件信息配置，和**生成元数据的自定义命令**。

``` shell
## TestPlugin.pro

TEMPLATE = lib
TARGET = testplugin
QT += core qml quick
CONFIG += plugin c++11
# CONFIG += qmltypes

# 插件的URI
uri = com.mycompany.qmlcomponents

QML_PLUGIN_NAME = $$TARGET
# 可以通过 import 的方式在qml中使用插件中的控件
QML_IMPORT_NAME = $$uri
# 插件的版本号
QML_IMPORT_MAJOR_VERSION = 1
QML_IMPORT_MINOR_VERSION = 0
# 生成插件的路径
DESTDIR = imports/$$replace(QML_IMPORT_NAME, \., $$QMAKE_DIR_SEP)
QMLTYPES_FILENAME = $$DESTDIR/plugins.qmltypes

TARGET = $$qtLibraryTarget($$TARGET)

HEADERS += ...
SOURCES += ...

# 拷贝qmldir文件到输出目录
DISTFILES = qmldir
!equals(_PRO_FILE_PWD_, $$OUT_PWD) {
    copy_qmldir.target = $$DESTDIR/qmldir
    copy_qmldir.depends = $$_PRO_FILE_PWD_/qmldir
    copy_qmldir.commands = $(COPY_FILE) "$$replace(copy_qmldir.depends, /, $$QMAKE_DIR_SEP)" "$$replace(copy_qmldir.target, /, $$QMAKE_DIR_SEP)"
    QMAKE_EXTRA_TARGETS += copy_qmldir
    PRE_TARGETDEPS += $$copy_qmldir.target
}

# 添加构建后生成元数据的自定义命令
qmltypes.commands = $$[QT_INSTALL_PREFIX]/bin/qmlplugindump -nonrelocatable $$QML_IMPORT_NAME "$${QML_IMPORT_MAJOR_VERSION}.$${QML_IMPORT_MINOR_VERSION}" $$OUT_PWD/imports > $$OUT_PWD/$$DESTDIR/plugins.qmltypes
qmltypes.depends = $$QML_PLUGIN_NAME.target
QMAKE_EXTRA_TARGETS += qmltypes
# 确保构建后执行元数据生成
# POST_TARGETDEPS += $$QML_PLUGIN_NAME
POST_TARGETDEPS += qmltypes
```

这里有几点需要注意的：

- 有资料说加入 `CONFIG += qmltypes` 配置就可以自动生成插件元数据信息，这点我测试确实可以，但生成的元数据是不完整的。这个配置只能生成使用了`QML_ELEMENT`宏的类，无法生成`QObject`类和`QML`控件的元数据，基本上无用。
- 生成元数据必须使用`qmlplugindump`工具，这个工具和`qmake`在同一个目录下。

## qmlplugindump工具的使用

这里使用的命令如下：

`qmlplugindump -nonrelocatable QML导入URI 插件版本 插件目录 > 生成plugins.qmltypes路径`

- QML导入URI：编译插件时指定的uri，如：com.mycompany.qmlcomponents
- 插件本本：编译插件时指定的版本，如：1.0
- **插件目录**：这个需要特别注意，这个指的是插件的目录，不是插件库的目录。比如我生成的插件是在`imports`目录，构建套件会根据插件的uri自动生成插件库的目录，如：`imports/com/mycompany/qmlcomponents/testplugin.so`，那么这里应该传入参数`/path/to/imports`，不要传入插件库所在的目录。
- 生成plugins.qmltypes路径：这个没什么好解释的，但注意`plugins.qmltypes`需要和插件库放在统一目录，所以这里最后传入参数`/path/to/imports/com/mycompany/qmlcomponents/plugins.qmltypes`。

注意：
1. qmldir文件和生成的插件需要放到同一目录下，否则`qmlplugindump`读不到插件信息。
2. 如果生成的插件依赖其他动态库，一定要确保`qmlplugindump`能找到这些库，否则插件无法加载就不能生成元数据。

问题：
1. plugin cannot be loaded for module "com.mycompany.qmlcomponents": Cannot load library testplugin.dll: 找不到指定的模块。(检查动态库的路径，可尝试添加到环境变量)
2. QQmlComponent: Component is not ready
  file:///path/to/imports/com/mycompany/qmlcomponents/qmldir: plugin cannot be loaded for module "": Module namespace 'com.mycompany.qmlcomponents' does not match import URI ''
  (检查传入参数的*插件目录*是否正确？)

## 使用QML插件

1. 确保插件目录中包含以下几个文件：testplugin.so、plugins.qmltypes、qmldir。
2. 将插件目录添加到Qml Emgine的导入目录列表：
  ``` cpp
  /* main.cpp */
  QGuiApplication app(argc, argv);
  QQmlApplicationEngine engine(&app);
  ...
  // 添加插件路径到导入目录列表
  engine.addImportPath("imports");
  engine.load(url);
  ```
3. 在qml中导入、使用自定义控件
  ``` qml
  /* main.qml */
  import com.mycompany.qmlcomponents 1.0

  Window {
    id: window
    width: 640
    height: 480
    visible: true
    title: qsTr("Hello World")  // @disable-check M16

    MyItem {
        id: myitem
    }

    MyLabel {
        id: mylabel
        text: qsTr("Hello World")
        backgroundColor: 'red'
    }
  }
  ```

## 如何使用插件中的静态函数？

这里需要对静态函数做一下封装。例如，我定义了一个静态函数：

``` cpp
/* myutils.h */
class MyUtils
{
public:
    static void myFunc()
    {
        printf("Hello from myFunc().");
    }
};
```

这里需要定义一个插件的Helper类，然后在插件里面注册这个类，在这个类里面实现静态函数调用。例如：

``` cpp
/* testpluginhelper.h */
#include <QObject>
#include "myutils.h"

class TestPluginHelper : public QObject
{
    Q_OBJECT
public:
    Q_INVOKABLE void callMyFunc()
    {
        MyUtils::myFunc();
    }
};
```

``` cpp
/* testplugin.cpp */
void TestPlugin::initializeEngine(QQmlEngine *engine, const char *uri)
{
    QQmlExtensionPlugin::initializeEngine(engine, uri);
    // 注册对象到上下文
    engine->rootContext()->setContextProperty("testPluginHelper", new TestPluginHelper);
}
```

在QML中调用：

``` qml
Item {
    id: item

    Component.onCompleted: {
        testPluginHelper.callMyFunc()
    }
}
```
