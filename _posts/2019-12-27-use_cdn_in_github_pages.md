---
layout:     post
title:      "给 Github pages 用上 CDN加速"
subtitle:   "阿里云 & 腾讯云"
date:       2019-12-27
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - C
    - gcc
    - cpp
    - Linux
---

# 给博客用上 CDN 加速

我们先假设大家已经知道如何使用 github pages 托管自己的静态博客网站了，这样我们不用花钱自己维护一个云服务器，也不需要在C某家充满广告的博客网站上写博客，很美好对吧。
但 Github pages 在国内的效果不佳，常常出现打开缓慢、或者无法打开等情况。

为了提高网站的访问速度，除了减少网页的请求数量，精简网页所依赖的js、css文件外，最常用的就是使用CDN缓存大文件到节点服务器上，这样用户访问网站的时候就会从最近的，响应最快的CDN节点服务器上拉取资源，从而提高了大文件的下载速度。

给 github 博客加速需要哪些东西呢？
1. Github Pages博客；
2. 一个自己的域名；
3. 给自己域名备案；
4. 一个免费的SSL证书；

## 阿里云 全站加速 DCDN
阿里云加速可以使用全站加速功能。


## 腾讯云 内容分发网络 CDN
1. [为Github pages增加腾讯云cdn加速功能](https://yejiayong.com/%E4%B8%BAGithub-pages%E5%A2%9E%E5%8A%A0%E8%85%BE%E8%AE%AF%E4%BA%91cdn%E5%8A%A0%E9%80%9F%E5%8A%9F%E8%83%BD/)
2. [Github Pages + CDN全站加速](https://www.jianshu.com/p/ccc0cc8c14a0)

# 循环 重定向 解决方案
我碰到了出现了重定向次数过多导致浏览器无法打开网页的问题。起初根据[接入CDN/WAF后出现循环重定向问题的排查记录](https://yq.aliyun.com/articles/594115)博客做分析，做了些调整，问题短暂性解决。然后之后，还是数次出现了重定向多次的错误。

## 产生错误的原因
之后偶然间我注意到了阿里称为**回源域名**、腾讯称为**回源HOST的东西**。

我错误的将这个回源域名设置成了github的域名`mcoder2014.github.io`，我以为这个回源的概念是，我要去哪一个仓库去取资源，但似乎是CDN用哪个网站名称去取资源。

## github pages 设置自定义域名
github pages 设置好了 CNAME 自定义域名之后，便强制用户的浏览器跳转到 用户自定义域名。
同样，我们在不使用CDN时，打开[https://mcoder2014.github.io](https://mcoder2014.github.io)也会被跳转到[https://mcoder.cc][https://mcoder.cc]，而CDN中错误的设置了回源host，会导致出现一个死循环，即 github 通知跳转了，结果CDN依旧使用[https://mcoder2014.github.io](https://mcoder2014.github.io)去回源取资源。

就向下图所画的。

![](/post_img/201901/wrong_cdn.png)


而正确的流程如下，CDN在HTTP请求中携带正确的地址，于是github便会将正确的资源发送给CDN，也就解决了循环重定向的问题。
![](/post_img/201901/right_cdn.jpg)


# Reference
1. [接入CDN/WAF后出现循环重定向问题的排查记录](https://yq.aliyun.com/articles/594115)