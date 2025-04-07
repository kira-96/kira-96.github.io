---
title: "QML控件访问EPICS过程变量的思路"
date: 2025-02-20T16:27:36+08:00
lastmod: 2025-03-17T08:53:34+08:00
draft: false
description: 自定义QML控件访问EPICS过程变量的思路
searchHidden: true
tags: ["EPICS", "Qt"]
keywords: ["EPICS", "Qt"]
categories: ["EPICS"]
---

## 前言

写这篇文章的想法是使用自定义QML控件的方式访问EPICS过程变量，现有的Qt EPICS框架[之前](../epics-qt安装/)也介绍过了，还有配套的**QEGui**工具，框架的基本功能已经比较完善，但实际使用过程中还是遇到了一些问题。我总结有以下2点：

1. QEGui工具只能加载基于Widget的.ui界面文件，文件只能包含布局、控件和控件属性，不能嵌入代码（C++代码必须编译）。用户的操作（如：输入、点击按钮等）的相应槽函数封装在框架自定义的控件里实现，这样极大方便了用户的开发，用户只需要设计界面，填写变量名就可以实现变量访问，但相应的也失去了很大的灵活性，程序的功能变得很单一。

2. QEGui工具不一定符合实际开发过程中的需求，仅使用QEGui加载.ui界面，脱离了C++代码，程序很难实现用户想要的效果。而更好的选择是调用Qt EPICS框架动态库提供的控件，结合C++代码开发应用程序，但却不得不进行编译操作。

那么，有没有更好的基于Qt的EPICS框架方案呢？

刚好最近我也学习实践了QML相关的内容，这种前后端分离开发的方式给了我一些灵感。我们完全可以在Qt EPICS框架的基础上，实现自定义QML控件访问EPICS过程变量，用户使用QML编写界面，调用自定义QML控件即可，可以实现和Qt EPICS框架加载.ui界面文件类似的功能。但QML在动画、3D显示等方面明显具有优势，还有一点是基于Widget的.ui界面文件不具备的：QML本身可以嵌入`javascript`函数，动态控制界面的显示、切换等，甚至可以实现和底层C++接口的交互。而QML本身也不需要进行编译，完全可以只使用QML语言实现程序开发，且具有很高灵活性。

## 实现思路

![QML EPICS](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/QML+EPICS.excalidraw.svg)

Qt EPICS 框架提供了`QCaObject`类访问EPICS过程变量，但该类几乎是完全为`QEWidget`服务的，并没有声明属性（property），无法直接在QML中使用。

所以第1步需要对`QCaObject`做一层封装，将EPICS过程变量的字段声明为`QPvObject`类的属性，然后就可以在QML中访问EPICS过程变量了。

第2步是将`QPvObject`封装进自定义的QML控件`QmlPvControl`，实现数据的显示、输入等操作。

第3步将`QmlPvControl`导入（import）到QML文件，在外部QML文件中使用自定义控件。

## 代码示例

由于完整的代码较多，这里只放部分代码。

**QPvObject类的定义**

