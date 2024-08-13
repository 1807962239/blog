---
title: 基于Qt进行简单D-Bus的使用
date: 2024-08-08 20:43:35
tags:
	- demo
	- Qt
	- CMake
	- D-Bus
categories: demo项目
keywords: 
	- Qt
	- CMake
	- D-Bus
	- Linux
description: 在Qt中分别用两种不同的方式注册两个D-Bus服务端，再实现一个客户端程序对服务端中的接口方法进行调用。
cover:
---

# 1 概念

## 1.1 简介

**D-Bus**是一种用于**进程间通信 (IPC) 的消息总线系统**，它使得应用程序可以在不同的进程之间交换数据和调用远程过程。**D-Bus**被广泛应用于**Linux桌面环境**中，如**GNOME**和**KDE**。

## 1.2 组件与功能

1. **Bus（总线）**：
   
    - 总线是进程间通信的基础。
    - 一个系统通常有一个“系统总线”供所有用户和服务使用，还有一个“会话总线”为每个用户的登录会话提供服务。

2. **Message（消息）**：
   
    - 所有通信都通过消息完成。
    - 消息类型包括方法调用、信号和错误。

3. **Object Path（对象路径）**：
   
    - 类似于文件系统的路径，用于标识特定的接口实例。

4. **Interface（接口）**：
   
    - 定义了一组方法、信号和属性。

5. **Method（方法）**：
   
    - 接口提供的功能。

6. **Signal（信号）**：
   
    - 事件通知。

7. **Property（属性）**：
   
    - 对象的状态信息。

# 2 基本步骤

1. **安装必要的软件包**：
   
    - 在大多数 Linux 发行版中，D-Bus 库通常已经预先安装，可以通过查看版本确定：
	    `dbus-binding-tool --version`

	- 如果没安装好则安装：`sudo apt-get install libdbus-1-dev`

2. **编写客户端或服务端代码**：
   
    - 使用 C, Python, Java 或其他支持的语言来编写代码。
    - 使用 `g_dbus_code_generator` 或类似的工具生成绑定代码。

3. **注册服务**：
   
    - 服务需要注册到总线上才能接收请求。
    - 可以通过 `dbus-daemon` 命令手动启动总线守护进程，或者在系统启动时自动启动。

4. **发送和接收消息**：
   
    - 客户端发送方法调用消息。
    - 服务端处理这些消息并返回结果或发送信号。

# 3 Qt关于D-Bus的组件

1. **QDBusConnection**：
   
    - 代表 D-Bus 连接。
    - 用于注册服务、获取连接等操作。

2. **QDBusInterface**：
   
    - 用于访问远程对象的接口。
    - 支持同步和异步调用。

3. **QDBusMessage**：
   
    - 用于发送低级别的消息。
    - 通常用于不需要返回值的情况。

4. **QDBusObjectPath**：
   
    - 用于标识对象路径。

5. **QDBusArgument**：
   
    - 用于序列化和反序列化 D-Bus 参数。

6. **QDBusServiceWatcher**：
   
    - 监控服务是否出现在总线上。

7. **QDBusAdapter**：
   
    - 适配器模式，用于简化 D-Bus 的使用。

# 4 Qt具体实现

## 4.1 服务端

### 4.1.1 CMakeLists

- 其中的**MOC（Meta-Object Compiler）处理**，在使用Qt库时是必须的，通常**Qt6 CMake**会自动完成构建，无需这样手动显示处理。

```
cmake_minimum_required(VERSION 3.5)

  

project(demoserver LANGUAGES CXX)

  

set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

  


#添加Qt中的DBus库

  

find_package(Qt5 COMPONENTS Core Widgets DBus REQUIRED)

  

 
  

add_executable(demoserver

  main.cpp
  
  dbusmanager.cpp

  dbusmanager.h

  myserver.cpp

  myserver.h

)

  

  


# MOC（Meta-Object Compiler）处理

#将包含QObject宏的头文件放进MOC_SRCS变量中

#qt5_wrap_cpp生成MOC处理后的源文件

#然后将其添加到项目中

set(MOC_SRCS ${CMAKE_CURRENT_SOURCE_DIR}/myserver.h)

qt5_wrap_cpp(MOCS ${MOC_SRCS})

target_sources(demoserver PRIVATE ${MOCS})

  

  

#链接DBus库

  

target_link_libraries(demoserver Qt5::Core Qt5::DBus)
```

