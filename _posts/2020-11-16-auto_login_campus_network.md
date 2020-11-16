---
layout:     post
title:      "linux 定时任务"
subtitle:   "校园网自动登录"
date:       2020-11-16
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - linux
    - 树莓派
---

今年的九月份，我们学校更换了校园网计费系统，将多个网络（校园网、移动宽带、校园内网）整个为统一的登录入口。新的计费方式带来了一些令人不爽的新特性：**掌握不到规律的自动断线**。

这个自动断线怎么理解呢，有好多种情况会导致自动断线：

1. 长时间没有数据访问；
2. 关机了一段时间重新开机也会断线；
3. 有时晚上会突然的断线；

断线就需要重新登录，这对一个在桌子底下放着树莓派的人很不友好，为了让我的树莓派一直在线，**我计划研究下学校的网络连接机制，找到校园网登陆认证的接口，然后通过定时任务定期调用接口保证网络在线。**

## 找到校园网登陆认证的接口

开机连接到网络时，打开任何页面都会自动跳转到网络验证页面，地址如下：

```http
http://10.**.**.155/eportal/index.jsp?wlanuserip=747782dfc4c71585c4738aba9ddfac74&wlanacname=4d9b42af64216a4c&ssid=&nasip=33a23ff6f4079ffae1efa910adf8630a&snmpagentip=&mac=cea4ee74202c8e19054f5242fed8a793&t=wireless-v2&url=d93a492329ace846b5542d1a14450dc01c67fe12a7c37ec6&apmac=&nasid=4d9b42af64216a4c&vid=80e2309e88a7305c&port=ec0e0d680caa36ef&nasportid=75961153f3a76ba5b8d71721a48c6a36d27cfbc13a5acc9ea2aa9e8eb1cf5c88
```

![校园网验证页面](/post_img/202001/campus_network_auth.jpg)

为了实现自动登录，我们先尝试找到校园网登陆认证的接口，通过浏览器的开发者工具页面(按键盘 F12 进入)，找到登录按钮对应的函数调用。

![登录按钮的函数调用](/post_img/202001/campus_network_auth_function.jpg)

接着在 source 栏全文搜索，找到函数的调用定义处（如果 js 文件是一个整行，尝试使用 chrome 自带的功能用 pretty 的方式显示代码）。

![认证代码](/post_img/202001/campus_network_doauthen.jpg)

然后阅读大致代码逻辑，找到 AJAX 相关的代码，在认证成功的回调函数地方打上断点。

![回调函数](/post_img/202001/campus_network_doauthen_callback.jpg)

之后，在页面中输入自己的账号密码，点击登录，最终页面会卡在断点处，查看 network 栏，找到 auth 的接口和数据内容。

![formdata](/post_img/202001/campus_network_formdata.jpg)

综上，我得到了网络认证的接口和我的账户的登录参数。

```bash
#接口地址： http://10.**.**.5/eportal/******.do?method=login
#方法类型： POST
#参数： userId=******&password=**********&service=%25E6%25A0%25A1%25E5%259B%25AD%25E5%25A4%2596%25E7%25BD%2591%25E6%259C%258D%25E5%258A%25A1&queryString=wlanuserip%253D44e4a1ba863c3912ce26bb6a08b619f4%2526wlanacname%253Dd6c254d78aab1b41f824e30727d4a56a%2526ssid%253D%2526nasip%253D33a23ff6f4079ffae1efa910adf8630a%2526snmpagentip%253D%2526mac%253Dac90160b94f74a83bcc03f62577c799a%2526t%253Dwireless-v2%2526url%253D97bcca4b4607b74ecd5a8936b3fedd1caf8c58cf889fa299708ec8a1018a4fd68feb45fec04791db%2526apmac%253D%2526nasid%253Dd6c254d78aab1b41f824e30727d4a56a%2526vid%253D661483fc15f609b3%2526port%253D49738f779d182dee%2526nasportid%253D75961153f3a76ba5cff6c0330a8d162cfde51ccd257f7b51f060e9d8c10167f0&operatorPwd=&operatorUserId=&validcode=&passwordEncrypt=false
```