``` cpp { title="qpvobject.h" }
class QPvObject : public QObject
{
    Q_OBJECT
public:
    enum epicsAlarmSeverity {
        NO_ALARM,              /**< No alarm */
        MINOR_ALARM,           /**< Minor alarm severity */
        MAJOR_ALARM,           /**< Major alarm severity */
        INVALID_ALARM,         /**< Invalid alarm severity */
        ALARM_NSEV             /**< Number of alarm severities */
    };
    Q_ENUM(epicsAlarmSeverity)
    
    enum pvConnectionMode {
        NONE,
        WR_ONLY,     /**< Write only */
        RD_ONCE,     /**< Read once */
        MONITOR      /**< Read and Write */
    };
    Q_ENUM(pvConnectionMode)

private:
    Q_PROPERTY(QString pvName READ pvName WRITE setPvName NOTIFY pvNameChanged FINAL)
    Q_PROPERTY(QVariant value READ value WRITE setValue NOTIFY valueChanged FINAL)
    Q_PROPERTY(QPvObject::pvConnectionMode mode READ mode WRITE setMode NOTIFY modeChanged FINAL)
    Q_PROPERTY(QString hostName READ hostName NOTIFY hostNameChanged FINAL)
    Q_PROPERTY(QString fieldType READ fieldType NOTIFY fieldTypeChanged FINAL)
    Q_PROPERTY(QString descriptor READ descriptor NOTIFY descriptorChanged FINAL)
    Q_PROPERTY(QString egu READ egu NOTIFY eguChanged FINAL)
    Q_PROPERTY(QCaDateTime dateTime READ dateTime NOTIFY dateTimeChanged FINAL)
    Q_PROPERTY(quint16 status READ status NOTIFY statusChanged FINAL)
    Q_PROPERTY(QPvObject::epicsAlarmSeverity severity READ severity NOTIFY severityChanged FINAL)
    Q_PROPERTY(QString statusName READ statusName NOTIFY statusNameChanged FINAL)
    Q_PROPERTY(QString severityName READ severityName NOTIFY severityNameChanged FINAL)
    Q_PROPERTY(quint32 hostElementCount READ hostElementCount NOTIFY hostElementCountChanged FINAL)
    Q_PROPERTY(quint32 dataElementCount READ dataElementCount NOTIFY dataElementCountChanged FINAL)
    Q_PROPERTY(bool readAccess READ readAccess NOTIFY readAccessChanged FINAL)
    Q_PROPERTY(bool writeAccess READ writeAccess NOTIFY writeAccessChanged FINAL)

public:
    QString pvName() const;
    QVariant value() const;
    QPvObject::pvProtocol protocol() const;
    QPvObject::pvConnectionMode mode() const;
    QString hostName() const;
    QString fieldType() const;
    QString descriptor() const;
    QString egu() const;
    const QCaDateTime& dateTime() const;
    quint16 status() const;
    QPvObject::epicsAlarmSeverity severity() const;
    QString statusName() const;
    QString severityName() const;
    quint32 hostElementCount() const;
    quint32 dataElementCount() const;
    bool readAccess() const;
    bool writeAccess() const;

public Q_SLOTS:
    virtual void setPvName(const QString &pvname) = 0;
    virtual void setValue(const QVariant &value) = 0;
    virtual void setMode(const QPvObject::pvConnectionMode &mode);

Q_SIGNALS:
    void pvNameChanged(const QString&);
    void valueChanged(const QVariant&);
    void protocolChanged(const QPvObject::pvProtocol&);
    void modeChanged(const QPvObject::pvConnectionMode&);
    void dateTimeChanged(const QCaDateTime&);
    void fieldTypeChanged(const QString&);
    void descriptorChanged(const QString&);
    void eguChanged(const QString&);
    void hostNameChanged(const QString&);
    void statusChanged(const quint16);
    void severityChanged(const QPvObject::epicsAlarmSeverity);
    void statusNameChanged(const QString);
    void severityNameChanged(const QString);
    void hostElementCountChanged(const quint32);
    void dataElementCountChanged(const quint32);
    void readAccessChanged(const bool);
    void writeAccessChanged(const bool);

protected:
    QPointer<qcaobject::QCaObject> m_caobject;
    // ...
};
```

注意到`QPvObject`类有两个虚函数`setPvName`和`setValue`，这两个函数需要子类实现。

在`setPvName`中实现`QCaObject`变量的实例化和EPICS过程变量的连接等操作，在`setValue`中将值写入到EPICS过程变量。

例如：整数类型的过程变量

``` cpp { title="qpvint.h" }
class QPvInt : public QPvObject
{
    Q_OBJECT
    Q_DISABLE_COPY(QPvInt)
public:
    explicit QPvInt(QObject *parent = Q_NULLPTR);

public Q_SLOTS:
    virtual void setPvName(const QString &pvname) Q_DECL_OVERRIDE;
    virtual void setValue(const QVariant &value) Q_DECL_OVERRIDE;
};
```

```cpp {title="qpvint.cpp", linenos=inline, hl_lines=["16-21"]}
QPvInt::QPvInt(QObject *parent)
    : QPvObject{parent}
{}

void QPvInt::setPvName(const QString &pvname)
{
    if (!pvname.isEmpty() && pvname != m_pv_name) {
        m_pv_name = pvname;
        Q_EMIT pvNameChanged(pvname);

        if (!m_caobject.isNull()) {
            m_caobject->closeChannel();
            m_caobject->deleteLater();
        }

        m_caobject = new QEInteger(pvname, this, Q_NULLPTR, 0);
        connect(m_caobject, &QCaObject::connectionChanged, this, &QPvInt::onConnectionChanged);
        connect(m_caobject,
                QOverload<const QVariant&, QCaAlarmInfo&, QCaDateTime&, const unsigned int&>::of(&QCaObject::dataChanged),
                this, &QPvInt::onDataChanged);
        m_caobject->subscribe();
    }
}

void QPvInt::setValue(const QVariant &value)
{
    if (!m_write_access) {
        return;
    }

    if (value != m_value) {
        bool ok = false;
        int val = value.toInt(&ok);
        if (ok) {
            m_caobject->writeIntegerValue(val);
        }
    }
}
```

