---
layout:     post
title:      "虚拟化"
subtitle:   "总览"
date:       2020-09-13
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - 云计算
    - 后台开发
    - 虚拟化
---

虚拟化技术是云计算的基础，我们在各种云平台可以购买 IaaS(基础设施即服务) 的云服务器，包括但不局限于 `ECS,VPS` 等，这些云服务器可以自选配置，动态调整 cpu 核数、内存大小、甚至是硬盘大小、网卡数量等。他们底层的实现原理是虚拟化技术。

## 虚拟化技术的分类

根据虚拟的资源进行划分可以分为操作系统虚拟化和平台虚拟化：

* 操作系统虚拟化： 如果把操作系统及其提供的系统调用作为资源，那么虚拟化表现为操作系统虚拟化，比如docker。
* 平台虚拟化： 如果把整个 X86 平台包括处理器、内存、外设等硬件作为资源，在虚拟的 x86 平台之上可以安装完整的操作系统，比如 virtualbox。

虚拟化技术的实现也有很多种方式，比如软件虚拟化和硬件虚拟化。实现虚拟化的关键在于虚拟化层能够截获计算元件对物理资源的直接访问，并将它重新定位到虚拟资源池中。

* 软件虚拟化： 软件虚拟化是通过纯软件的方式在现有的物理平台实现对物理平台访问的截获和模拟，比如 QEMU 便支持软件虚拟化，它可以通过纯软件仿真 X86 平台处理器的取值、解码和执行，客户机所有的操作都由软件翻译后再执行到真实的物理平台。
* 硬件虚拟化： 物理平台本身提供了对特殊指令的截获和重定向的硬件支持，比如 intel 的 VT-x。

根据客户机是否需要进行特殊的改动可以分为部分虚拟化和完全虚拟化。

* 部分虚拟化(para-virtualization)： 部分虚拟化实现上可能存在较大的性能损失，但如果改动客户机操作系统，让他知道自己运行在虚拟环境下，从而可以对部分操作（比如IO）优化与宿主机的通信，从而提升性能。这个技术争议的地方在于需要对客户机的操作系统进行改动。
* 全虚拟化(full virtualization)： 全虚拟化技术为客户机提供了完整的虚拟 x86 平台，包括处理器、内存和外设，支持运行任何理论上可以在真是物理机上运行的操作系统，不需要针对客户机操作系统进行修改即可使用。

## KVM 介绍

当前主流云计算平台使用多是 KVM 技术，它是 Linux 原生支持的全虚拟化解决方案，代码已经内置 linux 内核源码中。在 KVM 架构中，虚拟机实现为常规的 linux 进程，每个虚拟 CPU 为一个常规的 linux 进程。通常 KVM 需要和 硬件虚拟化软件一同工作，比如同 libvirt、QEMU 一同工作。

当我们在阿里云的 ECS 中查看机器的 PCI 设备时，可以发现部分设备的描述为`Red Hat, Inc. Virtio network device`，这便是使用功能 libvirt 虚拟化了网卡、块存储等硬件设备。

···shell
Last login: Sun Sep 13 16:12:24 2020 from xxx.xxx.xxx.xxx
aliyun@localhost:~$ lspci
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma] (rev 02)
00:01.0 ISA bridge: Intel Corporation 82371SB PIIX3 ISA [Natoma/Triton II]
00:01.1 IDE interface: Intel Corporation 82371SB PIIX3 IDE [Natoma/Triton II]
00:01.2 USB controller: Intel Corporation 82371SB PIIX3 USB [Natoma/Triton II] (rev 01)
00:01.3 Bridge: Intel Corporation 82371AB/EB/MB PIIX4 ACPI (rev 03)
00:02.0 VGA compatible controller: Cirrus Logic GD 5446
00:03.0 Ethernet controller: Red Hat, Inc. Virtio network device
00:04.0 Communication controller: Red Hat, Inc. Virtio console
00:05.0 SCSI storage controller: Red Hat, Inc. Virtio block device
00:06.0 Unclassified device [00ff]: Red Hat, Inc. Virtio memory balloon
···
