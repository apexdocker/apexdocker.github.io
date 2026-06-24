---
title: "Android 通过 VirtualXposed 进行 Charles https 抓包"
date: 2020-11-25T11:16:41+08:00
tags: ['charles']
---

> Android7.0以上，无需root，即可抓包的方法

由于Android的机制，7.0以上不再信任用户证书，如[Charles](https://www.charlesproxy.com/documentation/using-charles/ssl-certificates/)中的介绍

所以有3种方法可以解决这个问题：
- 最暴力的，root，把证书加入到系统证书
- 如果是自家APP，参考上面链接，在APP中设置信任用户证书
- 通过VirtualXposed+插件绕过证书检查（本文）

### 1、手机安装VirtualXposed

不多介绍，直接下载安装
地址：https://github.com/android-hacker/VirtualXposed/releases

### 2、VirtualXposed中安装 JustTrustMe 或  SSLUnpinning 用于绕过 SSL 证书检查

- JustTrustMe：https://github.com/Fuzion24/JustTrustMe/releases
- TrustMeAlready：https://github.com/ViRb3/TrustMeAlready
- SSLUnpinning：https://github.com/ac-pm/SSLUnpinning_Xposed/blob/master/mobi.acpm.sslunpinning_latest.apk

在VirtualXposed中安装apk并启用模块
并在设置中，点击重启以使模块生效

### 3、VirtualXposed中安装目标apk

在VirtualXposed中安装目标apk，并配置好抓包工具，从VirtualXposed中启动APP开始抓包
