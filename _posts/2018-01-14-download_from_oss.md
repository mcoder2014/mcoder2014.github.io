---
layout:     post
title:      "linux 终端从阿里云 OSS 下载文件"
subtitle:   "写了 python 脚本方便在 linux 终端下下载、上传、查看阿里云 OSS 上的文件"
date:       2018-01-14
author:     Mcoder
header-img: img/JCQ_0010.jpg
catalog: true
tags:
  - 阿里云
---

# 说明

最近放寒假回家了，准备租用阿里云的 [GPU 云服务器 ]( https://www.aliyun.com/product/ecs/gpu ) 按量收费，训练时临时购买，训练完了就释放。但这样的话，每次要上传训练数据，需要耗费很多时间。这时我想到了阿里云的 [OSS对象存储服务](https://www.aliyun.com/product/oss)。

阿里云的 OSS服务相当于一个云盘，按存储量、访问次数、下载流量 三项计费，而且外网下载收费，内网之间访问不收钱。如此一来，我便可以只花少量的存储费用便可以把我的数据集托管在阿里云上。

使用体验，阿里云的内网传输速度确实极快，经过我用多次传输 1G大小的文件的体验，内网速度在50M以上，如果你选择的是SSD云盘，估计速度会更快吧。

## Notes
因为阿里云的 OSS 也按访问量收费，所以你就不要上传太多的碎文件了，最好打包成一个压缩包，下载到服务器后再解压。碎文件不仅上传速度慢，还需要多收费，得不偿失呀！

# OSS的使用

OSS使用起来还算方便，有提供 [OSS-browser](https://help.aliyun.com/document_detail/61872.html) 这样的客户端软件，这个软件有界面，很方便操作使用。但问题就在于它有界面……

所以在 linux 云服务器上使用起来并不是很方便，因此，我打算结合[阿里云OSS-SDKpython](https://help.aliyun.com/document_detail/32026.html) 来写一个简单的脚本来实现上传、下载、查看 OSS 文件的功能。

# 功能描述
总共实现了三个功能: 下载、上传、查看文件。

实现的功能很简单，先设置好阿里云的 AccessKeyId 和 AccessKeySecret ，然后设置你所访问的 bucket 所在的区的链接和你所需要访问的 bucket 的名称。之后就可以在 linux 终端上访问。

![](/post_img/201801/showfiles.png)

# 用法描述

## 下载
```shell
python download_from_oss.py -f file1 -f file2 -o ./dest/
# -f , --files 你需要下载的OSS上的文件名称，一个 -f 后面只跟一个文件
# -o, --outputPath 你需要统一放置在哪个本地路径下，路径不存在会自动创建
# -i, --internal 是否是阿里云内网, 不是内网的话，不用填写
```

## 查看文件列表
```shell
python download_from_oss.py -l
# -l, --listfiles 查看文件
# -i, --internal 是否是阿里云内网, 不是内网的话，不用填写
```

## 上传文件
```shell
python download_from_oss.py -f ./file1 -f ./file2 -p log/test1 --upload
# -f , --files 你需要上传的本地文件，一个 -f 后面只跟一个文件
# -p, --prefix 给你在 oss 上统一添加前缀，可以模仿把文件全部上传到某个文件夹中的操作
# -i, --internal 是否是阿里云内网, 不是内网的话，不用填写

```

# [脚本下载链接](http://mcoder-download.oss-cn-shanghai.aliyuncs.com/download_from_oss.py)

我的脚本原本使用 git 作为版本控制，但当前该深度学习工程还没有完全完成，因此暂时不开源。待到6月后验收完成后，我就会放到 github 上开源。到时补上一个 github 项目链接。

# Reference
1. [阿里云GPU云服务器](https://www.aliyun.com/product/ecs/gpu)
2. [阿里云OSS对象存储服务](https://www.aliyun.com/product/oss)
3. [阿里云OSS-SDKpython](https://help.aliyun.com/document_detail/32026.html)
