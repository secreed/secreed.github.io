---
type: 工作
title: 业务安全监控系统：session、cookies、token
date: 2020-04-03
category: 探针

tags:
- php
- 工作
description: 业务安全监控系统：session、cookies、token。主要包含session、cookies、token,以及如何进行登陆状态控制
---
>积硅步至千里--总有一天你会看到不一样的风景，当你坚持不懈含着泪水全力向前！



## php配置

1. php.ini 开启 `extension=php_openssl.dll`

2. 生成密钥的配置信息，此处需要设备环境变量，新建 OPENSSL_CONF ，为php下openssl.conf d的路径（php/extra）；php开启openssl；$config["config"]设置成openssl.conf 的路径报错，设置环境变量可以解决该报错

## 配置python

安装anaconda

设置环境变量  anaconda  anaconda/scripts
修改php内调用python程序的路径
换源
```sh
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/conda-forge/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/msys2/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/bioconda/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/menpo/

conda config --set show_channel_urls yes
```

## 配置 HTTrack Website Copier

修改python内调用 HTTrack Website Copier 程序的路径

## session

busGenerate.php 

attackAlarmSearch.php 

## 安全分析

### 安装MySQLdb

管理员权限 运行anaconda  ， 执行 `conda install mysql-python`