我原本打算继续分析参数的内容的，结果发现大部分的参数是从自动跳转的登录页面参数中获得的，因此不需要我自己编辑，经过 html decode，我认为参数的含义如下，里面存在部分加密字段，所以这段参数自己生成难度较大：

```bash
userId=*********
&password=*********
&service=校园外网服务
&queryString=
    wlanuserip=44e4a1ba863c3912ce26bb6a08b619f4
    &wlanacname=d6c254d78aab1b41f824e30727d4a56a
    &ssid=
    &nasip=33a23ff6f4079ffae1efa910adf8630a
    &snmpagentip=
    &mac=ac90160b94f74a83bcc03f62577c799a
    &t=wireless-v2
    &url=97bcca4b4607b74ecd5a8936b3fedd1caf8c58cf889fa299708ec8a1018a4fd68feb45fec04791db
    &apmac=
    &nasid=d6c254d78aab1b41f824e30727d4a56a
    &vid=661483fc15f609b3
    &port=49738f779d182dee
    &nasportid=75961153f3a76ba5cff6c0330a8d162cfde51ccd257f7b51f060e9d8c10167f0
&operatorPwd=
&operatorUserId=
&validcode=
&passwordEncrypt=false
```

PS：幸好登录校园网输入验证码的概率非常低（没碰到过），所以我便可以直接使用这段参数通过 `curl` 工具登录校园网。

```bash
curl -d 'userId=********&password=*********&service=%25E6%25A0%25A1%25E5%259B%25AD%25E5%25A4%2596%25E7%25BD%2591%25E6%259C%258D%25E5%258A%25A1&queryString=wlanuserip%253D44e4a1ba863c3912ce26bb6a08b619f4%2526wlanacname%253Dd6c254d78aab1b41f824e30727d4a56a%2526ssid%253D%2526nasip%253D33a23ff6f4079ffae1efa910adf8630a%2526snmpagentip%253D%2526mac%253Dac90160b94f74a83bcc03f62577c799a%2526t%253Dwireless-v2%2526url%253D97bcca4b4607b74ecd5a8936b3fedd1caf8c58cf889fa299708ec8a1018a4fd68feb45fec04791db%2526apmac%253D%2526nasid%253Dd6c254d78aab1b41f824e30727d4a56a%2526vid%253D661483fc15f609b3%2526port%253D49738f779d182dee%2526nasportid%253D75961153f3a76ba5cff6c0330a8d162cfde51ccd257f7b51f060e9d8c10167f0&operatorPwd=&operatorUserId=&validcode=&passwordEncrypt=false' -X POST http://10.**.**.155/eportal/InterFace.do?method=login
```

## 自动登录脚本

为了实现自动登录，我创建一个脚本，逻辑大致如下：

1. 检查网络是否连接状态；
   1. 如果状态是连接，则结束；
   2. 如果状态是断线，则调用网络验证接口，保证自己在线；

### 检查网络连接状态

