---
layout: post
title: "虚机复活记"
keywords: ["git pages"]
description: "hehe"
category: misc
tags: [git]
---
{% include JB/setup %}

因为需要编译spark&shark，唤醒了沉睡许久的虚拟机。用的是VirtualBox，系统装的是centos6。唤醒之路记录一下……

### 密码

密码不记得了[看这里](http://sdbaby.blog.51cto.com/149645/325242) 。系统启动的时候按e，折腾几下进入root然后修改密码

### 上网和ssh连接

上网比较麻烦。因为是在公司内部，如果走网卡桥接，公司内部路由器不给上。所以走NAT，这时候走NAT无法从宿主机ssh连过去，这时候就要设置一个端口转发即可

![nat-redirect]({{ IMAGE_PATH }}/nat-port-redirect.jpg)

默认ssh端口是22，防火墙是开了的。如果是别的端口就可能需要开防火墙了。iptables防火墙设置看[这里](http://my.oschina.net/blindcat/blog/169657)

开端口映射是不需要重启虚机的。







