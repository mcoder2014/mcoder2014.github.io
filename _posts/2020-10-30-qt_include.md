---
layout:     post
title:      "qmake language include()"
subtitle:   "关于 qt *.pro 引用其他文件的问题"
date:       2020-10-30
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - Qt
    - qmake language
---

## Qmake Language include()

使用 Qt 的开发者都多多少少使用过 QtCreator 这个轻量的 IDE，除了部分 windows 平台的开发者通过插件使用 VS 进行开发。
`*.pro` 是 qt 的工程管理文件，通过 `qmake ../xxx.pro` 可以在当前文件夹生成编译所需要的的 Makefile 等文件，是一个类似 CMake 的工程管理软件。

通常情况下，我们可以直接在当前项目的 `*.pro` 文件中引用一个第三方库，如当前所示的小工程 `mirror.pro` 的配置文件，摘自我的一些小练习[openmsh-smallcode](https://gitee.com/mcoder2014/openmsh-smallcode)。

```qmake

# 工程类型是生成可以执行程序
TEMPLATE = app

# 工程的一些配置信息
CONFIG += console c++14
CONFIG -= app_bundle
CONFIG -= qt

# 目标可执行程序的名称
TARGET = mirror

# 目标程序的存放位置
DESTDIR = ../bin/

# 工程中的 cpp 文件
SOURCES += \
        main.cpp \
        MeshAlgorithm.cpp \

# 工程中的头文件
HEADERS += \
        MeshAlgorithm.h \
        $$PWD/include/Mesh.h

        HEADERS +=\
    $$PWD/include/Mesh.h \

unix {
    INCLUDEPATH += /usr/include/eigen3

    LIBS+= \
        -L/usr/lib \
        -lOpenMeshCore -lOpenMeshTools
}
```

## 哪里需要改进

这样的 pro 文件一点儿问题都没有，可以正常编译。唯一的缺点就是，如果你面临和我一样的场景：

1. 围绕着某个第三方框架写很多小的测试程序；
2. 要写很多个可执行程序，做功能测试，不方便写在同一个工程下。

那么你肯定需要 Qt 的 [Subdirs Project](https://gitee.com/mcoder2014/openmsh-smallcode) 了，这个方式下可以比较容易的同时管理多个子工程。
不过，新的问题出现了：**这么多个*.pro文件都用到了重复的内容，会不会有些不雅观，不仅不符合计算机中抽象的思路，还面临如果第三方库配置需要修改，则一次要修改很多个*.pro文件的尴尬场景。**

## qmake 中一个可以用来解决的方案

qmake language 中有很多内置函数，其中有一个 `include()` 函数，可以将其他文件的内容引入当前 pro 文件，有些像 `#include <iostream>` 的意思。
> Includes the contents of the file specified by filename into the current project at the point where it is included. This function succeeds if filename is included; otherwise it fails. The included file is processed immediately.
> You can check whether the file was included by using this function as the condition for a scope. For example:

```qmake
include( shared.pri )
OPTIONS = standard custom
!include( options.pri ) {
    message( "No custom build options specified" )
OPTIONS -= custom
}
```

那么我产生了些疑问，被引入的 pri 文件和引入的 pro 文件的环境路径上有什么区别？

我做了简单的测试，发现 pri 中能访问到 `include(xx.pri)`语句以前定义的变量，存在例外

* `$$PWD`变量指的是 pri 文件所在的文件夹路径
* `$$OUT_PWD`变量则与 pro 文件一致，指向的 qmake 到处的编译工程的路径

## 改写 mirror 工程

我把 openmesh 相关的文件提取出来，写了个 `openmesh.pri`，我利用 `$$PWD`变量，让 `openmesh.pri` 也额外管理一个头文件。
也就是说， `*.pri` 除了存在目的是为了被其他工程引用，和 `*.pro` 文件没有什么不同

```qmake
# openmesh.pri
message("PWD of openmesh.pri $$PWD")

HEADERS +=\
    $$PWD/include/Mesh.h \

unix {
    INCLUDEPATH += /usr/include/eigen3 \
                $$PWD/include \

    LIBS+= \
        -L/usr/lib \
        -lOpenMeshCore -lOpenMeshTools \
}

win32 {

}
```

并重写 mirror.pro 文件中的内容，这次可以减少很多重复内容：

```qmake

# mirror.pro
TEMPLATE = app
CONFIG += console c++14
CONFIG -= app_bundle
CONFIG -= qt

TARGET = mirror
DESTDIR = ../bin/

SOURCES += \
        main.cpp \
        MeshAlgorithm.cpp \

HEADERS += \
        MeshAlgorithm.h

include(../openmesh.pri)
```

之后，我有写了其他的测试程序，也可以通过这个方式引用，例如 projection 投影的测试程序：

```qmake
# projection.pro

TEMPLATE = app
CONFIG += console c++11
CONFIG -= app_bundle
CONFIG -= qt

DESTDIR=../bin
TARGET=projection

SOURCES += \
        Projection.cpp \
        main.cpp

HEADERS += \
    Projection.h

include(../openmesh.pri)
```

在这种情况下，如果我更换了操作系统，把程序放在 windows 上编译，只要修改好 `openmesh.pri` 这一个文件的内容就好了，不需要修改那么多引用同样配置文件的工程的配置文件。
同理，我还可以对每个第三方库都写一个对应的 pri 文件，这样在实际使用中可以自由的组合。

```qmake
include(path/openmesh.pri)
include(path/boost.pri)
include(path/cgal.pri)
```

甚至，你可以把这些文件放置在一个系统路径下，这样在当前环境下开发的所有工程都可以引用到该配置文件，从而后续的其他工程都无须重新配置该第三方库，实现一劳永逸。

## 总结

借助 qmake 内置函数 `include()` ，我们可以很容易的实现第三方库环境配置的复用，可以在一个 subdirs project 中复用，甚至可以在整台机器的所有工程中复用。

## Reference

1. [amake Manual](https://doc.qt.io/qt-5/qmake-manual.html)
2. [openmsh-smallcode](https://gitee.com/mcoder2014/openmsh-smallcode)
