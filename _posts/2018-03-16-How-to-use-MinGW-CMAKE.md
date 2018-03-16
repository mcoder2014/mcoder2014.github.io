---
layout:     post
title:      "然并卵系列"
subtitle:   "windows 下使用CMAKE-gui来编译第三方库的源代码"
date:       2018-03-16
author:     Mcoder
header-img: img/JCQ_0117_00001.jpg
catalog: true
tags:
  - 环境配置
  - Linux
---

很多 linux 和 windows 平台通用的第三方库喜欢使用 CMAKE 来管理整个工程文件，当我们需要编译安装该库时，在 linux 上往往非常简单。

> 1. ./configure
2. make
3. make install

而在 windows 下配置该库难度就比较麻烦，一般使用 cmake-gui 进行配置，生成对应开发环境的工程（比如VS2010、VS2015、MinGW 之类的），这里描述一下使用 CMAKE-gui 编译配置第三方库的通常的过程。

# MinGW
以 MinGW 作为编译器举例。

## 环境
1. CMake 3.8.2
2. MinGW 5.30 32bit

CMake 和 MinGW 都要确保已经加入了系统环境变量。<br>
其中 MinGW 的环境变量是`MinGW -> 安装路径`,`PATH -> %MinGW%\bin`。

## opencv 2.4.11
opence 2.4.11 是 opencv库的一个早前版本，官方已经预编译好了几个vs版本的安装包，而我想要使用 MinGW环境进行编程，我就需要自己编译这个 opencv 库。

## 过程
1. 我们将下载的库文件解压，然后在外边新建一个文件夹叫 `build` 之类的。
![](/post_img/201803/cmake01.jpg)

2. 打开 cmake-gui，设置两个路径。Where is the source code 放置刚才的源代码文件夹路径。Where to build the binaries 放置想要编译好的文件放置的路径。
![](/post_img/201803/cmake02.jpg)

3. 点击 Configure 按钮，选择 MinGW Makefiles 和 Specify native compilers。然后在手动选择C和C++的编辑工具。一般用 gcc.exe 编译c语言，用 g++.exe编译C++语言(位于 MinGW 安装路径的bin文件夹中)。
![](/post_img/201803/cmake03.jpg)
![](/post_img/201803/cmake04.jpg)

4. 之后会自行配置一下工程，等到进度条完成了后，上边有很多红色的可以设置的选项，你根据需求进行勾选，然后再次点击 Configure 按钮。
![](/post_img/201803/cmake05.jpg)
![](/post_img/201803/cmake06.jpg)
5. 点击 Generate 生成工程文件。
6. 使用 CMD 窗口，通过 `cd `命令切换到 build 文件夹，使用`mingw32-make`命令开始编译工程。
![](/post_img/201803/cmake07.jpg)
7. 将编译好的库的动态文件添加到系统路径，然后就可以使用了。

# VS
与MinGW的操作基本相同，你可以选择合适自己版本的VS版本，然后 Configure， 再 Generate出工程。

比较容易的是，这个工程可以直接用 vs 打开，然后选择ALL_BUILD项目进行编译。
![](/post_img/201803/cmake08.jpg)

# Reference
1. [windows下使用cmake+mingw编译opencv2.4.13.3版](http://blog.csdn.net/wkr2005/article/details/78915272)
2. [cmake-gui使用教程](http://blog.csdn.net/qq_35488967/article/details/56480053)
