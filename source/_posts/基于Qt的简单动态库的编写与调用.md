---
title: 基于Qt的简单动态库的编写与调用
date: 2024-08-06 20:49:43
tags: 
	- demo
	- Qt
	- CMake
	- QMake
	- 库
categories: demo项目
keywords: Qt编写库,CMake构建,QMake构建
description: 在Qt creator中分别用CMake和QMake构建并编写一个库，在分别用CMake和QMake构建并编写一个可执行程序对自己编写的库进行调用。
cover:
---

# 1 编写库

## 1.1 库的入门

### 1.1.1 创建一个库

1、**在Qt creator**中自动创建，选择**Library**

![](https://1807962239hml-1324287819.cos.ap-beijing.myqcloud.com/202408060848977.png)

2、选择安装位置及项目名称

3、因为正在练习**CMake**，所以选择**CMake**为构建方式

![](https://1807962239hml-1324287819.cos.ap-beijing.myqcloud.com/202408060852619.png)

4、继续选择是**动态库、静态库还是插件**；选择库中第一个类的名称，以及选择需要包含的Qt组件

- 这里创建的**动态库**

![](https://1807962239hml-1324287819.cos.ap-beijing.myqcloud.com/202408060856903.png)

5、后续Kits等基本选择默认设置就可以

### 1.1.2 库的目录结构

以下为一个用CMake构建的简单库的主要目录结构

![](https://1807962239hml-1324287819.cos.ap-beijing.myqcloud.com/202408060905999.png)

1、**CMakeLists.txt**是构建项目的配置文件

2、**MyLibrary_global.h**是全局头文件，用来定义库的一些**基本配置、宏、类型定义等**

3、剩下是库的**头文件和源文件**

## 1.2 具体代码及解析

### 1.2.1 CMakeLists.txt

```C
#设置需要的cmake的最小版本

cmake_minimum_required(VERSION 3.5)

  
#设置项目名称

project(MyLibrary LANGUAGES CXX)

  
#设置一些基础变量

set(CMAKE_INCLUDE_CURRENT_DIR ON)

set(CMAKE_AUTOUIC ON)

set(CMAKE_AUTOMOC ON)

set(CMAKE_AUTORCC ON)

set(CMAKE_CXX_STANDARD 11)

set(CMAKE_CXX_STANDARD_REQUIRED ON)



#找Qt5中的Widgets组件
#COMPONENTS关键字代表找特定的组件
#REQUIRED关键字代表组件是必须的，没有找到会直接构建失败

find_package(Qt5 COMPONENTS Widgets REQUIRED)


  
#添加库中所包含的文件

add_library(MyLibrary SHARED

  MyLibrary_global.h

  mylibrary.cpp

  mylibrary.h

)

  


#设置安装前缀

set(CMAKE_INSTALL_PREFIX "/home/hml/文档/test")

  

#file（GLOB 变量名 某路径下的某格式） 这里的路径默认为当前项目目录
#将项目目录中的头文件存到HEADER_FILES变量中

file(GLOB HEADER_FILES "*.h")



#将当前库MyLibrary与其他库链接 PRIVATE关键字代表链接信息只对当前目标有效，不会传递

target_link_libraries(MyLibrary PRIVATE Qt5::Widgets)



#定义一个预处理宏MYLIBRARY_LIBRARY只对目标MyLibrary有效

target_compile_definitions(MyLibrary PRIVATE MYLIBRARY_LIBRARY)

  


#设置库的安装位置
#TARGETS指定要安装的目标 LIBRARY代表要安装的是库文件 DESTINATION则是要安装的目标位置
#这里的路径会把前面设置的安装前缀加在一起

install(TARGETS MyLibrary LIBRARY DESTINATION mylib)



#设置头文件的安装位置
#FILES直接指定要安装的文件列表 ${HEADER_FILES}是前面存放库中头文件的变量

install(FILES ${HEADER_FILES} DESTINATION mylib/include)
```

### 1.2.2 MyLibrary_global.h

- 主要用于处理共享库中的符号可见性

- 定义了一个宏 `MYLIBRARY_EXPORT`，用来决定哪些符号应该被导出或导入到库中

```C++
//MYLIBRARY_GLOBAL_H作为一个变量来避免此头文件被重复包含

#ifndef MYLIBRARY_GLOBAL_H

#define MYLIBRARY_GLOBAL_H



//包含Qt的全局定义头文件
//这个头文件是必要的，因为它包含了宏Q_DECL_EXPORT和Q_DECL_IMPORT
//这两个宏用于控制 Qt 库中的符号可见性。

#include <QtCore/qglobal.h>

  

//根据MYLIBRARY_LIBRARY来定义符号导出还是导入
//构件库时可导出 使用库时可导入

#if defined(MYLIBRARY_LIBRARY)

#  define MYLIBRARY_EXPORT Q_DECL_EXPORT

#else

#  define MYLIBRARY_EXPORT Q_DECL_IMPORT

#endif

  

#endif // MYLIBRARY_GLOBAL_H
```

### 1.2.3 mylibrary.h

```C++
#ifndef MYLIBRARY_H

#define MYLIBRARY_H

  

#include "MyLibrary_global.h"

  

//前面定义MYLIBRARY_EXPORT来控制这个类内容的导入和导出

class MYLIBRARY_EXPORT MyLibrary

{

public:

    MyLibrary();

    int getSum(int a,int b);

};

  

#endif // MYLIBRARY_H
```

### 1.2.4 mylibrary.cpp

```cpp
#include "mylibrary.h"

  

MyLibrary::MyLibrary()

{

}

int MyLibrary::getSum(int a,int b){

    return a+b;

}
```

## 1.3 CMake编译步骤

1、cd到项目目录

2、添加文件夹**build**：`mkdir build`

3、cd到build

4、`cmake ..`

5、`make`

6、（需要安装库和头文件时：`sudo make install`）

# 2 编写CMake构建的可执行文件

## 2.1 CMakeList.txt

```C
cmake_minimum_required(VERSION 3.5)

  

project(MyApp LANGUAGES CXX)

  

set(CMAKE_INCLUDE_CURRENT_DIR ON)

  

set(CMAKE_AUTOUIC ON)

set(CMAKE_AUTOMOC ON)

set(CMAKE_AUTORCC ON)

  

set(CMAKE_CXX_STANDARD 11)

set(CMAKE_CXX_STANDARD_REQUIRED ON)

  

find_package(Qt5Core)


#以绝对路径在库的安装位置找到自己编写的库
#LIBRARY_NAME作为变量存储库的名字 PATHS关键字指定绝对路径

find_library(LIBRARY_NAME MyLibrary PATHS /home/hml/文档/test/mylib)

  

#添加项目参与编译的文件

add_executable(MyApp

  main.cpp

)



#链接Qt5的Core库和自己编写的库

target_link_libraries(MyApp Qt5::Core ${LIBRARY_NAME})




#以绝对路径将自己编写的库中的头文件添加进项目中

target_include_directories(MyApp PRIVATE /home/hml/文档/test/mylib/include)
```

## 2.2  main.cpp

```cpp
#include <QCoreApplication>

#include "mylibrary.h"

#include "MyLibrary_global.h"

#include <QDebug>

  

int main(int argc, char *argv[])

{

    QCoreApplication a(argc, argv);

    int x=1;

    int y=2;


	//先创建库的类的实例化对象，再调用其中的成员函数

    MyLibrary* lib=new MyLibrary;

    int z=lib->getSum(x,y);

    qDebug()<<z;

    delete lib;

  

    return a.exec();

}
```

# 3 补充：QMake构建库的方法

## 3.1 qmakelibrary.pro

```
#减去qt的一些默认配置，表示其不是一个qt的默认程序

CONFIG -= qt

  

  

  

#指定项目模板为库

TEMPLATE = lib

  

  

#添加预处理宏来记录是构建库还是使用库

DEFINES += QMAKELIBRARY_LIBRARY

  


#添加c++11的标准

CONFIG += c++11

  

  

  

#添加预处理宏来警告已废弃了qt API

DEFINES += QT_DEPRECATED_WARNINGS

  

  

#添加源文件

SOURCES += \

    qmakelibrary.cpp

  

#添加头文件

HEADERS += \

    qmakelibrary_global.h \

    qmakelibrary.h

  

###############以下为安装设置################



#针对unix系统的特定配置，设定库的目标安装路径

unix {

    target.path = /home/hml/文档/test/mylib1

}

  

  
#如果库的目标路径不为空，则将库安装到目标路径

!isEmpty(target.path): INSTALLS += target

  

  

#设置安装的头文件和库的目标路径

#LIB_INSTALL_DIR = /home/hml/文档/test/mylib1

HEADERS_INSTALL_DIR = /home/hml/文档/test/mylib1/include




#如果路径存在，则安装头文件到目标位置

!isEmpty(HEADERS_INSTALL_DIR): INSTALLS += headers

    headers.path = $$HEADERS_INSTALL_DIR

    headers.files = $$HEADERS
```

>**QMake**安装时不会自动创建目录，需要`mkdir -p`手动创建

## 3.2 qmakelibrary_global.h

```
#ifndef QMAKELIBRARY_GLOBAL_H

#define QMAKELIBRARY_GLOBAL_H



//如果Windows和不同系统定义不同的Q_DECL_EXPORT和Q_DECL_IMPORT

#if defined(_MSC_VER) || defined(WIN64) || defined(_WIN64) || defined(__WIN64__) || defined(WIN32) || defined(_WIN32) || defined(__WIN32__) || defined(__NT__)

#  define Q_DECL_EXPORT __declspec(dllexport)

#  define Q_DECL_IMPORT __declspec(dllimport)

#else

#  define Q_DECL_EXPORT     __attribute__((visibility("default")))

#  define Q_DECL_IMPORT     __attribute__((visibility("default")))

#endif

  
//下面和CMake一样


#if defined(QMAKELIBRARY_LIBRARY)

#  define QMAKELIBRARY_EXPORT Q_DECL_EXPORT

#else

#  define QMAKELIBRARY_EXPORT Q_DECL_IMPORT

#endif

  

#endif // QMAKELIBRARY_GLOBAL_H
```

## 3.3 QMake编译步骤

1、cd到项目目录

2、`qmake .pro文件名`

3、`make`

4、（需要安装库和头文件时：`sudo make install`）

# 4 编写QMake构建的可执行文件

## 4.1 qmakeapp.pro

```
QT -= gui

  

CONFIG += c++11 console

CONFIG -= app_bundle

  

  

DEFINES += QT_DEPRECATED_WARNINGS

  

  

  

  

SOURCES += \

        main.cpp

  

  

  

#导入库和头文件

#-L表示文件夹，-l表示文件夹下的具体的库的名字

#$$PWD是当前项目目录

  

INCLUDEPATH+=/home/hml/文档/test/mylib1/include

LIBS+=-L/home/hml/文档/test/mylib1 -lqmakelibrary

  

  

  

  

  

  

#根据不同系统设置安装路径

  

qnx: target.path = /tmp/$${TARGET}/bin

else: unix:!android: target.path = /opt/$${TARGET}/bin

!isEmpty(target.path): INSTALLS += target
```

## 4.2 main.cpp

```cpp
#include <QCoreApplication>

  

#include "qmakelibrary.h"

#include "qmakelibrary_global.h"

#include <QDebug>

  

int main(int argc, char *argv[])

{

    QCoreApplication a(argc, argv);

  

    int x=1;

  

    int y=2;

  

  

    //先创建库的类的实例化对象，再调用其中的成员函数

  

    QMakeLibrary* lib=new QMakeLibrary;

  

    int z=lib->getSum(x,y);

  

    qDebug()<<z;

  

    delete lib;

  

    return a.exec();

}
```

# 5 补充：CMake将库和可执行文件绑定为一个工程

1、创建一个文件夹，并将所有子项目放进来

2、在文件夹内添加相应的**CMakeLists.txt**并编写

```
cmake_minimum_required(VERSION 3.5)

  

  

  

project(libdemo_cmake LANGUAGES CXX)

  

add_subdirectory(MyApp)

add_subdirectory(MyLibrary)
```