根据Qt EPICS框架提供的数据类型，定义QML中可使用的过程变量类，如：`QPvInt`、`QPvDouble`、`QPvString`。

然后需要注册自定义的数据类型，才能在QML中使用。

``` cpp { title="main.cpp" }
qmlRegisterUncreatableType<QPvObject>(uri, 1, 0, "QPvObject", "Not creatable as it is an abstract class");
qmlRegisterType<QPvInt>("com.example.epics", 1, 0, "QPvInt");
qmlRegisterType<QPvDouble>("com.example.epics", 1, 0, "QPvDouble");
qmlRegisterType<QPvString>("com.example.epics", 1, 0, "QPvString");
```

**QmlPvControl 控件的定义**

这里自定义了*Label*控件，可以自动更新EPICS过程变量的值，根据严重等级自动改变控件背景色。控件声明了`pvName`属性，用户在使用时需要填写此项才可以连接到EPICS过程变量。

``` qml { title="PvLabel.qml" }
import QtQuick 2.15
import QtQuick.Controls 2.15
import com.example.epics 1.0

Label {
    id: root
    /* 声明pvName属性 */
    property alias pvName: pv.pvName
    /* 提示信息 */
    ToolTip.delay: 1000
    ToolTip.visible: mouseArea.containsMouse
    ToolTip.text: pvName
    /* 背景色 */
    background: Rectangle {
        id: backgroundRect
    }

    MouseArea {
        id: mouseArea
        anchors.fill: parent
        hoverEnabled: true
    }
    /* EPICS过程变量 */
    QPvString {
        id: pv

        onValueChanged: {
            console.log(pvName + " value changed to: " + value)
            root.text = value.toString()
        }

        onSeverityChanged: (severity) => {
            switch (severity) {
            case QPvObject.NO_ALARM:
                backgroundRect.color = 'transparent'
                break
            case QPvObject.MINOR_ALARM:
                backgroundRect.color = 'yellow'
                break
            case QPvObject.MAJOR_ALARM:
                backgroundRect.color = 'red'
                break
            case QPvObject.INVALID_ALARM:
                backgroundRect.color = '#FF00FF'
                root.text = '---'
                break
            default:
                backgroundRect.color = 'lightgray'
                break
            }
        }
    }
}
```

**使用自定义控件**

``` qml { title="main.qml" }
import QtQuick 2.15
import QtQuick.Window 2.15
// 导入自定义控件
import 'qrc:/'

Window {
    id: window
    width: 640
    height: 480
    visible: true
    title: qsTr("Hello World")  // @disable-check M16

    PvLabel {
        id: testLabel
        pvName: "ca://user:circle:angle"
    }

    PvLabel {
        id: baseLabel
        pvName: "user:iocExample:version"
    }

    PvEdit {
        id: testEdit
        pvName: "pva://user:circle:period"
    }
}
```

`pvName`支持`CA`/`PVA`协议的变量，例如：  
`ca://user:circle:angle`，使用`CA`协议  
`pva://user:circle:angle`，使用`PVA`协议  
`user:circle:angle`，默认使用`CA`协议  

最后，定义程序启动时加载的qml文件路径即可。

``` cpp { title="main.cpp " }
QQmlApplicationEngine engine;
// 定义qml文件的路径或通过 argv 参数传入
const QUrl url("./main.qml");
engine.load(url);
```

1. ~~使用`qputenv`设置**EPICS**相关的环境变量。~~
2. 可使用传入程序的参数动态加载界面文件。例如：*myapp myui.qml*

**运行结果**

![QML EPICS](https://cdn.jsdelivr.net/gh/kira-96/Picture@main/blog/images/PixPin_2025-02-20_15-48-48.png)

## 总结

本文给出了自定义QML控件实现EPICS过程变量访问的思路，后续可能需要添加更多的自定义控件和更多的功能实现。
