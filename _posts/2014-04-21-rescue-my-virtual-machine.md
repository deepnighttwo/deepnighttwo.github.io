---
layout: post
title: "虚机复活记"
keywords: ["linux,CentOS,virtual-box"]
description: ""
category: devtools
tags: [linux, CentOS, virtual-box]
---

因为需要编译spark&shark，唤醒了沉睡许久的虚拟机。用的是VirtualBox，系统装的是centos6。唤醒之路记录一下……

### 密码

密码不记得了[看这里](http://sdbaby.blog.51cto.com/149645/325242) 。系统启动的时候按e，折腾几下进入root然后修改密码

### 上网和ssh连接

上网比较麻烦。因为是在公司内部，如果走网卡桥接，公司内部路由器不给上。所以走NAT，这时候走NAT无法从宿主机ssh连过去，这时候就要设置一个端口转发即可

![nat-redirect](/images/nat-port-redirect.jpg)

默认ssh端口是22，防火墙是开了的。如果是别的端口就可能需要开防火墙了。iptables防火墙设置看[这里](http://my.oschina.net/blindcat/blog/169657)

开端口映射是不需要重启虚机的。

### yum repo

不知道为什么，yum 无法安装软件了。报错说

```bash
removing mirrorlist with no valid mirrors: /var/cache/yum/x86_64/6/base/mirrorlist.txt
```

查了一下是/var/cache/yum/x86_64/6/base/mirrorlist.txt这个文件找不到，这个里面放的是mirror list。创建出来就可以了，具体的内容可以参照[这里](http://gardenyuan.iteye.com/blog/1498032)

除了这个文件，还有一个/var/cache/yum/x86_64/6/update/mirrorlist.txt，也需要一起改一下。

到这里基本虚机就复活了～
{% include references.md %}