原本计划用 curl 访问 [百度](http://www.baidu.com)，根据 HTTP 状态码判断是否在线的。实际测试时发现，即便断线状态，返回的状态码也是 200，因为校园网自动跳转的页面返回的状态码就是 200。

然后我在断线的状态下访问百度，看看与在线状态有何不同。

```html
<script>top.self.location.href='http://10.**.**.155/eportal/index.jsp?wlanuserip=747782dfc4c7158565aefbadb92f08f5&wlanacname=4d9b42af64216a4c&ssid=&nasip=33a23ff6f4079ffae1efa910adf8630a&snmpagentip=&mac=e928a1f5c5556f26bb004e5f1879e64b&t=wireless-v2&url=97bcca4b4607b74e02e80da18ea91792d9dd6dcc5d2aa5a3&apmac=&nasid=4d9b42af64216a4c&vid=80e2309e88a7305c&port=ec0e0d680caa36ef&nasportid=75961153f3a76ba5b8d71721a48c6a36d27cfbc13a5acc9ea2aa9e8eb1cf5c88'</script>
```

呵，这一下子就知道校园网自动跳转到登录认证页面的原理了，劫持所有未认证的网络的访问，返回固定的HTML内容，在页面中要求浏览器跳转到认证页面。

这就好办了，保存访问百度的返回内容，检查内容中是否有和登录页面相关的内容，通过对比可以知道当前的网络是已经成功连接，还是断线状态。

```bash
#检测网络链接畅通
function network()
{
    #超时时间
    local timeout=1
    #目标网站
    local target=www.baidu.com
    #获取响应状态码

    local content=`curl http://www.baidu.com | grep eportal/index.jsp`
    if [ ${#content} -lt 10 ]; then
        #网络畅通
        return 1
    else
        #网络不畅通
        return 0
    fi
    return 0
}
```

### 最终自动登录脚本

```bash
#! /bin/bash

LOG_FILE=/var/log/xxxx.log

#检测网络链接畅通
function network()
{
    #超时时间
    local timeout=1
    #目标网站
    local target=www.baidu.com
    #获取响应状态码

    local content=`curl http://www.baidu.com | grep eportal/index.jsp`
    if [ ${#content} -lt 10 ]; then
        #网络畅通
        return 1
    else
        #网络不畅通
        return 0
    fi
    return 0
}

# 执行函数
network

if [ $? -eq 0 ];then
    echo "网络不畅通，请检查网络设置！"
    ret_value=$(curl -d "userId=****&password=****&service=%25E6%25A0%25A1%25E5%259B%25AD%25E5%25A4%2596%25E7%25BD%2591%25E6%259C%258D%25E5%258A%25A1&queryString=wlanuserip%253D747782dfc4c7158565aefbadb92f08f5%2526wlanacname%253D4d9b42af64216a4c%2526ssid%253D%2526nasip%253D33a23ff6f4079ffae1efa910adf8630a%2526snmpagentip%253D%2526mac%253De928a1f5c5556f26bb004e5f1879e64b%2526t%253Dwireless-v2%2526url%253Ddcc9f4c1c6174d15aa0451f13f9102fce3746944e780141b39c9ba28c4ed01fe%2526apmac%253D%2526nasid%253D4d9b42af64216a4c%2526vid%253D80e2309e88a7305c%2526port%253Dec0e0d680caa36ef%2526nasportid%253D75961153f3a76ba5b8d71721a48c6a36d27cfbc13a5acc9ea2aa9e8eb1cf5c88&operatorPwd=&operatorUserId=&validcode=&passwordEncrypt=false" -X POST http://10.**.**.155/eportal/InterFace.do?method=login)

    echo ${ret_value}
    date >> ${LOG_FILE}
    echo "try to login!" >> ${LOG_FILE}
    echo "login result: ${ret_value}" >> ${LOG_FILE}
    exit -1
fi
    echo "网络畅通，你可以上网冲浪！"
exit 0
```

## 定时执行

自动登录脚本写好了，怎么让脚本定时执行呢？使用linux 的 crontab 功能，这个工具可以设置定时程序，按照用户想要的频率定时执行任务。但其默认的编辑工具非常不好用(vi)，所以我先更改了默认的编辑工具。直接执行命令：`select-editor` 选择自己顺手的编辑工具，然后 `crontab -e` 配置定时任务。

```bash
# auto connect
*/5 * * * * /home/pi/auto-connect.sh

# auto clean log
0 0 1 * * /home/pi/clean_log.sh
```

## 总结

通过上述三个流程，我解决了自己的树莓派连接校园网的问题。首先我研究校园网的登录认证方式，找到使用 curl 工具进行登录认证的接口。然后我编写一个自动检查连接状态，调用登录认证接口的脚本代码。最后我设置了自动调用脚本的定时任务，每隔五分钟调用一下脚本，从而保证自己的树莓派一直接入网络。

## Reference

1. [curl 用法](https://www.ruanyifeng.com/blog/2019/09/curl-reference.html)
2. [crontab 的用法](https://www.jianshu.com/p/838db0269fd0)
