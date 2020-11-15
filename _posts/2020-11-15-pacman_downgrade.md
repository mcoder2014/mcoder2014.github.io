---
layout:     post
title:      "软件包降级"
subtitle:   "manjaro arch pacman"
date:       2020-11-14
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - linux
---

Manjaro Linux（或简称 Manjaro）是基于 Arch Linux 的 Linux 发行版，使用 Xfce 、GNOME和 KDE Plasma 作为默认桌面环境，和 Arch 一样，采用滚动更新。其目标是为 PC 提供易于使用的自由的操作系统。

## 滚动升级的优点

滚动更新带来了一个好处，我们可以很快用上最新的软件包，比如：截止目前(2020-11-15)，在 ubuntu 的官方仓库中 opencv 版本仅仅是 3.20，而在 manjaro 中的 opencv 版本早就升级到了 4.50。如果你是一个需要 opencv 4 特性的开发者，那么在 ubuntu 上需要自行编译 4 以上的源码包，而在 manjaro 上可以直接下载使用。

```sh
# ubuntu 18.04 LTS
mcoder@Chaoqun-PC:~/workspace/my_doc/blog$ sudo apt search opencv-dev
Sorting... Done
Full Text Search... Done
libopencv-dev/bionic-security,bionic-updates,now 3.2.0+dfsg-4ubuntu0.1 amd64 [installed]
  development files for opencv

# manjaro 20
[chaoqun@manjaro local]$ sudo pacman -Ss opencv
extra/gst-plugin-opencv 1.18.1-1
    Multimedia graph framework - opencv plugin
extra/opencv 4.5.0-1 [已安装]
    Open Source Computer Vision Library
extra/opencv-samples 4.5.0-1 [已安装]
    Open Source Computer Vision Library (samples)
```

## 滚动升级的缺点

基于滚动升级的优点，我养成了个习惯，每隔那么几天，就同步一下软件库：

```sh
[chaoqun@manjaro local]$ sudo pacman -Ss opencv
extra/gst-plugin-opencv 1.18.1-1
    Multimedia graph framework - opencv plugin
extra/opencv 4.5.0-1 [已安装]
    Open Source Computer Vision Library
extra/opencv-samples 4.5.0-1 [已安装]
    Open Source Computer Vision Library (samples)
[chaoqun@manjaro local]$ sudo pacman -Syu
[sudo] chaoqun 的密码：
:: 正在同步软件包数据库...
 core 已经是最新版本
 extra 已经是最新版本
 community 已经是最新版本
 multilib 已经是最新版本
:: 正在进行全面系统更新...
正在解析依赖关系...
正在查找软件包冲突...

软件包 (1) eigen-3.3.8-3

全部安装大小：  6.54 MiB
净更新大小：  0.08 MiB

:: 进行安装吗？ [Y/n]
```

然后今天，我碰到了数个大坑：

1. Qt 5.15.1 3 导致我 debug 出现了很奇怪的现象；
2. eigen-3.3.8-3 做的一些修改致使我原本可以编译通过的代码遇到了bug，但我目前不想根据该版本修改自己的代码；

而在更早的时候，我曾因为更新了系统，导致进不去桌面的问题，我在 tty 模式里卸载了桌面，重新安装后才解决了这个问题，当时折腾我好久。

所以滚动升级的缺点显而易见了：**不稳定！最新的软件包可能带来不可控的 bug。**

## pacman 如何降级

先说关键：安装 **downgrade** 程序！在 Arch Linux 中，有一个名为 “downgrade” 的实用程序，可帮助你将安装的软件包降级为任何可用的旧版本。此实用程序将检查你的本地缓存和远程服务器（Arch Linux 仓库）以查找所需软件包的旧版本。你可以从该列表中选择任何一个旧的稳定的软件包并进行安装。

Manjaro的软件源中有该软件，直接通过 `pacman -S downgrade` 下载安装。直接通过 `sudo downgrade <软件名>`，可以检查可以回滚的软件版本号，直接输入选项可以回滚到指定的版本。

```sh
[chaoqun@manjaro local]$ sudo downgrade eigen

Downgrading from A.L.A. is disabled on the stable branch. To override this behavior, set DOWNGRADE_FROM_ALA to 1 .
See https://wiki.manjaro.org/index.php?title=Using_Downgrade  for more details.

可选的包：

-  1)  eigen    3.3.7  7  any  (本地)
+  2)  eigen    3.3.8  3  any  (本地)

输入数字以选择包：1
正在加载软件包...
警告：正在降级软件包 eigen (3.3.8-3 => 3.3.7-7)
正在解析依赖关系...
正在查找软件包冲突...

软件包 (1) eigen-3.3.7-7

全部安装大小：   6.47 MiB
净更新大小：  -0.08 MiB

:: 进行安装吗？ [Y/n] y
(1/1) 正在检查密钥环里的密钥                                                [###########################################] 100%
(1/1) 正在检查软件包完整性                                                  [###########################################] 100%
(1/1) 正在加载软件包文件                                                    [###########################################] 100%
(1/1) 正在检查文件冲突                                                      [###########################################] 100%
(1/1) 正在检查可用存储空间                                                  [###########################################] 100%
:: 正在处理软件包的变化...
(1/1) 正在降级 eigen                                                        [###########################################] 100%
:: 正在运行事务后钩子函数...
(1/1) Arming ConditionNeedsUpdate...
添加eigen到IgnorePkg? [y/N] n
```

发现 bug 是 eigen 引入后，我降级了 eigen，代码成功通过编译，等过段时间 eigen 出更新的版本后我再重新更新吧。

## Reference

1. [如何在 Arch Linux 中降级软件包](https://linux.cn/article-9730-1.html)
