---
layout:     post
title:      "OpenMesh"
subtitle:   "半边数据结构汇总"
date:       2019-07-06
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
  - C++
  - openGL
  - Manjaro
  - linux
  - 半边数据结构
---

# 半边数据结构
![](post_img/201901/half_edge.png)
半边数据结构具有如下特点：
1. 每个顶点对应一个出边。例如，一个半边从顶点1出发；
2. 每个面对应一个半边为它的边界，例如面2；
3. 每个半边指向的对象如下：
  - 指向下一个顶点3；
  - 指向一个面4；
  - 指向下一边半边5；
  - 指向反向的半边6；
  - 指向上一个半边7（可选）

# OpenMesh


# 想关算法

# Reference

1.[OpenMesh](https://www.openmesh.org/media/Documentations/OpenMesh-Doc-Latest/a04074.html)