### 4.1.2 myserver.h

- 只包含接口方法

```cpp
#ifndef MYSERVER_H

#define MYSERVER_H

  

#include <QObject>


#include <QDBusConnection>

//用于创建DBus适配器

#include <QDBusAbstractAdaptor>

  

class MyServer : public QObject

{

    Q_OBJECT

  

    //Q_CLASSINFO为类提供额外的元信息

    //告诉Qt该类将作为D-Bus接口暴露给其他应用程序
    //也就是设置接口名

    Q_CLASSINFO("D-Bus Interface", "com.example.MyInterface")

  

public:

    explicit MyServer(QObject *parent = nullptr);

 Q_SIGNALS:

    void shotSignals(QString);

signals:


//对于 D-Bus，`slots` 可以简单地理解为普通的成员函数
public slots:

    QString echo(const QString &message);

  

};

  

#endif // MYSERVER_H
```

### 4.1.3 myserver.cpp

```cpp
#include "myserver.h"

#include <QDebug>

  

MyServer::MyServer(QObject *parent) : QObject(parent)

{



}

  

QString MyServer::echo(const QString &message)

{

    Q_EMIT shotSignals("plz kill me..");

    return "Echo: " + message;

}
```

### 4.1.4 dbusmanager.h

- 用来管理服务的创建和关闭

```cpp
#ifndef DBUSMANAGER_H

#define DBUSMANAGER_H

  

#include <QObject>

#include "myserver.h"

class DbusManager : public QObject

{

    Q_OBJECT

public:

    DbusManager();

  

private Q_SLOTS:


	//关闭服务
    void doCloseDBus(QString msg);

private:

    MyServer *m_pServer = nullptr;

};

  

#endif // DBUSMANAGER_H
```

### 4.1.5 dbusmanager.cpp

```cpp
#include "dbusmanager.h"

#include <QDebug>

DbusManager::DbusManager(): m_pServer(nullptr)

{
	//在管理类的构造函数中初始化服务端

	

    m_pServer = new MyServer(this);

  

  

    //注册服务到D-Bus

    //QDBusConnection::sessionBus()获取会话总线的连接

    //registerService尝试在D-Bus注册一个服务名称，名称要符合D-Bus命名规范

    if (!QDBusConnection::sessionBus().registerService("com.example.MyService")) {

        qWarning() << "Cannot register the service";

        return;

    }

  

  

  

    // 注册对象到D-Bus

    //registerObject的两个参数分别是对象路径和要注册的对象指针

    //这里的对象指针通常是包含Q_CLASSINFO("D-Bus Interface", "com.example.MyService")的类

    //或继承自QDBusAbstractAdaptor的类

    if (!QDBusConnection::sessionBus().registerObject("/MyServer", m_pServer,QDBusConnection::ExportAllSlots | QDBusConnection::ExportAllSignals)) {

        qWarning() << "Cannot register the object";

        return;

    }

  


  

    qInfo() << "Service registered and ready to accept connections";

  



    //服务本身发出杀死自己的信号，管理器接收并执行

    connect(m_pServer, &MyServer::shotSignals, this, &DbusManager::doCloseDBus);

}

  

void DbusManager::doCloseDBus(QString msg)

{

    if (QDBusConnection::sessionBus().unregisterService("com.example.MyService")) {

        QDBusConnection::sessionBus().unregisterObject("/MyServer");

    }

  


    delete m_pServer;

}
```

### 4.1.6 main.cpp

```cpp
#include <QCoreApplication>  


#include "myserver.h"  


int main(int argc, char *argv[])  


{  
    QCoreApplication a(argc, argv);  

	//创建管理类对象
    DbusManager dbusManger;
      
    return a.exec();  
}
```

## 4.2 客户端

- 在Qt中，编写一个D-Bus客户端以与先前注册的D-Bus服务（如服务器）交互，通常涉及以下几个步骤：

	1. 连接到D-Bus会话总线。
	2. 使用服务名称和对象路径来查找接口。
	3. 调用接口上的方法或访问其属性。

### 4.2.1 CMakeLists.txt

