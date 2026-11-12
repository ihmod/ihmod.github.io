---
title: kali搭建Nessus
tags:
  - 环境搭建
categories: 网安
top_img: false
description: kali搭建Nessus
abbrlink: d672923f
---

>请直接在kali里面操作，否则需要安装vmtools，且有传输的现象

### 软件

下载地址：https://www.tenable.com/downloads/nessus



### 安装

```
dpkg -i ./Nessus-10.1.1-debian6_amd64.deb
```

### 启动

```
systemctl start nessusd
```

### 访问

https://<ip>:8843/#/

勾选

continue

Managed Scanner

Tenable.s.c

设置账号密码（root:root）

等初始化完成进入

### 离线激活

- 获取激活码

```
  注册地址  https://zh-cn.tenable.com/products/nessus/nessus-essentials
  临时邮箱  https://mail.cx/zh/
```

- 获取质询码

```
/opt/nessus/sbin/nessuscli fetch --challenge
```

- 获取插件包和激活证书

```
https://plugins.nessus.org/v2/offline.php
```

- 安装插件

```
  service nessusd stop
  
  /opt/nessus/sbin/nessuscli update all-2.0.tar.gz
```
- 离线激活

```
  /opt/nessus/sbin/nessuscli fetch --register-offline nessus.license
```

- 家庭版升至专业版
```
https://plugins.nessus.org/v2/plugins.php  #查看版本
```

在`/opt/nessus/lib/nessus/plugins`和`/opt/nessus/var/nessus`目录下分别创建一个`plugin_feed_info.inc`，内容如下


```
PLUGIN SET = "202311130938"; 
PLUGIN_FEED ="ProfesslonalFeed (Direct)";
PLUGIN_FEED_TRANSPORT = "Tenable Network Security Lightning";


#第一项的数字来自上面网址
```


- 重启Nessus

```
service nessusd restart
```
