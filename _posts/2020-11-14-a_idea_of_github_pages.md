---
layout:     post
title:      "一种实现 github pages 的思路"
subtitle:   "基于 nginx 反向代理"
date:       2020-11-14
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - 后台开发
    - nginx
---

从 20 年开始，我一直通过 [github pages + CDN 加速](https://www.mcoder.cc/2019/12/27/use_cdn_in_github_pages/) 的方案维持着自己的博客，效果还可以，只要 CDN 中有缓存，网站可以说是秒开。

这个时候就有些思考了，github pages 到底是怎么设计的，可以在一台机器上维持着这么多人的博客？难道它有那么多域名给每个人用嘛？

## 不同用户的博客指向的是同一台服务器吗

通过 ping 两个不同用户的 github pages，我们可以发现，其实很多用户之间可能是共享同一个公网 IP 的，效果如下：

```sh
mcoder@Chaoqun-PC:~/workspace/my_doc/blog$ ping mcoder2014.github.io -c 4
PING mcoder2014.github.io (185.199.109.153) 56(84) bytes of data.
64 bytes from 185.199.109.153 (185.199.109.153): icmp_seq=1 ttl=51 time=69.8 ms
64 bytes from 185.199.109.153 (185.199.109.153): icmp_seq=2 ttl=51 time=75.6 ms
64 bytes from 185.199.109.153 (185.199.109.153): icmp_seq=3 ttl=51 time=67.4 ms
64 bytes from 185.199.109.153 (185.199.109.153): icmp_seq=4 ttl=51 time=67.4 ms

--- mcoder2014.github.io ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 67.430/70.097/75.604/3.332 ms

mcoder@Chaoqun-PC:~/workspace/my_doc/blog$ ping LingjieLi.github.io -c 4
PING LingjieLi.github.io (185.199.109.153) 56(84) bytes of data.
64 bytes from 185.199.109.153 (185.199.109.153): icmp_seq=1 ttl=51 time=68.0 ms
64 bytes from 185.199.109.153 (185.199.109.153): icmp_seq=2 ttl=51 time=67.2 ms
64 bytes from 185.199.109.153 (185.199.109.153): icmp_seq=3 ttl=51 time=70.2 ms
64 bytes from 185.199.109.153 (185.199.109.153): icmp_seq=4 ttl=51 time=67.6 ms

--- LingjieLi.github.io ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 67.279/68.338/70.286/1.189 ms
```

能够看出，[lilingjie.github.io](http://lilingjie.github.io) 和 [mcoder2014.github.io](http://mcoder2014.github.io) 背后的 IP 地址都是 **185.199.109.153**，这时就更加疑问了，**为什么可以做到这样？为什么我们自己只能在一个网页服务器上放一个网站，而 github 可以在同一个网页服务器上安置多个网站？**

## 如何通过 Host 区分用户

### 访问 github pages 时，host 的变化

我们浏览器想要访问一个 HTTP 协议的网站时，一般会向网站发送类似如下的 HTTP 报文请求，其中有个字段是 Host，如下所示是打开 mcoder2014.github.io 的请求头：

```http
GET  HTTP/1.1
Host: mcoder2014.github.io
Cache-Control: no-cache
Postman-Token: 57c4f8bc-f1fa-9280-a41f-f5dc5de4dc95
```

因为我绑定了 CNAME，所以 github 会回我一个 301 跳转让我跳转到我的自定义域名，会先回复我如下的内容，其中 `Location: http://www.mcoder.cc/` 表示告诉浏览器访问这个页面。

```http
HTTP/1.1 301 Moved Permanently
Content-Type: text/html
Server: GitHub.com
Location: http://www.mcoder.cc/
X-GitHub-Request-Id: 1980:21E6:20941E:23886A:5FAF89EB
Content-Length: 162
Accept-Ranges: bytes
Date: Sat, 14 Nov 2020 07:45:34 GMT
Via: 1.1 varnish
Age: 307
Connection: keep-alive
X-Served-By: cache-hkg17922-HKG
X-Cache: HIT
X-Cache-Hits: 1
X-Timer: S1605339934.389667,VS0,VE0
Vary: Accept-Encoding
X-Fastly-Request-ID: a568f42a57741996deba79254398f498d9812737
```

然后，浏览器再次向 [https://www.mcoder.cc](https://www.mcoder.cc) http 请求：

```http
GET  HTTP/1.1
Host: www.mcoder.cc
Cache-Control: no-cache
Postman-Token: 57c4f8bc-f1fa-9280-a41f-f5dc5de4dc95
```

最后，github 的服务器返回我如下内容，其中并不会包含 **Host** 字段，仅包含 HTTP 响应报文和 HTML 形式的页面正文。也就是说，浏览器只记得它需要访问的地址 [https://www.mcoder.cc](https://www.mcoder.cc)，然后将这个地址填在地址栏。

```http
HTTP/2 200
server: Tengine
content-type: text/html; charset=utf-8
content-length: 32763
date: Sat, 14 Nov 2020 07:41:01 GMT
via: 1.1 varnish, cache22.l2cn1833[0,200-0,H], cache28.l2cn1833[1,0], kunlun1.cn2479[97,200-0,M], kunlun4.cn2479[112,0]
cache-control: max-age=600
etag: "5fabe926-7ffb"
expires: Fri, 13 Nov 2020 13:59:13 GMT
x-served-by: cache-hnd18746-HND
x-cache-hits: 0
x-timer: S1605339662.633919,VS0,VE188
vary: Accept-Encoding
x-fastly-request-id: 7a5cca133de355025def6546b750d82a324894fd
last-modified: Wed, 11 Nov 2020 13:37:42 GMT
access-control-allow-origin: *
x-proxy-cache: MISS
x-github-request-id: 7A46:678B:13328D:14CA57:5FAE8ED8
accept-ranges: bytes
ali-swift-global-savetime: 1605336886
age: 0
x-cache: MISS TCP_MISS dirn:-2:-2
x-swift-savetime: Sat, 14 Nov 2020 07:46:17 GMT
x-swift-cachetime: 284
timing-allow-origin: *
eagleid: 249c511816053399772526644e

空一行后紧跟网页的正文内容，用 HTML 语言形式。
```

**对于 github pages 的服务器来说，便可以通过 HTTP 请求的 host 字段不同，区分需要访问的博客。**

### 使用 NGINX 模拟不同 host 访问不同内容

我们在 nginx 中多设置几个 server ，配置为不同的 **server_name**，并设置不同的工作路径，以下仅复制了 nginx 配置文件中的两个 server 的内容。

```nginx
server{
    listen 80;  # 指定端口
    server_name user1.github.io;  # 指定域名
    location / {
        root /home/user1/web-content;  # 指定静态网站根目录
        index index.html;  # 指定默认访问文件
    }
}

server{
    listen 80;  # 指定端口
    server_name user2.github.io;  # 指定域名
    location / {
        root /home/user2/web-content;  # 指定静态网站根目录
        index index.html;  # 指定默认访问文件
    }
}
```

这种情况下，我们配置好 user1.github.io 和 user2.github.io 的 host，可以发现访问不同的网址，均能访问到对应的内容。

如此，我们便实现了仅使用一个网页服务器，对不同的域名访问到不同的内容。
**有的同学这是可能会有新的疑问，那 github 是不是需要为每一个用户都建立一个DNS记录呀，那如果有百万用户，是不是要建立百万条 DNS 记录呀？**

实际上，大多数 DNS 域名解析支持泛解析，它可以将 `*.github.io` 解析到 `185.199.109.153`，也就是所有用户的博客都可以解析到一个网页服务器。而 github pages 可以通过 jekyll 或者 hexo 等工具，将 markdown 文件转换成网站目录，只要记录下每个用户的网站根路径，动态更改 nginx 中 server 中的内容即可。按照我的想法，github 极有可能自己实现了一个专用于 github pages 的静态网页服务器，这样子便不需要频繁修改 nginx 的配置文件，可用性更高。

## CNAME 功能实现

可能还有同学会问，github 可以通过 CNAME 实现自定义域名的覆盖，这是怎么实现的呢？ 我们可以在 nginx 中配置一个 rewrite 函数，主动要求用户浏览器跳转到指定的 CNAME 页面。比如：用户1 通过 CNAME 配置了自定义域名 `www.user1.com`，那我们为 server `user1.github.io`加入 rewrite 函数，要求跳转到 `www.user1.com`。这样，所有访问 `user1.github.io` 的请求都会被重定向到 `www.user1.com`。

```nginx
server{
    listen 80;
    server)name user1.github.io;
    location / {
        rewrite ^/(.*) http://www.user1.com permanent;
    }
}

server{
    listen 80;  # 指定端口
    server_name www.user1.com;  # 指定域名
    location / {
        root /home/user1/web-content;  # 指定静态网站根目录
        index index.html;  # 指定默认访问文件
    }
}
```

## 那么，总结下实现 github pages 的思路

如何自己模拟一个 github pages 呢？

1. 为每个用户配置个静态页面路径，配置 git 的钩子，当用户 git push 后，更新静态页面路径内容，并执行 `jekyll build`；
2. 配置 nginx，为不同用户配置不同的 server；
3. 如果用户配置了 CNAME，则通过 rewrite 函数让用户从 `xxx.github.io` 跳转到 `xxx.com`；
4. 如果新增用户、用户修改用户名、等等需要更新配置文件的操作，可以通过 nginx 热更新的方式更新配置文件，避免停机。

经过上述步骤，我们便可以自己实现一个类似于 github pages 的功能了，不过实际情况可能远比描述的复杂，需要具体问题具体分析了。

## Reference

1. [github pages + CDN 加速](https://www.mcoder.cc/2019/12/27/use_cdn_in_github_pages/)
2. [使用Nginx实现反向代理](https://blog.csdn.net/Daybreak1209/article/details/51549031)