```cpp
cmake_minimum_required(VERSION 3.5)

  

project(democlient LANGUAGES CXX)

  

set(CMAKE_INCLUDE_CURRENT_DIR ON)

  

set(CMAKE_AUTOUIC ON)

set(CMAKE_AUTOMOC ON)

set(CMAKE_AUTORCC ON)

  

set(CMAKE_CXX_STANDARD 11)

set(CMAKE_CXX_STANDARD_REQUIRED ON)

  

find_package(Qt5 COMPONENTS Widgets Core DBus REQUIRED)

  

add_executable(democlient

  main.cpp

)

target_link_libraries(democlient Qt5::Core Qt5::DBus)
```

### 4.2.2 main.cpp

```cpp
#include <QCoreApplication>

#include <QDBusConnection>

#include <QDBusInterface>

#include <QDebug>

//处理远程调用返回的数据

#include <QDBusReply>

  

int main(int argc, char *argv[])

{

    QCoreApplication a(argc, argv);

  

    // 创建 D-Bus 接口

    //参数分别为注册的服务器名称，服务的对象路径，想要使用的接口的名称，总线类型

    QDBusInterface interface("com.example.MyService", "/MyServer", "com.example.MyInterface",

                             QDBusConnection::sessionBus());


    //检测接口是否有效

    if (!interface.isValid()) {

  

        qCritical() << "Cannot create a valid D-Bus interface";

  

        return 1;

    }

  

  

    // 调用 echo 方法，call函数的两个参数分别为调用的方法名和参数

    //QDBusReply作为一个模板接受服务端运行方法返回的信息

    QDBusReply<QString> reply = interface.call("echo", "Hello, World!");

  

  

    if (reply.isValid()) {

  

        qDebug() << "Received echo:" << reply.value();

  

    } else {

  

        qCritical() << "Error calling method:" << reply.error().message();

  

    }

  

    return a.exec();

}
```

## 4.3 遇到问题

### 4.3.1 问题一

1、一开始服务端运行成功，后来一直运行失败，怀疑是之前的服务没有关闭。

2、调用`qdubs com.example.MyService`命令去查看，但报错`qdbus: could not find a Qt installation of ''`

3、**dbus**需要访问**Qt**的插件，于是设置`QT_PLUGIN_PATH`：

```
echo 'export QT_PLUGIN_PATH=/usr/lib/aarch64-linux-gnu/qt5/plugins' >> ~/.bashrc
source ~/.bashrc
```

4、其中**Qt路径**通过`qmake -query`查询,可以看到

```
QT_INSTALL_PLUGINS:/usr/lib/aarch64-linux-gnu/qt5/plugins
```

5、依旧报错，且调用`qdbus -v`也会报同样错误。于是去查询到的插件目录里寻找，没有找到`libqdbus.so`

6、安装**QtDbus开发库**

```
sudo apt-get install libqt5dbus5-dev
```

7、报错没有这个软件包，调用其他命令安装

```
sudo apt install qtbase5-dev
```

8、安装完发现插件目录里依然没有`libqdbus.so`

9、最后放弃使用`qdbus`，改为使用`d-feet`，先安装

```
sudo apt-get install d-feet
```

10、打开后发现确实没关闭之前的服务

### 4.3.2 问题二

1、 由上个问题中发现服务中没有预想的接口和方法

2、由于不是将特定的`QDBusAbstractAdaptor`的方法导出，而是将类中所有方法导出，所以做以下修改，问题解决。

原：（这时默认参数为
```
bool registerObject(const QString &path, QObject *object,RegisterOptions options = ExportAdaptors);
```
）

```cpp
if (!QDBusConnection: :sessionBus().registerObject("/", this)) {

        qWarning() << "Cannot register the object";

        return;

}
```

现：

```cpp
if (!QDBusConnection: :sessionBus().registerObject("/", this，
	 QDBusConnection: :ExportAllSlots |
	QDBusConnection: :ExportAllSignals)) {

        qWarning() << "Cannot register the object";

        return;

}
```

# 5 依靠QDBusAbstractAdaptor的改良实现

- 改良原因：前一种服务端再被调用时直接将所有的槽函数暴露，这里通过添加一个继承自`QDBusAbstractApator`类的`MyServerApator`来更细化的管理接口和方法

- 功能分配：`MyServer` 类持有 `MyServerAdaptor` 的实例，并在构造函数中创建适配器。`MyServerAdaptor` 负责实现 `echo` 方法并通过 D-Bus 发送信号。`DbusManager` 类则负责注册服务和对象，并监听信号以便在接收到信号时关闭服务。

