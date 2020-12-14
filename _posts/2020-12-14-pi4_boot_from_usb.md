---
layout:     post
title:      "树莓派4 Usb 引导"
subtitle:   "SSD 做为系统盘"
date:       2020-12-14
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - linux
    - 树莓派
---

哈喽，大家好，我在双十二期间购买了自己心心念念的树莓派4代。

老早就有一个树莓派2B，性能较弱，而且只有 100 Mbps 的有线网口和 四个 USB 2.0 的接口，存储设备主要用的 TF 卡的 IO 性能也十分的弱鸡。

而树莓派4 就改进了很多，在我看来**树莓派4 是全方位的性能提升，不仅更换了主频更高的四核 A72 核心的CPU（相比手机还是拉胯），而且网卡换成了 1000 Mbps，配了两个 USB 3.0，配了蓝牙和WIFI。**简直翻天覆地的变化，4G 内存版本日常售价 380 元左右，到了双十二，有淘宝店家给到了344.66元的优惠入手价，果断买了一块。

![树莓派购买截图](/post_img/202001/order_pi4.jpg)

通过研究，发现树莓派新换了个玩意儿，叫做啥 `SPI-attached EEPROM 4MBits/512KB`，似乎就是一个板载内存，可以把 bootloader 放到板载内存里，相当于树莓派的 BIOS。而这带来了一个非常**先进的**特性：**树莓派可以直接从USB引导启动！我们可以告别SD卡了！**

## 引导树莓派 USB 启动需要的

* **树莓派4**：必须，这才是主角；
* **USB 存储**：可以是 U盘、移动硬盘、移动SSD等，我选择自己组了个移动SSD；
* **确保树莓派4 的bootloader版本足够**：似乎支持 usb 启动的 bootloader 在20年年中才发布 release，为了确保版本正确，最好先用 SD 启动一下检查一下；
* **mini HDMI 转接 HDMI的转接头**：ubuntu 没有自带 openssh-server，必须显示器操作一下，下载个 openssh-server，后面就用不到了；
* **系统镜像**：我选择了 64 位的 ubuntu Desktop 镜像，后来才注意到是 20.10，不是 20.04 LTS，亏了；

## 主要步骤

### 检查树莓派的 bootloader版本

首先我们下载一下树莓派的操作系统，我一开始用的树莓派官方系统，后来发现运行在了32位上，没有运行在 64 位。即便将内核切换为64位的内核，操作系统及大多数软件也是编译在32位上的，感觉不是很喜欢。所以我下载了 [ubuntu 的操作系统](https://ubuntu.com/download/raspberry-pi)，选择对应的版本即可下载。

之后，使用 树莓派的镜像写入软件，先刻录到SD卡上，正常启动树莓派。

![imager](/post_img/202001/rasp_imager.png)

然后，运行如下命令，检查 bootloader 的版本：

```sh
chaoqun@pi4:/$ sudo vcgencmd bootloader_version
Sep  3 2020 13:11:43
version c305221a6d7e532693cc7ff57fddfc8649def167 (release)
timestamp 1599135103
```

基本上你检查的版本在 20年5月 以后就是已经支持 USB 启动的了。

如果不幸，你的树莓派购买的稍早，那么你需要做如下操作：

1. 更新 bootloader
2. 下载最新的系统镜像（因为老的系统镜像的引导文件也不支持 USB 启动

更新 bootloader 参照[官方描述]((https://www.raspberrypi.org/documentation/hardware/raspberrypi/booteeprom.md))，最简单的方法是利用apt直接从官方源中更新。
复杂的方法则要手动下载最新版本的文件手动安装，这里不推荐也不再赘述了。

```sh
sudo apt update
sudo apt full-upgrade
```

### 把系统安装到 USB存储设备上

在 USB 存储设备的选择上，我进行了些对比：

* **U盘**： 支持 USB 3.0 的 U 盘速度不一定快，很多便宜的 U 盘比 SD 卡还要慢。速度快的 U 盘（指读写都超过 100M/S 的速度） 32G 售价大约百元；
* **机械移动硬盘**：自己有几个移动硬盘，但是又不太舍得把硬盘格式化了，而且硬盘容量比较大，放在树莓派上非常浪费；
* **小容量SSD + 硬盘盒**：我淘宝搜了下，选择了最便宜的 NGFF 的 SATA 协议的 SSD 固态硬盘，64G的容量只需要69元可以买到，硬盘盒约30+元的售价，综合起来110元左右可以实现64G的SATA固态存储。

综合对比，我选择从淘宝购买了64G的固态和硬盘盒，最后到手一看还挺有质感，测速数据如下：

![ssd 订单](/post_img/202001/order_ssd.jpg)

![ssd 测速](/post_img/202001/ssd_speed.png)

![ssd 外观](/post_img/202001/ssd_photo.png)

简陋的国产 ssd 本体图片

![ssd 硬盘盒](/post_img/202001/ssd_portal.jpg)

简陋的硬盘装入了铝合金外壳硬盘盒，顿时显得高大上了起来。

系统安装到 USB 存储的操作和写入到 SD 卡基本一致，通过 树莓派的 imager 软件，将镜像写到对应的u盘即可。

最后扣掉SD卡，正常启动树莓派，进行自己的配置。

![自己配置](/post_img/202001/config_youself.jpg)

## 总结

用好了固态的树莓派依旧比较慢。。。不过瓶颈不在 io 读写上了，应该在四颗弱鸡A72本身了，支持无sd卡usb启动很满足，希望后面树莓能直接出一个板载存储，速度就更好了。

我的树莓派平日里就丢在桌子的下面，和交换机接在一起。

![树莓派日常](/post_img/202001/pi_run.jpg)

## Reference

1. [关于树莓派4 的 boot EEPROM](https://www.raspberrypi.org/documentation/hardware/raspberrypi/booteeprom.md)
2. [树莓派4 的CPU](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2711/README.md)
3. [树莓派的系统镜像](https://www.raspberrypi.org/software/)
