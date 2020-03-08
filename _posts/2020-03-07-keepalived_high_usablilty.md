---
layout:     post
title:      "双机热备份"
subtitle:   "如何通过 keepalived 实现双机热备份"
date:       2020-03-07
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - 服务器
    - 后台开发
---

# 双机热备份

前文我们讲了负载均衡，通过在多个后端服务器前加设一个负载均衡服务器（Load Balancing），对接入的请求分发给后端服务器。可以实现水平扩容，提升总体服务性能的功能；还可以将少量大的宕机后端服务器剔除队列，实现冗余服务器，提高服务稳定性的作用。

**这时机智的小伙伴便会提出疑问：万一负载均衡服务器宕机了怎么办？**

emmm，我们可以在负载均衡服务器前再加一层负载均衡服务器，这样就不用担心负载均衡服务器宕机了（**误！紧致套娃**

实际上双机热备份的思路与负载均衡服务器相似，但具体实现不够一样。双机热备份通过虚拟IP地址（VIP）实现，比如我们有一个公网地址 `VIP 59.80.39.110`，而我们准备了两台负载均衡服务器，分别称为主机和备机`LB_MASTER 192.168.1.100, LB_BACKUP 192.168.1.101`。

我们监控主机的状态，当主机正常时，将访问 VIP 的流量发送到 `LB_MASTER` 机器，也可以说是`LB_MASTER`绑定了这个 VIP。当主机的负载均衡服务器出现故障，主机便调用程序取消这个 VIP 来防止流量继续流入主机；备机检测到主机故障，便选举自己成为新的主机，为自己设置这个 VIP 地址，这样后续的流量便会转发到`LB_BACKUP`。

**注：** 一个电脑可以同时拥有很多个 IP 地址，这种方法便是首先准备一个富余的IP地址，通过给机器设置服务，监控 LB 服务器的运行状态，动态的将这个 富余的 IP 地址绑定到备用机器上。这样DNS服务器不用做任何修改，客户端也不用做任何修改，就可以自动的将服务由主机切换到备机。

![](/post_img/202001/keepalived.gif)

# keepalived
keepalived 是一个流行的开源的基于 vrrp 协议的轻量级高可用软件。

KeepAlived是基于VRRP(Virtual Router Redundancy Protocol，虚拟路由冗余协议)实现的一个高可用方案，通过VIP（虚拟IP）和心跳检测来实现高可用。

Keepalived有两个角色，Master和Backup。一般会是１个Master,多个Backup。

Master会绑定VIP到自己网卡上，对外提供服务。Master和Backup会定时确定对方状态，当Master不可用的时候，Backup会通知网关，并把VIP绑定到自己的网卡上，实现服务不中断，高可用。


下面是 keepalived 配置文件参考，对于云服务器，通常云服务器的路由会关闭组播功能，因此需要指定主机和备机的 ip ，进行单播通信。这是个注意点，要考
```
# 主机的配置文件， 假设主机 ip 为 192.168.100
! Configuration File for keepalived

global_defs {
   # 通知邮件服务器的配置
   notification_email {
     # 当master失去VIP或则VIP的时候，会发一封通知邮件到your-email@qq.com
     your-email@qq.com
   }
   # 发件人信息
   notification_email_from keepalived@qq.com
   # 邮件服务器地址
   smtp_server 127.0.0.1
   # 邮件服务器超时时间
   smtp_connect_timeout 30
   # 邮件TITLE
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    # 主机: MASTER
    # 备机: BACKUP
    state MASTER
    # 实例绑定的网卡, 用ip a命令查看网卡编号
    interface eth0
    # 虚拟路由标识，这个标识是一个数字(1-255)，在一个VRRP实例中主备服务器ID必须一样
    virtual_router_id 88
    # 优先级，数字越大优先级越高，在一个实例中主服务器优先级要高于备服务器
    priority 100
    # 主备之间同步检查的时间间隔单位秒
    advert_int 1
    # 验证类型和密码
    authentication {
        # 验证类型有两种 PASS和HA
        auth_type PASS
        # 验证密码，在一个实例中主备密码保持一样
        auth_pass 11111111
    }

    # 指定通信的 ip ，设置 keepalived 为单播通信
    unicast_src_ip 192.168.1.100
    unicast_peer {
        192.168.1.101
        # 可以设置多个 IP
    }
    # 虚拟IP地址,可以有多个，每行一个
    virtual_ipaddress {
        59.80.39.110
    }
}
```

```
# 备机的配置文件
! Configuration File for keepalived

global_defs {
   # 通知邮件服务器的配置
   notification_email {
     # 当master失去VIP或则VIP的时候，会发一封通知邮件到your-email@qq.com
     your-email@qq.com
   }
   # 发件人信息
   notification_email_from keepalived@qq.com
   # 邮件服务器地址
   smtp_server 127.0.0.1
   # 邮件服务器超时时间
   smtp_connect_timeout 30
   # 邮件TITLE
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    # 主机: MASTER
    # 备机: BACKUP
    state BACKUP
    # 实例绑定的网卡, 用ip a命令查看网卡编号
    interface etho
    # 虚拟路由标识，这个标识是一个数字(1-255)，在一个VRRP实例中主备服务器ID必须一样
    virtual_router_id 88
    # 优先级，数字越大优先级越高，在一个实例中主服务器优先级要高于备服务器
    priority 99
    # 主备之间同步检查的时间间隔单位秒
    advert_int 1
    # 验证类型和密码
    authentication {
        # 验证类型有两种 PASS和HA
        auth_type PASS
        # 验证密码，在一个实例中主备密码保持一样
        auth_pass 11111111
    }

    unicast_src_ip 192.168.1.101
    unicast_peer {
        192.168.1.100
    }

    # 虚拟IP地址,可以有多个，每行一个
    virtual_ipaddress {
        59.80.39.110
    }
}
```


# 参考链接
1. [负载均衡](https://juejin.im/post/5e63b434e51d4526fd0692b6)
2. [keepalived 配置文件参考](https://yq.aliyun.com/articles/609851)
