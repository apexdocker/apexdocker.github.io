---
title: "iOS 越狱后安装 ssl-kill-switch2 进行 https 抓包"
date: 2020-11-22T22:06:34+08:00
tags: ['charles']
---

> 今天使用charles开启了SSL抓包，发现有些APP抓不到，有的APP就可以，例如优X二手车，经过google发现抓不到包是因为app使用了SSL Pinning技术

查询发现，越狱的ios只要安装[ssl-kill-switch2](https://github.com/nabla-c0d3/ssl-kill-switch2)就可以抓到了 

#### 1、在Cydia中安装下面3个依赖
- Debian Packager
- Cydia Substrate
- PreferenceLoader

#### 2、下载ssl-kill-switch2.deb
https://github.com/nabla-c0d3/ssl-kill-switch2/releases

#### 3、Mac通过USB连接手机，安装ssl-kill-switch2.deb
ios9 需要通过Cydia安装 openssh
```
brew install libimobiledevice
iproxy 2222 22
scp -P 2222 ~/Downloads/ssl-kill-switch2.deb root@127.0.0.1:~     #输入密码 默认  alpine
ssh -p 2222 root@127.0.0.1
dpkg -i <package>.deb
killall -HUP SpringBoard
```
然后在设置中就会增加一个选项 SSL Kill Switch2，开启后，重启APP即可抓包
