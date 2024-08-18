---
title: osd_server：基于Qt中的QDBus实现一个Server端
date: 2024-08-18 17:29:35
tags:
	- Qt
	- CMake
	- D-Bus
	- GSettings
	- Linux
categories: 桌面项目
keywords: 
	- Qt
	- CMake
	- D-Bus
	- Linux
	- GSettings
description: 提供一个接口方法，注册在D-Bus总线上。桌面组件可以通过D-Bus调用这个方法，并且传入一个字符串作为图标名。之后会在桌面屏幕的特定位置根据图标名来显示特定的图标以及相应的视觉效果，并且随着主题切换还会实时刷新显示效果。
cover: https://1807962239hml-1324287819.cos.ap-beijing.myqcloud.com/202408161559223.png
---

# 1 主要功能
- 提供一个接口方法，注册在D-Bus总线上。桌面组件可以通过D-Bus调用这个方法，并且传入一个字符串作为图标名。之后会在桌面屏幕的特定位置根据图标名来显示特定的图标以及相应的视觉效果，并且随着主题切换还会实时刷新显示效果。

- **效果**：

![](https://1807962239hml-1324287819.cos.ap-beijing.myqcloud.com/202408161559223.png)

![](https://1807962239hml-1324287819.cos.ap-beijing.myqcloud.com/202408161603934.png)

# 2 部分模块使用

## LinguistTools

1、在实际代码中将需要翻译的字符串用**tr()** 标记。

`tr("我爱学习")`


2、在项目目录用**lupdate**创建 **.ts文件**，会根据项目中被标记的内容生成。

```
mkdir translations
cd translations
lupdate *.cpp *.h -ts en.ts
```

3、手动编辑生成的 **.ts文件**（需要自己翻译）。

4、用**lrelease**根据 **.ts文件**生成 **.qm文件**。

`lrelease en.ts -qm en.qm`

5、**CMakeLists.txt**中添加如下内容。

```
# 找包
find_package(Qt5 COMPONENTS LinguistTools REQUIRED)


#定义资源文件位置

file(GLOB RESOURCE_FILES "*.qrc")

  

#生成资源文件变量

qt5_add_resources(QRC_RCC ${RESOURCE_FILES})

  

#将资源文件添加到可执行文件

target_sources(osd_server PRIVATE ${QRC_RCC})
```

6、在程序中导入 **.qm文件**，记得导入头文件 `#include <QTranslator>`，需要导入多个则需要创建多个**QTranslator**对象。

```cpp
QTranslator translator;

//导入qm翻译文件
if (translator.load(":/translations/en.qm")) {

    a.installTranslator(&translator);

}
```

7、最后程序运行时，被翻译的字符串会根据系统的语言来显示翻译的。如果需要翻译更多的语言，需要手动创建更多 **.ts文件**。

# 3 具体实现

## 3.1 CMakeLists.txt

```
cmake_minimum_required(VERSION 3.5)

  

project(osd_server LANGUAGES CXX)

  

set(CMAKE_INCLUDE_CURRENT_DIR ON)

  

set(CMAKE_AUTOUIC ON)

set(CMAKE_AUTOMOC ON)

set(CMAKE_AUTORCC ON)

  

set(CMAKE_CXX_STANDARD 11)

set(CMAKE_CXX_STANDARD_REQUIRED ON)

  

  

find_package(PkgConfig REQUIRED)

find_package(Qt5 COMPONENTS Core Widgets DBus LinguistTools REQUIRED)

  

find_package(KF5WindowSystem REQUIRED)

  

add_executable(osd_server

    main.cpp

    myservers.h

    myservers.cpp

    myadaptor.cpp

    myadaptor.h

    servermanager.cpp

    servermanager.h

    widget.cpp

    widget.h

    abstractwidget.cpp

    abstractwidget.h

)

  

#定义资源文件位置

file(GLOB RESOURCE_FILES "*.qrc")

  

#生成资源文件变量

qt5_add_resources(QRC_RCC ${RESOURCE_FILES})

  

#将资源文件添加到可执行文件

target_sources(osd_server PRIVATE ${QRC_RCC})

  

  
pkg_check_modules(gsettings-qt REQUIRED IMPORTED_TARGET gsettings-qt)

  

target_link_libraries(osd_server PRIVATE Qt5::Widgets Qt5::DBus Qt5::Core PkgConfig::gsettings-qt KF5::WindowSystem)
```

## 3.2 Widget类

- 主要实现窗体的显示设计，包含窗体的初始化以及绘画，还有其他一些显示逻辑。

### widget.h

```cpp
#ifndef WIDGET_H

#define WIDGET_H

  

#include <QWidget>

#include <QPixmap>

#include <QIcon>

#include <QLabel>

#include <QPainter>

#include <QGSettings>

  

class Widget : public QWidget

{

    Q_OBJECT

  

public:

    /**

     * @brief Widget

     * @param parent

     */

    Widget(QObject *parent = nullptr);

    ~Widget();

  

private:

  

    QLabel *p_mIconLabel = nullptr;

  

    QFrame *p_mBackGroundFrame = nullptr;

  

    QPixmap m_iconPixmap;

  

    QString m_localIconName = "";

  

    QString m_localIconPath = "";

  

    QGSettings *p_mThemeSettings = nullptr;

  

private:

  

    /**

     * @brief initLabel

     */

    void initLabel();

  

    void initFrame();

  

    void initWidget();

  

    double getGlobalOpacity();

  

    QPixmap drawLightColoredPixmap(QPixmap source,QString theme);

  

public:

  

    //给适配器设定图标的方法

    void setIconPixmap(QString iconName);

  

public Q_SLOTS:

  

    void onThemeChanged();

  

public:

    void paintEvent(QPaintEvent *event);

};

#endif // WIDGET_H
```

### widget.cpp

```cpp
#include "widget.h"

  

#include <QDebug>

#include <QScreen>

#include <QApplication>

#include <QPainterPath>

#include <KWindowEffects>

  

#define ICONSIZE 48

#define WIDGETSIZE 92

#define BACKGROUNDSIZE 72

  

#define QT_THEME_SCHEMA "org.ukui.style"

  

#define PANEL_SCHEMA "org.ukui.panel.settings"

#define PANEL_SIZE_KEY "panelsize"

  

#define LOCALICONPREFIX ":/ukui_res/ukui/"

  

  

Widget::Widget(QObject *parent)

{

    //获取系统主题信息

    p_mThemeSettings = new QGSettings(QT_THEME_SCHEMA);

  

    initWidget();

  

    connect(p_mThemeSettings,&QGSettings::changed,this,&Widget::onThemeChanged);

}

  

Widget::~Widget()

{

    delete p_mThemeSettings;

    p_mThemeSettings = nullptr;

}

  

//初始化界面信息

void Widget::initWidget(){

    //设置窗口属性

    //参数依次为无边框、较小的工具窗口、保持窗口永远在最上面、绕过窗口管理器（此窗口完全不受管理）、弹出式窗口

    setWindowFlags(Qt::FramelessWindowHint |

                   Qt::Tool |

                   Qt::WindowStaysOnTopHint |

                   Qt::X11BypassWindowManagerHint |

                   Qt::Popup);

  

    //获取屏幕尺寸

    QScreen *screen = QApplication::primaryScreen();

    QRect rect = screen->geometry();

    QSize size = screen->size();

    int x = rect.x();

    int y = rect.y();

    int width = size.width();

    int height = size.height();

  

    //根据schema文件的屏幕属性去适配位置

    int pSize = 0;

    const QByteArray id(PANEL_SCHEMA);

    if (QGSettings::isSchemaInstalled(id)){

        QGSettings * settings = new QGSettings(id);

        pSize = settings->get(PANEL_SIZE_KEY).toInt();

  

        delete settings;

        settings = nullptr;

    }


    int ax,ay;

    ax = x+width - WIDGETSIZE - 200;

    ay = y+height - WIDGETSIZE - pSize - 8;

  
    setGeometry(ax,ay,WIDGETSIZE,WIDGETSIZE);

    setFixedSize(WIDGETSIZE,WIDGETSIZE);

  

    //设置背板颜色

    setAttribute(Qt::WA_TranslucentBackground);

  

    initFrame();

  

}

  

//初始化背景

void Widget::initFrame(){

  

    p_mBackGroundFrame = new QFrame(this);

    p_mBackGroundFrame->move(10,10);

  

    //设置背板颜色

    if(p_mThemeSettings->get("style-name").toString() == "ukui-light"){

        setPalette(QPalette(QColor("#F5F5F5")));

    } else{

        setPalette(QPalette(QColor("#232426")));

    }

  

  

    initLabel();

}

  

//初始化标签信息

void Widget::initLabel(){

  

    p_mIconLabel = new QLabel(p_mBackGroundFrame);

  

    //设置位置

    p_mIconLabel->setGeometry((BACKGROUNDSIZE - ICONSIZE)/2,(BACKGROUNDSIZE - ICONSIZE)/2,ICONSIZE,ICONSIZE);

  

    p_mIconLabel->setFixedSize(QSize(ICONSIZE,ICONSIZE));

  

    //设置背景透明

    p_mIconLabel->setAttribute(Qt::WA_TranslucentBackground);

  

}

void Widget::setIconPixmap(QString iconName){

  

    m_localIconName = iconName;

  

    m_localIconPath = QString(LOCALICONPREFIX) + m_localIconName + QString(".png");

  

    //根据图标名和完整路径初始化QIcon，再用pixmap将其转换成一定大小的QPixmap

    m_iconPixmap = QIcon::fromTheme(m_localIconName,QIcon(m_localIconPath)).pixmap(QSize(ICONSIZE,ICONSIZE));

  

    p_mIconLabel->setPixmap(drawLightColoredPixmap(m_iconPixmap,p_mThemeSettings->get("style-name").toString()));

}

  

//根据主题深浅改变图案颜色

QPixmap Widget::drawLightColoredPixmap(QPixmap source,QString theme)

{

    int adaptiveValue = 255;

  

    if(theme == "ukui-light"){

        adaptiveValue = 0;

    }

  

    QImage img = source.toImage();

    for (int x = 0; x < img.width(); x++) {

        for (int y = 0; y < img.height(); y++) {

            QColor color = img.pixelColor(x, y);

  

            //图像中的透明像素点不管

            if (color.alpha() > 0) {

                //因为要保留原图片有色像素点的透明度，所以继续用color去设置像素颜色

                color.setRed(adaptiveValue);

                color.setGreen(adaptiveValue);

                color.setBlue(adaptiveValue);

                img.setPixelColor(x, y, color);

  

            }

        }

    }

    return QPixmap::fromImage(img);

}

  

//系统主题改变时，做出相应改变

void Widget::onThemeChanged(){

  

  

  

    //先是更改背板颜色，也会调用重绘

    if(p_mThemeSettings->get("style-name").toString() == "ukui-light"){

        setPalette(QPalette(QColor("#F5F5F5")));

    } else{

        setPalette(QPalette(QColor("#232426")));

    }

  

  

    //重新更改图标的颜色并设置

    p_mIconLabel->setPixmap(drawLightColoredPixmap(m_iconPixmap,p_mThemeSettings->get("style-name").toString()));

  

  

}

  

  

//获取系统中的不透明度

double Widget::getGlobalOpacity()

{

    double transparency=0.0;

    if(QGSettings::isSchemaInstalled("org.ukui.control-center.personalise"))

    {

        QGSettings gsetting("org.ukui.control-center.personalise");

        if(gsetting.keys().contains(QString("transparency")))

            transparency=gsetting.get("transparency").toDouble();

    }

    return transparency;

}

  

  

  

void Widget::paintEvent(QPaintEvent *event)

{

  

  

    Q_UNUSED(event)

  

    QPainter painter(this);

  

  

    //绘制描边

    QPainterPath linePath;

  

    //添加圆角矩形路径

    //最后两个参数是圆角水平和垂直半径

    linePath.addRoundedRect(QRect(9,9,p_mBackGroundFrame->width()+1,p_mBackGroundFrame->height()+1),12,12);

  

    //将合成模式设置为差值混合模式，结合填充的黑色和背景的透明色去绘制特殊视觉效果

    painter.setCompositionMode(QPainter::CompositionMode_Difference);

    //取和当前调色板形成高对比度的颜色

    painter.setPen(this->palette().color(QPalette::ColorRole::BrightText));

    painter.setRenderHint(QPainter::Antialiasing);

    painter.setBrush(Qt::transparent);

    painter.setOpacity(0.15);

    painter.drawPath(linePath);

  

    //毛玻璃

    //qreal可以根据系统编译器自然的转换为float或double

    qreal opacity = getGlobalOpacity();

    painter.setRenderHint(QPainter::Antialiasing);

  

    //设置描边颜色和填充颜色

    painter.setPen(Qt::transparent);

    painter.setBrush(this->palette().base());

  

    //设置不透明度

    painter.setOpacity(opacity);

    painter.drawPath(linePath);

  

  

    //KDE模块，控制窗口模糊

    KWindowEffects::enableBlurBehind(this->winId(), true, QRegion(linePath.toFillPolygon().toPolygon()));

  
}
```

## 3.3 MyAdaptor类

- 继承自**QDBusAbstractAdaptor类**，用来实现**D-Bus**中接口的方法。这里方法主要控制窗体的显示与隐藏，以及添加窗体消失特效。

### myadaptor.h

```cpp
#ifndef MYADAPTOR_H

#define MYADAPTOR_H

  

#include <QObject>

#include <QLabel>

#include <QTimer>

#include <QDebug>

#include <QDBusAbstractAdaptor>

  

#include "widget.h"

  

#define SERVERNAMEPREFIX "org.ukui."

#define SERVERNAME "OSDService"

  

class MyAdaptor : public QDBusAbstractAdaptor

{

    Q_OBJECT

  

    //用##对宏做字符串拼接

    Q_CLASSINFO("D-Bus Interface",SERVERNAMEPREFIX ## "OSDInterface")

  

public:

    explicit MyAdaptor(QObject *parent = nullptr);

  

  

private:

    Widget* p_mIconWidget = nullptr;

    QTimer* p_mTimer = nullptr;

  

public Q_SLOTS:

    void showIcon(QString iconId);

  

    //控制代码

    void hideAutomatically();

};

  

#endif // MYADAPTOR_H
```

### myadaptor.cpp

```cpp
#include "myadaptor.h"

#include <KWindowSystem>

  

MyAdaptor::MyAdaptor(QObject *parent) : QDBusAbstractAdaptor(parent)

{

    //初始化界面时，不会调用绘画

    p_mIconWidget = new Widget(this);

  

    p_mTimer = new QTimer(this);

  

  

    connect(p_mTimer,&QTimer::timeout,this,&MyAdaptor::hideAutomatically);

  

    //测试

    //showIcon("ukui-capslock-on-symbolic");

}

  

  

void MyAdaptor::showIcon(QString iconId){

  

  

    if(!p_mIconWidget->isHidden()){

  

        p_mIconWidget->hide();

  

        p_mTimer->stop();

    }

  

    p_mIconWidget->setIconPixmap(iconId);

  

    //每次show()时都会重绘

    p_mIconWidget->show();

  

    //KDE模块设置窗口像通知信息一样渐渐消失

    //需要每次show()的时候都调用，不然有bug

    KWindowSystem::setType(p_mIconWidget->winId(), NET::Notification);

  

    p_mTimer->start(2500);

}

void MyAdaptor::hideAutomatically(){

  

    p_mIconWidget->hide();

  

    p_mTimer->stop();

}
```

## 3.4 MyServers类

- 用于管理实现接口方法的**MyAdaptor类**。

### myservers.h

```cpp
#ifndef MYSERVERS_H

#define MYSERVERS_H

  

#include <QObject>

#include "myadaptor.h"

  

class MyServers : public QObject

{

    Q_OBJECT

public:

    explicit MyServers(QObject *parent = nullptr);

    MyAdaptor* data();

private:

    MyAdaptor* p_mAdaptor = nullptr;

signals:

  

};

  

#endif // MYSERVERS_H
```

### myservers.cpp

```cpp
#include "myservers.h"

#include <QDebug>

  

MyServers::MyServers(QObject *parent) : QObject(parent)

{

    p_mAdaptor=new MyAdaptor(this);

    qDebug()<<tr("Adaptor初始化成功");

  

}

  

MyAdaptor* MyServers::data(){

    return p_mAdaptor;

}
```

## 3.5 ServerManager类

- 管理**MyServer类**，控制服务的注册与注销。

### servermanager.h

```cpp
#ifndef SERVERMANAGER_H

#define SERVERMANAGER_H

  

  

#include <QObject>

#include "myservers.h"

  

  

class ServerManager : public QObject

{

    Q_OBJECT

public:

    explicit ServerManager(QObject *parent = nullptr);

    ~ ServerManager();

  

  

  

private:

    MyServers *p_mServer=nullptr;

  

    //判断是否完全注册成功，来实现单例程序

    bool m_constructed = false;

  

public:

    bool getBool();

signals:

  

};

  

#endif // SERVERMANAGER_H
```
### servermanager.cpp

```cpp
#include "servermanager.h"

#include <QDebug>

#include <QDBusConnection>

  

  

ServerManager::ServerManager(QObject *parent) : QObject(parent)

{

    p_mServer = new MyServers(this);

  

  

    //不支持两个char*之间直接+

    if(!QDBusConnection::sessionBus().registerService(QString(SERVERNAMEPREFIX) + SERVERNAME)){

        qDebug()<<tr("服务注册失败");

        return;

    }

  

    if(!QDBusConnection::sessionBus().registerObject("/"+QString(SERVERNAME),p_mServer)){

        qDebug()<<tr("对象注册失败");

        return;

    }

  

    m_constructed = true;

  

    qDebug()<<tr("服务注册成功");

}

ServerManager::~ServerManager(){

  

  

  

    if(QDBusConnection::sessionBus().unregisterService(QString(SERVERNAMEPREFIX) + SERVERNAME)){

        QDBusConnection::sessionBus().unregisterObject(QString(SERVERNAME));

        qDebug()<<tr("服务注销成功");

    }

  

  

}

  

bool ServerManager::getBool(){

    return m_constructed;

}
```

## 3.6 main.cpp

- 需要完成翻译文件的导入以及单例程序的实现

```cpp
#include "servermanager.h"

  

#include <QString>

#include <QTranslator>

#include <QApplication>

  

  

int main(int argc, char *argv[])

{

    QApplication a(argc, argv);

  

  

    QTranslator translator;

  

        //导入qm翻译文件

        if (translator.load(":/translations/en.qm")) {

  

            a.installTranslator(&translator);

  

        }

  

    ServerManager serverManager;

  

    if(!serverManager.getBool()){

  

  

        qDebug()<<QObject::tr("已有程序正在运行");

  

        return 0;

    }

  
    return a.exec();

}
```
