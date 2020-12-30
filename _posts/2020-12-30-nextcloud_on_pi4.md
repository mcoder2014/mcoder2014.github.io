---
layout:     post
title:      "树莓派当私有云"
subtitle:   "nextcloud on raspiberry4"
date:       2020-12-30
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - linux
    - 树莓派
    - nextcloud
---

不出意外这就是 2020 年的最后一篇博客，写个好玩的东西跟大家分享一下。话接上文，我在双十二期间买了一个树莓派4 4G 内存版，还给它配了个 64G 小固态。但好几天没想好怎么用它，在使用 oneDrive 时，被它超级慢的同步速度感到沮丧时，突然灵光一现，我自己装一个云盘好了。

那么先说结论，树莓派4安装 nextcloud 完全可行，而且性能非常吃的消，4 颗核心，长时间负载仅有不到一颗核心，树莓派温度日常 40 摄氏度左右，内存使用约 600 M，远没有超出树莓派的负载能力。

![load](/post_img/202001/load.jpg)

``` sh
chaoqun@pi4:~$ top

top - 13:36:42 up 8 days, 21:05,  1 user,  load average: 0.74, 0.87, 0.78
Tasks: 179 total,   2 running, 177 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.7 us,  1.3 sy,  0.0 ni, 98.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  4.3 us,  2.7 sy,  0.0 ni, 93.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  :  3.3 us,  2.7 sy,  0.0 ni, 94.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  :  0.7 us,  1.0 sy,  0.0 ni, 98.0 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   3740.1 total,    172.9 free,    570.4 used,   2996.9 buff/cache
MiB Swap:   1024.0 total,   1023.0 free,      1.0 used.   3093.4 avail Mem 

chaoqun@pi4:~$ sudo vcgencmd measure_temp
temp=38.0'C
```

## nextcloud 效果

nextcloud 是开源的云盘系统，不仅可以搭建私有云存储和同步数据，也提供联系人、日程管理功能，web端还提供私密的语音视频通话功能，云端的数据均可选择加密，客户端支持windows、macos、linux 三大 pc 平台，以及安卓、ios两大移动平台，完全足够家庭或中小型团队协作使用。

### 手机端

![phone1](/post_img/202001/nextcloud_phone1.jpg)

![phone1](/post_img/202001/nextcloud_phone2.jpg)

手机端相比百度云盘的网盘的最大特点就是没有广告，可能有些功能上的不够全，但常见的功能还是有的。

* 相册自动上传
* 文件上传、下载
* 文件分享、加密分享、限制时间分享等

### 电脑端

电脑端有点儿像 oneDrive，主要功能是对文件夹自动同步，像下载文件、浏览网盘文件等功能都通过网页端实现。

![desktop](/post_img/202001/nextcloud_desktop.png)

### 网页端

网页端功能比较多，不过初次使用时会有些不习惯（布局逻辑和百度网盘不同。经过简单的使用，我发现 nextcloud 还可以通过安装插件实现很多原本不能做到的功能，我装了个 pdf 阅读的功能，可以在网页上直接阅读 pdf，不需要把文件下载到本机。

而且 nextcloud 还支持 WebDAV 的协议，可以给一些软件作为云备份空间使用。

![login](/post_img/202001/nextcloud_login.jpg)

![file](/post_img/202001/nextcloud_web.jpg)

## 安装配置方法

### 硬件配置

* 树莓派 4
* USB 固态硬盘（TF卡的读写太弱，可能会严重影响体验

### 软件配置

* Ubuntu or 或者树莓派的官方系统
* mysql 数据库（可选）
* nginx、php 环境
* redis （可选）

### 配置过程

因为我使用的是 ubtuntu 20.10, 我就直接 apt 安装了

```sh
sudo apt update
sudo apt upgrade

# 视频解码
sudo apt install ffmpeg

# 数据库
sudo apt install mariadb-server redis

# 网页服务器 及 php
# 如果源里不是 php 7.4 根据版本自行配置这些
sudo apt install nginx php-common php-igbinary php-imagick php-mysql php-pear php-redis php7.4-bz2 php7.4-cli php7.4-common php7.4-curl php7.4-dev php7.4-fpm php7.4-gd php7.4-intl php7.4-json php7.4-mbstring php7.4-mysql php7.4-opcache php7.4-readline php7.4-xml php7.4-zip php7.4
```

从官网下载 nextcloud 的部署程序 [nextcloud-20.0.4.tar.bz2](https://nextcloud.com/install/#instructions-server)

后续配置过程参考 [树莓派安装 nextcloud 搭建私有云](https://tlanyan.me/raspberry-pi-install-nextcloud/) 配置，不做赘述。

### 外部访问

这里简单的讲一下如何在外部访问 nextcloud，通常情况下，配置好了 nextcloud ，也仅能在内网下访问，所以想要外部访问需要配置 frp 等内网穿透工具。
我购买了阿里云云服务器，在上面配置了 frps 服务，通过在树莓派配置端口映射，将 `127.0.0.1:80` 端口映射到服务器的随便一个端口 `xxx.mcoder.cc:7999`。

之后，配置一条 DNS 解析至云服务器，例如: `mycloud.mcoder.cc`。然后在 nginx 中添加一条代理服务器记录，之后便可通过 `mycloud.mcoder.cc` 在外网访问自己的私有云了。

```nginx
server {
        listen 80;
        listen [::]:80;

        server_name mycloud.mcoder.cc;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                #try_files $uri $uri/ =404;
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Scheme $scheme;

                proxy_pass http://127.0.0.1:7999/;
                index index.html index.htm index.jsp index.php;
        }

        # 设置最大上传文件大小
        client_max_body_size 4192M;
        fastcgi_buffers 64 4K;
}
```

注意：有一点需要注意的是，文件上传最大大小需要在好几处做设置：nextcloud、树莓派的 nginx 服务器、云服务器代理的 nginx 服务器。

有私有云盘需求的小伙伴可以自己尝试一下哦，蛮好玩的。

## 参考链接

1. [树莓派4 Usb 引导](https://www.mcoder.cc/2020/12/14/pi4_boot_from_usb/)
2. [树莓派安装 nextcloud 搭建私有云](https://tlanyan.me/raspberry-pi-install-nextcloud/)
