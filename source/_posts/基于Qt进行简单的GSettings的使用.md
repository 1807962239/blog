---
title: 基于Qt进行简单的GSettings的使用
date: 2024-08-12 19:05:40
tags:
	- Qt
	- demo
	- CMake
	- GSettings
categories: demo
keywords:
	- Qt 
	- GSettings
	- QGSettings
	- CMake
	- C++
	- Linux
description: 练习QGSetting配合schema文件的使用，主要是get和set两个基础方法的使用。
cover:
---

# 1 概念

## 1.1 GSettings

- 在 **GNOME 桌面环境**中，`GSettings` 是一个用于存储和访问应用配置设置的系统。它提供了一种简单的方式来保存和恢复程序的状态，并且允许程序声明它们需要什么样的配置选项，以及这些选项应该如何被解释。

## 1.2 schema

- `schema` 在 `GSettings` 中指的是定义配置项结构的文件。schema 文件描述了每个配置项的数据类型、默认值、范围（如果适用）、以及是否是可写等属性。这些 schema 文件通常以 `.gschema.xml` 为扩展名，并且存储在特定的目录中。

## 1.3 schema id

- 命名格式：`schema id` 通常是反转的域名格式加上应用程序或组件的名称，这有助于避免不同应用程序之间 schema id 的冲突。例如，如果应用程序名为 `MyApp` 并且域名为 `example.com`，那么 schema id 可能是 `com.example.MyApp`。

- 作用：当在程序中想要读取或更改某个配置项时，需要使用schema id。

# 2 具体实现

## 2.1 schema文件

- 首先需要一个`.gschema.xml`文件也就是**schema文件**，一般将其创建在系统的`/usr/share/glib-2.0/schemas/`目录中，这样项目就能根据名字找到它。

- 这里直接找来一个现成的`com.example.app.gschema.xml`文件，使用时要先对其编译：

	`sudo glib-compile-schemas /usr/share/glib-2.0/schemas/`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE gschema SYSTEM "gschema.dtd">
<schemalist>
<schema  id="com.example.app">
  <!-- 定义一个布尔类型的设置 -->
  <key name="show-notifications" type="b">
    <summary>Enable or disable notifications</summary>
    <description>Whether the application should display notifications.</description>
    <default>true</default>
  </key>
</schema>
</schemalist>
```

## 2.2 CMakeLists.txt

- 导入**GSettings**的包

```
cmake_minimum_required(VERSION 3.5)

  

project(demoqgsettings LANGUAGES CXX)

  

set(CMAKE_INCLUDE_CURRENT_DIR ON)

  

  

set(CMAKE_AUTOUIC ON)

set(CMAKE_AUTOMOC ON)

set(CMAKE_AUTORCC ON)

  

set(CMAKE_CXX_STANDARD 11)

set(CMAKE_CXX_STANDARD_REQUIRED ON)

  

find_package(PkgConfig)

find_package(Qt5 COMPONENTS Core  REQUIRED)

  

add_executable(demoqgsettings

  main.cpp

  settingswatcher.h

  settingswatcher.cpp

)

  

set_property(SOURCE main.cpp PROPERTY AUTOMOC OFF)

  

pkg_check_modules(gsettings-qt REQUIRED IMPORTED_TARGET gsettings-qt)

  

  

target_link_libraries(demoqgsettings Qt5::Core PkgConfig::gsettings-qt)
```

## 2.3 settingswatcher.h

```cpp
#ifndef SETTINGSWATCHER_H

#define SETTINGSWATCHER_H

  

#include <QObject>

#include <QGSettings>

#include <QDebug>

  

class SettingsWatcher : public QObject

{

    Q_OBJECT

public:

    explicit SettingsWatcher(QObject *parent = nullptr);

    ~SettingsWatcher();

  

signals:

private slots:

    void onSettingChanged(const QString &key);

  

  

private:

    QGSettings *m_settings=nullptr;

  

};

  

#endif // SETTINGSWATCHER_H
```

## 2.4 settingswatcher.cpp

```cpp
#include "settingswatcher.h"

  

SettingsWatcher::SettingsWatcher(QObject *parent) : QObject(parent)

{

    // 创建 QGSettings 实例

    //第一个参数是schema id，第二个参数是schema文件的路径，/省略

    m_settings = new QGSettings("com.example.app","/");

  

    connect(m_settings,&QGSettings::changed,this,&SettingsWatcher::onSettingChanged);

  

    // 获取并打印当前值

    //GSettings的get返回一个QVariant

    QVariant value = m_settings->get("showNotifications");

  

    //检查QVariant的对象是否可以转换为bool类型

    //<bool>是一个内部改写的代码糖，原canConvert的函数是传入一个int类型的标志来代表转入类型

    if (value.canConvert<bool>()) {

        bool showNotifications = value.toBool();

        qDebug() << "Initial value of showNotifications:" << showNotifications;

    }

  

    // 更改值

    m_settings->set("showNotifications", false);

  

    m_settings->set("showNotifications", true);

  

  

}

  

SettingsWatcher::~SettingsWatcher()

{

    qDebug()<<"wo si l...";

}

void SettingsWatcher::onSettingChanged(const QString &key){

    qDebug() << "Setting changed:" << key;

}
```

## 2.5 main.cpp

```cpp
#include <QCoreApplication>

  

#include "settingswatcher.h"

  

  

int main(int argc, char *argv[])

{

    QCoreApplication a(argc, argv);

  

    SettingsWatcher myWatcher;

  

    return a.exec();

}
```

## 2.6 问题及解决

### 2.6.1 问题一

1、通过`find_package`又找不到**QGSettings**的包

2、通过`pkg-config --modversion Qt5GSettings`查找电脑中是否安装了这部分组件，结果是没有：

```
Package Qt5GSettings was not found in the pkg-config search path.
Perhaps you should add the directory containing `Qt5GSettings.pc'
to the PKG_CONFIG_PATH environment variable
No package 'Qt5GSettings' found
```

3、再通过`qmake -v`检查Qt版本，因为**Qt 5.9**以上版本才支持QGSettings，显示版本也达标：

```
QMake version 3.1
Using Qt version 5.12.12-kylin in /usr/lib/aarch64-linux-gnu
```

4、尝试进行补充安装，但没有这个安装包

```
sudo apt update
sudo apt install libqt5gsettings5-dev
```

5、改变方法，通过`PkgConfig` 模块来查找和链接 **Qt5GSettings**。

```
find_package(PkgConfig)

//第一个参数设置变量名
//第三个关键字表示导入一个目标
//第四个参数是这个目标的名字
pkg_check_modules(gsettings-qt REQUIRED IMPORTED_TARGET gsettings-qt)

target_link_libraries(demoqgsettings Qt5::Core PkgConfig::gsettings-qt)
```
### 2.6.2 问题二

问题：将**SettingsWatcher类**放在**main.cpp**中编译不通过，原因是**moc没能正确处理Q_OBJECT宏**。

具体原因：**SettingsWatcher类**中包含了**Q_OBJECT宏**，但由于这个类在**main.cpp**文件中，**main**文件一般不包含**moc文件**，所以无法对**Q_OBJECT宏**进行正常的处理。

解决办法：将**SettingsWatcher类**重新分开写在头文件和源文件里，moc自动处理**SettingsWatcher.h**中的**Q_OBJECT宏**。