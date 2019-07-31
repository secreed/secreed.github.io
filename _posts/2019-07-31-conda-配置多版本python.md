---
type: python
title: conda-配置多版本python
date: 2019-07-31
category: conda

tags:
- [conda,python]
description: conda安装多版本python环境基本操作
---

* 1.查看已安装的环境
`conda env list`
* 2.安装python3
`conda create -n py3 python=3`
* 3.进入环境
```sh 
activate my_env -- win
source activate my_env --  OSX/Linux
activate my_env --  win
```
* 4.离开环境
```sh 
deactivate -- win
source deactivate --  OSX/Linux
```
* 5.删除环境
`conda env remove -n env_name`
* 6.查看当前python环境
`python --version`