- 细节变化：

	注册对象时不再需要设定参数
	`QDBusConnection: :ExportAllSlots |QDBusConnection: :ExportAllSignals`

## 5.1 myserver.h

```cpp
#ifndef MYSERVER_H

#define MYSERVER_H

  

#include <QObject>

#include "myserveradaptor.h"

  

class MyServer : public QObject

{

    Q_OBJECT


public:

    explicit MyServer(QObject *parent = nullptr);

  

    //通过一个成员函数返回m_adaptor

    MyServerAdaptor* data();

private:

  

    MyServerAdaptor* m_adaptor=nullptr;

};

  

#endif // MYSERVER_H
```

## 5.2 myserver.cpp

```cpp
#include "myserver.h"

  

MyServer::MyServer(QObject *parent) : QObject(parent),m_adaptor(new MyServerAdaptor(this))

{

  

}

MyServerAdaptor* MyServer::data(){

    return m_adaptor;

}
```

## 5.3 myserveradaptor.h

```cpp
#ifndef MYSERVERAPATOR_H

#define MYSERVERAPATOR_H

  

#include <QObject>

#include <QDBusAbstractAdaptor>

  

class MyServerAdaptor : public QDBusAbstractAdaptor

{

    Q_OBJECT

    Q_CLASSINFO("D-Bus Interface","com.example.MyInterface")

public:

    explicit MyServerAdaptor(QObject *parent = nullptr);

  

  

signals:

    void closeService(QString message);

  

public Q_SLOTS:

    QString echo(QString message);

};

  

#endif // MYSERVERAPATOR_H
```

## 5.4 myserveradaptor.cpp

```cpp
#include "myserveradaptor.h"

  

//这里记得初始化基类QDBusAbstractAdaptor的值

MyServerAdaptor::MyServerAdaptor(QObject *parent) : QDBusAbstractAdaptor(parent)

{

  

}

QString MyServerAdaptor::echo(QString message){

    Q_EMIT closeService("please kill me...");

    return "Echo:"+message;

}
```

## 5.5 dbusmanager.h

```cpp
#ifndef DBUSMANAGER_H

#define DBUSMANAGER_H

  

#include <QObject>

#include "myserver.h"

  

class DbusManager : public QObject

{

    Q_OBJECT

public:

    explicit DbusManager(QObject *parent = nullptr);

  

private Q_SLOTS:

    void doCloseDBus();

private:

    MyServer *m_pServer=nullptr;

  

};

  

#endif // DBUSMANAGER_H
```

## 5.6 dbusmanager.cpp

```cpp
#include "dbusmanager.h"

#include <QDBusConnection>

#include <QDebug>

  

DbusManager::DbusManager(QObject *parent) : QObject(parent),m_pServer(new MyServer(this))

{

    if(!QDBusConnection::sessionBus().registerService("com.example.MyService")){

        qWarning()<<"Cannot register the service";

        return;

    }

  

    if(!QDBusConnection::sessionBus().registerObject("/MyServer",m_pServer)){

        qWarning() << "Cannot register the object";

        return;

    }

    qInfo() << "Service registered and ready to accept connections";

  

  

    connect(m_pServer->data(), &MyServerAdaptor::closeService, this, &DbusManager::doCloseDBus);

}

void DbusManager::doCloseDBus(){

    if(QDBusConnection::sessionBus().unregisterService("com.example.MyService")){

        QDBusConnection::sessionBus().unregisterObject("/MyServer");

    }

    delete m_pServer;

}
```

## 5.7 main.cpp

```cpp
#include <QCoreApplication>

#include "dbusmanager.h"

  

int main(int argc, char *argv[])

{

    QCoreApplication a(argc, argv);

  

    DbusManager myDbusManager;

  

    return a.exec();

}
```

## 5.8 补充

- 因为觉得**MyServer**比较多余，尝试将其功能合并到**DBusManager**，但会找不到接口及方法。在注册对象时，第三个参数设置为`QDBusConnection: :ExportAllSlots |QDBusConnection: :ExportAllSignals`才可以，不清楚具体原因。因此考虑可能需要一个**MyServer**作为中间类去管理**MyServerAdaptor**。