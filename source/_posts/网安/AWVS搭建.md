---
title: kali搭建AWVS
tags:
  - 环境搭建
categories: 网安
top_img: false
description: 手把手kali搭建AWVS
abbrlink: 29e10dfb
---
### 下载资源

链接

### 解压安装
```
chmod 777 acunetix_trial.sh

bash acunetix_trial.sh
```
enter 再按 Q 输入yes回车

输入主机 、账号、密码就可以了
### 破解补丁

```
cp patch_awvs /home/acunetix/.acunetix_trial/v_190325161/scanner/
chmod 777 /home/acunetix/.acunetix_trial/v_190325161/scanner/patch_awvs
/home/acunetix/.acunetix_trial/v_190325161/scanner/patch_awvs
```

给文件上锁

```
chattr +i /home/acunetix/.acunetix_trial/data/license/license_info.json
rm -fr /home/acunetix/.acunetix_trial/data/license/wa_data.dat
touch /home/acunetix/.acunetix_trial/data/license/wa_data.dat
chattr +i  /home/acunetix/.acunetix_trial/data/license/wa_data.dat
```

### 重启服务

```
systemctl restart acunetix_trial.service
```

### 卸载

```
systemctl stop acunetix_trial.service
```

```
cd /home/acunetix/.acunetix_trial/
./uninstall.sh
提示两个文件删不掉
选yes回车
```
```
解锁文件
chattr -i /home/acunetix/.acunetix_trial/data/license/license_info.json
chattr -i  /home/acunetix/.acunetix_trial/data/license/wa_data.dat
单独删掉
rm -fr /home/acunetix/.acunetix_trial/data/license/license_info.json
rm -fr /home/acunetix/.acunetix_trial/data/license/wa_data.dat
删除用户
userdel  acunetix
```

