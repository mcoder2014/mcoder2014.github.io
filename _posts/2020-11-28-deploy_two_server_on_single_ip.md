---
layout:     post
title:      "在树莓派上部署多个帮助文档镜像"
subtitle:   "讨论如何在同一个ip下部署多个网站"
date:       2020-11-28
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - linux
    - 树莓派
    - nginx
---

之前我们有讨论过[如何使用 nginx 实现一个简易版的 Github Pages](https://www.mcoder.cc/2020/11/14/a_idea_of_github_pages/)，这次我们讨论下如何在一个 ip 下部署多个网站。

先谈谈自己的需求：**众所周知，在国内访问境外网站可能会很慢，或者不通。对于程序员使用框架时查看文档非常的不利，但有些第三方库的文档是可以下载到本地的，比如：Qt 自带了文档浏览工具 `Qt Assistant`，在该软件上看文档，比看在线文档快的不止一点点。**

![qt assistant](/post_img/202001/qt_assistant.jpg)
qt assistant 截图

![qt doc online](/post_img/202001/qt_doc_online.jpg)
qt 在线文档截图

但有些第三方库的文档不提供这么方便的工具，是以 html 等形式提供文档，比如 [eigen](http://eigen.tuxfamily.org/dox/) [openmesh](https://www.graphics.rwth-aachen.de/media/openmesh_static/Documentations/OpenMesh-Doc-Latest/index.html) 等等。还有些教程博客、电子图书则以 mkdoc jekyll hexo 等形式提供静态网页。他们大都具有这些特点：

1. 在线的文档、电子书从国内访问速度极慢;
2. 提供了 html、markdown 等离线文档，且获取离线文档比较容易；

为了获得像在线文档那样丝滑的使用体验，我决定将部分文档、电子书部署到自己的树莓派上，这样在校园网内，我看文档时可以获得很好的体验，而且不会受电脑关机、重启等影响，甚至可以和同学共用加速了的文档。

## 部署方案

这里我使用两个离线 html 文档(eigen, openmesh)，一个 mkdoc 教程(learnOpenGL-CN)来演示

### 方案一

我可以通过网站目录分级的形式区分不同的文档，效果如下：

* `pi.mcoder.cc/eigen/` 表示 eigen 的文档；
* `pi.mcoder.cc/openmesh/` 表示 openmesh 的文档；
* `pi.mcoder.cc/learnOpenGL/` 表示 learningOpenGL 的教程网址

### 方案二

通过不同的二级域名来区分不同的文档，效果如下：

* `eigen.mcoder.cc` 表示 eigen 的文档；
* `openmesh.mcoder.cc` 表示 openmesh 的文档；
* `learnopengl.mcoder.cc` 表示 learningOpenGL 的教程网址；

两种方案有各自的特点，方案一强调了是同一个网站下的不同的分页面，方案二则在网站上给人一种隔离的感觉。

## 实现方案

### 准备文档

首先，我们需要准备好需要用作网站的文档，eigen 和 openmesh 的文档是通过 doxygen 工具生成的，需要先配置好环境。

```sh
sudo apt install nginx doxygen
```

然后下载源码，cmake 生成 Makefile， `make doc` 生成文档，用 eigen 举例子：

```sh
# 从 gitee 的加速仓库获得源码
git clone https://gitee.com/mirrors/eigen-git-mirrorsource.git
cd eigen-git-mirrorsource
mkdir build && cd build
cmake ..

# 编译生成文档
make doc

# 将生成的文档复制到指定路径
sudo cp -r doc/html /var/www/html

# 更改 eigen文档的名称
sudo mv /var/www/html/html /var/www/html/eigen-doc
sudo chmod a+wr  var/www/html/eigen-doc
```

编译完成后，文档位于 `build/doc/html` 中，我们将文档放在复制到一个好记的路径，这里选择了nignx 的默认路径 `/var/www/html/`，给予足够的权限，让 nignx 的进程可以读取该文件。

相同的原理，可以配置好 openmesh 的文档，位于 `/var/www/html/openmesh-doc`

对于 learnOpenGL-CN 则不太一样，它需要起一个服务才能够访问

```sh
# 安装 mkdocs 的内容
sudo pip3 install mkdocs pyton-markdown-math

# clone 仓库
git clone https://github.com/LearnOpenGL-CN/LearnOpenGL-CN.git
cd LearnOpenGL-CN

# 启动网页服务
mkdocs serve -a 0.0.0.0:8000
```

### 通过目录分级形式

通过目录形式分级，我们假设为树莓派配置一个 DNS 解析，可以通过域名解析实现，也可以用过改客户机的 hosts 文件实现。在实验中我们向 `/etc/hosts` 文件中加入如下记录，即实现:

```hosts
10.4.54.76 pi.mcoder.cc eigen.mcoder.cc openmesh.mcoder.cc learningopengl.mcoder.cc
```

然后我们在 `/etc/nginx/conf.d` 中新建一个配置文件 `doc.conf` 写入如下的内容，来实现这种方案的部署。

```nginx
server {
    # 绑定端口
    listen 80;
    listen [::]:80;

    # 配置服务器名称
    server_name pi.mcoder.cc 10.4.54.76;

    root /var/www/html;

    # eigen 文档
    location /eigen {
        alias /var/www/html/eigen_doc;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    # learning opengl 文档
    location /learnOpenGL/ {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_pass http://127.0.0.1:8000/;
        index index.html index.htm index.jsp;
    }

    # openMesh 文档
    location /openmesh {
        alias /var/www/html/openmesh_doc;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }
}
```

修改配置，重启网页服务器。

```sh
# 检查配置文件是否出错
sudo nignx -t

# 重启网页服务器
sudo systemctl restart nginx
```

至此，我们可以通过网页的不同层级来访问不同的博客。

![dir](/post_img/202001/doc_dir.jpg)

### 通过二级域名区分不同博客

在这种方式下，我们可以一个域名写一个文件，也可以把多个文档的配置写在同一个文档下，都是可以的，这里方便起见写在同一个文件 `/etc/nginx/conf.d/doc2.conf` 中。

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name eigen.mcoder.cc;
    root /var/www/html/eigen_doc;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name openmesh.mcoder.cc;

    root /var/www/html/openmesh_doc;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

}

server {
    listen 80;
    listen [::]:80;
    server_name learningopengl.mcoder.cc;

    root /var/www/html;
    index index.html index.htm;

    location / {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_pass http://127.0.0.1:8000/;
    }

}

```

```sh
# 检查配置文件是否出错
sudo nignx -t

# 重启网页服务器
sudo systemctl restart nginx
```

![doc in domain ways](/post_img/202001/doc_domain.jpg)

## 总结

通过上述操作，我们实现了两种方法来在一个树莓派上托管多个文档，既可以通过同一个域名下的多个层级方式，也可以通过不同域名的方式。如果把常用到的文档做成镜像挂载在校园网下供实验室同学一起使用的话，还是很令人高兴的事情。

## Reference

1. [如何使用 nginx 实现一个简易版的 Github Pages](https://www.mcoder.cc/2020/11/14/a_idea_of_github_pages/)
2. [eigen 在线文档](http://eigen.tuxfamily.org/dox/)
3. [openmesh 在线文档](https://www.graphics.rwth-aachen.de/media/openmesh_static/Documentations/OpenMesh-Doc-Latest/index.html)
4. [learningOpengl-cn](https://learnopengl-cn.github.io/)
