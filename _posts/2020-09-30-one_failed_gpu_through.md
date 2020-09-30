---
layout:     post
title:      "KVM 虚拟化"
subtitle:   "一次失败的显卡穿透经历"
date:       2020-09-30
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - 云计算
    - 后台开发
    - 虚拟化
    - 硬件穿透
---

了解到 KVM 虚拟化技术，可以直接将宿主机的物理硬件穿透到客户机，让客户机独占硬件，所以我尝试在自己的笔记本上做实验。

场景说明：

* 笔记本：拯救者 Y7000P 2018版 GTX 1060（这个版本的电脑一言难尽，首先它是双显卡，电脑显示器与 Intel 核显直连，而电脑的 HDMI 接口和独显直连。这个奇怪的结构让我没办法完美安装 nvidia 驱动，我尝试了交火驱动和单独nvidia驱动都有各种毛病。后来放弃折腾了，就只装了一个 nouveau 驱动勉强凑活着。
* 操作系统：Manjaro Linux 5.8

因为笔记本上的 linux 本身就没办法使用 NIVIDA 的独显，所以安装一个虚拟机，把 NVIDIA 独显穿透过去，岂不美哉？

不过，经过了一整天的折腾，我都没有成功。

## KVM 硬件穿透的思路

经过资料搜集，我理解的 kvm 硬件穿透的流程如下：

1. 首先硬件上要支持，主要是 cpu 和主板都支持硬件穿透才可以；
2. 配置好 kvm 相关的软件包，可以基于 kvm 安装虚拟机，大约需要先安装 kvm 和 kvm_intel 内核模块，再安装 `qemu libvirt virt-manager` 等相关软件包；
3. 在宿主机上屏蔽掉想要穿透的硬件，为他强行绑定一个do nonthing 的驱动 vfio，避免宿主机和客户机同时对硬件访问造成冲突；
4. 安装 OVMF ，其作用是提供 UEFI 的虚拟机引导方式；
5. 在 virt-manager 中设置客户机的配置时，将 PCI 设备分配给客户机；
6. 在客户机上为对应的设备安装驱动，完成使用。

Manjaro 的 KVM 硬件传统教程在[PCI passthrough via OVMF (简体中文)](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E8%AE%BE%E7%BD%AE_OVMF_%E8%99%9A%E6%8B%9F%E6%9C%BA)一文中讲的非常详细，我是参照这篇文章进行解决的。

## 碰到问题的地方

### NVIDIA 驱动的反虚拟机问题

首先我没想到的是，NVIDIA 的显卡驱动居然会检查是否在虚拟机环境，如果它认为自己在虚拟机环境，就不会工作。。表现为：

* Windows 下安装驱动报 43 错误
* Linux 安装驱动后，运行 nvidia-smi 无法找到显卡

需要编辑虚拟机的 xml 文件，添加下面参数，value 为任意12位即可。

```xml
<hyperv>
    <vendor_id state='on' value='0123456789ab'/>
</hyperv>
<kvm>
    <hidden state='on'/>
</kvm>
```

解决了这个问题后，报错变了，但依旧找不到显卡。。。。。

我尝试更换驱动，自己编译驱动，都没解决，暂时放弃，等收拾好情绪再次尝试。

## Reference

1. [PCI passthrough via OVMF (简体中文)](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E8%AE%BE%E7%BD%AE_OVMF_%E8%99%9A%E6%8B%9F%E6%9C%BA)
2. [Ubuntu+KVM显卡穿透](https://blog.csdn.net/m0_38139137/article/details/103579397)
