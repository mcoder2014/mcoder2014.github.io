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

首先我没想到的是，NVIDIA 的显卡驱动居然会检查是否在虚拟机环境，如果它认为自己在虚拟机环境，就不会工作。。其他人表现为：

* Windows 下安装驱动报 43 错误
* Linux 安装驱动后，运行 nvidia-smi 无法找到显卡

实验中实际表现为：

```sh
~ >>> nvidia-smi
Unable to determine the device handle for GPU 0000:07:00.0: Unknown Error
```

需要编辑 libvirt 虚拟机的 xml 文件，添加下面参数 到 `<features></features>`位置，value 为任意12位即可。

```xml
<hyperv>
    <relaxed state="on"/>
    <vapic state="on"/>
    <spinlocks state="on" retries="8191"/>
    <vendor_id state="on" value="123456789123"/>
</hyperv>
<kvm>
    <hidden state="on"/>
</kvm>
```

解决了这个问题后，报错内容发生了变化，但主要意思依旧是找不到显卡：

```sh
~ >>> nvidia-smi
No devices were found
```

这里，我尝试了多种 NVIDIA 驱动，都没能解决：

* 从官方源下载的最新驱动 linux58-nvidia-450xx
* 从官方源下载的稍久一些的驱动 linux58-nvidia-430xx
* 从 NVIDIA 网站下载的驱动 xx450xx.run 手动安装

## VBIOS 与 UEFI 兼容问题

后来经过一番搜索，可以认为在穿透的过程中发生了 VBIOS 的兼容问题（大概）

客户机启动后，通过排查 log 可以看到类似的错误信息:

```txt
sudo dmesg | grep NVRM

[    5.043645] NVRM: loading NVIDIA UNIX x86_64 Kernel Module  450.66  Wed Aug 12 19:42:48 UTC 2020
[    5.401373] NVRM: GPU 0000:07:00.0: Failed to enable MSI; falling back to PCIe virtual-wire interrupts.
[    5.433614] NVRM: GPU 0000:07:00.0: Failed to copy vbios to system memory.
[    5.433825] NVRM: GPU 0000:07:00.0: RmInitAdapter failed! (0x30:0xffff:794)
[    5.439493] NVRM: GPU 0000:07:00.0: rm_init_adapter failed, device minor number 0
[   24.653394] NVRM: GPU 0000:07:00.0: Failed to enable MSI; falling back to PCIe virtual-wire interrupts.
[   24.657640] NVRM: GPU 0000:07:00.0: Failed to copy vbios to system memory.
[   24.657827] NVRM: GPU 0000:07:00.0: RmInitAdapter failed! (0x30:0xffff:794)
[   24.658861] NVRM: GPU 0000:07:00.0: rm_init_adapter failed, device minor number 0
[   26.235602] NVRM: GPU 0000:07:00.0: Failed to enable MSI; falling back to PCIe virtual-wire interrupts.
[   26.241812] NVRM: GPU 0000:07:00.0: Failed to copy vbios to system memory.
[   26.241941] NVRM: GPU 0000:07:00.0: RmInitAdapter failed! (0x30:0xffff:794)
[   26.242174] NVRM: GPU 0000:07:00.0: rm_init_adapter failed, device minor number 0
[   27.023331] NVRM: GPU 0000:07:00.0: Failed to enable MSI; falling back to PCIe virtual-wire interrupts.
[   27.034210] NVRM: GPU 0000:07:00.0: Failed to copy vbios to system memory.
[   27.034393] NVRM: GPU 0000:07:00.0: RmInitAdapter failed! (0x30:0xffff:794)
......
```

其中关键的报错为 `GPU 0000:07:00.0: Failed to copy vbios to system memory.`

```txt
sudo cat /var/log/Xorg.0.log

[     5.394] (II) NVIDIA GLX Module  450.66  Wed Aug 12 19:41:37 UTC 2020
[     5.398] (II) NVIDIA: The X server supports PRIME Render Offload.
[     5.542] (EE) NVIDIA(GPU-0): Failed to initialize the NVIDIA GPU at PCI:7:0:0.  Please
[     5.542] (EE) NVIDIA(GPU-0):     check your system's kernel log for additional error
[     5.542] (EE) NVIDIA(GPU-0):     messages and refer to Chapter 8: Common Problems in the
[     5.542] (EE) NVIDIA(GPU-0):     README for additional information.
[     5.542] (EE) NVIDIA(GPU-0): Failed to initialize the NVIDIA graphics device!
[     5.542] (EE) NVIDIA(0): Failing initialization of X screen
[     5.542] (II) UnloadModule: "nvidia"
[     5.542] (II) UnloadSubModule: "glxserver_nvidia"
[     5.542] (II) Unloading glxserver_nvidia
[     5.542] (II) UnloadSubModule: "wfb"
[     5.542] (II) UnloadSubModule: "fb"
[     5.542] (EE) Screen(s) found, but none have a usable configuration.
[     5.542] (EE)
Fatal server error:
[     5.542] (EE) no screens found(EE)
[     5.542] (EE)
Please consult the The X.Org Foundation support
         at http://wiki.x.org
 for help.
[     5.542] (EE) Please also check the log file at "/var/log/Xorg.0.log" for additional information.
[     5.542] (EE)
[     5.543] (EE) Server terminated with error (1). Closing log file.
```

着相关的报错，让我们知道可能在 VBIOS 的地方出了问题。**但什么是 VBIOS 呢？大概相当于显卡的 BIOS。**

[PCI passthrough via OVMF (简体中文)](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E8%AE%BE%E7%BD%AE_OVMF_%E8%99%9A%E6%8B%9F%E6%9C%BA)中建议首先通过对比 VBIOS 的内容或版本来判断是否支持 EFI。

该文中建议使用如下方法提取 rom，但实际操作时提取不出来，会提示输入输出错误。

``` sh
sudo su root
echo 1 > /sys/bus/pci/devices/0000:01:00.0/rom
cat /sys/bus/pci/devices/0000:01:00.0/rom > /tmp/image.rom
echo 0 > /sys/bus/pci/devices/0000:01:00.0/rom
```

```sh
[manjaro 0000:01:00.0]# echo 1 > rom
[manjaro 0000:01:00.0]# cat rom
cat: rom: 输入/输出错误
[manjaro 0000:01:00.0]#
```

此外，还有一种可以读取、刷些 VBIOS 的程序 `nvflash`，可以从外网搜索下载。

```bash
[manjaro tmp]# ./nvflash_linux --version

NVIDIA Firmware Update Utility (Version 5.414.0)
Simplified Version For OEM Only
mmap(): /dev/mem[0xa3000000:0x1000000]: Operation not permitted
Device:01:00:00=10DE:1C20:0000:0000 GPU
ERROR: attempt to map physical memory failed
```

该软件却无法解读我的电脑的 VBIOS 的内容信息。所以一时半会不知道如何解决这个问题，只能大概认为是笔记本设计的问题了，等后面有机会用台式试一下。（因为稍微麻烦下，要多准备一张显卡）

所以一次失败而又愉快的 KVM 显卡穿透实验就此结束，稍晚点儿试试其他硬件穿透（移动硬盘、USB 分线器之类的），到时如果成功了会重新写一篇文章。

## Reference

1. [PCI passthrough via OVMF (简体中文)](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E8%AE%BE%E7%BD%AE_OVMF_%E8%99%9A%E6%8B%9F%E6%9C%BA)
2. [Ubuntu+KVM显卡穿透](https://blog.csdn.net/m0_38139137/article/details/103579397)
3. [VBIOS](https://en.wikipedia.org/wiki/Video_BIOS)
